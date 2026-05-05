# TIQ M5 — MAC patching, live-device findings

A companion to [`tiq_m5_offline_analysis.md`](tiq_m5_offline_analysis.md), this doc records what was observed running this repo's BT and WiFi MAC patching flow against a connected **TIQ M5** (MT6761, dual-SIM, Android 13 + Magisk). Every output reproduced below is from a real run; nothing is illustrative.

> **Headline.**
>
> - **BT MAC patching works end-to-end via `live_patch_mac.sh`.** Verified live across multiple reboot cycles: `BT_Addr` file persists byte-identical, the MTK NVRAM daemon validates the trailer, and `settings get secure bluetooth_address` returns the patched MAC byte-for-byte every time. Same behavior as on F21 Pro and F25.
> - **WiFi MAC patching writes the file correctly but does not update the runtime MAC.** The kernel WiFi driver loads the patched MAC into the wiphy at boot (visible in `dmesg`), but the chipset firmware's on-die cache — which Android's WiFi HAL queries for the "factory MAC" — keeps serving whatever was last written through the Java app port [`flipphoneguy/mtk-imei-switcheroo-app`](https://github.com/flipphoneguy/mtk-imei-switcheroo-app), regardless of file content, partition flashes, or reboots. The app port uses the same algorithm and the same on-device primitives as `live_patch_mac.sh`; we could not identify what its `cp` does differently from the script's `cp` such that it reaches the chipset's cache. Working hypothesis is in [§ Working hypothesis](#working-hypothesis) below.
>
> F21 Pro and F25 do not exhibit this asymmetry — `live_patch_mac.sh` updates WiFi runtime there as expected.

## Status

| What | Tool | M5 result |
|---|---|---|
| BT MAC read | `mac_tool.py read BT_Addr` | works |
| BT MAC write (live) | `live_patch_mac.sh` | **works end-to-end — verified** |
| BT MAC write (offline + flash) | `mac_tool.py write nvdata.img` + `fastboot flash nvdata` | works (BT side; the WiFi runtime quirk is independent of flash path) |
| WiFi MAC read | `mac_tool.py read WIFI` | works |
| WiFi MAC **file** write (live) | `live_patch_mac.sh` | works — file persists byte-identical, daemon's `NVM_CheckFile` accepts it |
| WiFi MAC **file** write (offline + flash) | `mac_tool.py write nvdata.img` + `fastboot flash nvdata` | works — partition block holds the patched MAC in every signature-matching copy (verified by post-flash `dd`) |
| WiFi MAC **runtime** propagation | `live_patch_mac.sh` or `fastboot flash nvdata` | **does not update runtime MAC** — see [§ The WiFi runtime quirk](#the-wifi-runtime-quirk) |

## BT — verified working

```
$ ./live_patch_mac.sh
  Current BT_Addr : 4e:3b:46:90:34:7c
  Current WIFI MAC: ...
Change BT MAC? [y/N] y
  New BT MAC: 02:11:22:33:44:55
  Patching BT_Addr...
Wrote tmp/patched_BT_Addr.bin: BT=02:11:22:33:44:55
BT_Addr: 02:11:22:33:44:55
... [reboot prompt → y → device reboots]

$ adb shell su -c "cp '/mnt/vendor/nvdata/APCFG/APRDEB/BT_Addr' '/sdcard/post' && chmod 644 '/sdcard/post'"
$ adb pull /sdcard/post tmp/post_BT_Addr.bin
$ md5sum tmp/patched_BT_Addr.bin tmp/post_BT_Addr.bin
3c97dddf3d330f6408dda8736b12f326  tmp/patched_BT_Addr.bin
3c97dddf3d330f6408dda8736b12f326  tmp/post_BT_Addr.bin

$ adb shell svc bluetooth enable
$ adb shell settings get secure bluetooth_address
02:11:22:33:44:55
```

File persists byte-identical across reboot, daemon's trailer-checksum validates, and the runtime BT MAC reflects the patched value. Re-run with any locally-administered BT MAC; the result is consistent across reboots.

## The WiFi runtime quirk

### Layer-by-layer: where the chain breaks

Patching with `live_patch_mac.sh` (or with `mac_tool.py write nvdata.img --wifi <X>` + `fastboot flash nvdata`) propagates the new MAC through every layer we can inspect — except the chipset's internal cache, which is what Android's HAL ultimately reports.

| Layer | What `live_patch_mac.sh` produces on M5 | Reflects new MAC? |
|---|---|---|
| `/mnt/vendor/nvdata/APCFG/APRDEB/WIFI` (filesystem view) | new MAC, valid trailer | yes |
| Magisk mirror namespace (`su -mm`) view of the same file | new MAC | yes |
| Partition block `/dev/block/by-name/nvdata` (`dd` post-sync) | new MAC in every signature-matching copy | yes |
| `nvdata` partition image post-`fastboot flash <patched>` | new MAC | yes |
| Daemon's `NVM_CheckFile` validation across reboot | accepts (no `NVM_RestoreFromBinRegion` event in dmesg) | yes (no rollback) |
| Kernel `kalUpdateMACAddress` on next boot | new MAC loaded into wiphy | yes |
| Driver's `wlanOnPreNetRegister` | new MAC | yes |
| HAL `wifi_get_factory_mac_address` reply | **stale chipset value** | **no** |
| HAL → driver `wlanSetMacAddress` for connect | stale chipset value forced onto wlan0 | overrides |
| `/sys/class/net/wlan0/address` | stale chipset value | **no** |
| ARP TX SRC MAC (kernel TX) and `dumpsys wifi` `mWifiInfo MAC` | stale chipset value | **no** |
| The MAC the AP / router sees | stale chipset value | **no** |

### Side-by-side: F21 Pro vs TIQ M5, same script, same time

After a clean `live_patch_mac.sh` run that wrote `02:22:44:66:88:aa` to F21 Pro and `02:11:33:55:77:99` to TIQ M5 (both with `MacRandomizationSetting=0` on the connected SSID):

```
F21 Pro:
  APRDEB/WIFI on disk:           02:22:44:66:88:aa
  /sys/class/net/wlan0/address:  02:22:44:66:88:aa
  ARP TX SRC MAC (kernel):       02:22:44:66:88:aa
  Connecting with (logcat):      02:22:44:66:88:aa
  → file == kernel == HAL == network

TIQ M5:
  APRDEB/WIFI on disk:           02:11:33:55:77:99    ← script's patch
  /sys/class/net/wlan0/address:  02:11:33:84:7c:0c    ← last app-patched value
  ARP TX SRC MAC (kernel):       02:11:**:**:**:0c    ← network sees this
  Connecting with (logcat):      02:11:33:84:7c:0c
  → file is the script's patch; kernel/HAL/network is whatever the app last wrote
```

### dmesg sequence on M5

The kernel driver does load the file's MAC into the wiphy at boot. Android's HAL then overrides it:

```
[T+34] kalUpdateMACAddress:(INIT INFO) <addr>:02:**:**:**:99      ← driver loaded patched MAC
[T+34] wlanOnPreNetRegister: MAC address: 02:**:**:**:99          ← wiphy has patched MAC
[T+91] wlanSetMacAddress: Set connect random macaddr to 02:**:**:**:0c.
                                                                    ↑↑↑ HAL forces stale chipset value
[T+91] nicActivateNetworkEx: OwnMac1=02:**:**:**:0c               ← chipset uses stale value for connect
```

The `wlanNvramUpdateOnTestMode:(INIT ERROR) wlanNvramUpdateOnTestMode invalid!!` line that appears at every M5 boot also appears on F21 Pro at every boot — it is the runtime push from `wlan_assistant` failing on both, and is **not** the differentiator. What differs is what the chipset firmware reports as the "factory MAC" when the HAL queries it via NL80211 vendor command. F21 Pro's chipset firmware reports the wiphy MAC (the file's value); M5's chipset firmware reports an internal cached value.

### Where the cached value lives

The cached MAC is **not** on any block partition or filesystem path we can read. Searched exhaustively for the cached MAC bytes in:

```
/dev/block/by-name/{nvdata, nvram, nvcfg, protect1, protect2, sec1, seccfg,
                    expdb, proinfo, flashinfo, mmcblk0boot0, mmcblk0boot1, otp}
/vendor /system /system_ext /odm /apex
/data/vendor /data/misc /data/system /data/local /data/data /data/app
/mnt/vendor/nvdata /mnt/vendor/nvcfg
```

No hits anywhere. The cached value survives `adb reboot`, `su -c reboot`, `fastboot reboot`, **and** `fastboot flash nvdata <patched.img>`. After flashing a freshly-patched `nvdata.img` (every signature-matching WIFI copy holds the new MAC, verified via post-flash `dd` of the partition block), the runtime MAC is still the previous app-port-patched value. The only thing that changes the cached value is a write done through the app port.

The only place left where the cached value can live is the MTK MT6635-class chipset's **on-die memory** — the connsys subsystem has its own SRAM/RAM that the SoC firmware loads firmware into at boot, and that does not appear as a block device on the SoC side. Whatever interaction the app port performs, it triggers the chipset firmware to update that on-die cache; whatever `live_patch_mac.sh` performs from the host shell does not.

### What was attempted and ruled out

| Hypothesis | Test | Result |
|---|---|---|
| Script wrote a structurally-bad file the daemon reverted | `cmp -l` post-reboot file vs `mac_tool.py write` output, full structural validation (size, header, MAC, trailer magic, computed checksum) | byte-identical to expected, daemon does not roll back |
| Mount namespace mismatch (Magisk per-call NS vs mount-master) | `su -mm -c stat`, `cat /proc/self/mountinfo`, `dd` partition block, compare to fs view | shell-su and mount-master-su see the same inode, same content, same mount; no NS difference |
| SELinux context / UID / source-path origin | Stage the patched file at `/data/data/com.flipphoneguy.imeiswitcher/cache/patched_WIFI.bin` with the app's UID `u0_a154` and SELinux `app_data_file:s0:c154,c256,c512,c768`, then run identical primitives | runtime unchanged |
| Reboot mechanism | `adb reboot` vs `su -c reboot` vs `fastboot reboot` | all three produce the same outcome |
| Time delay between `cp` and reboot (wlan_assistant inotify race) | sleep 100ms, 5s, 15s | runtime unchanged |
| Sequencing — many separate `su -c` calls vs one `su -c script.sh` | Push a single script that runs mount → cp → chmod → chown in one `su` invocation | runtime unchanged |
| WiFi-on vs WiFi-off at the moment of `cp` | `svc wifi disable` before patching, then patch + reboot | runtime unchanged |
| Chipset state was sticky after first boot | Multiple consecutive reboots without re-patching | runtime keeps reporting whatever the app last wrote, regardless of file content |
| Offline patch + `fastboot flash nvdata` | `mac_tool.py write nvdata.img --wifi <X>` → `fastboot flash nvdata` → reboot; verified the partition block holds X via post-flash `dd` | partition holds X, runtime still holds previous app-patched value |
| Magisk grants different policy per uid | `sqlite3 /data/adb/magisk.db "SELECT * FROM policies"` | shell uid 2000 and the app uid 10154 have **identical** rows: `policy=2, until=0, logging=1, notification=1` |
| Chipset firmware blob differs and is the actual MAC source | Pulled `WIFI_RAM_CODE_soc1_0_1_1.bin` from F21 vs M5; bind-mounted F21's blob over M5's via Magisk | both blobs are encrypted (different bytes 0x20-0x3F, identical magic 0x00-0x1F); SELinux blocks the kernel from opening the bind-mounted file (`tcontext=u:object_r:unlabeled` after `chcon`) — couldn't conclusively test |
| Chipset needs META mode for NV update | dmesg shows `wlanNvramUpdateOnTestMode invalid!!` at every boot on both devices | F21 Pro logs the same line and works; M5 logs it and doesn't. Not the differentiator |

### Working hypothesis

After ruling out the above, the remaining observable difference between the script's invocation of `su` and the app port's invocation of `su` is the **calling-process domain**:

- App's `su` is forked from the Dalvik VM process — calling uid `u0_a154`, calling SELinux context `untrusted_app:s0:c154,c256,c512,c768`.
- Script's `su` is forked by `adbd → sh` — calling uid `2000`, calling SELinux context `shell:s0`.

Magisk's policy DB is identical for both, but the resulting `su`'s parent-process ancestry, capability inheritance, or SELinux transition path may differ in ways the user-visible config doesn't capture. Whatever Magisk does differently for app-forked `su` vs shell-forked `su` on this M5 build, combined with whatever the chipset firmware accepts as a trigger to update its on-die cache, produces the asymmetry. Verifying this requires kernel-level instrumentation (`strace -f` of both flows side by side, or ftrace of `do_open` / `do_exec` across the patch). We could not get `strace` to run on the device because Termux's `strace` deb has a `libdw → libandroid-support` transitive dependency that does not link against the system linker, and `simpleperf` requires a tracee PID before the syscall happens — we could not bracket the full app flow without instrumenting from inside the app's process.

This is **not a bug in `live_patch_mac.sh`**. The script writes byte-perfect files to the same destination, with the same SELinux context, ownership, and primitive sequence the app uses. The chipset firmware on this M5 build accepts updates to its on-die cache only through the path the Java app exercises.

## Recommended workflow on TIQ M5

| Want to change | Use |
|---|---|
| Bluetooth MAC | `live_patch_mac.sh` from this repo |
| WiFi MAC at the file level (for inspection / forensics) | `live_patch_mac.sh` works fine |
| WiFi MAC at the runtime level (what the AP / router actually sees) | The Java app port [`flipphoneguy/mtk-imei-switcheroo-app`](https://github.com/flipphoneguy/mtk-imei-switcheroo-app), installed as an APK and run on-device |

The app's source code states it is a Java wrapper around this repo's logic — its `MacCrypto.java` is a Java port of `mac_tool.compute_checksum`, and its `RootRunner.replaceFile` runs the same `mount -o remount,rw → cp → chmod 660 → chown root:system` sequence. On F21 Pro and F25 either path works. On TIQ M5 only the app's path updates the chipset's on-die cache; the script's identical-from-the-outside path does not.

## How to reproduce

1. Connect a TIQ M5 to a host with `adb` and root granted to `shell` via Magisk.
2. Run the live patch with a fresh test MAC:
   ```
   cd ~/mtk-imei-switcheroo
   ./live_patch_mac.sh
   # Change BT MAC? [y/N] n
   # Change WiFi MAC? [y/N] y
   # New WiFi MAC: 02:11:33:55:77:99
   # Reboot device now? [y/N] y
   ```
3. After boot, verify the file:
   ```
   adb shell su -c 'dd if=/mnt/vendor/nvdata/APCFG/APRDEB/WIFI bs=1 skip=4 count=6 status=none' | xxd
   # → 02 11 33 55 77 99
   ```
4. Verify the partition block (independent of the FS cache):
   ```
   adb shell su -c 'dd if=/dev/block/by-name/nvdata of=/sdcard/nvdata_post.img bs=1M'
   adb pull /sdcard/nvdata_post.img tmp/
   python3 mac_tool.py read tmp/nvdata_post.img
   # → WIFI MAC (N copies, first @ ...): 02:11:33:55:77:99
   ```
5. Verify the runtime MAC (what the AP sees):
   ```
   adb shell svc wifi enable
   sleep 20
   adb shell cmd wifi start-scan
   sleep 12
   adb shell su -c 'cat /sys/class/net/wlan0/address'
   adb shell 'dumpsys wifi 2>/dev/null | grep -E "mWifiInfo MAC" | head -1'
   adb shell su -c 'dmesg 2>/dev/null | grep "ARP REQ SRC MAC" | tail -1'
   ```
   On TIQ M5, the runtime MAC is whatever the **app port** last wrote, **not** `02:11:33:55:77:99`. On F21 Pro running the same flow, the runtime MAC is `02:11:33:55:77:99`.

Then install the Java app port APK on the TIQ M5, open it, type a new WiFi MAC, tap **Apply WiFi**, tap **Reboot**. After boot, repeat step 5. The runtime MAC is now the value the app wrote.

## Hardware context relevant to this finding

- **WiFi/BT chipset**: MT6635-class connac1x (driver `wlan_drv_gen4m`, BT driver `bt_drv_connac1x`). The chipset's eFuse-default MAC observed on this device is `00:08:22:04:81:fd` (OUI `00:08:22` is registered to InProComm, now MediaTek). This default reappears at runtime whenever the chipset's on-die cache has not yet been programmed by the app port.
- **SoC**: MT6761 — confirmed via consys chipid `0x6761` and the `md1img_a` banner `LR12A.R3.MP MT6761 / MT6761_S00`.
- **Magisk**: present; policy DB at `/data/adb/magisk.db`. Both shell uid 2000 and the app uid 10154 have `policy=2 (allow)` with identical `logging`, `notification`, and `until` fields.
- **Android version**: 13. Both `adb exec-out` CRLF injection and `chown` on app-cache files via `su` exhibit the Android 13 + Magisk quirks documented elsewhere in this repo; they are unrelated to the MAC runtime issue.

For the rest of the partition layout, the AllMap/AllFile structure, the per-record offsets, and the per-device byte comparison, see [`tiq_m5_offline_analysis.md`](tiq_m5_offline_analysis.md). For the WiFi/BT NVRAM record format and the trailer-checksum algorithm, see [`wifi_bt_reverse_engineering.md`](wifi_bt_reverse_engineering.md).
