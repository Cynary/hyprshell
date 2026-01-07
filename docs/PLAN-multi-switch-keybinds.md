# Plan: Multiple Switch Profiles + Multiple Keybinds (Forward/Reverse)

## Summary

Hyprshell currently supports only one effective “switch open” key chord at a time via the Hyprland plugin, and its non-plugin fallback keybind generation hardcodes `Tab`, `Shift+Tab`, and `grave` regardless of `switch.key`.

This plan proposes:

- A **list of switch profiles** (`windows.switches: Vec<_>`) where each profile can have different filters/behavior.
- Each profile supports **multiple keybinds** to open **forward** or **reverse**.
- Each keybind can require **multiple base modifiers** (e.g. `Super+Alt+K`).
- **Back-compat**: migrate existing config and preserve current “reverse defaults” (Shift+Tab + grave) when reverse binds are not explicitly specified.
- **Keep fallback behavior**: when the Hyprland plugin is unavailable/disabled, hyprshell continues to behave like today (limited/default binds); document clearly that multi-binds/profiles require the plugin.

## Current State (code pointers)

- Plugin path uses `switch.key` for the *forward* “open switch” key:
  - `src/keybinds.rs` calls `exec_lib::plugin::load_plugin(switch, overview)`
  - `crates/exec-lib/src/plugin.rs` passes `xkb_key_switch_mod/key` to the plugin
- Plugin path hardcodes reverse open to `Shift+Tab` and `grave`:
  - `crates/hyprland-plugin/plugin/src/key-press.cpp` checks `ISO_Left_Tab` and `grave` for reverse open
- Fallback (non-plugin) binds hardcode `tab`, `shift+tab`, and `grave` and ignore `switch.key`:
  - `crates/windows-lib/src/keybinds.rs`
- Switch window navigation currently treats `switch.key` as the “cycle forward” key:
  - `crates/windows-lib/src/switch/create.rs` maps `s_key` to “right”

## Goals

1. **Multiple switch profiles**: allow N profiles with independent properties (filters, switch_workspaces, etc.).
2. **Multiple keybinds per profile**: allow multiple open key combos for both forward and reverse.
3. **Multiple base modifiers**: allow chords like `Super+Alt+K`.
4. **Stable behavior**:
   - Opening with any configured bind activates the intended profile.
   - Releasing one of the “hold” modifiers closes switch (configurable/defaulted).
   - Cycling within the switch UI recognizes the profile’s forward/reverse keys.
5. **Backwards compatible upgrade path** from current config.

## Non-goals / explicitly deferred

- **Making multi-profile/multi-binds fully work without the Hyprland plugin.**
  - We will keep fallback behavior (limited) and clearly document it.
  - If the plugin is disabled/unavailable, hyprshell may only support a limited subset (likely profile 0 and legacy keys).

## Proposed Config Model (v4)

### New/updated types (conceptual)

- `windows.switches: Vec<SwitchProfile>`

`SwitchProfile` includes existing behavioral settings plus keybinds:

- `filter_by: Vec<FilterBy>`
- `switch_workspaces: bool`
- `exclude_special_workspaces: String`
- `binds: SwitchBinds`

`SwitchBinds`:

- `forward: Vec<KeyCombo>` (open forward; also used as “cycle forward” keys inside the UI)
- `reverse: Vec<KeyCombo>` (open reverse; also used as “cycle reverse” keys inside the UI)

`KeyCombo`:

- `mods: Vec<KeyMod>` (supports multiple base modifiers; includes `Shift` when desired)
- `key: String`
- `hold_mods: Option<Vec<HoldMod>>` (mods that close switch on release; default = `mods` minus `Shift`)

`KeyMod` includes at least: `Alt`, `Ctrl`, `Super`, `Shift`.
`HoldMod` includes at least: `Alt`, `Ctrl`, `Super`.

### Example (RON-ish)

```ron
windows: Some((
  switches: [
    (
      filter_by: [current_monitor],
      switch_workspaces: false,
      binds: (
        forward: [
          (mods: [alt], key: "Tab"),                // classic
          (mods: [super, alt], key: "K"),           // chord
        ],
        reverse: [
          (mods: [alt, shift], key: "Tab"),         // classic reverse
          (mods: [alt], key: "grave"),              // classic reverse
        ],
      ),
    ),
    (
      filter_by: [same_class],
      switch_workspaces: true,
      binds: (
        forward: [(mods: [super], key: "Tab")],
        reverse: [(mods: [super, shift], key: "Tab")],
      ),
    ),
  ],
))
```

## Migration Plan (v3 -> v4)

Inputs in v3:

- `windows.switch: Option<Switch>`
- `windows.switch_2: Option<Switch>`
- `Switch { modifier, key, filter_by, switch_workspaces, exclude_special_workspaces }`

Migration output:

- Build `windows.switches` from any present `switch` and `switch_2` (in that order).
- For each migrated profile:
  - If `binds.forward` not specified (will be empty in migrated output), set `forward = [(mods: [modifier], key: key)]`.
  - If `binds.reverse` not specified, set reverse defaults to match today’s behavior:
    - `[(mods: [modifier, shift], key: "Tab"), (mods: [modifier], key: "grave")]`
  - Set `hold_mods` implicitly (default behavior) unless we decide to encode explicitly.
- If neither `switch` nor `switch_2` existed, `switches` is empty (feature disabled).

## Implementation Tasks (suggested execution order)

### Phase 0: Design/decisions (small, but must be explicit)

1. **Lock down config schema**
   - Confirm exact serialized shape for `KeyCombo`, `KeyMod`, `HoldMod`, and `SwitchProfile`.
   - Decide whether `binds.forward/reverse` can be empty and how that interacts with defaults.
2. **Define close semantics**
   - Default: close when *any* `hold_mod` is released.
   - Explicit override via `hold_mods` per key combo (optional).
3. **Define how UI cycling works**
   - Use the profile’s `binds.forward[*].key` as “cycle forward” keys.
   - Use the profile’s `binds.reverse[*].key` as “cycle reverse” keys.
   - Keep arrow keys and `h/j/k/l` as additional navigation.

### Phase 1: Core schema + migration (config-lib)

4. **Add v4 structs**
   - Update `crates/config-lib/src/structs.rs`:
     - Add `SwitchProfile`, `SwitchBinds`, `KeyCombo`, `KeyMod`, `HoldMod`.
     - Replace `Windows.switch`/`Windows.switch_2` with `Windows.switches: Vec<SwitchProfile>`.
   - Ensure `SmartDefault`/`serde(default)` behavior is correct and `deny_unknown_fields` remains valid.
5. **Add migration module**
   - Bump `crates/config-lib/src/lib.rs` `CURRENT_CONFIG_VERSION` to 4.
   - Add `crates/config-lib/src/migrate/m3t4/...` with conversion logic described above.
   - Update `migrate/mod.rs` wiring to include the new migration.
6. **Update docs-generated config**
   - Ensure `hyprshell config generate` (wherever implemented) emits v4 with `switches` and reasonable defaults.

### Phase 2: IPC payload changes (core-lib) + runtime routing (src)

7. **Extend transfer payload**
   - Update `crates/core-lib/src/transfer/structs.rs`:
     - Extend `OpenSwitch` to include `profile_id` and `hold_mods`.
     - Use `#[serde(default)]` for new fields to keep old messages working.
8. **Select profile in main event handler**
   - Update `src/receive_handle.rs`:
     - Route `OpenSwitch` to the requested profile.
     - Track “currently open profile id” so `CloseSwitch` and `SwitchSwitch` apply to the correct instance.
9. **Create switch windows for profiles**
   - Update `src/start.rs`:
     - Create/hold one switch window per profile *or* keep one switch window and dynamically apply profile config on open.
   - Preferred approach (less UI duplication): keep a single switch window and update its `WindowsSwitchConfig` on open.

### Phase 3: Switch window behavior updates (windows-lib)

10. **Make switch window understand multiple keys**
   - Update `crates/windows-lib/src/switch/create.rs`:
     - Replace single `s_key` with `HashSet<Key>` for forward keys and for reverse keys.
     - Update key handler to cycle right/left based on those sets.
11. **Make close-on-release support multiple hold modifiers**
   - Update `crates/windows-lib/src/switch/create.rs` release handler to close switch if released key corresponds to any active `hold_mods`.
   - Store “active hold_mods” per activation in `WindowsSwitchData` (set when opening).
12. **Apply profile properties at open**
   - Update `crates/windows-lib/src/switch/open.rs` and/or `WindowsSwitchData` so open uses the selected profile’s filter/switch_workspaces properties.

### Phase 4: Hyprland plugin support for multi-bind + multi-mod chords

13. **Update Rust-side plugin config**
   - Update `crates/exec-lib/src/plugin.rs` and `crates/hyprland-plugin/src/configure.rs`:
     - Replace single `xkb_key_switch_mod/key` with a list of switch bindings.
     - Each binding includes:
       - Required key + modifiers (including Shift when used)
       - Serialized transfer string for `OpenSwitch { profile_id, reverse, hold_mods }`
14. **Update generated C++ plugin behavior**
   - Update `crates/hyprland-plugin/plugin/src/defs.h` templating to generate arrays/tables for bindings.
   - Update `crates/hyprland-plugin/plugin/src/key-press.cpp`:
     - Detect required modifier combinations (supports multiple base mods).
     - Match key press against binding list and send corresponding transfer.
     - Track current “active hold_mods” (or the exact binding used) and close switch on release of any hold modifier.
15. **Keep reverse defaults for back-compat**
   - If config’s reverse list is empty, ensure plugin uses the default reverse bindings described above.

> Note: plugin reload detection currently compares a `PluginConfig` string embedded in plugin description. With many bindings this may become long; plan to replace it with a stable hash of the config payload to keep the description short/deterministic.

### Phase 5: “Fallback behavior” (non-plugin) stays limited + documentation note

16. **Keep fallback binds limited**
   - Update `crates/windows-lib/src/keybinds.rs` only as needed to compile with the new config schema.
   - Behavior goal:
     - Still provide legacy open binds for *one* profile (likely profile 0).
     - Do not attempt to support multiple profiles/multi-binds without plugin.
17. **Document plugin requirement**
   - Update `docs/CONFIGURE.md` and/or `README.md`:
     - Clearly state: multi-switch/multi-bind functionality requires the Hyprland plugin; fallback behavior is limited (legacy defaults).
   - Optionally emit a `notify_warn` message when plugin is disabled/unavailable and config uses `windows.switches` with multiple binds/profiles.

### Phase 6: Config editor updates (config-edit-lib)

18. **Update editor structs**
   - Update `crates/config-edit-lib/src/structs.rs` to represent `switches: Vec<...>` and bind lists.
19. **Update editor UI**
   - Replace fixed `switch` + `switch_2` UI with a list UI:
     - Add/remove/reorder switch profiles
     - Edit per-profile properties
     - Edit forward/reverse bind lists with multi-mod selection
20. **Update “changes” diff view**
   - Update `crates/config-edit-lib/src/components/changes.rs` to show list diffs and bind changes.

### Phase 7: Tests + sanity checks

21. **Serialization tests**
   - Add a unit test for `TransferType::OpenSwitch` serde round-trip with new fields + compatibility with old payloads.
22. **Plugin build test**
   - Extend `crates/hyprland-plugin/src/test.rs` to cover multiple bindings and multiple modifiers.

## Rollout / Review checkpoints

- After Phase 1: config loads + migrates cleanly; config generation emits v4.
- After Phase 2–3: runtime can open the correct profile and close correctly.
- After Phase 4: plugin recognizes multiple keybinds and multi-mod chords reliably; reverse defaults work.
- After Phase 6: config editor can create/edit profiles and binds.
- After Phase 7: basic tests cover serialization + plugin build.

