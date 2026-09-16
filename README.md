# Android Tailscale Exit Node & Subnet Router

This repository documents the configuration of a rooted Android device acting as a Tailscale exit node and subnet router, providing secure remote access to homelab resources.

## Tech Stack

- **OS:** Rooted Android
- **Environment:** Termux, Magisk
- **Networking:** Tailscale, iptables (kernel routing), OpenSSH

## Architecture Overview

The Android device uses root permissions granted by Magisk to run advanced network routing. Custom iptables rules and kernel parameter adjustments enable the device to forward traffic between the Tailscale mesh network (tailnet) and the local homelab subnet seamlessly.

## Prerequisites

- Android device with root access (Magisk installed)
- Termux installed with `su` permissions granted
- Native Linux `tailscaled` and `tailscale` binaries placed in `/data/adb/tailscale/`

## Native Execution & Dynamic Kernel Routing

This setup bypasses the standard Tailscale Android app. Instead of relying on Android's restricted VpnService API, which limits true exit node and subnet routing capabilities, this project runs the native Linux `tailscaled` daemon directly from `/data/adb/tailscale`. By executing the binaries locally via Magisk, the Android device functions identically to a standard bare-metal Linux server.

Traffic routing is handled entirely at the kernel level using iptables. The NAT configuration (applied automatically on boot) uses a MASQUERADE rule bound to the wireless interface (`wlan0`). Because MASQUERADE dynamically maps the source IP of forwarded packets to the current IP address of the interface, the setup is network-agnostic: if the router is relocated to a new Wi-Fi network with a different subnet, the kernel automatically adapts to the new DHCP-assigned IP address. The iptables rules never need to be rewritten or reloaded, making the node instantly portable.

## Managing Subnet Routes

Because the Tailscale daemon runs natively from a custom path, all CLI commands must explicitly point to the custom socket.

To advertise local subnets (or maintain existing ones) alongside exit node functionality, pass a comma-separated list of CIDR blocks to `--advertise-routes`:

```bash
/data/adb/tailscale/tailscale --socket=/data/adb/tailscale/tailscaled.sock up \
  --advertise-routes=192.168.101.0/24,192.168.1.0/24 \
  --advertise-exit-node
```

- **To add a subnet:** append the new CIDR block to the comma-separated list and re-run the command.
- **To remove a subnet:** remove its CIDR block from the list and re-run the command.

> **Note:** always include `--advertise-exit-node` if you want the device to keep routing general internet traffic for the tailnet.

After modifying routes via the CLI, log into the Tailscale Admin Console, navigate to the **Machines** tab, select this device, and explicitly approve the newly advertised routes under **Edit route settings**.

## Boot Persistence (`master-boot.sh`)

To ensure the routing rules, Tailscale daemon, and SSH server start automatically on reboot, the following script lives at `/data/adb/service.d/master-boot.sh`, executed by Magisk during late-start service boot.

It waits for the Android boot process to complete, applies the NAT rules, initializes `tailscaled`, and safely launches `sshd` under the Termux user environment.

```bash
#!/system/bin/sh
until [ "$(getprop sys.boot_completed)" = "1" ]; do sleep 1; done
sleep 10

# Enable Kernel IP Forwarding and NAT
echo 1 > /proc/sys/net/ipv4/ip_forward
iptables -t nat -I POSTROUTING -o wlan0 -j MASQUERADE

# Launch Tailscale
env XDG_CACHE_HOME=/data/adb/tailscale \
/data/adb/tailscale/tailscaled \
--state=/data/adb/tailscale/tailscaled.state \
--socket=/data/adb/tailscale/tailscaled.sock &

# Launch SSHD with strict absolute paths and library preloads
TERMUX_UID=$(stat -c '%u' /data/data/com.termux)
su ${TERMUX_UID} -c 'export PATH=/data/data/com.termux/files/usr/bin:/system/bin; export PREFIX=/data/data/com.termux/files/usr; export HOME=/data/data/com.termux/files/home; export LD_PRELOAD=/data/data/com.termux/files/usr/lib/libtermux-exec.so; /data/data/com.termux/files/usr/bin/sshd'
```
