# Linux Kernel Contributions

![Kernel Status](https://img.shields.io/badge/Linux_Kernel-Upstream_Contributor-FCC624?logo=linux&logoColor=black)
![Focus](https://img.shields.io/badge/Focus-hwmon%20%7C%20net-blue)
![C](https://img.shields.io/badge/Language-C-A8B9CC?logo=c&logoColor=white)

A central tracking repository for my upstream Linux kernel patches. My work primarily focuses on modernizing legacy drivers, fixing hardware-level bugs (like endianness and DMA safety), and resolving memory safety issues caught by sanitizers.

## 📜 Patch Log

| Date | Subsystem | Patch / Series | Status |
| :--- | :--- | :--- | :--- |
| **Apr 2026** | `hwmon` | [hwmon: (ads7871) Modernize and fix DMA safety](#) | Under Review (v3) |
| **Early 2026** | `net/rds` | [rds: tcp: fix uninit-value in __inet_bind](https://git.kernel.org/pub/scm/linux/kernel/git/netdev/net.git/commit/?id=7b821da55b3f) | Accepted |

---

## 🛠️ The Engineering Process: Patch Walkthroughs

Below is a technical breakdown of how I approached, debugged, and authored these patches.

### 1. hwmon: ads7871 Modernization and Endianness Fix

**Context:** The `ads7871` driver was using legacy sysfs macro boilerplate, lacked DMA-safe SPI buffers, and harbored a hidden bug affecting Big-Endian CPU architectures.

* **Patch 1 (The Endianness Fix):** Discovered that the driver was using `spi_w8r16()` to read 16-bit sensor data. Because the ADS7871 transmits the Least Significant Byte (LSB) first, Big-Endian CPUs were byte-swapping the result natively, corrupting the voltage readings. I fixed this by dropping down to `spi_write_then_read()`, grabbing the raw bytes into a `u8` array, and explicitly reconstructing the integer using the `get_unaligned_le16()` macro to guarantee architecture-agnostic safety.
* **Patch 2 (API Migration):** Ripped out the old, heavily-macroed `hwmon_device_register()` boilerplate and migrated the driver to the modern `hwmon_device_register_with_info()` API. This allowed me to declare the sensor's capabilities using clean, declarative `hwmon_channel_info` arrays.
* **Patch 3 (DMA Safety):** The legacy code passed stack-allocated buffers to `spi_write()`. On modern systems with `CONFIG_VMAP_STACK` enabled, hardware DMA cannot safely access stack memory. I resolved this by moving the transmit buffer (`tx_buf[2]`) into the driver's dynamically allocated private data structure (`struct ads7871_data`) and ensuring it was `____cacheline_aligned`.
### 2. net/rds: tcp: fix uninit-value in __inet_bind (Commit `7b821da55b3f`)
syzkall report: https://syzkaller.appspot.com/bug?extid=aae646f09192f72a68dc
**Context:** Syzbot automated fuzzing caught an uninitialized memory access bug flagged by the Kernel Memory Sanitizer (KMSAN) during an RDS TCP socket binding. The bug had caused over 400 crashes in the upstream testing tree.

* **Triage & Debugging Workflow:** * **Analyzing the KMSAN Report:** I examined the Syzkaller crash log, which provided three critical stack traces: 
    1. *Where it crashed:* `__inet_bind`
    2. *Where the uninit value was stored:* `rds_tcp_conn_path_connect`
    3. *Where the uninit value was created:* `rds_tcp_conn_alloc` (via `kmem_cache_alloc`).
  * **Root Cause Analysis:** I traced the allocation up the call stack. The `rds_tcp_connection` structure was being allocated using `kmem_cache_alloc()`, which leaves memory uninitialized. I found that in `rds_tcp_conn_path_connect()`, the specific field `t_client_port_group` was being incremented (`++tc->t_client_port_group`) before ever being assigned a baseline value.
* **The Solution:** Swapped the allocation to `kmem_cache_zalloc()` in `rds_tcp_conn_alloc()` to ensure the entire structure is zero-initialized upon creation, preventing garbage data from reaching the socket binding APIs. I then sent a patch testing request to Syzbot to verify the fix against its C reproducers.
* **The Review Process & Iteration:** * **v1:** I submitted the initial fix pointing the `Fixes:` tag to the original commit that introduced RDS TCP (`70041088e3b9`).
  * **Community Feedback:** Reviewer Charalampos Mitrodimas and Maintainer Allison Henderson pointed out that while the allocator was old, the specific field causing the bug (`t_client_port_group`) was actually introduced much later in commit `a20a6992558f`. They also noted my commit message exceeded the strict 75-character limit.
  * **v2:** I completely rewrote the commit message to explicitly document the `t_client_port_group` field, corrected the `Fixes:` tag to accurately reflect the Git history, and wrapped the text to 75 characters. 
* **Outcome:** Syzbot successfully tested the fix (`Tested-by: syzbot...`). The `v2` patch received `Reviewed-by` tags from Allison Henderson and Charalampos Mitrodimas, and was merged into the main `netdev/net.git` tree by Paolo Abeni. (I also learned a quick formatting lesson from Paolo: never leave empty lines between the Signed-off-by and Reviewed-by tags!).
