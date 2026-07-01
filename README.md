# Linux Kernel Contributions

![Kernel Status](https://img.shields.io/badge/Linux_Kernel-Upstream_Contributor-FCC624?logo=linux&logoColor=black)
![Focus](https://img.shields.io/badge/Focus-hwmon%20%7C%20net%20%7C%20staging-blue)
![C](https://img.shields.io/badge/Language-C-A8B9CC?logo=c&logoColor=white)
![LKMP](https://img.shields.io/badge/LKMP-Spring_2026_Mentee-green)

A central tracking repository for my upstream Linux kernel patches. Currently participating in the **Linux Kernel Mentorship Program (LKMP) Spring 2026** under mentors Shuah Khan and Brigham Campbell.

---

## Patch Log

| Date | Subsystem | Patch | Commit |
| :--- | :--- | :--- | :--- |
| **Jun 2026** | `hwmon` | [hwmon: (ads7871) Use DMA-safe buffer for SPI writes](https://lore.kernel.org/r/20260502020844.110038-4-tabreztalks@gmail.com) | [`b46e1a0`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=b46e1a0bff3343ad8a50a5d9cdb56a3847ba69ef) |
| **Jun 2026** | `hwmon` | [hwmon: (ads7871) Convert to hwmon\_device\_register\_with\_info](https://lore.kernel.org/r/20260502020844.110038-3-tabreztalks@gmail.com) | [`a2b0986`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=a2b0986398e6dd952ab413f6dcd271ddf86ea9b8) |
| **May 2026** | `hwmon` | [hwmon: (ads7871) Fix endianness bug in 16-bit register reads](https://lore.kernel.org/r/20260502020844.110038-2-tabreztalks@gmail.com) | [`99076a1`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=99076a17a112ac43cbd37f6883898ae649166303) |
| **Mar 2026** | `hwmon` | [hwmon: (ads7871) Propagate SPI errors in voltage\_show](https://lore.kernel.org/r/20260308124714.84715-1-tabreztalks@gmail.com) | [`487a9ab`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=487a9ab28fdd4df773b68c953e69a6f6ecc2fe68) |
| **Mar 2026** | `hwmon` | [hwmon: (ads7871) Fix incorrect error code in voltage\_show](https://lore.kernel.org/r/20260307115226.25757-1-tabreztalks@gmail.com) | [`69694e9`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=69694e9622118e546a735affc311ac8e652111d6) |
| **Mar 2026** | `hwmon` | [hwmon: (ads7871) Replace sprintf() with sysfs\_emit()](https://lore.kernel.org/r/20260307083815.12095-1-tabreztalks@gmail.com) | [`4cd4489`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=4cd4489493531fce9046a135c6b99ce1abdb9053) |
| **Feb 2026** | `staging` | [staging: rtl8723bs: fix spacing around operators](https://patch.msgid.link/20260208051341.38631-1-tabreztalks@gmail.com) | [`94c1e3a`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=94c1e3abce312fe89a6a9da7690affc7df8839bf) |
| **Feb 2026** | `net/rds` | [rds: tcp: fix uninit-value in \_\_inet\_bind](https://patch.msgid.link/20260217135350.33641-1-tabreztalks@gmail.com) | [`7b821da`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=7b821da55b3f88c1703ff2c2074d182295a84f6b) |

---

## Engineering Walkthroughs

### 1. hwmon: ads7871 Modernization Series (6 patches)

The ads7871 is a TI SPI ADC with an aging driver. What started as a single error-code fix turned into a complete modernization after hwmon maintainer Guenter Roeck identified deeper structural problems during review. The series went to v6.

**Patch 1 — Replace sprintf() with sysfs\_emit()**
[`4cd4489`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=4cd4489493531fce9046a135c6b99ce1abdb9053)

`sprintf()` is unaware of the 4KB `PAGE_SIZE` limit on sysfs buffers, which can lead to memory corruption. Replaced it with `sysfs_emit()` in the driver's `show()` function. Caught this pattern by reading through `lore.kernel.org/linux-hwmon` history and noticing an active effort to clean it up across the subsystem.

**Patch 2 — Fix incorrect error code in voltage\_show**
[`69694e9`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=69694e9622118e546a735affc311ac8e652111d6)

The driver returned `-1` on ADC timeout, which the kernel maps to `-EPERM` (Operation not permitted). Any userspace monitoring dashboard hitting this would wrongly conclude it needed root access rather than checking the hardware connection. Changed it to `-ETIMEDOUT`. Accepted on v2.

**Patch 3 — Propagate SPI errors in voltage\_show**
[`487a9ab`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=487a9ab28fdd4df773b68c953e69a6f6ecc2fe68)

Guenter pointed out a more serious masking bug: SPI read errors return standard negative values, which set the high bit. The driver was checking that same bit with a bitwise AND against `0x80` to detect "conversion complete." A disconnected SPI bus looked identical to a successful voltage read. This patch propagates the SPI error instead of silently accepting bogus data.

**Patch 4 — Fix endianness bug in 16-bit register reads**
[`99076a1`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=99076a17a112ac43cbd37f6883898ae649166303)

`spi_w8r16()` returns a 16-bit value in CPU-native byte order. Because the ADS7871 transmits LSB first, Big-Endian CPUs were byte-swapping the result, corrupting voltage readings. Fixed by dropping to `spi_write_then_read()`, reading raw bytes into a `u8` buffer, and reconstructing the integer with `get_unaligned_le16()` for architecture-agnostic safety. Flagged by the sashiko bot; went to v6 after additional feedback from Guenter Roeck and David Laight.

**Patch 5 — Convert to hwmon\_device\_register\_with\_info**
[`a2b0986`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=a2b0986398e6dd952ab413f6dcd271ddf86ea9b8)

The driver's outdated registration API had no built-in locking; two userspace programs reading concurrently could collide and corrupt hardware state. Converting to `devm_hwmon_device_register_with_info()` gives automatic locking, declarative `hwmon_channel_info`-based sysfs generation, and `devm`-managed cleanup — all without writing a `remove()` handler. This required understanding the Linux Driver Model properly: the bus core matches driver to device by modalias, and the `with_info` API is the correct entry point into that lifecycle from the hwmon side.

**Patch 6 — Use DMA-safe buffer for SPI writes**
[`b46e1a0`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=b46e1a0bff3343ad8a50a5d9cdb56a3847ba69ef)

The legacy code passed a stack-allocated `u8 tmp[2]` buffer directly to `spi_write()`. On systems with `CONFIG_VMAP_STACK` enabled, hardware DMA cannot safely access stack memory. Moved the transmit buffer into the driver's `struct ads7871_data` (heap-allocated via `devm_kzalloc`) so DMA can reach it safely.

---

### 2. net/rds: fix uninit-value in \_\_inet\_bind

**Commit:** [`7b821da`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=7b821da55b3f88c1703ff2c2074d182295a84f6b)
**Syzbot report:** https://syzkaller.appspot.com/bug?extid=aae646f09192f72a68dc

Syzbot flagged a KMSAN uninit-value access in `__inet_bind()` during RDS TCP socket binding, with over 400 crashes in the upstream testing tree.

**Root cause:** The KMSAN trace gave three stack frames: where it crashed (`__inet_bind`), where the uninit value was stored (`rds_tcp_conn_path_connect`), and where it was created (`rds_tcp_conn_alloc`). `rds_tcp_conn_alloc()` was using `kmem_cache_alloc()`, which leaves memory uninitialized. The field `t_client_port_group` was incremented (`++tc->t_client_port_group`) before ever being assigned a baseline value.

**Fix:** Swapped `kmem_cache_alloc()` for `kmem_cache_zalloc()` in `rds_tcp_conn_alloc()` to zero-initialize the structure on allocation. Sent a patch-testing request to syzbot, which applied the fix, ran its C reproducer, and replied with `Tested-by`.

**Review iteration:**
- **v1:** `Fixes:` tag pointed at the original commit introducing RDS TCP (`70041088e3b9`). Commit message also exceeded the 75-character line limit.
- **v2:** Reviewers Charalampos Mitrodimas and Allison Henderson corrected the `Fixes:` tag to `a20a6992558f` (the commit that actually introduced `t_client_port_group`), and I rewrote the commit message to document the specific field and wrap lines correctly. Paolo Abeni applied the patch to `netdev/net.git` with one additional note: no empty lines between trailers.

---

### 3. staging: rtl8723bs: fix spacing around operators

**Commit:** [`94c1e3a`](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=94c1e3abce312fe89a6a9da7690affc7df8839bf)

A `checkpatch.pl`-flagged formatting fix in the rtl8723bs staging driver. Merged upstream. Useful early on for learning the patch submission workflow end-to-end (checkpatch → get_maintainer.pl → git send-email → mailing list).

---

## Tools Used

| Tool | Purpose |
| :--- | :--- |
| `scripts/checkpatch.pl` | Style and formatting validation before every send |
| `scripts/get_maintainer.pl` | Finding correct maintainers and mailing lists to Cc |
| `cscope` | Navigating the hwmon subsystem and tracing API usage across drivers |
| `syzbot / syzkaller` | Fuzzing reports and reproducer-replay for fix verification |
| `QEMU + gdb` | Local kernel debugging (KMSAN build; hit memory limits at 24GB RAM) |
| `Vim + git` | Editing and version control |
