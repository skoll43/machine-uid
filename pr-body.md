Hi — this adds Android support following the crate's existing pattern, and engages with the question from #10 (2023) about a JDK-free identifier.

**What is true about Android identifiers (verified, 2026-08-29)**
- There is no `/etc/machine-id` or `/var/lib/dbus/machine-id` (verified: absent on a stock Android device).
- The JDK-free, root-free, world-readable option is the boot id at `/proc/sys/kernel/random/boot_id` (verified on-device). Caveat: it changes on reboot, so it is stable per-boot, not persistent.
- Persistent identifiers exist but are gated: `Settings.Secure.ANDROID_ID` (persistent, but scoped per app-signing-key + user, and only via the Java/Settings APIs — not usable from a Rust crate) and `ro.serialno` (restricted on modern Android).
- So a pure-Rust, JDK-free `machine_id` on Android is necessarily boot-scoped. I also verified there is no persistent + JDK-free + root-free + per-device identifier on modern Android (settings db shell-gated, birth times absent on f2fs, serial restricted, build props per-build). If persistence is required, the Android-idiomatic route is the JDK side (your 2023 instinct; the Tauri ecosystem does exactly this via a Kotlin plugin reading `Settings.Secure.ANDROID_ID`) — happy to document that option instead. (For reference: the Tauri ecosystem's machine-uid plugin ships exactly this — it reads `Settings.Secure.ANDROID_ID` from a Kotlin plugin.)

**Changes**
- `src/lib.rs`: `#[cfg(target_os = "android")]` `machine_id` module reading `/proc/sys/kernel/random/boot_id` (mirrors the linux module's `read_file` pattern; cfg-gated, no impact on other platforms).
- `.github/workflows/rust.yml`: `android-check` job (`cargo check --target aarch64-linux-android`) — the target currently fails with `unresolved import machine_id` (reported by a user in #10, Dec 2023); this job guards it.

**Verification**
- Compile: CI `android-check` green; on-device `cargo check` green.
- Runtime: on-device `machine_uid::get()` returns the boot id (3/3 runs, exit 0); crate's own tests pass (2 passed).

Happy to adjust anything — including dropping this in favor of a JDK-side approach if you prefer persistence semantics.
