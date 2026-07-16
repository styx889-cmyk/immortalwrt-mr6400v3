# ImmortalWrt for TP-Link TL-MR6400 (EU) V3

Builds a minimal ImmortalWrt firmware image via GitHub Actions, targeting
TP-Link TL-MR6400 (EU) V3 hardware — MediaTek MT7628N, 64MB RAM, 8MB Flash.

## Why this profile, and the risk you should know about

TL-MR6400 V3 has **no official OpenWrt/ImmortalWrt device profile**. This repo
builds using `tplink_tl-mr6400-v4`, which is natively supported by
ImmortalWrt (no source patches required). The reasoning:

- v4 is the same product line/hardware revision, not a different model —
  a stronger match than reusing an unrelated model like the TL-MR3420.
- v4's device packages already assume the same internal USB-attached LTE
  modem wiring this board almost certainly has (`kmod-usb-net-qmi-wwan`,
  `uqmi`), which we strip below since LTE isn't needed, but could be
  re-added later.

**This is still unverified for V3 specifically.** What's actually confirmed
by the community:

- An OpenWrt forum moderator [explicitly warned](https://forum.openwrt.org/t/tl-mr6400-eu-v3-help-needed/188476)
  against flashing firmware built for a different device unless the wiki
  states it's compatible — the original poster in that thread never
  confirmed a working method, for fear of bricking.
- In a [separate thread](https://forum.openwrt.org/t/openwrt-for-tp-link-tl-mr6400-v3/87918),
  a user reported success **compiling from source** with proper V3 device-tree
  recognition (including a working LTE modem) — this repo does not do that,
  it uses ImageBuilder against the v4 profile for speed/simplicity.
- [aaronjauf/openwrt-tplink-tl-mr6400-v3](https://github.com/aaronjauf/openwrt-tplink-tl-mr6400-v3)
  is a community repo that also builds against `tplink_tl-mr6400-v4` (old
  OpenWrt 21.02.1 ImageBuilder) and has been running scheduled builds — the
  closest thing to prior art for this exact approach.

If you want a higher-confidence path, the source-compiled V3-specific build
mentioned above is worth tracking down before flashing this image.

## Building

Actions tab → **Build ImmortalWrt firmware (TL-MR6400 v4 profile)** →
*Run workflow*. Optionally override the ImmortalWrt version (defaults to
`25.12.1`). Takes a few minutes (uses ImageBuilder, not a full source
compile). Output `.bin` files are attached as a workflow artifact.

Package set: `luci`, `luci-app-wireguard` (+ WireGuard kmod/tools), LAN and
WiFi (already in the target's defaults). Stripped: USB, LTE/QMI modem
support, PPP — none of it is needed for this use case.

## Flashing — do this carefully

1. **Have a recovery path before you start.** Ideally serial/UART console
   access to the board (U-Boot), which lets you recover regardless of what
   the OEM firmware's upgrade validator does. Don't attempt this if you
   can't tolerate the router becoming unusable.
2. Try the **stock TP-Link web UI firmware upgrade** first with the
   `*-squashfs-sysupgrade.bin` file — worst case it rejects the file for a
   hardware ID mismatch, which is non-destructive. Don't force it through
   if it warns about a mismatch.
3. If that's rejected, research the TFTP recovery procedure for your
   specific board revision before attempting it — the recovery filename and
   trigger sequence are bootloader-specific and were not confirmed for V3 in
   any source checked while building this. Guessing wrong here is how
   devices get bricked.
4. After a successful flash: LuCI is at `192.168.1.1`, set a root password
   on first login.

## WireGuard

`luci-app-wireguard` is installed — configure interfaces under
**Network → Interfaces** in LuCI rather than hand-editing UCI.

## Tailscale (post-install)

Not baked into the image. After flashing:

```
opkg update
opkg install tailscale
```

The official ImmortalWrt package feed for `mipsel_24kc` has **not always
carried a `tailscale` package** (removed at times per the
[OpenWrt forum](https://forum.openwrt.org/t/unknown-package-tailscale-no-longer-supported/201496)).
If `opkg install tailscale` fails, check
[lanrat/openwrt-tailscale-repo](https://github.com/lanrat/openwrt-tailscale-repo)
as a third-party prebuilt feed for this architecture.
