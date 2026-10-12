<div align="center">
  <h1>SculkBedrock 版本包仓库</h1>
  <h3><a href="https://github.com/SculkBedrock/SculkBedrock">SculkBedrock</a> 使用的版本包（.scver）</h3>

  ### [English](README.md) | 简体中文
</div>

## 📦 什么是版本包？

版本包（`.scver`）是一个压缩包，装着**某个 Minecraft版本的全部游戏数据**——方块、物品、配方、群系、
属性表、运行时 ID 以及插件。服务器启动时读取它，构建方块／物品／配方注册表，并以此服务对应游戏版本的玩家。

- 服务器**必须**有版本包才能启动并接受玩家连接。
- 游戏版本更新时**不需要**重新下载服务端：换版本包即可。
- 版本包是**数据，不是代码**。修改其中的 JSON／NBT 就是定制服务器的方式。

## 🗂 目录结构

```text
<包名>.scver                       （zip 压缩包）
├── manifest.json                  包身份：name / 版本 / protocol_version
├── definitions/
│   ├── blocks/**/*.block.json     每种方块一个文件（原版结构 + sc: 扩展）
│   ├── runtime.json               物品标识 → 网络运行时 ID                 [必需]
│   ├── attributes.json            实体属性表                               [必需]
│   ├── entity_identifiers.nbt     原版实体标识表                           [必需]
│   ├── block_palette.nbt          旧版方块调色板（仅在未声明 block_data 时使用）
│   ├── block_tags.json            版本包级别的方块标签表
│   ├── recipe_groups.json         合成原料分组表
│   ├── creative_items.json        创造模式物品表
│   ├── biomes/*.biome.json        群系定义
│   ├── biome_ids.json             群系标识 → 数字 ID
│   ├── biome_features.json        世界生成特征调度数据
│   └── worldgen/                  density_function / noise / noise_settings JSON
├── behavior_packs/                原版行为包（*.zip / *.mcpack）：配方、物品…
└── plugins/                       *.scplugin 插件包
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

| 字段 | 必需 | 含义 |
|---|---|---|
| `name` / `description` / `authors` | ✅ | 包身份，显示在服务端版本栏 |
| `minecraft_version` | ✅ | 该版本包对应的游戏版本 |
| `protocol_version` | ✅ | 网络协议版本 |
| `minecraft_network_version` | – | 联机版本号；缺省时回落到 `minecraft_version` |
| `block_data` | – | 声明"每方块一个 JSON"的新方块路径。**一旦声明，解析失败即整包拒绝，服务端绝不会静默回退到旧调色板。** |

### 方块：`definitions/blocks/**/*.block.json`

每个文件对应**一种方块类型**（不是每个状态一个文件，也不是每个世界坐标一个文件）。方块的状态以"属性值
集合"声明，规范状态顺序（笛卡尔积）与 `sc:protocol_runtime_ids` 一一对应。

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

- `format_version` 必须为 `"1.10.0"`。
- `sc:default_state` —— 必需；必须恰好覆盖全部已声明属性，且取值在值域内（缺失即拒绝，不猜测）。
- `sc:protocol_runtime_ids` —— 必需；长度 = 状态组合数，按**规范状态顺序**一一对应。
  **永远不要排序或重新推导，它来自提取源。**
- 当前在用的 `sc:` 扩展：`sc:mining`、`sc:drops`、`sc:unbreakable`、`sc:replaceable`、`sc:liquid`、
  `sc:random_tick`、`sc:needs_support`、`sc:can_contain_liquid`、`sc:workstation`。
- 未声明的组件就保持未声明——服务端绝不猜测。

### 插件：`plugins/*.scplugin`

每个 `.scplugin` 是一个 zip，内含 `manifest.json` 加一个本地动态库。

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

- `kind: "rust"` —— 进程内 Rust 插件（仅打包清单，代码已编入服务端）。
- `kind: "cabi"`（默认）—— 跨语言动态插件；包内放 `plugin.dll` / `plugin.so` / `plugin.dylib`，
  通过稳定的 C-ABI 宿主表加载。
- 原版包的插件源码位于本仓库的 `crates/ur_vanilla` 下。

## 🚀 使用方法

1. 从本仓库的 Releases 下载 `.scver`。
2. 放进服务端的 `version_packs/` 目录。
3. 启动服务端：

```shell
cargo run -p sc_bootstrap
```

服务端同时也接受**解压后的目录**（只要里面有 `manifest.json`），方便边改边调试——同名时目录优先于压缩包。
以点开头的目录（例如自动生成的 `.scdb/` 缓存）永远不会被当作版本包。

> `.scdb/` 是自动生成的旁路缓存（加速重复读取压缩包），不需要提交；版本包文件变化时会自动重建。

## 📏 限制

服务端对每个条目都会做校验和限额。超限时**整包拒绝**（跳过并报错），绝不做"部分加载"。

| 限额 | 数值 |
|---|---|
| 单个 JSON 条目大小 | 16 MiB |
| 单个二进制条目大小 | 128 MiB |
| 每包方块文件数 | 8192 |
| 单个方块文件大小 | 1 MiB |
| 方块总字节数 | 128 MiB |
| 方块状态总数 | 65536 |
| 单种方块状态数 | 2048 |

报错信息带"包名 + 文件 + 字段"，例如：

```text
Vanilla: definitions/blocks/minecraft/oak_log.block.json:
$.minecraft:block.sc:protocol_runtime_ids: 长度 2 与状态组合数 3 不符
```

## 📜 约定

- 压缩包命名：`[类别] <名称>-<版本>.scver`，例如 `[SC原版版本包] Vanilla-1.26.40.scver`。
- 数据要可复现：重新生成方块或配方时，生成器必须确定性，且生成器／配置要显式带版本。
  改动生成器**不得**让已存在的存档地形被静默改变。
- 版本包里只放数据。游戏逻辑属于插件，不属于版本包数据。

## 🙏 反馈反馈

欢迎提交 Bug 与建议。请附上：版本包文件名、`minecraft_version`、`protocol_version`、
出错的文件路径，以及完整报错行。请勿粘贴日志中的登录令牌。

## ⚖️ 免责声明 ⚖️

本仓库与 Mojang AB、Microsoft、网易无关联。未对其游戏／软件进行任何反编译或破解。
所有内容均基于公开文档／开源项目。