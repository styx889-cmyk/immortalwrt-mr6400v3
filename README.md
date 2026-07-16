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

**Update: confirmed working on real V3 (EU) hardware** — booted successfully,
LAN/WiFi/LuCI all work. But also see the flash-space problem below, found on
this same unit — the fact that it boots is not the same as it being reliable
long-term.

## Flash is extremely tight on this device — read this before flashing

The "firmware" MTD partition on this profile is a fixed **7808k**. It's split
at build time into kernel + squashfs rootfs, and whatever's left over
(rounded down to erase-block size) becomes the writable `rootfs_data`
(overlay) partition — the one that stores your root password, WiFi config,
literally everything you change.

A build with just `luci` on top of this profile's defaults (stripped
USB/LTE/PPP, nothing else added) already used up all but ~256KB of that
budget. 256KB is 4 erase blocks, and JFFS2 refuses to mount on that few:

```
kernel: Too few erase blocks (4)
mount_root: failed - mount -t jffs2 /dev/mtdblock4 /rom/overlay
mount_root: jffs2 not ready yet, using temporary tmpfs overlay
```

The router then silently falls back to a **RAM-only tmpfs overlay** — it
boots fine and LuCI looks completely normal, but *nothing you configure
survives a reboot*: root password, WiFi settings, everything reverts to
firmware defaults every time. This existed from the very first successful
build, before tailscale or anything else was added — it's a structural
property of this profile at this flash size, not something a specific
package broke.

The `PACKAGES` list now trims further specifically to claw back headroom
(see comments in `build.yml` for the full reasoning):
`dnsmasq-full` → `dnsmasq`, and drops `odhcp6c`/`odhcpd-ipv6only` (IPv6
DHCP client/server userspace bits — the kernel still has IPv6 itself, that's
not a removable module). Swapping the TLS backend from OpenSSL to mbedTLS was
also tried, on the (usually reasonable) assumption that mbedTLS is smaller —
measured *worse* here (720611 bytes free vs. 917219 with OpenSSL), so that
change was reverted. Don't assume that optimization helps without measuring
it on this specific build.

The build has a CI check (`Check overlay headroom`) that fails the workflow
if the image would leave less than 512KB (8 erase blocks) free for the
overlay. JFFS2 measurably refused to mount at 4 erase blocks (256KB) on real
hardware; 8 is a commonly-cited practical minimum for it to actually work
long-term rather than just barely mount, so that's the bar rather than a more
arbitrary round number.

**After flashing any build from this repo, verify persistence before trusting
it**: SSH in, run `mount | grep overlay` — if the source is
`overlayfs:/tmp/root`, you're on the broken tmpfs fallback, not real flash.

## Building

Actions tab → **Build ImmortalWrt firmware (TL-MR6400 v4 profile)** →
*Run workflow*. Optionally override the ImmortalWrt version (defaults to
`25.12.1`). Output `.bin` files are attached as a workflow artifact.

Package set: `luci`, `dnsmasq`, plus LAN and WiFi (already in the target's
defaults). Stripped: USB, LTE/QMI modem support, PPP, IPv6 DHCP client/server,
`dnsmasq-full`'s extra features — see the flash-space section above for why.
No Tailscale, no dedicated WireGuard packages — see below.

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
4. After a successful flash: LuCI is at `192.168.1.1`. **No root password is
   set by default** — LuCI will nag with a banner ("未设置密码!") until you
   set one under 系统 (System) → 管理权 (Administration). Do this before
   exposing the router to anything untrusted; until then anyone on the LAN
   has unauthenticated root access via LuCI and SSH.

## VPN: Tailscale only, no standalone WireGuard

ImmortalWrt does **not build the WireGuard kernel module (`kmod-wireguard`)
for ramips/mt76x8 at all** — confirmed absent from the package feed across
23.05, 24.10, 25.12, and snapshot. `wireguard-tools` and
`luci-proto-wireguard` exist as packages, but installing them alone would
give you a LuCI page that can never actually establish a tunnel, since the
kernel-side implementation is missing. Getting real kernel WireGuard here
would require a full source build with a patched kernel config, not
ImageBuilder — out of scope for this repo.

Instead, VPN needs are covered by **Tailscale**, which bundles its own
userspace WireGuard protocol implementation (`wireguard-go`) over a TUN
device and doesn't need the kernel module.

## Tailscale — not in this image, install it after confirming persistence works

An earlier version of this repo baked `tailscale` + `luci-app-tailscale-community`
straight into the firmware. That's on hold: baking it in made the flash-space
problem above worse, not better (tailscale's binary is a few MB even
compressed, on a device with barely any margin to begin with), and there was
no point trying to fit it in before the base image could even hold a
persistent config.

The prebuilt `tailscale` .apk still isn't published for `mipsel_24kc` on any
OpenWrt/ImmortalWrt feed, official or third-party — the one third-party feed
that exists, [lanrat/openwrt-tailscale-repo](https://github.com/lanrat/openwrt-tailscale-repo),
only publishes old-format `opkg` `.ipk` packages, incompatible with this
image's `apk` package manager. So installing it later still means
cross-compiling it (via the ImmortalWrt SDK, targeting this same
release/arch) and side-loading the resulting `.apk` through LuCI's
System → Software → "Upload Package", rather than a plain `apk add tailscale`.
Once the base image's overlay is confirmed persistent (see above), that's the
next thing to sort out — not done yet.
