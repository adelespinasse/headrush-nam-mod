# Proposal: a mod/plugin architecture for headrush-nam-mod

**Status:** design proposal, for discussion. Nothing here is implemented.

## The problem

Every HeadRush firmware mod today has to live in its own fork of this repo,
because `core/patch_pipeline.c` is a fixed sequence hardcoded to the NAM hijack.
A user who wants two mods from two authors has to merge two forks by hand.

That is already limiting, and it gets worse as more people build effect-hijack
mods: those are all *the same operation with different constants*, so every one
of them currently means another fork of the same pipeline.

This proposes a mod format and a host that can install any number of
independently distributed mods into one firmware image.

## Overview

A mod is a **zip** containing a `mod.json` manifest and a `payload/` directory.
The user drops mods into a known directory. `headrush-nam-gui` then:

1. downloads the stock updater for the detected model,
2. unpacks it (FIT → rootfs),
3. reads every mod's manifest,
4. shows a checklist, greying out mods that don't target this firmware, have
   unmet dependencies, or conflict with another selected mod,
5. applies the selected mods' operations in dependency order,
6. repacks and produces the flashable updater.

**The NAM mod becomes the first plugin.** No mod gets a privileged code path;
`core/` becomes a manifest parser plus an op executor.

```
mods/
  nam/                     ops: elf_hijack + write + exec_env + patch_bytes
  nam-extra-instances/     requires: [nam]
  mx5-usb-console/
  mx5-remote-screen/       requires: [mx5-usb-console]
```

## The central decision: ops, not scripts

**Mods declare operations. They do not ship code the host executes.**

The obvious alternative — each mod ships a script (Lua, QuickJS) that calls a
host API — was rejected. The reasoning below is the part most worth arguing
with, so it is given in full.

### What the real mods actually need

Five mods exist to design against. Their *complete* set of firmware operations:

| Mod | Operations used |
|---|---|
| `mx5-usb-console` | write ×4, symlink ×2, text-substitute-at-anchor ×2, precondition check |
| `mx5-remote-screen` | write ×2, symlink ×1, precondition check |
| `mx5-ui-mods` (tuner tempo) | byte-patch ×2, both with an expected-value guard |
| `nam` | ELF hijack, write ×3, launcher env vars, QML byte-patch ×3 |
| `nam-extra-instances` | ELF hijack (second slot), launcher env var |

The first three need **no computation at all**. They are pure data.

### The hard case: effect hijacks

`nam` does compute at install time. `core/elf_patch.c` parses the ELF program
headers, finds the gap after the R+E segment, verifies it is genuinely zeroed,
places a 32-byte trampoline there, derives `hook_slot_addr` from the data
segment, and repoints the vtable slot — refusing if the slot doesn't already
hold the expected `orig_fn`.

That is real logic. But **the logic is identical for every hijack.** Only three
inputs vary:

* `engine_vtable` — which effect
* `orig_fn` — its `process()`
* the 32-byte trampoline blob

Compare the three models in `core/model_targets.c`: same code, different
numbers. Compare v1 and V2 on one model: same code, different numbers. A new
"hijack effect X" mod is new *numbers*, not new *logic*.

That is the definition of a parameterized operation. Giving every mod author a
scripting language so they can each re-implement `elf_patch.c` — each with their
own bugs, on a device where a bad patch bricks hardware — is the wrong trade.
One audited implementation in the host, parameterized, serves all of them.

### Two findings that settle it

**1. Cave allocation is already multi-mod safe.** `core/elf_patch.c:214-230`
reclaims only `TRAMP_CODE_LEN` bytes rather than the whole inter-segment gap,
and grows the code segment's `p_filesz`/`p_memsz` by 32 and the data segment's
`p_memsz` by 4 per call. Allocation state therefore lives in the ELF's own
program headers, and the next call's `code_file_end` advances automatically.

The comment there says this was done so the V2 hijack could take its own
trampoline from the same cave. The consequence is bigger than intended: **N
independently written hijack mods compose correctly, with no host-side
allocator.** The hardest operation in the system is already re-entrant.

**2. Conflict detection is impossible with scripts.** Two mods writing the same
path, or byte-patching overlapping ranges, can be caught *before anything is
applied* when operations are declared. With scripts it is undecidable — you find
out when the device won't boot.

Costs of the script approach, for the record: a vendored interpreter on three
platforms, arbitrary code execution from downloaded zips, and no conflict
detection.

### The escape hatch that isn't needed at install time

A mod that genuinely needs computation runs it at **mod-build time**, on the
author's machine, emitting a flat manifest. This is exactly how `blobs/` already
works — `libnam_hook.so` is cross-compiled by the author, not by the user.

## The one gap in a purely static format

`elf_hijack` **produces** a value (`hook_slot_addr`) that a later op must
consume — it goes into the launcher's env. Pure static data can't express that.

Covered by **bounded substitution**: `${op_id.field}` may reference only the
declared outputs of an earlier op in the same manifest. No expressions, no
control flow, no user input. Two lines of spec, and it closes the only real
dataflow requirement found.

```json
[
  {"op": "elf_hijack", "id": "h1", "vtable": "0x17ee460",
   "orig_fn": "0x302ed0", "trampoline": "payload/tramp.bin"},
  {"op": "exec_env", "var": "NAM_HOOK_SLOT_GONK_ADDR", "value": "${h1.hook_slot}"}
]
```

## Manifest schema

```jsonc
{
  "manifest_version": 1,
  "id": "mx5-remote-screen",        // unique, kebab-case
  "name": "MX5 Remote Screen",      // shown in the UI
  "version": "1.2.0",               // semver
  "description": "Mirror and control the screen over USB.",
  "author": "…",
  "homepage": "…",

  "targets": [                      // firmware compatibility
    { "model": "mx5", "firmware": "2.7" }
  ],

  "requires": [                     // other mods, by id or capability
    { "id": "mx5-usb-console", "min_version": "1.0.0" }
  ],
  "provides": ["usb-gadget-serial"],// capability names others may require

  "ops": [ /* see below */ ]
}
```

## Op reference (v1)

| Op | Fields | Outputs |
|---|---|---|
| `require` | `path` \| `capability` | — |
| `write` | `path`, `src`, `mode` | — |
| `symlink` | `path`, `target` | — |
| `patch_bytes` | `file`, `offset`, `expect`, `src`\|`bytes` | — |
| `replace_text` | `file`, `anchor`, `replacement`, `where` | — |
| `elf_hijack` | `file`, `vtable`, `orig_fn`, `trampoline` | `hook_slot`, `trampoline_vaddr` |
| `exec_env` | `var`, `value` \| `ld_preload` | — |

Notes:

* **`expect` is mandatory on `patch_bytes`.** Offsets differ per firmware build,
  and an unguarded byte patch is how you brick a device. This follows the
  refuse-rather-than-guess convention already used throughout `core/` and in
  every one of our mods.
* **`exec_env` is a contribution, not a rewrite.** The host collects `exec_env`
  from all selected mods and rewrites Evil's exec line once. This is what
  replaces `core/launcher_script.c`'s hardcoded pair of NAM env vars.
* **`require` fails the mod, not the install.** An unmet requirement makes the
  mod unselectable in the UI with a stated reason.

## Worked examples

These are the real mods, transcribed from their existing implementations.

### `mx5-remote-screen`

From `tools/mx5_remote_screen/build_install_screen.py:58-67`.

```json
{
  "manifest_version": 1,
  "id": "mx5-remote-screen",
  "name": "MX5 Remote Screen",
  "version": "1.0.0",
  "targets": [{ "model": "mx5", "firmware": "2.7" }],
  "requires": [{ "id": "mx5-usb-console" }],
  "ops": [
    { "op": "require", "path": "/usr/Evil/Scripts/setup-usb-console.sh" },
    { "op": "write", "path": "/usr/Evil/mx5_screen_server",
      "src": "payload/mx5_screen_server", "mode": "0755" },
    { "op": "write", "path": "/lib/systemd/system/mx5-screen.service",
      "src": "payload/mx5-screen.service", "mode": "0644" },
    { "op": "symlink",
      "path": "/etc/systemd/system/multi-user.target.wants/mx5-screen.service",
      "target": "/lib/systemd/system/mx5-screen.service" }
  ]
}
```

### `mx5-usb-console`

From `tools/mx5_usb_console/build_usb_console.py`. Note the two `replace_text`
ops — this mod must insert a gadget-teardown call into two stock scripts,
because the SoC has a single UDC.

```json
{
  "id": "mx5-usb-console",
  "version": "1.0.0",
  "targets": [{ "model": "mx5", "firmware": "2.7" }],
  "provides": ["usb-gadget-serial"],
  "ops": [
    { "op": "require", "path": "/usr/Evil/Scripts/usb-otg-audio-start.sh" },

    { "op": "write", "path": "/usr/Evil/Scripts/setup-usb-console.sh",
      "src": "payload/setup-usb-console.sh", "mode": "0755" },
    { "op": "write", "path": "/usr/Evil/Scripts/remove-usb-console.sh",
      "src": "payload/remove-usb-console.sh", "mode": "0755" },
    { "op": "write", "path": "/lib/systemd/system/usb-console.service",
      "src": "payload/usb-console.service", "mode": "0644" },
    { "op": "write", "path": "/lib/systemd/system/usb-console-shell.service",
      "src": "payload/usb-console-shell.service", "mode": "0644" },

    { "op": "symlink",
      "path": "/etc/systemd/system/multi-user.target.wants/usb-console.service",
      "target": "/lib/systemd/system/usb-console.service" },
    { "op": "symlink",
      "path": "/etc/systemd/system/multi-user.target.wants/usb-console-shell.service",
      "target": "/lib/systemd/system/usb-console-shell.service" },

    { "op": "replace_text", "file": "/usr/Evil/Scripts/usb-otg-audio-start.sh",
      "anchor": "modprobe configfs", "where": "before",
      "replacement": "[ -x /usr/Evil/Scripts/remove-usb-console.sh ] && /usr/Evil/Scripts/remove-usb-console.sh" },
    { "op": "replace_text", "file": "/usr/Evil/Scripts/setup-mass-storage.sh",
      "anchor": ". /usr/Evil/Scripts/def_vars", "where": "after",
      "replacement": "[ -x /usr/Evil/Scripts/remove-usb-console.sh ] && /usr/Evil/Scripts/remove-usb-console.sh" }
  ]
}
```

### `mx5-ui-tuner-tempo`

From `tools/mx5_ui_mods/build_tuner_tempo.py:71-88`. Both patches carry the
guard the original insists on. The second one corrupts a compiled-QML unit
magic so Qt falls back to the retained source — the same-length constraint is a
property of the payload, checked by the host against `expect`'s length.

```json
{
  "id": "mx5-ui-tuner-tempo",
  "version": "1.0.0",
  "targets": [{ "model": "mx5", "firmware": "2.7" }],
  "ops": [
    { "op": "patch_bytes", "file": "/usr/Evil/Evil",
      "offset": "0x187AFC7",
      "expect": "import \"../PageController\"",
      "src": "payload/Tuner.qml", "length": 3569 },
    { "op": "patch_bytes", "file": "/usr/Evil/Evil",
      "offset": "0x2B08C70",
      "expect": "qv4cdata", "bytes": "58" }
  ]
}
```

### `nam` and `nam-extra-instances`

The point of the whole design: NAM expressed in the same ops as everyone else,
and the "up to 4 instances" option decomposed into a dependent mod rather than
an install-time checkbox.

```json
{
  "id": "nam",
  "version": "1.0.0",
  "targets": [
    { "model": "mx5",        "firmware": "2.7" },
    { "model": "pedalboard", "firmware": "2.7" },
    { "model": "gigboard",   "firmware": "2.7" }
  ],
  "provides": ["nam-runtime"],
  "ops": [
    { "op": "elf_hijack", "id": "gonk", "file": "/usr/Evil/Evil",
      "vtable":   { "mx5": "0x17ee460", "pedalboard": "0x1839044", "gigboard": "0x17f2234" },
      "orig_fn":  { "mx5": "0x302ed0",  "pedalboard": "0x3260e0",  "gigboard": "0x302840" },
      "trampoline": "payload/trampoline_gonk.bin" },

    { "op": "write", "path": "/usr/Evil/libnam_hook.so",
      "src": "payload/libnam_hook.so", "mode": "0755" },
    { "op": "write", "path": "/usr/Evil/libnam_preload.so",
      "src": "payload/libnam_preload.so", "mode": "0755" },

    { "op": "exec_env", "ld_preload": "/usr/Evil/libnam_preload.so" },
    { "op": "exec_env", "var": "NAM_HOOK_SLOT_GONK_ADDR", "value": "${gonk.hook_slot}" },

    { "op": "patch_bytes", "file": "/usr/Evil/Evil", "model": "pedalboard",
      "offset": "0x1b7beba", "expect": "Drive", "bytes_utf8": "Model" }
  ]
}
```

```json
{
  "id": "nam-extra-instances",
  "name": "NAM: up to 4 instances",
  "version": "1.0.0",
  "description": "Also hijacks Anxiety OD V2, so a rig can hold more NAM models.",
  "requires": [{ "id": "nam", "min_version": "1.0.0" }],
  "ops": [
    { "op": "elf_hijack", "id": "gonk_v2", "file": "/usr/Evil/Evil",
      "vtable":  { "mx5": "0x17ee4dc", "pedalboard": "0x18390c0", "gigboard": "0x17f22b0" },
      "orig_fn": { "mx5": "0x302ed0",  "pedalboard": "0x3260e0",  "gigboard": "0x302840" },
      "trampoline": "payload/trampoline_gonk.bin" },
    { "op": "exec_env", "var": "NAM_HOOK_SLOT_GONK_V2_ADDR", "value": "${gonk_v2.hook_slot}" }
  ]
}
```

This decomposition is not hypothetical: `core/patch_pipeline.c:202-213` already
calls the same `nam_elf_patch_gonkulator` twice with the `v2_*` constants. The
proposal only moves the second call into its own mod.

## Dependencies

`requires: [{ id, min_version? }]` and `provides: [capability]`. **No version
ranges.**

The only real dependency edge in existence — `mx5-remote-screen` needs
`mx5-usb-console` for `/dev/ttyGS1` — needs no version logic whatsoever. A
minimum version captures most of the remaining value; ranges add a resolver and
a spec section for a problem nobody has had yet, and can be added later without
breaking manifests.

The compatibility axis that actually bites here is **firmware version**, not mod
version. Offsets like `0x187AFC7` are valid for exactly one build. `targets`
matters far more than `requires`.

`provides` exists so a mod can depend on a *capability* rather than one author's
implementation — if someone writes a better USB-gadget mod, it declares
`provides: ["usb-gadget-serial"]` and satisfies the dependency.

## Ordering and conflicts

**Order:** topological by `requires`, ties broken by mod id, so a given set of
mods always produces a byte-identical image. `exec_env` contributions merge in
that order.

**Conflicts,** detected before anything is applied:

* two mods `write` or `symlink` the same path
* two `patch_bytes` ranges overlap in the same file
* two `replace_text` ops use the same anchor in the same file
* two `exec_env` ops set the same `var` to different values

Two `elf_hijack` ops are *not* a conflict — they compose by design (see above),
provided they target different `vtable` slots. Same slot twice is a conflict.

**Uninstall is free.** Every build starts from a freshly downloaded stock image,
so "uninstall" means "don't tick the box". There is no removal logic and no
partially-modded state to reason about.

## Changes required in `core/`

Prerequisites, not implemented in this proposal:

* **`core/ext4_image.c` cannot create symlinks.** `nam_ext4_inject()` hardcodes
  `LINUX_S_IFREG` (lines 246, 256). Every mod that enables a systemd unit needs
  `LINUX_S_IFLNK`. **This is the single largest concrete gap.**
* **`core/launcher_script.c` does not generalize.** It hardcodes
  `NAM_HOOK_SLOT_GONK_ADDR` and `NAM_HOOK_SLOT_GONK_V2_ADDR`. It must become a
  merge of `exec_env` contributions from all selected mods.
* **`core/patch_pipeline.c` becomes an op executor** over a mod list.
* **`core/model_targets.c`** keeps `nam_select_target()` for model detection, but
  the NAM-specific addresses move into `nam`'s manifest.
* **`app/main.c`** gains the mod checklist, with reasons shown for unselectable
  mods.

## Deliberate non-goals

* **No install-time options.** Variants are separate mods. The V2 case proves
  this decomposes; adding an options system multiplies the conflict and
  dependency space and complicates the UI for no demonstrated need.
* **No scripting.** See above.
* **No dependency resolution across the network.** Users drop in the zips they
  want; the host never fetches a mod.

## Open questions

1. **Trust.** Declarative ops stop a mod running code on the *host*, but a mod
   can still write a file that runs as root on the *device*. The manifest is at
   least auditable and diffable, which a script is not — but is that enough, or
   does this want signing or a curated index?
2. **`replace_text` fragility.** It is needed (usb-console genuinely must edit
   two stock scripts), but anchors are brittle across firmware versions and two
   mods anchoring on the same line conflict in a way users won't understand. Is
   there a better primitive — a patch-file format, or a drop-in `.d` convention?
3. **Per-model values in ops.** The `{"mx5": …, "pedalboard": …}` map used in
   `nam`'s manifest above is convenient but adds a shape to the schema. The
   alternative is one mod per model, which duplicates payloads.
4. **Distribution.** A directory of zips is proposed. Is a URL index for
   discovery worth adding, and does that reopen the trust question?
5. **Firmware version detection.** `targets` matches on model plus firmware
   version; the image exposes a `compatible` string and a description. Is that
   sufficient to pin an exact build, given offsets are build-specific?
