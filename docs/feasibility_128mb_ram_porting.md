# Feasibility Analysis: Porting OpenWrt + PicoClaw to 128MB RAM Board

**Date:** 2026-04-02
**Context:** The current AXU15EG has 4GB PS DDR4. A user asked whether this system can be ported to a board with only 128MB SDRAM.

## Current Memory Usage (AXU15EG, 4GB)

Measured via PicoClaw Discord system report after 5 minutes uptime:

| Metric | Value |
|--------|-------|
| Total Memory | 3.84GB (4,024,540 KB) |
| Used | 59.6MB (61,068 KB) |
| Available | 3.72GB (3,905,256 KB) |
| Usage | ~1.5% |
| Swap | None configured |

## Component-Level RAM Breakdown

| Component | Estimated RAM | Notes |
|-----------|--------------|-------|
| Linux Kernel (6.6.40-xilinx) | ~15-20MB | ZynqMP platform drivers, GPIO, USB, Ethernet, WiFi (cfg80211) |
| OpenWrt userspace | ~10-15MB | procd, dnsmasq, dropbear, netifd, ubus — very lean |
| LuCI (web UI) | ~5-10MB | uhttpd + Lua/ucode interpreter + templates |
| PicoClaw (Go static binary) | ~30-80MB | Go GC reserves large virtual heap; RSS grows during LLM API calls |
| **Total (typical)** | **~60-125MB** | Varies with workload |

### PicoClaw Memory Detail

- **Binary size on disk:** 22MB (static Go binary, CGO_ENABLED=0)
- **Virtual memory (VSZ):** ~1252MB — Go runtime reserves large address space (normal, not actual usage)
- **Resident memory (RSS):** ~30-50MB at idle, spikes to ~80MB+ during LLM API response processing (JSON parsing, string buffers, HTTP response body)
- **Go GC behavior:** Aggressively allocates then collects; memory usage is bursty

## Feasibility Assessment

### Verdict: Very tight, but possible with compromises

With 128MB total SDRAM:
- Kernel reserves ~15-20MB at boot
- Usable memory for userspace: ~100-110MB
- OpenWrt core (no LuCI): ~10-15MB
- Remaining for PicoClaw: ~85-95MB
- PicoClaw typical idle RSS: ~30-50MB → **fits at idle**
- PicoClaw peak during API calls: ~80MB+ → **may trigger OOM killer**

### What Makes It Challenging

1. **Go runtime memory overhead** — Go's garbage collector and runtime take ~10-15MB before any application code runs
2. **Bursty allocation** — LLM API responses can be large JSON payloads; parsing temporarily doubles memory usage
3. **No swap safety net** — SD card swap is possible but introduces latency and wear
4. **Kernel size** — ZynqMP requires many platform-specific drivers that can't easily be trimmed

### What Makes It Feasible

1. **OpenWrt is extremely lean** — the entire userspace (procd, networking, firewall) fits in ~15MB
2. **Static binaries have no libc overhead** — PicoClaw and led_ctrl don't need shared libraries
3. **Go supports memory limits** — `GOMEMLIMIT` environment variable caps heap allocation
4. **LuCI is optional** — CLI-only operation saves ~10MB

## Recommended Optimization Steps

### Priority 1: Drop LuCI (saves ~10MB)

Remove LuCI from the OpenWrt rootfs build:
```bash
# In build_openwrt_rootfs.sh, remove from PACKAGES:
#   luci luci-ssl
```
Use UCI commands for configuration instead of web UI.

### Priority 2: Cap Go Memory (prevents OOM)

Set Go memory limit in the PicoClaw procd service:
```sh
# In /etc/init.d/picoclaw
procd_set_param env HOME="$PICOCLAW_HOME" GOMEMLIMIT=40MiB
```
This forces Go's GC to collect more aggressively, trading CPU for memory.

### Priority 3: Trim Kernel Config

Disable unused drivers in PetaLinux kernel config:
- Xen support (`CONFIG_XEN=n`)
- CAN bus (`CONFIG_CAN=n`)
- USB gadget (`CONFIG_USB_GADGET=n`)
- Media/V4L2 (`CONFIG_MEDIA_SUPPORT=n`)
- Virtualization (`CONFIG_VIRTUALIZATION=n`)

Estimated savings: 3-5MB kernel memory.

### Priority 4: Add Swap (emergency headroom)

```bash
dd if=/dev/zero of=/swap bs=1M count=64
mkswap /swap
swapon /swap
echo "/swap swap swap defaults 0 0" >> /etc/fstab
```
Provides 64MB emergency headroom. Avoid heavy use — SD card wear and latency.

### Priority 5: Rewrite PicoClaw in C or Rust (maximum savings)

A C/Rust rewrite of the PicoClaw gateway would reduce memory from ~50MB to ~5MB:
- HTTP client for LLM API: ~1MB
- Discord WebSocket client: ~1MB
- JSON parsing: ~1MB with streaming parser
- Application logic: ~2MB

**Effort:** 2-4 weeks. Only justified if shipping a product at volume on 128MB boards.

## Alternative Architectures for 128MB

| Approach | RAM Usage | Effort | Notes |
|----------|-----------|--------|-------|
| OpenWrt + PicoClaw (Go) + GOMEMLIMIT | ~80-100MB | Low | Current stack, risk of OOM under load |
| OpenWrt + PicoClaw (Go, no LuCI) + GOMEMLIMIT | ~70-90MB | Low | Best quick option |
| OpenWrt + lightweight LLM proxy (C) | ~30-40MB | Medium | Rewrite PicoClaw as minimal C HTTP proxy |
| OpenWrt only (no LLM, CLI led_ctrl) | ~25-35MB | None | Plenty of room, no cloud AI features |
| Minimal Linux (buildroot, no OpenWrt) | ~15-25MB | High | Maximum savings, lose opkg/UCI/LuCI ecosystem |

## Conclusion

A 128MB board can run the OpenWrt hybrid stack if:
1. LuCI is removed
2. PicoClaw runs with `GOMEMLIMIT=40MiB`
3. Small swap partition exists as safety net

The system will function but with minimal headroom (~10-20MB free). For a robust production deployment on 128MB, rewriting PicoClaw in C/Rust is the proper long-term solution.

If PicoClaw is not needed (board used only for OpenWrt + GPIO LED control), 128MB is more than sufficient with ~90MB free.
