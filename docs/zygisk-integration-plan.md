# Zygisk Integration Plan

Status: **Design / proposal** — no code changes to the build yet. This document
lays out how we intend to get Zygisk (and therefore Riru-free LSPosed, Shamiko,
etc.) working on Waydroid, why it does not work today, and the concrete steps to
try, ordered from most to least promising.

## 1. Why Zygisk does not work today

The current build (`.github/workflows/magisk.yml`, step **Integrate Magisk**)
uses the MagiskOnWSA approach:

1. Magisk binaries (`magisk64`, `magisk32`, `magiskinit`, `magiskpolicy`,
   `busybox`, `stub.apk`, `loadpolicy.sh`) are copied into
   `/debug_ramdisk` on the system image.
2. `init.rc` is patched so that **`on post-fs-data`** it mounts a tmpfs at
   `/debug_ramdisk`, copies the binaries in, runs `loadpolicy.sh` to live-patch
   sepolicy, and then runs `magisk --post-fs-data`.
3. `init.zygoteNN.rc` gets an injected
   `exec ... magisk --zygote-restart` line right after `service zygote`.

The fundamental timing problem: **`post-fs-data` runs, and the magisk daemon
starts, only shortly before the UI comes up — long after `zygote` has already
been started by init.** Zygisk works by having `magiskd` present and able to
hook `zygote` **at the moment zygote launches** (it injects `libzygisk.so` into
the zygote process, which then loads modules into every app/system_server fork).
By the time our `magisk --post-fs-data` runs, zygote is already up un-hooked, so
Zygisk has nothing to attach its hook to for that boot.

On real devices and on WSA (LSPosed's MagiskOnWSA), this is solved by patching
the **boot/recovery image or the kernel** so `magiskinit` runs as PID 1's
first-stage init and is present before `zygote`. Waydroid cannot do that: it is
an **LXC container sharing the host kernel**, with no boot image and no
first-stage init of its own to patch.

So the crux is: **we need `magiskd` alive and zygote hookable before or exactly
when `zygote` starts, using only userspace changes to the system/vendor images
(no host-kernel changes).**

## 2. Constraints specific to Waydroid

- **No boot image / no kernel patch.** Anything that works must live inside
  `system.img` / `vendor.img` (or `/data`), which is all we control in this
  workflow.
- **Shared host kernel.** ptrace, `/proc`, SELinux behavior all depend on the
  host kernel config. `PTRACE_ATTACH` across the container and correct
  `yama/ptrace_scope` are not guaranteed.
- **`init` is Android's `init`, but launched by LXC**, not by a Magisk-patched
  ramdisk. We can edit `init.rc` and the `.rc` files under
  `system/system/etc/init/hw/`, which is exactly what the current build already
  does.
- **We keep the "direct install to system partition" UX** — the user still
  finishes setup inside the Magisk Delta app.

## 3. What Zygisk actually needs

For Zygisk to load, at zygote start time we need:
1. `magiskd` (from `magisk64`/`magisk32`) already running.
2. The zygote service to be (re)started **after** magiskd is up, so magiskd can
   inject `libzygisk.so` into it (Magisk Delta hooks via ptrace on the freshly
   started zygote, or via an `app_process` shim on some builds).
3. sepolicy permitting the magisk domain to ptrace/transition and zygote to
   `execmem`/load the injected lib.
4. `zygisk` enabled in Magisk config (`/data/adb/magisk` `zygisk=1`), which the
   Magisk Delta app toggles.

The current build already does (3) partially via `magiskpolicy --magisk` and the
`loadpolicy.sh` live rules. The missing piece is (1)+(2): **magiskd before a
(re)started zygote.**

## 4. Approach A (preferred): move Magisk startup to `early-init` / pre-zygote and force a zygote restart

The key insight: we do not need to be PID 1. We only need magiskd running
**before the zygote instance that apps use**. We can get there by (a) starting
magisk as early as possible in `init.rc`, and (b) making zygote start *after*
that, then relying on Magisk's existing `--zygote-restart` path so the zygote
that survives is a magiskd-hooked one.

Steps:

1. **Split the current `on post-fs-data` block.** Move the tmpfs mount + binary
   copy + `magiskpolicy` live load + `magisk --post-fs-data` up into an
   **`on early-init`** (or the earliest `on init` that has `/data`-independent
   pieces) action. The parts that genuinely need `/data` (module mounts) stay in
   `post-fs-data`, but `magiskd` itself and the daemon socket come up early.
   - Caveat: `/data` may not be mounted at `early-init`. Magisk's daemon can
     start without `/data` and re-attach later; module mounting is deferred. We
     start the daemon early, mount modules late.
2. **Delay / gate zygote.** In `init.zygoteNN.rc`, make the `zygote` service
   depend on a property that magiskd sets once it is ready, e.g. add
   `disabled` to the zygote service and `start zygote` from a magiskd
   trigger (`on property:magisk.daemon=ready`). Magisk sets a prop when up; if
   not, wrap `magisk --post-fs-data` with a `setprop magisk.daemon ready`.
   - This guarantees magiskd exists before zygote's first start, so the very
     first zygote is hookable — no restart race.
3. **Keep the `--zygote-restart` injection** as a fallback for the case where
   zygote still wins the race.
4. **Enable Zygisk config by default** so the direct-install completes with
   `zygisk=1` (write `zygisk=1` into the magisk db/config seeded under
   `/data/adb/magisk` via `loadpolicy.sh`, or let the app do it).

Risk: reordering init actions can break Waydroid boot; needs iteration. This is
the approach most likely to work without kernel changes because it removes the
race entirely by gating zygote on magiskd readiness.

## 5. Approach B: `app_process` / `libnativebridge` shim (no init reorder)

Instead of racing init, replace/shim the process that zygote's launch goes
through so the Zygisk lib is loaded in-process from the start:

- Replace `/system/bin/app_process64` (and `app_process32`) with a Magisk
  shim that `dlopen`s `libzygisk.so` (or magiskd's loader) and then `exec`s the
  real `app_process`. Magisk historically shipped exactly this kind of
  `app_process` hijack before ptrace-based Zygisk.
- Pros: no init timing games; the hook is in every zygote by construction.
- Cons: must match the exact `app_process` ABI of this LineageOS 18.1 (Android
  11) image; SELinux must allow the shim; Magisk Delta's current Zygisk may not
  ship a standalone `app_process` shim, so we may need to build one. Higher
  maintenance.

Treat B as the fallback if A's init reordering proves too fragile.

## 6. Approach C: LXC-side pre-start hook (documented, opt-in)

Waydroid runs under LXC on the host. The host controls the container config
(`/var/lib/waydroid/lxc/waydroid/config`). We can document an **optional**
`lxc.hook.pre-start` / `lxc.hook.mount` that ensures magisk bits are present in
the rootfs before init runs, or that adjusts `ptrace_scope`.

- This is the "kernel module when lxc session starts" idea from the README, but
  in pure userspace (no host kernel module). It cannot be shipped inside the
  image; it is host-side config the user opts into, so it stays a documented
  manual step, not part of the CI artifact.
- Security note (from README): host-side hooks touch the host; keep them
  minimal, opt-in, and clearly documented. Do **not** load anything into the
  host kernel.

## 7. Implementation checklist (for Approach A first)

- [ ] In `magisk.yml` **Integrate Magisk**, factor the `init.rc` injection into a
      separate file we can diff/test rather than a large heredoc.
- [ ] Add an `early-init` action that mounts `/debug_ramdisk`, copies binaries,
      and starts magiskd; keep module mounting in `post-fs-data`.
- [ ] Mark `zygote`/`zygote_secondary` services `disabled` and start them from
      `on property:magisk.daemon=ready` (set that prop from the magisk startup
      script once `magisk --post-fs-data` returns).
- [ ] Extend `loadpolicy.sh` to seed `zygisk=1` in the Magisk config so the
      direct-install lands with Zygisk on.
- [ ] Verify sepolicy: `magiskpolicy --magisk` already grants the magisk domain;
      add explicit rules for `magisk` to `ptrace` `zygote` and for `zygote`
      `execmem` if denials show up (`--live` rules in `loadpolicy.sh`).
- [ ] Add a build input `zygisk` (choice: on/off) to `workflow_dispatch` so we
      can build both variants and compare.

## 8. How to validate

There is no CI runtime for Waydroid, so validation is manual on a Fedora/Arch
host (per README, Arch needs `linux-xanmod-anbox`):

1. Build the artifact from this branch (Zygisk variant), install per README.
2. `waydroid session start`, open Magisk Delta → confirm **Zygisk: Yes** and no
   "requires reboot" nag persists after one restart.
3. Install a Zygisk-only module (Shamiko or LSPosed) and confirm it loads
   (`Zygisk` shows the module, LSPosed manager reports "activated").
4. Collect denials: `adb shell su -c dmesg | grep avc` and
   `logcat | grep -i zygisk` to feed back sepolicy rules.
5. Regression: confirm existing su + non-Zygisk modules still work (Busybox NDK,
   MagiskHidePropsConf) and normal boot time is not badly regressed by the
   zygote gating.

## 9. Fallbacks / exit criteria

- If Approach A boots but Zygisk still races → try the `disabled`+prop gating
  more aggressively, or move to Approach B (`app_process` shim).
- If neither A nor B hooks zygote reliably on the shared host kernel → document
  Approach C as the supported (manual, host-side) path and keep Zygisk marked
  experimental in the README rather than claiming full support.
- Update README **Bugs → point 1** once any path reproducibly loads a
  Zygisk-only module, downgrading it from "not working" to "experimental, see
  docs/zygisk-integration-plan.md".

## 10. Open questions

- Does this LineageOS 18.1 Android 11 zygote respect a `disabled` +
  property-triggered start without breaking Waydroid's own init overlays?
- Does the host kernel Waydroid requires allow the ptrace scope Magisk Delta's
  Zygisk needs, or must we set `kernel.yama.ptrace_scope=0` host-side?
- Does Magisk Delta's current release still support a non-ptrace (`app_process`)
  Zygisk path we could lean on for Approach B?

These need answering during the first manual build/boot cycle before committing
to one approach.
