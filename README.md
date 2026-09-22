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
- [**Termux:Boot**](https://github.com/termux/termux-boot) (from the same source as your Termux install — F-Droid and Play Store builds are not always signature-compatible with each other, so mismatched sources will fail to talk to each other)

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

3. **Install the Magisk boot script.**
   Copy `master-boot.sh` (below) to `/data/adb/service.d/tailscale.sh` and mark it executable:
   ```sh
   chmod 755 /data/adb/service.d/tailscale.sh
   ```
   Magisk's `service.d` scripts run post-boot, after CE storage is decrypted — this is what you want for a daemon that needs persistent state.

4. **Install Termux:Boot and the sshd boot script** (see [SSH access](#ssh-access) below) — do this now, before your first reboot, so both pieces are in place together.

5. **Reboot**, then confirm the kernel tun device exists:
   ```sh
   ip link show tailscale0
   ```

6. **Authenticate and configure the router** (one-time, via a root shell):
   ```sh
   /data/adb/tailscale/tailscale --socket=/data/adb/tailscale/tailscaled.sock up \
     --accept-dns=false \
     --advertise-routes=192.168.X.0/24 \
     --advertise-exit-node
   ```
   Open the printed login URL and approve the device. Settings persist in the daemon's state file — you won't need to run `up` again after reboots.

7. **Approve the route and exit-node capability** in the [Tailscale admin console](https://login.tailscale.com/admin/machines) — advertising isn't the same as active; both need manual approval before any client can actually use them.

## Master boot script (`master-boot.sh`)

Install at `/data/adb/service.d/tailscale.sh`. Handles kernel forwarding, NAT, and launching `tailscaled` — **does not** launch sshd (see [SSH access](#ssh-access) for why).

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

# 3. Determine the dynamic Android routing table for wlan0
IDX=$(cat /sys/class/net/wlan0/ifindex)
TABLE=$((1000 + IDX))

# 4. Add policy routing rules for Tailscale traffic
ip rule add iif tailscale0 lookup main pref 4999 2>/dev/null
ip rule add iif tailscale0 lookup $TABLE pref 5000 2>/dev/null

# 5. Configure NAT and allow forwarding for the tailscale0 interface
iptables -t nat -I POSTROUTING -o wlan0 -j MASQUERADE
iptables -I FORWARD -i tailscale0 -j ACCEPT
iptables -I FORWARD -o tailscale0 -j ACCEPT

# 6. Launch Tailscale in True Kernel Mode
env XDG_CACHE_HOME=/data/adb/tailscale \
/data/adb/tailscale/tailscaled \
--state=/data/adb/tailscale/tailscaled.state \
--socket=/data/adb/tailscale/tailscaled.sock &

# 7. Wait for the tailscale0 interface to initialize, then disable its rp_filter
sleep 5
echo 0 > /proc/sys/net/ipv4/conf/tailscale0/rp_filter 2>/dev/null
```

Adjust `wlan0` throughout if your device's uplink interface has a different name (check with `ip route get 8.8.8.8`).

> **Note on rule 4:** the `pref 5000` dynamic-table rule is redundant in practice — the `pref 4999 lookup main` rule above it matches first and resolves the route, so the dynamic table is rarely if ever consulted. It's kept here for parity with the original working setup; you can drop it safely if you want one less moving part (tested working without it).

## SSH access

**Don't launch sshd from the Magisk `service.d` script.** Doing so via `su ${TERMUX_UID} -c ...` switches the process to Termux's UID number, but the resulting process does **not** get the same Linux group memberships or SELinux context a normally-launched Termux app process gets:

| | Normal Termux launch | `su <uid>` from Magisk script |
|---|---|---|
| Groups | `...,3003(inet),9997(everybody),...` | *(none — just the base UID group)* |
| SELinux context | `u:r:untrusted_app_27:s0:...` | `u:r:magisk:s0` |

Critically, it's missing the `inet` group (gid 3003), which gates permission to create network sockets the way Android expects for an app process. In practice this causes exactly the symptom that prompted this section: SSH into a `service.d`-launched sshd and `pkg update`/`pkg install` fail with "all mirrors bad," while the same commands work fine the moment you physically open Termux and start sshd from inside it — because *that* process goes through Android's normal Zygote-fork app-launch path and gets the correct groups and context.

**Fix: launch sshd through Termux's own boot mechanism (Termux:Boot) instead of through Magisk.**

1. Install [Termux:Boot](https://github.com/termux/termux-boot) from the same source (F-Droid or Play Store, matching whichever you used for Termux itself).
2. Open the Termux:Boot app once so Android registers its boot receiver.
3. In Termux (physically opened):
   ```sh
   mkdir -p ~/.termux/boot
   ```
   Create `~/.termux/boot/start-sshd.sh`:
   ```sh
   #!/data/data/com.termux/files/usr/bin/sh
   sshd
   ```
   ```sh
   chmod +x ~/.termux/boot/start-sshd.sh
   ```
4. Reboot and test SSH **without** opening Termux on-screen first — that's the real test that the daemon is launching through the correct app-context path.

If Termux:Boot proves unreliable on your specific OEM skin (a known occasional issue on some Android builds, since it depends on the boot-completed broadcast reaching a non-foreground app reliably), the fallback is opening Termux manually once per reboot before relying on SSH-triggered package operations.

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
| Complete internet loss ("no internet" wifi warning), fixed only by full reboot (not wifi toggle) | Possible `nf_conntrack` table exhaustion from sustained exit-node traffic. Check `/proc/sys/net/netfilter/nf_conntrack_count` vs `nf_conntrack_max`; raise the max if pegged, and add it to the boot script. |
| Router's own outbound traffic works by raw IP (`ping 8.8.8.8`) but not by hostname (`ping google.com`) | DNS resolution issue, unrelated to routing. See below. |
| `pkg update`/`pkg install` fail with "all mirrors bad" over SSH, but work fine when Termux is opened physically on-device | The sshd process launched via Magisk `service.d`/`su <uid>` is missing the `inet` group and correct SELinux app context. See [SSH access](#ssh-access) above. |

### DNS troubleshooting deep-dive

If hostname resolution fails device-wide (not just over SSH) while raw-IP connectivity works, check in this order:
1. `dig google.com @8.8.8.8` — if this works, DNS traffic itself is fine; the problem is in Android's automatic resolver selection.
2. Check Settings → Connections → Private DNS. If it's set to "Automatic," Android may be attempting DNS-over-TLS against your network's DHCP-assigned DNS servers — which can hang if those servers don't actually support DoT. This is common on **CGNAT ISP connections**: if your router's WAN IP is itself in `100.64.0.0/10`, your ISP may hand out internal-only DNS resolvers (also in that range) that are unreachable from LAN clients and don't speak DoT.
3. Fix at the router: find the LAN-side DHCP settings and set static public DNS servers (`1.1.1.1`, `8.8.8.8`) instead of passing through WAN-assigned DNS.
4. Fix on-device as a workaround: Wi-Fi → network settings → Advanced → IP settings → Static, and set DNS 1/DNS 2 manually to `8.8.8.8`/`1.1.1.1`.
5. This is unrelated to the Tailscale/kernel-routing setup — CGNAT on your ISP's WAN connection does not interfere with Tailscale itself (Tailscale is designed to work behind CGNAT via DERP relay/direct connections), only with Android's own DNS server selection.

## Credits

- [Tailscale](https://github.com/tailscale/tailscale) — upstream project
- [android-kxxt/external_tailscale](https://github.com/android-kxxt/external_tailscale) — the Android fwmark/IP-rule/go-iptables patches this setup depends on
- [Termux](https://github.com/termux) / [Termux:Boot](https://github.com/termux/termux-boot)

## Disclaimer

This runs a modified, community-patched build of Tailscale outside its officially supported deployment path, on a rooted device with SELinux partially set to permissive. Understand the security tradeoffs before exposing this as an exit node to devices or people you don't fully trust.
