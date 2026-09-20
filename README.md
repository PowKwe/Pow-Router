# Tailscale Kernel-Mode Exit Node / Subnet Router on Rooted Android

Turns a rooted Android device (tested on a Samsung Galaxy S10+, Exynos, Magisk-rooted) into a genuine **kernel-mode** Tailscale subnet router and exit node — a real `tailscale0` TUN interface with in-kernel routing and NAT, not Tailscale's `userspace-networking` fallback mode.

This matters because stock Android and stock Tailscale binaries don't get along at the routing layer. This repo documents the fix and ships a working boot-persistence script.

## Why this is harder than it looks on Linux

On a normal Linux box, Tailscale's router code inserts a handful of `ip rule` entries and everything works. Android breaks this in two independent ways:

1. **fwmark collisions.** Android's `netd` reserves the lower bits of the 32-bit socket fwmark for its own per-network/per-UID bookkeeping. Stock Tailscale's default fwmark allocation overlaps that reserved range, corrupting or being corrupted by Android's own packet marking.
2. **Policy-routing (RPDB) interference.** Android maintains dozens of `ip rule` entries per network/UID range at priorities generally ≥ 10000, used for per-app VPN routing, metered-connection logic, and multi-network handling. Stock Tailscale's rule-insertion logic assumes a much simpler Linux routing setup and installs rules at priorities that either get shadowed by Android's rules, or — worse — bypass them entirely and corrupt Android's own routing decisions for unrelated traffic.

The result: stock `tailscaled` either silently falls back to non-functional routing, or actively interferes with the phone's normal network behavior.

## The fix

This setup relies on a patched Android-aware fork of Tailscale ([`android-kxxt/external_tailscale`](https://github.com/android-kxxt/external_tailscale)) that:
- Remaps Tailscale's fwmark bits to the unclaimed bit range Android's `netd` leaves free, avoiding collision.
- Replaces Tailscale's default multi-rule `ip rule` insertion with a single simplified rule placed at a priority that doesn't shadow Android's own rules.
- Patches `go-iptables` to tolerate Android's non-standard iptables error strings.

On top of the patched binary, **one additional fix was required on-device** that isn't part of the upstream patch: an explicit `ip rule` telling the kernel to resolve forwarded `tailscale0` traffic against the `main` routing table. Without it, packets arrive on the tun interface and are correctly accepted into the `FORWARD` chain by iptables, but the kernel has no rule that resolves a next-hop route for them, so nothing ever actually gets forwarded — despite every iptables rule looking correct. This is the fix that took the router from "device shows exit node offered, client shows connected, zero actual internet passes through" to working.

## Requirements

- Rooted Android device with Magisk
- Root shell access (via Termux + `su`, or ADB)
- A prebuilt or self-built `tailscale` / `tailscaled` binary from the patched fork, matching your device's ABI (arm64 for most modern devices)
- A Tailscale account

## Installation

1. **Get the patched binaries.**
   Either download a prebuilt release matching your ABI, or build from source:
   ```sh
   git clone https://github.com/android-kxxt/external_tailscale.git
   cd external_tailscale
   GOOS=android GOARCH=arm64 CGO_ENABLED=0 go build -o tailscaled ./cmd/tailscaled
   GOOS=android GOARCH=arm64 CGO_ENABLED=0 go build -o tailscale ./cmd/tailscale
   ```
   Verify the fork isn't badly stale against current Tailscale releases — an old enough build may fail to authenticate against the current control-plane protocol.

2. **Push the binaries to persistent storage on-device:**
   ```sh
   su -c 'mkdir -p /data/adb/tailscale'
   su -c 'cp tailscale tailscaled /data/adb/tailscale/'
   su -c 'chmod 755 /data/adb/tailscale/tailscale /data/adb/tailscale/tailscaled'
   su -c 'chown root:root /data/adb/tailscale/tailscale /data/adb/tailscale/tailscaled'
   ```

3. **Install the boot script.**
   Copy `tailscale.sh` (below) to `/data/adb/service.d/tailscale.sh` and mark it executable:
   ```sh
   chmod 755 /data/adb/service.d/tailscale.sh
   ```
   Magisk's `service.d` scripts run post-boot, after CE storage is decrypted — this is what you want for a daemon that needs persistent state.

4. **Reboot**, then confirm the kernel tun device exists:
   ```sh
   ip link show tailscale0
   ```

5. **Authenticate and configure the router** (one-time, via a root shell):
   ```sh
   /data/adb/tailscale/tailscale --socket=/data/adb/tailscale/tailscaled.sock up \
     --accept-dns=false \
     --advertise-routes=192.168.X.0/24 \
     --advertise-exit-node
   ```
   Open the printed login URL and approve the device. Settings persist in the daemon's state file — you won't need to run `up` again after reboots.

6. **Approve the route and exit-node capability** in the [Tailscale admin console](https://login.tailscale.com/admin/machines) — advertising isn't the same as active; both need manual approval before any client can actually use them.

## Boot persistence script

```sh
#!/system/bin/sh
until [ "$(getprop sys.boot_completed)" = "1" ]; do sleep 1; done
sleep 10

# 1. Enable IP forwarding for IPv4 and IPv6
echo 1 > /proc/sys/net/ipv4/ip_forward
echo 1 > /proc/sys/net/ipv6/conf/all/forwarding

# 2. Disable Reverse Path Filtering to prevent Android from dropping packets
echo 0 > /proc/sys/net/ipv4/conf/all/rp_filter
echo 0 > /proc/sys/net/ipv4/conf/default/rp_filter
echo 0 > /proc/sys/net/ipv4/conf/wlan0/rp_filter

# 3. Route forwarded tailscale0 traffic via the main table — the key fix.
#    Without this, forwarded packets arriving on tailscale0 have no rule
#    that resolves a next-hop route, so they never leave the FORWARD chain
#    despite every iptables rule being correct.
ip rule add iif tailscale0 lookup main pref 4999 2>/dev/null

# 4. Configure NAT and allow forwarding for the tailscale0 interface
iptables -t nat -I POSTROUTING -o wlan0 -j MASQUERADE
iptables -I FORWARD -i tailscale0 -j ACCEPT
iptables -I FORWARD -o tailscale0 -j ACCEPT

# 5. Launch Tailscale in true kernel mode (no --tun flag = real tun device)
env XDG_CACHE_HOME=/data/adb/tailscale \
/data/adb/tailscale/tailscaled \
--state=/data/adb/tailscale/tailscaled.state \
--socket=/data/adb/tailscale/tailscaled.sock &

# 6. Wait for the tailscale0 interface to initialize, then disable its rp_filter
sleep 5
echo 0 > /proc/sys/net/ipv4/conf/tailscale0/rp_filter 2>/dev/null

# 7. Launch Termux sshd for remote administration
TERMUX_UID=$(stat -c '%u' /data/data/com.termux)
su ${TERMUX_UID} -c 'export PATH=/data/data/com.termux/files/usr/bin:/system/bin; export PREFIX=/data/data/com.termux/files/usr; export HOME=/data/data/com.termux/files/home; export LD_PRELOAD=/data/data/com.termux/files/usr/lib/libtermux-exec.so; /data/data/com.termux/files/usr/bin/sshd'
```

Adjust `wlan0` throughout if your device's uplink interface has a different name (check with `ip route get 8.8.8.8`).

## Verification

Confirm the setup is actually forwarding traffic, not just configured to:

```sh
# Policy routing: should show Tailscale's fwmark rule below Android's netd rules
ip rule

# Tailscale's own routing table
ip route show table 52

# Daemon status and peer connectivity
/data/adb/tailscale/tailscale --socket=/data/adb/tailscale/tailscaled.sock status

# The real proof: FORWARD and NAT counters should climb under active client load,
# not just exist
iptables -L FORWARD -n -v
iptables -t nat -L POSTROUTING -n -v
```

A route being "approved" in the admin console and a route being "actually forwarding packets" are different things — always confirm with nonzero, growing packet counters under real client traffic, not just green checkmarks in the UI.

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| `avc: denied` in `dmesg`/`logcat` | SELinux blocking netlink/iptables calls. Patch with `magiskpolicy --live "permissive <domain>"` for the specific denied domain rather than a blanket `setenforce 0`. |
| `tailscale0` never appears | Daemon crashed or fell back silently — check `logcat` for the process, confirm binary matches device ABI. |
| Client shows exit node selected but no internet, FORWARD counters stay at zero | Missing `ip rule ... lookup main` for forwarded `tailscale0` traffic (see the boot script above) — packets arrive but have no route to resolve. |
| `ts-input` counters climb but `ts-forward` stays at zero | Traffic is reaching the device as its *destination*, not passing through it — confirms the client isn't actually routing through this node yet, look at the client side. |
| Client-side: exit node shows "connected" but nothing routes | On non-rooted Android clients, check battery optimization isn't throttling the Tailscale background service (Samsung devices are notably aggressive here), and confirm no conflicting VPN/private-DNS app is registered. |

## Credits

- [Tailscale](https://github.com/tailscale/tailscale) — upstream project
- [android-kxxt/external_tailscale](https://github.com/android-kxxt/external_tailscale) — the Android fwmark/IP-rule/go-iptables patches this setup depends on

## Disclaimer

This runs a modified, community-patched build of Tailscale outside its officially supported deployment path, on a rooted device with SELinux partially set to permissive. Understand the security tradeoffs before exposing this as an exit node to devices or people you don't fully trust.
