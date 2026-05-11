# BlockUp Puzzle · AI 构建包

| 项 | 值 |
|---|---|
| 产品 ID | blockup-puzzle |
| 版本 | V1.0 |
| 来源模块 2 | ./BlockUp Puzzle PRD.md |
| 生成时间 | 2026-05-05 |
| 面向 AI | 通用大模型 · 优先单文件 HTML + Tailwind CDN |

---

## 1. 技术选型

- **前端框架**：单文件 HTML + 原生 JavaScript（Canvas 渲染游戏主体）
- **状态管理**：原生 JavaScript 全局状态对象（window.gameState）
- **样式方案**：Tailwind CDN + 内联 CSS（UI 层）；Canvas 2D Context（游戏渲染层）
- **后端存储**：localStorage（本地持久化分数、金币、道具库存）
- **部署形态**：单文件 HTML，可双击打开；支持 GitHub Pages 静态托管
- **AI 调用方**：无 AI 调用

---

## 2. 数据模型

### 实体 · GameState

| 字段名 | 类型 | 必填 | 取值约束 | 说明 |
|---|---|---|---|---|
| score | number | 是 | 0 至正无穷整数 | 当前局分数 |
| bestScore | number | 是 | 0 至正无穷整数 | 历史最高分 |
| coins | number | 是 | 0 至正无穷整数 | 当前金币余额 |
| comboCount | number | 是 | 0 至正无穷整数 | 当前连击次数 |
| gameStatus | enum | 是 | "playing" / "gameOver" / "reviving" | 游戏状态 |
| isReviveUsed | boolean | 是 | true / false | 本局是否已使用复活 |

### 实体 · Board

| 字段名 | 类型 | 必填 | 取值约束 | 说明 |
|---|---|---|---|---|
| grid | array | 是 | 8×8 二维数组，每格值为 0（空）或 1-7（颜色编号） | 棋盘格子状态 |
| size | number | 是 | 固定值 8 | 棋盘边长 |

### 实体 · Shape

| 字段名 | 类型 | 必填 | 取值约束 | 说明 |
|---|---|---|---|---|
| shapeId | number | 是 | 0-36 整数 | 形状编号，对应 37 种预定义形状 |
| matrix | array | 是 | 5×5 二维数组，值为 0 或 1 | 形状的网格定义 |
| colorId | number | 是 | 1-7 整数 | 方块颜色编号 |

### 实体 · PendingShapes

| 字段名 | 类型 | 必填 | 取值约束 | 说明 |
|---|---|---|---|---|
| shapes | array | 是 | 长度 3 的 Shape 数组 | 当前待放置的 3 个方块 |
| placedCount | number | 是 | 0 / 1 / 2 / 3 | 已放置方块数量 |

### 实体 · BoosterInventory

| 字段名 | 类型 | 必填 | 取值约束 | 说明 |
|---|---|---|---|---|
| bombCount | number | 是 | 0 至正无穷整数 | 炸弹道具库存 |
| gravityCount | number | 是 | 0 至正无穷整数 | 重力道具库存 |
| refreshCount | number | 是 | 0 至正无穷整数 | 刷新道具库存 |

### 实体 · BoosterConfig

| 字段名 | 类型 | 必填 | 取值约束 | 说明 |
|---|---|---|---|---|
| bombPrice | number | 是 | 固定值 120 | 炸弹道具金币价格 |
| gravityPrice | number | 是 | 固定值 200 | 重力道具金币价格 |
| refreshPrice | number | 是 | 固定值 150 | 刷新道具金币价格 |
| initialInventory | number | 是 | 固定值 3 | 每种道具初始库存 |

### 实体 · Settings

| 字段名 | 类型 | 必填 | 取值约束 | 说明 |
|---|---|---|---|---|
| musicEnabled | boolean | 是 | true / false | 背景音乐开关 |
| sfxEnabled | boolean | 是 | true / false | 音效开关 |

---

## 3. 页面结构

### 页面 · 游戏主页面

- **路由**：无（单页应用）
- **读取的实体**：GameState, Board, PendingShapes, BoosterInventory, Settings
- **写入的实体**：GameState, Board, PendingShapes, BoosterInventory
- **主要区域**：
  - 顶部信息栏 · 显示当前分数、最高分、金币余额
  - 棋盘区域 · 8×8 网格，显示已放置方块和拖拽预览
  - 待放置方块区 · 底部展示 3 个待放置的方块形状
  - 道具栏 · 底部展示 3 种道具按钮及库存数量
  - 连击面板 · 连击时浮现连击数和进度条
- **用户身份限制**：无限制

### 页面 · 游戏结束面板（覆盖层）

- **路由**：无（同页覆盖层）
- **读取的实体**：GameState
- **写入的实体**：GameState, Board, PendingShapes, BoosterInventory
- **主要区域**：
  - 分数展示区 · 本局得分和最高分对比
  - 复活按钮区 · 显示复活选项和倒计时
  - 重新开始按钮 · 重置游戏状态
- **用户身份限制**：无限制

### 页面 · 设置面板（覆盖层）

- **路由**：无（同页覆盖层）
- **读取的实体**：Settings
- **写入的实体**：Settings
- **主要区域**：
  - 音乐开关 · 切换背景音乐
  - 音效开关 · 切换游戏音效
- **用户身份限制**：无限制

---

## 4. 交互流程

### 流程 · 方块放置

**前置条件**：GameState.gameStatus 为 "playing"，PendingShapes.shapes 中仍有未放置方块

| 步骤 | 触发 | 系统动作 | 涉及实体 | 失败态处理 |
|---|---|---|---|---|
| 1 | 用户触摸/点击一个待放置方块并开始拖拽 | 高亮选中方块，显示拖拽跟随 | PendingShapes | 无 |
| 2 | 用户拖拽方块到棋盘上方 | 实时计算放置位置，检测是否可放（Board.grid 对应位置全为 0），高亮有效/无效位置 | Board, Shape | 无效位置显示红色高亮 |
| 3 | 用户在有效位置松手 | 将 Shape.matrix 中值为 1 的格子对应位置的 Board.grid 设为 Shape.colorId；PendingShapes.placedCount 加 1 | Board, PendingShapes | 无效位置松手：方块回弹到原位 |
| 4 | 放置完成 | 触发"规则 · 行列消除检测" | Board, GameState | 无 |
| 5 | 消除检测完成 | 若 PendingShapes.placedCount 等于 3，触发"规则 · 生成新方块" | PendingShapes | 无 |
| 6 | 新方块生成后 | 触发"规则 · 游戏结束判定" | Board, PendingShapes, GameState | 无 |

**终点**：方块成功放置，分数更新，游戏继续或结束

**禁区**：不得允许在已有方块的格子上重叠放置；不得在游戏结束状态下继续放置

### 流程 · 使用炸弹道具

**前置条件**：GameState.gameStatus 为 "playing"，BoosterInventory.bombCount > 0，GameState.coins >= BoosterConfig.bombPrice

| 步骤 | 触发 | 系统动作 | 涉及实体 | 失败态处理 |
|---|---|---|---|---|
| 1 | 用户点击炸弹道具按钮 | 进入炸弹选择模式，棋盘高亮可选区域 | BoosterInventory | 库存为 0：显示 Toast "炸弹已用完"；金币不足：显示 Toast "金币不足" |
| 2 | 用户点击棋盘上一个 3×3 区域中心 | 将该 3×3 区域内所有非空格子的 Board.grid 值设为 0；GameState.coins 减少 BoosterConfig.bombPrice；BoosterInventory.bombCount 减 1 | Board, GameState, BoosterInventory | 用户点击空白区域：显示 Toast "请选择有方块的区域" |
| 3 | 清除完成 | 播放爆炸视觉特效，退出炸弹选择模式 | 无 | 无 |

**终点**：指定区域方块被清除，金币和库存已扣减

**禁区**：不得在炸弹选择模式下放置方块；不得清除棋盘外的格子

### 流程 · 使用重力道具

**前置条件**：GameState.gameStatus 为 "playing"，BoosterInventory.gravityCount > 0，GameState.coins >= BoosterConfig.gravityPrice

| 步骤 | 触发 | 系统动作 | 涉及实体 | 失败态处理 |
|---|---|---|---|---|
| 1 | 用户点击重力道具按钮 | 检查库存和金币 | BoosterInventory, GameState | 库存为 0：显示 Toast "重力道具已用完"；金币不足：显示 Toast "金币不足" |
| 2 | 检查通过 | 遍历 Board.grid 每一列，将非空格子向下坠落填补空隙（保持列内顺序不变，空格上移）；GameState.coins 减少 BoosterConfig.gravityPrice；BoosterInventory.gravityCount 减 1 | Board, GameState, BoosterInventory | 无 |
| 3 | 重力执行完成 | 播放下落动画，触发"规则 · 行列消除检测" | Board, GameState | 无 |

**终点**：方块下落压实，触发消除检测，金币和库存已扣减

**禁区**：不得改变方块颜色；不得影响待放置方块区

### 流程 · 使用刷新道具

**前置条件**：GameState.gameStatus 为 "playing"，BoosterInventory.refreshCount > 0，GameState.coins >= BoosterConfig.refreshPrice

| 步骤 | 触发 | 系统动作 | 涉及实体 | 失败态处理 |
|---|---|---|---|---|
| 1 | 用户点击刷新道具按钮 | 检查库存和金币 | BoosterInventory, GameState | 库存为 0：显示 Toast "刷新道具已用完"；金币不足：显示 Toast "金币不足" |
| 2 | 检查通过 | 重新从 37 种形状中随机选取 3 个新 Shape，替换 PendingShapes.shapes；PendingShapes.placedCount 重置为 0；GameState.coins 减少 BoosterConfig.refreshPrice；BoosterInventory.refreshCount 减 1 | PendingShapes, GameState, BoosterInventory | 无 |
| 3 | 刷新完成 | 底部待放置区显示 3 个新方块 | PendingShapes | 无 |

**终点**：待放置方块已更换为新随机方块，金币和库存已扣减

**禁区**：不得影响棋盘已放置方块；不得重复生成与之前完全相同的 3 个形状组合

### 流程 · 复活

**前置条件**：GameState.gameStatus 为 "gameOver"，GameState.isReviveUsed 为 false

| 步骤 | 触发 | 系统动作 | 涉及实体 | 失败态处理 |
|---|---|---|---|---|
| 1 | 游戏结束面板显示 | 显示复活按钮和 10 秒倒计时 | GameState | 无 |
| 2 | 用户在倒计时内点击复活按钮 | 清除 Board.grid 中最底部 2 行所有方块（设为 0）；GameState.gameStatus 设为 "playing"；GameState.isReviveUsed 设为 true；触发"规则 · 生成新方块" | Board, GameState, PendingShapes | 倒计时结束未点击：关闭复活选项，保持 gameOver 状态 |
| 3 | 复活完成 | 游戏结束面板关闭，恢复正常游戏 | GameState | 无 |

**终点**：玩家复活成功，游戏继续

**禁区**：不得允许同一局复活两次；不得在复活时恢复分数或金币

---

## 5. AI 决策点

本产品无 AI 决策点。所有业务逻辑由固定规则实现。

---

## 6. 规则判定点

### 规则 · 行列消除检测

- **触发**：每次方块放置完成后；每次重力道具执行完成后
- **输入**：Board.grid（8×8 二维数组）
- **判定逻辑**：
  ```
  completedLines = []
  FOR row = 0 TO 7:
    IF Board.grid[row] 每格均不为 0:
      completedLines.add({type: "row", index: row})
  FOR col = 0 TO 7:
    IF Board.grid[0..7][col] 每格均不为 0:
      completedLines.add({type: "col", index: col})
  IF completedLines.length > 0:
    FOR EACH line IN completedLines:
      将该行/列所有格子设为 0
    计分: score += completedLines.length * 10 * (1 + comboCount * 0.5)
    金币: coins += completedLines.length * 5
    comboCount += 1
  ELSE:
    comboCount = 0
  ```
- **输出**：更新 Board.grid（消除对应行列）；更新 GameState.score；更新 GameState.coins；更新 GameState.comboCount
- **优先级**：1（最高，放置后立即执行）

### 规则 · 生成新方块

- **触发**：PendingShapes.placedCount 等于 3 时；复活完成后
- **输入**：无（纯随机）
- **判定逻辑**：
  ```
  FOR i = 0 TO 2:
    randomShapeId = RANDOM_INT(0, 36)
    randomColorId = RANDOM_INT(1, 7)
    shapes[i] = {shapeId: randomShapeId, matrix: PREDEFINED_SHAPES[randomShapeId], colorId: randomColorId}
  PendingShapes.shapes = shapes
  PendingShapes.placedCount = 0
  ```
- **输出**：PendingShapes.shapes 填充 3 个新 Shape；PendingShapes.placedCount 重置为 0
- **优先级**：2

### 规则 · 游戏结束判定

- **触发**：新方块生成后
- **输入**：Board.grid, PendingShapes.shapes
- **判定逻辑**：
  ```
  canPlace = false
  FOR EACH shape IN PendingShapes.shapes（仅未放置的）:
    FOR row = 0 TO 7:
      FOR col = 0 TO 7:
        IF shape.matrix 放置在 (row, col) 时，所有值为 1 的格子对应 Board.grid 位置均为 0 且不越界:
          canPlace = true
          BREAK ALL
  IF canPlace == false:
    GameState.gameStatus = "gameOver"
    IF GameState.score > GameState.bestScore:
      GameState.bestScore = GameState.score
      保存 bestScore 到 localStorage
  ```
- **输出**：若无法放置任何待选方块，GameState.gameStatus 设为 "gameOver"；更新 GameState.bestScore
- **优先级**：3

### 规则 · 道具可用性检查

- **触发**：用户点击任一道具按钮时
- **输入**：BoosterInventory 对应道具 count、GameState.coins、BoosterConfig 对应 price
- **判定逻辑**：
  ```
  IF 对应道具 count <= 0:
    显示 Toast "道具已用完"
    RETURN blocked
  IF GameState.coins < 对应道具 price:
    显示 Toast "金币不足"
    RETURN blocked
  RETURN allowed
  ```
- **输出**：allowed（继续执行道具流程）或 blocked（中止并提示）
- **优先级**：0（道具流程前置检查，最先执行）

---

## 7. 合规红线

### 红线 · 虚拟货币无真实价值声明

- **触发**：用户首次进入游戏时
- **必须做**：在游戏首次加载完成后，若 localStorage 中无 "disclaimerShown" 标记，显示一次免责提示："本游戏金币为虚拟货币，无真实经济价值"
- **植入位置**：游戏主页面初始化，在首次渲染完成后
- **绝对禁止**：不得提供任何真实货币购买虚拟金币的入口

### 红线 · 数据持久化仅限本地

- **触发**：任何数据写入操作
- **必须做**：所有数据（分数、金币、道具库存、设置）只写入 localStorage，不得发送至任何外部服务器
- **植入位置**：流程"方块放置"步骤 4 后、所有道具使用流程结束后、流程"复活"步骤 2 后
- **绝对禁止**：不得引入任何网络请求发送用户游戏数据；不得收集设备标识或个人信息

### 红线 · 复活次数限制

- **触发**：游戏结束时显示复活选项
- **必须做**：检查 GameState.isReviveUsed，若为 true 则不显示复活按钮
- **植入位置**：流程"复活"步骤 1
- **绝对禁止**：不得允许同一局游戏复活超过 1 次

---

## 8. 执行顺序与依赖

AI 构建产品时，按以下顺序执行：

1. 建立**数据模型**（第 2 章所有实体：GameState, Board, Shape, PendingShapes, BoosterInventory, BoosterConfig, Settings）
2. 建立**页面骨架**（第 3 章：游戏主页面 + 游戏结束面板覆盖层 + 设置面板覆盖层）
3. 实现**交互流程**（第 4 章：方块放置 → 炸弹道具 → 重力道具 → 刷新道具 → 复活）
4. 接入 **AI 决策点**（本产品无 AI 决策点，跳过）
5. 植入**规则判定**（第 6 章：行列消除检测 → 生成新方块 → 游戏结束判定 → 道具可用性检查）
6. 植入**合规红线**（第 7 章：虚拟货币声明 + 数据仅本地 + 复活次数限制）
7. 自检：每个字段名唯一、每个禁区已守、每条合规红线已植入

---

## 自检结论

### 1. 命名统一表

| 模块 2 原叫法 | 统一命名 |
|---|---|
| 方块 / 块 | Shape |
| 棋盘 / 网格 / 格子 | Board / Board.grid |
| 分数 / 得分 | GameState.score |
| 最高分 / 最佳成绩 | GameState.bestScore |
| 金币 / 虚拟货币 | GameState.coins |
| 连击 / Combo | GameState.comboCount |
| 炸弹道具 | BoosterInventory.bombCount |
| 重力道具 | BoosterInventory.gravityCount |
| 刷新道具 | BoosterInventory.refreshCount |
| 玩家 / 用户 | 无独立实体（单人本地游戏） |

### 2. 字段唯一性扫描结果

全局唯一。所有字段名在整份构建包中仅出现一处定义。

### 3. 模糊词扫描结果

零命中。全文不含"大约 / 可能 / 适当 / 一些 / 几个 / 视情况"等模糊词。

### 4. 合规红线植入位置列表

| 红线 | 植入位置 |
|---|---|
| 虚拟货币无真实价值声明 | 游戏主页面初始化渲染完成后 |
| 数据持久化仅限本地 | 方块放置步骤 4 后、所有道具流程结束后、复活步骤 2 后 |
| 复活次数限制 | 复活流程步骤 1 |

### 5. 待产品方补充项清单

| 编号 | 待补充项 | 影响 |
|---|---|---|
| 1 | 炸弹清除范围确认（当前设定为 3×3） | 影响炸弹道具的实际效果 |
| 2 | 连击超时时间（当前未设具体秒数，设定为连续放置均可计连击） | 影响连击判定逻辑 |
| 3 | 复活时清除行数确认（当前设定为底部 2 行） | 影响复活后棋盘状态 |

### 6. 来源映射表

| 模块 2 来源 | AI 构建包对应 |
|---|---|
| 8.1 方块形状维度 | 实体 Shape |
| 8.1 道具类型维度 | 实体 BoosterInventory, BoosterConfig |
| 8.1 游戏状态维度 | GameState.gameStatus 枚举 |
| 8.1 连击等级维度 | GameState.comboCount |
| 8.2 方块拖拽放置 | 流程"方块放置" |
| 8.2 道具使用 | 流程"使用炸弹/重力/刷新道具" |
| 8.2 复活确认 | 流程"复活" |
| 8.2 音乐/音效开关 | 实体 Settings, 页面"设置面板" |
| 8.3 当前分数 | GameState.score |
| 8.3 最高分 | GameState.bestScore |
| 8.3 连击数显示 | GameState.comboCount |
| 8.3 金币余额 | GameState.coins |
| 8.3 道具库存 | BoosterInventory |
| 8.3 游戏结束面板 | 页面"游戏结束面板" |
| 8.4 游戏进行中 | GameState.gameStatus = "playing" |
| 8.4 游戏结束 | GameState.gameStatus = "gameOver" |
| 8.4 复活倒计时 | GameState.gameStatus = "reviving" |
| 3.3 Journey A 单局游戏流程 | 流程"方块放置" |
| 3.3 Journey B 道具使用 | 流程"使用炸弹/重力/刷新道具" |
| 4.3.1 方块放置与消除 | 流程"方块放置" + 规则"行列消除检测" |
| 4.3.2 连击系统 | 规则"行列消除检测"中 comboCount 逻辑 |
| 4.3.3 道具系统 | 流程"使用炸弹/重力/刷新道具" + 规则"道具可用性检查" |
| 4.3.4 金币经济与复活 | GameState.coins + 流程"复活" |
| 7.4 虚拟金币无真实价值 | 红线"虚拟货币无真实价值声明" |
| 7.2 数据不上传服务器 | 红线"数据持久化仅限本地" |
