# SecretFlasherManaka 自定义任务开发文档

> 基于 v1.1.1 版本 · 通过实际任务「夜の放課後 —— 月光下的独白」逆向分析整理

---

## 目录

1. [概述](#1-概述)
2. [文件结构与存放位置](#2-文件结构与存放位置)
3. [JSON 顶层结构](#3-json-顶层结构)
4. [Zone 系统 —— 区域定义](#4-zone-系统--区域定义)
5. [SubCondition —— 子条件系统](#5-subcondition--子条件系统)
6. [Checkpoint —— 任务核心单元](#6-checkpoint--任务核心单元)
   - [6.1 纯描述 CP](#61-纯描述-cp)
   - [6.2 条件触发 CP](#62-条件触发-cp)
   - [6.3 Oncomplete 动作 CP](#63-oncomplete-动作-cp)
   - [6.4 ItemConditions 物品检测 CP](#64-itemconditions-物品检测-cp)
   - [6.5 复合 CP](#65-复合-cp)
7. [Condition 完整清单](#7-condition-完整清单)
   - [7.1 动作检测 (Action_*)](#71-动作检测-action_-)
   - [7.2 状态检测](#72-状态检测)
   - [7.3 装备检测 (Cosplay_*)](#73-装备检测-cosplay_-)
   - [7.4 物品检测 (Item_*)](#74-物品检测-item_-)
   - [7.5 子条件引用](#75-子条件引用)
   - [7.6 复合条件](#76-复合条件)
   - [7.7 否定条件](#77-否定条件)
8. [Oncomplete 完整清单](#8-oncomplete-完整清单)
9. [ItemConditions 物品世界检测](#9-itemconditions-物品世界检测)
10. [任务流程设计模式](#10-任务流程设计模式)
    - [10.1 区域导航模式](#101-区域导航模式)
    - [10.2 物品链模式](#102-物品链模式)
    - [10.3 传送与穿衣控制](#103-传送与穿衣控制)
    - [10.4 Teleport + Start 双 Zone 模式](#104-teleport--start-双-zone-模式)
    - [10.5 约束与解除模式](#105-约束与解除模式)
11. [叙事技巧](#11-叙事技巧)
12. [常见问题与陷阱](#12-常见问题与陷阱)
13. [完整任务参考](#13-完整任务参考)

---

## 1. 概述

SecretFlasherManaka 的自定义任务系统允许通过 JSON 文件创建完整的剧情任务。任务由一系列 **Checkpoint（CP）** 组成，玩家按顺序触发，配合区域定位、动作检测、状态条件等机制实现交互叙事。

核心概念：

- **Zone（区域）**：地图上的物理区域，用于定位玩家位置
- **Checkpoint（检查点）**：任务的最小执行单元，按数组顺序触发
- **Condition（条件）**：CP 触发所需的玩家状态/动作检测
- **Oncomplete（完成动作）**：CP 结束后自动执行的操作（传送、换装、振动控制等）
- **ItemConditions（物品条件）**：检测场景中是否存在特定掉落物品

---

## 2. 文件结构与存放位置

任务文件存放于游戏目录下的 `CustomMissions/` 文件夹：

```
SecretFlasherManaka v1.1.1/
├── CustomMissions/
│   ├── blackmail.json          # 社区任务：Blackmail
│   ├── blackmail2.json         # 社区任务：Blackmail 2
│   ├── blackmail3.json         # 社区任务：Blackmail 3
│   └── moonlight_monologue.json # 自定义任务示例
├── BepInEx/
├── NPCBehaviorMod/
└── ...
```

任务文件为标准的 UTF-8 编码 JSON 格式。文件名会显示在游戏任务选择界面中（基于 `title` 字段）。

---

## 3. JSON 顶层结构

```json
{
  "title": "任务标题",
  "zones": [ /* 区域定义数组 */ ],
  "subconditions": [ /* 子条件定义数组（可选） */ ],
  "checkpoints": [ /* 检查点数组 */ ]
}
```

### 字段说明

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `title` | string | 是 | 任务标题，显示在游戏任务选择界面 |
| `zones` | array | 是 | 地图区域定义，至少需要一个区域 |
| `subconditions` | array | 否 | 可复用的子条件，用于简化 CP 编写 |
| `checkpoints` | array | 是 | 检查点列表，按数组顺序依次执行 |

---

## 4. Zone 系统 —— 区域定义

Zone 是地图上的物理区域，CP 通过 `zone` 字段引用。一个 Zone 可包含多个 Area（碰撞体）。

### Zone 结构

```json
{
  "id": "zone_identifier",
  "areas": [
    {
      "type": "sphere",
      "stage": "Park",
      "x": 67.99,
      "y": 50.04,
      "z": 31.76,
      "r": 8,
      "outlinehidden": false,
      "compasshidden": false
    }
  ]
}
```

### 参数说明

| 参数 | 类型 | 说明 |
|---|---|---|
| `id` | string | 区域唯一标识符，CP 中通过 `"zone": "id"` 引用 |
| `areas[].type` | string | 碰撞体类型，目前已知 `sphere`（球体） |
| `areas[].stage` | string | 所属场景，已知值：`Park`（公园）、`Residence`（自宅）、`Mansion`（公寓）、`Apart`（另一公寓） |
| `areas[].x/y/z` | float | 世界坐标 |
| `areas[].r` | float | 球体半径（必填），控制 zone 检测范围 |
| `areas[].outlinehidden` | bool | 是否隐藏地图上的 zone 轮廓环。必须为 `false` 才能看到任务区域 |
| `areas[].compasshidden` | bool | 是否隐藏罗盘上的 zone 指示。必须为 `false` 才能看到方向引导 |

### 坐标获取方法

坐标通过 Unity 游戏内的坐标读取工具（如 Cheat Engine、Cinematic Unity Explorer）获取。玩家位置坐标在内存中的偏移量可通过 Cheat Engine 表定位。当你需要坐标时，向玩家索要即可。

### 任务「月光下的独白」Zone 定义

| zone id | x | y | z | 说明 |
|---|---|---|---|---|
| `teleport` | 67.99 | 50.04 | 31.76 | 传送点（与 start 同坐标） |
| `start` | 67.99 | 50.04 | 31.76 | 公园入口 |
| `fountain` | 74.45 | 52.06 | 79.38 | 喷泉 |
| `garden` | 113.67 | 49.0 | 42.17 | 花坛 |
| `vending` | 30.85 | 50.07 | 41.07 | 自动贩卖机 |
| `tree` | 49.88 | 51.2 | 129.58 | 树下 |
| `toilet` | 114.03 | 49.06 | 123.03 | 公厕外 |
| `urinal` | 123.79 | 49.06 | 125.18 | 公厕内小便池 |
| `parkArea` | 68 | 50 | 80 | 公园通用区域（兜底用） |

> **注意**：CP 中应尽量使用具体的 zone，避免使用 parkArea 这种兜底区域，否则玩家可能在任意位置触发 CP。`parkArea` 可以作为未到达特定区域时的过渡处理。

---

## 5. SubCondition —— 子条件系统

SubCondition 是可复用的条件模板，在 CP 中通过 `[SubCondition_模板ID]` 引用。主要用于任务开始前的状态重置。

### 定义格式

```json
{
  "subconditions": [
    {
      "id": "startConditions",
      "condition": "!(Crouching,HandcuffsBack,Blindfolded,Bodypaint,VibrationHigh,PistonHigh)"
    }
  ]
}
```

### 语法

- 格式：`!(State1,State2,State3)`
- 含义：**不在**所列出的任何一种状态下
- 多条用逗号分隔，`!` 开头的括号组整体取反

### 使用方式

在 CP 的 condition 中引用：
```json
"condition": "[SubCondition_startConditions]"
```

这会在进入该 CP 前检查玩家不处于 Crouching、HandcuffsBack、Blindfolded、Bodypaint、VibrationHigh、PistonHigh 中的任何一种状态。如果玩家处于这些状态，CP 不会触发。

---

## 6. Checkpoint —— 任务核心单元

Checkpoint 是任务的基本组成单元，存储在 `checkpoints` 数组中，按顺序依次触发（除非被条件阻塞）。

### 6.1 纯描述 CP

最简单的 CP 类型——只有叙事文本，无任何条件检测。通常作为过渡、场景描写。

```json
{
  "zone": "start",
  "condition": {
    "description": "（放学后的公园格外安静……）",
    "duration": 6
  }
}
```

执行流程：
1. 玩家进入 `start` 区域 → 显示描述文字（6 秒）
2. 倒计时结束后 → 自动进入下一个 CP

> **关键**：如果两个连续的 CP 在**同一 zone 且均为纯描述**，它们会按顺序依次播放，不需要玩家移动。

### 6.2 条件触发 CP

需要玩家执行特定动作或处于特定状态才能触发。

```json
{
  "zone": "urinal",
  "condition": {
    "description": "（意识往深渊沉下去……身体被贯穿的感觉像潮水一样淹过头顶。某人的呼吸喷在后颈上。）",
    "condition": "Orgasm",
    "duration": 4
  }
}
```

执行流程：
1. 玩家在 `urinal` 区域
2. 玩家进入高潮状态（Orgasm）
3. 显示描述文字（4 秒）
4. 进入下一个 CP

### 6.3 Oncomplete 动作 CP

CP 结束后自动执行一系列游戏操作。

```json
{
  "zone": "teleport",
  "condition": {
    "description": "正在重置状态……",
    "condition": "[SubCondition_startConditions]",
    "duration": 5,
    "oncomplete": [
      { "type": "setStage", "daytime": false },
      { "type": "unequipAllCosplay" },
      { "type": "unequipAdultToy", "parts": ["Vibrator", "TitRotor", "KuriRotor"] },
      { "type": "equipCosplay", "parts": ["m_cosplay_school_hoodie_hoodie", "…"] },
      { "type": "setPlayerPosition", "x": 67.99, "y": 50.04, "z": 31.76 },
      { "type": "setVibrator", "level": "Off" },
      { "type": "dropItem", "itemtype": "HandcuffKey", "stage": "Park", "x": 49.88, "y": 51.2, "z": 129.58 }
    ]
  }
}
```

### 6.4 ItemConditions 物品检测 CP

检测世界中是否有特定类型的掉落物品，用于确认玩家已捡起/丢弃物品。

```json
{
  "zone": "tree",
  "condition": {
    "description": "（我蹲下来，伸出手，指尖触到了什么冰凉的东西）……果然在这里。",
    "itemconditions": [
      { "type": "VibeRemocon", "zone": "tree" }
    ],
    "duration": 5,
    "oncomplete": [
      { "type": "dropItem", "itemtype": "VibeRemocon", "stage": "Apart", "x": 0, "y": 0, "z": 0 }
    ]
  }
}
```

### 6.5 复合 CP

同时使用多种机制。

```json
{
  "zone": "urinal",
  "condition": {
    "description": "（不知道什么时候手铐已经从手腕上松脱了……）",
    "condition": "Action_None",
    "duration": 8,
    "oncomplete": [
      { "type": "setVibrator", "level": "Off" },
      { "type": "unequipAdultToy", "parts": ["Vibrator", "KuriRotor"] },
      { "type": "setPlayerPosition", "x": 67.99, "y": 50.04, "z": 31.76 }
    ]
  }
}
```

---

## 7. Condition 完整清单

以下条件基于官方任务和实际测试整理。

### 7.1 动作检测 (Action_\*)

玩家必须**主动执行**该动作。游戏内通过物品栏选择对应道具操作。

| Condition | 说明 | 适用场景 |
|---|---|---|
| `Action_None` | 空闲/未执行任何动作 | 检测玩家站立不动 |
| `Action_Dogeza` | 土下座（跪地叩首） | 忏悔、谢罪、屈服场景 |
| `Action_DrinkWater` | 喝水动作 | 口渴、平复心情 |
| `Action_HandcuffsAtMap` | 将手铐铐在地图物体上（如水管） | 壁铐/束缚场景 |
| `Action_SexStandBack` | 背后式站立 | 小便池等场景 |
| `Action_UseBuyMachine` | 使用自动贩卖机 | 购买饮料等 |
| `Action_SitDildo` | 坐在假阳具上 | 自慰场景 |
| `Action_PeeStand` | 站立小便 | 羞耻场景 |
| `Action_PeeDog` | 狗爬式小便 | 羞辱场景 |
| `Action_OnaniNeGanimata` | 虾弓腿自慰 | 自慰场景 |
| `Action_MituasiOnani` | 蜜脚自慰 | 自慰场景 |
| `Action_UseDildoFloorAnal1` | 使用地板假阳具（肛） | 插入场景 |
| `Action_UseDildoWallAnal1` | 使用墙壁假阳具（肛） | 插入场景 |
| `Action_UseDildoWallFella1` | 使用墙壁假阳具（口） | 口交场景 |

### 7.2 状态检测

玩家正在经历某种状态。

| Condition | 说明 |
|---|---|
| `Orgasm` | 正在高潮 |
| `VibrationHigh` | 跳蛋/振动棒处于高档 |
| `VibrationLow` | 跳蛋/振动棒处于低档 |
| `VibrationOff` | 振动器已关闭 |
| `Blindfolded` | 蒙眼状态 |
| `Sitting` | 坐姿 |
| `KeyedHandcuffs` | 被钥匙手铐反铐双手（背后） |
| `Item_HandcuffKey` | 物品栏中有手铐钥匙 |
| `Item_VibeRemocon` | 物品栏中有跳蛋遥控器 |
| `Crouching` | 蹲伏 |
| `Bodypaint` | 身体涂鸦状态 |
| `PistonHigh` | 活塞高频状态 |

### 7.3 装备检测 (Cosplay_\*)

检测是否**未**穿着指定服装部件。所有已知条件均以否定形式使用。

| Condition | 说明 |
|---|---|
| `!Cosplay_m_cosplay_school_hoodie_hoodie` | 未穿连帽衫 |
| `!Cosplay_m_cosplay_school_hoodie_ribbon` | 未戴蝴蝶结 |
| `!Cosplay_m_cosplay_school_hoodie_shirt` | 未穿衬衫 |
| `!Cosplay_m_cosplay_school_hoodie_skirt` | 未穿裙子 |
| `!Cosplay_m_cosplay_school_gal_bag` | 未背包 |
| `!Cosplay_m_cosplay_suit_chic_jacket` | 未穿 chic 外套 |
| `!Cosplay_m_cosplay_suit_chic_panty` | 未穿 chic 内裤 |
| `!Cosplay_m_cosplay_suit_chic_shirt` | 未穿 chic 衬衫 |
| `!Cosplay_m_cosplay_suit_chic_skirt` | 未穿 chic 裙子 |
| `!Cosplay_m_cosplay_suit_chic_stocking` | 未穿 chic 丝袜 |
| `!Sitting` | 非坐姿 |

> **推测命名规则**：服装部件 ID 格式为 `m_cosplay_<套装>_<部件>`。

### 7.4 物品检测 (Item_\*)

检测玩家物品栏中是否有指定物品。

| Condition | 说明 |
|---|---|
| `Item_HandcuffKey` | 物品栏中有手铐钥匙 |
| `Item_VibeRemocon` | 物品栏中有跳蛋遥控器 |

否定形式 `!Item_HandcuffKey` 检测物品栏中没有该物品。

> **`Item_*` vs `itemconditions` 的区别**：
> - `Item_HandcuffKey`：检测玩家**物品栏**中是否有手铐钥匙（已在包里）
> - `itemconditions: [{type: "HandcuffKey", zone: "urinal"}]`：检测世界中**某区域地板上**是否有掉落的手铐钥匙

### 7.5 子条件引用

```json
"condition": "[SubCondition_startConditions]"
```

引用 `subconditions` 中定义的预置条件模板。

### 7.6 复合条件

多个条件用逗号分隔，形成 AND 关系：

```json
"condition": "[KeyedHandcuffs,!Item_HandcuffKey]"
```

表示玩家被手铐反铐 **且** 物品栏中没有钥匙。

```json
"condition": "[Action_UseDildoFloorAnal1,Orgasm]"
```

表示玩家使用地板假阳具 **且** 正在高潮。

### 7.7 否定条件

在条件名前加 `!`：

| 写法 | 含义 |
|---|---|
| `!Item_HandcuffKey` | 物品栏中没有钥匙 |
| `!Cosplay_m_cosplay_school_hoodie_hoodie` | 没穿连帽衫 |
| `!Blindfolded` | 未被蒙眼 |
| `!KeyedHandcuffs` | 未被手铐反铐 |
| `!Sitting` | 非坐姿 |

---

## 8. Oncomplete 完整清单

Oncomplete 是 CP 条件满足后自动执行的一系列动作，从上到下顺序执行。

### 传送与场景

| Type | 参数 | 说明 |
|---|---|---|
| `setPlayerPosition` | `x, y, z, rx?, ry?, rz?, rw?` | 传送玩家到指定坐标 |
| `setStage` | `daytime: true/false` | 切换昼夜 |

> `setPlayerPosition` 传送后**不会自动穿衣服**（Option A 模式），适合裸身传送叙事。
> 旋转参数（rx, ry, rz, rw）为四元数，可选。

### 换装

| Type | 参数 | 说明 |
|---|---|---|
| `equipCosplay` | `parts: [数组]` | 穿上指定的服装部件 |
| `unequipAllCosplay` | — | 脱掉所有服装 |
| `equipAdultToy` | `parts: [数组]` | 装备指定的成人玩具（插入） |
| `unequipAdultToy` | `parts: [数组]` | 卸下指定的成人玩具 |

### 振动控制

| Type | 参数 | 说明 |
|---|---|---|
| `setVibrator` | `level: "Off"/"Low"/"High"` | 设置振动器档位 |

### 物品操作

| Type | 参数 | 说明 |
|---|---|---|
| `dropItem` | `itemtype, stage, x, y, z?` | 在世界中掉落指定物品 |
| `dropItem` | `itemtype, stage: "Apart"` | 将物品从世界中移除（销毁） |

> `dropItem` 的两种用法：
> 1. 掉落物品到指定坐标供玩家拾取：`{"type":"dropItem","itemtype":"HandcuffKey","stage":"Park","x":49.88,"y":51.2,"z":129.58}`
> 2. 物品消失/清理：`{"type":"dropItem","itemtype":"HandcuffKey","stage":"Apart","x":0,"y":0,"z":0}`
>
> **⚠️ 注意**：`stage: "Apart"` 时必须显式填写 `x:0, y:0, z:0` 三个坐标参数。漏写会导致游戏无法正确解析任务文件，表现为 zone 圈圈不渲染（无报错，静默失败）。

### 动作控制

| Type | 参数 | 说明 |
|---|---|---|
| `setAction` | `action: string` | **强制设置玩家当前动作/姿势**。参数值为条件名去掉 `Action_` 前缀，如 `"None"`（空闲）、`"HandcuffsAtMap"`（壁铐姿势） |
| `unlockHandcuffs` | — | 解除手铐状态（无论类型） |

### 完整示例

```json
"oncomplete": [
  { "type": "setVibrator", "level": "Off" },
  { "type": "unequipAdultToy", "parts": ["Vibrator", "KuriRotor"] },
  { "type": "setPlayerPosition", "x": 67.99, "y": 50.04, "z": 31.76 }
]
```

### 重要：物品栏由游戏引擎自动管理

自定义任务 JSON 中**没有**直接操作玩家物品栏（添加/删除/清空道具）的 oncomplete 类型。但游戏引擎会在任务开始/结束时自动处理物品栏：

| 时机 | 引擎行为 |
|---|---|
| 任务开始时 | 自动保存玩家当前物品栏快照，然后清空物品栏 |
| 任务完成/取消时 | 自动恢复任务开始时保存的物品栏快照 |

这意味着：
- 任务开发**无需**关心玩家进入任务前带了什么
- 任务内给玩家的道具只能通过 `dropItem` 掉落在世界中，由玩家手动拾取
- 任务内丢失/被丢弃的服装，在任务结束后引擎会自动归还（玩家原本穿着的会被恢复）
- `equipCosplay` 作用于身体装备栏，不影响物品栏

> 这一机制在 `blackmail.json` 中得到验证：任务中玩家被迫丢弃 chic 衬衫、内裤、丝袜等，但任务结束后所有物品完好归还。

---

## 9. ItemConditions 物品世界检测

`itemconditions` 用于检测世界中特定位置是否有掉落物品。与 `Item_*` 条件不同，它检测的是**地面上的物品**而非玩家的物品栏。

### 格式

```json
{
  "zone": "tree",
  "condition": {
    "description": "（我蹲下来摸到了那个东西……）",
    "itemconditions": [
      { "type": "VibeRemocon", "zone": "tree" }
    ],
    "duration": 5
  }
}
```

### 参数

| 参数 | 类型 | 说明 |
|---|---|---|
| `type` | string | 物品类型。已知值：`VibeRemocon`, `HandcuffKey`, `DildoWall` |
| `zone` | string | 物品所在区域 ID，玩家必须进入该区域才能检测到 |

### 典型流程

```
dropItem (oncomplete) → 物品出现在地面 → 玩家走进该 zone
  → itemconditions 检测到物品存在 → CP 触发
  → oncomplete 清理物品（dropItem to Apart）
```

### 物品链模式

以「月光下的独白」为例：

```
CP0: dropItem HandcuffKey 到 tree 区域
     dropItem VibeRemocon 到 start 区域
     ↓
玩家经过 start 时捡起遥控器（游戏内手动拾取）
     ↓
CP34 (tree): itemconditions 检测到 VibeRemocon 在 tree
     → oncomplete 将 VibeRemocon 从世界移除 (Apart)
     （玩家从物品栏中手动拿出遥控器丢在地上）
     ↓
CP42 (urinal): !Item_HandcuffKey 检测（玩家已从物品栏拿出钥匙）
     ↓
CP43: itemconditions 检测到 HandcuffKey 在 urinal
     → oncomplete 将 HandcuffKey 从世界移除 (Apart)
```

> **注意**：itemconditions 检测到物品存在后，应立即用 `dropItem to Apart` 清理，否则检测条件会一直满足。
> 且 `dropItem` 的 `stage: "Apart"` 必须带 `x:0, y:0, z:0` 参数（详见 Oncomplete 章节）。

---

## 10. 任务流程设计模式

### 10.1 区域导航模式

当玩家需要从一个区域移动到另一个区域时，在文本中加入**方向指引（compass guide）**帮助玩家找到路。

```json
{
  "zone": "start",
  "condition": {
    "description": "（我攥紧书包带子）……算了，想再多也没用。既然来了，就没有回头路了。迈出这一步吧。沿着砂石路往前走——喷泉的水声在夜里应该会很清晰。",
    "duration": 5
  }
}
```

导航写作要点：
- 目标到目标之间用**地标**指引（喷泉→贩卖机→大树→公厕）
- 描述**环境声音/光影**辅助定位（"水声"、"路灯"、"月光"）
- 每个区域过渡 CP 提示**下一个目标的方向**

典型导航序列：
```
start → "沿着砂石路往前走—喷泉的水声在夜里应该会很清晰"
fountain → "沿着喷泉边绕过，前面的花坛在月光下泛着灰白色"
garden → "穿过花坛，左手边能看到自动贩卖机的灯光"
vending → "贩卖机旁边有一条小径通向公园深处，尽头那棵大榕树在月光下很显眼"
tree → "沿着小径继续往里走，公厕的轮廓在前面的树影间浮现"
toilet → "男厕的门虚掩着……"
urinal → （进入目的地）
```

### 10.2 物品链模式

用于"拾取→使用→丢弃"的流程。

```
① 掉落物品到世界（CP0 oncomplete: dropItem）
  ↓
② 玩家手动拾取（游戏内交互）
  ↓
③ 玩家走到目标区域，拿出物品丢弃（游戏内操作）
  ↓
④ itemconditions 检测到物品在地面 → CP 触发
  ↓
⑤ oncomplete 清理物品（dropItem to Apart）
```

关键点：
- 物品掉落坐标与 zone 定义坐标应保持一致
- 清理物品的 `dropItem` 使用 `stage: "Apart"` 使物品消失
- `VibeRemocon` 掉落检测通常搭配 `setVibrator` 使用（跳蛋遥控器丢在地上 → 确认跳蛋还在体内工作）
- `HandcuffKey` 在壁铐流程中需要比手铐动作**先**被丢弃

### 10.3 传送与穿衣控制

`setPlayerPosition` 传送时**不会自动穿衣服**，这是设计特性（Option A）——可以利用它在传送后保持裸身状态延续叙事。

```json
// 传送——裸身到达
{ "type": "setPlayerPosition", "x": 67.99, "y": 50.04, "z": 31.76 }

// 后续 CP 手动穿衣服
CP62: （回过神来的时候，我发现自己站在公园入口的路灯下。全身赤裸……）
CP63: （我蹲下来，手忙脚乱地从书包里翻出校服……）
```

传送时机：
- 建议在 `Action_None`（空闲检测）后传送，确保玩家已完成当前动作
- 传送前关闭振动器、卸下玩具，避免状态残留

### 10.4 Teleport + Start 双 Zone 模式

每个任务需要至少两个 zone：`teleport` 和 `start`。两者通常位于同一坐标，但半径不同。

```json
{
  "id": "teleport",
  "areas": [{
    "type": "sphere", "stage": "Park",
    "x": 67.99, "y": 50.04, "z": 31.76,
    "r": 2,
    "outlinehidden": false,
    "compasshidden": false
  }]
},
{
  "id": "start",
  "areas": [{
    "type": "sphere", "stage": "Park",
    "x": 67.99, "y": 50.04, "z": 31.76,
    "r": 12,
    "outlinehidden": false,
    "compasshidden": false
  }]
}
```

**设计意图**：

- `teleport`：半径 2~4，用作 CP0 的初始触发区。玩家一进入任务，CP0 立即触发，通过 `setPlayerPosition` 将玩家传送至正确的起始位置
- `start`：半径 10~15，用作主要游戏区域。所有后续 CP 都放在这个 zone

即使 `teleport` 和 `start` 坐标完全相同，双 zone 结构也保证了 CP0 能正确触发初始化流程。官方所有任务都遵循此模式。

**Zone 可见性参数**：每个 area 必须显式设置：
- `"outlinehidden": false` —— 地图上显示 zone 轮廓环
- `"compasshidden": false` —— 罗盘上显示 zone 方向
- 不设置这两个字段默认为 `true`（隐藏），玩家看不到任务区域

### 10.5 约束与解除模式

```
① 手铐掉落在树下 → 玩家拾取
② 玩家走到小便池 → 将手铐铐在 pipe 上（Action_HandcuffsAtMap）
③ 性爱场景（Action_SexStandBack → 叙事描写 → Orgasm）
④ 高潮后察觉手铐松动（叙事描写，oncomplete unlockHandcuffs 静默解锁）
⑤ Action_None 检测空闲 → 传送回家
```

> `Action_HandcuffsAtMap` 是"将手铐固定在场景物体上"的动作，不同于背铐（`KeyedHandcuffs`）。两者使用不同的条件检测。
> `unlockHandcuffs` 可以解除所有类型的手铐状态，不需要玩家手动操作钥匙。

---

## 11. 叙事技巧

### 第一人称内独白

任务使用第一人称（我）内心独白风格，括号 `（）` 表示内心活动。

```json
"description": "（我攥紧书包带子）……算了，想再多也没用。既然来了，就没有回头路了。"
```

括号内是内心活动，括号外可能包含动作或对话。

### 感官描写优先级

蒙眼状态下，视觉信息应被**听觉、触觉、嗅觉**取代：

```json
// 蒙眼前的视觉描写
"description": "（走近了。两台并排的机器嗡嗡低鸣，饮料瓶在玻璃后反射着光。）"

// 蒙眼后的触觉/听觉描写  
"description": "（眼睛被遮住之后，其他的感官变得异常敏锐。我能听到自己踩在砂石路上的脚步声——每一步都像是被放大了。夜风穿过树梢的声音听起来很近，但又分不清是从哪个方向来的。）"
```

### 渐进式堕落曲线

叙事节奏建议：

1. **犹豫期**（任务前期）：犹豫、害怕、自我说服
2. **行动期**（中期）：逐一执行计划步骤，心理防线逐步退让
3. **突破期**（高潮前）：接受现实，停止抵抗
4. **高潮期**：感官淹没，自我认同崩塌
5. **余韵期**（结尾）：回甘、接受新的自我认知

### 字数控制

- 每段描述建议 **≤ 120 字（中文）**，否则游戏内对话窗口可能显示不完整或字体过小
- 较长的描述应拆分为多个连续的 CP（同 zone 连续纯描述 CP 会自动播放）
- 常用拆分点：动作停顿、心理转折、场景切换

### Callback 技巧

前期埋下的细节在后期呼应，增强叙事一致性：

```json
// CP1: "今晚——我是真空出门的，裙摆下面什么都没有穿。"
// CP19（贩卖机前）: "冷柜的灯光照在膝盖上方的皮肤上——没有布料的遮挡，风直接从裙摆下面灌上来，凉飕飕的。"
// CP45（被铐住后）: "我喘着气，在黑暗中感受着这个姿势。"
```

---

## 12. 常见问题与陷阱

### ❌ `travelcondition` 导致卡死

不要在 CP 中使用未知的 `travelcondition` 字段。该字段不可靠且会导致 CP 无法触发。如果需要在 zone 之间导航，使用叙事文本中的**方向指引**即可。

### ❌ 兜底 Zone 导致提前触发

`parkArea` 覆盖整个公园地图。如果大量 CP 使用 `parkArea`，玩家可能在非预期位置触发对话。应使用具体的 zone（`fountain`、`garden`、`tree` 等）。

### ❌ 条件不匹配导致 CP 卡住

检查点不触发时，排查顺序：
1. 玩家是否在正确的 zone？
2. condition 是否拼写正确？（区分大小写）
3. condition 中的状态是否被其它状态阻塞？
4. itemconditions 检测的物品是否在正确 zone？

### ❌ 手铐流程顺序错误

壁铐流程中，**必须先丢弃钥匙再铐手铐**：
```
✅ 正确：丢钥匙 → Action_HandcuffsAtMap
❌ 错误：Action_HandcuffsAtMap → 丢钥匙（叙事矛盾——已经被铐住了怎么丢？）
```

### ❌ 多个 Action 条件冲突

一个 CP 只能有一个 `condition` 字段。如果需要检测复合条件，使用 `[条件1,条件2]` 格式。不要使用多个独立的 `condition`。

### ❌ JSON 格式错误

常见错误：
- 最后一个 CP 后多余的逗号
- 字符串引号不匹配
- 缩进对齐问题
- 中文字符使用了全角引号

建议编辑后用 JSON 验证工具检查。

### ❌ 描述文本过长

单段文本超过 120 字可能导致：
- 游戏内对话窗口滚动条出现
- 字体自动缩小影响阅读
- 文本被截断

**解决方案**：拆分为多个连续 CP（同 zone 无条件的 CP 会自动顺序播放）。

### ❌ 传送后状态残留

`setPlayerPosition` 不会重置玩家状态。传送前应在 oncomplete 中手动：
1. `setVibrator` → `Off`
2. `unequipAdultToy` → 卸下所有玩具
3. 根据需要 `unequipAllCosplay` 或 `equipCosplay`

### ❌ `dropItem` 的 `stage: "Apart"` 缺少坐标参数

清理物品时，`dropItem` 的 `stage: "Apart"` **必须显式填写** `"x": 0, "y": 0, "z": 0`：

```json
// ✅ 正确
{ "type": "dropItem", "itemtype": "HandcuffKey", "stage": "Apart", "x": 0, "y": 0, "z": 0 }

// ❌ 错误——会静默失败，zone 圈圈消失
{ "type": "dropItem", "itemtype": "HandcuffKey", "stage": "Apart" }
```

漏写坐标**不会报错**，但会导致游戏无法正确解析任务文件（表现为 zone 地面圈圈不渲染）。这是最隐蔽的坑，因为 JSON 验证也是通过的。

### ❌ 多个 Cleanup 操作合并在同一个 Oncomplete

`dropItem`、`unlockHandcuffs`、`setAction` 等多个 cleanup 操作不能放在同一个 CP 的 `oncomplete` 里。必须拆成独立的 CP，每个 CP 只做一件事：

```json
// ✅ 正确——每个 cleanup 独立 CP
CP4: { "description": "解除手铐", "oncomplete": [{ "type": "unlockHandcuffs" }] }
CP5: { "description": "重置动作", "oncomplete": [{ "type": "setAction", "action": "None" }] }

// ❌ 错误——合并在一个 oncomplete 里会导致 zone 圈圈消失
{ "oncomplete": [
  { "type": "unlockHandcuffs" },
  { "type": "setAction", "action": "None" }
]}
```

`dropItem`、`unlockHandcuffs`、`setAction` 这些写操作各自独占一个 CP。但 `setVibrator` + `unequipAdultToy` + `setPlayerPosition` 这类"结束状态重置"可以放在同一个 oncomplete 中（参考 moonlight CP61）。

---

## 13. 完整任务参考

### 「夜の放課後 —— 月光下的独白」结构总览

```
CP0:   teleport - 状态重置、初始化（SubCondition + oncomplete 全量设置）
      [传送 → 公园入口]
CP1:   start - 场景建立：交代真空出门
CP2-3: start - 犹豫、自我说服
CP4-5: start - 迈出第一步，导航指引→喷泉
      ── 导航：start → fountain ──
CP6-9: fountain - 喷泉场景描写、心理挣扎
      ── 导航：fountain → garden ──
CP10-13: garden - 花坛、Dogeza 屈服、继续前行
      ── 导航：garden → vending ──
CP14-22: vending - 买饮料、喝水、导航→树
      ── 导航：vending → tree ──
CP23-34: tree - 树下描写、捡手铐钥匙、导航→公厕
      ── 导航：tree → toilet ──
CP35-39: toilet - 蒙眼、走进公厕
      ── 导航：toilet → urinal ──
CP40-46: urinal - 找到手铐、丢钥匙、壁铐
CP47-53: urinal - 性爱描写、背后式
CP54-57: urinal - 高潮
CP58-61: urinal - 失神→发现手铐松了→犹豫→空闲检测→传送
      ── 传送回公园入口 ──
CP62-69: start - 裸身到达→穿衣服→长椅→Dogeza→幻想→离开
```

总共 **9 个 zone**、**69 个 CP**。

### 官方任务参考

| 任务 | CP 数 | 特点 |
|---|---|---|
| blackmail.json | 48 | 基础模式参考，Dogeza/贩卖机/振动器/穿脱衣 |
| blackmail2.json | 40 | 手铐链模式（KeyedHandcuffs + !KeyedHandcuffs） |
| blackmail3.json | 48 | 扩展条件（复合条件、setAction、unlockHandcuffs） |

建议开发新任务时以 `blackmail.json` 为起点，逐步增加复杂性。

---

> 本文档基于 v1.1.1 版本的逆向工程编写。未知的条件/动作类型需要通过游戏内调试工具（如 Cheat Engine、Cinematic Unity Explorer）进一步发掘。
