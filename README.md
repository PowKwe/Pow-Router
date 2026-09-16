# Android Tailscale Exit Node & Subnet Router

This repository documents the configuration of a rooted Android device acting as a Tailscale exit node and subnet router, providing secure remote access to homelab resources.

## Tech Stack
*   **OS:** Rooted Android
*   **Environment:** Termux, Magisk
*   **Networking:** Tailscale, `iptables` (Kernel Routing), OpenSSH

## Architecture Overview
The Android device utilizes root permissions granted by Magisk to run advanced network routing. Custom `iptables` rules and kernel parameter adjustments enable the device to forward traffic between the Tailscale mesh network (tailnet) and the local homelab subnet seamlessly.

## Boot Persistence (`master-boot.sh`)
To ensure the routing rules, Tailscale daemon, and SSH server automatically start upon device reboot, the following script is placed in `/data/adb/service.d/master-boot.sh` (executed by Magisk during late-start service boot). 

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
