<div align="center">
  <h1>SculkBedrock Version Packs</h1>
  <h3>Version packs (.scver) for <a href="https://github.com/SculkBedrock/SculkBedrock">SculkBedrock</a></h3>
  
  ### English | [简体中文](README_zh_CN.md)
</div>

## 📦 What is a version pack?

A version pack (`.scver`) is a single archive that carries **all game data of one Minecraft version** —
blocks, items, recipes, biomes, attributes, runtime ids, and plugins. The server reads it at boot to build
its block/item/recipe registries and to serve players of that game version.

- The server **requires** a version pack to start and accept players.
- You do **not** need to re-download the server when the game version changes: swap the pack.
- Packs are **data, not code**. Editing the JSON/NBT inside is how you customize a server.

## 🗂 Layout

```text
<pack>.scver                       (zip archive)
├── manifest.json                  pack identity: name / version / protocol_version
├── definitions/
│   ├── blocks/**/*.block.json     one file per block type (vanilla structure + sc: extensions)
│   ├── runtime.json               item identifier → network runtime id            [required]
│   ├── attributes.json            entity attribute table [required]
│   ├── entity_identifiers.nbt     vanilla entity identifier table                 [required]
│   ├── block_palette.nbt          legacy block palette (only when block_data is NOT declared)
│   ├── block_tags.json            pack-level block tags
│   ├── recipe_groups.json         crafting ingredient groups
│   ├── creative_items.json        creative inventory table
│   ├── biomes/*.biome.json        biome definitions
│   ├── biome_ids.json             biome identifier → numeric id
│   ├── biome_features.json        worldgen feature scheduling data
│   └── worldgen/                  density_function / noise / noise_settings JSON
├── behavior_packs/                vanilla behavior packs (*.zip / *.mcpack): recipes, items…
└── plugins/                       *.scplugin plugin archives
```

### `manifest.json`

```json
{
  "name": "Vanilla",
  "description": "Minecraft: Bedrock Edition vanilla version pack",
  "authors": ["..."],
  "minecraft_version": "1.26.40",
  "minecraft_network_version": "1.26.40",
  "protocol_version": 2168,
  "block_data": {
    "schema_version": 1,
    "directory": "definitions/blocks",
    "network_id_mode": "hashed"
  }
}
```

| Field | Required | Meaning |
|---|---|---|
| `name` / `description` / `authors` | ✅ | Pack identity, shown in the server version bar |
| `minecraft_version` | ✅ | Game version of this pack |
| `protocol_version` | ✅ | Network protocol version |
| `minecraft_network_version` | – | Online version string; defaults to `minecraft_version` |
| `block_data` | – | Declares the per-block JSON path. **Once declared, load failures are loud — the server never silently falls back to the legacy palette.** |

### Blocks: `definitions/blocks/**/*.block.json`

One file per **block type** (not per state, not per world coordinate). A block's states are declared as
property value sets, and the canonical state order (Cartesian product) maps 1:1 onto
`sc:protocol_runtime_ids`.

```json
{
  "format_version": "1.10.0",
  "minecraft:block": {
    "description": {
      "identifier": "minecraft:acacia_door",
      "states": { "open_bit": [false, true], "upper_block_bit": [false, true] }
    },
    "components": {
      "minecraft:destructible_by_mining": { "value": 3.0 },
      "sc:mining":  { "base_time_seconds": 3.0, "tools": [ ... ] },
      "sc:drops":  { ... }
    }
  },
  "sc:default_state": { "open_bit": false, "upper_block_bit": false },
  "sc:protocol_runtime_ids": [ ... ]
}
```

- `format_version` must be `"1.10.0"`.
- `sc:default_state` — required; must cover exactly the declared properties.
- `sc:protocol_runtime_ids` — required; length = number of state combinations, aligned1:1 with the
  canonical state order. **Never re-sorted or recomputed — it comes from the extraction source.**
- `sc:` extensions currently in use: `sc:mining`, `sc:drops`, `sc:unbreakable`, `sc:replaceable`,
  `sc:liquid`, `sc:random_tick`, `sc:needs_support`, `sc:can_contain_liquid`, `sc:workstation`.
- Unknown/undeclared components stay undeclared — the server never guesses.

### Plugins: `plugins/*.scplugin`

Each `.scplugin` is a zip containing `manifest.json` plus one native library.

```json
{
  "name": "sc_vanilla_overworld",
  "version": "0.1.0",
  "description": "Overworld world generator",
  "authors": [],
  "dependencies": [],
  "kind": "rust"
}
```

- `kind: "rust"` — in-process Rust plugin (manifest-only packaging, code compiled into the server).
- `kind: "cabi"` (default) — cross-language dynamic plugin; the archive holds `plugin.dll` / `plugin.so` /
  `plugin.dylib` and is loaded via the stable C-ABI host table.
- Plugin sources for the vanilla packs live under `crates/ur_vanilla` in this repository.

## 🚀 Usage

1. Download a `.scver` from the releases of this repository.
2. Drop it into the server's `version_packs/` directory.
3. Start the server:

```shell
cargo run -p sc_bootstrap
```

The server also accepts an **extracted directory** as long as it contains `manifest.json` (useful for
active editing — the directory form wins over a same-named archive). Dot-directories (e.g. the generated
`.scdb/` cache) are never treated as packs.

> `.scdb/` is an auto-generated sidecar cache (fast re-read of the archive). Do not commit it; it is
> rebuilt automatically when the pack file changes.

## 📏 Limits

The server validates and bounds every entry. Exceeding a limit fails the pack loudly (it is skipped) rather
than being partially loaded.

| Limit | Value |
|---|---|
| JSON entry size | 16 MiB |
| Binary entry size | 128 MiB |
| Block files per pack | 8192 |
| Bytes per block file | 1 MiB |
| Total block bytes | 128 MiB |
| Total block states | 65536 |
| States per block | 2048 |

Error messages carry pack + file + field, e.g.

```text
Vanilla: definitions/blocks/minecraft/oak_log.block.json:
$.minecraft:block.sc:protocol_runtime_ids: 长度 2 与状态组合数 3 不符
```

## 📜 Conventions

- Archive naming: `[Kind] <Name>-<Version>.scver`, e.g. `[SC原版版本包] Vanilla-1.26.40.scver`.
- Pack data must stay reproducible: when regenerating blocks or recipes, keep the generator deterministic
  and version the generator/configuration explicitly. Changing a generator must not silently change terrain
  for an existing save.
- Do not store anything else in a pack. Game logic belongs in plugins, not in pack data.

## 🙏 Feedback

Bug reports and suggestions are welcome. Please include the pack filename, `minecraft_version`,
`protocol_version`, the failing file path, and the full error line. Do not paste login tokens from logs.

## ⚖️ Disclaimer ⚖️

This repository is not affiliated with Mojang AB, Microsoft, or NetEase. No decompilation/cracking of their
games/software was performed. All content is based on public documentation / open-source projects.