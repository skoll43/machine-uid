Hi — this adds Android support following the crate's existing pattern, and answers the persistence question from #10 (2023).

**The identifier: `gsm.serial` (persistent), fallback `boot_id` (per-boot)**
- Android's property namespace is readable JDK-free, root-free, and without subprocesses via `__system_property_get` (bionic libc export). On this device `gsm.serial` is present, stable across reads, and returns a per-device persistent serial (`6D329Q404946`).
- The `/sys` serial sources (`soc0/serial_number`, eMMC CID, `wlan0/address`) are permission-restricted on modern Android; `ANDROID_ID` and `ro.serialno` are gated; file birth times are absent on f2fs. The property namespace is the accessible path.
- Fallback: if `gsm.serial` is absent (e.g. WiFi-only devices, some RILs), we fall back to `/proc/sys/kernel/random/boot_id` (per-boot, world-readable).

**Changes**
- `src/lib.rs`: `#[cfg(target_os = "android")]` `machine_id` module (gsm.serial → boot_id fallback; mirrors the linux module's structure; cfg-gated, no impact on other platforms).
- `.github/workflows/rust.yml`: `android-check` job (`cargo check --target aarch64-linux-android`) — the target currently fails with `unresolved import machine_id` (reported in #10, Dec 2023); this job guards it.

**Verification**
- Compile: CI `android-check` green; on-device `cargo check` green.
- Runtime (on-device, Termux aarch64): `machine_uid::get()` returns the serial `6D329Q404946` (3/3 runs, exit 0); crate's own tests pass.

**Honest caveats**
- `gsm.serial` availability varies by device/RIL (hence the fallback); it is the telephony serial — persistent per device, but absent on some hardware.
- Serial-like identifiers carry privacy considerations on Android; happy to discuss if you'd prefer a different primary.
- cfg-gated: no impact on linux/macos/windows/illumos.

Happy to adjust anything.
