# 插件指令与 ConVar 参考

本文档由 SourcePawn 插件源码机械式提取生成，覆盖本服务器自定义插件所注册的全部**指令**与**ConVar**，并按提供它们的插件归类。

## 扫描方式与统计

- 扫描根目录：`/home/steam/l4d2/left4dead2/addons/sourcemod/scripting/`
- 覆盖范围：顶层 `*.sp`、`extend/`、`optional/`、`confoglcompmod/`、`anticheat/`、`duoren/`、`fixes/`（均递归）
- 检索方式：先用 grep 做穷举式检索确认没有遗漏，再用脚本解析每个调用的实参（保留字符串、跨行续行、宏名）：
- 除 `CreateConVar` 外还跟踪源码中的辅助创建函数：`CreateConVarEx`（confogl，名称自动加 `confogl_` 前缀）、`CreateConVarHook`、`getOrCreateLegacyConVar`；由宏（如 `CONVAR_VERSION_NAME`）或运行时拼接（如 `PLUGIN_NAME + "_enable"`）得到的动态名称已按源码取值解析为实际名称。

```bash
grep -rn -E "Reg(Console|Admin|Server)Cmd\(|CreateConVar\(|HookConVarChange\(|AutoExecConfig\(" --include="*.sp"
     . extend optional confoglcompmod anticheat duoren fixes
grep -rhoE "Reg[A-Za-z_]*Cmd[A-Za-z_]*\(" --include="*.sp" . extend optional confoglcompmod anticheat duoren fixes | sort | uniq -c
```

结果：该范围内共出现 `RegConsoleCmd` / `RegAdminCmd` / `RegServerCmd` / `CreateConVar` 等注册调用；另有 2 处自定义注册函数 `RegisterCmds()` 已人工核对。

| 项目 | 数量 |
|---|---|
| 扫描的 .sp 源文件 | 419 |
| 收录插件（以 myinfo name 为单位聚合的根文件） | 384 |
| 收录指令 | 485 条（RegConsoleCmd 254，RegAdminCmd 182，RegServerCmd 49） |
| 其中唯一指令名 | 429 个（同一名称可能被多个插件重复注册，如 `sm_bonus`、`sm_tank`） |
| 收录 ConVar | 1623 条 |
| 其中经辅助函数间接创建（CreateConVarEx 63，CreateConVarHook 26，getOrCreateLegacyConVar 9） | 98 条 |
| 其中唯一 ConVar 名 | 1547 个（另有 11 条为包装函数内部的占位调用，已从清单中剔除） |
| HookConVarChange 变更钩子 | 150 处 |
| AutoExecConfig 自动配置声明 | 28 处 |

### 未纳入本文档的目录（透明说明）

任务给定的覆盖范围只包含上面列出的目录。以下目录**不在**本文档收录范围内（其中 `sourcemod/` 是 SourceMod 官方自带插件源码，`sm_kick`、`sm_ban`、`sm_map`、`sm_cvar` 等基础指令来自那里，请勿在本文档中查找）：

| 目录 | .sp 数量 | 说明 |
|---|---|---|
| `archive/` | 85 | 历史归档，按任务要求只作参考 |
| `disabled/` | 16 | 已禁用插件，按任务要求只作参考 |
| `sourcemod/` | 85 | SourceMod 官方插件源码（基础指令与 ConVar 的来源） |
| `include/` | 0 | 头文件，仅用于理解语义 |
| `lilac/` | 16 | 反作弊引擎组件，非独立注册插件 |
| `l4dd/` | 5 | left4dhooks 的内部包含文件（已计入 left4dhooks） |
| `vector/` | 1 | 测试辅助源码 |
| `dev_plugins/` | 1 | 开发测试插件 |
| `advertisements/` | 2 | 广告插件的配置/子模块 |
| `sbpp_admcfg/` | 2 | SourceBans++ 管理配置子模块 |
| `gamedata_backup/` | 0 | gamedata 备份，无源码 |

### 字段与推断规则

- **权限**：直接取自 `RegAdminCmd` 的第三参数（`ADMFLAG_*`），`RegConsoleCmd` 默认无管理员 flag，`RegServerCmd` 为服务器控制台命令。
- **语法/参数**：来自源码注册时给出的用法字符串与回调内 `GetCmdArg*` 的取参情况；源码未给用法时，按回调内 `GetCmdArg`/`GetCmdArgCount` 的实际取参情况推断，仍无法判断则明确给出推测与置信度，不再以“未给说明”一句收尾。
- **功能**：优先取自源码中注册描述或回调内的提示文本；凡源码描述为空、仅占位（如 `description`、`desc`）或语义不明处，均回到源码分析（回调函数体、所调用的 native/SDKCall、读写的数据、相近命名的 ConVar、翻译文本）后给出「推测」，并在同一单元格内给出证据位置与置信度（高/中/低）。
- **类型**：SourceMod 的 `CreateConVar` 不显式声明类型，下表按默认值与上下界的字面形式推断：全为整数记「整数」；取值恰为 0/1 记「开关」；含小数记「浮点」；非数值字面量（宏/变量）记「字符串或表达式」。
- **取值范围**：来自 `CreateConVar` 第 5、7 参数（`hasMin` / `hasMax`）及其后的 min/max 数值；未提供上下界记「无上下界」。
- **影响范围**：由该 ConVar 的创建标记（`FCVAR_*`）推出其生效范围（服务器端、是否复制到客户端、是否需 sv_cheats、是否写入 cfg 等）；其具体影响的行为见「作用」列。
- **来源**：`文件:行号`，行号为该注册调用在 .sp 文件中的起始行，便于回查源码。
- 同一 ConVar 名或指令名可能被多个插件重复注册（本仓库的 `optional/` 与顶层存在同源插件副本），本文档逐条列出，不做去重。

---

## L4D2 Admin Mission Menu —— `adminmenu_mission_list.sp`

- 源文件：`adminmenu_mission_list.sp`
- myinfo：author=HoongDou
- myinfo description（源码原文）：Adds a 'Switch Map/Mission' item to the admin menu for direct map changes.

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_vpk_reload` | 无参数 | 需要 ADMFLAG_RCON（i，RCON 命令） | 重新加载 VPK 与任务列表，并执行 update_addon_paths 与 mission_reload | `adminmenu_mission_list.sp:112` |
| `sm_adminmap_gentrans` | 无参数 | 需要 ADMFLAG_RCON（i，RCON 命令） | 强制重新生成地图翻译文件 | `adminmenu_mission_list.sp:113` |

### ConVar

（本插件未注册 ConVar）

## Simple Anti-Bunnyhop —— `anticheat/l4d2_nobhaps.sp`

- 源文件：`anticheat/l4d2_nobhaps.sp`
- myinfo：version=0.5.1，author=CanadaRox, ProdigySim, blodia, CircleSquared, robex, A1m`
- myinfo description（源码原文）：Stops bunnyhops by restricting speed when a player lands on the ground to their MaxSpeed

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_check_bhop` | 无参数 | 无（任意玩家可用） | 在聊天中报告当前是否允许连跳以及按特感职业的豁免情况 | `l4d2_nobhaps.sp:85` |

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `simple_antibhop_enable` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 总开关：0 关闭 Simple Anti-Bhop，1 启用 | 服务器端本插件逻辑 | `l4d2_nobhaps.sp:53` |
| `bhop_except_si_flags` | `0` | 整数，源码写作浮点 | 0.0 ~ 127.0 | 豁免禁跳的特感位掩码：1 smoker，2 boomer，4 hunter，8 spitter，16 jockey，32 charger，64 tank | 服务器端本插件逻辑 | `l4d2_nobhaps.sp:64` |
| `bhop_allow_survivor` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否允许生还者连跳：1 允许，0 禁止 | 服务器端本插件逻辑 | `l4d2_nobhaps.sp:72` |

## L4D2 Ghost-Cheat Preventer —— `anticheat/l4d2_noghostcheat.sp`

- 源文件：`anticheat/l4d2_noghostcheat.sp`
- myinfo：version=1.0，author=Sir
- myinfo description（源码原文）：Don't broadcast Infected entities to Survivors while in ghost mode, disabling them from hooking onto the entities with 3rd party programs.

### 指令

（本插件未注册指令）

### ConVar

（本插件未注册 ConVar）

## chat-processor.sp（myinfo 缺 name，用文件名代替） —— `chat-processor.sp`

- 源文件：`chat-processor.sp`
- myinfo：源码中未找到 myinfo 块

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `sm_chatprocessor_version` | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | 服务器端本插件逻辑；复制到客户端；仅服务器；不写入 cfg 存档 | `chat-processor.sp:114` |
| `sm_chatprocessor_status` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 插件状态显示开关 | 服务器端本插件逻辑 | `chat-processor.sp:116` |
| `sm_chatprocessor_config` | `configs/chat_processor.cfg` | 字符串或表达式 | 无上下界 | 消息格式配置文件路径 | 服务器端本插件逻辑 | `chat-processor.sp:117` |
| `sm_chatprocessor_process_colors_default` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否默认给转发处理颜色 | 服务器端本插件逻辑 | `chat-processor.sp:118` |
| `sm_chatprocessor_remove_colors_default` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否默认给转发移除颜色 | 服务器端本插件逻辑 | `chat-processor.sp:119` |
| `sm_chatprocessor_strip_colors` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 输出前是否先移除名字与消息中的颜色标签 | 服务器端本插件逻辑 | `chat-processor.sp:120` |
| `sm_chatprocessor_colors_flag` | `b` | 字符串或表达式 | 无上下界 | 使用彩色名字与消息所需的权限 flag，需 strip_colors 为 1 | 服务器端本插件逻辑 | `chat-processor.sp:121` |
| `sm_chatprocessor_deadchat` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 死者聊天开关：0 关，1 开 | 服务器端本插件逻辑 | `chat-processor.sp:122` |
| `sm_chatprocessor_allchat` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否允许双方通过团队聊天互相交流：0 关，1 开 | 服务器端本插件逻辑 | `chat-processor.sp:123` |
| `sm_chatprocessor_restrictdeadchat` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否完全禁止死者的所有聊天：0 关，1 开 | 服务器端本插件逻辑 | `chat-processor.sp:124` |
| `sm_chatprocessor_addgotv` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否把 GOTV 客户端加入接收列表，仅对有 GOTV/SourceTV 的游戏生效 | 服务器端本插件逻辑 | `chat-processor.sp:125` |

## Confogl's Competitive Mod —— `confoglcompmod.sp`

- 源文件：`confoglcompmod/EntityRemover.sp`、`confoglcompmod/UnprohibitBosses.sp`、`confoglcompmod/CvarSettings.sp`、`confoglcompmod/MapInfo.sp`、`confoglcompmod/includes/configs.sp`、`confoglcompmod/includes/customtags.sp`、`confoglcompmod/ItemTracking.sp`、`confoglcompmod/PasswordSystem.sp`、`confoglcompmod/WeaponInformation.sp`、`confoglcompmod/ClientSettings.sp`、`confoglcompmod.sp`、`confoglcompmod/UnreserveLobby.sp`、`confoglcompmod/WeaponCustomization.sp`、`confoglcompmod/GhostTank.sp`、`confoglcompmod/GhostWarp.sp`、`confoglcompmod/includes/constants.sp`、`confoglcompmod/FinaleSpawn.sp`、`confoglcompmod/includes/survivorindex.sp`、`confoglcompmod/ReqMatch.sp`、`confoglcompmod/includes/functions.sp`、`confoglcompmod/BotKick.sp`、`confoglcompmod/BossSpawning.sp`、`confoglcompmod/ScoreMod.sp`、`confoglcompmod/includes/debug.sp`、`confoglcompmod/WaterSlowdown.sp`
- myinfo：author=Confogl Team, A1m`
- myinfo description（源码原文）：A competitive mod for L4D2

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `confogl_clientsettings` | 无参数 | 无（任意玩家可用） | 列出 confogl 强制跟踪的客户端 ConVar | `ClientSettings.sp:39` |
| `confogl_trackclientcvar` | <cvar> <hasMin> <min> [<hasMax> <max> [...]] | 服务器控制台命令，玩家无法使用 | 把某个客户端 ConVar 加入 confogl 的跟踪与强制列表，服务器控制台命令 | `ClientSettings.sp:42` |
| `confogl_resetclientcvars` | 无参数 | 服务器控制台命令，玩家无法使用 | 清除所有已跟踪的客户端 ConVar，比赛中使用会被拒绝，服务器控制台命令 | `ClientSettings.sp:43` |
| `confogl_startclientchecking` | 无参数 | 服务器控制台命令，玩家无法使用 | 开始检查并强制已跟踪的客户端 ConVar，服务器控制台命令 | `ClientSettings.sp:44` |
| `confogl_cvarsettings` | [0 或 1] | 无（任意玩家可用） | 列出 confogl 正在强制的所有 ConVar | `CvarSettings.sp:40` |
| `confogl_cvardiff` | [0 或 1] | 无（任意玩家可用） | 列出被改动过初始值的 ConVar | `CvarSettings.sp:41` |
| `confogl_addcvar` | <cvar> <newValue> | 服务器控制台命令，玩家无法使用 | 把某个 ConVar 加入由 Confogl 设置与强制的列表，服务器控制台命令 | `CvarSettings.sp:43` |
| `confogl_setcvars` | 无参数 | 服务器控制台命令，玩家无法使用 | 开始强制已添加的 ConVar，服务器控制台命令 | `CvarSettings.sp:44` |
| `confogl_resetcvars` | 无参数 | 服务器控制台命令，玩家无法使用 | 重置被强制的 ConVar，比赛中不可用，服务器控制台命令 | `CvarSettings.sp:45` |
| `confogl_erdata_reload` | 无参数 | 需要 ADMFLAG_CONFIG（h，服务器配置） | 重新加载 EntityRemoveData 配置 | `EntityRemover.sp:53` |
| `sm_warptosurvivor` | <生还者序号> | 无（任意玩家可用） | 把自己从幽灵状态传送到指定序号的生还者 | `GhostWarp.sp:33` |
| `confogl_midata_save` | 无参数 | 需要 ADMFLAG_CONFIG（h，服务器配置） | 把当前地图的 EntityRemoveData 配置保存下来 | `MapInfo.sp:43` |
| `confogl_save_location` | <位置名> | 需要 ADMFLAG_CONFIG（h，服务器配置） | 把当前所在坐标保存到地图配置 | `MapInfo.sp:44` |
| `sm_forcematch` | <cfg> [mode] | 需要 ADMFLAG_CONFIG（h，服务器配置） | 强制加载比赛模式 | `ReqMatch.sp:64` |
| `sm_fm` | <cfg> [mode] | 需要 ADMFLAG_CONFIG（h，服务器配置） | 强制加载比赛模式，sm_forcematch 的短别名 | `ReqMatch.sp:65` |
| `sm_resetmatch` | 无参数 | 需要 ADMFLAG_CONFIG（h，服务器配置） | 强制关闭比赛模式，无论此前是强制还是常开 | `ReqMatch.sp:66` |
| `sm_forcechangematch` | <cfg> | 需要 ADMFLAG_CONFIG（h，服务器配置） | 强制切换比赛配置 | `ReqMatch.sp:67` |
| `sm_fchmatch` | <cfg> | 需要 ADMFLAG_CONFIG（h，服务器配置） | 强制切换比赛配置，sm_forcechangematch 的短别名 | `ReqMatch.sp:68` |
| `sm_health` | 无参数 | 无（任意玩家可用） | 在聊天中报告当前生还者平均血量与回合奖励 | `ScoreMod.sp:95` |
| `sm_bonus` | 无参数 | 无（任意玩家可用） | 在聊天中报告当前回合奖励，sm_health 的别名 | `ScoreMod.sp:96` |
| `sm_killlobbyres` | 无参数 | 需要 ADMFLAG_BAN（d，封禁） | 清除大厅预留，使更多玩家可以加入 | `UnreserveLobby.sp:18` |

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `confogl_lock_boss_spawns` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 强制 Tank 与 Witch 在相同坐标刷新 | 服务器端本插件逻辑；经 CreateConVarEx 包装，实际名称自动加 confogl_ 前缀并附加 CVAR_FLAGS | `BossSpawning.sp:36` |
| `confogl_remove_parachutist` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否移除 c3m2 的跳伞员 | 服务器端本插件逻辑；经 CreateConVarEx 包装，实际名称自动加 confogl_ 前缀并附加 CVAR_FLAGS | `EntityRemover.sp:37` |
| `confogl_reduce_finalespawnrange` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否把终局特感刷新范围调整为普通刷新范围 | 服务器端本插件逻辑；经 CreateConVarEx 包装，实际名称自动加 confogl_ 前缀并附加 CVAR_FLAGS | `FinaleSpawn.sp:19` |
| `confogl_remove_escape_tank` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 终局救援载具到来时刷出的 Tank 是否移除 | 服务器端本插件逻辑；经 CreateConVarEx 包装，实际名称自动加 confogl_ 前缀并附加 CVAR_FLAGS | `GhostTank.sp:45` |
| `confogl_disable_tank_hordes` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 坦克在场时是否禁止自然尸潮 | 服务器端本插件逻辑；经 CreateConVarEx 包装，实际名称自动加 confogl_ 前缀并附加 CVAR_FLAGS | `GhostTank.sp:46` |
| `confogl_block_punch_rock` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否禁止坦克同时拳击与投掷石头 | 服务器端本插件逻辑；经 CreateConVarEx 包装，实际名称自动加 confogl_ 前缀并附加 CVAR_FLAGS | `GhostTank.sp:47` |
| `confogl_ghost_warp` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 幽灵特感能否按右键传送到下一个生还者 | 服务器端本插件逻辑；经 CreateConVarEx 包装，实际名称自动加 confogl_ 前缀并附加 CVAR_FLAGS | `GhostWarp.sp:23` |
| `confogl_ghost_warp_reload` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 幽灵传送使用鼠标右键还是换弹键 | 服务器端本插件逻辑；经 CreateConVarEx 包装，实际名称自动加 confogl_ 前缀并附加 CVAR_FLAGS | `GhostWarp.sp:24` |
| `confogl_customcfg` | `` | 字符串或表达式 | 无上下界 | 内部使用，源码标注请勿修改 | 服务器端本插件逻辑；不写入 cfg 存档；经 CreateConVarEx 包装，实际名称自动加 confogl_ 前缀并附加 CVAR_FLAGS | `configs.sp:32` |
| `confogl_debug` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否开启所有 confogl 模块的调试日志 | 服务器端本插件逻辑；经 CreateConVarEx 包装，实际名称自动加 confogl_ 前缀并附加 CVAR_FLAGS | `debug.sp:15` |
| `confogl_enable_itemtracking` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用物品跟踪模块 | 服务器端本插件逻辑；经 CreateConVarEx 包装，实际名称自动加 confogl_ 前缀并附加 CVAR_FLAGS | `ItemTracking.sp:164` |
| `confogl_itemtracking_savespawns` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否让两回合的物品刷新保持一致 | 服务器端本插件逻辑；经 CreateConVarEx 包装，实际名称自动加 confogl_ 前缀并附加 CVAR_FLAGS | `ItemTracking.sp:165` |
| `confogl_itemtracking_mapspecific` | `0` | 开关 0 或 1 | 0.0 ~ 3.0 | mapinfo.txt 覆盖方式：0 忽略该文件，1 允许减少上限，2 允许提高上限 | 服务器端本插件逻辑；经 CreateConVarEx 包装，实际名称自动加 confogl_ 前缀并附加 CVAR_FLAGS | `ItemTracking.sp:166` |
| `confogl_itemtracking_playeritems` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否忽略玩家出生自带的物品，非对战模式无影响 | 服务器端本插件逻辑；经 CreateConVarEx 包装，实际名称自动加 confogl_ 前缀并附加 CVAR_FLAGS | `ItemTracking.sp:167` |
| `confogl_pills_flow_min` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 止痛药可能刷出的最小地图流程比例，早于该值的会被移除；0 且 max 为 1、separation 为 0 时关闭流程过滤 | 服务器端本插件逻辑；经 CreateConVarEx 包装，实际名称自动加 confogl_ 前缀并附加 CVAR_FLAGS | `ItemTracking.sp:172` |
| `confogl_pills_flow_max` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 止痛药可能刷出的最大地图流程比例，晚于该值的会被移除 | 服务器端本插件逻辑；经 CreateConVarEx 包装，实际名称自动加 confogl_ 前缀并附加 CVAR_FLAGS | `ItemTracking.sp:173` |
| `confogl_pills_flow_separation` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 两个保留的止痛药刷新点之间的最小流程比例间隔，0 表示不强制间隔，距离过近时保留较早的那个 | 服务器端本插件逻辑；经 CreateConVarEx 包装，实际名称自动加 confogl_ 前缀并附加 CVAR_FLAGS | `ItemTracking.sp:174` |
| `confogl_pills_flow_finale` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否在终局地图同样应用止痛药流程窗口，0 表示终局豁免 | 服务器端本插件逻辑；经 CreateConVarEx 包装，实际名称自动加 confogl_ 前缀并附加 CVAR_FLAGS | `ItemTracking.sp:175` |
| `confogl_pills_flow_max_detour` | `0` | 开关 0 或 1 | ≥ 0.0 | 止痛药刷新点偏离最短路线允许的最大绕行距离，单位为导航单位，0 关闭；偏离路线的会被移除 | 服务器端本插件逻辑；经 CreateConVarEx 包装，实际名称自动加 confogl_ 前缀并附加 CVAR_FLAGS | `ItemTracking.sp:176` |
| `confogl_pills_flow_fill` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否在主线随机流程位置补齐缺失的止痛药，0 关闭 | 服务器端本插件逻辑；经 CreateConVarEx 包装，实际名称自动加 confogl_ 前缀并附加 CVAR_FLAGS | `ItemTracking.sp:177` |
| `confogl_pills_flow_fill_min` | `0.3` | 浮点 | 0.0 ~ 1.0 | 随机补齐的止痛药所允许的最小地图流程比例 | 服务器端本插件逻辑；经 CreateConVarEx 包装，实际名称自动加 confogl_ 前缀并附加 CVAR_FLAGS | `ItemTracking.sp:178` |
| `confogl_pills_flow_fill_max` | `1.0` | 开关 0 或 1 | 0.0 ~ 1.0 | 随机补齐的止痛药所允许的最大地图流程比例，安全室区域被排除 | 服务器端本插件逻辑；经 CreateConVarEx 包装，实际名称自动加 confogl_ 前缀并附加 CVAR_FLAGS | `ItemTracking.sp:179` |
| `confogl_pills_flow_visualize` | `0` | 开关 0 或 1 | 0.0 ~ 2.0 | 调试用：1 不移除止痛药而是发光标注，2 正常移除后把保留的发光 | 服务器端本插件逻辑；经 CreateConVarEx 包装，实际名称自动加 confogl_ 前缀并附加 CVAR_FLAGS | `ItemTracking.sp:180` |
| `confogl_match_restart` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 强制或请求比赛模式时是否重开地图 | 服务器端本插件逻辑；经 CreateConVarEx 包装，实际名称自动加 confogl_ 前缀并附加 CVAR_FLAGS | `ReqMatch.sp:52` |
| `confogl_match_autoload` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 玩家连接且服务器不在比赛模式时是否自动进入比赛模式 | 服务器端本插件逻辑；经 CreateConVarEx 包装，实际名称自动加 confogl_ 前缀并附加 CVAR_FLAGS | `ReqMatch.sp:54` |
| `confogl_match_autoconfig` | `` | 字符串或表达式 | 无上下界 | 自动加载启用时加载哪个配置 | 服务器端本插件逻辑；经 CreateConVarEx 包装，实际名称自动加 confogl_ 前缀并附加 CVAR_FLAGS | `ReqMatch.sp:55` |
| `confogl_match_execcfg_on` | `confogl.cfg` | 字符串或表达式 | 无上下界 | 比赛模式开始及之后每张图要执行的 cfg 文件 | 服务器端本插件逻辑；经 CreateConVarEx 包装，实际名称自动加 confogl_ 前缀并附加 CVAR_FLAGS | `ReqMatch.sp:56` |
| `confogl_match_execcfg_plugins` | `generalfixes.cfg;confogl_plugins.cfg;sharedplugins.cfg` | 字符串或表达式 | 无上下界 | 比赛模式开始时执行的插件 cfg，仅执行一次 | 服务器端本插件逻辑；经 CreateConVarEx 包装，实际名称自动加 confogl_ 前缀并附加 CVAR_FLAGS | `ReqMatch.sp:58` |
| `confogl_match_execcfg_off` | `confogl_off.cfg` | 字符串或表达式 | 无上下界 | 比赛模式结束时执行的 cfg 文件 | 服务器端本插件逻辑；经 CreateConVarEx 包装，实际名称自动加 confogl_ 前缀并附加 CVAR_FLAGS | `ReqMatch.sp:59` |
| `confogl_match_untracked_cvar_reset_file` | `RM_UNTRACKED_CVAR_RESET_PATH` | 字符串或表达式 | 无上下界 | addons/sourcemod 下列出在 resetmatch 时恢复的未跟踪 cvar 的文件 | 服务器端本插件逻辑；经 CreateConVarEx 包装，实际名称自动加 confogl_ 前缀并附加 CVAR_FLAGS | `ReqMatch.sp:60` |
| `confogl_match_untracked_cvar_reset_debug` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否记录 resetmatch 期间未跟踪 cvar 的恢复细节 | 服务器端本插件逻辑；经 CreateConVarEx 包装，实际名称自动加 confogl_ 前缀并附加 CVAR_FLAGS | `ReqMatch.sp:61` |
| `confogl_match_reloaded` | `0` | 开关 int 0 或 1 | 无上下界 | 内部使用，源码标注请勿修改，用于防止比赛模式反复循环 | 服务器端本插件逻辑；不写入 cfg 存档；经 CreateConVarEx 包装，实际名称自动加 confogl_ 前缀并附加 CVAR_FLAGS | `ReqMatch.sp:75` |
| `confogl_match_map` | `` | 字符串或表达式 | 无上下界 | 内部使用，源码标注请勿修改，用于保存将要切换到的地图 | 服务器端本插件逻辑；不写入 cfg 存档；经 CreateConVarEx 包装，实际名称自动加 confogl_ 前缀并附加 CVAR_FLAGS | `ReqMatch.sp:81` |
| `confogl_SM_enable` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | L4D2 自定义计分系统开关 | 服务器端本插件逻辑；经 CreateConVarEx 包装，实际名称自动加 confogl_ 前缀并附加 CVAR_FLAGS | `ScoreMod.sp:59` |
| `confogl_SM_healthbonusratio` | `2.0` | 浮点 | 0.25 ~ 5.0 | 生命奖励倍率，范围 0.25~5 | 服务器端本插件逻辑；经 CreateConVarEx 包装，实际名称自动加 confogl_ 前缀并附加 CVAR_FLAGS | `ScoreMod.sp:60` |
| `confogl_SM_survivalbonusratio` | `0.0` | 开关 0 或 1 | 无上下界 | 按地图距离计算固定生存奖励的比例 | 服务器端本插件逻辑；经 CreateConVarEx 包装，实际名称自动加 confogl_ 前缀并附加 CVAR_FLAGS | `ScoreMod.sp:61` |
| `confogl_SM_tempmulti_incap_0` | `0.30625` | 浮点 | 0.0 ~ 1.0 | 未有倒地记录的生还者其临时生命的重要程度 | 服务器端本插件逻辑；经 CreateConVarEx 包装，实际名称自动加 confogl_ 前缀并附加 CVAR_FLAGS | `ScoreMod.sp:62` |
| `confogl_SM_tempmulti_incap_1` | `0.17500` | 浮点 | 0.0 ~ 1.0 | 倒地一次的生还者其临时生命的重要程度 | 服务器端本插件逻辑；经 CreateConVarEx 包装，实际名称自动加 confogl_ 前缀并附加 CVAR_FLAGS | `ScoreMod.sp:63` |
| `confogl_SM_tempmulti_incap_2` | `0.10000` | 浮点 | 0.0 ~ 1.0 | 倒地两次即黑白的生还者其临时生命的重要程度 | 服务器端本插件逻辑；经 CreateConVarEx 包装，实际名称自动加 confogl_ 前缀并附加 CVAR_FLAGS | `ScoreMod.sp:64` |
| `confogl_SM_mapmulti` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否把生命奖励上限提升到距离上限 | 服务器端本插件逻辑；经 CreateConVarEx 包装，实际名称自动加 confogl_ 前缀并附加 CVAR_FLAGS | `ScoreMod.sp:65` |
| `confogl_SM_custommaxdistance` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否使用配置中的自定义最大距离 | 服务器端本插件逻辑；经 CreateConVarEx 包装，实际名称自动加 confogl_ 前缀并附加 CVAR_FLAGS | `ScoreMod.sp:66` |
| `confogl_boss_unprohibit` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否允许 boss 在所有地图刷新，即使该地图原本不允许 | 服务器端本插件逻辑；经 CreateConVarEx 包装，实际名称自动加 confogl_ 前缀并附加 CVAR_FLAGS | `UnprohibitBosses.sp:16` |
| `confogl_match_killlobbyres` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 \ | 比赛开始后是否清除大厅预留 | 服务器端本插件逻辑；经 CreateConVarEx 包装，实际名称自动加 confogl_ 前缀并附加 CVAR_FLAGS | `UnreserveLobby.sp:13` |
| `confogl_waterslowdown` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用额外的水中减速 | 服务器端本插件逻辑；经 CreateConVarEx 包装，实际名称自动加 confogl_ 前缀并附加 CVAR_FLAGS | `WaterSlowdown.sp:22` |
| `confogl_slowdown_factor` | `0.90` | 浮点 | 无上下界 | 水对生还者的减速程度，1.00 为原生值 | 服务器端本插件逻辑；经 CreateConVarEx 包装，实际名称自动加 confogl_ 前缀并附加 CVAR_FLAGS | `WaterSlowdown.sp:23` |
| `confogl_limit_sniper` | `1` | 开关 0 或 1 | 0.0 ~ 4.0 | 同时最多允许的狙击步枪数量 | 服务器端本插件逻辑；经 CreateConVarEx 包装，实际名称自动加 confogl_ 前缀并附加 CVAR_FLAGS | `WeaponCustomization.sp:30` |
| `confogl_replace_cssweapons` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否把 CSS 武器替换为普通 L4D2 武器 | 服务器端本插件逻辑；经 CreateConVarEx 包装，实际名称自动加 confogl_ 前缀并附加 CVAR_FLAGS | `WeaponInformation.sp:577` |
| `confogl_remove_grenade` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否移除所有榴弹发射器 | 服务器端本插件逻辑；经 CreateConVarEx 包装，实际名称自动加 confogl_ 前缀并附加 CVAR_FLAGS | `WeaponInformation.sp:583` |
| `confogl_remove_chainsaw` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否移除所有电锯 | 服务器端本插件逻辑；经 CreateConVarEx 包装，实际名称自动加 confogl_ 前缀并附加 CVAR_FLAGS | `WeaponInformation.sp:584` |
| `confogl_remove_m60` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否移除所有 M60 | 服务器端本插件逻辑；经 CreateConVarEx 包装，实际名称自动加 confogl_ 前缀并附加 CVAR_FLAGS | `WeaponInformation.sp:585` |
| `confogl_remove_statickits` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否移除地图内置的静态医疗包 | 服务器端本插件逻辑；经 CreateConVarEx 包装，实际名称自动加 confogl_ 前缀并附加 CVAR_FLAGS | `WeaponInformation.sp:587` |
| `confogl_remove_defib` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否移除所有除颤器 | 服务器端本插件逻辑；经 CreateConVarEx 包装，实际名称自动加 confogl_ 前缀并附加 CVAR_FLAGS | `WeaponInformation.sp:588` |
| `confogl_remove_upg_explosive` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否移除所有高爆弹药包 | 服务器端本插件逻辑；经 CreateConVarEx 包装，实际名称自动加 confogl_ 前缀并附加 CVAR_FLAGS | `WeaponInformation.sp:589` |
| `confogl_remove_upg_incendiary` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否移除所有燃烧弹药包 | 服务器端本插件逻辑；经 CreateConVarEx 包装，实际名称自动加 confogl_ 前缀并附加 CVAR_FLAGS | `WeaponInformation.sp:590` |
| `confogl_replace_tier2` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否把起点与终点安全室的二级武器替换为一级武器 | 服务器端本插件逻辑；经 CreateConVarEx 包装，实际名称自动加 confogl_ 前缀并附加 CVAR_FLAGS | `WeaponInformation.sp:601` |
| `confogl_replace_tier2_finale` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 终局时是否把起点安全室的二级武器替换为一级武器 | 服务器端本插件逻辑；经 CreateConVarEx 包装，实际名称自动加 confogl_ 前缀并附加 CVAR_FLAGS | `WeaponInformation.sp:602` |
| `confogl_replace_tier2_all` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否在任何位置把所有二级武器替换为一级武器 | 服务器端本插件逻辑；经 CreateConVarEx 包装，实际名称自动加 confogl_ 前缀并附加 CVAR_FLAGS | `WeaponInformation.sp:603` |
| `confogl_limit_tier2` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否限制安全室外二级武器的数量，首次拾取时把二级武器堆替换为一级 | 服务器端本插件逻辑；经 CreateConVarEx 包装，实际名称自动加 confogl_ 前缀并附加 CVAR_FLAGS | `WeaponInformation.sp:604` |
| `confogl_limit_tier2_saferoom` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否限制安全室内二级武器的数量，首次拾取时把二级武器堆替换为一级 | 服务器端本插件逻辑；经 CreateConVarEx 包装，实际名称自动加 confogl_ 前缀并附加 CVAR_FLAGS | `WeaponInformation.sp:605` |
| `confogl_replace_startkits` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否把起点的医疗包替换为止痛药 | 服务器端本插件逻辑；经 CreateConVarEx 包装，实际名称自动加 confogl_ 前缀并附加 CVAR_FLAGS | `WeaponInformation.sp:606` |
| `confogl_replace_finalekits` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否把终局的医疗包替换为止痛药 | 服务器端本插件逻辑；经 CreateConVarEx 包装，实际名称自动加 confogl_ 前缀并附加 CVAR_FLAGS | `WeaponInformation.sp:607` |
| `confogl_remove_lasersight` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否移除所有激光瞄准升级 | 服务器端本插件逻辑；经 CreateConVarEx 包装，实际名称自动加 confogl_ 前缀并附加 CVAR_FLAGS | `WeaponInformation.sp:608` |
| `confogl_remove_saferoomitems` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否移除安全室内除医疗包以外的额外物品 | 服务器端本插件逻辑；经 CreateConVarEx 包装，实际名称自动加 confogl_ 前缀并附加 CVAR_FLAGS | `WeaponInformation.sp:609` |

## [L4D2]Charger_Collision_Patch —— `duoren/Charger_Collision_patch.sp`

- 源文件：`duoren/Charger_Collision_patch.sp`
- myinfo：author=Lux
- myinfo description（源码原文）：Fixes charger only allow to his 1 survivor index & allows charging same target more than once
- HookConVarChange：`hConvar`→`ScaleDownCvar`、`hConvar`→`ScaleDownCvar`

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `charger_collision_patch_version` | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | 服务器端本插件逻辑；不写入 cfg 存档 | `Charger_Collision_patch.sp:56` |

## [L4D2]Defib_Fix —— `duoren/Defib_Fix.sp`

- 源文件：`duoren/Defib_Fix.sp`
- myinfo：author=Lux
- myinfo description（源码原文）：Fixes defibbing from failing when defibbing an alive character index

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `defib_fix_version` | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | 服务器端本插件逻辑；不写入 cfg 存档 | `Defib_Fix.sp:98` |

## [L4D2][NIX] IsReachable_Detour —— `duoren/IsReachable_Detour.sp`

- 源文件：`duoren/IsReachable_Detour.sp`
- myinfo：author=Dragokas
- myinfo description（源码原文）：Fixing the valve crash with null pointer dereference in SurvivorBot::IsReachable(CBaseEntity *)

### 指令

（本插件未注册指令）

### ConVar

（本插件未注册 ConVar）

## [L4D2]Witch_Double_Start_Fix —— `duoren/Witch_Double_Startle_Fix.sp`

- 源文件：`duoren/Witch_Double_Startle_Fix.sp`
- myinfo：author=Lux
- myinfo description（源码原文）：Fixes witch when wandering playing startle twice by forcing the NextThink to end the startle.

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `witch_double_start_fix` | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | 服务器端本插件逻辑；不写入 cfg 存档 | `Witch_Double_Startle_Fix.sp:50` |

## [L4D1/2]Witch_Target_Patch —— `duoren/Witch_Target_patch.sp`

- 源文件：`duoren/Witch_Target_patch.sp`
- myinfo：author=Lux
- myinfo description（源码原文）：Fixes witch targeting wrong person

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `witch_target_patch_version` | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | 服务器端本插件逻辑；不写入 cfg 存档 | `Witch_Target_patch.sp:70` |

## [L4D2]Character_manager —— `duoren/l4d2_character_manager.sp`

- 源文件：`duoren/l4d2_character_manager.sp`
- myinfo：author=Lux, $atanic $pirit
- myinfo description（源码原文）：Sets bots to least used survivor character when spawned from(0-7)
- HookConVarChange：`hCvar_SurvivorSet`→`eConvarChanged`、`hCvar_IdentityFix`→`eConvarChanged`、`hCvar_ManagePeople`→`eConvarChanged`

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `l4d2_character_manager_version` | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | 服务器端本插件逻辑；不写入 cfg 存档 | `l4d2_character_manager.sp:105` |
| `l4d2_survivor_set` | `3` | 整数，源码写作浮点 | 0.0 ~ 3.0 | 生还者角色集：0 用地图默认，1 用 L4D1，2 用 L4D2，3 两者都用 | 服务器端本插件逻辑 | `l4d2_character_manager.sp:106` |
| `l4d2_identity_fix` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否对真人玩家启用身份修复：0 关，1 开 | 服务器端本插件逻辑 | `l4d2_character_manager.sp:108` |
| `l4d2_manage_people` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否同时管理真人：0 关，1 开，接管 Bot 时会覆盖身份修复 | 服务器端本插件逻辑 | `l4d2_character_manager.sp:110` |

## Finale rescue vehicle mover for 4+ survivors —— `duoren/l4d2_rescue_vehicle_mover.sp`

- 源文件：`duoren/l4d2_rescue_vehicle_mover.sp`
- myinfo：version=1.0，author=sorallll
- myinfo description（源码原文）：Properly moves extra 4+ survivors to their intended location during finale rescue sequences

### 指令

（本插件未注册指令）

### ConVar

（本插件未注册 ConVar）

## [L4D2] 4+ Survivor Afk dead bot Fix —— `duoren/l4dafkfix_deadbot.sp`

- 源文件：`duoren/l4dafkfix_deadbot.sp`
- myinfo：author=MI 5, HarryPotter
- myinfo description（源码原文）：Fixes issue when a bot die, his IDLE player become fully spectator rather than take over dead bot in 4+ survivors games

### 指令

（本插件未注册指令）

### ConVar

（本插件未注册 ConVar）

## [L4D2]Survivor_AFK_Fix —— `duoren/survivor_afk_fix.sp`

- 源文件：`duoren/survivor_afk_fix.sp`
- myinfo：author=Lux
- myinfo description（源码原文）：Fixes survivor going AFK game function.

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_afktest` | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 源码未给描述；据 `duoren/survivor_afk_fix.sp:108-117` 的 `PrepSDKCall_SetFromConf(hGamedata, SDKConf_Signature, "CTerrorPlayer::GoAwayFromKeyboard")` 与 :123-131 回调 `AFKTEST` 中的 `SDKCall(hAFKSDKCall, client)`，推测为：调试命令，对指定玩家强制触发 `CTerrorPlayer::GoAwayFromKeyboard`（令其进入挂机/AFK 状态），用于验证本插件的 AFK 修复逻辑；另注意 :29 为 `#define DEBUG 0`，默认编译时该命令不会注册（置信度：高） | `survivor_afk_fix.sp:117` |

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `survivor_afk_fix_ver` | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | 服务器端本插件逻辑；不写入 cfg 存档 | `survivor_afk_fix.sp:66` |

## survivor_chat_select.sp（myinfo 缺 name，用文件名代替） —— `duoren/survivor_chat_select.sp`

- 源文件：`duoren/survivor_chat_select.sp`
- myinfo：author=DeatChaos25, Mi123456 & Merudo
- myinfo description（源码原文）：Select a survivor character by typing their name into the chat.

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_zoey` | 无参数 | 无（任意玩家可用） | 把自己使用的生还者切换为 Zoey | `survivor_chat_select.sp:79` |
| `sm_nick` | 无参数 | 无（任意玩家可用） | 把自己使用的生还者切换为 Nick | `survivor_chat_select.sp:80` |
| `sm_ellis` | 无参数 | 无（任意玩家可用） | 把自己使用的生还者切换为 Ellis | `survivor_chat_select.sp:81` |
| `sm_coach` | 无参数 | 无（任意玩家可用） | 把自己使用的生还者切换为 Coach | `survivor_chat_select.sp:82` |
| `sm_rochelle` | 无参数 | 无（任意玩家可用） | 把自己使用的生还者切换为 Rochelle | `survivor_chat_select.sp:83` |
| `sm_bill` | 无参数 | 无（任意玩家可用） | 把自己使用的生还者切换为 Bill | `survivor_chat_select.sp:84` |
| `sm_francis` | 无参数 | 无（任意玩家可用） | 把自己使用的生还者切换为 Francis | `survivor_chat_select.sp:85` |
| `sm_louis` | 无参数 | 无（任意玩家可用） | 把自己使用的生还者切换为 Louis | `survivor_chat_select.sp:86` |
| `sm_z` | 无参数 | 无（任意玩家可用） | 把自己切换为 Zoey，sm_zoey 的短别名 | `survivor_chat_select.sp:88` |
| `sm_n` | 无参数 | 无（任意玩家可用） | 把自己切换为 Nick，短别名 | `survivor_chat_select.sp:89` |
| `sm_e` | 无参数 | 无（任意玩家可用） | 把自己切换为 Ellis，短别名 | `survivor_chat_select.sp:90` |
| `sm_c` | 无参数 | 无（任意玩家可用） | 把自己切换为 Coach，短别名 | `survivor_chat_select.sp:91` |
| `sm_r` | 无参数 | 无（任意玩家可用） | 把自己切换为 Rochelle，短别名 | `survivor_chat_select.sp:92` |
| `sm_b` | 无参数 | 无（任意玩家可用） | 把自己切换为 Bill，短别名 | `survivor_chat_select.sp:93` |
| `sm_f` | 无参数 | 无（任意玩家可用） | 把自己切换为 Francis，短别名 | `survivor_chat_select.sp:94` |
| `sm_l` | 无参数 | 无（任意玩家可用） | 把自己切换为 Louis，短别名 | `survivor_chat_select.sp:95` |
| `sm_csc` | 无参数 | 需要 ADMFLAG_GENERIC（b，通用管理员） | 打开菜单选择指定玩家的生还者角色 | `survivor_chat_select.sp:97` |
| `sm_csm` | 无参数 | 无（任意玩家可用） | 打开生还者角色选择菜单，是否仅管理员可用由 l4d_csm_admins_only 控制 | `survivor_chat_select.sp:98` |

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `l4d_csm_admins_only` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否把 sm_csm 命令限制为管理员使用：1 仅管理员 | 服务器端本插件逻辑 | `survivor_chat_select.sp:103` |
| `l4d_scs_zoey` | `1` | 整数，源码写作浮点 | 0.0 ~ 2.0 | Zoey 的模型：0 用 Rochelle，1 用 Zoey，2 用 Nick | 服务器端本插件逻辑 | `survivor_chat_select.sp:104` |
| `l4d_scs_botschange` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否把新出现的 Bot 换成当前数量最少的生还者：1 开，0 关 | 服务器端本插件逻辑 | `survivor_chat_select.sp:105` |
| `l4d_scs_cookies` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否记住玩家上次使用的生还者：1 开，0 关 | 服务器端本插件逻辑 | `survivor_chat_select.sp:106` |

## [L4D1/2]witch_prevent_target_loss —— `duoren/witch_prevent_target_loss.sp`

- 源文件：`duoren/witch_prevent_target_loss.sp`
- myinfo：author=Lux
- myinfo description（源码原文）：Prevents the witch from randomly loosing target.

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `witch_prevent_target_loss` | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | 服务器端本插件逻辑；不写入 cfg 存档 | `witch_prevent_target_loss.sp:61` |

## Kills Statistic —— `extend/HitStatistics.sp`

- 源文件：`extend/HitStatistics.sp`
- myinfo：author=Lin,Hoongdou
- myinfo description（源码原文）：Kills Statistic

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_kills` | 无参数 | 无（任意玩家可用） | 查看击杀统计，即 MVP 统计 | `HitStatistics.sp:46` |
| `sm_killsme` | 无参数 | 无（任意玩家可用） | 查看自己的击杀统计 | `HitStatistics.sp:47` |

### ConVar

（本插件未注册 ConVar）

## SendFile Exploit Fix (v3.3) —— `extend/SendFileExploitFixV3.3.sp`

- 源文件：`extend/SendFileExploitFixV3.3.sp`
- myinfo：version=3.3，author=backwards
- myinfo description（源码原文）：Prevents Clients From Exploiting The Un-Patched SRCDS SendFile Command, V3 Adds ReceiveFile Support.)

### 指令

（本插件未注册指令）

### ConVar

（本插件未注册 ConVar）

## SpecListener —— `extend/SpecListener.sp`

- 源文件：`extend/SpecListener.sp`
- myinfo：version=3.0，author=waertf, bear, bman, lechuga
- myinfo description（源码原文）：Allows spectator listen others team voice for l4d
- HookConVarChange：`g_hAllTalk`→`OnConvarChange_Alltalk`

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_listen` | <菜单项> | 无（任意玩家可用） | 打开旁观者监听菜单，包含准备与暂停面板 | `SpecListener.sp:86` |

### ConVar

（本插件未注册 ConVar）

## [L4D2]Survivor_Legs_Restore —— `extend/Survivor_Legs.sp`

- 源文件：`extend/Survivor_Legs.sp`
- myinfo：author=Lux
- myinfo description（源码原文）：Add's Left 4 Dead 1 Style ViewModel Legs
- HookConVarChange：`FindConVar("mp_facefronttime")`→`eConvarChanged`

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `survivor_legs_version` | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | 服务器端本插件逻辑；仅服务器；不写入 cfg 存档 | `Survivor_Legs.sp:87` |

## ThirdPersonShoulder_Detect —— `extend/ThirdPersonShoulder_Detect.sp`

- 源文件：`extend/ThirdPersonShoulder_Detect.sp`
- myinfo：author=MasterMind420 & Lux
- myinfo description（源码原文）：Detects thirdpersonshoulder command for other plugins to use
- HookConVarChange：`hCvar_GameMode`→`eConvarChanged`、`hCvar_AllowVersus`→`eConvarChanged`

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `ThirdPersonShoulder_Detect_Version` | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | 服务器端本插件逻辑；仅服务器；不写入 cfg 存档 | `ThirdPersonShoulder_Detect.sp:37` |
| `thirdpersonshoulder_detect_allow_versus` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 0 在对战类配置中强制第一人称，1 检测客户端真实的第三人称状态 | 服务器端本插件逻辑 | `ThirdPersonShoulder_Detect.sp:38` |

## Advertisements —— `extend/advertisements.sp`

- 源文件：`extend/advertisements.sp`
- myinfo：author=Tsunami
- myinfo description（源码原文）：Display advertisements

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_advertisements_reload` | 无参数 | 服务器控制台命令，玩家无法使用 | 重新加载广告配置，服务器控制台命令 | `advertisements.sp:76` |

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `sm_advertisements_version` | `PL_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | 服务器端本插件逻辑 | `advertisements.sp:63` |
| `sm_advertisements_enabled` | `1` | 整数 | 无上下界 | 是否启用广告显示 | 服务器端本插件逻辑 | `advertisements.sp:64` |
| `sm_advertisements_file` | `advertisements.txt` | 字符串或表达式 | 无上下界 | 读取广告的文本文件名 | 服务器端本插件逻辑 | `advertisements.sp:65` |
| `sm_advertisements_interval` | `30` | 整数 | 无上下界 | 两条广告之间的间隔秒数 | 服务器端本插件逻辑 | `advertisements.sp:66` |
| `sm_advertisements_random` | `0` | 整数 | 无上下界 | 是否随机播放广告 | 服务器端本插件逻辑 | `advertisements.sp:67` |

## Anne DB Connection Hub —— `extend/anne_db.sp`

- 源文件：`extend/anne_db.sp`
- myinfo：author=morzlee
- myinfo description（源码原文）：Shares one MySQL connection per database target across plugins.

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_annedb_status` | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 在控制台输出共享数据库连接状态、目标数与待处理数 | `anne_db.sp:97` |

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `anne_db_charset` | `utf8mb4` | 字符串或表达式 | 无上下界 | 共享连接建立后统一设置的字符集，留空不设置 | 服务器端本插件逻辑 | `anne_db.sp:87` |
| `anne_db_keepalive` | `120` | 整数，源码写作浮点 | ≥ 0.0 | 共享连接保活间隔秒数，须小于 MySQL wait_timeout，0 关闭 | 服务器端本插件逻辑 | `anne_db.sp:88` |
| `anne_db_request_timeout` | `30` | 整数，源码写作浮点 | ≥ 1.0 | 异步连接请求最长等待秒数，超时后回调失败 | 服务器端本插件逻辑 | `anne_db.sp:89` |
| `anne_db_sync_cooldown` | `30` | 整数，源码写作浮点 | ≥ 0.0 | 连接失败后多少秒内同步获取直接返回失败，避免主线程反复阻塞 | 服务器端本插件逻辑 | `anne_db.sp:90` |
| `anne_db_version` | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | 服务器端本插件逻辑；不写入 cfg 存档 | `anne_db.sp:91` |

## [ANY] Attachments API —— `extend/attachments_api.sp`

- 源文件：`extend/attachments_api.sp`
- myinfo：author=SilverShot
- myinfo description（源码原文）：Allows plugins to attach things to weapons. Fixes player attachments when their model changes.

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_attachment_qc` | <qc 文件夹路径> | 需要 ADMFLAG_ROOT（z，最高权限） | 解析 .qc 文件以获取模型挂点名称 | `attachments_api.sp:203` |
| `sm_attachment_reload` | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 重新加载 attachments_api 数据配置 | `attachments_api.sp:204` |

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `attachments_api_check` | `0.1` | 浮点 | 无上下界 | 检查玩家模型是否变化的间隔，需 attachments_api_models 为 1 才启用 | 服务器端本插件逻辑 | `attachments_api.sp:188` |
| `attachments_api_equip` | `0.1` | 浮点 | 无上下界 | 玩家模型变化时，为修复挂件先丢弃武器再重新装备的延迟秒数 | 服务器端本插件逻辑 | `attachments_api.sp:189` |
| `attachments_api_models` | `sChk` | 字符串或表达式 | 无上下界 | 0 关闭；1 检测玩家模型变化以修复玩家身上的挂件 | 服务器端本插件逻辑 | `attachments_api.sp:190` |
| `attachments_api_weapons` | `sChk` | 字符串或表达式 | 无上下界 | 0 关闭；1 检测武器模型变化以修复武器挂件 | 服务器端本插件逻辑 | `attachments_api.sp:191` |
| `attachments_api_version` | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | 服务器端本插件逻辑；不写入 cfg 存档 | `attachments_api.sp:192` |

## My plugins/file auto update —— `extend/autoupdate.sp`

- 源文件：`extend/autoupdate.sp`
- myinfo：version=2022.05.05，author=东
- myinfo description（源码原文）：自动更新插件等文件

### 指令

（本插件未注册指令）

### ConVar

（本插件未注册 ConVar）

## Chat Log —— `extend/chatlog.sp`

- 源文件：`extend/chatlog.sp`
- myinfo：version=1.2，author=venus, 东
- myinfo description（源码原文）：Save all user messages in database, add server name and port

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `sm_chatlog_cleartable_enabled` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用清空聊天记录表：1 开，0 关 | 服务器端本插件逻辑 | `chatlog.sp:29` |
| `sm_chatlog_cleartable_duration` | `12 MONTH` | 字符串或表达式 | 无上下界 | 聊天记录表重置的周期 | 服务器端本插件逻辑 | `chatlog.sp:30` |

## SM Fortnite Emotes Extended —— `extend/fornite_l4d.sp`

- 源文件：`extend/fornite_l4d.sp`
- myinfo：version=2020，author=Kodua, Franc1sco franug, TheBO$$, Foxhound
- myinfo description（源码原文）：This plugin is for demonstration of some animations from Fortnite in L4D2

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_emotes` | 无参数 | 无（任意玩家可用） | 打开表情菜单，是否可用由 sm_emotes_admin_flag_menu 控制 | `fornite_l4d.sp:93` |
| `sm_emote` | 无参数 | 无（任意玩家可用） | 打开表情菜单，sm_emotes 的别名 | `fornite_l4d.sp:94` |
| `sm_dances` | 无参数 | 无（任意玩家可用） | 打开舞蹈菜单，是否可用由 sm_dances_admin_flag_menu 控制 | `fornite_l4d.sp:95` |
| `sm_dance` | 无参数 | 无（任意玩家可用） | 打开舞蹈菜单，sm_dances 的别名 | `fornite_l4d.sp:96` |
| `sm_setemotes` | <目标玩家> [表情 ID] | 需要 ADMFLAG_GENERIC（b，通用管理员） | 给目标玩家设置表情，用法 sm_setemotes <目标玩家> [表情 ID] | `fornite_l4d.sp:97` |
| `sm_setemote` | <目标玩家> [表情 ID] | 需要 ADMFLAG_GENERIC（b，通用管理员） | 给目标玩家设置表情，sm_setemotes 的别名 | `fornite_l4d.sp:98` |
| `sm_setdances` | <目标玩家> [舞蹈 ID] | 需要 ADMFLAG_GENERIC（b，通用管理员） | 给目标玩家设置舞蹈 | `fornite_l4d.sp:99` |
| `sm_setdance` | <目标玩家> [舞蹈 ID] | 需要 ADMFLAG_GENERIC（b，通用管理员） | 给目标玩家设置舞蹈，sm_setdances 的别名 | `fornite_l4d.sp:100` |

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `sm_emotes_sounds` | `1` | 整数 | 无上下界 | 是否启用表情声音 | 服务器端本插件逻辑 | `fornite_l4d.sp:120` |
| `sm_emotes_cooldown` | `2.0` | 整数，源码写作浮点 | 无上下界 | 表情冷却秒数，-1 或 0 表示无冷却 | 服务器端本插件逻辑 | `fornite_l4d.sp:121` |
| `sm_emotes_soundvolume` | `1.0` | 整数，源码写作浮点 | 无上下界 | 舞蹈音量 | 服务器端本插件逻辑 | `fornite_l4d.sp:122` |
| `sm_emotes_admin_flag_menu` | `` | 字符串或表达式 | 无上下界 | 表情管理菜单所需权限 flag，留空表示所有玩家可用 | 服务器端本插件逻辑 | `fornite_l4d.sp:123` |
| `sm_dances_admin_flag_menu` | `` | 字符串或表达式 | 无上下界 | 舞蹈管理菜单所需权限 flag，留空表示所有玩家可用 | 服务器端本插件逻辑 | `fornite_l4d.sp:124` |
| `sm_emotes_hide_weapons` | `0` | 整数 | 无上下界 | 跳舞时是否隐藏武器：1 是，0 否 | 服务器端本插件逻辑 | `fornite_l4d.sp:125` |
| `sm_emotes_hide_enemies` | `0` | 整数 | 无上下界 | 跳舞时是否隐藏敌方玩家：1 是，0 否 | 服务器端本插件逻辑 | `fornite_l4d.sp:126` |
| `sm_emotes_teleportonend` | `0` | 整数 | 无上下界 | 跳舞开始时是否传送回原地，部分地图需要以此触发传送 | 服务器端本插件逻辑 | `fornite_l4d.sp:127` |
| `sm_emotes_speed` | `0.80` | 浮点 | 无上下界 | 动画播放速度，源码注明默认值为 1.0 | 服务器端本插件逻辑 | `fornite_l4d.sp:128` |
| `sm_emotes_add_downloads` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否由舞蹈插件一次性注册全部模型与音乐下载：1 是，0 否 | 服务器端本插件逻辑 | `fornite_l4d.sp:129` |

## Anne Global Chat —— `extend/global_chat.sp`

- 源文件：`extend/global_chat.sp`
- myinfo：author=OpenAI
- myinfo description（源码原文）：Cross-server chat through a shared MySQL table

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_qf` | <内容> | 无（任意玩家可用） | 发送一条全服聊天消息 | `global_chat.sp:189` |
| `sm_quanfu` | <内容> | 无（任意玩家可用） | 发送一条全服聊天消息，sm_qf 的别名 | `global_chat.sp:190` |
| `sm_zd` | <内容> | 无（任意玩家可用） | 发送找队友信息到全服，受每日次数限制 | `global_chat.sp:192` |
| `sm_zudui` | <内容> | 无（任意玩家可用） | 发送找队友信息，sm_zd 的别名 | `global_chat.sp:193` |
| `sm_zdy` | <内容> | 无（任意玩家可用） | 发送找队友信息，sm_zd 的别名 | `global_chat.sp:194` |
| `sm_qfmenu` | 无参数 | 无（任意玩家可用） | 打开全服聊天接收设置菜单 | `global_chat.sp:195` |
| `sm_qfadmin` | 无参数 | 无（任意玩家可用） | 打开全服聊天接收设置菜单，sm_qfmenu 的别名 | `global_chat.sp:196` |
| `sm_zdmenu` | 无参数 | 无（任意玩家可用） | 打开找队友提示接收设置菜单 | `global_chat.sp:197` |

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `sm_qf_enabled` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用全服聊天 | 服务器端本插件逻辑 | `global_chat.sp:161` |
| `sm_qf_database` | `globalchat` | 字符串或表达式 | 无上下界 | databases.cfg 中的数据库配置名 | 服务器端本插件逻辑 | `global_chat.sp:162` |
| `sm_qf_poll_interval` | `5.0` | 整数，源码写作浮点 | 2.0 ~ 30.0 | 全服聊天轮询间隔秒数，范围 2~30 | 服务器端本插件逻辑 | `global_chat.sp:163` |
| `sm_qf_idle_poll_interval` | `30.0` | 整数，源码写作浮点 | 5.0 ~ 300.0 | 无真人玩家时的全服聊天轮询间隔秒数，范围 5~300 | 服务器端本插件逻辑 | `global_chat.sp:164` |
| `sm_qf_poll_batch` | `30` | 整数，源码写作浮点 | 1.0 ~ 200.0 | 每次最多拉取的全服聊天消息条数，范围 1~200 | 服务器端本插件逻辑 | `global_chat.sp:165` |
| `sm_qf_cleanup_interval` | `21600` | 整数，源码写作浮点 | ≥ 300.0 | 清理旧全服聊天记录的间隔秒数，最小 300 | 服务器端本插件逻辑 | `global_chat.sp:166` |
| `sm_qf_retention_days` | `7` | 整数，源码写作浮点 | ≥ 0.0 | 全服聊天记录保留天数，0 表示不清理 | 服务器端本插件逻辑 | `global_chat.sp:167` |
| `sm_qf_prefix` | `[全服]` | 字符串或表达式 | 无上下界 | 全服聊天前缀 | 服务器端本插件逻辑 | `global_chat.sp:168` |
| `sm_qf_blacklist_filter` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否按 l4d2_blacklist 屏蔽全服聊天显示 | 服务器端本插件逻辑 | `global_chat.sp:169` |
| `sm_qf_blacklist_database` | `l4dstats` | 字符串或表达式 | 无上下界 | l4d2_blacklist 使用的 databases.cfg 配置名 | 服务器端本插件逻辑 | `global_chat.sp:170` |
| `sm_qf_blacklist_table` | `player_blocks` | 字符串或表达式 | 无上下界 | l4d2_blacklist 使用的数据库表名 | 服务器端本插件逻辑 | `global_chat.sp:171` |
| `sm_qf_blacklist_refresh_interval` | `60.0` | 整数，源码写作浮点 | 10.0 ~ 600.0 | 刷新在线玩家屏蔽缓存的间隔秒数，范围 10~600 | 服务器端本插件逻辑 | `global_chat.sp:172` |
| `sm_qf_limit_default` | `3` | 整数，源码写作浮点 | ≥ 0.0 | 普通玩家每日全服聊天次数上限 | 服务器端本插件逻辑 | `global_chat.sp:173` |
| `sm_qf_limit_1m` | `5` | 整数，源码写作浮点 | ≥ 0.0 | 积分 100 万以上玩家每日全服聊天次数上限 | 服务器端本插件逻辑 | `global_chat.sp:174` |
| `sm_qf_limit_5m` | `10` | 整数，源码写作浮点 | ≥ 0.0 | 积分 500 万以上玩家每日全服聊天次数上限 | 服务器端本插件逻辑 | `global_chat.sp:175` |
| `sm_qf_limit_10m` | `20` | 整数，源码写作浮点 | ≥ 0.0 | 积分 1000 万以上玩家每日全服聊天次数上限 | 服务器端本插件逻辑 | `global_chat.sp:176` |
| `sm_qf_limit_20m` | `30` | 整数，源码写作浮点 | ≥ 0.0 | 积分 2000 万以上玩家每日全服聊天次数上限 | 服务器端本插件逻辑 | `global_chat.sp:177` |
| `sm_qf_limit_kick_admin` | `50` | 整数，源码写作浮点 | ≥ 0.0 | 有 kick 权限但无 z 权限的管理员每日全服聊天次数上限 | 服务器端本插件逻辑 | `global_chat.sp:178` |
| `sm_qf_show_global` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 本服玩家能否看到普通全服聊天与上线提示 | 服务器端本插件逻辑 | `global_chat.sp:179` |
| `sm_zd_show_global` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 本服旁观玩家能否看到找队友全服提示 | 服务器端本插件逻辑 | `global_chat.sp:180` |
| `sm_zd_limit_default` | `3` | 整数，源码写作浮点 | ≥ 0.0 | 普通玩家每日找队友次数上限 | 服务器端本插件逻辑 | `global_chat.sp:182` |
| `sm_zd_limit_1m` | `5` | 整数，源码写作浮点 | ≥ 0.0 | 积分 100 万以上玩家每日找队友次数上限 | 服务器端本插件逻辑 | `global_chat.sp:183` |
| `sm_zd_limit_5m` | `10` | 整数，源码写作浮点 | ≥ 0.0 | 积分 500 万以上玩家每日找队友次数上限 | 服务器端本插件逻辑 | `global_chat.sp:184` |
| `sm_zd_limit_10m` | `20` | 整数，源码写作浮点 | ≥ 0.0 | 积分 1000 万以上玩家每日找队友次数上限 | 服务器端本插件逻辑 | `global_chat.sp:185` |
| `sm_zd_limit_20m` | `30` | 整数，源码写作浮点 | ≥ 0.0 | 积分 2000 万以上玩家每日找队友次数上限 | 服务器端本插件逻辑 | `global_chat.sp:186` |
| `sm_zd_limit_kick_admin` | `50` | 整数，源码写作浮点 | ≥ 0.0 | 有 kick 权限但无 z 权限的管理员每日找队友次数上限 | 服务器端本插件逻辑 | `global_chat.sp:187` |

## [L4D2] Healing Gnome —— `extend/gnome.sp`

- 源文件：`extend/gnome.sp`
- myinfo：author=SilverShot
- myinfo description（源码原文）：Heals players with temporary or main health when they hold the Gnome.

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_gnome` | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 在准星位置生成一只临时地精，仅可用于服务器端 | `gnome.sp:245` |
| `sm_gnomesave` | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 在准星位置生成地精并把位置保存到配置 | `gnome.sp:246` |
| `sm_gnomedel` | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 删除准星指向的地精，若已保存则同时从配置中删除 | `gnome.sp:247` |
| `sm_gnomekill` | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 删除当前地图所有地精并从配置中删除 | `gnome.sp:248` |
| `sm_gnomeglow` | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 切换是否让所有地精发光，便于查看摆放位置 | `gnome.sp:249` |
| `sm_gnomelist` | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 列出地精坐标与总数 | `gnome.sp:250` |
| `sm_gnometele` | <序号 1 至 MAX_GNOMES> | 需要 ADMFLAG_ROOT（z，最高权限） | 传送到指定序号的地精 | `gnome.sp:251` |
| `sm_gnomeang` | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 打开菜单调整准星所指地精的角度 | `gnome.sp:252` |
| `sm_gnomepos` | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 打开菜单调整准星所指地精的坐标 | `gnome.sp:253` |

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `l4d2_gnome_allow` | `1` | 整数 | 无上下界 | 0 关闭插件，1 开启插件 | 服务器端本插件逻辑 | `gnome.sp:183` |
| `l4d2_gnome_glow` | `200` | 整数 | 无上下界 | 地精发光的最远距离，0 关闭 | 服务器端本插件逻辑 | `gnome.sp:184` |
| `l4d2_gnome_glow_color` | `255 0 0` | 字符串或表达式 | 无上下界 | 地精发光颜色，三个 0~255 数值以空格分隔的 RGB，0 表示默认发光色 | 服务器端本插件逻辑 | `gnome.sp:185` |
| `l4d2_gnome_full` | `0` | 整数 | 无上下界 | 0 关闭；开启后在治疗到生命上限时移除黑白效果并补满生命，1 给临时生命，其余取值源码描述被截断 | 服务器端本插件逻辑 | `gnome.sp:186` |
| `l4d2_gnome_heal` | `1` | 整数 | 无上下界 | 是否用本插件的 cvar 来治疗手持地精的玩家，不影响治疗力场 | 服务器端本插件逻辑 | `gnome.sp:187` |
| `l4d2_gnome_max_main` | `100` | 整数 | 无上下界 | 治疗的主生命上限 | 服务器端本插件逻辑 | `gnome.sp:188` |
| `l4d2_gnome_max_temp` | `100.0` | 整数，源码写作浮点 | 无上下界 | 治疗的临时生命上限 | 服务器端本插件逻辑 | `gnome.sp:189` |
| `l4d2_gnome_modes` | `` | 字符串或表达式 | 无上下界 | 在这些游戏模式中启用插件，逗号分隔，留空为全部模式 | 服务器端本插件逻辑 | `gnome.sp:190` |
| `l4d2_gnome_modes_off` | `` | 字符串或表达式 | 无上下界 | 在这些游戏模式中关闭插件，逗号分隔，留空为不关闭 | 服务器端本插件逻辑 | `gnome.sp:191` |
| `l4d2_gnome_modes_tog` | `0` | 整数 | 无上下界 | 按模式位掩码启用插件：0 全部，1 战役，2 生存，4 对战，8 清道夫，可相加 | 服务器端本插件逻辑 | `gnome.sp:192` |
| `l4d2_gnome_random` | `0` | 整数 | 无上下界 | 从地图配置中随机刷出的地精数量：-1 全部，0 不刷 | 服务器端本插件逻辑 | `gnome.sp:193` |
| `l4d2_gnome_safe` | `0` | 整数 | 无上下界 | 回合开始刷出地精的位置：0 关，1 安全区内，2 装备给随机玩家 | 服务器端本插件逻辑 | `gnome.sp:194` |
| `l4d2_gnome_temp` | `-1` | 整数 | 无上下界 | -1 给临时生命，0 加到主生命；1~100 表示给临时生命的概率，否则加主生命 | 服务器端本插件逻辑 | `gnome.sp:195` |
| `l4d2_gnome_healing_field` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否治疗地精持有者周围的玩家：0 关，1 开 | 服务器端本插件逻辑 | `gnome.sp:196` |
| `l4d2_gnome_healing_field_refresh_time` | `2.0` | 整数，源码写作浮点 | ≥ 0.0 | 治疗力场重新触发治疗与光柱的间隔秒数，0 关闭 | 服务器端本插件逻辑 | `gnome.sp:197` |
| `l4d2_gnome_healing_field_heal_amount` | `1` | 整数，源码写作浮点 | ≥ 0.0 | 在治疗力场内每次的治疗量 | 服务器端本插件逻辑 | `gnome.sp:198` |
| `l4d2_gnome_healing_field_heal_amount_incap` | `0` | 整数，源码写作浮点 | ≥ 0.0 | 倒地玩家在治疗力场内的治疗量 | 服务器端本插件逻辑 | `gnome.sp:199` |
| `l4d2_gnome_healing_field_self` | `1` | 整数 | 无上下界 | 0 只治疗他人，1 同时治疗自己与他人 | 服务器端本插件逻辑 | `gnome.sp:200` |
| `l4d2_gnome_healing_field_heal_beacon` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否生成治疗力场光柱：0 关，1 开 | 服务器端本插件逻辑 | `gnome.sp:201` |
| `l4d2_gnome_healing_field_color` | `0 255 0` | 字符串或表达式 | 无上下界 | 治疗力场颜色，三个 0~255 的 RGB 值以空格分隔，填 random 则随机生成 | 服务器端本插件逻辑 | `gnome.sp:202` |
| `l4d2_gnome_healing_field_start_radius` | `100.0` | 整数，源码写作浮点 | ≥ 0.0 | 治疗力场起始半径 | 服务器端本插件逻辑 | `gnome.sp:203` |
| `l4d2_gnome_healing_field_end_radius` | `350.0` | 整数，源码写作浮点 | ≥ 0.0 | 治疗力场结束半径，同时决定治疗周围玩家的最大距离 | 服务器端本插件逻辑 | `gnome.sp:204` |
| `l4d2_gnome_healing_field_duration` | `1.0` | 整数，源码写作浮点 | ≥ 0.0 | 治疗力场持续秒数 | 服务器端本插件逻辑 | `gnome.sp:205` |
| `l4d2_gnome_healing_field_width` | `3.0` | 整数，源码写作浮点 | ≥ 0.0 | 治疗力场宽度 | 服务器端本插件逻辑 | `gnome.sp:206` |
| `l4d2_gnome_healing_field_amplitude` | `0.0` | 整数，源码写作浮点 | ≥ 0.0 | 治疗力场振幅 | 服务器端本插件逻辑 | `gnome.sp:207` |
| `l4d2_gnome_version` | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | 服务器端本插件逻辑；不写入 cfg 存档 | `gnome.sp:208` |

## hextags —— `extend/hextags.sp`

- 源文件：`extend/hextags.sp`
- myinfo：（无 version/author）
- myinfo description（源码原文）：Edit Tags & Colors!

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_reloadtags` | 无参数 | 需要 ADMFLAG_GENERIC（b，通用管理员） | 重新加载 HexTags 配置 | `hextags.sp:146` |
| `sm_toggletags` | 无参数 | 需要 ADMFLAG_GENERIC（b，通用管理员） | 切换自己的称号是否可见 | `hextags.sp:147` |
| `sm_anonymous` | 无参数 | 需要 ADMFLAG_GENERIC（b，通用管理员） | 切换匿名模式，忽略 SteamID 与管理员组、权限相关的称号 | `hextags.sp:148` |
| `sm_cz` | 无参数 | 无（任意玩家可用） | 重新加载 HexTags 配置，sm_reloadtags 的别名 | `hextags.sp:149` |
| `sm_tagslist` | 无参数 | 无（任意玩家可用） | 打开称号选择菜单 | `hextags.sp:150` |
| `sm_chenghao` | 无参数 | 无（任意玩家可用） | 打开称号选择菜单，sm_tagslist 的别名 | `hextags.sp:151` |
| `sm_ch` | 无参数 | 无（任意玩家可用） | 打开称号选择菜单，短别名 | `hextags.sp:152` |
| `sm_getteam` | 无参数 | 无（任意玩家可用） | 显示当前队伍名称 | `hextags.sp:153` |
| `sm_gettagvars` | 无参数 | 无（任意玩家可用） | 源码未给描述；据 `extend/hextags.sp:167-170` 该命令注册在 `#if defined DEBUG` 块内，回调 `Cmd_GetVars`（:460-467）连续四次 `ReplyToCommand` 输出 `selectedTags[client]` 的 `ScoreTag`、`ChatTag`、`ChatColor`、`NameColor`，推测为：调试命令，把调用者当前选中的计分板标签、聊天前缀、聊天颜色、名字颜色四个变量值回显到控制台，用于排查标签为何不生效；:22 的 `//#define DEBUG 0` 仍处于被注释状态，故默认编译不会注册（置信度：高） | `hextags.sp:168` |
| `sm_firesel` | 无参数 | 无（任意玩家可用） | 触发称号选择相关前向，源码仅给出触发参数 thistoggle | `hextags.sp:169` |

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `sm_hextags_version` | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | 服务器端本插件逻辑；复制到客户端；仅服务器 | `hextags.sp:139` |
| `sm_hextags_roundend` | `0` | 整数 | 无上下界 | 为 1 时回合结束也重新加载称号 | 服务器端本插件逻辑 | `hextags.sp:140` |
| `sm_hextags_enable_tagslist` | `1` | 整数 | 无上下界 | 为 1 时启用 sm_tagslist 命令 | 服务器端本插件逻辑 | `hextags.sp:141` |

## HexTags Lite —— `extend/hextags_lite.sp`

- 源文件：`extend/hextags_lite.sp`
- myinfo：author=东 (Simplified)
- myinfo description（源码原文）：轻量级称号插件 - 适配veterans和rpg

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_reloadtags` | 无参数 | 需要 ADMFLAG_GENERIC（b，通用管理员） | 重新加载称号配置，HexTags Lite 版本 | `hextags_lite.sp:59` |
| `sm_tagslist` | 无参数 | 无（任意玩家可用） | 打开称号选择菜单 | `hextags_lite.sp:60` |
| `sm_chenghao` | 无参数 | 无（任意玩家可用） | 打开称号选择菜单，sm_tagslist 的别名 | `hextags_lite.sp:61` |
| `sm_ch` | 无参数 | 无（任意玩家可用） | 打开称号选择菜单，短别名 | `hextags_lite.sp:62` |
| `sm_toggletags` | 无参数 | 无（任意玩家可用） | 切换自己的称号显示 | `hextags_lite.sp:63` |

### ConVar

（本插件未注册 ConVar）

## hp_tank_show.sp（myinfo 缺 name，用文件名代替） —— `extend/hp_tank_show.sp`

- 源文件：`extend/hp_tank_show.sp`
- myinfo：源码中未找到 myinfo 块

### 指令

（本插件未注册指令）

### ConVar

（本插件未注册 ConVar）

## simple join —— `extend/join.sp`

- 源文件：`extend/join.sp`
- myinfo：version=1.7，author=东
- myinfo description（源码原文）：A plugin designed CompetitiveWithAnne package change player team.

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_away` | 无参数 | 无（任意玩家可用） | 把自己转为旁观挂机状态，被控制时会被拒绝 | `join.sp:115` |
| `sm_afk` | 无参数 | 无（任意玩家可用） | sm_away 的别名 | `join.sp:116` |
| `sm_spec` | 无参数 | 无（任意玩家可用） | sm_away 的别名 | `join.sp:117` |
| `sm_s` | 无参数 | 无（任意玩家可用） | sm_away 的短别名 | `join.sp:118` |
| `sm_joininfected` | 无参数 | 无（任意玩家可用） | 加入特感队伍，受内鬼模式与人数上限限制 | `join.sp:119` |
| `sm_team3` | 无参数 | 无（任意玩家可用） | 加入特感队伍，sm_joininfected 的别名 | `join.sp:120` |
| `sm_inf` | 无参数 | 无（任意玩家可用） | 加入特感队伍，短别名 | `join.sp:121` |
| `sm_infected` | 无参数 | 无（任意玩家可用） | 加入特感队伍，sm_joininfected 的别名 | `join.sp:122` |
| `sm_zombie` | 无参数 | 无（任意玩家可用） | 加入特感队伍，sm_joininfected 的别名 | `join.sp:123` |
| `sm_join` | 无参数 | 无（任意玩家可用） | 加入生还者队伍，受人数上限限制 | `join.sp:124` |
| `sm_jg` | 无参数 | 无（任意玩家可用） | 加入生还者队伍，短别名 | `join.sp:125` |
| `sm_team2` | 无参数 | 无（任意玩家可用） | 加入生还者队伍，sm_join 的别名 | `join.sp:126` |
| `sm_joingame` | 无参数 | 无（任意玩家可用） | 加入生还者队伍，sm_join 的别名 | `join.sp:127` |
| `sm_survivor` | 无参数 | 无（任意玩家可用） | 加入生还者队伍，sm_join 的别名 | `join.sp:128` |
| `sm_ip` | 无参数 | 无（任意玩家可用） | 打开服务器 IP 与 MOTD 页面 | `join.sp:132` |
| `sm_web` | 无参数 | 无（任意玩家可用） | 打开按客户端语言本地化的 MOTD 页面 | `join.sp:133` |
| `sm_restartmap` | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 立即重启地图 | `join.sp:135` |

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `join_enable_inf` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否允许玩家加入特感 | 服务器端本插件逻辑 | `join.sp:96` |
| `join_enable_kickfamilyaccount` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否踢出家庭共享账户 | 服务器端本插件逻辑 | `join.sp:97` |
| `join_autoupdate` | `0` | 整数，源码写作浮点 | 0.0 ~ 2.0 | 自动更新模式：0 关闭，1 无数据库插件，2 含数据库插件 | 服务器端本插件逻辑 | `join.sp:98` |
| `join_autoupdate_public_url` | `UPDATE_URL_PUBLIC` | 字符串或表达式 | 无上下界 | 自动更新为 1 时使用的无数据库更新清单 URL | 服务器端本插件逻辑 | `join.sp:99` |
| `join_autoupdate_private_url` | `UPDATE_URL_PRIVATE` | 字符串或表达式 | 无上下界 | 自动更新为 2 时使用的数据库更新清单 URL | 服务器端本插件逻辑 | `join.sp:100` |
| `sm_cfgmotd_title` | `AnneHappy电信服` | 字符串或表达式 | 无上下界 | MOTD 标题 | 服务器端本插件逻辑 | `join.sp:101` |
| `sm_cfgmotd_url` | `http://anne.trygek.com/l4d2/` | 字符串或表达式 | 无上下界 | MOTD 页面 URL | 服务器端本插件逻辑 | `join.sp:102` |
| `sm_cfgmotd_language_redirect` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否按客户端语言为 !web/MOTD 添加 lang 参数或使用语言专用 URL | 服务器端本插件逻辑 | `join.sp:103` |
| `sm_cfgmotd_url_en` | `` | 字符串或表达式 | 无上下界 | 英语客户端专用 MOTD URL，留空则用 sm_cfgmotd_url | 服务器端本插件逻辑 | `join.sp:104` |
| `sm_cfgmotd_url_chi` | `` | 字符串或表达式 | 无上下界 | 简体中文客户端专用 MOTD URL，留空则用 sm_cfgmotd_url | 服务器端本插件逻辑 | `join.sp:105` |
| `sm_cfgmotd_url_zho` | `` | 字符串或表达式 | 无上下界 | 繁体中文客户端专用 MOTD URL，留空则用 sm_cfgmotd_url | 服务器端本插件逻辑 | `join.sp:106` |
| `sm_cfgmotd_url_jp` | `` | 字符串或表达式 | 无上下界 | 日语客户端专用 MOTD URL，留空则用 sm_cfgmotd_url | 服务器端本插件逻辑 | `join.sp:107` |
| `sm_cfgmotd_url_ko` | `` | 字符串或表达式 | 无上下界 | 韩语客户端专用 MOTD URL，留空则用 sm_cfgmotd_url | 服务器端本插件逻辑 | `join.sp:108` |
| `sm_new_player_guide_url` | `http://anne.trygek.com/l4d2/guide` | 字符串或表达式 | 无上下界 | 新玩家进服自动打开的玩法指南 URL，留空则用 sm_cfgmotd_url | 服务器端本插件逻辑 | `join.sp:109` |
| `sm_cfgip_url` | `http://anne.trygek.com/ip.php` | 字符串或表达式 | 无上下界 | 查询 IP 的页面 URL | 服务器端本插件逻辑 | `join.sp:110` |

## l4d2_blacklist.sp（myinfo 缺 name，用文件名代替） —— `extend/l4d2_blacklist.sp`

- 源文件：`extend/l4d2_blacklist.sp`
- myinfo：（无 version/author）
- HookConVarChange：`gCvarTableName`→`OnCvarChanged`、`gCvarKickMsg`→`OnCvarChanged`、`gCvarDBSection`→`OnCvarChanged`、`gCvarLimitUser`→`OnCvarChanged`、`gCvarLimitAdmin`→`OnCvarChanged`、`gCvarAdminFlag`→`OnCvarChanged`、`gCvarConsiderTeams`→`OnCvarChanged`、`gCvarImmuneFlag`→`OnCvarChanged`、`gCvarQuiet`→`OnCvarChanged`、`gCvarLogEnable`→`OnCvarChanged`、`gCvarLogFile`→`OnCvarChanged`、`gCvarExposeBlocker`→`OnCvarChanged`、`gCvarUseI18NKick`→`OnCvarChanged`、`gCvarMutual`→`OnCvarChanged`

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_block` | <目标玩家或 steam64 或名字> | 无（任意玩家可用） | 把目标玩家加入自己的黑名单 | `l4d2_blacklist.sp:236` |
| `sm_unblock` | <目标玩家或 steam64 或名字> | 无（任意玩家可用） | 把目标玩家从黑名单移除 | `l4d2_blacklist.sp:237` |
| `sm_blocklist` | [目标玩家或 steam64 或名字] | 无（任意玩家可用） | 查看自己或指定玩家的屏蔽列表 | `l4d2_blacklist.sp:238` |
| `sm_blocklimit` | 无参数 | 无（任意玩家可用） | 查看自己的屏蔽上限 | `l4d2_blacklist.sp:239` |
| `sm_blmenu` | 无参数 | 无（任意玩家可用） | 打开黑名单主菜单 | `l4d2_blacklist.sp:240` |

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `sm_bl_enable` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用插件：1 开，0 关 | 服务器端本插件逻辑 | `l4d2_blacklist.sp:242` |
| `sm_bl_db_section` | `g_sDBSection` | 字符串或表达式 | 无上下界 | databases.cfg 区块名 | 服务器端本插件逻辑 | `l4d2_blacklist.sp:243` |
| `sm_bl_table` | `g_sTable` | 字符串或表达式 | 无上下界 | 数据库表名 | 服务器端本插件逻辑 | `l4d2_blacklist.sp:244` |
| `sm_bl_kick_msg` | `g_sKickMsg` | 字符串或表达式 | 无上下界 | 踢出提示，留空则使用翻译短语 | 服务器端本插件逻辑 | `l4d2_blacklist.sp:245` |
| `sm_bl_limit_user` | `3` | 整数，源码写作浮点 | ≥ 0.0 | 普通玩家屏蔽上限 | 服务器端本插件逻辑 | `l4d2_blacklist.sp:246` |
| `sm_bl_limit_admin` | `10` | 整数，源码写作浮点 | ≥ 0.0 | 管理员屏蔽上限 | 服务器端本插件逻辑 | `l4d2_blacklist.sp:247` |
| `sm_bl_admin_flag` | `b` | 字符串或表达式 | 无上下界 | 享管理员上限所需的权限 flag，留空表示无 | 服务器端本插件逻辑 | `l4d2_blacklist.sp:248` |
| `sm_bl_consider_teams` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否仅检查队伍 2/3：1 是，0 否 | 服务器端本插件逻辑 | `l4d2_blacklist.sp:249` |
| `sm_bl_immune_flag` | `z` | 字符串或表达式 | 无上下界 | 免疫 flag，拥有该 flag 的玩家被忽略屏蔽检查，留空则关闭 | 服务器端本插件逻辑 | `l4d2_blacklist.sp:251` |
| `sm_bl_quiet` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 静默模式：1 不通知触发屏蔽的玩家，0 通知 | 服务器端本插件逻辑 | `l4d2_blacklist.sp:252` |
| `sm_bl_log` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用日志记录：1 开，0 关 | 服务器端本插件逻辑 | `l4d2_blacklist.sp:253` |
| `sm_bl_log_file` | `g_sLogFile` | 字符串或表达式 | 无上下界 | 日志文件名，位于 logs/ 下 | 服务器端本插件逻辑 | `l4d2_blacklist.sp:254` |
| `sm_bl_expose_blocker` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 踢出提示是否包含屏蔽者信息：1 是，0 否 | 服务器端本插件逻辑 | `l4d2_blacklist.sp:255` |
| `sm_bl_use_i18n_kick` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 踢出提示是否使用翻译短语：1 是，0 则优先用 cvar 文本 | 服务器端本插件逻辑 | `l4d2_blacklist.sp:256` |
| `sm_bl_mutual` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用双向不共存：加入者若屏蔽了在场玩家也禁止加入 | 服务器端本插件逻辑 | `l4d2_blacklist.sp:259` |

## [L4D2] Damage HUD (MySQL+Cookie, DB-first, fixes) —— `extend/l4d2_damage_show.sp`

- 源文件：`extend/l4d2_damage_show.sp`
- myinfo：author=Loqi + you (mod by ChatGPT)
- myinfo description（源码原文）：Per-client damage digits with DB-first persistence + cookies + menus + admin sharing gate

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_dmgmenu` | 无参数 | 无（任意玩家可用） | 打开伤害数字设置菜单 | `l4d2_damage_show.sp:1086` |
| `sm_dmgcookie` | 无参数 | 需要 Admin_Generic | 把玩家的 Cookie 与当前设置打印到控制台 | `l4d2_damage_show.sp:1087` |
| `sm_dmgdbstat` | 无参数 | 需要 Admin_Generic | 在控制台显示数据库状态与会话字符集 | `l4d2_damage_show.sp:1088` |
| `sm_dmgforcesavecookie` | 无参数 | 需要 Admin_Generic | 强制把当前设置保存到 Cookie | `l4d2_damage_show.sp:1089` |
| `sm_dmgdbprobe` | 无参数 | 需要 Admin_Generic | 执行一次同步数据库连接探测 | `l4d2_damage_show.sp:1090` |

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `sm_dmg_allowed_flags` | `` | 字符串或表达式 | 无上下界 | 允许使用本插件的管理员 flag，留空为所有人，例 bc | 服务器端本插件逻辑 | `l4d2_damage_show.sp:1092` |

## L4D2 Saferoom Locker —— `extend/l4d2_door_lock.sp`

- 源文件：`extend/l4d2_door_lock.sp`
- myinfo：author=alasfourom
- myinfo description（源码原文）：Lock Saferoom Door Until All Players Are Ready

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_lock` | 无参数 | 需要 ADMFLAG_GENERIC（b，通用管理员） | 强制上锁安全室的门 | `l4d2_door_lock.sp:219` |
| `sm_unlock` | 无参数 | 需要 ADMFLAG_GENERIC（b，通用管理员） | 强制解锁安全室的门 | `l4d2_door_lock.sp:220` |
| `sm_ready` | 无参数 | 无（任意玩家可用） | 把自己设为已准备状态 | `l4d2_door_lock.sp:221` |
| `sm_unready` | 无参数 | 无（任意玩家可用） | 把自己设为未准备状态 | `l4d2_door_lock.sp:222` |

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `l4d2_door_lock_version` | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | 服务器端本插件逻辑；仅服务器；不写入 cfg 存档 | `l4d2_door_lock.sp:196` |
| `l4d2_doorlock_plugin_enable` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 为 1 时启用插件 | 服务器端本插件逻辑 | `l4d2_door_lock.sp:197` |
| `l4d2_doorlock_game_mode` | `versus,coop` | 字符串或表达式 | 无上下界 | 在这些模式中启用插件，逗号分隔，留空为全模式 | 服务器端本插件逻辑 | `l4d2_door_lock.sp:198` |
| `l4d2_doorlock_countdown` | `0` | 整数 | 无上下界 | 解锁安全区门的倒计时秒数 | 服务器端本插件逻辑 | `l4d2_door_lock.sp:199` |
| `l4d2_doorlock_loaders_time` | `40` | 整数 | 无上下界 | 等待玩家加载的最长秒数 | 服务器端本插件逻辑 | `l4d2_door_lock.sp:200` |
| `l4d2_doorlock_glow_enable` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 为 1 时为安全室的门设置发光效果 | 服务器端本插件逻辑 | `l4d2_door_lock.sp:201` |
| `l4d2_doorlock_glow_range` | `500` | 整数 | 无上下界 | 安全门的发光范围 | 服务器端本插件逻辑 | `l4d2_door_lock.sp:202` |
| `l4d2_doorlock_lock_glow_color` | `255 0 0` | 字符串或表达式 | 无上下界 | 上锁安全门的发光颜色，0-255 RGB 以空格分隔 | 服务器端本插件逻辑 | `l4d2_door_lock.sp:203` |
| `l4d2_doorlock_unlock_glow_color` | `0 255 0` | 字符串或表达式 | 无上下界 | 解锁安全门的发光颜色，0-255 RGB 以空格分隔 | 服务器端本插件逻辑 | `l4d2_door_lock.sp:204` |
| `l4d2_doorlock_enable_ready_mode` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 为 1 时启用准备（ReadyUp）功能 | 服务器端本插件逻辑 | `l4d2_door_lock.sp:205` |
| `l4d2_doorlock_unready_counts` | `2` | 整数 | 无上下界 | 每轮允许玩家使用未准备的次数 | 服务器端本插件逻辑 | `l4d2_door_lock.sp:206` |
| `l4d2_doorlock_readyup_time` | `120` | 整数 | 无上下界 | 对未准备的团队等待多久后强制开始，单位秒 | 服务器端本插件逻辑 | `l4d2_door_lock.sp:207` |
| `l4d2_doorlock_readyup_percent` | `75.0` | 整数，源码写作浮点 | 无上下界 | 一方开始游戏所需的最低准备百分比 | 服务器端本插件逻辑 | `l4d2_door_lock.sp:208` |
| `l4d2_doorlock_readyup_notify` | `1` | 整数，源码写作浮点 | 0.0 ~ 3.0 | 团队准备就绪时的提示方式：0 禁用，1 聊天，2 中心文本，3 两者 | 服务器端本插件逻辑 | `l4d2_door_lock.sp:209` |
| `l4d2_doorlock_leavers_notify` | `2` | 整数，源码写作浮点 | 0.0 ~ 1.0 | 被传送时给离开者的提示：0 禁用，1 聊天，2 中心文本 | 服务器端本插件逻辑 | `l4d2_door_lock.sp:210` |
| `l4d2_doorlock_add_commands` | `!map,!buy,!shop` | 字符串或表达式 | 无上下界 | 需要屏蔽以免干扰 ReadyUp 面板的命令列表，无空格分隔 | 服务器端本插件逻辑 | `l4d2_door_lock.sp:211` |
| `l4d2_doorlock_freeze_survivor_bots` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 为 1 时锁门期或第一关真人离开前只冻结生还者 Bot，真人接管后自动解冻；0 保留 nb_player_stop 停止全部 Bot | 服务器端本插件逻辑 | `l4d2_door_lock.sp:212` |

## L4D2 Hit/Kill Feedback Plus —— `extend/l4d2_hitsound.sp`

- 源文件：`extend/l4d2_hitsound.sp`
- myinfo：author=TsukasaSato , Hesh233 (branch) , merged/updated by ChatGPT
- myinfo description（源码原文）：去重公共音效库、四类任意选音与三类图标偏好入库
- HookConVarChange：`cv_showtime`→`ConVarChanged_OverlayShowTime`、`cv_enable`→`ConVarChanged_FeedbackEnable`、`cv_pic_enable`→`ConVarChanged_FeedbackEnable`

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_snd` | 无参数 | 无（任意玩家可用） | 打开命中与击杀反馈主菜单，含音效、图标与范围设置 | `l4d2_hitsound.sp:773` |
| `sm_hitsound_reload` | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 重新从数据库与 KV 读取所有在线玩家的偏好 | `l4d2_hitsound.sp:774` |

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `l4d2_hitsound_plus_ver` | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | 服务器端本插件逻辑 | `l4d2_hitsound.sp:714` |
| `sm_hitsound_enable` | `1` | 整数 | 无上下界 | 是否开启本插件：0 关，1 开 | 服务器端本插件逻辑 | `l4d2_hitsound.sp:716` |
| `sm_hitsound_sound_enable` | `1` | 整数 | 无上下界 | 是否开启音效：0 关，1 开 | 服务器端本插件逻辑 | `l4d2_hitsound.sp:717` |
| `sm_hitsound_pic_enable` | `1` | 整数 | 无上下界 | 是否开启覆盖图标总开关：0 关，1 开 | 服务器端本插件逻辑 | `l4d2_hitsound.sp:718` |
| `sm_blast_damage_enable` | `0` | 整数 | 无上下界 | 是否开启爆炸反馈提示：0 关，1 开，源码建议关闭 | 服务器端本插件逻辑 | `l4d2_hitsound.sp:719` |
| `sm_hitsound_showtime` | `0.3` | 浮点 | 无上下界 | 覆盖图标显示时长秒数 | 服务器端本插件逻辑 | `l4d2_hitsound.sp:720` |
| `sm_hitsound_overlay_default` | `1` | 整数 | 无上下界 | 新玩家默认是否启用覆盖图：1 给套装 1，0 禁用 | 服务器端本插件逻辑 | `l4d2_hitsound.sp:721` |
| `sm_hitsound_db_enable` | `1` | 整数 | 无上下界 | 是否启用 RPG 表存储，运行时修改需重载插件 | 服务器端本插件逻辑 | `l4d2_hitsound.sp:723` |
| `sm_hitsound_db_conf` | `rpg` | 字符串或表达式 | 无上下界 | databases.cfg 中的连接名，运行时修改需重载插件 | 服务器端本插件逻辑 | `l4d2_hitsound.sp:724` |
| `sm_hitsound_db_table` | `RPG` | 字符串或表达式 | 无上下界 | 存储表名，运行时修改需重载插件 | 服务器端本插件逻辑 | `l4d2_hitsound.sp:725` |
| `sm_hitsound_debug` | `0` | 整数 | 无上下界 | 调试输出：0 关，1 开 | 服务器端本插件逻辑 | `l4d2_hitsound.sp:726` |

## L4D2 Item hint —— `extend/l4d2_item_hint.sp`

- 源文件：`extend/l4d2_item_hint.sp`
- myinfo：version=2.0，author=BHaType, fdxx, HarryPotter
- myinfo description（源码原文）：When using 'Look' in vocalize menu, print corresponding item to chat area and make item glow or create spot marker/infeced maker like back 4 blood.

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `l4d2_item_hint_cooldown_time` | `1.0` | 整数，源码写作浮点 | ≥ 0.0 | 玩家再次使用 Look 物品提示的冷却秒数 | 服务器端本插件逻辑 | `l4d2_item_hint.sp:132` |
| `l4d2_item_hint_use_range` | `150` | 整数，源码写作浮点 | ≥ 1.0 | 玩家可使用 Look 物品提示的距离 | 服务器端本插件逻辑 | `l4d2_item_hint.sp:133` |
| `l4d2_item_hint_use_sound` | `` | 字符串或表达式 | 无上下界 | 物品提示音效路径，留空为关闭 | 服务器端本插件逻辑 | `l4d2_item_hint.sp:134` |
| `l4d2_item_hint_announce_type` | `0` | 整数，源码写作浮点 | 0.0 ~ 3.0 | 物品提示显示方式：0 禁用，1 聊天，2 提示框，3 中心文本 | 服务器端本插件逻辑 | `l4d2_item_hint.sp:135` |
| `l4d2_item_hint_glow_timer` | `30.0` | 整数，源码写作浮点 | ≥ 0.0 | 物品发光持续时间 | 服务器端本插件逻辑 | `l4d2_item_hint.sp:136` |
| `l4d2_item_hint_glow_range` | `800` | 整数，源码写作浮点 | ≥ 0.0 | 物品发光范围 | 服务器端本插件逻辑 | `l4d2_item_hint.sp:137` |
| `l4d2_item_hint_glow_color` | `0 255 255` | 字符串或表达式 | 无上下界 | 物品发光颜色，0-255 RGB 以空格分隔，留空禁用物品发光 | 服务器端本插件逻辑 | `l4d2_item_hint.sp:138` |
| `l4d2_item_instructorhint_enable` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 为 1 时在标记的物品上创建教学提示 | 服务器端本插件逻辑 | `l4d2_item_hint.sp:139` |
| `l4d2_item_instructorhint_color` | `0 255 255` | 字符串或表达式 | 无上下界 | 标记物品的教学提示颜色 | 服务器端本插件逻辑 | `l4d2_item_hint.sp:140` |
| `l4d2_item_instructorhint_icon` | `icon_interact` | 字符串或表达式 | 无上下界 | 标记物品的教学提示图标名 | 服务器端本插件逻辑 | `l4d2_item_hint.sp:141` |
| `l4d2_spot_marker_cooldown_time` | `2.5` | 浮点 | ≥ 0.0 | 玩家再次使用 Look 标记点的冷却秒数 | 服务器端本插件逻辑 | `l4d2_item_hint.sp:143` |
| `l4d2_spot_marker_use_range` | `1800` | 整数，源码写作浮点 | ≥ 1.0 | 玩家可使用 Look 标记点的距离 | 服务器端本插件逻辑 | `l4d2_item_hint.sp:144` |
| `l4d2_spot_marker_use_sound` | `buttons/blip1.wav` | 字符串或表达式 | 无上下界 | 标记点音效路径，留空为关闭 | 服务器端本插件逻辑 | `l4d2_item_hint.sp:145` |
| `l4d2_spot_marker_announce_type` | `1` | 整数，源码写作浮点 | 0.0 ~ 3.0 | 标记点提示显示方式：0 禁用，1 聊天，2 提示框，3 中心文本 | 服务器端本插件逻辑 | `l4d2_item_hint.sp:146` |
| `l4d2_spot_marker_duration` | `15.0` | 整数，源码写作浮点 | ≥ 0.0 | 标记点持续时间 | 服务器端本插件逻辑 | `l4d2_item_hint.sp:147` |
| `l4d2_spot_marker_color` | `0 255 255` | 字符串或表达式 | 无上下界 | 标记点发光颜色，留空禁用标记点 | 服务器端本插件逻辑 | `l4d2_item_hint.sp:148` |
| `l4d2_spot_marker_sprite_model` | `materials/vgui/icon_arrow_down.vmt` | 字符串或表达式 | 无上下界 | 标记点精灵模型，留空禁用 | 服务器端本插件逻辑 | `l4d2_item_hint.sp:149` |
| `l4d2_spot_marker_instructorhint_enable` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 为 1 时为标记点创建教学提示 | 服务器端本插件逻辑 | `l4d2_item_hint.sp:150` |
| `l4d2_spot_marker_instructorhint_color` | `200 200 200` | 字符串或表达式 | 无上下界 | 标记点的教学提示颜色 | 服务器端本插件逻辑 | `l4d2_item_hint.sp:151` |
| `l4d2_spot_marker_instructorhint_icon` | `icon_info` | 字符串或表达式 | 无上下界 | 标记点的教学提示图标名 | 服务器端本插件逻辑 | `l4d2_item_hint.sp:152` |
| `l4d2_infected_marker_cooldown_time` | `0.25` | 浮点 | ≥ 0.0 | 玩家再次使用 Look 特感标记的冷却秒数 | 服务器端本插件逻辑 | `l4d2_item_hint.sp:154` |
| `l4d2_infected_marker_use_range` | `1800` | 整数，源码写作浮点 | ≥ 1.0 | 可使用 Look 特感标记的距离 | 服务器端本插件逻辑 | `l4d2_item_hint.sp:155` |
| `l4d2_infected_marker_use_sound` | `items/suitchargeok1.wav` | 字符串或表达式 | 无上下界 | 特感标记音效路径，留空为关闭 | 服务器端本插件逻辑 | `l4d2_item_hint.sp:156` |
| `l4d2_infected_marker_announce_type` | `1` | 整数，源码写作浮点 | 0.0 ~ 3.0 | 特感标记提示显示方式：0 禁用，1 聊天，2 提示框，3 中心文本 | 服务器端本插件逻辑 | `l4d2_item_hint.sp:157` |
| `l4d2_infected_marker_glow_timer` | `10.0` | 整数，源码写作浮点 | ≥ 0.0 | 特感标记发光持续时间 | 服务器端本插件逻辑 | `l4d2_item_hint.sp:158` |
| `l4d2_infected_marker_glow_range` | `2500` | 整数，源码写作浮点 | ≥ 0.0 | 特感标记发光范围 | 服务器端本插件逻辑 | `l4d2_item_hint.sp:159` |
| `l4d2_infected_marker_glow_color` | `` | 字符串或表达式 | 无上下界 | 特感标记发光颜色，留空禁用特感标记 | 服务器端本插件逻辑 | `l4d2_item_hint.sp:160` |
| `l4d2_infected_marker_witch_enable` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 为 1 时允许对 Witch 使用 Look 特感标记 | 服务器端本插件逻辑 | `l4d2_item_hint.sp:161` |

## L4D2 Native vote —— `extend/l4d2_nativevote.sp`

- 源文件：`extend/l4d2_nativevote.sp`
- myinfo：author=Powerlord, fdxx
- myinfo description（源码原文）：Voting API to use the game's native vote panels

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `l4d2_nativevote_version` | `VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | 服务器端本插件逻辑 | `l4d2_nativevote.sp:74` |
| `l4d2_nativevote_initiator_auto_voteyes` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 为 1 时发起者自动投赞成票 | 服务器端本插件逻辑 | `l4d2_nativevote.sp:75` |

## L4D2 Survivor Bot Fix —— `extend/l4d2_sb_fix.sp`

- 源文件：`extend/l4d2_sb_fix.sp`
- myinfo：version=1.01，author=DingbatFlat, HarryPotter
- myinfo description（源码原文）：Survivor Bot Fix. Improve Survivor Bot

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `sb_fix_enabled` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用插件：0 关，1 开 | 服务器端本插件逻辑 | `l4d2_sb_fix.sp:259` |
| `sb_fix_select_type` | `0` | 整数，源码写作浮点 | 0.0 ~ 2.0 | 优化哪些生还者 Bot：0 全部，1 离开安全区时随机选若干名，2 按指定角色名 | 服务器端本插件逻辑 | `l4d2_sb_fix.sp:261` |
| `sb_fix_select_number` | `1` | 整数，源码写作浮点 | ≥ 0.0 | 当 sb_fix_select_type 为 1 时，随机优化的 Bot 数量，0~4 | 服务器端本插件逻辑 | `l4d2_sb_fix.sp:262` |
| `sb_fix_select_character_name` | `` | 字符串或表达式 | 无上下界 | 当 sb_fix_select_type 为 4 时，指定要优化的角色名，空格分隔 | 服务器端本插件逻辑 | `l4d2_sb_fix.sp:263` |
| `sb_fix_dont_switch_secondary` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 在副武器弹药耗尽前不允许切换到副武器：0 否，1 是 | 服务器端本插件逻辑 | `l4d2_sb_fix.sp:265` |
| `sb_fix_help_enabled` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否帮助被控制的生还者：0 关，1 开 | 服务器端本插件逻辑 | `l4d2_sb_fix.sp:267` |
| `sb_fix_help_range` | `1200` | 整数，源码写作浮点 | 1.0 ~ 3000.0 | 搜索或射击被控制生还者的距离范围，1~3000 | 服务器端本插件逻辑 | `l4d2_sb_fix.sp:268` |
| `sb_fix_help_shove_type` | `2` | 整数，源码写作浮点 | 0.0 ~ 3.0 | 是否用推击帮助：0 不推，1 仅 Smoker，2 Smoker 与 Jockey，3 再加 Hunter | 服务器端本插件逻辑 | `l4d2_sb_fix.sp:269` |
| `sb_fix_help_shove_reloading` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 当 help_shove_type 为 2 及以上时是否仅在换弹时推击：0 否，1 是 | 服务器端本插件逻辑 | `l4d2_sb_fix.sp:270` |
| `sb_fix_ci_enabled` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否处理普通感染者：0 关，1 开 | 服务器端本插件逻辑 | `l4d2_sb_fix.sp:272` |
| `sb_fix_ci_range` | `500` | 整数，源码写作浮点 | 1.0 ~ 2000.0 | 搜索或射击普通感染者的距离范围，1~2000 | 服务器端本插件逻辑 | `l4d2_sb_fix.sp:273` |
| `sb_fix_ci_melee_allow` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否允许用近战武器处理：0 关，1 开 | 服务器端本插件逻辑 | `l4d2_sb_fix.sp:274` |
| `sb_fix_ci_melee_range` | `160` | 整数，源码写作浮点 | 1.0 ~ 500.0 | 近战处理普通感染者的距离范围，1~500 | 服务器端本插件逻辑 | `l4d2_sb_fix.sp:275` |
| `sb_fix_si_enabled` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否处理特殊感染者：0 关，1 开 | 服务器端本插件逻辑 | `l4d2_sb_fix.sp:277` |
| `sb_fix_si_range` | `500` | 整数，源码写作浮点 | 1.0 ~ 3000.0 | 搜索或射击特殊感染者的距离范围，1~3000 | 服务器端本插件逻辑 | `l4d2_sb_fix.sp:278` |
| `sb_fix_si_ignore_boomer` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否忽略生还者附近的 Boomer 并推开它：0 否，1 是 | 服务器端本插件逻辑 | `l4d2_sb_fix.sp:279` |
| `sb_fix_si_ignore_boomer_range` | `200` | 整数，源码写作浮点 | 1.0 ~ 500.0 | 忽略 Boomer 的距离范围，源码正文注明 1~900 | 服务器端本插件逻辑 | `l4d2_sb_fix.sp:280` |
| `sb_fix_tank_enabled` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否处理 Tank：0 关，1 开 | 服务器端本插件逻辑 | `l4d2_sb_fix.sp:282` |
| `sb_fix_tank_range` | `1200` | 整数，源码写作浮点 | 1.0 ~ 3000.0 | 搜索或射击 Tank 的距离范围，1~3000 | 服务器端本插件逻辑 | `l4d2_sb_fix.sp:283` |
| `sb_fix_si_tank_priority_type` | `0` | 整数，源码写作浮点 | 0.0 ~ 2.0 | 范围内同时有特感与 Tank 时优先目标：0 最近的，1 特感优先，其余源码描述被截断 | 服务器端本插件逻辑 | `l4d2_sb_fix.sp:285` |
| `sb_fix_bash_enabled` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否推击飞扑中的 Hunter 或 Jockey：0 关，1 开 | 服务器端本插件逻辑 | `l4d2_sb_fix.sp:287` |
| `sb_fix_bash_hunter_chance` | `100` | 整数，源码写作浮点 | 0.0 ~ 100.0 | 推击飞扑中 Hunter 的概率百分比，1~100 | 服务器端本插件逻辑 | `l4d2_sb_fix.sp:288` |
| `sb_fix_bash_hunter_range` | `145` | 整数，源码写作浮点 | 1.0 ~ 500.0 | 推击或搜索飞扑中 Hunter 的距离范围，1~500 | 服务器端本插件逻辑 | `l4d2_sb_fix.sp:289` |
| `sb_fix_bash_jockey_chance` | `100` | 整数，源码写作浮点 | 0.0 ~ 100.0 | 推击飞扑中 Jockey 的概率百分比，1~100 | 服务器端本插件逻辑 | `l4d2_sb_fix.sp:290` |
| `sb_fix_bash_jockey_range` | `125` | 整数，源码写作浮点 | 1.0 ~ 500.0 | 推击或搜索飞扑中 Jockey 的距离范围，1~500 | 服务器端本插件逻辑 | `l4d2_sb_fix.sp:291` |
| `sb_fix_rock_enabled` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否射击 Tank 投掷的石头：0 关，1 开 | 服务器端本插件逻辑 | `l4d2_sb_fix.sp:293` |
| `sb_fix_rock_range` | `700` | 整数，源码写作浮点 | 1.0 ~ 2000.0 | 搜索或射击 Tank 石头的距离范围，1~2000 | 服务器端本插件逻辑 | `l4d2_sb_fix.sp:294` |
| `sb_fix_witch_enabled` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否射击被激怒的 Witch：0 关，1 开 | 服务器端本插件逻辑 | `l4d2_sb_fix.sp:296` |
| `sb_fix_witch_range` | `1500` | 整数，源码写作浮点 | 1.0 ~ 2000.0 | 搜索或射击被激怒 Witch 的距离范围，1~2000 | 服务器端本插件逻辑 | `l4d2_sb_fix.sp:297` |
| `sb_fix_witch_range_incapacitated` | `1000` | 整数，源码写作浮点 | 0.0 ~ 2000.0 | 搜索或射击使生还者倒地的 Witch 的距离范围，0~2000 | 服务器端本插件逻辑 | `l4d2_sb_fix.sp:298` |
| `sb_fix_witch_range_killed` | `0` | 整数，源码写作浮点 | 0.0 ~ 2000.0 | 搜索或射击杀死生还者的 Witch 的距离范围，0 表示不处理，0~2000 | 服务器端本插件逻辑 | `l4d2_sb_fix.sp:299` |
| `sb_fix_witch_shotgun_control` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 若持有霰弹枪，是否由插件控制对 Witch 的开火时机：0 关，1 开 | 服务器端本插件逻辑 | `l4d2_sb_fix.sp:300` |
| `sb_fix_witch_shotgun_range_max` | `300` | 整数，源码写作浮点 | 1.0 ~ 1000.0 | Witch 距离在该值以内则停止攻击，1~1000 | 服务器端本插件逻辑 | `l4d2_sb_fix.sp:301` |
| `sb_fix_witch_shotgun_range_min` | `70` | 整数，源码写作浮点 | 1.0 ~ 500.0 | Witch 距离在该值及以上则停止攻击，1~500 | 服务器端本插件逻辑 | `l4d2_sb_fix.sp:302` |
| `sb_fix_prioritize_ownersmoker` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否优先处理正在控制自己的 Smoker：0 否，1 是 | 服务器端本插件逻辑 | `l4d2_sb_fix.sp:304` |
| `sb_fix_incapacitated_enabled` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用倒地相关命令：0 关，1 开 | 服务器端本插件逻辑 | `l4d2_sb_fix.sp:306` |
| `sb_fix_debug` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 调试用：是否打印行动状态 | 服务器端本插件逻辑 | `l4d2_sb_fix.sp:308` |

## l4d2_scripted_hud.sp（myinfo 缺 name，用文件名代替） —— `extend/l4d2_scripted_hud.sp`

- 源文件：`extend/l4d2_scripted_hud.sp`
- myinfo：源码中未找到 myinfo 块

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_l4d2_scripted_hud_reload_data` | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 重新加载 HUD 文本数据文件 | `l4d2_scripted_hud.sp:896` |
| `sm_print_cvars_l4d2_scripted_hud` | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 把本插件的 ConVar 及其取值打印到控制台 | `l4d2_scripted_hud.sp:897` |
| `sm_spechudon` | 无参数 | 无（任意玩家可用） | 打开 spechud | `l4d2_scripted_hud.sp:898` |
| `sm_spechudoff` | 无参数 | 无（任意玩家可用） | 关闭 spechud | `l4d2_scripted_hud.sp:899` |
| `sm_hudmenu` | 无参数 | 无（任意玩家可用） | 打开脚本化 HUD 设置菜单 | `l4d2_scripted_hud.sp:900` |
| `sm_hud` | 无参数 | 无（任意玩家可用） | 打开脚本化 HUD 设置菜单，sm_hudmenu 的别名 | `l4d2_scripted_hud.sp:901` |
| `sm_hudlang` | 无参数 | 无（任意玩家可用） | 打开显示语言菜单 | `l4d2_scripted_hud.sp:902` |
| `sm_lang` | 无参数 | 无（任意玩家可用） | 打开显示语言菜单，sm_hudlang 的别名 | `l4d2_scripted_hud.sp:903` |

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `l4d2_scripted_hud_version` | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | 服务器端本插件逻辑；仅服务器；不写入 cfg 存档 | `l4d2_scripted_hud.sp:710` |
| `l4d2_scripted_hud_enable` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用插件：0 关，1 开 | 服务器端本插件逻辑 | `l4d2_scripted_hud.sp:711` |
| `l4d2_scripted_hud_update_interval` | `0.1` | 浮点 | ≥ 0.1 | HUD 刷新间隔秒数，最小 0.1 | 服务器端本插件逻辑 | `l4d2_scripted_hud.sp:712` |
| `l4d2_scripted_hud_source_teams` | `2:5,7:5,13:5` | 字符串或表达式 | 无上下界 | 按内容来源的队伍白名单，格式 source:teammask,...；掩码 1 旁观、2 生还、4 特感（其余被截断） | 服务器端本插件逻辑 | `l4d2_scripted_hud.sp:713` |
| `l4d2_scripted_hud_hud1_text` | `` | 字符串或表达式 | 无上下界 | 要在 HUD 上显示的文本，留空则使用插件内置的预定义文本 | 服务器端本插件逻辑 | `l4d2_scripted_hud.sp:714` |
| `l4d2_scripted_hud_hud1_text_align` | `1` | 整数，源码写作浮点 | 1.0 ~ 3.0 | 文本水平对齐：1 左，2 中，3 右 | 服务器端本插件逻辑 | `l4d2_scripted_hud.sp:715` |
| `l4d2_scripted_hud_hud1_blink_tank` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 有 Tank 存活时文本是否在白色与红色间闪烁：0 关，1 开 | 服务器端本插件逻辑 | `l4d2_scripted_hud.sp:716` |
| `l4d2_scripted_hud_hud1_blink` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 文本是否在白色与红色间闪烁：0 关，1 开 | 服务器端本插件逻辑 | `l4d2_scripted_hud.sp:717` |
| `l4d2_scripted_hud_hud1_beep` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 闪烁时是否播放提示音，需闪烁开关为 1 | 服务器端本插件逻辑 | `l4d2_scripted_hud.sp:718` |
| `l4d2_scripted_hud_hud1_visible` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 文本是否可见：0 关，1 开 | 服务器端本插件逻辑 | `l4d2_scripted_hud.sp:719` |
| `l4d2_scripted_hud_hud1_background` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 文本是否显示黑色半透明背景 | 服务器端本插件逻辑 | `l4d2_scripted_hud.sp:720` |
| `l4d2_scripted_hud_hud1_team` | `0` | 整数，源码写作浮点 | 0.0 ~ 2.0 | 哪些队伍能看到文本：0 全部，1 生还者，2 特感 | 服务器端本插件逻辑 | `l4d2_scripted_hud.sp:721` |
| `l4d2_scripted_hud_hud1_flag_debug` | `0` | 整数，源码写作浮点 | 0.0 ~ 32767.0 | 覆盖 HUD flag，仅调试用，0 关闭 | 服务器端本插件逻辑 | `l4d2_scripted_hud.sp:722` |
| `l4d2_scripted_hud_hud1_x` | `0.05` | 浮点 | -1.0 ~ 1.0 | 文本水平位置 -1.0~1.0，小于 0 可能被屏幕裁切 | 服务器端本插件逻辑 | `l4d2_scripted_hud.sp:723` |
| `l4d2_scripted_hud_hud1_y` | `0.0` | 整数，源码写作浮点 | -1.0 ~ 1.0 | 文本垂直位置 -1.0~1.0，小于 0 可能被屏幕裁切 | 服务器端本插件逻辑 | `l4d2_scripted_hud.sp:724` |
| `l4d2_scripted_hud_hud1_x_speed` | `0.0` | 整数，源码写作浮点 | 0.0 ~ 2.0 | 文本水平移动动画速度，0 关闭 | 服务器端本插件逻辑 | `l4d2_scripted_hud.sp:725` |
| `l4d2_scripted_hud_hud1_y_speed` | `0.0` | 整数，源码写作浮点 | 0.0 ~ 2.0 | 文本垂直移动动画速度，0 关闭 | 服务器端本插件逻辑 | `l4d2_scripted_hud.sp:726` |
| `l4d2_scripted_hud_hud1_x_direction` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 文本水平移动方向：0 从右到左，1 从左到右 | 服务器端本插件逻辑 | `l4d2_scripted_hud.sp:727` |
| `l4d2_scripted_hud_hud1_y_direction` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 文本垂直移动方向：0 从上到下，1 从下到上 | 服务器端本插件逻辑 | `l4d2_scripted_hud.sp:728` |
| `l4d2_scripted_hud_hud1_x_min` | `0.0` | 整数，源码写作浮点 | -1.0 ~ 1.0 | 动画时 HUD 可到达的水平最小位置 | 服务器端本插件逻辑 | `l4d2_scripted_hud.sp:729` |
| `l4d2_scripted_hud_hud1_y_min` | `0.0` | 整数，源码写作浮点 | -1.0 ~ 1.0 | 动画时 HUD 可到达的垂直最小位置 | 服务器端本插件逻辑 | `l4d2_scripted_hud.sp:730` |
| `l4d2_scripted_hud_hud1_x_max` | `1.0` | 整数，源码写作浮点 | -1.0 ~ 1.0 | 动画时 HUD 可到达的水平最大位置 | 服务器端本插件逻辑 | `l4d2_scripted_hud.sp:731` |
| `l4d2_scripted_hud_hud1_y_max` | `1.0` | 整数，源码写作浮点 | -1.0 ~ 1.0 | 动画时 HUD 可到达的垂直最大位置 | 服务器端本插件逻辑 | `l4d2_scripted_hud.sp:732` |
| `l4d2_scripted_hud_hud1_width` | `1.0` | 整数，源码写作浮点 | 0.0 ~ 2.0 | 文本区域宽度 | 服务器端本插件逻辑 | `l4d2_scripted_hud.sp:733` |
| `l4d2_scripted_hud_hud1_height` | `0.026` | 浮点 | 0.0 ~ 2.0 | 文本区域高度 | 服务器端本插件逻辑 | `l4d2_scripted_hud.sp:734` |
| `l4d2_scripted_hud_hud2_text` | `` | 字符串或表达式 | 无上下界 | HUD 槽 2 显示文本，留空则使用插件内置的预定义文本 | 服务器端本插件逻辑 | `l4d2_scripted_hud.sp:735` |
| `l4d2_scripted_hud_hud2_text_align` | `1` | 整数，源码写作浮点 | 1.0 ~ 3.0 | HUD 槽 2 水平对齐：1 左，2 中，3 右 | 服务器端本插件逻辑 | `l4d2_scripted_hud.sp:736` |
| `l4d2_scripted_hud_hud2_blink_tank` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | HUD 槽 2 在有 Tank 存活时是否白红闪烁：0 关，1 开 | 服务器端本插件逻辑 | `l4d2_scripted_hud.sp:737` |
| `l4d2_scripted_hud_hud2_blink` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | HUD 槽 2 文本是否白红闪烁：0 关，1 开 | 服务器端本插件逻辑 | `l4d2_scripted_hud.sp:738` |
| `l4d2_scripted_hud_hud2_beep` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | HUD 槽 2 闪烁时是否播放提示音，需闪烁开关为 1 | 服务器端本插件逻辑 | `l4d2_scripted_hud.sp:739` |
| `l4d2_scripted_hud_hud2_visible` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | HUD 槽 2 文本是否可见：0 关，1 开 | 服务器端本插件逻辑 | `l4d2_scripted_hud.sp:740` |
| `l4d2_scripted_hud_hud2_background` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | HUD 槽 2 文本是否显示黑色半透明背景 | 服务器端本插件逻辑 | `l4d2_scripted_hud.sp:741` |
| `l4d2_scripted_hud_hud2_team` | `0` | 整数，源码写作浮点 | 0.0 ~ 2.0 | 哪些队伍能看到 HUD 槽 2：0 全部，1 生还者，2 特感 | 服务器端本插件逻辑 | `l4d2_scripted_hud.sp:742` |
| `l4d2_scripted_hud_hud2_flag_debug` | `0` | 整数，源码写作浮点 | 0.0 ~ 32767.0 | 覆盖 HUD 槽 2 的 flag，仅调试用，0 关闭 | 服务器端本插件逻辑 | `l4d2_scripted_hud.sp:743` |
| `l4d2_scripted_hud_hud2_x` | `0.65` | 浮点 | -1.0 ~ 1.0 | HUD 槽 2 文本水平位置 -1.0~1.0，小于 0 可能被裁切 | 服务器端本插件逻辑 | `l4d2_scripted_hud.sp:744` |
| `l4d2_scripted_hud_hud2_y` | `0.00` | 整数，源码写作浮点 | -1.0 ~ 1.0 | HUD 槽 2 文本垂直位置 -1.0~1.0，小于 0 可能被裁切 | 服务器端本插件逻辑 | `l4d2_scripted_hud.sp:745` |
| `l4d2_scripted_hud_hud2_x_speed` | `0.0` | 整数，源码写作浮点 | 0.0 ~ 2.0 | HUD 槽 2 文本水平移动动画速度，0 关闭 | 服务器端本插件逻辑 | `l4d2_scripted_hud.sp:746` |
| `l4d2_scripted_hud_hud2_y_speed` | `0.0` | 整数，源码写作浮点 | 0.0 ~ 2.0 | HUD 槽 2 文本垂直移动动画速度，0 关闭 | 服务器端本插件逻辑 | `l4d2_scripted_hud.sp:747` |
| `l4d2_scripted_hud_hud2_x_direction` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | HUD 槽 2 文本水平移动方向：0 从左到右，1 从右到左 | 服务器端本插件逻辑 | `l4d2_scripted_hud.sp:748` |
| `l4d2_scripted_hud_hud2_y_direction` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | HUD 槽 2 文本垂直移动方向：0 从上到下，1 从下到上 | 服务器端本插件逻辑 | `l4d2_scripted_hud.sp:749` |
| `l4d2_scripted_hud_hud2_x_min` | `0.0` | 整数，源码写作浮点 | -1.0 ~ 1.0 | HUD 槽 2 动画可到达的水平最小位置 | 服务器端本插件逻辑 | `l4d2_scripted_hud.sp:750` |
| `l4d2_scripted_hud_hud2_y_min` | `0.0` | 整数，源码写作浮点 | -1.0 ~ 1.0 | HUD 槽 2 动画可到达的垂直最小位置 | 服务器端本插件逻辑 | `l4d2_scripted_hud.sp:751` |
| `l4d2_scripted_hud_hud2_x_max` | `1.0` | 整数，源码写作浮点 | -1.0 ~ 1.0 | HUD 槽 2 动画可到达的水平最大位置 | 服务器端本插件逻辑 | `l4d2_scripted_hud.sp:752` |
| `l4d2_scripted_hud_hud2_y_max` | `1.0` | 整数，源码写作浮点 | -1.0 ~ 1.0 | HUD 槽 2 动画可到达的垂直最大位置 | 服务器端本插件逻辑 | `l4d2_scripted_hud.sp:753` |
| `l4d2_scripted_hud_hud2_width` | `1.0` | 整数，源码写作浮点 | 0.0 ~ 2.0 | HUD 槽 2 文本区域宽度 | 服务器端本插件逻辑 | `l4d2_scripted_hud.sp:754` |
| `l4d2_scripted_hud_hud2_height` | `0.026` | 浮点 | 0.0 ~ 2.0 | HUD 槽 2 文本区域高度 | 服务器端本插件逻辑 | `l4d2_scripted_hud.sp:755` |
| `l4d2_scripted_hud_hud3_text` | `` | 字符串或表达式 | 无上下界 | HUD 槽 3 显示文本，留空则使用插件内置的预定义文本 | 服务器端本插件逻辑 | `l4d2_scripted_hud.sp:756` |
| `l4d2_scripted_hud_hud3_text_align` | `1` | 整数，源码写作浮点 | 1.0 ~ 3.0 | HUD 槽 3 水平对齐：1 左，2 中，3 右 | 服务器端本插件逻辑 | `l4d2_scripted_hud.sp:757` |
| `l4d2_scripted_hud_hud3_blink_tank` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | HUD 槽 3 在有 Tank 存活时是否白红闪烁：0 关，1 开 | 服务器端本插件逻辑 | `l4d2_scripted_hud.sp:758` |
| `l4d2_scripted_hud_hud3_blink` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | HUD 槽 3 文本是否白红闪烁：0 关，1 开 | 服务器端本插件逻辑 | `l4d2_scripted_hud.sp:759` |
| `l4d2_scripted_hud_hud3_beep` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | HUD 槽 3 闪烁时是否播放提示音，需闪烁开关为 1 | 服务器端本插件逻辑 | `l4d2_scripted_hud.sp:760` |
| `l4d2_scripted_hud_hud3_visible` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | HUD 槽 3 是否允许玩家显示该槽位：0 全局禁用，1 按玩家偏好 | 服务器端本插件逻辑 | `l4d2_scripted_hud.sp:761` |
| `l4d2_scripted_hud_hud3_background` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | HUD 槽 3 文本是否显示黑色半透明背景 | 服务器端本插件逻辑 | `l4d2_scripted_hud.sp:762` |
| `l4d2_scripted_hud_hud3_team` | `0` | 整数，源码写作浮点 | 0.0 ~ 2.0 | 哪些队伍能看到 HUD 槽 3：0 全部，1 生还者，2 特感 | 服务器端本插件逻辑 | `l4d2_scripted_hud.sp:763` |
| `l4d2_scripted_hud_hud3_flag_debug` | `0` | 整数，源码写作浮点 | 0.0 ~ 32767.0 | 覆盖 HUD 槽 3 的 flag，仅调试用，0 关闭 | 服务器端本插件逻辑 | `l4d2_scripted_hud.sp:764` |
| `l4d2_scripted_hud_hud3_x` | `0.8` | 浮点 | -1.0 ~ 1.0 | HUD 槽 3 文本水平位置 -1.0~1.0，小于 0 可能被裁切 | 服务器端本插件逻辑 | `l4d2_scripted_hud.sp:765` |
| `l4d2_scripted_hud_hud3_y` | `0.11` | 浮点 | -1.0 ~ 1.0 | HUD 槽 3 文本垂直位置 -1.0~1.0，小于 0 可能被裁切 | 服务器端本插件逻辑 | `l4d2_scripted_hud.sp:766` |
| `l4d2_scripted_hud_hud3_x_speed` | `0.0` | 整数，源码写作浮点 | 0.0 ~ 2.0 | HUD 槽 3 文本水平移动动画速度，0 关闭 | 服务器端本插件逻辑 | `l4d2_scripted_hud.sp:767` |
| `l4d2_scripted_hud_hud3_y_speed` | `0.0` | 整数，源码写作浮点 | 0.0 ~ 2.0 | HUD 槽 3 文本垂直移动动画速度，0 关闭 | 服务器端本插件逻辑 | `l4d2_scripted_hud.sp:768` |
| `l4d2_scripted_hud_hud3_x_direction` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | HUD 槽 3 文本水平移动方向：0 从左到右，1 从右到左 | 服务器端本插件逻辑 | `l4d2_scripted_hud.sp:769` |
| `l4d2_scripted_hud_hud3_y_direction` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | HUD 槽 3 文本垂直移动方向：0 从上到下，1 从下到上 | 服务器端本插件逻辑 | `l4d2_scripted_hud.sp:770` |
| `l4d2_scripted_hud_hud3_x_min` | `0.0` | 整数，源码写作浮点 | -1.0 ~ 1.0 | HUD 槽 3 动画可到达的水平最小位置 | 服务器端本插件逻辑 | `l4d2_scripted_hud.sp:771` |
| `l4d2_scripted_hud_hud3_y_min` | `0.0` | 整数，源码写作浮点 | -1.0 ~ 1.0 | HUD 槽 3 动画可到达的垂直最小位置 | 服务器端本插件逻辑 | `l4d2_scripted_hud.sp:772` |
| `l4d2_scripted_hud_hud3_x_max` | `1.0` | 整数，源码写作浮点 | -1.0 ~ 1.0 | HUD 槽 3 动画可到达的水平最大位置 | 服务器端本插件逻辑 | `l4d2_scripted_hud.sp:773` |
| `l4d2_scripted_hud_hud3_y_max` | `1.0` | 整数，源码写作浮点 | -1.0 ~ 1.0 | HUD 槽 3 动画可到达的垂直最大位置 | 服务器端本插件逻辑 | `l4d2_scripted_hud.sp:774` |
| `l4d2_scripted_hud_hud3_width` | `1.5` | 浮点 | 0.0 ~ 2.0 | HUD 槽 3 文本区域宽度 | 服务器端本插件逻辑 | `l4d2_scripted_hud.sp:775` |
| `l4d2_scripted_hud_hud3_height` | `0.026` | 浮点 | 0.0 ~ 2.0 | HUD 槽 3 文本区域高度 | 服务器端本插件逻辑 | `l4d2_scripted_hud.sp:776` |
| `l4d2_scripted_hud_hud4_text` | `` | 字符串或表达式 | 无上下界 | HUD 槽 4 显示文本，留空则使用插件内置的预定义文本 | 服务器端本插件逻辑 | `l4d2_scripted_hud.sp:777` |
| `l4d2_scripted_hud_hud4_text_align` | `1` | 整数，源码写作浮点 | 1.0 ~ 3.0 | HUD 槽 4 水平对齐：1 左，2 中，3 右 | 服务器端本插件逻辑 | `l4d2_scripted_hud.sp:778` |
| `l4d2_scripted_hud_hud4_blink_tank` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | HUD 槽 4 在有 Tank 存活时是否白红闪烁：0 关，1 开 | 服务器端本插件逻辑 | `l4d2_scripted_hud.sp:779` |
| `l4d2_scripted_hud_hud4_blink` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | HUD 槽 4 文本是否白红闪烁：0 关，1 开 | 服务器端本插件逻辑 | `l4d2_scripted_hud.sp:780` |
| `l4d2_scripted_hud_hud4_beep` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | HUD 槽 4 闪烁时是否播放提示音，需闪烁开关为 1 | 服务器端本插件逻辑 | `l4d2_scripted_hud.sp:781` |
| `l4d2_scripted_hud_hud4_visible` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | HUD 槽 4 是否允许玩家显示该槽位：0 全局禁用，1 按玩家偏好 | 服务器端本插件逻辑 | `l4d2_scripted_hud.sp:782` |
| `l4d2_scripted_hud_hud4_background` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | HUD 槽 4 文本是否显示黑色半透明背景 | 服务器端本插件逻辑 | `l4d2_scripted_hud.sp:783` |
| `l4d2_scripted_hud_hud4_team` | `0` | 整数，源码写作浮点 | 0.0 ~ 2.0 | 哪些队伍能看到 HUD 槽 4：0 全部，1 生还者，2 特感 | 服务器端本插件逻辑 | `l4d2_scripted_hud.sp:784` |
| `l4d2_scripted_hud_hud4_flag_debug` | `0` | 整数，源码写作浮点 | 0.0 ~ 32767.0 | 覆盖 HUD 槽 4 的 flag，仅调试用，0 关闭 | 服务器端本插件逻辑 | `l4d2_scripted_hud.sp:785` |
| `l4d2_scripted_hud_hud4_x` | `0.75` | 浮点 | -1.0 ~ 1.0 | HUD 槽 4 文本水平位置 -1.0~1.0，小于 0 可能被裁切 | 服务器端本插件逻辑 | `l4d2_scripted_hud.sp:786` |
| `l4d2_scripted_hud_hud4_y` | `0.35` | 浮点 | -1.0 ~ 1.0 | HUD 槽 4 文本垂直位置 -1.0~1.0，小于 0 可能被裁切 | 服务器端本插件逻辑 | `l4d2_scripted_hud.sp:787` |
| `l4d2_scripted_hud_hud4_x_speed` | `0.0` | 整数，源码写作浮点 | 0.0 ~ 2.0 | HUD 槽 4 文本水平移动动画速度，0 关闭 | 服务器端本插件逻辑 | `l4d2_scripted_hud.sp:788` |
| `l4d2_scripted_hud_hud4_y_speed` | `0.0` | 整数，源码写作浮点 | 0.0 ~ 2.0 | HUD 槽 4 文本垂直移动动画速度，0 关闭 | 服务器端本插件逻辑 | `l4d2_scripted_hud.sp:789` |
| `l4d2_scripted_hud_hud4_x_direction` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | HUD 槽 4 文本水平移动方向：0 从左到右，1 从右到左 | 服务器端本插件逻辑 | `l4d2_scripted_hud.sp:790` |
| `l4d2_scripted_hud_hud4_y_direction` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | HUD 槽 4 文本垂直移动方向：0 从上到下，1 从下到上 | 服务器端本插件逻辑 | `l4d2_scripted_hud.sp:791` |
| `l4d2_scripted_hud_hud4_x_min` | `0.0` | 整数，源码写作浮点 | -1.0 ~ 1.0 | HUD 槽 4 动画可到达的水平最小位置 | 服务器端本插件逻辑 | `l4d2_scripted_hud.sp:792` |
| `l4d2_scripted_hud_hud4_y_min` | `0.0` | 整数，源码写作浮点 | -1.0 ~ 1.0 | HUD 槽 4 动画可到达的垂直最小位置 | 服务器端本插件逻辑 | `l4d2_scripted_hud.sp:793` |
| `l4d2_scripted_hud_hud4_x_max` | `1.0` | 整数，源码写作浮点 | -1.0 ~ 1.0 | HUD 槽 4 动画可到达的水平最大位置 | 服务器端本插件逻辑 | `l4d2_scripted_hud.sp:794` |
| `l4d2_scripted_hud_hud4_y_max` | `1.0` | 整数，源码写作浮点 | -1.0 ~ 1.0 | HUD 槽 4 动画可到达的垂直最大位置 | 服务器端本插件逻辑 | `l4d2_scripted_hud.sp:795` |
| `l4d2_scripted_hud_hud4_width` | `1.5` | 浮点 | 0.0 ~ 2.0 | HUD 槽 4 文本区域宽度 | 服务器端本插件逻辑 | `l4d2_scripted_hud.sp:796` |
| `l4d2_scripted_hud_hud4_height` | `0.026` | 浮点 | 0.0 ~ 2.0 | HUD 槽 4 文本区域高度 | 服务器端本插件逻辑 | `l4d2_scripted_hud.sp:797` |

## [L4D & 2] Survivor FF Announce —— `extend/l4d_ffannounce.sp`

- 源文件：`extend/l4d_ffannounce.sp`
- myinfo：author=AiMee, Forgetest
- myinfo description（源码原文）：Friendly Fire Announcements

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `l4d_ff_announce_enable` | `2` | 整数，源码写作浮点 | 0.0 ~ 2.0 | 是否播报友军伤害：0 关闭，1 私下播报，2 额外播报给旁观者 | 服务器端本插件逻辑；仅服务器 | `l4d_ffannounce.sp:57` |

## [L4D & L4D2] Gear Transfer —— `extend/l4d_gear_transfer.sp`

- 源文件：`extend/l4d_gear_transfer.sp`
- myinfo：author=SilverShot
- myinfo description（源码原文）：Survivor bots can automatically pickup and give items. Players can switch, grab or give items.

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `l4d_gear_transfer_allow` | `1` | 整数 | 无上下界 | 0 关闭插件，1 开启插件 | 服务器端本插件逻辑 | `l4d_gear_transfer.sp:697` |
| `l4d_gear_transfer_modes_bot` | `` | 字符串或表达式 | 无上下界 | 在这些游戏模式中禁止 Bot 自动给/取物品，逗号分隔，留空为不禁止 | 服务器端本插件逻辑 | `l4d_gear_transfer.sp:698` |
| `l4d_gear_transfer_modes_on` | `` | 字符串或表达式 | 无上下界 | 在这些游戏模式中启用插件，逗号分隔，留空为全部模式 | 服务器端本插件逻辑 | `l4d_gear_transfer.sp:699` |
| `l4d_gear_transfer_modes_off` | `` | 字符串或表达式 | 无上下界 | 在这些游戏模式中关闭插件，逗号分隔，留空为不关闭 | 服务器端本插件逻辑 | `l4d_gear_transfer.sp:700` |
| `l4d_gear_transfer_modes_tog` | `0` | 整数 | 无上下界 | 按模式位掩码启用插件：0 全部，1 战役，2 生存，4 对战，8 清道夫，可相加 | 服务器端本插件逻辑 | `l4d_gear_transfer.sp:701` |
| `l4d_gear_transfer_dist_give` | `150.0` | 整数，源码写作浮点 | 无上下界 | 转移物品所需的距离，同时影响 Bot 自动给予的范围 | 服务器端本插件逻辑 | `l4d_gear_transfer.sp:702` |
| `l4d_gear_transfer_dist_grab` | `150.0` | 整数，源码写作浮点 | 无上下界 | Bot 自动拾取物品所需的距离 | 服务器端本插件逻辑 | `l4d_gear_transfer.sp:703` |
| `l4d_gear_transfer_dying` | `0` | 整数 | 无上下界 | Bot 仅在接收方黑白时自动给予：0 忽略，1 急救包，2 止痛药或肾上腺素 | 服务器端本插件逻辑 | `l4d_gear_transfer.sp:704` |
| `l4d_gear_transfer_idle` | `0` | 整数 | 无上下界 | 是否允许与挂机玩家转移物品：0 否，1 是 | 服务器端本插件逻辑 | `l4d_gear_transfer.sp:705` |
| `l4d_gear_transfer_method` | `3` | 整数 | 无上下界 | 转移方式：0 关闭，1 仅推击，2 仅换弹键，3 推击与换弹键 | 服务器端本插件逻辑 | `l4d_gear_transfer.sp:706` |
| `l4d_gear_transfer_notifies` | `7` | 整数 | 无上下界 | 在哪些转移类型时提示：1 给予，2 拾取，4 交换，7 全部，可相加 | 服务器端本插件逻辑 | `l4d_gear_transfer.sp:707` |
| `l4d_gear_transfer_notify` | `1` | 整数 | 无上下界 | 转移提示方式：0 关闭，1 显示给所有人，2 额外显示游戏自带的药丸/肾上腺素转移，4 描述被截断 | 服务器端本插件逻辑 | `l4d_gear_transfer.sp:708` |
| `l4d_gear_transfer_sounds` | `1` | 整数 | 无上下界 | 0 关闭，1 给给予或接收物品的玩家播放音效 | 服务器端本插件逻辑 | `l4d_gear_transfer.sp:709` |
| `l4d_gear_transfer_start` | `0.0` | 整数，源码写作浮点 | 无上下界 | 回合开始后多少秒内禁止 Bot 自动给予与自动拾取 | 服务器端本插件逻辑 | `l4d_gear_transfer.sp:710` |
| `l4d_gear_transfer_timer_give` | `1.0` | 整数，源码写作浮点 | 0.0 ~ 10.0 | 检查生还者 Bot 与真人位置以自动给予的间隔秒数，0 关闭，范围 0~10 | 服务器端本插件逻辑 | `l4d_gear_transfer.sp:711` |
| `l4d_gear_transfer_timer_grab` | `0.5` | 浮点 | 0.0 ~ 10.0 | 检查生还者 Bot 与物品位置以自动拾取的间隔秒数，0 关闭，范围 0~10 | 服务器端本插件逻辑 | `l4d_gear_transfer.sp:712` |
| `l4d_gear_transfer_timeout` | `5.0` | 整数，源码写作浮点 | ≥ 1.0 | Bot 与玩家交换后停止归还物品的超时秒数，最小 1 | 服务器端本插件逻辑 | `l4d_gear_transfer.sp:713` |
| `l4d_gear_transfer_traces` | `15` | 整数，源码写作浮点 | 1.0 ~ 120.0 | 每帧用于自动给/取的最大射线检测次数，范围 1~120 | 服务器端本插件逻辑 | `l4d_gear_transfer.sp:714` |
| `l4d_gear_transfer_types_give` | `123456789` | 整数 | 无上下界 | Bot 可自动给予的物品类型位域：0 关闭，1 肾上腺素，2 止痛药，3 燃烧瓶，4 管式炸弹，5 呕吐瓶，6 急救包，其余截断 | 服务器端本插件逻辑 | `l4d_gear_transfer.sp:715` |
| `l4d_gear_transfer_types_grab` | `123456789` | 整数 | 无上下界 | Bot 可自动拾取的物品类型位域，取值含义同自动给予 | 服务器端本插件逻辑 | `l4d_gear_transfer.sp:716` |
| `l4d_gear_transfer_types_real` | `123456789` | 整数 | 无上下界 | 真人玩家可转移的物品类型位域，取值含义同上 | 服务器端本插件逻辑 | `l4d_gear_transfer.sp:717` |
| `l4d_gear_transfer_vocalize` | `1` | 整数 | 无上下界 | 0 关闭，1 玩家转移物品时发出语音，新回合前 60 秒内禁止 | 服务器端本插件逻辑 | `l4d_gear_transfer.sp:718` |
| `l4d_gear_transfer_version` | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | 服务器端本插件逻辑；不写入 cfg 存档 | `l4d_gear_transfer.sp:719` |

## [L4D & L4D2] Hats —— `extend/l4d_hats.sp`

- 源文件：`extend/l4d_hats.sp`
- myinfo：author=SilverShot
- myinfo description（源码原文）：Attaches specified models to players above their head.

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_hats` | 无参数 | 无（任意玩家可用） | 打开帽子自定义设置菜单 | `l4d_hats.sp:854` |
| `sm_hat` | [帽子编号] | 无（任意玩家可用） | 打开帽子列表菜单以更换自己佩戴的帽子 | `l4d_hats.sp:855` |
| `sm_hatoff` | 无参数 | 无（任意玩家可用） | 切换是否佩戴帽子 | `l4d_hats.sp:856` |
| `sm_hatshow` | [开或关] | 无（任意玩家可用） | 切换是否显示自己的帽子 | `l4d_hats.sp:857` |
| `sm_hatview` | [开或关] | 无（任意玩家可用） | 切换是否显示自己的帽子，sm_hatshow 的别名 | `l4d_hats.sp:858` |
| `sm_hatshowon` | 无参数 | 无（任意玩家可用） | 显示自己的帽子 | `l4d_hats.sp:859` |
| `sm_hatshowoff` | 无参数 | 无（任意玩家可用） | 隐藏自己的帽子 | `l4d_hats.sp:860` |
| `sm_hatall` | 无参数 | 无（任意玩家可用） | 切换所有人帽子的可见性 | `l4d_hats.sp:861` |
| `sm_hatclient` | <目标玩家> [帽子名或索引 0 至 128] | 需要 ADMFLAG_ROOT（z，最高权限） | 给目标玩家设置帽子 | `l4d_hats.sp:862` |
| `sm_hatoffc` | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 打开玩家列表以切换指定玩家能否佩戴帽子 | `l4d_hats.sp:863` |
| `sm_hatallc` | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 打开玩家列表以切换指定玩家帽子是否可见 | `l4d_hats.sp:864` |
| `sm_hatc` | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 打开玩家列表，选择一名玩家修改其帽子 | `l4d_hats.sp:865` |
| `sm_hatrandom` | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 随机化所有玩家的帽子 | `l4d_hats.sp:866` |
| `sm_hatrand` | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 随机化所有玩家的帽子，sm_hatrandom 的别名 | `l4d_hats.sp:867` |
| `sm_hatadd` | <完整模型路径> | 需要 ADMFLAG_ROOT（z，最高权限） | 把指定模型加入帽子配置 | `l4d_hats.sp:868` |
| `sm_hatdel` | <索引或部分名称> | 需要 ADMFLAG_ROOT（z，最高权限） | 从帽子配置中删除模型 | `l4d_hats.sp:869` |
| `sm_hatlist` | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 列出所有帽子模型，供 sm_hatdel 使用 | `l4d_hats.sp:870` |
| `sm_hatsave` | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 保存当前帽子的位置与角度到帽子配置 | `l4d_hats.sp:871` |
| `sm_hatload` | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 把所有玩家的帽子改成自己当前佩戴的帽子 | `l4d_hats.sp:872` |
| `sm_hatang` | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 打开菜单调整帽子角度，影响所有帽子与玩家 | `l4d_hats.sp:873` |
| `sm_hatpos` | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 打开菜单调整帽子位置，影响所有帽子与玩家 | `l4d_hats.sp:874` |
| `sm_hatsize` | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 打开菜单调整帽子大小，影响所有帽子与玩家 | `l4d_hats.sp:875` |

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `l4d_hats_allow` | `1` | 整数 | 无上下界 | 0 关闭插件，1 开启插件 | 服务器端本插件逻辑 | `l4d_hats.sp:816` |
| `l4d_hats_bots` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 0 不允许 Bot 生成时戴帽子，1 允许 | 服务器端本插件逻辑 | `l4d_hats.sp:817` |
| `l4d_hats_change` | `1.3` | 浮点 | 无上下界 | 0 关闭；其它值表示选择帽子时让玩家进入第三人称的持续秒数 | 服务器端本插件逻辑 | `l4d_hats.sp:818` |
| `l4d_hats_detect` | `0.3` | 浮点 | 无上下界 | 0 关闭；检测第三人称视角的间隔，若存在 ThirdPersonShoulder_Detect 插件也会使用 | 服务器端本插件逻辑 | `l4d_hats.sp:819` |
| `l4d_hats_make` | `c` | 字符串或表达式 | 无上下界 | 允许随机戴帽子的管理员 flag，留空为所有玩家，需 l4d_hats_random 生效 | 服务器端本插件逻辑 | `l4d_hats.sp:820` |
| `l4d_hats_menu` | `c` | 字符串或表达式 | 无上下界 | 允许访问帽子菜单的管理员 flag，留空为所有玩家 | 服务器端本插件逻辑 | `l4d_hats.sp:821` |
| `l4d_hats_modes` | `` | 字符串或表达式 | 无上下界 | 在这些游戏模式中启用插件，逗号分隔，留空为全部模式 | 服务器端本插件逻辑 | `l4d_hats.sp:822` |
| `l4d_hats_modes_off` | `` | 字符串或表达式 | 无上下界 | 在这些游戏模式中关闭插件，逗号分隔，留空为不关闭 | 服务器端本插件逻辑 | `l4d_hats.sp:823` |
| `l4d_hats_modes_tog` | `` | 字符串或表达式 | 无上下界 | 按模式位掩码启用插件：0 全部，1 战役，2 生存，4 对战，8 清道夫，可相加 | 服务器端本插件逻辑 | `l4d_hats.sp:824` |
| `l4d_hats_opaque` | `255` | 整数，源码写作浮点 | 0.0 ~ 255.0 | 帽子的不透明度：0 半透明，255 完全不透明 | 服务器端本插件逻辑 | `l4d_hats.sp:825` |
| `l4d_hats_precache` | `` | 字符串或表达式 | 无上下界 | 在这些地图上禁止预缓存模型，逗号分隔，源码警告在这些地图启用会导致服务器崩溃 | 服务器端本插件逻辑 | `l4d_hats.sp:826` |
| `l4d_hats_random` | `0` | 整数，源码写作浮点 | 0.0 ~ 3.0 | 生还者生成时是否随机戴帽子：0 从不，1 每回合开始，2 仅首次生成（下回合保持同一顶） | 服务器端本插件逻辑 | `l4d_hats.sp:827` |
| `l4d_hats_save` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 0 关闭，1 保存玩家选定的帽子并在其生成或重连时戴上，覆盖随机设置 | 服务器端本插件逻辑 | `l4d_hats.sp:828` |
| `l4d_hats_third` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 0 关闭，1 玩家处于第三人称时显示帽子，第一人称时隐藏 | 服务器端本插件逻辑 | `l4d_hats.sp:829` |
| `l4d_hats_wall` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 0 显示穿墙的帽子发光，1 隐藏墙后的帽子发光（每个帽子多消耗一个实体） | 服务器端本插件逻辑 | `l4d_hats.sp:830` |
| `l4d_hats_version` | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | 服务器端本插件逻辑；不写入 cfg 存档 | `l4d_hats.sp:831` |

## High Perk Server Status Control[Work in ZM] —— `extend/l4d_player_count_unload_mode.sp`

- 源文件：`extend/l4d_player_count_unload_mode.sp`
- myinfo：author=morzlee
- myinfo description（源码原文）：This is custom plugin

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_peakstatus` | 无参数 | 需要 ADMFLAG_GENERIC（b，通用管理员） | 查看当前全服高峰期的判定状态 | `l4d_player_count_unload_mode.sp:170` |

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `l4d_player_count_unload_mode_enable` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 0 关闭插件，1 开启插件 | 服务器端本插件逻辑 | `l4d_player_count_unload_mode.sp:121` |
| `l4d_player_count_unload_mode_time` | `18:00~22:59` | 字符串或表达式 | 无上下界 | 检测的时间段，格式 xx:xx~xx:xx 二十四小时制，多段用逗号分隔 | 服务器端本插件逻辑 | `l4d_player_count_unload_mode.sp:122` |
| `l4d_player_count_unload_mode_count` | `3` | 整数，源码写作浮点 | 1.0 ~ 32.0 | 当 survivor_limit 与 infected 空位之和小于等于该值时强制 sm_resetmatch 并卸载模式，范围 1~32 | 服务器端本插件逻辑 | `l4d_player_count_unload_mode.sp:123` |
| `l4d_player_count_unload_mode_flag` | `b` | 字符串或表达式 | 无上下界 | 拥有该权限的管理员在场时不会被强制卸载模式 | 服务器端本插件逻辑 | `l4d_player_count_unload_mode.sp:124` |
| `l4d_player_count_unload_mode_delay` | `60.0` | 整数，源码写作浮点 | ≥ 0.0 | 地图加载后经过这么多秒才开始检测时间与人数 | 服务器端本插件逻辑 | `l4d_player_count_unload_mode.sp:125` |
| `l4d_player_count_unload_mode_peak_mode` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 高峰期判定方式：0 按时间段，1 按共享数据库中所有服务器的有玩家比例 | 服务器端本插件逻辑 | `l4d_player_count_unload_mode.sp:126` |
| `l4d_player_count_unload_mode_peak_ratio` | `0.70` | 浮点 | 0.0 ~ 1.0 | peak_mode 为 1 时，有玩家的服务器数占有效服务器数的比例达到该值即视为高峰期 | 服务器端本插件逻辑 | `l4d_player_count_unload_mode.sp:127` |
| `l4d_player_count_unload_mode_peak_hold_time` | `3600` | 整数，源码写作浮点 | ≥ 0.0 | peak_mode 为 1 时，进入高峰期后至少持续限制的秒数，0 表示不保持 | 服务器端本插件逻辑 | `l4d_player_count_unload_mode.sp:128` |
| `l4d_player_count_unload_mode_db_config` | `l4dstats` | 字符串或表达式 | 无上下界 | peak_mode 为 1 时使用的 databases.cfg 区块名 | 服务器端本插件逻辑 | `l4d_player_count_unload_mode.sp:129` |
| `l4d_player_count_unload_mode_server_id` | `` | 字符串或表达式 | 无上下界 | 本服务器唯一 ID，留空时优先从 hostname 提取 #编号，失败则用 hostname:hostport | 服务器端本插件逻辑 | `l4d_player_count_unload_mode.sp:130` |
| `l4d_player_count_unload_mode_server_ip` | `` | 字符串或表达式 | 无上下界 | 写入网页状态表的服务器公网 IP 或域名，可填 host 或 host:port，留空则自动读取 | 服务器端本插件逻辑 | `l4d_player_count_unload_mode.sp:131` |
| `l4d_player_count_unload_mode_server_port` | `0` | 整数，源码写作浮点 | 0.0 ~ 65535.0 | 写入网页状态表的服务器外网端口，0 表示自动读取 | 服务器端本插件逻辑 | `l4d_player_count_unload_mode.sp:132` |
| `l4d_player_count_unload_mode_status_table` | `l4d_server_status` | 字符串或表达式 | 无上下界 | peak_mode 为 1 时使用的服务器状态表名 | 服务器端本插件逻辑 | `l4d_player_count_unload_mode.sp:133` |
| `l4d_player_count_unload_mode_status_interval` | `180.0` | 整数，源码写作浮点 | ≥ 5.0 | peak_mode 为 1 时本服人数写入数据库的心跳间隔秒数，最小 5 | 服务器端本插件逻辑 | `l4d_player_count_unload_mode.sp:134` |
| `l4d_player_count_unload_mode_status_max_age` | `540` | 整数，源码写作浮点 | ≥ 10.0 | peak_mode 为 1 时只统计多少秒内更新过的服务器，最小 10 | 服务器端本插件逻辑 | `l4d_player_count_unload_mode.sp:135` |
| `l4d_player_count_unload_mode_version` | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | 服务器端本插件逻辑；仅服务器；不写入 cfg 存档 | `l4d_player_count_unload_mode.sp:136` |
| `l4d_player_count_unload_mode_server_utc_offset` | `480` | 整数 | 无上下界 | 备用值：服务器时区相对 UTC 的偏移分钟数，仅在 %z 格式不受支持时使用 | 服务器端本插件逻辑 | `l4d_player_count_unload_mode.sp:137` |

## [L4D & L4D2] Ragdoll Fader —— `extend/l4d_ragdoll_fader.sp`

- 源文件：`extend/l4d_ragdoll_fader.sp`
- myinfo：author=SilverShot
- myinfo description（源码原文）：Fades common infected ragdolls.

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `l4d_ragdoll_fader` | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | 服务器端本插件逻辑；不写入 cfg 存档 | `l4d_ragdoll_fader.sp:82` |

## l4d_random_beam_item.sp（myinfo 缺 name，用文件名代替） —— `extend/l4d_random_beam_item.sp`

- 源文件：`extend/l4d_random_beam_item.sp`
- myinfo：源码中未找到 myinfo 块

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_beaminfo` | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 在聊天中输出准星指向实体的光束信息 | `l4d_random_beam_item.sp:365` |
| `sm_beamreload` | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 重新加载光束配置 | `l4d_random_beam_item.sp:366` |
| `sm_beamremove` | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 移除准星指向实体上的插件光束 | `l4d_random_beam_item.sp:367` |
| `sm_beamremoveall` | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 移除插件创建的所有光束 | `l4d_random_beam_item.sp:368` |
| `sm_beamadd` | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 给准星指向的实体添加一条默认配置的光束 | `l4d_random_beam_item.sp:369` |
| `sm_print_cvars_l4d_random_beam_item` | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 把本插件的 ConVar 及其取值打印到控制台 | `l4d_random_beam_item.sp:370` |
| `sm_beam` | [bright 或 subtle 或 off 或 default 或 reset] | 无（任意玩家可用） | 设置自己的物品光束样式 | `l4d_random_beam_item.sp:373` |

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `l4d_random_beam_item_version` | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | 服务器端本插件逻辑；仅服务器；不写入 cfg 存档 | `l4d_random_beam_item.sp:345` |
| `l4d_random_beam_item_enable` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用插件：0 关，1 开 | 服务器端本插件逻辑 | `l4d_random_beam_item.sp:346` |
| `l4d_random_beam_item_remove_spawner` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 当 *_spawn 实体计数归零时是否删除该实体：0 关，1 开 | 服务器端本插件逻辑 | `l4d_random_beam_item.sp:347` |
| `l4d_random_beam_item_min_brightness` | `0.5` | 浮点 | 0.0 ~ 1.0 | 判断随机颜色光束最低亮度的算法阈值，源码注明该值不精确 | 服务器端本插件逻辑 | `l4d_random_beam_item.sp:348` |
| `l4d_random_beam_item_use_glow_color` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 仅 L4D2：是否沿用发光颜色：0 关，1 开 | 服务器端本插件逻辑 | `l4d_random_beam_item.sp:350` |
| `l4d_random_beam_item_player_edict_limit` | `1900` | 整数，源码写作浮点 | 0.0 ~ 2048.0 | 当服务器已用实体数达到该值时，玩家为默认隐藏物品开启的光束不再创建，范围 0~2048 | 服务器端本插件逻辑 | `l4d_random_beam_item.sp:351` |

## [L4D & L4D2] Saferoom Door Spam Protection —— `extend/l4d_safe_door_spam.sp`

- 源文件：`extend/l4d_safe_door_spam.sp`
- myinfo：author=SilverShot
- myinfo description（源码原文）：Control opening of first saferoom door prevent spamming the last door.

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_door_drop` | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 测试命令，让准星指向的门倒下 | `l4d_safe_door_spam.sp:295` |
| `sm_door_fall` | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 测试命令，让第一扇上锁的安全门倒下 | `l4d_safe_door_spam.sp:296` |

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `l4d_safe_spam_allow` | `1` | 整数 | 无上下界 | 0 关闭插件，1 开启插件 | 服务器端本插件逻辑 | `l4d_safe_door_spam.sp:252` |
| `l4d_safe_spam_modes` | `` | 字符串或表达式 | 无上下界 | 在这些游戏模式中启用插件，逗号分隔，留空为全部模式 | 服务器端本插件逻辑 | `l4d_safe_door_spam.sp:253` |
| `l4d_safe_spam_modes_off` | `` | 字符串或表达式 | 无上下界 | 在这些游戏模式中关闭插件，逗号分隔，留空为不关闭 | 服务器端本插件逻辑 | `l4d_safe_door_spam.sp:254` |
| `l4d_safe_spam_modes_tog` | `0` | 整数 | 无上下界 | 按模式位掩码启用插件：0 全部，1 战役，2 生存，4 对战，8 清道夫，可相加 | 服务器端本插件逻辑 | `l4d_safe_door_spam.sp:255` |
| `l4d_safe_spam_fall_time` | `0.0` | 整数，源码写作浮点 | 无上下界 | 0 关闭；回合开始或被锁定类 cvar 解锁后多少秒内不得使用安全门 | 服务器端本插件逻辑 | `l4d_safe_door_spam.sp:256` |
| `l4d_safe_spam_hint` | `0` | 整数 | 无上下界 | 0 关闭；1 显示谁开关了安全门，2 在安全门自动……时显示（描述被截断） | 服务器端本插件逻辑 | `l4d_safe_door_spam.sp:257` |
| `l4d_safe_spam_last` | `0` | 整数 | 无上下界 | 回合开始时最后一扇安全门的最终状态：0 用地图默认，1 关闭，2 打开 | 服务器端本插件逻辑 | `l4d_safe_door_spam.sp:258` |
| `l4d_safe_spam_lock` | `0.0` | 整数，源码写作浮点 | 无上下界 | 0 关闭；回合开始后安全门保持上锁的秒数 | 服务器端本插件逻辑 | `l4d_safe_door_spam.sp:259` |
| `l4d_safe_spam_lock_2` | `0.0` | 整数，源码写作浮点 | 无上下界 | 同 lock，但作用于地图第二回合及之后 | 服务器端本插件逻辑 | `l4d_safe_door_spam.sp:260` |
| `l4d_safe_spam_touch` | `0.0` | 整数，源码写作浮点 | 无上下界 | 0 关闭；尝试打开上锁安全门后多少秒解锁，覆盖 _time 类 cvar | 服务器端本插件逻辑 | `l4d_safe_door_spam.sp:261` |
| `l4d_safe_spam_touch_2` | `0.0` | 整数，源码写作浮点 | 无上下界 | 同 touch，但作用于地图第二回合及之后（描述被截断） | 服务器端本插件逻辑 | `l4d_safe_door_spam.sp:262` |
| `l4d_safe_spam_open` | `2` | 整数 | 无上下界 | 0 关闭，1 第一扇安全门打开后保持打开，2 打开后让它倒下 | 服务器端本插件逻辑 | `l4d_safe_door_spam.sp:263` |
| `l4d_safe_spam_physics` | `3.0` | 整数，源码写作浮点 | 无上下界 | 0 表示始终保留物理；倒下的门在多长时间后禁用物理 | 服务器端本插件逻辑 | `l4d_safe_door_spam.sp:264` |
| `l4d_safe_spam_skin` | `0` | 整数 | 无上下界 | 第一与最后一扇安全门使用哪种模型：0 地图默认，1 经典，2 The Last Stand | 服务器端本插件逻辑 | `l4d_safe_door_spam.sp:266` |
| `l4d_safe_spam_time_close` | `0.0` | 整数，源码写作浮点 | 无上下界 | 关闭最后一扇安全门后禁止操作的秒数 | 服务器端本插件逻辑 | `l4d_safe_door_spam.sp:267` |
| `l4d_safe_spam_time_open` | `0.0` | 整数，源码写作浮点 | 无上下界 | 打开最后一扇安全门后禁止操作的秒数 | 服务器端本插件逻辑 | `l4d_safe_door_spam.sp:268` |
| `l4d_safe_spam_type` | `3` | 整数 | 无上下界 | 0 关闭；最后一扇安全门被使用时启用超时限制的方向：1 打开，2 关闭，3 两者 | 服务器端本插件逻辑 | `l4d_safe_door_spam.sp:269` |
| `l4d_safe_spam_version` | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | 服务器端本插件逻辑；不写入 cfg 存档 | `l4d_safe_door_spam.sp:271` |

## l4d_stats.sp（myinfo 缺 name，用文件名代替） —— `extend/l4d_stats.sp`

- 源文件：`extend/l4d_stats.sp`
- myinfo：author=东，Mikko Andersson (muukis)

### 指令

（本插件未注册指令）

### ConVar

（本插件未注册 ConVar）

## [L4D & L4D2] Use Priority Patch —— `extend/l4d_use_priority.sp`

- 源文件：`extend/l4d_use_priority.sp`
- myinfo：author=SilverShot
- myinfo description（源码原文）：Patches CBaseEntity::GetUsePriority preventing attached entities blocking +USE.

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `l4d_use_priority_version` | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | 服务器端本插件逻辑；不写入 cfg 存档 | `l4d_use_priority.sp:135` |

## lilac.sp（myinfo 缺 name，用文件名代替） —— `extend/lilac.sp`

- 源文件：`extend/lilac.sp`
- myinfo：源码中未找到 myinfo 块
- HookConVarChange：`cvar_bhop`→`cvar_change`

### 指令

（本插件未注册指令）

### ConVar

（本插件未注册 ConVar）

## SM Client Command Logging —— `extend/logcommands.sp`

- 源文件：`extend/logcommands.sp`
- myinfo：author=Franc1sco franug
- myinfo description（源码原文）：Logging every command that the client use

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `sm_clientcommandlogging_version` | `DATA` | 字符串或表达式 | 无上下界 | 插件版本号（默认值取自 DATA 宏，非行为配置） | 服务器端本插件逻辑；复制到客户端；仅服务器 | `logcommands.sp:54` |

## Anne Telecom Server Mode Guide —— `extend/new_player_guide.sp`

- 源文件：`extend/new_player_guide.sp`
- myinfo：author=morzlee
- myinfo description（源码原文）：Guides players through Anne server PvE and versus mode choices.

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_guide` | 无参数 | 无（任意玩家可用） | 打开模式引导主菜单 | `new_player_guide.sp:241` |
| `sm_modes` | 无参数 | 无（任意玩家可用） | 打开模式引导主菜单，sm_guide 的别名 | `new_player_guide.sp:242` |
| `sm_modeguide` | 无参数 | 无（任意玩家可用） | 打开模式引导主菜单，sm_guide 的别名 | `new_player_guide.sp:243` |
| `sm_anneguide` | 无参数 | 无（任意玩家可用） | 打开模式引导主菜单，sm_guide 的别名 | `new_player_guide.sp:244` |

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `sm_anne_mode_guide_enable` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用自动的 Anne 模式引导提示 | 服务器端本插件逻辑 | `new_player_guide.sp:233` |
| `sm_anne_mode_guide_initial_delay` | `1.0` | 整数，源码写作浮点 | 1.0 ~ 120.0 | 首次自动模式引导检查前的延迟秒数，范围 1~120 | 服务器端本插件逻辑 | `new_player_guide.sp:234` |
| `sm_anne_mode_guide_retry_delay` | `5.0` | 整数，源码写作浮点 | 1.0 ~ 60.0 | 游戏时长与偏好数据重试之间的延迟秒数，范围 1~60 | 服务器端本插件逻辑 | `new_player_guide.sp:235` |
| `sm_anne_mode_guide_max_retries` | `10` | 整数，源码写作浮点 | 0.0 ~ 30.0 | 等待游戏时长与偏好数据的最大重试次数，范围 0~30 | 服务器端本插件逻辑 | `new_player_guide.sp:236` |
| `sm_anne_mode_guide_server_stop_minutes` | `1800` | 整数，源码写作浮点 | 0.0 ~ 100000.0 | 本服累计游戏时长超过该分钟数后停止自动提示，范围 0~100000 | 服务器端本插件逻辑 | `new_player_guide.sp:237` |
| `sm_anne_mode_guide_permanent_disable_minutes` | `600` | 整数，源码写作浮点 | 0.0 ~ 100000.0 | 出现永久关闭提示开关前所需的本服游戏时长分钟数 | 服务器端本插件逻辑 | `new_player_guide.sp:238` |
| `sm_anne_mode_guide_suppress_mode_loaded` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 加载 Confogl 比赛模式时是否抑制自动提示 | 服务器端本插件逻辑 | `new_player_guide.sp:239` |

## [L4D2] Punch Angle (RPG-aware, recoil command) —— `extend/punch_angle.sp`

- 源文件：`extend/punch_angle.sp`
- myinfo：author=sorallll, blueblur, + morzlee/ChatGPT
- myinfo description（源码原文）：Remove recoil when shooting and getting hit. Uses RPG if present. Adds !recoil command.

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_recoil` | [0 或 1] | 无（任意玩家可用） | 切换自己的后坐力设置 | `punch_angle.sp:77` |
| `sm_punch` | [0 或 1] | 无（任意玩家可用） | 切换自己的后坐力设置，sm_recoil 的别名 | `punch_angle.sp:78` |

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `punch_angle_version` | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | 服务器端本插件逻辑；不写入 cfg 存档；仅开发用 | `punch_angle.sp:61` |
| `punch_angle_toggle` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否开启后坐力 | 服务器端本插件逻辑 | `punch_angle.sp:71` |

## 商店插件 —— `extend/rpg.sp`

- 源文件：`extend/rpg.sp`
- myinfo：author=东
- myinfo description（源码原文）：购买游戏道具,幸存者轮廓，帽子保存,生还者皮肤颜色

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_buy` | 无参数 | 无（任意玩家可用） | 打开购买菜单，仅限游戏内的存活玩家 | `rpg.sp:1121` |
| `sm_ammo` | 无参数 | 无（任意玩家可用） | 快速购买子弹 | `rpg.sp:1122` |
| `sm_pen` | 无参数 | 无（任意玩家可用） | 快速随机购买一把单发霰弹枪 | `rpg.sp:1123` |
| `sm_chr` | 无参数 | 无（任意玩家可用） | 快速购买一把二代单发霰弹枪 Chrome | `rpg.sp:1124` |
| `sm_pum` | 无参数 | 无（任意玩家可用） | 快速购买一把一代单发霰弹枪 Pump | `rpg.sp:1125` |
| `sm_smg` | 无参数 | 无（任意玩家可用） | 快速购买 SMG | `rpg.sp:1126` |
| `sm_uzi` | 无参数 | 无（任意玩家可用） | 快速购买乌兹 | `rpg.sp:1127` |
| `sm_pill` | 无参数 | 无（任意玩家可用） | 快速购买止痛药 | `rpg.sp:1128` |
| `sm_setch` | <称号文本> | 无（任意玩家可用） | 设置自定义称号，需积分不低于 50 万 | `rpg.sp:1129` |
| `sm_unsetch` | 无参数 | 无（任意玩家可用） | 取消自定义称号，需积分不低于 50 万 | `rpg.sp:1130` |
| `sm_applytags` | 无参数 | 无（任意玩家可用） | 佩戴自己的自定义称号 | `rpg.sp:1131` |
| `sm_rpg` | 无参数 | 无（任意玩家可用） | 打开购买菜单，sm_buy 的别名 | `rpg.sp:1132` |
| `sm_rpginfo` | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 把 RPG 人物信息输出到控制台，含近战、血包、轮廓、帽子、皮肤与后坐力 | `rpg.sp:1133` |
| `sm_rpgglowdebug` | <玩家> | 需要 ADMFLAG_ROOT（z，最高权限） | 输出指定玩家的 RPG 轮廓权限调试信息 | `rpg.sp:1134` |

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `shop_enable` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否打开商店购买 | 服务器端本插件逻辑 | `rpg.sp:1085` |
| `rpg_allow_biggun` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 商店是否允许购买大枪 | 服务器端本插件逻辑 | `rpg.sp:1086` |
| `rpg_allow_glow` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 商店是否打开轮廓 | 服务器端本插件逻辑 | `rpg.sp:1087` |
| `rpg_antikick_enable` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用管理员防踢（投票与命令） | 服务器端本插件逻辑 | `rpg.sp:1089` |
| `rpg_antikick_block_votekick` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否禁止对受保护管理员发起投票踢 | 服务器端本插件逻辑 | `rpg.sp:1090` |
| `rpg_antikick_block_cmdkick` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否禁止低级或同级管理员用 sm_kick 踢受保护管理员 | 服务器端本插件逻辑 | `rpg.sp:1091` |
| `rpg_antikick_min_immunity` | `0` | 整数，源码写作浮点 | 0.0 ~ 100.0 | 受保护阈值：管理员免疫等级大于等于该值即受保护，0 表示任意管理员都受保护 | 服务器端本插件逻辑 | `rpg.sp:1092` |
| `rpg_antikick_equal_block` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 同级免疫是否禁止互踢，仅对 sm_kick 生效 | 服务器端本插件逻辑 | `rpg.sp:1093` |
| `rpg_allow_UseB` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否允许消费 B 数，即价格大于 0 的商品：1 允许，0 仅允许 0B 商品 | 服务器端本插件逻辑 | `rpg.sp:1094` |
| `ReturnBlood` | `0` | 整数 | 无上下界 | 击杀特感回血（源码描述只有“回血模式”四字）；据 `extend/rpg.sp:1120` 的 `CreateConVar("ReturnBlood", "0", "回血模式")`、:1079 把 `EventReturnBlood` 挂在 `player_death`（EventHookMode_Pre），:1303-1334 中当死者为特感(team 3)、攻击者为幸存者且 `GetConVarBool(ReturnBlood)` 为真时，把攻击者永久血量写回为「当前永久血量（`player[attacker].ClientBlood>0` 时再 +2）」并以 `m_iMaxHealth` 封顶（:1319-1330），:2477 为真时购买菜单追加“回血技能”项，`optional/AnneHappy/text.sp:256-261` 也按“>0”显示回血已开启，推测为：纯开关，0=关闭、非 0=开启（源码用 `GetConVarBool` 读取，不存在多档取值语义）（置信度：高） | 服务器端本插件逻辑 | `rpg.sp:1120` |

## rygive.sp（myinfo 缺 name，用文件名代替） —— `extend/rygive.sp`

- 源文件：`extend/rygive.sp`
- myinfo：源码中未找到 myinfo 块

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_rygive` | 无参数 | 需要 ADMFLAG_CHAT（j，聊天管理） | 打开 rygive 给予物品菜单 | `rygive.sp:254` |

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `rygive_version` | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | 服务器端本插件逻辑；不写入 cfg 存档 | `rygive.sp:250` |

## SaveChat —— `extend/savechat.sp`

- 源文件：`extend/savechat.sp`
- myinfo：author=citkabuto, sorallll
- myinfo description（源码原文）：Records player chat messages to a file

### 指令

（本插件未注册指令）

### ConVar

（本插件未注册 ConVar）

## SourceBans++: Admin Config Loader —— `extend/sbpp_admcfg.sp`

- 源文件：`extend/sbpp_admcfg.sp`
- myinfo：version=1.6.4，author=AlliedModders LLC, SourceBans++ Dev Team
- myinfo description（源码原文）：Reads Admin Files

### 指令

（本插件未注册指令）

### ConVar

（本插件未注册 ConVar）

## SourceBans++: Bans Checker —— `extend/sbpp_checker.sp`

- 源文件：`extend/sbpp_checker.sp`
- myinfo：author=psychonic, Ca$h Munny, SourceBans++ Dev Team
- myinfo description（源码原文）：Notifies admins of prior bans from Sourcebans upon player connect.

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_listbans` | 无参数 | 需要 ADMFLAG_BAN（d，封禁） | 列出封禁记录 | `sbpp_checker.sp:56` |
| `sm_listcomms` | 无参数 | 需要 ADMFLAG_BAN（d，封禁） | 列出禁言与禁声记录 | `sbpp_checker.sp:57` |
| `sb_reload` | 无参数 | 需要 ADMFLAG_RCON（i，RCON 命令） | 重新加载 SourceBans 配置与封禁原因菜单项 | `sbpp_checker.sp:58` |

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `sbchecker_version` | `VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（默认值取自 VERSION 宏） | 服务器端本插件逻辑 | `sbpp_checker.sp:55` |

## SourceBans++: SourceComms —— `extend/sbpp_comms.sp`

- 源文件：`extend/sbpp_comms.sp`
- myinfo：author=Alex, SourceBans++ Dev Team
- myinfo description（源码原文）：Advanced punishments management for the Source engine in SourceBans style

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sc_fw_block` | 无参数 | 服务器控制台命令，玩家无法使用 | 由 SourceBans 网站调用，用于封禁玩家通信，服务器控制台命令 | `sbpp_comms.sp:203` |
| `sc_fw_ungag` | 无参数 | 服务器控制台命令，玩家无法使用 | 由 SourceBans 网站调用，用于解除禁言，服务器控制台命令 | `sbpp_comms.sp:204` |
| `sc_fw_unmute` | 无参数 | 服务器控制台命令，玩家无法使用 | 由 SourceBans 网站调用，用于解除禁声，服务器控制台命令 | `sbpp_comms.sp:205` |
| `sm_comms` | 无参数 | 无（任意玩家可用） | 显示当前玩家的通信状态 | `sbpp_comms.sp:206` |

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `sourcecomms_version` | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | 服务器端本插件逻辑；复制到客户端；仅服务器 | `sbpp_comms.sp:196` |

## SourceBans++: Main Plugin —— `extend/sbpp_main.sp`

- 源文件：`extend/sbpp_main.sp`
- myinfo：author=SourceBans Development Team, SourceBans++ Dev Team
- myinfo description（源码原文）：Advanced ban management for the Source engine

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_rehash` | 无参数 | 服务器控制台命令，玩家无法使用 | 重新加载 SQL 管理员，服务器控制台命令 | `sbpp_main.sp:172` |
| `sm_ban` | <目标玩家> <分钟数或 0> [原因] | 需要 ADMFLAG_BAN（d，封禁） | 封禁玩家，时间单位为分钟，0 表示永久 | `sbpp_main.sp:173` |
| `sm_banip` | <IP 或目标玩家> <时间> [原因] | 需要 ADMFLAG_BAN（d，封禁） | 按 IP 或玩家封禁 | `sbpp_main.sp:174` |
| `sm_addban` | <时间> <steamid> [原因] | 需要 ADMFLAG_RCON（i，RCON 命令） | 按 SteamID 添加封禁 | `sbpp_main.sp:175` |
| `sm_unban` | <steamid 或 IP> [原因] | 需要 ADMFLAG_UNBAN（e，解封） | 解除封禁 | `sbpp_main.sp:176` |
| `sb_reload` | 无参数 | 需要 ADMFLAG_RCON（i，RCON 命令） | 重新加载 SourceBans 配置与封禁原因菜单项 | `sbpp_main.sp:177` |
| `say` | 聊天文本 | 无（任意玩家可用） | 接管 say 命令，识别 !noreason 等内容以发起封禁 | `sbpp_main.sp:183` |
| `say_team` | 聊天文本 | 无（任意玩家可用） | 接管 say_team 命令，识别 !noreason 等内容以发起封禁 | `sbpp_main.sp:184` |

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `sb_version` | `SB_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（默认值取自 SB_VERSION 宏） | 服务器端本插件逻辑；复制到客户端；仅服务器 | `sbpp_main.sp:170` |
| `sbr_version` | `SBR_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（默认值取自 SBR_VERSION 宏） | 服务器端本插件逻辑；复制到客户端；仅服务器 | `sbpp_main.sp:171` |

## SourceBans++ Report Plugin —— `extend/sbpp_report.sp`

- 源文件：`extend/sbpp_report.sp`
- myinfo：（无 version/author）
- myinfo description（源码原文）：Adds ability for player to report offending players

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_report` | 无参数 | 无（任意玩家可用） | 打开玩家举报菜单 | `sbpp_report.sp:48` |

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `sbpp_report_version` | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | 服务器端本插件逻辑；复制到客户端；仅服务器；不写入 cfg 存档 | `sbpp_report.sp:41` |
| `sbpp_report_cooldown` | `60.0` | 整数，源码写作浮点 | ≥ 0.0 | 同一玩家两次举报之间的冷却秒数 | 服务器端本插件逻辑 | `sbpp_report.sp:43` |
| `sbpp_report_minlen` | `10` | 整数，源码写作浮点 | ≥ 0.0 | 举报理由的最小长度 | 服务器端本插件逻辑 | `sbpp_report.sp:44` |

## SourceBans++: SourceSleuth —— `extend/sbpp_sleuth.sp`

- 源文件：`extend/sbpp_sleuth.sp`
- myinfo：author=ecca, SourceBans++ Dev Team
- myinfo description（源码原文）：Useful for TF2 servers. Plugin will check for banned ips and ban the player.

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_sleuth_reloadlist` | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 重新加载 sleuth 排除列表，源码未给描述 | `sbpp_sleuth.sp:85` |

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `sm_sourcesleuth_version` | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | 服务器端本插件逻辑；复制到客户端；仅服务器；不写入 cfg 存档 | `sbpp_sleuth.sp:70` |
| `sm_sleuth_actions` | `3` | 整数，源码写作浮点 | 1.0 ~ 4.0 | SourceSleuth 的封禁方式：1 原时长，2 自定义时长，3 双倍时长，4 仅通知管理员 | 服务器端本插件逻辑 | `sbpp_sleuth.sp:72` |
| `sm_sleuth_duration` | `0` | 整数 | 无上下界 | 当 sm_sleuth_actions 为 1 时的封禁时长，0 表示永久 | 服务器端本插件逻辑 | `sbpp_sleuth.sp:73` |
| `sm_sleuth_prefix` | `sb` | 字符串或表达式 | 无上下界 | SourceBans 数据库表前缀，默认 sb | 服务器端本插件逻辑 | `sbpp_sleuth.sp:74` |
| `sm_sleuth_bansallowed` | `0` | 整数 | 无上下界 | 采取行动前允许的生效封禁数量 | 服务器端本插件逻辑 | `sbpp_sleuth.sp:75` |
| `sm_sleuth_bantype` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 0 对所有时长类型的封禁都处理，1 仅处理永久封禁 | 服务器端本插件逻辑 | `sbpp_sleuth.sp:76` |
| `sm_sleuth_adminbypass` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 0 不启用，1 允许所有有封禁 flag 的管理员跳过该检查 | 服务器端本插件逻辑 | `sbpp_sleuth.sp:77` |
| `sm_sleuth_excludeold` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 0 不启用，1 允许把旧封禁排除在检查之外 | 服务器端本插件逻辑 | `sbpp_sleuth.sp:78` |
| `sm_sleuth_excludetime` | `31536000` | 整数，源码写作浮点 | ≥ 1.0 | 可排除在检查之外的旧封禁时长阈值，单位秒，最小 1 | 服务器端本插件逻辑 | `sbpp_sleuth.sp:79` |

## Anne ServerName —— `extend/server_name.sp`

- 源文件：`extend/server_name.sp`
- myinfo：version=1.4.8，author=东
- myinfo description（源码原文）：动态服务器名

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `sn_main_name` | `电信服` | 字符串或表达式 | 无上下界 | 服务器主名称（源码未给描述） | 服务器端本插件逻辑 | `server_name.sp:76` |
| `sn_hostname_format` | `{hostname}{gamemode}` | 字符串或表达式 | 无上下界 | hostname 格式模板（源码未给描述） | 服务器端本插件逻辑 | `server_name.sp:79` |
| `sn_hostname_format1` | `{Confogl}{AIDifficulty}{Full}{MOD}{AnneHappy}` | 字符串或表达式 | 无上下界 | 备用 hostname 格式模板（源码未给描述） | 服务器端本插件逻辑 | `server_name.sp:82` |

## [L4D2] Voice Announce + Show MIC Hat. —— `extend/show_mic.sp`

- 源文件：`extend/show_mic.sp`
- myinfo：author=SupermenCJ & Harry Potter
- myinfo description（源码原文）：Voice Announce in centr text + create hat to Show Who is speaking.

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `show_mic_center_hat_enable` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 为 1 时在说话玩家的头顶显示帽子 | 服务器端本插件逻辑 | `show_mic.sp:54` |
| `show_mic_center_text_enable` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 为 1 时在中心文本显示玩家说话提示 | 服务器端本插件逻辑 | `show_mic.sp:55` |
| `show_mic_version` | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | 服务器端本插件逻辑；不写入 cfg 存档 | `show_mic.sp:56` |

## Source Scramble Manager —— `extend/sourcescramble_manager.sp`

- 源文件：`extend/sourcescramble_manager.sp`
- myinfo：author=nosoop
- myinfo description（源码原文）：Helper plugin to load simple assembly patches from a configuration file.

### 指令

（本插件未注册指令）

### ConVar

（本插件未注册 ConVar）

## Updater —— `extend/updater.sp`

- 源文件：`extend/updater/versioncache.sp`、`extend/updater/filesys.sp`、`extend/updater.sp`、`extend/updater/api.sp`、`extend/updater/download.sp`、`extend/updater/plugins.sp`
- myinfo：author=GoD-Tony, Tk /id/Teamkiller324
- myinfo description（源码原文）：Automatically updates SourceMod plugins and files

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_updater_check` | 无参数 | 需要 ADMFLAG_RCON（i，RCON 命令） | 强制检查更新，每小时只能检查一次 | `updater.sp:112` |
| `sm_updater_forcecheck` | 无参数 | 需要 ADMFLAG_RCON（i，RCON 命令） | 无限制地强制检查更新 | `updater.sp:113` |
| `sm_updater_status` | 无参数 | 需要 ADMFLAG_RCON（i，RCON 命令） | 查看 Updater 的状态 | `updater.sp:114` |

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `sm_updater_version` | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | 服务器端本插件逻辑；不写入 cfg 存档 | `updater.sp:105` |
| `sm_updater` | `2` | 整数，源码写作浮点 | 1.0 ~ 3.0 | 更新功能模式：1 仅通知，2 下载，3 同时包含源码 | 服务器端本插件逻辑 | `updater.sp:108` |

## download_curl.sp（myinfo 缺 name，用文件名代替） —— `extend/updater/download_curl.sp`

- 源文件：`extend/updater/download_curl.sp`
- myinfo：源码中未找到 myinfo 块

### 指令

（本插件未注册指令）

### ConVar

（本插件未注册 ConVar）

## download_socket.sp（myinfo 缺 name，用文件名代替） —— `extend/updater/download_socket.sp`

- 源文件：`extend/updater/download_socket.sp`
- myinfo：源码中未找到 myinfo 块

### 指令

（本插件未注册指令）

### ConVar

（本插件未注册 ConVar）

## download_steamtools.sp（myinfo 缺 name，用文件名代替） —— `extend/updater/download_steamtools.sp`

- 源文件：`extend/updater/download_steamtools.sp`
- myinfo：源码中未找到 myinfo 块

### 指令

（本插件未注册指令）

### ConVar

（本插件未注册 ConVar）

## download_steamworks.sp（myinfo 缺 name，用文件名代替） —— `extend/updater/download_steamworks.sp`

- 源文件：`extend/updater/download_steamworks.sp`
- myinfo：源码中未找到 myinfo 块

### 指令

（本插件未注册指令）

### ConVar

（本插件未注册 ConVar）

## VeteransOnly —— `extend/veterans.sp`

- 源文件：`extend/veterans.sp`
- myinfo：author=Soroush Falahati, 东
- myinfo description（源码原文）：Kicks the players without enough playtime in the game

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_veterans_exclude` | 无参数 | 需要 ADMFLAG_GENERIC（b，通用管理员） | 把指定用户排除出 VeteransOnly 检查 | `veterans.sp:207` |
| `sm_veterans_include` | 无参数 | 需要 ADMFLAG_GENERIC（b，通用管理员） | 把已被排除的用户重新纳入 VeteransOnly 检查 | `veterans.sp:208` |
| `sm_clear` | 无参数 | 需要 ADMFLAG_GENERIC（b，通用管理员） | 清空缓存 | `veterans.sp:209` |
| `sm_timeall` | 无参数 | 需要 ADMFLAG_GENERIC（b，通用管理员） | 显示所有玩家的游戏时长 | `veterans.sp:210` |
| `sm_time` | 无参数 | 无（任意玩家可用） | 显示自己的游戏时长 | `veterans.sp:211` |

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `sm_veterans_version` | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | 服务器端本插件逻辑；不写入 cfg 存档 | `veterans.sp:111` |
| `l4d2_playtime_apikey` | `C7B3FC46E6E6D5C87700963F0688FCB4` | 字符串或表达式 | 无上下界 | Steam 开发者 Web API key | 服务器端本插件逻辑；受保护 | `veterans.sp:112` |
| `sm_veterans_enable` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用 VeteransOnly 插件 | 服务器端本插件逻辑 | `veterans.sp:121` |
| `sm_veterans_gameid` | `550` | 整数，源码写作浮点 | 0.0 ~ MAX_FLOAT | 要检查玩家游戏时长的游戏的 Steam 商店 id | 服务器端本插件逻辑 | `veterans.sp:127` |
| `sm_veterans_excludegroupmemberplay` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否让 Steam 组成员免除时长门槛；据 `extend/veterans.sp:133-138` 的英文描述原文「Should we let exclude group member but not rechach mititaion to play」（语法不通），配合 :242 `if(!HasEnoughPlaytime(player[Player].servertime) && player[Player].isGroupMember && !GetConVarBool(cvar_excludeGroupMemberPlay))` 命中时提示 `Veterans_PlayerDurationDetectionNotMeet` 并 `return Plugin_Stop` 拦截入队，以及 :404-415 两个分支的提示文本（`Veterans_PlayerDurationDetectionPlayerGame`=“…may play normally” 与 `Veterans_PlayerTimeDetectionPlayerGame`=“…may only spectate”，见 `translations/veterans.phrases.txt:46-52`），推测为：1=组成员即使时长不达标也可正常游玩（豁免时长限制），0=时长不达标的组成员只能旁观并被拒绝加入队伍（置信度：中） | 服务器端本插件逻辑 | `veterans.sp:133` |
| `sm_veterans_excludereservedslots` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否把拥有预留槽位的玩家排除在惩罚之外 | 服务器端本插件逻辑 | `veterans.sp:139` |
| `sm_veterans_excludeprivileged` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否把有权限的玩家排除在惩罚之外 | 服务器端本插件逻辑 | `veterans.sp:145` |
| `sm_veterans_excludegroupmember` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否把 Steam 组成员排除在惩罚之外 | 服务器端本插件逻辑 | `veterans.sp:151` |
| `sm_veterans_excludegroupmembercount` | `2` | 整数，源码写作浮点 | 0.0 ~ MAX_FLOAT | 应从前多少个 sv_steamgroup 组中排除玩家，0 检查所有配置的组 | 服务器端本插件逻辑 | `veterans.sp:157` |
| `sm_veterans_bantime` | `10` | 整数，源码写作浮点 | 0.0 ~ MAX_FLOAT | 是否改为封禁而非踢出，以及封禁分钟数，0 表示不封禁 | 服务器端本插件逻辑 | `veterans.sp:172` |
| `sm_veterans_mintotal` | `0` | 整数，源码写作浮点 | 0.0 ~ MAX_FLOAT | 玩家需要的最低总游戏时长，单位分钟 | 服务器端本插件逻辑 | `veterans.sp:178` |
| `sm_veterans_minServertotal` | `0` | 整数，源码写作浮点 | 0.0 ~ MAX_FLOAT | 玩家需要的最低本服总游戏时长，单位分钟 | 服务器端本插件逻辑 | `veterans.sp:184` |
| `sm_veterans_mintotalminuslastweeks` | `0` | 整数，源码写作浮点 | 0.0 ~ MAX_FLOAT | 玩家需要的最低总游戏时长（不含最近两周），单位分钟 | 服务器端本插件逻辑 | `veterans.sp:190` |
| `sm_veterans_cachetime` | `14400` | 整数，源码写作浮点 | 0.0 ~ MAX_FLOAT | 对同一查询不重复发送请求的缓存秒数 | 服务器端本插件逻辑 | `veterans.sp:197` |

## Vote for run command or cfg file —— `extend/vote.sp`

- 源文件：`extend/vote.sp`
- myinfo：version=1.5，author=东
- myinfo description（源码原文）：使用!vote投票执行命令或cfg文件

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_vote` | <cfg 名> | 无（任意玩家可用） | 发起投票执行指定的 cfg 或命令 | `vote.sp:80` |
| `sm_votekick` | 无参数 | 无（任意玩家可用） | 打开投票踢人菜单 | `vote.sp:81` |
| `sm_voteban` | 无参数 | 无（任意玩家可用） | 打开投票封禁菜单 | `vote.sp:82` |
| `sm_cancelvote` | 无参数 | 需要 ADMFLAG_GENERIC（b，通用管理员） | 终止当前正在进行的投票 | `vote.sp:83` |

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `votecfgfile` | `VOTE_DEFAULT_CONFIG` | 字符串或表达式 | 无上下界 | 投票文件的位置，位于 sourcemod/ 文件夹下 | 服务器端本插件逻辑 | `vote.sp:77` |

## [L4D/2] Fix Pill Passing —— `fix_pill_pass.sp`

- 源文件：`fix_pill_pass.sp`
- myinfo：version=1.0，author=Alan, A1m`, Forgetest, Sir
- myinfo description（源码原文）：Fixes being unable to pass Pills to Survivor considered 'In Combat' + Pills being thrown away.

### 指令

（本插件未注册指令）

### ConVar

（本插件未注册 ConVar）

## Tickrate Fixes —— `fixes/TickrateFixes.sp`

- 源文件：`fixes/TickrateFixes.sp`
- myinfo：version=1.4.1，author=Sir, Griffin, A1m`
- myinfo description（源码原文）：Fixes a handful of silly Tickrate bugs

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `tick_door_speed` | `1.3` | 浮点 | 无上下界 | 设置地图上所有 prop_door 实体的速度，1.05 表示 105% 速度 | 服务器端本插件逻辑 | `TickrateFixes.sp:71` |

## [L4D2] Annoyance/Exploit Fixes —— `fixes/annoyance_exploit_fixes.sp`

- 源文件：`fixes/annoyance_exploit_fixes.sp`
- myinfo：version=0.2.1，author=Sir
- myinfo description（源码原文）：A compilation of 'fixes' to deal with annoyances and exploits.

### 指令

（本插件未注册指令）

### ConVar

（本插件未注册 ConVar）

## BeQuiet —— `fixes/bequiet.sp`

- 源文件：`fixes/bequiet.sp`
- myinfo：version=1.33.7，author=Sir
- myinfo description（源码原文）：Please be Quiet!

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `bq_cvar_change_suppress` | `1` | 整数 | 无上下界 | 是否屏蔽服务器 cvar 变更提示，使聊天更干净 | 服务器端本插件逻辑 | `bequiet.sp:30` |
| `bq_name_change_suppress` | `1` | 整数 | 无上下界 | 是否屏蔽玩家改名提示 | 服务器端本插件逻辑 | `bequiet.sp:31` |
| `bq_name_change_spec_suppress` | `1` | 整数 | 无上下界 | 是否屏蔽旁观玩家改名的提示 | 服务器端本插件逻辑 | `bequiet.sp:32` |
| `bq_show_player_team_chat_spec` | `1` | 整数 | 无上下界 | 是否向旁观者显示生还者与特感的团队聊天 | 服务器端本插件逻辑 | `bequiet.sp:33` |

## [ANY] Command and ConVar - Buffer Overflow Fixer —— `fixes/command_buffer.sp`

- 源文件：`fixes/command_buffer.sp`
- myinfo：author=SilverShot and Peace-Maker
- myinfo description（源码原文）：Fixes the 'Cbuf_AddText: buffer overflow' console error on servers, which causes ConVars to use their default value.

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_cvar_test` | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 创建一个测试 ConVar 并输出其结果，用于缓冲区溢出修复验证 | `command_buffer.sp:153` |

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `command_buffer_version` | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | 服务器端本插件逻辑；不写入 cfg 存档 | `command_buffer.sp:116` |
| `sm_cvar_test_N` | `0` | 整数 | 无上下界 | 仅当源码 DEBUGGING 宏为 1 时创建；名称为 sm_cvar_test_0 到 MAX_CVARS-1，默认值全部为 0 | 服务器端本插件逻辑 | `command_buffer.sp:150` |

## Bullet position fix —— `fixes/firebulletsfix.sp`

- 源文件：`fixes/firebulletsfix.sp`
- myinfo：version=1.0.3，author=xutaxkamay
- myinfo description（源码原文）：Fixes shoot position

### 指令

（本插件未注册指令）

### ConVar

（本插件未注册 ConVar）

## Fast melee fix —— `fixes/fix_fastmelee.sp`

- 源文件：`fixes/fix_fastmelee.sp`
- myinfo：version=2.3，author=sheo
- myinfo description（源码原文）：Fixes the bug with too fast melee attacks

### 指令

（本插件未注册指令）

### ConVar

（本插件未注册 ConVar）

## Fix frozen tanks —— `fixes/frozen_tank_fix.sp`

- 源文件：`fixes/frozen_tank_fix.sp`
- myinfo：version=2.2，author=sheo

### 指令

（本插件未注册指令）

### ConVar

（本插件未注册 ConVar）

## Bot SI skeet/level damage fix —— `fixes/l4d2_ai_damagefix.sp`

- 源文件：`fixes/l4d2_ai_damagefix.sp`
- myinfo：version=1.1.0，author=Tabun, dcx2
- myinfo description（源码原文）：Makes AI SI take (and do) damage like human SI.

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `sm_aidmgfix_enable` | `3` | 整数 | 无上下界 | 位标志：1 修复飞扑中的 AI 被击杀判定，2 削弱冲锋中的 AI，3 全部启用，0 关闭 | 服务器端本插件逻辑 | `l4d2_ai_damagefix.sp:99` |

## [L4D & L4D2] Additive Staged FastDL —— `fixes/l4d2_blackscreen_fix.sp`

- 源文件：`fixes/l4d2_blackscreen_fix.sp`
- myinfo：version=1.3.0，author=BHaType, Dragokas, AnneHappy
- myinfo description（源码原文）：Adds priority downloads on map start and optional assets in batches during map transitions

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_get_restricted_strings` | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 列出玩家首次连接时会加入的受限文件清单 | `l4d2_blackscreen_fix.sp:63` |

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `sm_fixscreen_deferred_group_count` | `8` | 整数，源码写作浮点 | 0.0 ~ float(MAX_DEFERRED_GROUPS) | 用于拆分延迟下载文件的分组数量，每次真实换图加入一组，0 表示关闭延迟下载 | 服务器端本插件逻辑 | `l4d2_blackscreen_fix.sp:52` |

## [L4D2] Block No Steam Logon —— `fixes/l4d2_block_no_steam_logon.sp`

- 源文件：`fixes/l4d2_block_no_steam_logon.sp`
- myinfo：author=blueblur, AnneHappy
- myinfo description（源码原文）：Bypasses Steam auth responses 1 and 6 while preserving other auth failures.

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `l4d2_block_no_steam_logon_version` | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | 服务器端本插件逻辑；不写入 cfg 存档 | `l4d2_block_no_steam_logon.sp:109` |
| `l4d2_block_no_steam_logon_enable` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否阻止 Steam 鉴权响应 1 与 6 导致的客户端断线 | 服务器端本插件逻辑 | `l4d2_block_no_steam_logon.sp:116` |
| `l4d2_block_no_steam_logon_check_timeout` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 当客户端网络通道超时后是否断开该客户端 | 服务器端本插件逻辑 | `l4d2_block_no_steam_logon.sp:128` |

## [L4D2] Boomer Ladder Fix —— `fixes/l4d2_boomer_ladder_fix.sp`

- 源文件：`fixes/l4d2_boomer_ladder_fix.sp`
- myinfo：author=BHaType

### 指令

（本插件未注册指令）

### ConVar

（本插件未注册 ConVar）

## L4D2 Boomer Shenanigans —— `fixes/l4d2_boomer_shenanigans.sp`

- 源文件：`fixes/l4d2_boomer_shenanigans.sp`
- myinfo：version=1.0，author=Sir
- myinfo description（源码原文）：Make sure Boomers are unable to bile Survivors during a stumble (basically reinforce shoves)

### 指令

（本插件未注册指令）

### ConVar

（本插件未注册 ConVar）

## L4D2 Car Alarm Fixes —— `fixes/l4d2_car_alarm_hittable_fix.sp`

- 源文件：`fixes/l4d2_car_alarm_hittable_fix.sp`
- myinfo：version=1.2，author=Sir & Silvers (Gamedata and general idea from l4d2_car_alarm_bots)
- myinfo description（源码原文）：Disables the Car Alarm when a Tank hittable hits the alarmed car and makes sure the Car Alarm triggers whenever a Survivor touches it

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `l4d2_car_alarm_settings` | `3` | 整数 | 无上下界 | 位掩码：1 生还者触碰时触发警报，2 可击打物击中警报车时禁用警报 | 服务器端本插件逻辑 | `l4d2_car_alarm_hittable_fix.sp:80` |
| `l4d2_car_alarm_touch_capped` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 仅在生还者触碰车时被特感控制才追加警报触发，需位掩码设置生效 | 服务器端本插件逻辑 | `l4d2_car_alarm_hittable_fix.sp:81` |
| `l4d2_car_alarm_touch_ai` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否统计 AI 生还者触碰警报车，需位掩码设置生效 | 服务器端本插件逻辑 | `l4d2_car_alarm_hittable_fix.sp:82` |

## [L4D2] Charger Target Fix —— `fixes/l4d2_charge_target_fix.sp`

- 源文件：`fixes/l4d2_charge_target_fix.sp`
- myinfo：author=Forgetest
- myinfo description（源码原文）：Fix multiple issues with charger targets.

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `z_charge_pinned_collision` | `3` | 整数，源码写作浮点 | 0.0 ~ 3.0 | 被 Charger 控制的生还者能否与特感碰撞：1 压制期间，2 起身期间，3 两者，0 完全不碰撞 | 服务器端本插件逻辑；仅服务器；经 CreateConVarHook 包装并注册变更回调 | `l4d2_charge_target_fix.sp:88` |
| `charger_knockdown_getup_window` | `0.1` | 浮点 | 0.0 ~ 4.0 | 击倒计时结束到起身完成之间的时长，值越大起身时越早变回可碰撞 | 服务器端本插件逻辑；仅服务器；经 CreateConVarHook 包装并注册变更回调 | `l4d2_charge_target_fix.sp:96` |

## L4D2 Ellis Hunter Band aid Fix —— `fixes/l4d2_ellis_hunter_bandaid_fix.sp`

- 源文件：`fixes/l4d2_ellis_hunter_bandaid_fix.sp`
- myinfo：version=1.2，author=Sir (with pointers from Rena)
- myinfo description（源码原文）：Band-aid fix for Ellis' getup not matching the other Survivors

### 指令

（本插件未注册指令）

### ConVar

（本插件未注册 ConVar）

## L4D2 Explosion Damage Prevention —— `fixes/l4d2_explosiondmg_prev.sp`

- 源文件：`fixes/l4d2_explosiondmg_prev.sp`
- myinfo：version=1.1，author=Sir, A1m`
- myinfo description（源码原文）：No more explosion damage to the infected from entity

### 指令

（本插件未注册指令）

### ConVar

（本插件未注册 ConVar）

## [L4D2] Fix Changelevel —— `fixes/l4d2_fix_changelevel.sp`

- 源文件：`fixes/l4d2_fix_changelevel.sp`
- myinfo：author=Lux (for \
- myinfo description（源码原文）：Fix issues due to forced changelevel + resolve partial map names before the engine sees them.

### 指令

（本插件未注册指令）

### ConVar

（本插件未注册 ConVar）

## [L4D2] Fix First-Hit —— `fixes/l4d2_fix_firsthit.sp`

- 源文件：`fixes/l4d2_fix_firsthit.sp`
- myinfo：author=Forgetest
- myinfo description（源码原文）：Fix first hit classes varying between halves and in scavenge staying the same for rounds.

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `l4d2_scvng_firsthit_shuffle` | `0` | 整数，源码写作浮点 | 0.0 ~ 2.0 | 是否打乱首发特感职业，仅影响清道夫模式：1 每回合打乱，2 每场比赛打乱，其余被截断 | 服务器端本插件逻辑；仅服务器 | `l4d2_fix_firsthit.sp:39` |

## [L4D2] Fix Rocket Pull —— `fixes/l4d2_fix_rocket_pull.sp`

- 源文件：`fixes/l4d2_fix_rocket_pull.sp`
- myinfo：version=0.2，author=Alan, Forgetest
- myinfo description（源码原文）：Fix smoker pull launching survivor up

### 指令

（本插件未注册指令）

### ConVar

（本插件未注册 ConVar）

## [L4D2] Fix Tank Rock Stuck —— `fixes/l4d2_fix_tank_rock_handoff.sp`

- 源文件：`fixes/l4d2_fix_tank_rock_handoff.sp`
- myinfo：author=Sir
- myinfo description（源码原文）：Cancels tank rocks that are in the middle of a throw when passing to another player/AI.

### 指令

（本插件未注册指令）

### ConVar

（本插件未注册 ConVar）

## L4D2 - Fix team shuffle —— `fixes/l4d2_fix_team_shuffle.sp`

- 源文件：`fixes/l4d2_fix_team_shuffle.sp`
- myinfo：version=1.0.1，author=Altair Sossai
- myinfo description（源码原文）：Fix teams shuffling during map switching

### 指令

（本插件未注册指令）

### ConVar

（本插件未注册 ConVar）

## L4D2 HLTV Crash Exploit Fix —— `fixes/l4d2_hltv_crash_fix.sp`

- 源文件：`fixes/l4d2_hltv_crash_fix.sp`
- myinfo：version=2.2，author=backwards, ProdigySim, A1m`
- myinfo description（源码原文）：Prevents Exploit That Crashes Servers

### 指令

（本插件未注册指令）

### ConVar

（本插件未注册 ConVar）

## [L4D2] Fix Jockey Hitbox —— `fixes/l4d2_jockey_hitbox_fix.sp`

- 源文件：`fixes/l4d2_jockey_hitbox_fix.sp`
- myinfo：author=Forgetest
- myinfo description（源码原文）：Fix jockey hitbox issues when riding survivors.

### 指令

（本插件未注册指令）

### ConVar

（本插件未注册 ConVar）

## L4D2 Jockey Jump-Cap Patch —— `fixes/l4d2_jockey_jumpcap_patch.sp`

- 源文件：`fixes/l4d2_jockey_jumpcap_patch.sp`
- myinfo：version=1.6，author=Visor, A1m`
- myinfo description（源码原文）：Prevent Jockeys from being able to land caps with non-ability jumps in unfair situations

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `l4d2_jumpcap_block_time` | `3.0` | 整数，源码写作浮点 | 1.0 ~ 10.0 | 设置 Jockey 跳扑被封锁的持续时间，单位秒，范围 1~10 | 服务器端本插件逻辑 | `l4d2_jockey_jumpcap_patch.sp:40` |

## Jockey Unteleport —— `fixes/l4d2_jockey_unteleport.sp`

- 源文件：`fixes/l4d2_jockey_unteleport.sp`
- myinfo：version=2.0，author=Krevik, larrybrains
- myinfo description（源码原文）：Teleports a survivor back into the map if they are randomly teleported outside or inside of the map while jockeyed.

### 指令

（本插件未注册指令）

### ConVar

（本插件未注册 ConVar）

## L4D2 Jockeyed Survivor Ladder Fix —— `fixes/l4d2_jockeyed_ladder_fix.sp`

- 源文件：`fixes/l4d2_jockeyed_ladder_fix.sp`
- myinfo：version=1.2，author=Visor, A1m`
- myinfo description（源码原文）：Fixes jockeyed Survivors slowly sliding down the ladders

### 指令

（本插件未注册指令）

### ConVar

（本插件未注册 ConVar）

## [L4D2] Ladder Server Crash - Patch Fix —— `fixes/l4d2_ladder_patch.sp`

- 源文件：`fixes/l4d2_ladder_patch.sp`
- myinfo：author=SilverShot and Peace-Maker
- myinfo description（源码原文）：Fixes a server crash from NavLadder::GetPosAtHeight. Patches out AvoidNeighbors.

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `l4d2_ladder_patch_version` | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | 服务器端本插件逻辑；不写入 cfg 存档 | `l4d2_ladder_patch.sp:85` |

## StopTrolls —— `fixes/l4d2_ladderblock.sp`

- 源文件：`fixes/l4d2_ladderblock.sp`
- myinfo：author=raziEiL [disawar1]
- myinfo description（源码原文）：Prevents people from blocking players who climb on the ladder.
- HookConVarChange：`g_hFlags`→`OnCvarChange_Flags`、`g_hImmune`→`OnCvarChange_Immune`

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `stop_trolls_version` | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | 服务器端本插件逻辑；复制到客户端 | `l4d2_ladderblock.sp:72` |
| `stop_trolls_flags` | `862` | 整数 | 无上下界 | 谁可以在爬梯时推动巨魔：0 关闭，2 Smoker，4 Boomer，8 Hunter，16 Spitter，64 Charger，256 Tank，可相加 | 服务器端本插件逻辑 | `l4d2_ladderblock.sp:74` |
| `stop_trolls_immune` | `256` | 整数 | 无上下界 | 什么职业免疫：0 关闭，2 Smoker，4 Boomer，8 Hunter，16 Spitter，32 Jockey，64 Charger，256 Tank，512 Survivor，可相加 | 服务器端本插件逻辑 | `l4d2_ladderblock.sp:75` |

## L4D2 Lag Compensation Manager —— `fixes/l4d2_lagcomp_manager.sp`

- 源文件：`fixes/l4d2_lagcomp_manager.sp`
- myinfo：version=1.3，author=ProdigySim, A1m`, Forgetest
- myinfo description（源码原文）：Provides lag compensation for entities in left 4 dead 2 (required enable sv_unlag).

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_show_lagcomp_list` | 无参数 | 无（任意玩家可用） | 把延迟补偿数组打印到服务器控制台 | `l4d2_lagcomp_manager.sp:173` |

### ConVar

（本插件未注册 ConVar）

## L4D2 Melee Damage Fix&Control —— `fixes/l4d2_melee_damage_control.sp`

- 源文件：`fixes/l4d2_melee_damage_control.sp`
- myinfo：version=2.1，author=Visor, Sir, A1m`
- myinfo description（源码原文）：Fix melees weapons not applying correct damage values on infected. Allows manipulate melee damage on some infected.

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `l4d2_melee_damage_fix` | `1.0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否修复近战武器对感染者伤害不正确的问题，即伤害不再取决于命中部位 | 服务器端本插件逻辑 | `l4d2_melee_damage_control.sp:77` |
| `l4d2_melee_damage_tank_nerf` | `-1.0` | 整数，源码写作浮点 | ≤ 100.0 | 近战对 Tank 伤害的削弱百分比，0 或负值表示关闭 | 服务器端本插件逻辑 | `l4d2_melee_damage_control.sp:85` |
| `l4d2_melee_damage_charger` | `-1.0` | 整数，源码写作浮点 | 无上下界 | 每次挥击对 Charger 的近战伤害，0 或负值表示关闭 | 服务器端本插件逻辑 | `l4d2_melee_damage_control.sp:92` |

## L4D2 No Post-Jockeyed Shoves —— `fixes/l4d2_no_post_jockey_deadstops.sp`

- 源文件：`fixes/l4d2_no_post_jockey_deadstops.sp`
- myinfo：version=1.0，author=Sir
- myinfo description（源码原文）：L4D2 has a nasty bug which Survivors would exploit and this fixes that. (Holding out a melee and spamming shove, even if the jockey was behind you, would self-clear yourself after the Jockey actually landed.

### 指令

（本插件未注册指令）

### ConVar

（本插件未注册 ConVar）

## L4D2 Lag Compensation Null CUserCmd fix —— `fixes/l4d2_null_cusercmd_fix.sp`

- 源文件：`fixes/l4d2_null_cusercmd_fix.sp`
- myinfo：author=fdxx
- myinfo description（源码原文）：Prevent crash: CLagCompensationManager::StartLagCompensation with NULL CUserCmd!!!

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `l4d2_null_cusercmd_fix_version` | `VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（默认值取自 VERSION 宏） | 服务器端本插件逻辑；不写入 cfg 存档 | `l4d2_null_cusercmd_fix.sp:20` |

## L4D2 pistol delay —— `fixes/l4d2_pistol_delay.sp`

- 源文件：`fixes/l4d2_pistol_delay.sp`
- myinfo：version=1.3，author=A1m`
- myinfo description（源码原文）：Allows you to adjust the rate of fire of pistols (with a high tickrate, the rate of fire of dual pistols is very high).

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `l4d_pistol_delay_dualies` | `0.1` | 浮点 | MIN_RATE_OF_FIRE ~ MAX_RATE_OF_FIRE | 双持手枪两次射击之间的最短秒数 | 服务器端本插件逻辑 | `l4d2_pistol_delay.sp:71` |
| `l4d_pistol_delay_single` | `sDefValue` | 字符串或表达式 | MIN_RATE_OF_FIRE ~ MAX_RATE_OF_FIRE | 单持手枪两次射击之间的最短秒数 | 服务器端本插件逻辑 | `l4d2_pistol_delay.sp:81` |
| `l4d_automatic_pistol` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 按住攻击键时手枪能否连续射击 | 服务器端本插件逻辑 | `l4d2_pistol_delay.sp:92` |

## [L4D2] Rock Trace Unblock —— `fixes/l4d2_rock_trace_unblock.sp`

- 源文件：`fixes/l4d2_rock_trace_unblock.sp`
- myinfo：author=Forgetest
- myinfo description（源码原文）：Prevent hunter/jockey/coinciding survivor from blocking the rock radius check.

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `l4d2_rock_trace_unblock_flag` | `5` | 整数，源码写作浮点 | 0.0 ~ 31.0 | 位掩码：阻止特感挡住石头半径检测，1 所有站立的特感，2 扑中的，4 其余被截断 | 服务器端本插件逻辑；需 sv_cheats 方可修改 | `l4d2_rock_trace_unblock.sp:74` |
| `l4d2_rock_jockey_dismount` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否强制 Jockey 从被石头击中的生还者身上下来：1 开，0 关 | 服务器端本插件逻辑；需 sv_cheats 方可修改 | `l4d2_rock_trace_unblock.sp:82` |
| `l4d2_rock_hurt_capper` | `5` | 整数，源码写作浮点 | 0.0 ~ 7.0 | 是否在控制生效前先伤害控制者：1 Hunter，2 Jockey，4 Charger，7 全部，0 关闭 | 服务器端本插件逻辑；需 sv_cheats 方可修改 | `l4d2_rock_trace_unblock.sp:90` |

## [L4D2] Script Command Swap - Mem Leak Fix —— `fixes/l4d2_script_cmd_swap.sp`

- 源文件：`fixes/l4d2_script_cmd_swap.sp`
- myinfo：author=SilverShot (Timocop's idea)
- myinfo description（源码原文）：Blocks the script command and replaces with a logic_script entity to execute the code instead.

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `l4d2_script_cmd_swap_version` | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | 服务器端本插件逻辑；不写入 cfg 存档 | `l4d2_script_cmd_swap.sp:34` |

## [L4D & 2] Scripted Tank Stage Fix —— `fixes/l4d2_scripted_tank_stage_fix.sp`

- 源文件：`fixes/l4d2_scripted_tank_stage_fix.sp`
- myinfo：author=Forgetest
- myinfo description（源码原文）：Fix some issues of skipping stages regarding Tanks in finale.

### 指令

（本插件未注册指令）

### ConVar

（本插件未注册 ConVar）

## [L4D2] SG552 - Zoom fix —— `fixes/l4d2_sg552_zoom_fix.sp`

- 源文件：`fixes/l4d2_sg552_zoom_fix.sp`
- myinfo：version=1.0，author=Altair Sossai
- myinfo description（源码原文）：Fix SG552 zoom, preventing the player's camera from getting stuck

### 指令

（本插件未注册指令）

### ConVar

（本插件未注册 ConVar）

## L4D2 Shadow Removal —— `fixes/l4d2_shadow_removal.sp`

- 源文件：`fixes/l4d2_shadow_removal.sp`
- myinfo：version=1.1，author=Sir
- myinfo description（源码原文）：A plugin that removes Shadows so that Survivors can't see Infected Players their shadows through walls and the like.

### 指令

（本插件未注册指令）

### ConVar

（本插件未注册 ConVar）

## [L4D2] Shove Direction Fix —— `fixes/l4d2_shove_fix.sp`

- 源文件：`fixes/l4d2_shove_fix.sp`
- myinfo：author=BHaType

### 指令

（本插件未注册指令）

### ConVar

（本插件未注册 ConVar）

## [L4D2] Spit Cooldown Frozen Fix —— `fixes/l4d2_spit_cooldown_frozen_fix.sp`

- 源文件：`fixes/l4d2_spit_cooldown_frozen_fix.sp`
- myinfo：author=Forgetest
- myinfo description（源码原文）：Simple fix for spit cooldown being \

### 指令

（本插件未注册指令）

### ConVar

（本插件未注册 ConVar）

## [L4D2] Spit Spread Patch —— `fixes/l4d2_spit_spread_patch.sp`

- 源文件：`fixes/l4d2_spit_spread_patch.sp`
- myinfo：author=Forgetest
- myinfo description（源码原文）：Fix various spit spread issues.

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `spit_spread_saferoom_except` | <地图名> | 服务器控制台命令，玩家无法使用 | 把指定地图排除在毒痰扩散修正之外，服务器控制台命令 | `l4d2_spit_spread_patch.sp:202` |

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `l4d2_spit_spread_saferoom` | `0` | 开关 0 或 1 | 0.0 ~ 2.0 | 毒痰在安全室区域的扩散方式：0 不扩散，1 在开场起始区域扩散，2 每张地图都扩散 | 服务器端本插件逻辑；仅服务器；经 CreateConVarHook 包装并注册变更回调 | `l4d2_spit_spread_patch.sp:161` |
| `l4d2_deathspit_trace_height` | `240.0` | 整数，源码写作浮点 | ≥ 0.0 | 死亡毒痰检测射线的长度，240.0 为默认长度 | 服务器端本插件逻辑；仅服务器；经 CreateConVarHook 包装并注册变更回调 | `l4d2_spit_spread_patch.sp:169` |
| `l4d2_spit_max_flames` | `10` | 整数，源码写作浮点 | ≥ 2.0 | 一次普通毒痰最多生成的痰池数量，最小 2，游戏默认 10 | 服务器端本插件逻辑；仅服务器；经 CreateConVarHook 包装并注册变更回调 | `l4d2_spit_spread_patch.sp:177` |
| `l4d2_spit_water_collision` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 毒痰投射物是否会与水碰撞：0 不碰撞，1 碰撞 | 服务器端本插件逻辑；仅服务器；经 CreateConVarHook 包装并注册变更回调 | `l4d2_spit_spread_patch.sp:185` |
| `l4d2_spit_prop_damage` | `10.0` | 整数，源码写作浮点 | ≥ 0.0 | 投射物弹跳时对道具造成的伤害，0 表示不造成伤害 | 服务器端本插件逻辑；仅服务器；经 CreateConVarHook 包装并注册变更回调 | `l4d2_spit_spread_patch.sp:193` |

## [L4D2] Flying Incap - Tank Punch —— `fixes/l4d2_tank_flying_incap.sp`

- 源文件：`fixes/l4d2_tank_flying_incap.sp`
- myinfo：version=2.0，author=Sir, Forgetest
- myinfo description（源码原文）：Sends Survivors flying on the incapping punch.

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `l4d2_tank_flying_incap_debug` | `0` | 整数 | 无上下界 | 是否输出调试信息 | 服务器端本插件逻辑 | `l4d2_tank_flying_incap.sp:31` |
| `l4d2_tank_flying_incap_anim_fix` | `0` | 整数 | 无上下界 | 是否移除飞翔结束时的起身动画，移除后生还者落地即可射击 | 服务器端本插件逻辑 | `l4d2_tank_flying_incap.sp:32` |

## [L4D2] Tank Spawn Anti-Rock Protect —— `fixes/l4d2_tank_spawn_antirock_protect.sp`

- 源文件：`fixes/l4d2_tank_spawn_antirock_protect.sp`
- myinfo：version=1.0.2，author=B[R]UTUS
- myinfo description（源码原文）：Protects a Tank player from randomly rock attack at his spawn

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `l4d2_antirock_protect_time` | `1.5` | 浮点 | 无上下界 | 保护时间秒数，避免 Tank 意外投掷石头 | 服务器端本插件逻辑 | `l4d2_tank_spawn_antirock_protect.sp:21` |

## Transition Info Fix —— `fixes/l4d2_transition_info_fix.sp`

- 源文件：`fixes/l4d2_transition_info_fix.sp`
- myinfo：version=1.0，author=IA/NanaNana
- myinfo description（源码原文）：Fix the transition info bug

### 指令

（本插件未注册指令）

### ConVar

（本插件未注册 ConVar）

## [L4D & 2] Backjump Fix —— `fixes/l4d_backjump_fix.sp`

- 源文件：`fixes/l4d_backjump_fix.sp`
- myinfo：author=Forgetest
- myinfo description（源码原文）：Fix hunter being unable to pounce off non-static props

### 指令

（本插件未注册指令）

### ConVar

（本插件未注册 ConVar）

## [L4D & 2] Consistent Escape Route —— `fixes/l4d_consistent_escaperoute.sp`

- 源文件：`fixes/l4d_consistent_escaperoute.sp`
- myinfo：author=Forgetest
- myinfo description（源码原文）：True L4D.

### 指令

（本插件未注册指令）

### ConVar

（本插件未注册 ConVar）

## [L4D & L4D2] Console Spam Patches —— `fixes/l4d_console_spam.sp`

- 源文件：`fixes/l4d_console_spam.sp`
- myinfo：author=SilverShot
- myinfo description（源码原文）：Prevents certain errors/warnings from being displayed in the server console.

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `l4d_console_spam_version` | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | 服务器端本插件逻辑；不写入 cfg 存档 | `l4d_console_spam.sp:90` |

## [L4D & 2] Fix Common Shove —— `fixes/l4d_fix_common_shove.sp`

- 源文件：`fixes/l4d_fix_common_shove.sp`
- myinfo：author=Forgetest
- myinfo description（源码原文）：Fix commons being immune to shoves when crouching, falling and landing.

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `l4d_common_shove_flag` | `15` | 整数，源码写作浮点 | ≥ 0.0 | 修复普通感染者推击的位标志：1 蹲下，2 下落，4 落地，8 攀爬 | 服务器端本插件逻辑；需 sv_cheats 方可修改；经 CreateConVarHook 包装并注册变更回调 | `l4d_fix_common_shove.sp:140` |

## [L4D2] Fix DeathFall Camera —— `fixes/l4d_fix_deathfall_cam.sp`

- 源文件：`fixes/l4d_fix_deathfall_cam.sp`
- myinfo：author=Forgetest
- myinfo description（源码原文）：Prevent \

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `l4d2_fix_deathfall_cam_version` | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | 服务器端本插件逻辑；复制到客户端；仅服务器；不写入 cfg 存档 | `l4d_fix_deathfall_cam.sp:36` |

## [L4D & 2] Fix Finale Breakable —— `fixes/l4d_fix_finale_breakable.sp`

- 源文件：`fixes/l4d_fix_finale_breakable.sp`
- myinfo：author=Forgetest
- myinfo description（源码原文）：Fix SI being unable to break props/walls within finale area before finale starts.

### 指令

（本插件未注册指令）

### ConVar

（本插件未注册 ConVar）

## [L4D & 2] Fix Prop LOS —— `fixes/l4d_fix_prop_los.sp`

- 源文件：`fixes/l4d_fix_prop_los.sp`
- myinfo：author=Forgetest
- myinfo description（源码原文）：Fix thin/small 'prop_*' entity not blocking LOS.

### 指令

（本插件未注册指令）

### ConVar

（本插件未注册 ConVar）

## [L4D & 2] Fix Punch Block —— `fixes/l4d_fix_punch_block.sp`

- 源文件：`fixes/l4d_fix_punch_block.sp`
- myinfo：author=Forgetest
- myinfo description（源码原文）：Fix common infected blocking the punch tracing.

### 指令

（本插件未注册指令）

### ConVar

（本插件未注册 ConVar）

## [L4D & 2] Fix Rocket Jump —— `fixes/l4d_fix_rocket_jump.sp`

- 源文件：`fixes/l4d_fix_rocket_jump.sp`
- myinfo：author=Forgetest
- myinfo description（源码原文）：Fix some \

### 指令

（本插件未注册指令）

### ConVar

（本插件未注册 ConVar）

## [L4D & 2] Entity VPhysics Solver —— `fixes/l4d_fix_rotated_physblocker.sp`

- 源文件：`fixes/l4d_fix_rotated_physblocker.sp`
- myinfo：author=Forgetest
- myinfo description（源码原文）：Fix rotated \

### 指令

（本插件未注册指令）

### ConVar

（本插件未注册 ConVar）

## [L4D & 2] Fix Saferoom Ghost Spawn —— `fixes/l4d_fix_saferoom_ghostspawn.sp`

- 源文件：`fixes/l4d_fix_saferoom_ghostspawn.sp`
- myinfo：author=Forgetest
- myinfo description（源码原文）：Fix a glitch that ghost can spawn in saferoom while it shouldn't.

### 指令

（本插件未注册指令）

### ConVar

（本插件未注册 ConVar）

## [L4D & 2] Fix Shove Duration —— `fixes/l4d_fix_shove_duration.sp`

- 源文件：`fixes/l4d_fix_shove_duration.sp`
- myinfo：author=Forgetest
- myinfo description（源码原文）：Fix SI getting shoved by \

### 指令

（本插件未注册指令）

### ConVar

（本插件未注册 ConVar）

## [L4D & 2] Fix Smoker Bot Float —— `fixes/l4d_fix_smokerbot_float.sp`

- 源文件：`fixes/l4d_fix_smokerbot_float.sp`
- myinfo：author=Forgetest
- myinfo description（源码原文）：Fixes smoker bot floating in mid-air using a binary patch.

### 指令

（本插件未注册指令）

### ConVar

（本插件未注册 ConVar）

## [L4D & 2] Fix Stagger Direction —— `fixes/l4d_fix_stagger_dir.sp`

- 源文件：`fixes/l4d_fix_stagger_dir.sp`
- myinfo：author=Forgetest
- myinfo description（源码原文）：Fix survivors getting stumbled to the \

### 指令

（本插件未注册指令）

### ConVar

（本插件未注册 ConVar）

## [L4D & 2] Prop Touching Rules —— `fixes/l4d_prop_touching_rules.sp`

- 源文件：`fixes/l4d_prop_touching_rules.sp`
- myinfo：author=Forgetest
- myinfo description（源码原文）：Rules of props' move away, moved above props.

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `prop_moveaway_mass_thres` | `900.0` | 整数，源码写作浮点 | ≥ 0.0 | 允许被推开的中等重量道具的最大质量，仅当 prop_touching_moveaway 开启时有效 | 服务器端本插件逻辑；经 CreateConVarHook 包装并注册变更回调 | `l4d_prop_touching_rules.sp:66` |
| `prop_touching_moveaway` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否在触碰时推开中等重量道具 | 服务器端本插件逻辑；经 CreateConVarHook 包装并注册变更回调 | `l4d_prop_touching_rules.sp:72` |
| `prop_heavy_touching_move_above` | `0` | 开关 0 或 1 | ≥ 0.0 | 是否阻止玩家被推上重型道具上方：0 关闭，1 生还者，2 除坦克外的特感，4 坦克，7 全部 | 服务器端本插件逻辑；经 CreateConVarHook 包装并注册变更回调 | `l4d_prop_touching_rules.sp:78` |

## [L4D & L4D2] First Map - Skip Intro Cutscenes —— `fixes/l4d_skip_intro.sp`

- 源文件：`fixes/l4d_skip_intro.sp`
- myinfo：author=SilverShot
- myinfo description（源码原文）：Makes players skip seeing the intro cutscene on first maps, so they can move right away.

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `l4d_skip_intro_allow` | `1` | 整数 | 无上下界 | 0 关闭插件，1 开启插件 | 服务器端本插件逻辑 | `l4d_skip_intro.sp:133` |
| `l4d_skip_intro_modes` | `` | 字符串或表达式 | 无上下界 | 在这些游戏模式中启用插件，逗号分隔，留空为全部模式 | 服务器端本插件逻辑 | `l4d_skip_intro.sp:134` |
| `l4d_skip_intro_modes_off` | `` | 字符串或表达式 | 无上下界 | 在这些游戏模式中关闭插件，逗号分隔，留空为不关闭 | 服务器端本插件逻辑 | `l4d_skip_intro.sp:135` |
| `l4d_skip_intro_modes_tog` | `0` | 整数 | 无上下界 | 按模式位掩码启用插件：0 全部，1 战役，2 生存，4 对战，8 清道夫，可相加 | 服务器端本插件逻辑 | `l4d_skip_intro.sp:136` |
| `l4d_skip_intro_version` | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | 服务器端本插件逻辑；不写入 cfg 存档 | `l4d_skip_intro.sp:137` |

## [L4D & 2] Static Punch Get-up —— `fixes/l4d_static_punch_getup.sp`

- 源文件：`fixes/l4d_static_punch_getup.sp`
- myinfo：author=Forgetest
- myinfo description（源码原文）：Fix punch get-up varying in length, along with flexible setting to it.

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `tank_punch_getup_scale` | `0.5` | 浮点 | 0.01 ~ 0.99 | Tank 拳击落地起身动画长度的缩放比例，范围 0.01~0.99 | 服务器端本插件逻辑；仅服务器 | `l4d_static_punch_getup.sp:52` |

## [L4D & 2] Tongue Bend Fix —— `fixes/l4d_tongue_bend_fix.sp`

- 源文件：`fixes/l4d_tongue_bend_fix.sp`
- myinfo：author=Forgetest
- myinfo description（源码原文）：Fix unexpected tongue breaks for \

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `tongue_bend_exception_flag` | `1` | 整数，源码写作浮点 | ≥ 0.0 | 允许舌头抓取特定类型实体的 flag：1 门，2 可搬运物，3 全部，0 关闭 | 服务器端本插件逻辑 | `l4d_tongue_bend_fix.sp:42` |

## [L4D & 2] Tongue Block Fix —— `fixes/l4d_tongue_block_fix.sp`

- 源文件：`fixes/l4d_tongue_block_fix.sp`
- myinfo：author=Forgetest
- myinfo description（源码原文）：Fix infected teammate blocking tongue chasing.

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `tongue_tip_through_teammate` | `0` | 整数，源码写作浮点 | 0.0 ~ 3.0 | Smoker 能否透过队友吐舌：1 透过普通特感，2 透过 Tank，3 全部，0 关闭 | 服务器端本插件逻辑；仅服务器 | `l4d_tongue_block_fix.sp:101` |
| `tongue_fly_through_teammate` | `1` | 整数，源码写作浮点 | 0.0 ~ 3.0 | 舌头射出后能否穿过队友：1 普通特感，2 Tank，3 全部，0 关闭 | 服务器端本插件逻辑；仅服务器 | `l4d_tongue_block_fix.sp:110` |

## [L4D & 2] Tongue Float Fix —— `fixes/l4d_tongue_float_fix.sp`

- 源文件：`fixes/l4d_tongue_float_fix.sp`
- myinfo：author=Forgetest
- myinfo description（源码原文）：Fix tongue instant choking survivors.

### 指令

（本插件未注册指令）

### ConVar

（本插件未注册 ConVar）

## l4d_unrestrict_panic_battlefield.sp（myinfo 缺 name，用文件名代替） —— `fixes/l4d_unrestrict_panic_battlefield.sp`

- 源文件：`fixes/l4d_unrestrict_panic_battlefield.sp`
- myinfo：源码中未找到 myinfo 块

### 指令

（本插件未注册指令）

### ConVar

（本插件未注册 ConVar）

## [L4D & L4D2] Vote Poll Fix —— `fixes/l4d_votepoll_fix.sp`

- 源文件：`fixes/l4d_votepoll_fix.sp`
- myinfo：author=raziEiL [disawar1]
- myinfo description（源码原文）：Changes number of players eligible to vote

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `l4d_votepoll_fix_version` | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | 服务器端本插件逻辑；复制到客户端；不写入 cfg 存档 | `l4d_votepoll_fix.sp:30` |

## SG-552 Tickrate Fix —— `fixes/lfd_both_fixSG552.sp`

- 源文件：`fixes/lfd_both_fixSG552.sp`
- myinfo：version=2，author=bullet28
- myinfo description（源码原文）：Tries to fix strange FOV behavior when using SG-552 with increased tickrate

### 指令

（本插件未注册指令）

### ConVar

（本插件未注册 ConVar）

## Smoker Animation Fix (windowed & safe reset, end-on-grab option) —— `fixes/smoker_anim_fix.sp`

- 源文件：`fixes/smoker_anim_fix.sp`
- myinfo：version=1.3，author=HoongDou, edited by ChatGPT for morzlee
- myinfo description（源码原文）：Fix AI Smoker tongue animation to match human players (with window, robust resets, and end-on-grab option)

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `smoker_anim_fix_enabled` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用 Smoker 动画修复 | 服务器端本插件逻辑 | `smoker_anim_fix.sp:83` |
| `smoker_anim_fix_hold_window` | `0.6` | 浮点 | 0.0 ~ 2.0 | 技能开始后强制舌头动画的秒数，范围 0~2 | 服务器端本插件逻辑 | `smoker_anim_fix.sp:86` |
| `smoker_anim_fix_safety_timer` | `2.0` | 浮点 | 0.5 ~ 5.0 | 技能开始后重置舌头 flag 的安全计时，范围 0.5~5 | 服务器端本插件逻辑 | `smoker_anim_fix.sp:89` |
| `smoker_anim_fix_end_on_grab` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 舌头抓住目标时是否立即结束保持窗口 | 服务器端本插件逻辑 | `smoker_anim_fix.sp:92` |

## sv_consistency fixes —— `fixes/sv_consistency_fix.sp`

- 源文件：`fixes/sv_consistency_fix.sp`
- myinfo：version=1.4.3，author=step, Sir, A1m`
- myinfo description（源码原文）：Fixes multiple sv_consistency issues.

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_consistencycheck` | [目标] | 需要 ADMFLAG_RCON（i，RCON 命令） | 对所有玩家或指定目标执行一致性检查 | `sv_consistency_fix.sp:53` |

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `svctyfix_message_enable` | `1.0` | 开关 0 或 1 | 0.0 ~ 1.0 | 玩家加入时是否在控制台打印提示信息 | 服务器端本插件逻辑 | `sv_consistency_fix.sp:28` |
| `svctyfix_welcome_message` | `a SoundM Protected Server` | 字符串或表达式 | 无上下界 | 在控制台向玩家显示的消息 | 服务器端本插件逻辑 | `sv_consistency_fix.sp:35` |
| `cl_consistencycheck_interval` | `180.0` | 整数，源码写作浮点 | 无上下界 | 距上次一致性检查多少秒后再次执行检查 | 服务器端本插件逻辑；复制到客户端 | `sv_consistency_fix.sp:41` |

## [L4D2] Weapon Duplicate Fix —— `fixes/weapon_spawn_duplicate_fix.sp`

- 源文件：`fixes/weapon_spawn_duplicate_fix.sp`
- myinfo：version=1.1.1，author=shqke
- myinfo description（源码原文）：Prevents a weapon to be taken from weapon spawn if its item counter has hit a zero

### 指令

（本插件未注册指令）

### ConVar

（本插件未注册 ConVar）

## [L4D2] Allow CS weapons on every map. —— `l4d2_allow_cs_weapons.sp`

- 源文件：`l4d2_allow_cs_weapons.sp`
- myinfo：version=1.0，author=Sir
- myinfo description（源码原文）：Force weapon spawners to ignore the 'no_cs_weapons' KeyValue, allowing for CS weapons on every map.

### 指令

（本插件未注册指令）

### ConVar

（本插件未注册 ConVar）

## [L4D/2] Incap Fire Fix —— `l4d2_incap_fire_fix.sp`

- 源文件：`l4d2_incap_fire_fix.sp`
- myinfo：version=1.1.0，author=Sir
- myinfo description（源码原文）：Lets incapacitated survivors fire their weapon normally (with sound) while holding shove

### 指令

（本插件未注册指令）

### ConVar

（本插件未注册 ConVar）

## L4D2 Lobby match manager —— `l4d2_lobby_match_manager.sp`

- 源文件：`l4d2_lobby_match_manager.sp`
- myinfo：author=fdxx

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_lobby_status` | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 显示大厅预留状态与槽位信息 | `l4d2_lobby_match_manager.sp:117` |
| `sm_lobby_set` | <cookie> <bAllowLobbyConnectOnly> <bHostingLobby> <bUpdateGameType> | 需要 ADMFLAG_ROOT（z，最高权限） | 直接设置大厅预留 Cookie 的各字段 | `l4d2_lobby_match_manager.sp:118` |
| `sm_lobby_unreserve` | 无参数 | 服务器控制台命令，玩家无法使用 | 移除大厅预留使更多玩家可以加入，服务器控制台命令 | `l4d2_lobby_match_manager.sp:119` |

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `l4d2_lobby_match_manager_version` | `VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（默认值取自 VERSION 宏） | 服务器端本插件逻辑；不写入 cfg 存档 | `l4d2_lobby_match_manager.sp:103` |
| `l4d2_lmm_unreserve_type` | `0` | 整数 | 无上下界 | 直接加入不创建预留：0 保留原有预留，1 Anne 模式下保留原预留至其他情况，其余描述被截断 | 服务器端本插件逻辑 | `l4d2_lobby_match_manager.sp:104` |
| `l4d2_lmm_reservation_modify_flags` | `7` | 整数 | 无上下界 | 修改客户端提交给服务器的 lobby 设置，见 RMFLAG_*，需 unreserve_type 不等于 1 | 服务器端本插件逻辑 | `l4d2_lobby_match_manager.sp:105` |

## l4d2_map_vote.sp（myinfo 缺 name，用文件名代替） —— `l4d2_map_vote.sp`

- 源文件：`l4d2_map_vote.sp`
- myinfo：源码中未找到 myinfo 块

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_v3` | 无参数 | 无（任意玩家可用） | 打开换图投票菜单，旁观者不可投票 | `l4d2_map_vote.sp:135` |
| `sm_maps` | 无参数 | 无（任意玩家可用） | 打开地图列表菜单，sm_v3 的别名 | `l4d2_map_vote.sp:136` |
| `sm_chmap` | 无参数 | 无（任意玩家可用） | 打开换图菜单，sm_v3 的别名 | `l4d2_map_vote.sp:137` |
| `sm_mapvote` | 无参数 | 无（任意玩家可用） | 打开换图投票菜单，sm_v3 的别名 | `l4d2_map_vote.sp:138` |
| `sm_votemap` | 无参数 | 无（任意玩家可用） | 打开换图投票菜单，sm_v3 的别名 | `l4d2_map_vote.sp:139` |
| `sm_mapnext` | 无参数 | 无（任意玩家可用） | 对下一张地图发起投票，终局地图需前置插件支持 | `l4d2_map_vote.sp:142` |
| `sm_votenext` | 无参数 | 无（任意玩家可用） | 对下一张地图发起投票，sm_mapnext 的别名 | `l4d2_map_vote.sp:143` |
| `sm_reload_vpk` | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 刷新 VPK 与战役列表 | `l4d2_map_vote.sp:145` |
| `sm_update_vpk` | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 刷新 VPK 与战役列表，sm_reload_vpk 的别名 | `l4d2_map_vote.sp:146` |
| `sm_missions_export` | <文件名> | 需要 ADMFLAG_ROOT（z，最高权限） | 把任务 KV 导出为文件 | `l4d2_map_vote.sp:148` |

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `l4d2_map_vote_version` | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | 服务器端本插件逻辑；不写入 cfg 存档 | `l4d2_map_vote.sp:109` |
| `notify_map_next` | `1` | 整数 | 无上下界 | 终局开始后提示投票下一张地图的方式：0 不提示，1 聊天栏，2 屏幕中央，4 弹出菜单，可相加 | 服务器端本插件逻辑 | `l4d2_map_vote.sp:110` |
| `l4d2_mapvote_autoreload` | `2` | 整数 | 无上下界 | 输入 !mapvote 时是否自动刷新 VPK 与战役列表：0 关，1 仅管理员触发，2 所有人触发 | 服务器端本插件逻辑 | `l4d2_map_vote.sp:111` |
| `l4d2_mapvote_reload_cd` | `10.0` | 整数，源码写作浮点 | 无上下界 | 自动刷新的冷却秒数，避免被频繁触发导致卡顿 | 服务器端本插件逻辑 | `l4d2_map_vote.sp:116` |
| `l4d2_mapvote_versus_from_coop` | `1` | 整数 | 无上下界 | 为缺少 versus 的战役临时注入 modes/versus：0 关，1 仅 Anne 派生 cfg，2 所有 cfg | 服务器端本插件逻辑 | `l4d2_map_vote.sp:120` |

## L4D2 Native vote —— `l4d2_nativevote.sp`

- 源文件：`l4d2_nativevote.sp`
- myinfo：author=Powerlord, fdxx
- myinfo description（源码原文）：Voting API to use the game's native vote panels

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `l4d2_nativevote_version` | `VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（默认值取自 VERSION 宏） | 服务器端本插件逻辑 | `l4d2_nativevote.sp:74` |
| `l4d2_nativevote_initiator_auto_voteyes` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 为 1 时发起者自动投赞成票 | 服务器端本插件逻辑 | `l4d2_nativevote.sp:75` |

## L4D2 Nav Variant Loader —— `l4d2_nav_variant.sp`

- 源文件：`l4d2_nav_variant.sp`
- myinfo：author=morzlee, OpenAI
- myinfo description（源码原文）：Redirects selected .nav loads to configured variants.
- HookConVarChange：`g_cvEnable`→`OnNavCvarChanged`、`g_cvVariant`→`OnNavCvarChanged`、`g_cvConfig`→`OnConfigCvarChanged`、`g_cvRequiredCfg`→`OnNavCvarChanged`、`g_cvRequiredStripperPath`→`OnNavCvarChanged`、`g_cvReadyCfgName`→`OnReadyCfgNameChanged`、`g_cvStripperCfgPath`→`OnStripperCfgPathChanged`

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_nav_variant_reload` | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 重新加载导航变体配置 | `l4d2_nav_variant.sp:90` |
| `sm_nav_variant_clearcache` | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 清除已缓存的 nav 文件数据，供下次加载使用 | `l4d2_nav_variant.sp:91` |
| `sm_nav_variant_status` | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 显示导航变体状态与最近一次重定向读取结果 | `l4d2_nav_variant.sp:92` |

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `l4d2_nav_variant_enable` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用导航网格变体重定向 | 服务器端本插件逻辑 | `l4d2_nav_variant.sp:76` |
| `l4d2_nav_variant_name` | `DEFAULT_VARIANT` | 字符串或表达式 | 无上下界 | 当前 stripper 路径符合条件时启用的导航变体名，留空使用地图默认 nav | 服务器端本插件逻辑 | `l4d2_nav_variant.sp:77` |
| `l4d2_nav_variant_config` | `DEFAULT_CONFIG` | 字符串或表达式 | 无上下界 | 导航变体 KeyValues 配置位于 addons/sourcemod 下的路径 | 服务器端本插件逻辑 | `l4d2_nav_variant.sp:78` |
| `l4d2_nav_variant_required_cfg` | `` | 字符串或表达式 | 无上下界 | 可选的 l4d_ready_cfg_name 子串守卫，留空则仅由 nav_variant_name 控制重定向 | 服务器端本插件逻辑 | `l4d2_nav_variant.sp:79` |
| `l4d2_nav_variant_stripper_path` | `DEFAULT_STRIPPER_PATH` | 字符串或表达式 | 无上下界 | 要求 stripper_cfg_path 等于该路径，留空关闭该守卫 | 服务器端本插件逻辑 | `l4d2_nav_variant.sp:80` |
| `l4d2_nav_variant_debug` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否把导航变体决策打印到服务器日志 | 服务器端本插件逻辑 | `l4d2_nav_variant.sp:81` |
| `l4d2_nav_variant_version` | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | 服务器端本插件逻辑；不写入 cfg 存档 | `l4d2_nav_variant.sp:82` |

## L4D2 Smoker Drag Damage Interval —— `l4d2_smoker_drag_damage_interval_zone.sp`

- 源文件：`l4d2_smoker_drag_damage_interval_zone.sp`
- myinfo：version=1.0，author=Visor, Sir, A1m`
- myinfo description（源码原文）：Implements a native-like cvar that should've been there out of the box

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `tongue_drag_damage_interval` | `value` | 字符串或表达式 | 无上下界 | 拖拽造成伤害的频率 | 服务器端本插件逻辑 | `l4d2_smoker_drag_damage_interval_zone.sp:45` |
| `tongue_drag_first_damage_interval` | `-1.0` | 整数，源码写作浮点 | 无上下界 | 首次伤害在多少秒后施加，0.0 表示关闭 | 服务器端本插件逻辑 | `l4d2_smoker_drag_damage_interval_zone.sp:46` |
| `tongue_drag_first_damage` | `3.0` | 整数，源码写作浮点 | 无上下界 | 舌头首次命中时施加的伤害，仅在启用 first_damage_interval 时生效 | 服务器端本插件逻辑 | `l4d2_smoker_drag_damage_interval_zone.sp:47` |
| `alonemode` | `0` | 整数 | 无上下界 | 是否处于 alonemode | 服务器端本插件逻辑 | `l4d2_smoker_drag_damage_interval_zone.sp:48` |

## L4D2 Source KeyValues —— `l4d2_source_keyvalues.sp`

- 源文件：`l4d2_source_keyvalues.sp`
- myinfo：author=fdxx
- myinfo description（源码原文）：Call the game's own KeyValues function

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `l4d2_source_keyvalues_version` | `VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（默认值取自 VERSION 宏） | 服务器端本插件逻辑；不写入 cfg 存档 | `l4d2_source_keyvalues.sp:78` |

## L4D2 Vote Return Lobby patch —— `l4d2_vote_returnlobby_patch.sp`

- 源文件：`l4d2_vote_returnlobby_patch.sp`
- myinfo：author=fdxx
- myinfo description（源码原文）：Create cvar sv_vote_returnlobby_allowed.

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `l4d2_vote_returnlobby_patch_version` | `VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（默认值取自 VERSION 宏） | 服务器端本插件逻辑；不写入 cfg 存档 | `l4d2_vote_returnlobby_patch.sp:22` |
| `sv_vote_returnlobby_allowed` | `0` | 整数 | 无上下界 | 玩家能否投票返回大厅 | 服务器端本插件逻辑 | `l4d2_vote_returnlobby_patch.sp:23` |

## Block Pause/Unpause Spam in Console —— `l4d_pause_message.sp`

- 源文件：`l4d_pause_message.sp`
- myinfo：version=1.0，author=Sir (Simplified version of Silver's)
- myinfo description（源码原文）：Simply block pause commands when the server doesn't even support pausing.

### 指令

（本插件未注册指令）

### ConVar

（本插件未注册 ConVar）

## [L4D & L4D2] Left 4 DHooks Direct —— `left4dhooks.sp`

- 源文件：`left4dhooks.sp`
- myinfo：author=SilverShot
- myinfo description（源码原文）：Left 4 Downtown and L4D Direct conversion and merger.

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_l4dd_unreserve` | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 移除大厅预留 | `left4dhooks.sp:723` |
| `sm_l4dd_reload` | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 重新加载 detour 钩子，按需启用或禁用 | `left4dhooks.sp:724` |
| `sm_l4dd_detours` | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 列出当前生效的前向及其使用插件 | `left4dhooks.sp:725` |
| `sm_l4dhooks_reload` | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 重新加载 detour 钩子，sm_l4dd_reload 的别名 | `left4dhooks.sp:726` |
| `sm_l4dhooks_detours` | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 列出当前生效的前向及其使用插件，sm_l4dd_detours 的别名 | `left4dhooks.sp:727` |

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `left4dhooks_version` | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | 服务器端本插件逻辑；不写入 cfg 存档 | `left4dhooks.sp:734` |
| `l4d2_vscript_return` | `` | 字符串或表达式 | 无上下界 | 用于返回 VScript 值的缓冲区，源码注明请勿使用 | 服务器端本插件逻辑；不写入 cfg 存档 | `left4dhooks.sp:739` |
| `l4d2_addons_eclipse` | `-1` | 整数 | 无上下界 | Addons 管理器：-1 使用 addonconfig，0 禁用 addons，1 启用 addons | 服务器端本插件逻辑 | `left4dhooks.sp:740` |

## [L4D & L4D2] Left 4 DHooks Direct - TESTER —— `left4dhooks_test.sp`

- 源文件：`left4dhooks_test.sp`
- myinfo：author=SilverShot
- myinfo description（源码原文）：Left 4 DHooks Direct - Demo and Test plugin.

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_l4df` | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 输出已触发前向的数量统计，来自测试插件 | `left4dhooks_test.sp:111` |
| `sm_l4dd` | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 输出区域与距离等调试信息，来自测试插件 | `left4dhooks_test.sp:112` |
| `sm_l4dd_calls` | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 把前向允许记录的调用次数加 1，来自测试插件 | `left4dhooks_test.sp:113` |

### ConVar

（本插件未注册 ConVar）

## L4D2 Auto restart —— `linux_auto_restart.sp`

- 源文件：`linux_auto_restart.sp`
- myinfo：author=Dragokas, Harry Potter, fdxx
- myinfo description（源码原文）：Auto restart server when the last player disconnects from the server. Only support Linux system

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_restart` | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 手动重启服务器 | `linux_auto_restart.sp:40` |
| `sm_crash` | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 手动重启服务器，与 sm_restart 等价 | `linux_auto_restart.sp:41` |

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `l4d2_auto_restart_version` | `VERSION` | 字符串或表达式 | 无上下界 | 插件版本 | 服务器端本插件逻辑；不写入 cfg 存档 | `linux_auto_restart.sp:34` |
| `l4d2_auto_restart_delay` | `30.0` | 整数，源码写作浮点 | 无上下界 | 重启宽限时间，单位秒 | 服务器端本插件逻辑 | `linux_auto_restart.sp:35` |

## map_changer.sp（myinfo 缺 name，用文件名代替） —— `map_changer.sp`

- 源文件：`map_changer.sp`
- myinfo：源码中未找到 myinfo 块

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_setnext` | <章节地图代码> | 需要 ADMFLAG_RCON（i，RCON 命令） | 设置下一张地图 | `map_changer.sp:159` |

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `map_changer_version` | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | 服务器端本插件逻辑；不写入 cfg 存档 | `map_changer.sp:137` |
| `mapchanger_finale_change_type` | `8` | 整数 | 无上下界 | 终局换图时机：0 不换图返回大厅，1 救援载具离开时，2 终局获胜时，4 统计屏幕出现时，8 统计屏幕结束时，可相加 | 服务器端本插件逻辑 | `map_changer.sp:139` |
| `mapchanger_finale_failure_count` | `3` | 整数 | 无上下界 | 终局团灭几次后自动换到下一张图 | 服务器端本插件逻辑 | `map_changer.sp:140` |
| `mapchanger_finale_random_nextmap` | `1` | 整数 | 无上下界 | 终局是否启用随机下一关地图 | 服务器端本插件逻辑 | `map_changer.sp:141` |
| `mapchanger_finale_failure_vote` | `1` | 整数 | 无上下界 | 救援关第一回合是否投票决定启用终局团灭自动换图 | 服务器端本插件逻辑 | `map_changer.sp:142` |
| `mapchanger_finale_failure_vote_default` | `1` | 整数 | 无上下界 | 投票关闭或无法发起时是否默认启用终局团灭自动换图 | 服务器端本插件逻辑 | `map_changer.sp:143` |
| `mapchanger_finale_failure_vote_delay` | `30.0` | 整数，源码写作浮点 | 1.0 ~ 120.0 | 救援关第一回合开始后延迟多少秒发起该投票，范围 1~120 | 服务器端本插件逻辑 | `map_changer.sp:144` |

## Match Vote —— `match_vote.sp`

- 源文件：`match_vote.sp`
- myinfo：version=1.5.1，author=vintik, Sir, StarterX4
- myinfo description（源码原文）：!match !rmatch !chmatch - Change Hostname and Slots while you're at it!

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_match` | <cfg 名> | 无（任意玩家可用） | 发起比赛模式切换请求 | `match_vote.sp:70` |
| `sm_chmatch` | <cfg 名> | 无（任意玩家可用） | 发起比赛模式变更请求 | `match_vote.sp:71` |
| `sm_rmatch` | 无参数 | 无（任意玩家可用） | 发起重置比赛模式的投票 | `match_vote.sp:72` |

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `sm_match_vote_enabled` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用插件 | 服务器端本插件逻辑 | `match_vote.sp:66` |
| `mv_maxplayers` | `30` | 整数，源码写作浮点 | 1.0 ~ 32.0 | 配置加载或卸载时服务器应有的槽位数，范围 1~32 | 服务器端本插件逻辑 | `match_vote.sp:67` |
| `sm_match_player_limit` | `1` | 整数，源码写作浮点 | 1.0 ~ 32.0 | 发起投票所需的最少在场玩家数，范围 1~32 | 服务器端本插件逻辑 | `match_vote.sp:68` |

## 1v1 EQ —— `optional/1v1.sp`

- 源文件：`optional/1v1.sp`
- myinfo：version=0.2.4，author=Blade + Confogl Team, Tabun, Visor
- myinfo description（源码原文）：A plugin designed to support 1v1.

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `sm_1v1_dmgthreshold` | `24` | 整数，源码写作浮点 | ≥ 1.0 | 特感自杀前一次性受到的伤害量，最小 1 | 服务器端本插件逻辑 | `1v1.sp:45` |

## 1v1 SkeetStats —— `optional/1v1_skeetstats.sp`

- 源文件：`optional/1v1_skeetstats.sp`
- myinfo：version=0.1h，author=Tabun
- myinfo description（源码原文）：Shows 1v1-relevant info at end of round.
- HookConVarChange：`hCountTankDamage`→`ConVarChange_CountTankDamage`、`hCountWitchDamage`→`ConVarChange_CountWitchDamage`、`hBrevityFlags`→`ConVarChange_BrevityFlags`、`hPounceDmgInt`→`ConVarChange_PounceDmgInt`、`hRUPActive`→`ConVarChange_RUPActive`

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_skeets` | 无参数 | 无（任意玩家可用） | 输出当前 skeetstats | `1v1_skeetstats.sp:269` |
| `say` | 聊天文本 | 无（任意玩家可用） | 接管 say，识别 !skeets 并输出 skeetstats | `1v1_skeetstats.sp:271` |
| `say_team` | 聊天文本 | 无（任意玩家可用） | 接管 say_team，识别 !skeets 并输出 skeetstats | `1v1_skeetstats.sp:272` |

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `sm_skeetstat_counttank` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 为 1 时对 Tank 的伤害计入统计 | 服务器端本插件逻辑 | `1v1_skeetstats.sp:238` |
| `sm_skeetstat_countwitch` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 为 1 时对 Witch 的伤害计入统计 | 服务器端本插件逻辑 | `1v1_skeetstats.sp:239` |
| `sm_skeetstat_brevity` | `32` | 整数，源码写作浮点 | ≥ 0.0 | 报告精简位标志：1 隐藏特感，2 隐藏普感，4 隐藏命中率，8 隐藏击杀与死停，32 近战命中率，64 伤害统计，可相加 | 服务器端本插件逻辑 | `1v1_skeetstats.sp:240` |

## 8Ball —— `optional/8ball.sp`

- 源文件：`optional/8ball.sp`
- myinfo：version=1.3.1，author=spoon
- myinfo description（源码原文）：Simple 8Ball Game Plugin. Works the same as Coinflip / Dice Roll.

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_8ball` | <问题文本> | 无（任意玩家可用） | 8ball 随机回答一个问题 | `8ball.sp:19` |

### ConVar

（本插件未注册 ConVar）

## 1vai function(quick standup, give damage and release survivor) —— `optional/AnneHappy/1vai.sp`

- 源文件：`optional/AnneHappy/1vai.sp`
- myinfo：version=1.2，author=东
- myinfo description（源码原文）：A plugin designed to support 1vAI.

### 指令

（本插件未注册指令）

### ConVar

（本插件未注册 ConVar）

## Advance Special Infected AI —— `optional/AnneHappy/AI_HardSI_2.sp`

- 源文件：`optional/AnneHappy/AI_HardSI_2.sp`
- myinfo：version=2022.12.16，author=def075, Caibiii, 夜羽真白，东
- myinfo description（源码原文）：Advanced Special Infected AI

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `ai_TankSequencePlayBackRate` | `3.0` | 整数，源码写作浮点 | ≥ 0.0 | 坦克攀爬动画加速速率 | 服务器端本插件逻辑 | `AI_HardSI_2.sp:80` |

## Advance Special Infected AI —— `optional/AnneHappy/AI_HardSI_new.sp`

- 源文件：`optional/AnneHappy/AI_HardSI_new.sp`
- myinfo：version=2022.05.02，author=def075, Caibiii, 夜羽真白，东
- myinfo description（源码原文）：Advanced Special Infected AI

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `ai_TankSequencePlayBackRate` | `3.0` | 整数，源码写作浮点 | ≥ 0.0 | 坦克攀爬动画加速速率（同名 ConVar，来自 Ai Tank 增强变体） | 服务器端本插件逻辑 | `AI_HardSI_new.sp:81` |

## SI target limit —— `optional/AnneHappy/SI_Target_limit.sp`

- 源文件：`optional/AnneHappy/SI_Target_limit.sp`
- myinfo：author=东
- myinfo description（源码原文）：限制单个玩家被特感选为目标的最大数量

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_targetlimit_status` | 无参数 | 需要 ADMFLAG_GENERIC（b，通用管理员） | 显示每名生还者的特感目标数量与上限 | `SI_Target_limit.sp:162` |

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `SI_target_enable` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否开启插件 | 服务器端本插件逻辑 | `SI_Target_limit.sp:132` |
| `SI_enable_option` | `53` | 整数，源码写作浮点 | 0.0 ~ 127.0 | 控制类特感掩码：1 舌头，2 Boomer，4 猎人，8 Spitter，16 猴子，32 牛，64 Tank；默认 53 为舌头加猎人加猴子加牛共用控制上限 | 服务器端本插件逻辑 | `SI_Target_limit.sp:133` |
| `SI_target_limit_auto` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 自动上限：已启用控制类职业预算除以可动生还者再加 1，预算为对应 z_* 与 inf_* 上限之和并钳在 l4d_infected_limit 内 | 服务器端本插件逻辑 | `SI_Target_limit.sp:134` |
| `SI_target_limit_manual` | `3` | 整数 | 无上下界 | 服务器不自动限制时手动设置的最大目标数 | 服务器端本插件逻辑 | `SI_Target_limit.sp:135` |
| `SI_target_rushman_scope` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | infected_control 检测到跑男时放开上限的范围：0 全体生还者抬到特感总上限，1 仅跑男本人放开 | 服务器端本插件逻辑 | `SI_Target_limit.sp:136` |

## ShieldTips.sp（myinfo 缺 name，用文件名代替） —— `optional/AnneHappy/ShieldTips.sp`

- 源文件：`optional/AnneHappy/ShieldTips.sp`
- myinfo：源码中未找到 myinfo 块

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `sms_game_idle_notify_block` | `1` | 整数 | 无上下界 | 是否屏蔽游戏自带的玩家闲置提示 | 服务器端本插件逻辑 | `ShieldTips.sp:35` |
| `sms_cvar_change_notify_block` | `1` | 整数 | 无上下界 | 是否屏蔽游戏自带的 ConVar 更改提示 | 服务器端本插件逻辑 | `ShieldTips.sp:36` |
| `sms_sourcemod_sm_notify_admin` | `0` | 整数 | 无上下界 | 是否屏蔽 SourceMod 自带的 SM 提示：1 只向管理员显示，0 对所有人屏蔽 | 服务器端本插件逻辑 | `ShieldTips.sp:37` |
| `sms_game_disconnect_notify_block` | `1` | 整数 | 无上下界 | 是否屏蔽游戏自带的玩家离开提示 | 服务器端本插件逻辑 | `ShieldTips.sp:38` |

## Ai Boomer 2.0 —— `optional/AnneHappy/ai_boomer_2.sp`

- 源文件：`optional/AnneHappy/ai_boomer_2.sp`
- myinfo：version=2023/1/17+rev2026-07-15.2，author=夜羽真白
- myinfo description（源码原文）：Ai Boomer 增强 2.0 版本 (integrated tweaks by ChatGPT)

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `ai_BoomerBhop` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否开启 Boomer 连跳 | 服务器端本插件逻辑 | `ai_boomer_2.sp:69` |
| `ai_BoomerBhopSpeed` | `150.0` | 整数，源码写作浮点 | ≥ 0.0 | Boomer 连跳速度 | 服务器端本插件逻辑 | `ai_boomer_2.sp:70` |
| `ai_BoomerJumpVomit` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否允许 Boomer 已在空中时主动喷吐，不额外强制起跳 | 服务器端本插件逻辑 | `ai_boomer_2.sp:71` |
| `ai_BoomerUpVision` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | Boomer 喷吐时是否上抬视角 | 服务器端本插件逻辑 | `ai_boomer_2.sp:72` |
| `ai_BoomerTurnVision` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | Boomer 喷吐时是否旋转视角 | 服务器端本插件逻辑 | `ai_boomer_2.sp:73` |
| `ai_BoomerForceBile` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否开启生还者进入 Boomer 喷吐范围后强制被喷 | 服务器端本插件逻辑 | `ai_boomer_2.sp:76` |
| `ai_BoomerBileFindRange` | `300` | 整数，源码写作浮点 | ≥ 0.0 | 该距离内有被控或倒地的生还者时 Boomer 优先攻击，0 禁用 | 服务器端本插件逻辑 | `ai_boomer_2.sp:78` |
| `ai_BoomerTurnInterval` | `15` | 整数，源码写作浮点 | ≥ 0.0 | Boomer 喷吐旋转视角时每隔多少帧切换目标 | 服务器端本插件逻辑 | `ai_boomer_2.sp:79` |
| `ai_BoomerDegreeForceBile` | `10` | 整数，源码写作浮点 | ≥ 0.0 | 目标与 Boomer 视角夹角在该值内且能看到目标头部时强制喷吐，0 禁用 | 服务器端本插件逻辑 | `ai_boomer_2.sp:81` |
| `ai_BoomerAutoFrame` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否按目标角度自动计算视野在下一目标的帧数 | 服务器端本插件逻辑 | `ai_boomer_2.sp:82` |

## Ai Boomer 3.0 —— `optional/AnneHappy/ai_boomer_3.sp`

- 源文件：`optional/AnneHappy/ai_boomer_3.sp`
- myinfo：version=3.0.10，author=夜羽真白
- myinfo description（源码原文）：Ai Boomer 增强 3.0（anne_nextbot Path Follow）

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `ai_BoomerBhop` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否开启 Boomer 连跳 | 服务器端本插件逻辑 | `ai_boomer_3.sp:99` |
| `ai_BoomerBhopSpeed` | `150.0` | 整数，源码写作浮点 | ≥ 0.0 | Boomer 连跳推力，落地后立即再起跳时追加，第一跳按 first_hop_ratio 折算 | 服务器端本插件逻辑 | `ai_boomer_3.sp:100` |
| `ai_BoomerBhopFirstHopRatio` | `0.8` | 浮点 | 0.0 ~ 1.0 | 从跑动直接起跳的第一跳可获得的推力比例，0.0 不加推力，1.0 与落地跳相同 | 服务器端本插件逻辑 | `ai_boomer_3.sp:101` |
| `ai_BoomerBhopMaxSpeed` | `1000.0` | 整数，源码写作浮点 | ≥ 0.0 | Boomer 连跳的最大水平速度，0 不限制 | 服务器端本插件逻辑 | `ai_boomer_3.sp:103` |
| `ai_BoomerBhopStartDistance` | `2500.0` | 整数，源码写作浮点 | ≥ 0.0 | Boomer 距最近生还者多远时开始连跳 | 服务器端本插件逻辑 | `ai_boomer_3.sp:104` |
| `ai_BoomerJumpVomit` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否允许 Boomer 已在空中时主动喷吐 | 服务器端本插件逻辑 | `ai_boomer_3.sp:105` |
| `ai_boomer3_path_bhop` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否优先使用 anne_nextbot 路径前视连跳 | 服务器端本插件逻辑 | `ai_boomer_3.sp:106` |
| `ai_boomer3_path_lookahead_depth` | `5` | 整数，源码写作浮点 | 1.0 ~ 16.0 | 路径连跳最大前视节点数，范围 1~16 | 服务器端本插件逻辑 | `ai_boomer_3.sp:107` |
| `ai_boomer3_path_lane_offset` | `10.0` | 整数，源码写作浮点 | 0.0 ~ 40.0 | 路径连跳的最大稳定侧向分流距离，范围 0~40 | 服务器端本插件逻辑 | `ai_boomer_3.sp:108` |
| `ai_BoomerUpVision` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | Boomer 喷吐时是否上抬视角 | 服务器端本插件逻辑 | `ai_boomer_3.sp:109` |
| `ai_BoomerTurnVision` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | Boomer 喷吐时是否旋转视角 | 服务器端本插件逻辑 | `ai_boomer_3.sp:110` |
| `ai_BoomerForceBile` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否开启生还者进入 Boomer 喷吐范围后强制被喷 | 服务器端本插件逻辑 | `ai_boomer_3.sp:113` |
| `ai_BoomerBileFindRange` | `300` | 整数，源码写作浮点 | ≥ 0.0 | 该距离内有被控或倒地的生还者时 Boomer 优先攻击，0 禁用 | 服务器端本插件逻辑 | `ai_boomer_3.sp:115` |
| `ai_BoomerTurnInterval` | `15` | 整数，源码写作浮点 | ≥ 0.0 | Boomer 喷吐旋转视角时每隔多少帧切换目标 | 服务器端本插件逻辑 | `ai_boomer_3.sp:116` |
| `ai_BoomerDegreeForceBile` | `10` | 整数，源码写作浮点 | ≥ 0.0 | 目标与 Boomer 视角夹角在该值内且能看到目标头部时强制喷吐，0 禁用 | 服务器端本插件逻辑 | `ai_boomer_3.sp:118` |
| `ai_BoomerAutoFrame` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否按目标角度自动计算视野在下一目标的帧数 | 服务器端本插件逻辑 | `ai_boomer_3.sp:119` |

## Ai-Boomer增强 —— `optional/AnneHappy/ai_boomer_new.sp`

- 源文件：`optional/AnneHappy/ai_boomer_new.sp`
- myinfo：version=1.0.1.0，author=夜羽真白，东
- myinfo description（源码原文）：觉得Ai-Boomer不够强， Try this！

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `ai_BoomerBhop` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否开启 Boomer 连跳 | 服务器端本插件逻辑 | `ai_boomer_new.sp:41` |
| `ai_BoomerBhopSpeed` | `150.0` | 整数，源码写作浮点 | ≥ 0.0 | Boomer 连跳的速度 | 服务器端本插件逻辑 | `ai_boomer_new.sp:42` |
| `ai_BoomerAirAngles` | `60.0` | 整数，源码写作浮点 | ≥ 0.0 | 空中速度向量与到生还者方向向量夹角大于该值时停止连跳 | 服务器端本插件逻辑 | `ai_boomer_new.sp:43` |

## Ai-Charger 3.0 —— `optional/AnneHappy/ai_charger3/ai_charger3.sp`

- 源文件：`optional/AnneHappy/ai_charger3/ai_charger3.sp`
- myinfo：version=1.0.1.18，author=夜羽真白
- myinfo description（源码原文）：Ai Charger 增强 3.0 版本

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `ai_charger3_bhop` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否允许 Charger 在接近状态时连跳：0 禁止，1 允许 | 服务器端本插件逻辑 | `ai_charger3.sp:51` |
| `ai_charger3_bhop_min_dist` | `100.0` | 整数，源码写作浮点 | ≥ 0.0 | 禁止连跳的最小距离，小于该距离转为博弈状态 | 服务器端本插件逻辑 | `ai_charger3.sp:53` |
| `ai_charger3_bhop_max_dist` | `9999.0` | 整数，源码写作浮点 | ≥ 0.0 | 允许连跳的最大距离 | 服务器端本插件逻辑 | `ai_charger3.sp:54` |
| `ai_charger3_bhop_direct_dist` | `400.0` | 整数，源码写作浮点 | ≥ 0.0 | 接近状态中切换到朝目标方向直线连跳的距离阈值 | 服务器端本插件逻辑 | `ai_charger3.sp:55` |
| `ai_charger3_bhop_impulse` | `100.0` | 整数，源码写作浮点 | ≥ 0.0 | 连跳加速度，落地后立即再起跳时追加，第一跳按 first_hop_ratio 折算 | 服务器端本插件逻辑 | `ai_charger3.sp:57` |
| `ai_charger3_bhop_first_hop_ratio` | `0.8` | 浮点 | 0.0 ~ 1.0 | 从跑动直接起跳的第一跳可获得的加速度比例，0.0 只改方向不加速，1.0 与落地跳相同 | 服务器端本插件逻辑 | `ai_charger3.sp:59` |
| `ai_charger3_bhop_min_speed` | `200` | 整数，源码写作浮点 | ≥ 0.0 | 允许连跳的最小速度 | 服务器端本插件逻辑 | `ai_charger3.sp:61` |
| `ai_charger3_bhop_max_speed` | `800` | 整数，源码写作浮点 | ≥ 0.0 | 连跳的最大限制速度 | 服务器端本插件逻辑 | `ai_charger3.sp:62` |
| `ai_charger3_bhop_before_charge` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否允许在冲锋前连跳：0 禁止，1 允许 | 服务器端本插件逻辑 | `ai_charger3.sp:64` |
| `ai_charger3_bhop_no_vision` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否允许 Charger 无目标视野时连跳：0 禁止，1 允许 | 服务器端本插件逻辑 | `ai_charger3.sp:66` |
| `ai_charger3_bhop_nvis_maxang` | `90.0` | 整数，源码写作浮点 | ≥ 0.0 | 无生还者视野时速度向量与视角前向向量夹角在该范围内允许连跳 | 服务器端本插件逻辑 | `ai_charger3.sp:68` |
| `ai_charger3_target_watch_maxdeg` | `22.5` | 浮点 | 0.0 ~ 180.0 | 目标视角与到 Charger 位置向量夹角小于该值时认为目标正在看 Charger，范围 0~180 | 服务器端本插件逻辑 | `ai_charger3.sp:70` |
| `ai_charger3_bhop_strafe_mindist` | `400.0` | 整数，源码写作浮点 | ≥ 0.0 | 与目标距离小于该值时禁止连跳方向左右偏移 | 服务器端本插件逻辑 | `ai_charger3.sp:72` |
| `ai_charger3_bhop_strafe_mindeg` | `30.0` | 整数，源码写作浮点 | ≥ -1.0 | 侧向连跳的最小随机偏移角度，大于 0 启用，-1.0 禁用该功能 | 服务器端本插件逻辑 | `ai_charger3.sp:74` |
| `ai_charger3_bhop_strafe_maxdeg` | `55.0` | 整数，源码写作浮点 | 0.0 ~ 89.0 | 侧向连跳的最大角度，范围 0~89 | 服务器端本插件逻辑 | `ai_charger3.sp:76` |
| `ai_charger3_bhop_strafe_once_dist` | `200.0` | 整数，源码写作浮点 | 无上下界 | 允许侧向连跳一次的最小距离，小于该距离不允许侧向连跳 | 服务器端本插件逻辑 | `ai_charger3.sp:78` |
| `ai_charger3_bhop_strafe_twice_dist` | `400.0` | 整数，源码写作浮点 | 无上下界 | 允许侧向连跳两次的最小距离，大于该距离才允许侧向连跳两次 | 服务器端本插件逻辑 | `ai_charger3.sp:80` |
| `_ai_charger3_melee_bait_minrange` | `15.0` | 整数，源码写作浮点 | ≥ 0.0 | 目标持有近战时近战博弈区的最小范围，等于 melee_range 加该值 | 服务器端本插件逻辑 | `ai_charger3.sp:82` |
| `_ai_charger3_melee_bait_maxrange` | `50.0` | 整数，源码写作浮点 | ≥ 0.0 | 目标持有近战时近战博弈区的最大范围，等于 melee_range 加该值 | 服务器端本插件逻辑 | `ai_charger3.sp:84` |
| `ai_charger3_airvec_modify_min_deg` | `45.0` | 整数，源码写作浮点 | ≥ 0.0 | 空中速度方向与自身到目标方向夹角超过该值时进行速度修正 | 服务器端本插件逻辑 | `ai_charger3.sp:86` |
| `ai_charger3_airvec_modify_max_deg` | `89.0` | 整数，源码写作浮点 | 0.0 ~ 89.0 | 空中转向允许的最大目标偏角，最大 89 度以禁止反向修正 | 服务器端本插件逻辑 | `ai_charger3.sp:88` |
| `ai_charger3_airvec_modify_interval` | `0.3` | 浮点 | ≥ 0.0 | 空中平滑转向的基准时间，实际以 0.05 秒短步长执行 | 服务器端本插件逻辑 | `ai_charger3.sp:90` |
| `_ai_charger3_airvec_modify_lerp` | `0.3` | 浮点 | 0.0 ~ 1.0 | 空中速度方向修正的插值因子，0.1~1.0，越小越平滑但需要更多帧 | 服务器端本插件逻辑 | `ai_charger3.sp:92` |
| `ai_charger3_air_turn_speed_loss` | `0.12` | 浮点 | 0.0 ~ 0.5 | 空中实际转向 90 度时损失的水平速度比例，范围 0~0.5 | 服务器端本插件逻辑 | `ai_charger3.sp:94` |
| `ai_charger3_air_speed_floor_ratio` | `0.50` | 浮点 | 0.0 ~ 1.0 | 空中方向修正使用的起跳保存速度下限比例 | 服务器端本插件逻辑 | `ai_charger3.sp:95` |
| `ai_charger3_air_turn_budget` | `30.0` | 整数，源码写作浮点 | 0.0 ~ 89.0 | 每次离地后空中速度方向最多偏离起跳方向的角度，用完后按惯性落地，0 不限制 | 服务器端本插件逻辑 | `ai_charger3.sp:97` |
| `ai_charger3_bait_max_duration` | `7.0` | 整数，源码写作浮点 | ≥ 0.0 | Charger 进入博弈状态的最大允许时间 | 服务器端本插件逻辑 | `ai_charger3.sp:99` |
| `ai_charger3_melee_bait_orbit` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 近战博弈处于博弈区内时是否沿目标切向绕圈代替原地急停：0 关闭则原地站定 | 服务器端本插件逻辑 | `ai_charger3.sp:100` |
| `ai_charger3_melee_bait_stalemate_switch` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 近战僵持升级到最后一级时是否允许改冲其他生还者 | 服务器端本插件逻辑 | `ai_charger3.sp:101` |
| `ai_charger3_melee_bait_blacklist_dur` | `10.0` | 整数，源码写作浮点 | ≥ 0.0 | 因近战僵持放弃某目标后，该目标进入选目标黑名单的时长 | 服务器端本插件逻辑 | `ai_charger3.sp:102` |
| `ai_charger3_prob_charge_chk_dur` | `0.5` | 浮点 | ≥ 0.0 | Charger 在博弈状态概率冲锋的检测间隔 | 服务器端本插件逻辑 | `ai_charger3.sp:104` |
| `ai_charger3_prob_charge_prob` | `0.8` | 浮点 | ≥ 0.0 | Charger 在博弈状态概率冲锋的概率 | 服务器端本插件逻辑 | `ai_charger3.sp:106` |
| `ai_charger3_anti_retreat` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否禁止 Charger 逃跑：0 禁止，1 允许 | 服务器端本插件逻辑 | `ai_charger3.sp:108` |
| `ai_charger3_evade_moveto_refresh_interval` | `1.0` | 浮点 | 0.1 ~ 10.0 | ChargerEvade 追击时检查并刷新 BehaviorMoveTo 目标坐标的间隔，范围 0.1~10 | 服务器端本插件逻辑 | `ai_charger3.sp:110` |
| `ai_charger3_path_lookahead_maxdepth` | `10` | 整数，源码写作浮点 | ≥ 0.0 | 向前搜索一步可到达 PathSegment 的最大深度 | 服务器端本插件逻辑 | `ai_charger3.sp:112` |
| `ai_charger3_plugin_name` | `ai_charger3` | 字符串或表达式 | 无上下界 | 插件名（非行为配置） | 服务器端本插件逻辑 | `ai_charger3.sp:135` |
| `ai_charger3_log_level` | `1` | 整数 | 无上下界 | 日志记录级别：1 关闭，2 控制台输出，4 log 文件，8 聊天框，16 服务器控制台，32 error 文件，可相加 | 服务器端本插件逻辑 | `ai_charger3.sp:141` |

## Ai Charger 增强 2.0 版本 —— `optional/AnneHappy/ai_charger_2.sp`

- 源文件：`optional/AnneHappy/ai_charger_2.sp`
- myinfo：version=2.0.0.0 / 2022/7/14，author=夜羽真白
- myinfo description（源码原文）：Ai Charger 2.0

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `ai_ChargerBhop` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否开启 Charger 连跳 | 服务器端本插件逻辑 | `ai_charger_2.sp:38` |
| `ai_ChagrerBhopSpeed` | `90.0` | 整数，源码写作浮点 | ≥ 0.0 | Charger 连跳速度 | 服务器端本插件逻辑 | `ai_charger_2.sp:39` |
| `ai_ChargerChargeDistance` | `250.0` | 整数，源码写作浮点 | ≥ 0.0 | Charger 只能在与目标距离小于该值时冲锋 | 服务器端本插件逻辑 | `ai_charger_2.sp:40` |
| `ai_ChargerExtraTargetDistance` | `0,350` | 字符串或表达式 | 无上下界 | Charger 在该范围内寻找其他有效目标，逗号分隔且无空格 | 服务器端本插件逻辑 | `ai_charger_2.sp:41` |
| `ai_ChargerAimOffset` | `30.0` | 整数，源码写作浮点 | ≥ 0.0 | 目标瞄准水平与 Charger 夹角处于该范围内时 Charger 不冲锋 | 服务器端本插件逻辑 | `ai_charger_2.sp:42` |
| `ai_ChargerMeleeAvoid` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否开启 Charger 近战回避 | 服务器端本插件逻辑 | `ai_charger_2.sp:43` |
| `ai_ChargerMeleeDamage` | `350` | 整数，源码写作浮点 | ≥ 0.0 | Charger 血量小于该值时不会直接冲锋持有近战的生还者 | 服务器端本插件逻辑 | `ai_charger_2.sp:44` |
| `ai_ChargerTarget` | `1` | 整数，源码写作浮点 | 1.0 ~ 2.0 | Charger 目标选择：1 自然目标选择，2 优先取最近目标，3 优先撞人多处 | 服务器端本插件逻辑 | `ai_charger_2.sp:45` |
| `ai_ChargerChargeHeightDiff` | `80.0` | 整数，源码写作浮点 | 无上下界 | 允许直接冲锋时目标高出自身的最大高度差，小于等于 0 关闭检测 | 服务器端本插件逻辑 | `ai_charger_2.sp:46` |

## Ai-Charger增强 —— `optional/AnneHappy/ai_charger_new.sp`

- 源文件：`optional/AnneHappy/ai_charger_new.sp`
- myinfo：version=2022/5/2，author=Breezy，High Cookie，Standalone，Newteee，cravenge，Harry，Sorallll，PaimonQwQ，夜羽真白，东
- myinfo description（源码原文）：觉得Ai-Charger不够强？ Try this！

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `ai_ChargerCoolTime` | `12` | 整数，源码写作浮点 | 0.0 ~ 1.0 | Charger 多少秒后才能再次冲锋 | 服务器端本插件逻辑 | `ai_charger_new.sp:42` |
| `ai_ChargerBhop` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否开启 Charger 连跳 | 服务器端本插件逻辑 | `ai_charger_new.sp:43` |
| `ai_ChargerBhopSpeed` | `90.0` | 整数，源码写作浮点 | ≥ 0.0 | Charger 连跳的速度 | 服务器端本插件逻辑 | `ai_charger_new.sp:44` |
| `ai_ChargerTarget` | `3` | 整数，源码写作浮点 | 1.0 ~ 2.0 | Charger 目标选择：1 自然目标选择，2 优先撞人多处，3 优先取最近目标 | 服务器端本插件逻辑 | `ai_charger_new.sp:45` |
| `ai_ChargerStartChargeDistance` | `300` | 整数，源码写作浮点 | ≥ 0.0 | Charger 只能在与目标距离小于该值时冲锋 | 服务器端本插件逻辑 | `ai_charger_new.sp:46` |
| `ai_ChargerAimOffset` | `30` | 整数，源码写作浮点 | ≥ 0.0 | 目标瞄准角度与 Charger 处于该角度内时 Charger 不冲锋 | 服务器端本插件逻辑 | `ai_charger_new.sp:47` |
| `ai_ChargerStartChargeHealth` | `350` | 整数，源码写作浮点 | ≥ 0.0 | Charger 生命值低于该值才会冲锋 | 服务器端本插件逻辑 | `ai_charger_new.sp:48` |
| `ai_ChargerAirAngles` | `60.0` | 整数，源码写作浮点 | ≥ 0.0 | 空中速度向量与到生还者方向向量夹角大于该值时停止连跳 | 服务器端本插件逻辑 | `ai_charger_new.sp:49` |

## Ai Hunter 2.0 (fixed) —— `optional/AnneHappy/ai_hunter_2.sp`

- 源文件：`optional/AnneHappy/ai_hunter_2.sp`
- myinfo：version=2025-09-24，author=夜羽真白, patch by ChatGPT
- myinfo description（源码原文）：Ai Hunter 增强（修正若干编译/运行问题与健壮性）

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `ai_hunter_fast_pounce_distance` | `1000.0` | 整数，源码写作浮点 | ≥ 0.0 | hunter 开始快速突袭的距离 | 服务器端本插件逻辑 | `ai_hunter_2.sp:114` |
| `ai_hunter_vertical_angle` | `7.0` | 整数，源码写作浮点 | ≥ 0.0 | hunter 突袭的垂直角度上限，单位度 | 服务器端本插件逻辑 | `ai_hunter_2.sp:116` |
| `ai_hunter_angle_mean` | `10.0` | 整数，源码写作浮点 | ≥ 0.0 | 由随机数生成的基本角度 | 服务器端本插件逻辑 | `ai_hunter_2.sp:118` |
| `ai_hunter_angle_std` | `20.0` | 整数，源码写作浮点 | ≥ 0.0 | 与基本角度允许的偏差范围 | 服务器端本插件逻辑 | `ai_hunter_2.sp:120` |
| `ai_hunter_straight_pounce_distance` | `200.0` | 整数，源码写作浮点 | ≥ 0.0 | hunter 允许直扑的范围 | 服务器端本插件逻辑 | `ai_hunter_2.sp:122` |
| `ai_hunter_aim_offset` | `360.0` | 整数，源码写作浮点 | 0.0 ~ 360.0 | 与目标水平角度在该范围内且在直扑范围外时 hunter 不直扑，范围 0~360 | 服务器端本插件逻辑 | `ai_hunter_2.sp:124` |
| `ai_hunter_no_sight_pounce_range` | `300.0,250.0` | 字符串或表达式 | 无上下界 | 不可见目标时允许飞扑的范围，格式为水平,垂直，0 表示该维度禁用 | 服务器端本插件逻辑 | `ai_hunter_2.sp:128` |
| `ai_hunter_back_vision` | `25` | 整数，源码写作浮点 | 0.0 ~ 100.0 | hunter 在空中背对生还者视角的概率百分比，0 禁用，范围 0~100 | 服务器端本插件逻辑 | `ai_hunter_2.sp:132` |
| `ai_hunter_melee_first` | `300.0,1000.0` | 字符串或表达式 | 无上下界 | 每次准备突袭时是否先按右键，格式为最小,最大距离，0 禁用 | 服务器端本插件逻辑 | `ai_hunter_2.sp:135` |
| `ai_hunter_high_pounce` | `400` | 整数，源码写作浮点 | ≥ 0.0 | 高度差超过该值时可直接高扑，单位为 Hammer 坐标 Z | 服务器端本插件逻辑 | `ai_hunter_2.sp:138` |
| `ai_hunter_wall_detect_distance` | `-1.0` | 整数，源码写作浮点 | 无上下界 | 视线前方墙体检测的射线长度，-1 关闭 | 服务器端本插件逻辑 | `ai_hunter_2.sp:142` |
| `ai_hunter_angle_diff` | `3` | 整数，源码写作浮点 | ≥ 0.0 | 随机侧飞时左右累计次数差的上限 | 服务器端本插件逻辑 | `ai_hunter_2.sp:145` |

## AI HUNTER —— `optional/AnneHappy/ai_hunter_new.sp`

- 源文件：`optional/AnneHappy/ai_hunter_new.sp`
- myinfo：version=1.0，author=Breezy
- myinfo description（源码原文）：Improves the AI behaviour of special infected

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `ai_fast_pounce_proximity` | `1000.0` | 整数，源码写作浮点 | 无上下界 | 从多远开始快速飞扑 | 服务器端本插件逻辑 | `ai_hunter_new.sp:47` |
| `ai_pounce_vertical_angle` | `7.0` | 整数，源码写作浮点 | 无上下界 | AI hunter 飞扑的垂直角度限制 | 服务器端本插件逻辑 | `ai_hunter_new.sp:48` |
| `ai_pounce_angle_mean` | `10.0` | 整数，源码写作浮点 | 无上下界 | 高斯随机数生成的角度均值 | 服务器端本插件逻辑 | `ai_hunter_new.sp:49` |
| `ai_pounce_angle_std` | `20.0` | 整数，源码写作浮点 | 无上下界 | 高斯随机数生成的一个标准差 | 服务器端本插件逻辑 | `ai_hunter_new.sp:50` |
| `ai_straight_pounce_proximity` | `200.0` | 整数，源码写作浮点 | 无上下界 | 距最近生还者多远时 hunter 考虑直扑 | 服务器端本插件逻辑 | `ai_hunter_new.sp:51` |
| `ai_aim_offset_sensitivity_hunter` | `180.0` | 整数，源码写作浮点 | 0.0 ~ 180.0 | 若目标水平瞄准在该半径内则 hunter 不会直扑，范围 0~180 | 服务器端本插件逻辑 | `ai_hunter_new.sp:52` |
| `ai_wall_detection_distance` | `-1.0` | 整数，源码写作浮点 | 无上下界 | 感染者 Bot 在前方多远检测墙壁，填 -1 关闭该功能 | 服务器端本插件逻辑 | `ai_hunter_new.sp:53` |

## Ai_Jockey 2.0 版本 —— `optional/AnneHappy/ai_jockey_2.sp`

- 源文件：`optional/AnneHappy/ai_jockey_2.sp`
- myinfo：version=2026.08.29，author=Breezy，High Cookie，Standalone，Newteee，cravenge，Harry，Sorallll，PaimonQwQ，夜羽真白
- myinfo description（源码原文）：觉得Ai猴子太弱了？ Try this！

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `ai_JockeyBhopSpeed` | `80.0` | 整数，源码写作浮点 | ≥ 0.0 | Jockey 连跳的速度 | 服务器端本插件逻辑 | `ai_jockey_2.sp:60` |
| `ai_JockeyStartHopDistance` | `800` | 整数，源码写作浮点 | ≥ 0.0 | Jockey 距生还者多远开始主动连跳 | 服务器端本插件逻辑 | `ai_jockey_2.sp:61` |
| `ai_JockeyStumbleRadius` | `50` | 整数，源码写作浮点 | ≥ 0.0 | Jockey 骑上人后对多少范围内的生还者产生硬直 | 服务器端本插件逻辑 | `ai_jockey_2.sp:62` |
| `ai_JockeySpecialJumpAngle` | `60` | 整数，源码写作浮点 | 0.0 ~ 180.0 | 目标正看着 Jockey 且夹角在该范围内时 Jockey 尝试骗推，范围 0~180 | 服务器端本插件逻辑 | `ai_jockey_2.sp:64` |
| `ai_JockeySpecialJumpChance` | `60` | 整数，源码写作浮点 | 0.0 ~ 100.0 | Jockey 执行骗推的概率百分比，范围 0~100 | 服务器端本插件逻辑 | `ai_jockey_2.sp:65` |
| `ai_jockeyNoActionChance` | `20,20,60` | 字符串或表达式 | 0.0 ~ 100.0 | Jockey 执行冻结行动、向后跳、高跳三种行为的概率，逗号分隔，范围 0~100 | 服务器端本插件逻辑 | `ai_jockey_2.sp:66` |
| `ai_JockeyAllowInterControl` | `0` | 整数 | 无上下界 | Jockey 优先寻找被这些特感控制的生还者以抢控或补控，0 表示关闭该功能 | 服务器端本插件逻辑 | `ai_jockey_2.sp:67` |
| `ai_JockeyBackVision` | `50` | 整数，源码写作浮点 | 0.0 ~ 100.0 | Jockey 在空中时以该概率向当前视角反方向看，范围 0~100 | 服务器端本插件逻辑 | `ai_jockey_2.sp:68` |
| `ai_JockeyShovedCooldown` | `3.0` | 整数，源码写作浮点 | ≥ 0.0 | Jockey 被推后多少秒内禁止再次扑跳 | 服务器端本插件逻辑 | `ai_jockey_2.sp:69` |
| `ai_ChargerBhop` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否开启 Charger 连跳，兼容旧插件名 | 服务器端本插件逻辑；经 getOrCreateLegacyConVar 包装，ConVar 已存在时不重复创建 | `ai_charger3.sp:115` |
| `ai_ChagrerBhopSpeed` | `90.0` | 整数，源码写作浮点 | ≥ 0.0 | Charger 连跳速度与加速度的兼容值 | 服务器端本插件逻辑；经 getOrCreateLegacyConVar 包装，ConVar 已存在时不重复创建 | `ai_charger3.sp:116` |
| `ai_ChargerChargeDistance` | `250.0` | 整数，源码写作浮点 | ≥ 0.0 | Charger 旧版冲锋距离，转换为 3.0 的直线连跳阈值 | 服务器端本插件逻辑；经 getOrCreateLegacyConVar 包装，ConVar 已存在时不重复创建 | `ai_charger3.sp:117` |
| `ai_ChargerExtraTargetDistance` | `0,350` | 字符串或表达式 | 无上下界 | Charger 额外目标范围，格式为最小距离,最大距离 | 服务器端本插件逻辑；经 getOrCreateLegacyConVar 包装，ConVar 已存在时不重复创建 | `ai_charger3.sp:118` |
| `ai_ChargerAimOffset` | `30.0` | 整数，源码写作浮点 | ≥ 0.0 | 目标视角与 Charger 的兼容判定角度 | 服务器端本插件逻辑；经 getOrCreateLegacyConVar 包装，ConVar 已存在时不重复创建 | `ai_charger3.sp:119` |
| `ai_ChargerMeleeAvoid` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用 Charger 近战回避 | 服务器端本插件逻辑；经 getOrCreateLegacyConVar 包装，ConVar 已存在时不重复创建 | `ai_charger3.sp:120` |
| `ai_ChargerMeleeDamage` | `350` | 整数，源码写作浮点 | ≥ 0.0 | Charger 血量低于该值时避免近战目标 | 服务器端本插件逻辑；经 getOrCreateLegacyConVar 包装，ConVar 已存在时不重复创建 | `ai_charger3.sp:121` |
| `ai_ChargerTarget` | `1` | 开关 0 或 1 | 1.0 ~ 3.0 | Charger 目标选择：1 原生，2 最近，3 人群中心 | 服务器端本插件逻辑；经 getOrCreateLegacyConVar 包装，ConVar 已存在时不重复创建 | `ai_charger3.sp:122` |
| `ai_ChargerChargeHeightDiff` | `80.0` | 整数，源码写作浮点 | 无上下界 | 允许直接冲锋的最大高度差，小于等于 0 时使用默认值 | 服务器端本插件逻辑；经 getOrCreateLegacyConVar 包装，ConVar 已存在时不重复创建 | `ai_charger3.sp:123` |

## Ai_Jockey增强 —— `optional/AnneHappy/ai_jockey_new.sp`

- 源文件：`optional/AnneHappy/ai_jockey_new.sp`
- myinfo：version=2022/11/1，author=Breezy，High Cookie，Standalone，Newteee，cravenge，Harry，Sorallll，PaimonQwQ，夜羽真白, 东
- myinfo description（源码原文）：觉得Ai猴子太弱了？ Try this！

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `ai_JockeyBhopSpeed` | `80.0` | 整数，源码写作浮点 | ≥ 0.0 | Jockey 连跳的速度 | 服务器端本插件逻辑 | `ai_jockey_new.sp:45` |
| `ai_JockeyStartHopDistance` | `800.0` | 整数，源码写作浮点 | ≥ 0.0 | Jockey 距生还者多远开始主动连跳 | 服务器端本插件逻辑 | `ai_jockey_new.sp:46` |
| `ai_JockeyStumbleRadius` | `50.0` | 整数，源码写作浮点 | ≥ 0.0 | Jockey 骑上人后对多少范围内的生还者产生硬直 | 服务器端本插件逻辑 | `ai_jockey_new.sp:47` |
| `ai_JockeyAirAngles` | `60.0` | 整数，源码写作浮点 | 0.0 ~ 180.0 | Jockey 速度方向与到目标向量方向夹角大于该角度时改变方向，范围 0~180 | 服务器端本插件逻辑 | `ai_jockey_new.sp:48` |

## Ai-Smoker 3.0 —— `optional/AnneHappy/ai_smoker3.sp`

- 源文件：`optional/AnneHappy/ai_smoker3.sp`
- myinfo：version=1.0.1.5，author=夜羽真白
- myinfo description（源码原文）：Ai-Smoker 增强 3.0 版本

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `ai_smoker3_bhop` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否允许 Smoker 连跳 | 服务器端本插件逻辑 | `ai_smoker3.sp:131` |
| `ai_smoker3_bhop_no_vision` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否允许无目标视野情况下连跳 | 服务器端本插件逻辑 | `ai_smoker3.sp:133` |
| `ai_SmokerBhopSpeed` | `120` | 整数，源码写作浮点 | ≥ 0.0 | 连跳加速度 | 服务器端本插件逻辑 | `ai_smoker3.sp:135` |
| `ai_smoker3_bhop_min_speed` | `200` | 整数，源码写作浮点 | ≥ 0.0 | 允许连跳的最小速度 | 服务器端本插件逻辑 | `ai_smoker3.sp:137` |
| `ai_smoker3_bhop_max_speed` | `1000` | 整数，源码写作浮点 | ≥ 0.0 | 连跳时的最大速度 | 服务器端本插件逻辑 | `ai_smoker3.sp:138` |
| `ai_smoker3_bhop_min_dist` | `75` | 整数，源码写作浮点 | ≥ 0.0 | 允许连跳的最小距离，与目标距离小于该值不允许连跳 | 服务器端本插件逻辑 | `ai_smoker3.sp:140` |
| `ai_smoker3_bhop_max_dist` | `9999` | 整数，源码写作浮点 | ≥ 0.0 | 允许连跳的最大距离，与目标距离大于该值不允许连跳 | 服务器端本插件逻辑 | `ai_smoker3.sp:141` |
| `ai_smoker3_bhop_side_minang` | `15.0` | 整数，源码写作浮点 | 0.0 ~ 180.0 | 连跳方向最小左右侧偏角度，范围 0~180 | 服务器端本插件逻辑 | `ai_smoker3.sp:143` |
| `ai_smoker3_bhop_side_maxang` | `30.0` | 整数，源码写作浮点 | 0.0 ~ 180.0 | 连跳方向最大左右侧偏角度，范围 0~180 | 服务器端本插件逻辑 | `ai_smoker3.sp:144` |
| `_ai_smoker3_bhop_nvis_maxang` | `75.0` | 整数，源码写作浮点 | ≥ 0.0 | 无生还者视野时速度向量与视角前向向量夹角在该范围内允许连跳 | 服务器端本插件逻辑 | `ai_smoker3.sp:146` |
| `ai_smoker3_airvec_modify_degree` | `50.0` | 整数，源码写作浮点 | ≥ 0.0 | 空中速度方向与自身到目标方向夹角超过该值时进行速度修正 | 服务器端本插件逻辑 | `ai_smoker3.sp:148` |
| `ai_smoker3_airvec_modify_degree_max` | `105.0` | 整数，源码写作浮点 | ≥ 0.0 | 空中速度方向与自身到目标方向夹角超过该值时不进行速度修正 | 服务器端本插件逻辑 | `ai_smoker3.sp:149` |
| `ai_smoker3_airvec_modify_interval` | `0.3` | 浮点 | ≥ 0.1 | 空中速度修正间隔，最小 0.1 | 服务器端本插件逻辑 | `ai_smoker3.sp:150` |
| `ai_smoker3_jump_pull` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否允许 Smoker 已在空中时主动吐舌 | 服务器端本插件逻辑 | `ai_smoker3.sp:151` |
| `ai_smoker3_pull_back_vision` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否允许 Smoker 拉人时视角转向背后 | 服务器端本插件逻辑 | `ai_smoker3.sp:153` |
| `ai_smoker3_anti_retreat` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否防止 Smoker 无技能时逃跑，将其改为追击 | 服务器端本插件逻辑 | `ai_smoker3.sp:156` |
| `ai_smoker3_move2_newtar_interval` | `1.0` | 整数，源码写作浮点 | ≥ 0.0 | 无技能追击时检测最近生还者并更换目标的间隔秒数 | 服务器端本插件逻辑 | `ai_smoker3.sp:158` |
| `ai_smoker3_stop_warn_snd` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否停止 Smoker 准备吐舌时的警告音效 | 服务器端本插件逻辑 | `ai_smoker3.sp:161` |
| `ai_smoker3_plugin_name` | `ai_smoker3` | 字符串或表达式 | 无上下界 | 插件名（非行为配置） | 服务器端本插件逻辑 | `ai_smoker3.sp:164` |
| `ai_smoker3_log_level` | `32` | 整数 | 无上下界 | 日志记录级别：1 关闭，2 控制台，4 log 文件，8 聊天框，16 服务器控制台，32 error 文件，可相加 | 服务器端本插件逻辑 | `ai_smoker3.sp:169` |

## Ai_Smoker增强 —— `optional/AnneHappy/ai_smoker_new.sp`

- 源文件：`optional/AnneHappy/ai_smoker_new.sp`
- myinfo：version=2022/5/2，author=Breezy，High Cookie，Standalone，Newteee，cravenge，Harry，Sorallll，PaimonQwQ，夜羽真白，东
- myinfo description（源码原文）：觉得Ai舌头太弱了？ Try this！

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `ai_SmokerBhop` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否开启 Smoker 连跳 | 服务器端本插件逻辑 | `ai_smoker_new.sp:73` |
| `ai_SmokerBhopSpeed` | `90.0` | 整数，源码写作浮点 | ≥ 0.0 | Smoker 连跳的速度 | 服务器端本插件逻辑 | `ai_smoker_new.sp:74` |
| `ai_SmokerTarget` | `1` | 整数，源码写作浮点 | 1.0 ~ 4.0 | Smoker 优先目标：1 距离最近，2 手持霰弹枪者，3 落单或超前者，4 正在换弹者 | 服务器端本插件逻辑 | `ai_smoker_new.sp:75` |
| `ai_SmokerMeleeAvoid` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | Smoker 的目标若手持近战则切换目标 | 服务器端本插件逻辑 | `ai_smoker_new.sp:76` |
| `ai_SmokerLeftBehindDistance` | `7.0` | 整数，源码写作浮点 | ≥ 0.0 | 玩家距离团队多远判定为落后或超前 | 服务器端本插件逻辑 | `ai_smoker_new.sp:79` |
| `ai_SmokerDistantPercent` | `0.80` | 浮点 | ≥ 0.0 | 舌头处于该系数乘以舌头长度的距离内时立刻拉人 | 服务器端本插件逻辑 | `ai_smoker_new.sp:80` |

## Ai-Spitter-Enhance 2.0 —— `optional/AnneHappy/ai_spitter_2.sp`

- 源文件：`optional/AnneHappy/ai_spitter_2.sp`
- myinfo：version=2023-1-3，author=夜羽真白
- myinfo description（源码原文）：Ai Spitter 增强 2.0 版本

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `ai_SpitterBhop` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否开启 Spitter 连跳功能 | 服务器端本插件逻辑 | `ai_spitter_2.sp:50` |
| `ai_SpitterBhopSpeed` | `100` | 整数，源码写作浮点 | ≥ 0.0 | Spitter 连跳的速度 | 服务器端本插件逻辑 | `ai_spitter_2.sp:51` |
| `ai_SpitterTarget` | `3` | 整数，源码写作浮点 | 1.0 ~ 4.0 | Spitter 目标选择：1 默认，2 最近，3 被控优先否则第一个生还，4 人多处 | 服务器端本插件逻辑 | `ai_spitter_2.sp:52` |
| `ai_SpitterPinnedPr` | `6,3,1,5` | 字符串或表达式 | 无上下界 | 被控目标优先级，按被控特感编号逗号分隔 | 服务器端本插件逻辑 | `ai_spitter_2.sp:53` |
| `ai_SpiiterDieAfterSpit` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否开启 Spitter 吐完痰后处死功能 | 服务器端本插件逻辑 | `ai_spitter_2.sp:54` |

## Ai Spitter 3.0 —— `optional/AnneHappy/ai_spitter_3.sp`

- 源文件：`optional/AnneHappy/ai_spitter_3.sp`
- myinfo：version=3.0.11，author=夜羽真白
- myinfo description（源码原文）：Ai Spitter 增强 3.0（anne_nextbot Path Follow）

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `ai_SpitterBhop` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否开启 Spitter 连跳功能 | 服务器端本插件逻辑 | `ai_spitter_3.sp:67` |
| `ai_SpitterBhopSpeed` | `100` | 整数，源码写作浮点 | ≥ 0.0 | Spitter 连跳推力，落地后立即再起跳时追加，第一跳按 first_hop_ratio 折算 | 服务器端本插件逻辑 | `ai_spitter_3.sp:68` |
| `ai_SpitterBhopFirstHopRatio` | `0.8` | 浮点 | 0.0 ~ 1.0 | 从跑动直接起跳的第一跳可获得的推力比例，0.0 不加推力，1.0 与落地跳相同 | 服务器端本插件逻辑 | `ai_spitter_3.sp:69` |
| `ai_SpitterBhopMaxSpeed` | `1000.0` | 整数，源码写作浮点 | ≥ 0.0 | Spitter 连跳的最大水平速度，0 不限制 | 服务器端本插件逻辑 | `ai_spitter_3.sp:70` |
| `ai_SpitterBhopStartDistance` | `2500.0` | 整数，源码写作浮点 | ≥ 0.0 | Spitter 距最近生还者多远开始连跳 | 服务器端本插件逻辑 | `ai_spitter_3.sp:71` |
| `ai_spitter3_path_bhop` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否优先使用 anne_nextbot 路径前视连跳 | 服务器端本插件逻辑 | `ai_spitter_3.sp:72` |
| `ai_spitter3_path_lookahead_depth` | `6` | 整数，源码写作浮点 | 1.0 ~ 16.0 | 路径连跳最大前视节点数，范围 1~16 | 服务器端本插件逻辑 | `ai_spitter_3.sp:73` |
| `ai_spitter3_path_lane_offset` | `12.0` | 整数，源码写作浮点 | 0.0 ~ 40.0 | 路径连跳的最大稳定侧向分流距离，范围 0~40 | 服务器端本插件逻辑 | `ai_spitter_3.sp:74` |
| `ai_SpitterTarget` | `3` | 整数，源码写作浮点 | 1.0 ~ 4.0 | Spitter 目标选择：1 默认，2 最近，3 被控优先否则第一个生还，4 人多处 | 服务器端本插件逻辑 | `ai_spitter_3.sp:75` |
| `ai_SpitterPinnedPr` | `6,3,1,5` | 字符串或表达式 | 无上下界 | 被控目标优先级，按被控特感编号逗号分隔 | 服务器端本插件逻辑 | `ai_spitter_3.sp:76` |
| `ai_SpiiterDieAfterSpit` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否开启 Spitter 吐完痰后处死功能 | 服务器端本插件逻辑 | `ai_spitter_3.sp:77` |
| `ai_spitter3_air_spit` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否允许 AI Spitter 空中吐痰：0 禁止，1 允许 | 服务器端本插件逻辑 | `ai_spitter_3.sp:78` |

## Ai-Spitter增强 —— `optional/AnneHappy/ai_spitter_new.sp`

- 源文件：`optional/AnneHappy/ai_spitter_new.sp`
- myinfo：version=22-4-24，author=Breezy，High Cookie，Standalone，Newteee，cravenge，Harry，Sorallll，PaimonQwQ，夜羽真白
- myinfo description（源码原文）：觉得Ai Spitter不够强？ Try this！

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `ai_SpitterBhop` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否开启 Spitter 连跳 | 服务器端本插件逻辑 | `ai_spitter_new.sp:35` |
| `ai_SpitterBhopSpeed` | `90.0` | 整数，源码写作浮点 | ≥ 0.0 | Spitter 连跳的速度 | 服务器端本插件逻辑 | `ai_spitter_new.sp:36` |
| `ai_SpitterBhopStartBhopDistance` | `2000.0` | 整数，源码写作浮点 | ≥ 0.0 | Spitter 在什么距离开始连跳 | 服务器端本插件逻辑 | `ai_spitter_new.sp:37` |
| `ai_SpitterTarget` | `3` | 整数，源码写作浮点 | 1.0 ~ 3.0 | Spitter 目标选择：1 默认，2 人多处优先，3 被扑撞拉者优先，无则取 2 | 服务器端本插件逻辑 | `ai_spitter_new.sp:38` |
| `ai_SpitterInstantKill` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | Spitter 吐完痰之后是否处死 | 服务器端本插件逻辑 | `ai_spitter_new.sp:39` |
| `ai_SpitterAirAngle` | `55.0` | 整数，源码写作浮点 | ≥ 0.0 | Spitter 连跳时速度与到目标生还者方向夹角超过该角度即停止连跳 | 服务器端本插件逻辑 | `ai_spitter_new.sp:40` |

## Ai-Tank 3 —— `optional/AnneHappy/ai_tank3.sp`

- 源文件：`optional/AnneHappy/ai_tank3.sp`
- myinfo：version=2.3.0，author=夜羽真白, AnneHappy
- myinfo description（源码原文）：Ai Tank 增强 3.0 版本（路径感知连跳、梯子让行、寻路距离选目标、反头顶卡、骑头反制、投石瞄准等）

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `ai_tank3_enable` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用插件：0 禁用，1 启用 | 服务器端本插件逻辑 | `ai_tank3.sp:63` |
| `ai_tank_bhop` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否允许坦克连跳 | 服务器端本插件逻辑 | `ai_tank3.sp:66` |
| `ai_Tank_StopDistance` | `135` | 整数，源码写作浮点 | ≥ 0.0 | 停止连跳的最小距离 | 服务器端本插件逻辑 | `ai_tank3.sp:67` |
| `ai_tank3_bhop_max_dist` | `9999` | 整数，源码写作浮点 | ≥ 0.0 | 开始连跳的最大距离 | 服务器端本插件逻辑 | `ai_tank3.sp:68` |
| `ai_tank3_bhop_min_speed` | `200` | 整数，源码写作浮点 | ≥ 0.0 | 连跳的最小速度 | 服务器端本插件逻辑 | `ai_tank3.sp:69` |
| `ai_tank3_bhop_max_speed` | `1000` | 整数，源码写作浮点 | ≥ 0.0 | 连跳的最大速度 | 服务器端本插件逻辑 | `ai_tank3.sp:70` |
| `ai_tank3_bhop_impulse` | `60` | 整数，源码写作浮点 | ≥ 0.0 | 连跳的加速度，落地后立即再起跳时追加，第一跳按 first_hop_ratio 折算 | 服务器端本插件逻辑 | `ai_tank3.sp:71` |
| `ai_tank3_bhop_first_hop_ratio` | `1.0` | 开关 0 或 1 | 0.0 ~ 1.0 | 从跑动直接起跳的第一跳可获得的加速度比例，0.0 不加速只改方向，1.0 与落地跳相同 | 服务器端本插件逻辑 | `ai_tank3.sp:72` |
| `ai_tank3_bhop_no_vision` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 无视野时是否允许连跳 | 服务器端本插件逻辑 | `ai_tank3.sp:73` |
| `_ai_tank3_bhop_nvis_maxang` | `75.0` | 整数，源码写作浮点 | ≥ 0.0 | 无视野时速度向量与视角前向向量夹角阈值，单位度 | 服务器端本插件逻辑 | `ai_tank3.sp:74` |
| `ai_tank3_path_lookahead_maxdepth` | `10` | 整数，源码写作浮点 | ≥ 1.0 | 沿路径连跳时向前搜索 PathSegment 的最大深度，最小 1 | 服务器端本插件逻辑 | `ai_tank3.sp:75` |
| `_ai_tank3_direct_chase_max_angle` | `45.0` | 整数，源码写作浮点 | 0.0 ~ 180.0 | 有视野时路径前瞻方向与目标方向夹角不超过该值才朝预测点直追，否则沿路径连跳，单位度 | 服务器端本插件逻辑 | `ai_tank3.sp:76` |
| `ai_tank3_bhop_strafe_angle` | `15.0` | 整数，源码写作浮点 | 0.0 ~ 35.0 | 远距离安全直追时逐跳左右交替的偏角，0 关闭，范围 0~35 | 服务器端本插件逻辑 | `ai_tank3.sp:77` |
| `ai_tank3_bhop_strafe_min_dist` | `600.0` | 整数，源码写作浮点 | ≥ 0.0 | 距离目标超过该值才主动左右连跳 | 服务器端本插件逻辑 | `ai_tank3.sp:78` |
| `ai_tank3_bhop_reverse_hop` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 直追时目标跑到身后则落地改为掉头跳；0 为旧行为，会顺着原方向继续跳 | 服务器端本插件逻辑 | `ai_tank3.sp:79` |
| `ai_tank3_bhop_reverse_brake` | `1500` | 整数，源码写作浮点 | ≥ 0.0 | 直追的一跳中目标跑到身后时空中每秒减掉的水平速度，最多减到跑速，0 不刹车 | 服务器端本插件逻辑 | `ai_tank3.sp:80` |
| `ai_tank3_airvec_modify_degree` | `45.0` | 整数，源码写作浮点 | ≥ 0.0 | 追人时空中速度方向与目标方向夹角大于等于该值开始修正，单位度 | 服务器端本插件逻辑 | `ai_tank3.sp:83` |
| `ai_tank3_airvec_modify_degree_max` | `135.0` | 整数，源码写作浮点 | ≥ 0.0 | 角度大于该值时不再修正，实际最大 89 度 | 服务器端本插件逻辑 | `ai_tank3.sp:84` |
| `ai_tank3_airvec_modify_interval` | `0.3` | 浮点 | ≥ 0.1 | 空中转向平滑响应时间，单位秒，每 0.05 秒检查一次，最小 0.1 | 服务器端本插件逻辑 | `ai_tank3.sp:85` |
| `ai_tank3_throw_min_dist` | `0` | 整数，源码写作浮点 | ≥ 0.0 | 允许扔石头的最小距离 | 服务器端本插件逻辑 | `ai_tank3.sp:88` |
| `ai_tank3_throw_max_dist` | `800` | 整数，源码写作浮点 | ≥ 0.0 | 允许扔石头的最大距离 | 服务器端本插件逻辑 | `ai_tank3.sp:89` |
| `ai_tank3_rock_target_adjust` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 出手时改为瞄准最近的可视生还者，强制投石计划指定的目标除外 | 服务器端本插件逻辑 | `ai_tank3.sp:90` |
| `ai_tank3_jump_rock` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 扔石头起手时是否允许跳砖 | 服务器端本插件逻辑 | `ai_tank3.sp:91` |
| `ai_tank3_back_fist` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否允许通背拳，即可拍到背后的人 | 服务器端本插件逻辑 | `ai_tank3.sp:92` |
| `ai_tank3_back_fist_range` | `128.0` | 整数，源码写作浮点 | ≥ -1.0 | 通背拳距离，-1 表示使用 tank_swing_range | 服务器端本插件逻辑 | `ai_tank3.sp:93` |
| `ai_tank3_back_fist_max_spd` | `50.0` | 整数，源码写作浮点 | ≥ -1.0 | 通背拳允许的最大移动速度，超过则禁用，-1 表示不限制 | 服务器端本插件逻辑 | `ai_tank3.sp:94` |
| `ai_tank3_back_fist_window` | `3.0` | 整数，源码写作浮点 | ≥ 0.0 | 通背拳窗口秒数，Tank 爪击命中后开启或刷新 | 服务器端本插件逻辑 | `ai_tank3.sp:95` |
| `ai_tank3_punch_lock_vision` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 挥拳时是否把视角锁定到目标 | 服务器端本插件逻辑 | `ai_tank3.sp:96` |
| `ai_tank3_target_select` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | Tank 选目标方式：0 交给 l4d_target_override 或原生排序，1 沿用其过滤口径但按寻路距离排序并带换目标粘滞 | 服务器端本插件逻辑 | `ai_tank3.sp:99` |
| `ai_tank3_target_switch_ratio` | `0.85` | 浮点 | 0.1 ~ 1.0 | 同一层换目标时新目标得分须不超过当前目标的该比例 | 服务器端本插件逻辑 | `ai_tank3.sp:100` |
| `ai_tank3_target_switch_gain` | `75` | 整数，源码写作浮点 | ≥ 0.0 | 同一层换目标时新目标得分至少要比当前目标低这么多，按寻路距离单位 | 服务器端本插件逻辑 | `ai_tank3.sp:101` |
| `ai_tank3_target_commit_time` | `1.0` | 整数，源码写作浮点 | ≥ 0.0 | 两次主动换目标之间的最短间隔秒数 | 服务器端本插件逻辑 | `ai_tank3.sp:102` |
| `ai_tank3_target_decisive_ratio` | `0.5` | 浮点 | 0.0 ~ 1.0 | 新目标得分不超过当前目标的该比例时视为明显更好打，0 关闭 | 服务器端本插件逻辑 | `ai_tank3.sp:103` |
| `ai_tank3_head_block_enable` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用 Tank 反头顶卡逻辑 | 服务器端本插件逻辑 | `ai_tank3.sp:106` |
| `ai_tank3_head_block_time` | `2.0` | 整数，源码写作浮点 | ≥ 0.0 | Tank 位于目标脚下的持续时间阈值秒数，可经梯子走到目标时按 3 倍计算 | 服务器端本插件逻辑 | `ai_tank3.sp:107` |
| `ai_tank3_head_block_vertical` | `80.0` | 整数，源码写作浮点 | ≥ 0.0 | 触发头顶卡判定需要的垂直距离 | 服务器端本插件逻辑 | `ai_tank3.sp:108` |
| `ai_tank3_head_block_horizontal` | `65.0` | 整数，源码写作浮点 | ≥ 0.0 | 触发头顶卡判定的水平距离上限 | 服务器端本插件逻辑 | `ai_tank3.sp:109` |
| `ai_tank3_head_block_ignore_time` | `10.0` | 整数，源码写作浮点 | ≥ 0.0 | 判定恶意卡位后屏蔽该生还者的秒数 | 服务器端本插件逻辑 | `ai_tank3.sp:110` |
| `ai_tank3_head_block_force_rock_time` | `20.0` | 整数，源码写作浮点 | ≥ 0.0 | 强制投石尝试的最长秒数 | 服务器端本插件逻辑 | `ai_tank3.sp:111` |
| `ai_tank3_head_block_force_rock_range` | `250.0` | 整数，源码写作浮点 | ≥ 0.0 | 强制投石前 Tank 需与目标拉开的最小水平距离 | 服务器端本插件逻辑 | `ai_tank3.sp:112` |
| `ai_tank3_head_block_force_rock_release_h` | `400` | 整数，源码写作浮点 | ≥ 0.0 | 强制投石期间目标水平离开多远即清除强制状态，小于等于 0 不检测 | 服务器端本插件逻辑 | `ai_tank3.sp:113` |
| `ai_tank3_head_block_force_rock_release_v` | `250` | 整数，源码写作浮点 | ≥ 0.0 | 强制投石期间目标垂直离开多远即清除强制状态，小于等于 0 不检测 | 服务器端本插件逻辑 | `ai_tank3.sp:114` |
| `ai_tank3_head_block_ride_enable` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否把 Tank 正上方的生还者按骑头处理，能命中则上挥否则原地投石 | 服务器端本插件逻辑 | `ai_tank3.sp:115` |
| `ai_tank3_head_block_ride_horizontal` | `40.0` | 整数，源码写作浮点 | ≥ 0.0 | 骑头判定的水平距离上限 | 服务器端本插件逻辑 | `ai_tank3.sp:116` |
| `ai_tank3_head_block_ride_vertical_min` | `40.0` | 整数，源码写作浮点 | ≥ 0.0 | 骑头判定的最小垂直差 | 服务器端本插件逻辑 | `ai_tank3.sp:117` |
| `ai_tank3_head_block_ride_vertical_max` | `100.0` | 整数，源码写作浮点 | ≥ 0.0 | 骑头判定的最大垂直差，超过视为高台 | 服务器端本插件逻辑 | `ai_tank3.sp:118` |
| `ai_tank3_head_block_ride_rock_time` | `3.0` | 整数，源码写作浮点 | ≥ 0.0 | 拳打不到的骑头目标持续多久后开始原地投石 | 服务器端本插件逻辑 | `ai_tank3.sp:119` |
| `ai_tank3_head_block_up_swing` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 挥拳时是否对骑头生还者额外向上 SweepFist | 服务器端本插件逻辑 | `ai_tank3.sp:120` |
| `ai_tank3_retreat_timeout` | `3.0` | 浮点 | ≥ 0.5 | 反头顶卡撤离投石时单次 MOVE 命令的最长持续秒数，到时必定 RESET | 服务器端本插件逻辑 | `ai_tank3.sp:121` |
| `ai_tank3_ladder_look_lock` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | Tank 在梯子上时是否把视角锁到梯子朝向 | 服务器端本插件逻辑 | `ai_tank3.sp:124` |
| `ai_tank3_ladder_nearby_disable` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | Tank 即将爬梯时是否暂停连跳与挥拳锁视角并压回跑速 | 服务器端本插件逻辑 | `ai_tank3.sp:125` |
| `ai_tank3_ladder_nearby_radius` | `180.0` | 整数，源码写作浮点 | ≥ 0.0 | 沿路径距离梯子入口多近算即将爬梯 | 服务器端本插件逻辑 | `ai_tank3.sp:126` |
| `ai_tank3_ladder_nearby_cache` | `0.20` | 浮点 | ≥ 0.0 | 路径快照不可用时梯子实体检测的缓存秒数 | 服务器端本插件逻辑 | `ai_tank3.sp:127` |
| `ai_tank3_plugin_name` | `ai_tank3` | 字符串或表达式 | 无上下界 | 插件名（非行为配置） | 服务器端本插件逻辑 | `ai_tank3.sp:130` |
| `ai_tank3_log_level` | `32` | 整数 | 无上下界 | 日志级别：1 关，2 控制台，4 log，8 聊天，16 服务器控制台，32 error 文件，可相加 | 服务器端本插件逻辑 | `ai_tank3.sp:134` |

## Ai_Tank_Enhance2.0 —— `optional/AnneHappy/ai_tank_2.sp`

- 源文件：`optional/AnneHappy/ai_tank_2.sp`
- myinfo：version=2.0.1.0，author=夜羽真白，东
- myinfo description（源码原文）：Tank 增强插件 2.0 版本

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_checkladder` | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 输出当前地图的梯子数量 | `ai_tank_2.sp:167` |

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `ai_Tank_Bhop` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否开启坦克连跳 | 服务器端本插件逻辑 | `ai_tank_2.sp:123` |
| `ai_TankBhopSpeed` | `60` | 整数，源码写作浮点 | ≥ 0.0 | 坦克连跳速度 | 服务器端本插件逻辑 | `ai_tank_2.sp:124` |
| `ai_Tank_StopDistance` | `135` | 整数，源码写作浮点 | ≥ 0.0 | Tank 距目标多远时停止连跳 | 服务器端本插件逻辑 | `ai_tank_2.sp:125` |
| `ai_TankConsume` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否开启坦克消耗 | 服务器端本插件逻辑 | `ai_tank_2.sp:127` |
| `ai_TankSneakTime` | `0` | 整数，源码写作浮点 | 0.0 ~ 28.0 | tank 会消耗到下一波生成时间小于该值，0 为关闭 | 服务器端本插件逻辑 | `ai_tank_2.sp:128` |
| `ai_TankConsumeInfSub` | `1` | 整数，源码写作浮点 | ≥ 0.0 | 当前特感数量小于等于特感上限减去该值时坦克可以消耗 | 服务器端本插件逻辑 | `ai_tank_2.sp:129` |
| `ai_TankConsumeRayRaidus` | `1500` | 整数，源码写作浮点 | ≥ 0.0 | 射线寻找消耗位的范围，从坦克当前位置开始计算 | 服务器端本插件逻辑 | `ai_tank_2.sp:130` |
| `ai_TankConsumeDistance` | `1200` | 整数，源码写作浮点 | ≥ 0.0 | 射线找到的消耗位需要离生还者这么远 | 服务器端本插件逻辑 | `ai_tank_2.sp:131` |
| `ai_TankFindNewConsumePosDistance` | `750` | 整数，源码写作浮点 | ≥ 0.0 | 最近的生还者离坦克这么远时坦克重新找消耗位 | 服务器端本插件逻辑 | `ai_tank_2.sp:132` |
| `ai_TankForceAttackDist` | `350` | 整数，源码写作浮点 | ≥ 0.0 | 生还者距离坦克这么近时坦克强制攻击 | 服务器端本插件逻辑 | `ai_tank_2.sp:133` |
| `ai_TankForceAttackProgress` | `10` | 整数，源码写作浮点 | ≥ 0.0 | 开始消耗时记录生还者路程，超过该值加上这个数后不允许消耗 | 服务器端本插件逻辑 | `ai_tank_2.sp:134` |
| `ai_TankConsumePosRaidus` | `100` | 整数，源码写作浮点 | ≥ 0.0 | 坦克走出消耗位中心该半径的圆范围后强制重新进入 | 服务器端本插件逻辑 | `ai_tank_2.sp:135` |
| `ai_TankConsumeHealth` | `2000` | 整数，源码写作浮点 | ≥ 0.0 | 坦克血量少于该值时强制压制 | 服务器端本插件逻辑 | `ai_tank_2.sp:136` |
| `ai_TankVomitAttackNum` | `1` | 整数，源码写作浮点 | ≥ 0.0 | 有该数量的生还者被喷吐时正在消耗的坦克强制压制 | 服务器端本插件逻辑 | `ai_tank_2.sp:137` |
| `ai_TankConsumeIncapNum` | `1` | 整数，源码写作浮点 | ≥ 0.0 | 坦克强制压制时若令该数量的生还者倒地则允许时继续消耗 | 服务器端本插件逻辑 | `ai_tank_2.sp:138` |
| `ai_TankAirAngleRestrict` | `57` | 整数，源码写作浮点 | 0.0 ~ 90.0 | 坦克当前速度与到目标向量夹角大于该角度即停止连跳，范围 0~90 | 服务器端本插件逻辑 | `ai_tank_2.sp:139` |
| `ai_TankConsumeRockInterval` | `4` | 整数，源码写作浮点 | ≥ 0.0 | 坦克在消耗位上每多少秒扔一次石头 | 服务器端本插件逻辑 | `ai_tank_2.sp:140` |
| `ai_TankTarget` | `0` | 整数，源码写作浮点 | 0.0 ~ 3.0 | 坦克目标选择：0 自然选择，1 最近，2 血量最低，3 血量最高 | 服务器端本插件逻辑 | `ai_tank_2.sp:142` |
| `ai_TankTreeDetect` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否开启防止绕树功能 | 服务器端本插件逻辑 | `ai_tank_2.sp:143` |
| `ai_TankAntiTreeMethod` | `1` | 整数，源码写作浮点 | 1.0 ~ 2.0 | 防止绕树的方法：1 选择新目标，2 传送到绕树生还者位置 | 服务器端本插件逻辑 | `ai_tank_2.sp:144` |
| `ai_TankThow` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否允许坦克丢石头 | 服务器端本插件逻辑 | `ai_tank_2.sp:146` |
| `ai_TankThrowRange` | `250,500` | 字符串或表达式 | 无上下界 | 允许坦克丢石头的范围，逗号分隔且不能有空格 | 服务器端本插件逻辑 | `ai_tank_2.sp:147` |

## Ai_Tank_Enhance —— `optional/AnneHappy/ai_tank_new.sp`

- 源文件：`optional/AnneHappy/ai_tank_new.sp`
- myinfo：version=2022-5-02，author=Breezy，High Cookie，Standalone，Newteee，cravenge，Harry，Sorallll，PaimonQwQ，夜羽真白,东
- myinfo description（源码原文）：觉得Ai克太弱了？ Try this！

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_con` | 无参数 | 需要 ADMFLAG_BAN（d，封禁） | 让 AI Tank 立即执行消耗行为，测试用 | `ai_tank_new.sp:178` |

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `ai_Tank_BhopSpeed` | `60.0` | 整数，源码写作浮点 | ≥ 0.0 | Tank 连跳的速度 | 服务器端本插件逻辑 | `ai_tank_new.sp:94` |
| `ai_Tank_StopDistance` | `130` | 整数，源码写作浮点 | ≥ 0.0 | Tank 距目标多远时停下来 | 服务器端本插件逻辑 | `ai_tank_new.sp:95` |
| `ai_Tank_Bhop` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否开启 Tank 连跳：0 关，1 开 | 服务器端本插件逻辑 | `ai_tank_new.sp:96` |
| `ai_Tank_Throw` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否允许 Tank 投掷石块：0 关，1 开 | 服务器端本插件逻辑 | `ai_tank_new.sp:97` |
| `ai_TankThrowDistance` | `450` | 整数，源码写作浮点 | ≥ 0.0 | Tank 距目标多近允许投掷石块 | 服务器端本插件逻辑 | `ai_tank_new.sp:98` |
| `ai_TankBlockThrowDistance` | `200` | 整数，源码写作浮点 | ≥ 0.0 | Tank 距目标多近时阻止投掷石块 | 服务器端本插件逻辑 | `ai_tank_new.sp:99` |
| `ai_TankTarget` | `1` | 整数，源码写作浮点 | 1.0 ~ 3.0 | Tank 目标选择：1 最近，2 血量最少，3 血量最多 | 服务器端本插件逻辑 | `ai_tank_new.sp:100` |
| `ai_TankTreeDetect` | `1` | 整数，源码写作浮点 | 0.0 ~ 2.0 | 生还者与 Tank 绕柱时的操作：0 关闭，1 切换目标，2 把 Tank 传送到绕树的生还者后 | 服务器端本插件逻辑 | `ai_tank_new.sp:101` |
| `ai_TankTreeNewTargetDistance` | `300` | 整数，源码写作浮点 | ≥ 0.0 | 记录绕树生还者并选择新目标后，距新目标多近重置绕树记录 | 服务器端本插件逻辑 | `ai_tank_new.sp:102` |
| `ai_TankAirAngles` | `60.0` | 整数，源码写作浮点 | 0.0 ~ 90.0 | 空中速度向量与到生还者方向向量夹角大于该值即停止连跳，范围 0~90 | 服务器端本插件逻辑 | `ai_tank_new.sp:103` |
| `ai_TankConsume` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否开启 Tank 消耗功能 | 服务器端本插件逻辑 | `ai_tank_new.sp:104` |
| `ai_TankConsumeHeight` | `100` | 整数，源码写作浮点 | ≥ 0.0 | 消耗时优先选择高于该高度的位置，无则随机选位 | 服务器端本插件逻辑 | `ai_tank_new.sp:105` |
| `ai_TankConsumeLimit` | `0` | 整数，源码写作浮点 | ≥ 0.0 | 感染者团队的特感数小于等于当前刷特数量减去该值时触发，源码描述不完整，未说明具体动作 | 服务器端本插件逻辑 | `ai_tank_new.sp:106` |
| `ai_TankConsumeRaidus` | `80.0` | 整数，源码写作浮点 | ≥ 0.0 | Tank 消耗位置的范围半径，以中心坐标画圆 | 服务器端本插件逻辑 | `ai_tank_new.sp:107` |
| `ai_TankAttackVomitedNum` | `1` | 整数，源码写作浮点 | ≥ 0.0 | 若有该数量的生还者被 Boomer 喷吐到，正在消耗的坦克将发动攻击 | 服务器端本插件逻辑 | `ai_tank_new.sp:108` |
| `ai_TankVomitCanInstantAttack` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否开启固定数量生还者被喷吐后 Tank 立刻攻击 | 服务器端本插件逻辑 | `ai_tank_new.sp:109` |
| `ai_TankVomitAttackInterval` | `20.0` | 整数，源码写作浮点 | ≥ 0.0 | 从开始被喷且 Tank 允许攻击时起，该时间内 Tank 允许攻击 | 服务器端本插件逻辑 | `ai_tank_new.sp:110` |
| `ai_TankTeleportForwardPercent` | `10` | 整数，源码写作浮点 | ≥ 0.0 | 开始消耗时记录生还者行进距离 x，前压超过 x 加该值时 Tank 传送到生还者处压制 | 服务器端本插件逻辑 | `ai_tank_new.sp:111` |
| `ai_TankConsumeLimitNum` | `5` | 整数，源码写作浮点 | ≥ 0.0 | Tank 最多进行消耗的次数 | 服务器端本插件逻辑 | `ai_tank_new.sp:112` |
| `ai_TankConsumeType` | `6` | 整数，源码写作浮点 | 1.0 ~ 8.0 | Tank 按哪种特感类型找消耗位：1 Smoker，2 Boomer，3 Hunter，4 Spitter，5 Jockey，6 Charger，8 Tank | 服务器端本插件逻辑 | `ai_tank_new.sp:113` |
| `ai_TankRetreatAirAngles` | `75.0` | 整数，源码写作浮点 | ≥ 0.0 | Tank 在回避连跳过程中视角与速度夹角超过该值即停止连跳 | 服务器端本插件逻辑 | `ai_tank_new.sp:114` |
| `ai_TankConsumeAction` | `2` | 整数，源码写作浮点 | 1.0 ~ 2.0 | Tank 在消耗范围内的行为：1 冰冻，2 可活动但不允许超出消耗范围 | 服务器端本插件逻辑 | `ai_tank_new.sp:115` |
| `ai_TankConsumeDamagePercent` | `50` | 整数，源码写作浮点 | 0.0 ~ 100.0 | Tank 在消耗过程中只受到该百分比的伤害，范围 0~100 | 服务器端本插件逻辑 | `ai_tank_new.sp:116` |
| `ai_TankForceAttackDistance` | `300` | 整数，源码写作浮点 | ≥ 0.0 | Tank 离最近生还者该距离时即使可消耗也会强制压制 | 服务器端本插件逻辑 | `ai_tank_new.sp:118` |
| `ai_TankIncappedCount` | `1` | 整数，源码写作浮点 | ≥ 0.0 | 强制压制时需拍倒该数量的生还者才允许继续检测消耗 | 服务器端本插件逻辑 | `ai_tank_new.sp:119` |
| `ai_TankConsumeHealthLimit` | `1200` | 整数，源码写作浮点 | ≥ 0.0 | Tank 血量少于该值时不会消耗 | 服务器端本插件逻辑 | `ai_tank_new.sp:120` |
| `ai_TankConsumeValidRaidus` | `1800` | 整数，源码写作浮点 | ≥ 0.0 | 当前消耗位不能直视生还时以该半径重新找位 | 服务器端本插件逻辑 | `ai_tank_new.sp:121` |
| `ai_TankConsumeDistance` | `1200` | 整数，源码写作浮点 | ≥ 0.0 | Tank 消耗找位的位置必须离生还者大于该距离 | 服务器端本插件逻辑 | `ai_tank_new.sp:122` |

## anne_cvar_shield.sp（myinfo 缺 name，用文件名代替） —— `optional/AnneHappy/anne_cvar_shield.sp`

- 源文件：`optional/AnneHappy/anne_cvar_shield.sp`
- myinfo：author=morzlee / Codex
- myinfo description（源码原文）：Keeps Anne vote-selected SI limit cvars authoritative.

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `anne_cvar_shield_capture` | 无参数 | 服务器控制台命令，玩家无法使用 | 把当前 Anne 特感上限 cvar 捕获为保护目标，服务器控制台命令 | `anne_cvar_shield.sp:137` |

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `anne_cvar_shield_enable` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用 Anne 特感上限 cvar 保护 | 服务器端本插件逻辑 | `anne_cvar_shield.sp:117` |
| `anne_cvar_shield_debug` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否记录 Anne CVar Shield 的操作日志 | 服务器端本插件逻辑 | `anne_cvar_shield.sp:122` |
| `anne_cvar_shield_sync_versus_limits` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否为每个特感职业同步 z_*_limit、z_versus_*_limit 与 infected_control 的 inf_*_limit 目标 | 服务器端本插件逻辑 | `anne_cvar_shield.sp:127` |

## Anne NavGraph Export —— `optional/AnneHappy/anne_navgraph_export.sp`

- 源文件：`optional/AnneHappy/anne_navgraph_export.sp`
- myinfo：author=morzlee & Anne
- myinfo description（源码原文）：导出当前地图 Nav 图 JSON 供刷特可视化页使用

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_navgraph_export` | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 把当前地图 Nav 图导出为 JSON 到 data/anne_navgraph/，请在非对局时执行 | `anne_navgraph_export.sp:46` |

### ConVar

（本插件未注册 ConVar）

## Anne Pill Hint —— `optional/AnneHappy/anne_pill_hint.sp`

- 源文件：`optional/AnneHappy/anne_pill_hint.sp`
- myinfo：author=morzlee
- myinfo description（源码原文）：When survivors leave the saferoom, tells them at what map progress each pain pill lies

### 指令

（本插件未注册指令）

### ConVar

（本插件未注册 ConVar）

## Anne Traitor Quota —— `optional/AnneHappy/anne_traitor_quota.sp`

- 源文件：`optional/AnneHappy/anne_traitor_quota.sp`
- myinfo：version=1.0.0，author=AnneHappy
- myinfo description（源码原文）：Optional MySQL-backed daily quota and Tank block provider for infected_control traitor mode

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `inf_traitor_daily_quota` | `100` | 整数，源码写作浮点 | 0.0 ~ 10000.0 | 管理员的每日内鬼基础配额，sb_admins.expires 每剩满一年加 50 | 服务器端本插件逻辑 | `anne_traitor_quota.sp:23` |
| `inf_traitor_public_daily_quota` | `20` | 整数，源码写作浮点 | 0.0 ~ 10000.0 | 合格非管理员玩家每日可用的内鬼配额 | 服务器端本插件逻辑 | `anne_traitor_quota.sp:27` |
| `inf_traitor_quota_db` | `DEFAULT_TRAITOR_QUOTA_DB_CONFIG` | 字符串或表达式 | 无上下界 | 共享内鬼每日配额存储所用的 MySQL databases.cfg 区块，默认使用 l4d_stats | 服务器端本插件逻辑 | `anne_traitor_quota.sp:31` |
| `inf_traitor_quota_table` | `infected_control_traitor_quota` | 字符串或表达式 | 无上下界 | 内鬼每日配额存储的 SQL 表名 | 服务器端本插件逻辑 | `anne_traitor_quota.sp:35` |

## AnneHappy Dynamic AI Difficulty —— `optional/AnneHappy/annehappy_dynamic_ai_difficulty.sp`

- 源文件：`optional/AnneHappy/annehappy_dynamic_ai_difficulty.sp`
- myinfo：author=morzlee + ChatGPT
- myinfo description（源码原文）：根据生还者积分/游玩时间(PPM)动态调整 AnneHappy 特感难度

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_aippm` | 无参数 | 无（任意玩家可用） | 显示当前 AnneHappy 动态难度与 PPM | `annehappy_dynamic_ai_difficulty.sp:124` |
| `sm_aidiff` | <0 至 6> | 需要 ADMFLAG_CONFIG（h，服务器配置） | 设置动态难度：0 自动，1 简单，2 普通，3 困难，4 专家，5 极限，6 音理 | `annehappy_dynamic_ai_difficulty.sp:125` |
| `sm_aidiff_reload` | 无参数 | 需要 ADMFLAG_CONFIG（h，服务器配置） | 重新读取难度配置并应用当前难度 | `annehappy_dynamic_ai_difficulty.sp:126` |

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `ah_ai_dynamic_enable` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用 AnneHappy 动态难度 | 服务器端本插件逻辑 | `annehappy_dynamic_ai_difficulty.sp:94` |
| `ah_ai_dynamic_check_interval` | `5.0` | 整数，源码写作浮点 | 1.0 ~ 60.0 | 回合定档前每隔多少秒从 l4d_stats 重试检查一次平均 PPM，范围 1~60 | 服务器端本插件逻辑 | `annehappy_dynamic_ai_difficulty.sp:95` |
| `ah_ai_dynamic_ppm_normal` | `30.89` | 浮点 | ≥ 0.0 | 进入普通难度所需的 l4d_stats 平均 PPM 阈值 | 服务器端本插件逻辑 | `annehappy_dynamic_ai_difficulty.sp:96` |
| `ah_ai_dynamic_ppm_hard` | `43.23` | 浮点 | ≥ 0.0 | 进入困难难度所需的 l4d_stats 平均 PPM 阈值 | 服务器端本插件逻辑 | `annehappy_dynamic_ai_difficulty.sp:97` |
| `ah_ai_dynamic_ppm_expert` | `63.70` | 浮点 | ≥ 0.0 | 进入专家难度所需的 l4d_stats 平均 PPM 阈值 | 服务器端本插件逻辑 | `annehappy_dynamic_ai_difficulty.sp:98` |
| `ah_ai_dynamic_ppm_extreme` | `77.57` | 浮点 | ≥ 0.0 | 进入极限难度所需的 l4d_stats 平均 PPM 阈值 | 服务器端本插件逻辑 | `annehappy_dynamic_ai_difficulty.sp:99` |
| `ah_ai_dynamic_fixed_level` | `0` | 整数，源码写作浮点 | 0.0 ~ 6.0 | 固定动态难度：0 自动，1 简单，2 普通，3 困难，4 专家，5 极限，6 音理 | 服务器端本插件逻辑 | `annehappy_dynamic_ai_difficulty.sp:100` |
| `ah_ai_dynamic_config` | `DEFAULT_CONFIG_PATH` | 字符串或表达式 | 无上下界 | 难度配置文件路径，相对 addons/sourcemod | 服务器端本插件逻辑 | `annehappy_dynamic_ai_difficulty.sp:101` |
| `ah_ai_dynamic_use_quarter_stats` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否优先使用季度积分与季度时间计算玩家 PPM，当前季度数据失真时应关闭 | 服务器端本插件逻辑 | `annehappy_dynamic_ai_difficulty.sp:102` |
| `ah_ai_dynamic_quarter_min_minutes` | `300` | 整数，源码写作浮点 | ≥ 0.0 | 玩家本季度样本低于该分钟数时回退使用总积分 PPM | 服务器端本插件逻辑 | `annehappy_dynamic_ai_difficulty.sp:103` |
| `ah_ai_dynamic_threshold_mode` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | PPM 阈值来源：0 使用本 cfg 固定阈值，1 从数据库读取每日分位阈值 | 服务器端本插件逻辑 | `annehappy_dynamic_ai_difficulty.sp:104` |
| `ah_ai_dynamic_threshold_db_config` | `DEFAULT_THRESHOLD_DB_CONFIG` | 字符串或表达式 | 无上下界 | 每日 PPM 阈值数据库配置名，对应 databases.cfg | 服务器端本插件逻辑 | `annehappy_dynamic_ai_difficulty.sp:105` |
| `ah_ai_dynamic_threshold_table` | `DEFAULT_THRESHOLD_TABLE` | 字符串或表达式 | 无上下界 | 每日 PPM 阈值表名，只允许字母数字下划线 | 服务器端本插件逻辑 | `annehappy_dynamic_ai_difficulty.sp:106` |
| `ah_ai_dynamic_threshold_max_age` | `172800` | 整数，源码写作浮点 | ≥ 0.0 | 数据库阈值的最大有效秒数，0 不检查过期，默认 2 天 | 服务器端本插件逻辑 | `annehappy_dynamic_ai_difficulty.sp:107` |
| `ah_ai_dynamic_enforce_interval` | `0.0` | 整数，源码写作浮点 | 0.0 ~ 60.0 | 难度锁定后每隔多少秒重刷当前档位 cvar，0 关闭，范围 0~60 | 服务器端本插件逻辑 | `annehappy_dynamic_ai_difficulty.sp:108` |
| `ah_ai_dynamic_tank_bhop_override` | `-1` | 整数，源码写作浮点 | -1.0 ~ 1.0 | Tank 连跳覆盖：-1 跟随档位配置，0 强制关闭，1 强制开启 | 服务器端本插件逻辑 | `annehappy_dynamic_ai_difficulty.sp:109` |
| `ah_ai_dynamic_survivor_max_incaps` | `2` | 整数，源码写作浮点 | -1.0 ~ 10.0 | 动态难度应用时强制恢复的生还者最大倒地次数，-1 不处理 | 服务器端本插件逻辑 | `annehappy_dynamic_ai_difficulty.sp:110` |
| `ah_ai_dynamic_announce` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 调档时是否在聊天框提示 | 服务器端本插件逻辑 | `annehappy_dynamic_ai_difficulty.sp:111` |
| `ah_ai_dynamic_debug` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否输出动态难度调试日志 | 服务器端本插件逻辑 | `annehappy_dynamic_ai_difficulty.sp:112` |
| `ah_ai_dynamic_current_level` | `0` | 整数，源码写作浮点 | 0.0 ~ 6.0 | 当前回合动态难度：0 未定档，1 简单，2 普通，3 困难，4 专家，5 极限，6 音理 | 服务器端本插件逻辑；不写入 cfg 存档 | `annehappy_dynamic_ai_difficulty.sp:113` |
| `ah_ai_dynamic_current_mode` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 当前回合动态难度来源：0 自动，1 固定 | 服务器端本插件逻辑；不写入 cfg 存档 | `annehappy_dynamic_ai_difficulty.sp:114` |
| `ah_ai_dynamic_current_ppm` | `0.0` | 整数，源码写作浮点 | ≥ 0.0 | 当前回合自动定档使用的平均个人 PPM，固定模式为 0 | 服务器端本插件逻辑；不写入 cfg 存档 | `annehappy_dynamic_ai_difficulty.sp:115` |
| `ah_ai_dynamic_current_locked` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 当前回合动态难度是否已锁定 | 服务器端本插件逻辑；不写入 cfg 存档 | `annehappy_dynamic_ai_difficulty.sp:116` |
| `ah_ai_dynamic_version` | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | 服务器端本插件逻辑；不写入 cfg 存档 | `annehappy_dynamic_ai_difficulty.sp:117` |

## coop_round_delay.sp（myinfo 缺 name，用文件名代替） —— `optional/AnneHappy/coop_round_delay.sp`

- 源文件：`optional/AnneHappy/coop_round_delay.sp`
- myinfo：源码中未找到 myinfo 块

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `coop_round_restart_delay_version` | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | 服务器端本插件逻辑；不写入 cfg 存档 | `coop_round_delay.sp:29` |
| `coop_round_restart_delay` | `2.0` | 整数，源码写作浮点 | ≥ 0.0 | 战役模式回合重开延迟时间，最小 0 | 服务器端本插件逻辑 | `coop_round_delay.sp:31` |

## Direct InfectedSpawn (directed-nav + maxdist-fallback) —— `optional/AnneHappy/infected_control.sp`

- 源文件：`optional/AnneHappy/infected_control.sp`
- myinfo：version=2026-09-04.1，author=东, Caibiii, 夜羽真白, Paimon-Kawaii, fdxx (inspiration)
- myinfo description（源码原文）：特感刷新控制 / 传送 / 跑男 / 有向Nav候选 + 当前帧安全精判 + 最大距离兜底

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_startspawn` | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 管理员重置刷特时钟 | `infected_control.sp:511` |
| `sm_stopspawn` | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 管理员停止刷特 | `infected_control.sp:512` |
| `sm_rebuildnavcache` | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 重建当前地图的 Nav 图与进度缓存 | `infected_control.sp:513` |
| `sm_navpeek` | 无参数 | 需要 ADMFLAG_GENERIC（b，通用管理员） | 查看准星 Nav 的分桶与属性 | `infected_control.sp:514` |
| `sm_np` | 无参数 | 需要 ADMFLAG_GENERIC（b，通用管理员） | sm_navpeek 的别名 | `infected_control.sp:515` |
| `sm_navtest` | 无参数 | 需要 ADMFLAG_GENERIC（b，通用管理员） | 测试准星 Nav 能否生成特感及评分 | `infected_control.sp:516` |
| `sm_nt` | 无参数 | 需要 ADMFLAG_GENERIC（b，通用管理员） | sm_navtest 的别名 | `infected_control.sp:517` |
| `sm_wavestatus` | 无参数 | 需要 ADMFLAG_GENERIC（b，通用管理员） | 查看当前波决策器状态 | `infected_control.sp:518` |
| `sm_neigui` | [特感职业] | 无（任意玩家可用） | 进入内鬼刷特队列 | `infected_control.sp:519` |
| `sm_it` | [特感职业] | 无（任意玩家可用） | sm_neigui 的别名 | `infected_control.sp:520` |
| `sm_neiguicancel` | 无参数 | 无（任意玩家可用） | 取消内鬼刷特队列 | `infected_control.sp:521` |
| `sm_itcancel` | 无参数 | 无（任意玩家可用） | sm_neiguicancel 的别名 | `infected_control.sp:522` |

### ConVar

（本插件未注册 ConVar）

## Direct InfectedSpawn (fdxx-nav + buckets + maxdist-fallback) —— `optional/AnneHappy/infected_control26-07.sp`

- 源文件：`optional/AnneHappy/infected_control26-07.sp`
- myinfo：version=2026-07-20，author=东, Caibiii, 夜羽真白, Paimon-Kawaii, fdxx (inspiration)
- myinfo description（源码原文）：特感刷新控制 / 传送 / 跑男 / fdxx NavArea选点 + 进度分桶 + 最大距离兜底

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_startspawn` | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 管理员重置刷特时钟 | `infected_control26-07.sp:317` |
| `sm_stopspawn` | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 管理员停止刷特 | `infected_control26-07.sp:318` |
| `sm_rebuildnavcache` | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 强制重建当前地图的 Nav 分桶缓存 | `infected_control26-07.sp:319` |
| `sm_navpeek` | 无参数 | 需要 ADMFLAG_GENERIC（b，通用管理员） | 查看准星 Nav 的分桶与属性 | `infected_control26-07.sp:320` |
| `sm_np` | 无参数 | 需要 ADMFLAG_GENERIC（b，通用管理员） | sm_navpeek 的别名 | `infected_control26-07.sp:321` |
| `sm_navtest` | 无参数 | 需要 ADMFLAG_GENERIC（b，通用管理员） | 测试准星 Nav 能否生成特感及评分 | `infected_control26-07.sp:322` |
| `sm_nt` | 无参数 | 需要 ADMFLAG_GENERIC（b，通用管理员） | sm_navtest 的别名 | `infected_control26-07.sp:323` |
| `sm_wavestatus` | 无参数 | 需要 ADMFLAG_GENERIC（b，通用管理员） | 查看当前波决策器状态 | `infected_control26-07.sp:324` |
| `sm_neigui` | [特感职业] | 无（任意玩家可用） | 进入内鬼刷特队列 | `infected_control26-07.sp:325` |
| `sm_it` | [特感职业] | 无（任意玩家可用） | sm_neigui 的别名 | `infected_control26-07.sp:326` |
| `sm_neiguicancel` | 无参数 | 无（任意玩家可用） | 取消内鬼刷特队列 | `infected_control26-07.sp:327` |

### ConVar

（本插件未注册 ConVar）

## Anne Stuck Tank Teleport System (ASTT) —— `optional/AnneHappy/l4d2_Anne_stuck_tank_teleport.sp`

- 源文件：`optional/AnneHappy/l4d2_Anne_stuck_tank_teleport.sp`
- myinfo：author=东, re-arch by ChatGPT
- myinfo description（源码原文）：Tank卡住/跑男：按门口阈值决定传门口或传近点（均排除 CHECKPOINT）
- HookConVarChange：`g_hEnable`→`CvarChanged`、`g_hBlockCheckpoint`→`CvarChanged`、`g_hDoorThreshold`→`CvarChanged`、`g_hDebug`→`CvarChanged`

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `l4d2_Anne_stuck_tank_teleport` | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | 服务器端本插件逻辑；不写入 cfg 存档 | `l4d2_Anne_stuck_tank_teleport.sp:104` |
| `l4d2_astt_enable` | `1` | 整数 | 无上下界 | 是否启用插件：1 开，0 关 | 服务器端本插件逻辑 | `l4d2_Anne_stuck_tank_teleport.sp:106` |
| `l4d2_astt_stuck_check_interval` | `3` | 整数 | 无上下界 | Tank 卡死检测间隔秒数 | 服务器端本插件逻辑 | `l4d2_Anne_stuck_tank_teleport.sp:107` |
| `l4d2_astt_non_stuck_radius` | `20` | 整数 | 无上下界 | 间隔内移动距离小于该值即判定为卡死 | 服务器端本插件逻辑 | `l4d2_Anne_stuck_tank_teleport.sp:108` |
| `l4d2_astt_rusher_punish` | `1` | 整数 | 无上下界 | 是否通过传送坦克惩罚跑男：1 是，0 否 | 服务器端本插件逻辑 | `l4d2_Anne_stuck_tank_teleport.sp:110` |
| `l4d2_astt_rusher_dist` | `2800` | 整数 | 无上下界 | 距最近坦克小于该距离视为跑男 | 服务器端本插件逻辑 | `l4d2_Anne_stuck_tank_teleport.sp:111` |
| `l4d2_astt_rusher_checks` | `6` | 整数 | 无上下界 | 确认跑男前的检测次数 | 服务器端本插件逻辑 | `l4d2_Anne_stuck_tank_teleport.sp:112` |
| `l4d2_astt_rusher_interval` | `3` | 整数 | 无上下界 | 跑男检测间隔秒数 | 服务器端本插件逻辑 | `l4d2_Anne_stuck_tank_teleport.sp:113` |
| `l4d2_astt_rusher_minplayers` | `2` | 整数 | 无上下界 | 启用跑男规则所需的最少存活生还者数 | 服务器端本插件逻辑 | `l4d2_Anne_stuck_tank_teleport.sp:114` |
| `l4d2_astt_block_checkpoint` | `1` | 整数 | 无上下界 | 是否禁止传送到 CHECKPOINT 导航区域：1 是，0 否 | 服务器端本插件逻辑 | `l4d2_Anne_stuck_tank_teleport.sp:116` |
| `l4d2_astt_door_threshold` | `1000` | 整数 | 无上下界 | 生还者与安全室距离小于等于该值时传送到 DOOR，否则传送到 NEAR，单位为地图单位 | 服务器端本插件逻辑 | `l4d2_Anne_stuck_tank_teleport.sp:117` |
| `l4d2_astt_debug` | `0` | 整数 | 无上下界 | 是否输出调试信息：1 开，0 关 | 服务器端本插件逻辑 | `l4d2_Anne_stuck_tank_teleport.sp:118` |

## [L4D2] Infected Ladder Speed Boost —— `optional/AnneHappy/l4d2_ai_ladder_boost.sp`

- 源文件：`optional/AnneHappy/l4d2_ai_ladder_boost.sp`
- myinfo：author=YourName, AiMee, AnneHappy
- myinfo description（源码原文）：AI infected ladder booster with visibility-gated boost and climb animation controls

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_ladder_debug` | 无参数 | 需要 ADMFLAG_GENERIC（b，通用管理员） | 在控制台输出爬梯加速调试信息 | `l4d2_ai_ladder_boost.sp:109` |
| `sm_ladder_status` | 无参数 | 需要 ADMFLAG_GENERIC（b，通用管理员） | 显示当前获得爬梯加速的特感数量 | `l4d2_ai_ladder_boost.sp:110` |

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `l4d2_ladder_boost_enabled` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用未被生还者看见时 AI 特感爬梯加速：1 开，0 关 | 服务器端本插件逻辑 | `l4d2_ai_ladder_boost.sp:87` |
| `l4d2_ladder_boost_multiplier` | `10.0` | 整数，源码写作浮点 | 1.0 ~ 20.0 | 未被看见时的爬梯速度倍数，范围 1~20 | 服务器端本插件逻辑 | `l4d2_ai_ladder_boost.sp:88` |
| `l4d2_ladder_boost_detection` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 检测方法：0 威胁感知加射线，1 传统 FOV 加射线 | 服务器端本插件逻辑 | `l4d2_ai_ladder_boost.sp:89` |
| `l4d2_ladder_boost_cooldown` | `3.0` | 整数，源码写作浮点 | 1.0 ~ 10.0 | 被看见后禁用未视野加速的冷却秒数，范围 1~10 | 服务器端本插件逻辑 | `l4d2_ai_ladder_boost.sp:90` |
| `l4d2_ladder_boost_use_sdkhook` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 兼容旧配置：该项已停用，AI 特感固定使用 Timer 检测 | 服务器端本插件逻辑 | `l4d2_ai_ladder_boost.sp:91` |
| `l4d2_ladder_boost_debug` | `0` | 整数，源码写作浮点 | 0.0 ~ 2.0 | 调试模式：0 关闭，1 基本调试，2 详细调试 | 服务器端本插件逻辑 | `l4d2_ai_ladder_boost.sp:92` |
| `l4d2_ai_ladder_boost` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 兼容旧 l4d2_si_ladder_booster：AI 特感在梯子上固定加速 | 服务器端本插件逻辑；仅服务器 | `l4d2_ai_ladder_boost.sp:94` |
| `l4d2_pz_ladder_boost` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 兼容旧 l4d2_si_ladder_booster：已停用，真人特感不会获得加速 | 服务器端本插件逻辑；仅服务器 | `l4d2_ai_ladder_boost.sp:95` |
| `l4d2_boost_multiplier` | `3.2` | 浮点 | 1.0 ~ 10.0 | 兼容旧 l4d2_si_ladder_booster：固定爬梯加速倍数，范围 1~10 | 服务器端本插件逻辑；仅服务器 | `l4d2_ai_ladder_boost.sp:96` |
| `l4d2_ladder_boost_tank` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否允许该通用插件加速 Tank 爬梯 | 服务器端本插件逻辑 | `l4d2_ai_ladder_boost.sp:97` |
| `l4d2_climb_anim_boost` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否由该插件处理 AI Tank 翻越或爬小障碍的动画加速 | 服务器端本插件逻辑 | `l4d2_ai_ladder_boost.sp:98` |
| `l4d2_si_climb_anim_rate` | `3.2` | 浮点 | ≥ 0.0 | 兼容旧配置：普通 AI 特感仅做梯子移动加速，不再修改翻越动画速度 | 服务器端本插件逻辑 | `l4d2_ai_ladder_boost.sp:99` |
| `l4d2_tank_climb_anim_rate` | `3.5` | 浮点 | ≥ 0.0 | Tank 高翻越动画播放倍速，1.0 为原速 | 服务器端本插件逻辑 | `l4d2_ai_ladder_boost.sp:100` |
| `l4d2_tank_low_climb_anim_rate` | `2.5` | 浮点 | ≥ 0.0 | Tank 低翻越动画播放倍速，1.0 为原速 | 服务器端本插件逻辑 | `l4d2_ai_ladder_boost.sp:101` |
| `l4d2_tank_ladder_anim_rate` | `1.0` | 整数，源码写作浮点 | ≥ 0.0 | Tank 梯子动画播放倍速，真实爬梯速度由 m_flLaggedMovementValue 控制 | 服务器端本插件逻辑 | `l4d2_ai_ladder_boost.sp:102` |
| `l4d2_ladder_boost_clamp_exit_speed` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 特感离开梯子时是否把水平速度限制到当前走路速度，防止 10 倍速度带出 | 服务器端本插件逻辑 | `l4d2_ai_ladder_boost.sp:103` |

## Anne Thirdperson Shoulder Fix —— `optional/AnneHappy/l4d2_anne_thirdperson_fix.sp`

- 源文件：`optional/AnneHappy/l4d2_anne_thirdperson_fix.sp`
- myinfo：author=morzlee, OpenAI
- myinfo description（源码原文）：Enables !tp by spoofing mp_gamemode before client thirdpersonshoulder.
- HookConVarChange：`g_cvEnabled`→`OnControlCvarChanged`、`g_cvCommands`→`OnControlCvarChanged`、`g_cvSpoofGameMode`→`OnControlCvarChanged`、`g_cvFakeGameMode`→`OnControlCvarChanged`、`g_cvCfgNames`→`OnControlCvarChanged`、`g_cvMPGameMode`→`OnControlCvarChanged`、`g_cvReadyCfgName`→`OnControlCvarChanged`、`g_cvReadyCfgName`→`OnControlCvarChanged`

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_tp` | 无参数 | 无（任意玩家可用） | 切换自己的 Anne 第三人称肩部视角 | `l4d2_anne_thirdperson_fix.sp:67` |
| `sm_third` | 无参数 | 无（任意玩家可用） | sm_tp 的别名 | `l4d2_anne_thirdperson_fix.sp:68` |
| `sm_thirdperson` | 无参数 | 无（任意玩家可用） | sm_tp 的别名 | `l4d2_anne_thirdperson_fix.sp:69` |
| `sm_3rd` | 无参数 | 无（任意玩家可用） | sm_tp 的别名 | `l4d2_anne_thirdperson_fix.sp:70` |
| `sm_3rdon` | 无参数 | 无（任意玩家可用） | 启用 Anne 第三人称肩部视角 | `l4d2_anne_thirdperson_fix.sp:71` |
| `sm_3rdoff` | 无参数 | 无（任意玩家可用） | 关闭 Anne 第三人称肩部视角 | `l4d2_anne_thirdperson_fix.sp:72` |
| `sm_anne_thirdperson_status` | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 显示 Anne 第三人称修复的状态 | `l4d2_anne_thirdperson_fix.sp:73` |

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `l4d2_anne_thirdperson_fix_version` | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | 服务器端本插件逻辑；不写入 cfg 存档 | `l4d2_anne_thirdperson_fix.sp:33` |
| `l4d2_anne_thirdperson_fix_enabled` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 0 关闭，1 在 l4d_ready_cfg_name 匹配时启用 Anne 第三人称修复 | 服务器端本插件逻辑 | `l4d2_anne_thirdperson_fix.sp:34` |
| `l4d2_anne_thirdperson_fix_commands` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 0 关闭，1 启用 !tp 与 !third 命令 | 服务器端本插件逻辑 | `l4d2_anne_thirdperson_fix.sp:35` |
| `l4d2_anne_thirdperson_fix_spoof_gamemode` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 0 关闭，1 为 !tp 在启用 thirdpersonshoulder 前伪造 mp_gamemode | 服务器端本插件逻辑 | `l4d2_anne_thirdperson_fix.sp:36` |
| `l4d2_anne_thirdperson_fix_fake_gamemode` | `coop` | 字符串或表达式 | 无上下界 | 仅发送给 !tp 客户端的 mp_gamemode 值 | 服务器端本插件逻辑 | `l4d2_anne_thirdperson_fix.sp:37` |
| `l4d2_anne_thirdperson_fix_cfg_names` | `DEFAULT_CFG_NAMES` | 字符串或表达式 | 无上下界 | 逗号分隔的 l4d_ready_cfg_name 片段，匹配时启用该修复，留空为全部配置 | 服务器端本插件逻辑 | `l4d2_anne_thirdperson_fix.sp:38` |
| `l4d2_anne_thirdperson_fix_debug` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 0 关闭，1 记录客户端伪造与恢复操作 | 服务器端本插件逻辑 | `l4d2_anne_thirdperson_fix.sp:39` |

## l4d2_dirspawn.sp（myinfo 缺 name，用文件名代替） —— `optional/AnneHappy/l4d2_dirspawn.sp`

- 源文件：`optional/AnneHappy/l4d2_dirspawn.sp`
- myinfo：（无 version/author）
- myinfo description（源码原文）：控制特感总数/间隔/每类上限（KV） + M4修复 + 人数自适应(仅改总数) + MaxSpecial解锁 + 间隔联动节奏 + 三方图脚本兜底
- HookConVarChange：`gCvarCount`→`CvarChanged`、`gCvarInterval`→`CvarChanged`、`gCvarDomLimit`→`CvarChanged`、`gCvarKvEnable`→`CvarChanged`、`gCvarKvPath`→`CvarChanged`、`gCvarLimitStyle`→`CvarChanged`、`gCvarDpsSiLimit`→`CvarChanged`、`gCvarAllowSIWithTank`→`CvarChanged`、`gCvarRelaxEnable`→`CvarChanged`、`gCvarRelaxMin`→`CvarChanged`、`gCvarRelaxMax`→`CvarChanged`、`gCvarLockTempo`→`CvarChanged`、`gCvarInitialMin`→`CvarChanged`、`gCvarInitialMax`→`CvarChanged`、`gCvarRelaxOffBattlefieldRespawn`→`CvarChanged`、`gCvarRelaxOffInitialDelayMax`→`CvarChanged`、`gCvarRelaxOffInitialDelayMaxExtra`→`CvarChanged`、`gCvarRelaxOffInitialDelayMin`→`CvarChanged`、`gCvarRelaxOffFinaleOffer`→`CvarChanged`、`gCvarRelaxOffOriginalOffer`→`CvarChanged`、`gCvarAutoEnable`→`CvarChanged`、`gCvarAutoCountMode`→`CvarChanged`、`gCvarAutoBaseCount`→`CvarChanged`、`gCvarAutoPerAdd`→`CvarChanged`、`gCvarAutoMinCount`→`CvarChanged`、`gCvarAutoMaxCount`→`CvarChanged`、`gCvarAutoBaseInterval`→`CvarChanged`、`gCvarAutoPerIntervalSub`→`CvarChanged`、`gCvarAutoMinInterval`→`CvarChanged`

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_dirspawn_apply` | [总特数] [间隔] | 需要 ADMFLAG_GENERIC（b，通用管理员） | 立即应用导演刷特设置 | `l4d2_dirspawn.sp:1149` |
| `sm_dirspawn_genkv` | [min] [max] | 需要 ADMFLAG_ROOT（z，最高权限） | 生成均衡的每类上限 KV 文件到 dirspawn_kv_path | `l4d2_dirspawn.sp:1150` |

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `dirspawn_enable` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 启用导演特感控制：0 关，1 开 | 服务器端本插件逻辑 | `l4d2_dirspawn.sp:1062` |
| `dirspawn_count` | `4` | 整数，源码写作浮点 | 0.0 ~ 30.0 | 并发特感总数，对应 cm_MaxSpecials，范围 0~30 | 服务器端本插件逻辑 | `l4d2_dirspawn.sp:1063` |
| `dirspawn_interval` | `35` | 整数，源码写作浮点 | 0.0 ~ 120.0 | 特感复活间隔，对应 cm_SpecialRespawnInterval，单位秒，范围 0~120 | 服务器端本插件逻辑 | `l4d2_dirspawn.sp:1064` |
| `dirspawn_dominator_limit` | `-1` | 整数，源码写作浮点 | -1.0 ~ 30.0 | DominatorLimit，-1 表示自动取 dirspawn_count | 服务器端本插件逻辑 | `l4d2_dirspawn.sp:1065` |
| `dirspawn_apply_on_roundstart` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 回合开始是否自动应用：0 否，1 是 | 服务器端本插件逻辑 | `l4d2_dirspawn.sp:1066` |
| `dirspawn_apply_delay` | `1.0` | 浮点 | 0.1 ~ 10.0 | 回合开始首次应用的延迟秒数，范围 0.1~10 | 服务器端本插件逻辑 | `l4d2_dirspawn.sp:1067` |
| `dirspawn_kv_enable` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否使用 KV 设置每类上限：0 否，1 是 | 服务器端本插件逻辑 | `l4d2_dirspawn.sp:1068` |
| `dirspawn_kv_path` | `cfg/sourcemod/dirspawn_si_limits.cfg` | 字符串或表达式 | 无上下界 | 每类上限 KV 文件路径 | 服务器端本插件逻辑 | `l4d2_dirspawn.sp:1069` |
| `dirspawn_limit_style` | `0` | 整数，源码写作浮点 | 0.0 ~ 2.0 | 每类上限分配方式：0 KV 或均衡，1 Not0721，2 Not0721 community2 | 服务器端本插件逻辑 | `l4d2_dirspawn.sp:1070` |
| `dirspawn_dps_si_limit` | `10` | 整数，源码写作浮点 | 0.0 ~ 30.0 | Not0721 分配中 Spitter 与 Boomer 的数量限制，范围 0~30 | 服务器端本插件逻辑 | `l4d2_dirspawn.sp:1071` |
| `dirspawn_verbose` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否输出详细服务器日志：0 否，1 是 | 服务器端本插件逻辑 | `l4d2_dirspawn.sp:1072` |
| `dirspawn_active_challenge` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否设置 ActiveChallenge、Aggressive、Assault 标志：0 否，1 是 | 服务器端本插件逻辑 | `l4d2_dirspawn.sp:1073` |
| `dirspawn_unlock_maxspecial` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否解锁战役模式 3 特上限，需 sourcescramble 与 gamedata | 服务器端本插件逻辑 | `l4d2_dirspawn.sp:1076` |
| `dirspawn_allow_si_with_tank` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 坦克在场时是否允许刷特：0 禁刷，1 允许并存 | 服务器端本插件逻辑 | `l4d2_dirspawn.sp:1079` |
| `dirspawn_relax_enable` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否保留导演 Relax 阶段：0 按 Not0721 源服方式压掉 Relax | 服务器端本插件逻辑 | `l4d2_dirspawn.sp:1080` |
| `dirspawn_relax_min` | `30` | 整数，源码写作浮点 | 0.0 ~ 120.0 | Relax 阶段最小秒数，范围 0~120 | 服务器端本插件逻辑 | `l4d2_dirspawn.sp:1081` |
| `dirspawn_relax_max` | `45` | 整数，源码写作浮点 | 0.0 ~ 180.0 | Relax 阶段最大秒数，范围 0~180 | 服务器端本插件逻辑 | `l4d2_dirspawn.sp:1082` |
| `dirspawn_lock_tempo` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否锁节奏：0 否，1 是 | 服务器端本插件逻辑 | `l4d2_dirspawn.sp:1083` |
| `dirspawn_relax_off_battlefield_respawn` | `2` | 整数，源码写作浮点 | 0.0 ~ 30.0 | Relax 关闭时 director_special_battlefield_respawn_interval 的取值 | 服务器端本插件逻辑 | `l4d2_dirspawn.sp:1084` |
| `dirspawn_relax_off_initial_delay_max` | `1` | 整数，源码写作浮点 | 0.0 ~ 60.0 | Relax 关闭时 director_special_initial_spawn_delay_max 的取值 | 服务器端本插件逻辑 | `l4d2_dirspawn.sp:1085` |
| `dirspawn_relax_off_initial_delay_max_extra` | `2` | 整数，源码写作浮点 | 0.0 ~ 180.0 | Relax 关闭时 director_special_initial_spawn_delay_max_extra 的取值 | 服务器端本插件逻辑 | `l4d2_dirspawn.sp:1086` |
| `dirspawn_relax_off_initial_delay_min` | `0` | 整数，源码写作浮点 | 0.0 ~ 60.0 | Relax 关闭时 director_special_initial_spawn_delay_min 的取值 | 服务器端本插件逻辑 | `l4d2_dirspawn.sp:1087` |
| `dirspawn_relax_off_finale_offer` | `1` | 整数，源码写作浮点 | 0.0 ~ 30.0 | Relax 关闭时 director_special_finale_offer_length 的取值 | 服务器端本插件逻辑 | `l4d2_dirspawn.sp:1088` |
| `dirspawn_relax_off_original_offer` | `1` | 整数，源码写作浮点 | 0.0 ~ 60.0 | Relax 关闭时 director_special_original_offer_length 的取值 | 服务器端本插件逻辑 | `l4d2_dirspawn.sp:1089` |
| `dirspawn_initial_min` | `30` | 整数，源码写作浮点 | 0.0 ~ 60.0 | 首次刷特的最小延迟秒数，范围 0~60 | 服务器端本插件逻辑 | `l4d2_dirspawn.sp:1092` |
| `dirspawn_initial_max` | `60` | 整数，源码写作浮点 | 0.0 ~ 60.0 | 首次刷特的最大延迟秒数，范围 0~60 | 服务器端本插件逻辑 | `l4d2_dirspawn.sp:1093` |
| `dirspawn_relax_auto` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否根据 interval 自动调整 Relax 与 Lock：0 否，1 是 | 服务器端本插件逻辑 | `l4d2_dirspawn.sp:1096` |
| `dirspawn_relax_kmin` | `0.75` | 浮点 | 无上下界 | RelaxMin 等于该系数乘以 interval | 服务器端本插件逻辑 | `l4d2_dirspawn.sp:1097` |
| `dirspawn_relax_kmax` | `1.10` | 浮点 | 无上下界 | RelaxMax 等于该系数乘以 interval | 服务器端本插件逻辑 | `l4d2_dirspawn.sp:1098` |
| `dirspawn_relax_floor` | `0` | 整数 | 无上下界 | RelaxMin 的下限秒数 | 服务器端本插件逻辑 | `l4d2_dirspawn.sp:1099` |
| `dirspawn_relax_ceil` | `120` | 整数 | 无上下界 | RelaxMax 的上限秒数 | 服务器端本插件逻辑 | `l4d2_dirspawn.sp:1100` |
| `dirspawn_lock_tempo_threshold` | `6` | 整数 | 无上下界 | interval 小于等于该阈值时自动把 LockTempo 设为 1，单位秒 | 服务器端本插件逻辑 | `l4d2_dirspawn.sp:1101` |
| `dirspawn_initial_auto` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否根据 interval 自动调整首刷延迟：0 否，1 是 | 服务器端本插件逻辑 | `l4d2_dirspawn.sp:1103` |
| `dirspawn_initial_kmin` | `0.80` | 浮点 | 无上下界 | 首刷最小延迟等于该系数乘以 interval | 服务器端本插件逻辑 | `l4d2_dirspawn.sp:1104` |
| `dirspawn_initial_kmax` | `1.00` | 整数，源码写作浮点 | 无上下界 | 首刷最大延迟等于该系数乘以 interval | 服务器端本插件逻辑 | `l4d2_dirspawn.sp:1105` |
| `dirspawn_auto_enable` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用人数自适应，仅调整总特数量：0 否，1 是 | 服务器端本插件逻辑 | `l4d2_dirspawn.sp:1108` |
| `dirspawn_auto_count_mode` | `1` | 整数，源码写作浮点 | 0.0 ~ 2.0 | 计数模式：0 全部真人，1 仅生还，2 生还加感染但不含观察者 | 服务器端本插件逻辑 | `l4d2_dirspawn.sp:1109` |
| `dirspawn_auto_base_count` | `6` | 整数，源码写作浮点 | 0.0 ~ 30.0 | 4 名真人时的基线总特数 | 服务器端本插件逻辑 | `l4d2_dirspawn.sp:1110` |
| `dirspawn_auto_per_player_add` | `1` | 整数，源码写作浮点 | 0.0 ~ 6.0 | 每多 1 名真人增加的特感数 | 服务器端本插件逻辑 | `l4d2_dirspawn.sp:1111` |
| `dirspawn_auto_min_count` | `1` | 整数，源码写作浮点 | 0.0 ~ 30.0 | 总特最小值 | 服务器端本插件逻辑 | `l4d2_dirspawn.sp:1112` |
| `dirspawn_auto_max_count` | `30` | 整数，源码写作浮点 | 0.0 ~ 30.0 | 总特最大值 | 服务器端本插件逻辑 | `l4d2_dirspawn.sp:1113` |
| `dirspawn_auto_base_interval` | `35` | 整数，源码写作浮点 | 0.0 ~ 120.0 | 4 名真人时的基线刷特间隔 | 服务器端本插件逻辑 | `l4d2_dirspawn.sp:1114` |
| `dirspawn_auto_per_player_interval_sub` | `0` | 整数，源码写作浮点 | 0.0 ~ 30.0 | 每多 1 名真人减少的刷特间隔 | 服务器端本插件逻辑 | `l4d2_dirspawn.sp:1115` |
| `dirspawn_auto_min_interval` | `0` | 整数，源码写作浮点 | 0.0 ~ 120.0 | 人数自适应刷特间隔的下限 | 服务器端本插件逻辑 | `l4d2_dirspawn.sp:1116` |
| `dirspawn_auto_announce` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 人数自适应变更时是否公告：0 否，1 是 | 服务器端本插件逻辑 | `l4d2_dirspawn.sp:1117` |

## L4D2 Dynamic Ammo (dirspawn only) —— `optional/AnneHappy/l4d2_dynamic_ammo.sp`

- 源文件：`optional/AnneHappy/l4d2_dynamic_ammo.sp`
- myinfo：version=1.0.1，author=morzlee
- myinfo description（源码原文）：仅依据 dirspawn_count 与 dirspawn_interval 动态调整弹药
- HookConVarChange：`g_DirCount`→`OnDirChanged`、`g_DirItv`→`OnDirChanged`

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_da_recalc` | 无参数 | 需要 ADMFLAG_GENERIC（b，通用管理员） | 手动重算並应用动态弹药倍率 | `l4d2_dynamic_ammo.sp:235` |

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `l4d2_dynamic_ammo_version` | `1.0.1` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置，默认值为 1.0.1） | 服务器端本插件逻辑；不写入 cfg 存档 | `l4d2_dynamic_ammo.sp:210` |
| `l4d2_dynamic_ammo_enable` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用：1 开，0 关 | 服务器端本插件逻辑 | `l4d2_dynamic_ammo.sp:212` |
| `l4d2_dynamic_ammo_base_si` | `4` | 整数，源码写作浮点 | ≥ 1.0 | 基准 SI 数量，最小 1 | 服务器端本插件逻辑 | `l4d2_dynamic_ammo.sp:213` |
| `l4d2_dynamic_ammo_base_interval` | `35.0` | 整数，源码写作浮点 | ≥ 1.0 | 基准刷特间隔秒数，最小 1 | 服务器端本插件逻辑 | `l4d2_dynamic_ammo.sp:214` |
| `l4d2_dynamic_ammo_alpha` | `1.0` | 整数，源码写作浮点 | ≥ 0.0 | SI 指数，最小 0 | 服务器端本插件逻辑 | `l4d2_dynamic_ammo.sp:215` |
| `l4d2_dynamic_ammo_beta` | `0.5` | 浮点 | ≥ 0.0 | 间隔指数，最小 0 | 服务器端本插件逻辑 | `l4d2_dynamic_ammo.sp:216` |
| `l4d2_dynamic_ammo_min_mult` | `1.0` | 浮点 | ≥ 0.1 | 倍率下限，最小 0.1 | 服务器端本插件逻辑 | `l4d2_dynamic_ammo.sp:217` |
| `l4d2_dynamic_ammo_max_mult` | `6.0` | 浮点 | ≥ 0.5 | 倍率上限，最小 0.5 | 服务器端本插件逻辑 | `l4d2_dynamic_ammo.sp:218` |
| `l4d2_dynamic_ammo_refill_mode` | `1` | 整数，源码写作浮点 | 0.0 ~ 2.0 | 弹药补充模式：0 仅限上限，1 回满上限（默认），2 强制等于目标 | 服务器端本插件逻辑 | `l4d2_dynamic_ammo.sp:219` |
| `l4d2_dynamic_ammo_allow_m60` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否处理 M60 弹药 | 服务器端本插件逻辑 | `l4d2_dynamic_ammo.sp:220` |
| `l4d2_dynamic_ammo_debug` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否输出调试信息 | 服务器端本插件逻辑 | `l4d2_dynamic_ammo.sp:221` |
| `l4d2_dynamic_ammo_base_smg` | `650` | 整数 | 无上下界 | SMG 基础预备弹数 | 服务器端本插件逻辑 | `l4d2_dynamic_ammo.sp:223` |
| `l4d2_dynamic_ammo_base_rifle` | `360` | 整数 | 无上下界 | 步枪基础预备弹数 | 服务器端本插件逻辑 | `l4d2_dynamic_ammo.sp:224` |
| `l4d2_dynamic_ammo_base_pump` | `72` | 整数 | 无上下界 | 泵动霰弹枪基础预备弹数 | 服务器端本插件逻辑 | `l4d2_dynamic_ammo.sp:225` |
| `l4d2_dynamic_ammo_base_auto` | `90` | 整数 | 无上下界 | 连发霰弹枪基础预备弹数 | 服务器端本插件逻辑 | `l4d2_dynamic_ammo.sp:226` |
| `l4d2_dynamic_ammo_base_sniper` | `180` | 整数 | 无上下界 | 狙击枪基础预备弹数 | 服务器端本插件逻辑 | `l4d2_dynamic_ammo.sp:227` |
| `l4d2_dynamic_ammo_base_gl` | `30` | 整数 | 无上下界 | 榴弹发射器基础预备弹数 | 服务器端本插件逻辑 | `l4d2_dynamic_ammo.sp:228` |

## L4D2 hunter patch —— `optional/AnneHappy/l4d2_hunter_patch.sp`

- 源文件：`optional/AnneHappy/l4d2_hunter_patch.sp`
- myinfo：author=fdxx
- myinfo description（源码原文）：Patched some hunter function.

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_hunter_patch_print_cvars` | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 输出 l4d2_hunter_patch 相关 cvar 的当前取值 | `l4d2_hunter_patch.sp:46` |

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `l4d2_hunter_patch_version` | `VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（默认值取自 VERSION 宏） | 服务器端本插件逻辑；不写入 cfg 存档 | `l4d2_hunter_patch.sp:37` |
| `l4d2_hunter_patch_convert_leap` | `0` | 整数 | 无上下界 | 是否把跳跃转为飞扑：0 游戏默认，1 总是，2 从不 | 服务器端本插件逻辑 | `l4d2_hunter_patch.sp:39` |
| `l4d2_hunter_patch_crouch_pounce` | `0` | 整数 | 无上下界 | 在地面是否需要按蹲下键才能飞扑：0 游戏默认，1 总是，2 从不 | 服务器端本插件逻辑 | `l4d2_hunter_patch.sp:40` |
| `l4d2_hunter_patch_bonus_damage` | `0` | 整数 | 无上下界 | 是否启用飞扑附加伤害：0 游戏默认，1 总是，2 从不 | 服务器端本插件逻辑 | `l4d2_hunter_patch.sp:41` |
| `l4d2_hunter_patch_pounce_interrupt` | `0` | 整数 | 无上下界 | 是否启用飞扑被打断：0 游戏默认，1 总是，2 从不 | 服务器端本插件逻辑 | `l4d2_hunter_patch.sp:42` |

## l4d2_med_dynamic.sp（myinfo 缺 name，用文件名代替） —— `optional/AnneHappy/l4d2_med_dynamic.sp`

- 源文件：`optional/AnneHappy/l4d2_med_dynamic.sp`
- myinfo：（无 version/author）
- myinfo description（源码原文）：Remove saferoom medkits and hand them out on exit based on survivor count.
- HookConVarChange：`gC_Enable`→`CvarChanged`、`gC_RemoveDelay`→`CvarChanged`、`gC_Debug`→`CvarChanged`

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_srmedkit_apply` | 无参数 | 需要 ADMFLAG_GENERIC（b，通用管理员） | 立即移除安全室医疗包并按生还者人数分配 | `l4d2_med_dynamic.sp:88` |

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `sr_medkit_enable` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用安全室医疗包控制：1 开，0 关 | 服务器端本插件逻辑 | `l4d2_med_dynamic.sp:75` |
| `sr_medkit_scan_delay` | `1.5` | 浮点 | 0.0 ~ 10.0 | round_start 后剥离安全室医疗包的延迟秒数，范围 0~10 | 服务器端本插件逻辑 | `l4d2_med_dynamic.sp:76` |
| `sr_medkit_debug` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否输出调试日志 | 服务器端本插件逻辑 | `l4d2_med_dynamic.sp:77` |

## l4d2_npc_manager —— `optional/AnneHappy/l4d2_npc_manager.sp`

- 源文件：`optional/AnneHappy/l4d2_npc_manager.sp`
- myinfo：version=1.0，author=洛琪,小燐RM
- myinfo description（源码原文）：插件控制每种特感、僵尸的刷新率，不同特感可以不同刷新率，适用tank、witch、小僵尸和特感

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `nb_uf_onoff` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 插件是否接管 update frequency：1 接管，0 不接管 | 服务器端本插件逻辑 | `l4d2_npc_manager.sp:110` |
| `nb_uf_Common` | `0.02` | 浮点 | 0.0 ~ 1.0 | Common 的 update frequency 更新频率，范围 0~1 | 服务器端本插件逻辑 | `l4d2_npc_manager.sp:111` |
| `nb_uf_Smoker` | `0.05` | 浮点 | 0.0 ~ 1.0 | Smoker 的 update frequency 更新频率，范围 0~1 | 服务器端本插件逻辑 | `l4d2_npc_manager.sp:112` |
| `nb_uf_Boomer` | `0.05` | 浮点 | 0.0 ~ 1.0 | Boomer 的 update frequency 更新频率，范围 0~1 | 服务器端本插件逻辑 | `l4d2_npc_manager.sp:113` |
| `nb_uf_Hunter` | `0.02` | 浮点 | 0.0 ~ 1.0 | Hunter 的 update frequency 更新频率，范围 0~1 | 服务器端本插件逻辑 | `l4d2_npc_manager.sp:114` |
| `nb_uf_Spitter` | `0.05` | 浮点 | 0.0 ~ 1.0 | Spitter 的 update frequency 更新频率，范围 0~1 | 服务器端本插件逻辑 | `l4d2_npc_manager.sp:115` |
| `nb_uf_Jockey` | `0.05` | 浮点 | 0.0 ~ 1.0 | Jockey 的 update frequency 更新频率，范围 0~1 | 服务器端本插件逻辑 | `l4d2_npc_manager.sp:116` |
| `nb_uf_Charger` | `0.05` | 浮点 | 0.0 ~ 1.0 | Charger 的 update frequency 更新频率，范围 0~1 | 服务器端本插件逻辑 | `l4d2_npc_manager.sp:117` |
| `nb_uf_Witch` | `0.02` | 浮点 | 0.0 ~ 1.0 | Witch 的 update frequency 更新频率，范围 0~1 | 服务器端本插件逻辑 | `l4d2_npc_manager.sp:118` |
| `nb_uf_Tank` | `0.02` | 浮点 | 0.0 ~ 1.0 | Tank 的 update frequency 更新频率，范围 0~1 | 服务器端本插件逻辑 | `l4d2_npc_manager.sp:119` |
| `nb_uf_sb` | `0.1` | 浮点 | 0.0 ~ 1.0 | Survivor Bot 的 update frequency 更新频率，范围 0~1 | 服务器端本插件逻辑 | `l4d2_npc_manager.sp:120` |

## L4D2 SI LADDER BOOSTER —— `optional/AnneHappy/l4d2_si_ladder_booster.sp`

- 源文件：`optional/AnneHappy/l4d2_si_ladder_booster.sp`
- myinfo：author=AiMee

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `l4d2_ai_ladder_boost` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否允许 AI 特感在梯子上加速，范围 0~1 | 服务器端本插件逻辑；仅服务器 | `l4d2_si_ladder_booster.sp:30` |
| `l4d2_pz_ladder_boost` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否允许真人特感在梯子上加速，范围 0~1 | 服务器端本插件逻辑；仅服务器 | `l4d2_si_ladder_booster.sp:31` |
| `l4d2_boost_multiplier` | `3.2` | 浮点 | 0.0 ~ 10.0 | 爬梯加速倍数，范围 0~10 | 服务器端本插件逻辑；仅服务器 | `l4d2_si_ladder_booster.sp:32` |

## Tank刷新提示 —— `optional/AnneHappy/l4d2_tank_announce.sp`

- 源文件：`optional/AnneHappy/l4d2_tank_announce.sp`
- myinfo：version=1.0.1.1，author=夜羽真白
- myinfo description（源码原文）：当Tank生成时，进行提示

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `l4d2_tankannounce_playsound` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否在 Tank 生成时播放声音 | 服务器端本插件逻辑 | `l4d2_tank_announce.sp:29` |
| `l4d2_tankannounce_messagetype` | `1` | 整数，源码写作浮点 | 0.0 ~ 3.0 | Tank 生成提示类型：0 不提示，1 聊天框，2 中央提示框，3 中央文字 | 服务器端本插件逻辑 | `l4d2_tank_announce.sp:30` |

## [L4D1 & L4D2] CreateSurvivorBot —— `optional/AnneHappy/l4d_CreateSurvivorBot.sp`

- 源文件：`optional/AnneHappy/l4d_CreateSurvivorBot.sp`
- myinfo：author=MicroLeo (port by Dragokas)
- myinfo description（源码原文）：Provides CreateSurvivorBot Native

### 指令

（本插件未注册指令）

### ConVar

（本插件未注册 ConVar）

## l4d_air_abilities_patch.sp（myinfo 缺 name，用文件名代替） —— `optional/AnneHappy/l4d_air_abilities_patch.sp`

- 源文件：`optional/AnneHappy/l4d_air_abilities_patch.sp`
- myinfo：源码中未找到 myinfo 块

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `l4d_air_abilities_patch_neri` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用 Anne-Neri 的 smoker 与 boomer 空中技能补丁 | 服务器端本插件逻辑 | `l4d_air_abilities_patch.sp:22` |

## [L4D2] Vote Boss —— `optional/AnneHappy/l4d_boss_vote.sp`

- 源文件：`optional/AnneHappy/l4d_boss_vote.sp`
- myinfo：author=Spoon, Forgetest
- myinfo description（源码原文）：Votin for boss change.

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_voteboss` | <是否坦克 0或1> <百分比> | 无（任意玩家可用） | 发起坦克或女巫刷新百分比投票 | `l4d_boss_vote.sp:53` |
| `sm_bossvote` | <是否坦克 0或1> <百分比> | 无（任意玩家可用） | sm_voteboss 的别名 | `l4d_boss_vote.sp:54` |
| `sm_ftank` | <百分比> | 需要 ADMFLAG_BAN（d，封禁） | 管理员强制设置坦克刷新百分比 | `l4d_boss_vote.sp:56` |
| `sm_fwitch` | <百分比> | 需要 ADMFLAG_BAN（d，封禁） | 管理员强制设置女巫刷新百分比 | `l4d_boss_vote.sp:57` |

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `l4d_boss_vote` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用 boss 投票 | 服务器端本插件逻辑 | `l4d_boss_vote.sp:48` |
| `l4d_boss_vote_limit` | `0` | 整数，源码写作浮点 | ≥ 0.0 | 第几回合之后才允许发起 boss 投票，最小 0 | 服务器端本插件逻辑 | `l4d_boss_vote.sp:49` |

## [L4D & L4D2] Special Infected Ability Movement —— `optional/AnneHappy/l4d_infected_movement.sp`

- 源文件：`optional/AnneHappy/l4d_infected_movement.sp`
- myinfo：author=SilverShot
- myinfo description（源码原文）：Continue normal movement speed while spitting/smoking/tank throwing rocks.

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `l4d_infected_movement_allow` | `3` | 整数 | 无上下界 | 0 关闭插件，1 仅真人可用，2 仅 Bot 可用，3 两者都可用 | 服务器端本插件逻辑 | `l4d_infected_movement.sp:121` |
| `l4d_infected_movement_modes` | `` | 字符串或表达式 | 无上下界 | 在这些游戏模式中启用插件，逗号分隔，留空为全部模式 | 服务器端本插件逻辑 | `l4d_infected_movement.sp:122` |
| `l4d_infected_movement_modes_off` | `` | 字符串或表达式 | 无上下界 | 在这些游戏模式中关闭插件，逗号分隔，留空为不关闭 | 服务器端本插件逻辑 | `l4d_infected_movement.sp:123` |
| `l4d_infected_movement_modes_tog` | `0` | 整数 | 无上下界 | 按模式位掩码启用插件：0 全部，1 战役，2 生存，4 对战，8 清道夫，可相加 | 服务器端本插件逻辑 | `l4d_infected_movement.sp:124` |
| `l4d_infected_movement_bots` | `2` | 整数 | 无上下界 | 哪些 AI 特感可使用技能期间移动：1 Smoker，2 Spitter，4 Tank，7 全部，可相加 | 服务器端本插件逻辑 | `l4d_infected_movement.sp:125` |
| `l4d_infected_movement_smoker` | `2` | 整数 | 无上下界 | 0 仅吐舌时；1 Smoker 拉人时可移动；2 舌头挂住目标时也可移动 | 服务器端本插件逻辑 | `l4d_infected_movement.sp:126` |
| `l4d_infected_movement_speed_smoker` | `250` | 整数 | 无上下界 | Smoker 使用技能时的移动速度 | 服务器端本插件逻辑 | `l4d_infected_movement.sp:127` |
| `l4d_infected_movement_speed_tank` | `210` | 整数 | 无上下界 | Tank 使用技能时的移动速度 | 服务器端本插件逻辑 | `l4d_infected_movement.sp:128` |
| `l4d_infected_movement_speed_spitter` | `250` | 整数 | 无上下界 | Spitter 使用技能时的移动速度 | 服务器端本插件逻辑 | `l4d_infected_movement.sp:130` |
| `l4d_infected_movement_type` | `2` | 整数 | 无上下界 | 哪些真人特感可使用技能期间移动：1 Smoker，2 Spitter，4 Tank，7 全部，可相加 | 服务器端本插件逻辑 | `l4d_infected_movement.sp:131` |
| `l4d_infected_movement_version` | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | 服务器端本插件逻辑；不写入 cfg 存档 | `l4d_infected_movement.sp:132` |

## [L4D1 & L4D2] Multi witches —— `optional/AnneHappy/l4d_multi_witches.sp`

- 源文件：`optional/AnneHappy/l4d_multi_witches.sp`
- myinfo：author=Sheleu (Fork by Dragokas)
- myinfo description（源码原文）：Spawns more witches on the map

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `l4d_multi_witches_version` | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | 服务器端本插件逻辑；不写入 cfg 存档 | `l4d_multi_witches.sp:71` |
| `l4d_witches_limit` | `20` | 整数 | 无上下界 | 允许刷出的 Witch 数量上限，0 表示不检查数量 | 服务器端本插件逻辑 | `l4d_multi_witches.sp:73` |
| `l4d_witches_limit_alive` | `3` | 整数 | 无上下界 | 允许同时存活的 Witch 上限，0 表示不检查存活数量 | 服务器端本插件逻辑 | `l4d_multi_witches.sp:74` |
| `l4d_witches_spawn_time_min` | `20.0` | 整数，源码写作浮点 | 无上下界 | 插件刷 Witch 的最小间隔秒数 | 服务器端本插件逻辑 | `l4d_multi_witches.sp:75` |
| `l4d_witches_spawn_time_max` | `35.0` | 整数，源码写作浮点 | 无上下界 | 插件刷 Witch 的最大间隔秒数 | 服务器端本插件逻辑 | `l4d_multi_witches.sp:76` |
| `l4d_witches_distance` | `1600.0` | 整数，源码写作浮点 | 无上下界 | 距生还者多远以外的 Witch 会被移除，0 表示不移除 | 服务器端本插件逻辑 | `l4d_multi_witches.sp:77` |
| `l4d_witches_director_witch` | `1` | 整数 | 无上下界 | 1 启用导演 Witch，0 禁用导演 Witch | 服务器端本插件逻辑 | `l4d_multi_witches.sp:78` |

## L4D(2) Tank Rock Lag Compensation —— `optional/AnneHappy/l4d_rock_lagcomp.sp`

- 源文件：`optional/AnneHappy/l4d_rock_lagcomp.sp`
- myinfo：version=2.1-anne，author=Luckylockm, harry, Silvers, AnneHappy
- myinfo description（源码原文）：Provides lag compensation and weapon-attribute damage handling for tank rocks

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `sm_rock_print` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否打印石头伤害与距离数值 | 服务器端本插件逻辑 | `l4d_rock_lagcomp.sp:53` |
| `sm_rock_hitbox` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用自定义石头命中盒与伤害处理 | 服务器端本插件逻辑 | `l4d_rock_lagcomp.sp:54` |
| `sm_rock_lagcomp` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否对即时命中的石头射击启用延迟补偿 | 服务器端本插件逻辑 | `l4d_rock_lagcomp.sp:55` |
| `sm_rock_godframes` | `1.7` | 浮点 | 0.0 ~ 10.0 | 未检测到石头投掷时的兜底保护秒数，从石头创建算起，范围 0~10 | 服务器端本插件逻辑 | `l4d_rock_lagcomp.sp:56` |
| `sm_rock_release_godframes` | `0.15` | 浮点 | 0.0 ~ 10.0 | 实际投出后石头的保护秒数，0 表示可立即造成伤害，范围 0~10 | 服务器端本插件逻辑 | `l4d_rock_lagcomp.sp:57` |
| `sm_rock_godframes_render` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 石头受保护期间是否显示视觉反馈 | 服务器端本插件逻辑 | `l4d_rock_lagcomp.sp:58` |
| `sm_rock_hitbox_radius` | `30` | 整数，源码写作浮点 | 0.0 ~ 10000.0 | 自定义石头命中盒半径，范围 0~10000 | 服务器端本插件逻辑 | `l4d_rock_lagcomp.sp:59` |
| `sm_rock_range_min_all` | `1` | 整数，源码写作浮点 | 0.0 ~ 10000.0 | 即时命中石头伤害的全局最小距离，范围 0~10000 | 服务器端本插件逻辑 | `l4d_rock_lagcomp.sp:60` |
| `sm_rock_range_max_all` | `2000` | 整数，源码写作浮点 | 0.0 ~ 10000.0 | 石头伤害的全局最大距离，0 表示不限制，范围 0~10000 | 服务器端本插件逻辑 | `l4d_rock_lagcomp.sp:61` |

## Tank Damage Announce 2.0 —— `optional/AnneHappy/l4d_tank_damage_announce.sp`

- 源文件：`optional/AnneHappy/l4d_tank_damage_announce.sp`
- myinfo：version=2023/1/16，author=夜羽真白
- myinfo description（源码原文）：Tank 伤害统计 2.0 版本

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `tank_damage_enable` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否允许在 Tank 死亡后输出生还者对 Tank 的伤害统计 | 服务器端本插件逻辑 | `l4d_tank_damage_announce.sp:66` |
| `tank_damage_force_kill_announce` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | Tank 被强制处死或自杀时是否输出生还者对 Tank 的伤害统计 | 服务器端本插件逻辑 | `l4d_tank_damage_announce.sp:67` |
| `tank_damage_print_livetime` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否显示 Tank 的存活时间 | 服务器端本插件逻辑 | `l4d_tank_damage_announce.sp:68` |
| `tank_damage_failed_announce` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 生还者团灭而场上仍有 Tank 时是否显示伤害统计 | 服务器端本插件逻辑 | `l4d_tank_damage_announce.sp:69` |
| `tank_damage_print_zero` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否显示对 Tank 零伤害的玩家 | 服务器端本插件逻辑 | `l4d_tank_damage_announce.sp:70` |
| `tank_damage_enable_healthset` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否把坦克生命值设置为 z_tank_health 的数值 | 服务器端本插件逻辑 | `l4d_tank_damage_announce.sp:71` |

## [L4D & L4D2] Target Override —— `optional/AnneHappy/l4d_target_override.sp`

- 源文件：`optional/AnneHappy/l4d_target_override.sp`
- myinfo：author=SilverShot
- myinfo description（源码原文）：Overrides Special Infected targeting of Survivors.

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_to_reload` | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 重新加载数据配置 | `l4d_target_override.sp:491` |
| `sm_to_stats` | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 显示目标选择函数的性能统计，含最小、平均与最大耗时 | `l4d_target_override.sp:494` |

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `l4d_target_override_allow` | `1` | 整数 | 无上下界 | 0 关闭插件，1 开启插件 | 服务器端本插件逻辑 | `l4d_target_override.sp:457` |
| `l4d_target_override_modes` | `` | 字符串或表达式 | 无上下界 | 在这些游戏模式中启用插件，逗号分隔，留空为全部模式 | 服务器端本插件逻辑 | `l4d_target_override.sp:458` |
| `l4d_target_override_modes_off` | `` | 字符串或表达式 | 无上下界 | 在这些游戏模式中关闭插件，逗号分隔，留空为不关闭 | 服务器端本插件逻辑 | `l4d_target_override.sp:459` |
| `l4d_target_override_modes_tog` | `0` | 整数 | 无上下界 | 按模式位掩码启用插件：0 全部，1 战役，2 生存，4 对战，8 清道夫，可相加 | 服务器端本插件逻辑 | `l4d_target_override.sp:460` |
| `l4d_target_override_forward` | `0` | 整数 | 无上下界 | 0 关闭；1 为其他插件提供每帧左右触发的被瞄准前向回调 | 服务器端本插件逻辑 | `l4d_target_override.sp:461` |
| `l4d_target_override_specials` | `127` | 整数 | 无上下界 | 覆盖哪些特感的目标函数：1 Smoker，2 Boomer，4 Hunter，8 Spitter，16 Jockey，32 Charger，64 Tank，127 为全部 | 服务器端本插件逻辑 | `l4d_target_override.sp:463` |
| `l4d_target_override_specials` | `15` | 整数 | 无上下界 | 覆盖哪些特感的目标函数：1 Smoker，2 Boomer，4 Hunter，8 Tank，15 为全部，可相加 | 服务器端本插件逻辑 | `l4d_target_override.sp:465` |
| `l4d_target_override_team` | `2` | 整数 | 无上下界 | 应对哪些生还者队伍生效：2 默认生还者，4 Holdout 与 Passing 的 Bot，6 两者 | 服务器端本插件逻辑 | `l4d_target_override.sp:466` |
| `l4d_target_override_type` | `1` | 整数 | 无上下界 | 插件搜索生还者的方式：1 最近的可见目标，2 全部生还者，其余源码描述被截断 | 服务器端本插件逻辑 | `l4d_target_override.sp:467` |
| `l4d_target_override_version` | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | 服务器端本插件逻辑；不写入 cfg 存档 | `l4d_target_override.sp:468` |

## Remove Kits or replace kits and remove defib —— `optional/AnneHappy/remove.sp`

- 源文件：`optional/AnneHappy/remove.sp`
- myinfo：version=2022.12.22，author=Caibiii, 夜羽真白, 东
- myinfo description（源码原文）：开局删除(非救援)或者替换(救援)已经缓存在地图上的急救包,让药的数量刚好为confogl_pills_limit的值或者l4d2_remove_pillsLimit的值

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `l4d2_remove_pillsLimit` | `4` | 整数 | 无上下界 | 限制止痛药最多出现数量 | 服务器端本插件逻辑 | `remove.sp:29` |

## [L4D & L4D2] Simple AFK Manager VS —— `optional/AnneHappy/sam_vs.sp`

- 源文件：`optional/AnneHappy/sam_vs.sp`
- myinfo：author=raziEiL [disawar1]
- myinfo description（源码原文）：Players constantly take slot on your server? Plugin take care of them
- HookConVarChange：`hSpecT`→`OnCvarChange_SpecT`、`hKickT`→`OnCvarChange_KickT`、`hImBack`→`OnCvarChange_ImBack`、`hTank`→`OnCvarChange_Tank`、`hKickF`→`OnCvarChange_KickF`、`hAdmin`→`OnCvarChange_Admin`

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `sam_vs_version` | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | 服务器端本插件逻辑；仅服务器；不写入 cfg 存档 | `sam_vs.sp:47` |
| `sam_vs_spec_time` | `35` | 整数，源码写作浮点 | ≥ 10.0 | 闲置玩家被移到旁观者前的等待秒数，最小 10 | 服务器端本插件逻辑 | `sam_vs.sp:49` |
| `sam_vs_kick_time` | `120` | 整数，源码写作浮点 | ≥ 0.0 | 闲置旁观玩家被踢出前的等待秒数，0 表示永不踢出 | 服务器端本插件逻辑 | `sam_vs.sp:50` |
| `sam_vs_respect_spec` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 已不再挂机的旁观玩家是否不踢出 | 服务器端本插件逻辑 | `sam_vs.sp:51` |
| `sam_vs_respect_tank` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 正在扮演 Tank 的挂机玩家是否不移到旁观 | 服务器端本插件逻辑 | `sam_vs.sp:52` |
| `sam_vs_respect_admins` | `k` | 字符串或表达式 | 无上下界 | 管理员免疫 AFK 管理器的 flag 值，留空表示不保护管理员 | 服务器端本插件逻辑 | `sam_vs.sp:53` |
| `sam_vs_force_kick` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 换图时是否踢出所有闲置的旁观玩家，用于与其他插件的兼容 | 服务器端本插件逻辑 | `sam_vs.sp:54` |

## Scene Processor —— `optional/AnneHappy/sceneprocessor.sp`

- 源文件：`optional/AnneHappy/sceneprocessor.sp`
- myinfo：author=Buster \
- myinfo description（源码原文）：Provides forwards and natives for manipulation of scenes
- HookConVarChange：`convar`→`OnJailBreakConVarChanged`

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `sceneprocessor_jailbreak_vocalize` | `1` | 整数 | 无上下界 | 内鬼模式发泄命令的 ConVar 名，源码中由宏 CONVAR_JAILBREAK_VOCALIZE_COMMAND_NAME 定义 | 服务器端本插件逻辑；插件创建 | `sceneprocessor.sp:186` |
| `sceneprocessor_version` | `1.0.1` | 字符串 | 无上下界 | 插件版本号，源码中由宏 CONVAR_VERSION_NAME 定义 | 服务器端本插件逻辑；插件创建 | `sceneprocessor.sp:192` |

## AnneServer Server Function (quiet minimal) —— `optional/AnneHappy/server.sp`

- 源文件：`optional/AnneHappy/server.sp`
- myinfo：version=2026.08.14，author=def075, Caibiii, 东, simplified by ChatGPT
- myinfo description（源码原文）：Helpers + BeQuiet-style suppressors only (server_cvar / namechange / spec chat)

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_setbot` | 无参数 | 无（任意玩家可用） | 在随机生还者附近生成一个生还者 Bot，源码未给描述 | `server.sp:111` |
| `sm_kicktank` | 无参数 | 需要 ADMFLAG_KICK（c，踢出） | 有多只 Tank 时随机踢到只剩一只 | `server.sp:112` |
| `sm_addbot` | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 添加一个不会被本插件踢出的生还者 Bot | `server.sp:114` |
| `sm_delbot` | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 删除一个未被接管的生还者 Bot | `server.sp:115` |
| `sm_zs` | 无参数 | 无（任意玩家可用） | 自杀，源码未给描述 | `server.sp:116` |
| `sm_kill` | 无参数 | 无（任意玩家可用） | 自杀，sm_zs 的别名 | `server.sp:117` |

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `l4d_multislots_survivors_manager_enable` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用生还者数量管理：0 否，1 是 | 服务器端本插件逻辑 | `server.sp:120` |
| `l4d_multislots_max_survivors` | `4` | 整数，源码写作浮点 | 4.0 ~ 8.0 | 生还者最大人数，仅踢 Bot 不踢真人，范围 4~8 | 服务器端本插件逻辑 | `server.sp:122` |
| `l4d_multislots_autokicktank` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 当场上 Tank 大于 1 时是否自动踢到只剩 1 只：0 否，1 是 | 服务器端本插件逻辑 | `server.sp:124` |
| `anne_reset_on_transition` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 通关或切图时是否执行 RestoreHealth 与 ResetInventory：0 否，1 是 | 服务器端本插件逻辑 | `server.sp:128` |
| `anne_heal50_on_transition` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 通关或切图时若生还者实血小于 50 则补至 50 并重置倒地次数，仅在 anne_reset_on_transition 为 0 时生效 | 服务器端本插件逻辑 | `server.sp:132` |
| `anne_spawn_warp_to_start` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 生还者 player_spawn 后是否在未离开安全区前传送到起始点：0 否，1 是 | 服务器端本插件逻辑 | `server.sp:136` |
| `anne_round_wipe_count` | `0` | 整数，源码写作浮点 | ≥ 0.0 | 当前地图已发生的团灭次数，只读状态 | 服务器端本插件逻辑；变更时通知客户端；不写入 cfg 存档 | `server.sp:138` |
| `l4d_bw_notify_enable` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用生还者黑白提醒：0 否，1 是 | 服务器端本插件逻辑 | `server.sp:148` |
| `l4d_bw_notify_team` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 0 只提醒本人，1 全队广播 | 服务器端本插件逻辑 | `server.sp:149` |
| `l4d_bw_notify_sound` | `ui/beep07.wav` | 字符串或表达式 | 无上下界 | 玩家变黑白时播放的音效，留空不播 | 服务器端本插件逻辑 | `server.sp:150` |
| `bq_cvar_change_suppress` | `1` | 整数 | 无上下界 | 是否屏蔽服务器 cvar 变更提示，使聊天更干净 | 服务器端本插件逻辑 | `server.sp:155` |
| `bq_name_change_suppress` | `1` | 整数 | 无上下界 | 是否屏蔽玩家改名提示 | 服务器端本插件逻辑 | `server.sp:156` |
| `bq_name_change_spec_suppress` | `1` | 整数 | 无上下界 | 是否屏蔽旁观玩家改名提示 | 服务器端本插件逻辑 | `server.sp:157` |
| `bq_show_player_team_chat_spec` | `1` | 整数 | 无上下界 | 是否向旁观者显示生还者与特感的团队聊天 | 服务器端本插件逻辑 | `server.sp:158` |

## L4d2-Si-Push-When-Spawn —— `optional/AnneHappy/si_push_when_spawn.sp`

- 源文件：`optional/AnneHappy/si_push_when_spawn.sp`
- myinfo：version=1.0.1.0，author=夜羽真白
- myinfo description（源码原文）：感染者生成时进行推动

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `l4d2_si_push_enable` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否开启特感刷出时朝生还方向推动的效果 | 服务器端本插件逻辑 | `si_push_when_spawn.sp:34` |
| `l4d2_si_push_infected` | `2,4,5,6` | 字符串或表达式 | 无上下界 | 允许刷出时推动的特感种类，逗号分隔 | 服务器端本插件逻辑 | `si_push_when_spawn.sp:35` |
| `l4d2_si_push_allow_player` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否允许真人特感刷出时推动 | 服务器端本插件逻辑 | `si_push_when_spawn.sp:36` |
| `l4d2_si_push_force` | `600` | 整数，源码写作浮点 | ≥ 0.0 | 特感刷出时的推动力度 | 服务器端本插件逻辑 | `si_push_when_spawn.sp:37` |
| `l4d2_si_push_only_high` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否只对在高处刷出的特感启用推动 | 服务器端本插件逻辑 | `si_push_when_spawn.sp:38` |
| `l4d2_si_push_height` | `200` | 整数，源码写作浮点 | ≥ 0.0 | 特感刷出位置高于目标生还者该高度即视为高处 | 服务器端本插件逻辑 | `si_push_when_spawn.sp:39` |

## Anne Spawn Vote Menu —— `optional/AnneHappy/spawn_vote_menu.sp`

- 源文件：`optional/AnneHappy/spawn_vote_menu.sp`
- myinfo：author=morzlee
- myinfo description（源码原文）：SourceMod menu based spawn tuning vote menu for Anne and campaign modes.

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_spawnvote` | 无参数 | 无（任意玩家可用） | 打开刷特投票菜单 | `spawn_vote_menu.sp:107` |
| `sm_sivote` | 无参数 | 无（任意玩家可用） | 打开刷特投票菜单，sm_spawnvote 的别名 | `spawn_vote_menu.sp:108` |
| `sm刷特` | 无参数 | 无（任意玩家可用） | 打开刷特投票菜单，中文别名 | `spawn_vote_menu.sp:109` |
| `sm_spawnpreset_save` | <名字> | 需要 ADMFLAG_CONFIG（h，服务器配置） | 把当前刷特设置保存为预设 | `spawn_vote_menu.sp:110` |
| `sm_spawnpreset_delete` | <名字> | 需要 ADMFLAG_CONFIG（h，服务器配置） | 删除指定的刷特预设 | `spawn_vote_menu.sp:111` |
| `sm_spawnpreset_list` | 无参数 | 无（任意玩家可用） | 列出当前模式的刷特预设 | `spawn_vote_menu.sp:112` |

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `sm_spawnvote_preset_db` | `storage-local` | 字符串或表达式 | 无上下界 | 刷特预设数据库配置名，支持 MySQL 或 SQLite | 服务器端本插件逻辑 | `spawn_vote_menu.sp:114` |
| `sm_spawnvote_preset_table` | `spawn_vote_presets` | 字符串或表达式 | 无上下界 | 刷特预设数据库表名 | 服务器端本插件逻辑 | `spawn_vote_menu.sp:115` |

## text.sp（myinfo 缺 name，用文件名代替） —— `optional/AnneHappy/text.sp`

- 源文件：`optional/AnneHappy/text.sp`
- myinfo：源码中未找到 myinfo 块
- HookConVarChange：`g_hCvarInfectedTime`→`Cvar_InfectedTime`、`g_hCvarInfectedLimit`→`Cvar_InfectedLimit`、`g_hCvarTankBhop`→`CvarTankBhop`、`g_hCvarWeapon`→`CvarWeapon`、`g_hCvarPluginVersion`→`CvarPluginVersion`

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_xx` | 无参数 | 无（任意玩家可用） | 源码未给描述；据 `optional/AnneHappy/text.sp:396-400` 回调 `InfectedStatus` 只调用 `printinfo(Client)`，:180-195 的 `printinfo` 再对该玩家调用 `PrintInfoToClient`，后者（:207-280）逐行输出坦克 Bhop 开关（:223-224）、当前武器配置档位（:226-228）、AI 难度、特感上限与刷新间隔（:240）以及刷怪距离/传送检查/回血/坦克消耗等 ConVar（:242-274），推测为：查询本局特感与插件配置状态的玩家命令（`sm_xx` 命名随意，实质等同 `sm_info`；`event_RoundStart` 开局用同一函数全服播报，:401-404）（置信度：高） | `text.sp:58` |
| `sm_killall` | 无参数 | 需要 ADMFLAG_BAN（d，封禁） | 处死所有玩家 | `text.sp:64` |

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `ZonemodWeapon` | `0` | 整数 | 无上下界 | 整服配置档位开关；据 `optional/AnneHappy/text.sp:41` 的 `CreateConVar("ZonemodWeapon", "0", "", 0, false, 0.0, false, 0.0)`（描述为空字符串），:150-177 `CvarWeapon` 按值执行 `ServerCommand("exec vote/weapon/...cfg")`：1→`zonemod.cfg`、0→`AnneHappy.cfg`、2→特感上限≥10 或配置名含 Alone/1vHunters 时执行 `AnneHappyPlus.cfg`，否则回退成 0（:158-176），:226 又把它映射为显示名 `Weapon>1?"Anne+":(Weapon>0?"Zone":"Anne")`，推测为：切换整服武器/玩法配置风格，0=AnneHappy、1=Zonemod、2=AnneHappyPlus（置信度：高） | 服务器端本插件逻辑 | `text.sp:41` |
| `AnnePluginVersion` | `Latest` | 字符串或表达式 | 无上下界 | Anne 插件版本 | 服务器端本插件逻辑 | `text.sp:42` |
| `coopmode` | `0` | 整数 | 无上下界 | 合作模式标记；据 `optional/AnneHappy/text.sp:59` 的 `CreateConVar("coopmode", "0")`（连描述参数都没有），全文唯一读取处在 :99-105 `Incap_Event`（钩子为 `player_incapacitated_start`/`player_incapacitated`，:60-61）：`if(GetConVarBool(g_hCvarCoop)) ForcePlayerSuicide(Incap);`，即被击倒的幸存者立即被处死，随后 :106-108 再按 `IsTeamImmobilised()` 决定是否全队处死，推测为：标记本局是否为合作(campaign)模式；1=倒地即死（跳过倒地挣扎/被救流程），0（默认，对抗）=保持原版倒地逻辑（置信度：中） | 服务器端本插件逻辑 | `text.sp:59` |

## Restore Blocked Vocalize —— `optional/AnneHappy/tls_restore_vocalize.sp`

- 源文件：`optional/AnneHappy/tls_restore_vocalize.sp`
- myinfo：author=Forgetest
- myinfo description（源码原文）：Annoyments outside TLS are back.

### 指令

（本插件未注册指令）

### ConVar

（本插件未注册 ConVar）

## versus_coop_mode.sp（myinfo 缺 name，用文件名代替） —— `optional/AnneHappy/versus_coop_mode.sp`

- 源文件：`optional/AnneHappy/versus_coop_mode.sp`
- myinfo：源码中未找到 myinfo 块

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `versus_coop_mode_version` | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | 服务器端本插件逻辑；不写入 cfg 存档 | `versus_coop_mode.sp:56` |

## Witch Damage Announce —— `optional/AnneHappy/witch_announce.sp`

- 源文件：`optional/AnneHappy/witch_announce.sp`
- myinfo：version=1.2，author=Sir
- myinfo description（源码原文）：Print Witch Damage to chat

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `witch_show_true_damage` | `0` | 整数 | 无上下界 | 是否显示伤害输出而非 Witch 实际受到的伤害，0 显示实际生命伤害 | 服务器端本插件逻辑 | `witch_announce.sp:93` |

## L4D2 Witch glow —— `optional/AnneHappy/witch_glow.sp`

- 源文件：`optional/AnneHappy/witch_glow.sp`
- myinfo：author=fdxx

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `l4d2_witch_glow_version` | `VERSION` | 字符串或表达式 | 无上下界 | 插件版本 | 服务器端本插件逻辑；不写入 cfg 存档 | `witch_glow.sp:23` |
| `l4d2_witch_glow_min_range` | `500` | 整数 | 无上下界 | 发光最小距离 | 服务器端本插件逻辑 | `witch_glow.sp:25` |
| `l4d2_witch_glow_max_range` | `2000` | 整数 | 无上下界 | 发光最大距离 | 服务器端本插件逻辑 | `witch_glow.sp:26` |

## Witch on incap. —— `optional/AnneHappy/witch_on_incap.sp`

- 源文件：`optional/AnneHappy/witch_on_incap.sp`
- myinfo：version=1.1，author=epilimic, morzlee
- myinfo description（源码原文）：Spawns a witch anytime someone goes down!

### 指令

（本插件未注册指令）

### ConVar

（本插件未注册 ConVar）

## [L4D2]Zombie Spawn Fix —— `optional/AnneHappy/zombie_spawn_fix.sp`

- 源文件：`optional/AnneHappy/zombie_spawn_fix.sp`
- myinfo：version=1.0.9，author=sorallll & Psyk0tik (Crasher_3637)
- myinfo description（源码原文）：Fixed Special Inected and Player Zombie spawning failures in some cases

### 指令

（本插件未注册指令）

### ConVar

（本插件未注册 ConVar）

## Melee In The Saferoom —— `optional/MeleeInTheSafeRoom.sp`

- 源文件：`optional/MeleeInTheSafeRoom.sp`
- myinfo：author=N3wton
- myinfo description（源码原文）：Spawns a selection of melee weapons in the saferoom, at the start of each round.

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_melee` | 无参数 | 需要 ADMFLAG_KICK（c，踢出） | 列出当前战役可刷出的所有近战武器 | `MeleeInTheSafeRoom.sp:92` |

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `l4d2_MITSR_Version` | `VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（默认值取自 VERSION 宏） | 服务器端本插件逻辑 | `MeleeInTheSafeRoom.sp:72` |
| `l4d2_MITSR_Enabled` | `1` | 整数 | 无上下界 | 是否启用插件 | 服务器端本插件逻辑 | `MeleeInTheSafeRoom.sp:73` |
| `l4d2_MITSR_Random` | `1` | 整数 | 无上下界 | 随机刷武器为 1，使用自定义列表为 0 | 服务器端本插件逻辑 | `MeleeInTheSafeRoom.sp:74` |
| `l4d2_MITSR_Amount` | `8` | 整数 | 无上下界 | 当 Random 为 1 时刷出的武器数量 | 服务器端本插件逻辑 | `MeleeInTheSafeRoom.sp:75` |
| `l4d2_MITSR_BaseballBat` | `1` | 整数 | 无上下界 | 刷出的棒球棍数量，需 Random 为 0 | 服务器端本插件逻辑 | `MeleeInTheSafeRoom.sp:76` |
| `l4d2_MITSR_CricketBat` | `1` | 整数 | 无上下界 | 刷出的板球棍数量，需 Random 为 0 | 服务器端本插件逻辑 | `MeleeInTheSafeRoom.sp:77` |
| `l4d2_MITSR_Crowbar` | `1` | 整数 | 无上下界 | 刷出的撬棍数量，需 Random 为 0 | 服务器端本插件逻辑 | `MeleeInTheSafeRoom.sp:78` |
| `l4d2_MITSR_ElecGuitar` | `1` | 整数 | 无上下界 | 刷出的电吉他数量，需 Random 为 0 | 服务器端本插件逻辑 | `MeleeInTheSafeRoom.sp:79` |
| `l4d2_MITSR_FireAxe` | `1` | 整数 | 无上下界 | 刷出的消防斧数量，需 Random 为 0 | 服务器端本插件逻辑 | `MeleeInTheSafeRoom.sp:80` |
| `l4d2_MITSR_FryingPan` | `1` | 整数 | 无上下界 | 刷出的平底锅数量，需 Random 为 0 | 服务器端本插件逻辑 | `MeleeInTheSafeRoom.sp:81` |
| `l4d2_MITSR_GolfClub` | `1` | 整数 | 无上下界 | 刷出的高尔夫球杆数量，需 Random 为 0 | 服务器端本插件逻辑 | `MeleeInTheSafeRoom.sp:82` |
| `l4d2_MITSR_Knife` | `1` | 整数 | 无上下界 | 刷出的刀数量，需 Random 为 0 | 服务器端本插件逻辑 | `MeleeInTheSafeRoom.sp:83` |
| `l4d2_MITSR_Katana` | `1` | 整数 | 无上下界 | 刷出的武士刀数量，需 Random 为 0 | 服务器端本插件逻辑 | `MeleeInTheSafeRoom.sp:84` |
| `l4d2_MITSR_Machete` | `1` | 整数 | 无上下界 | 刷出的砍刀数量，需 Random 为 0 | 服务器端本插件逻辑 | `MeleeInTheSafeRoom.sp:85` |
| `l4d2_MITSR_RiotShield` | `1` | 整数 | 无上下界 | 刷出的防暴盾数量，需 Random 为 0 | 服务器端本插件逻辑 | `MeleeInTheSafeRoom.sp:86` |
| `l4d2_MITSR_Tonfa` | `1` | 整数 | 无上下界 | 刷出的拐棍数量，需 Random 为 0 | 服务器端本插件逻辑 | `MeleeInTheSafeRoom.sp:87` |

## AI Tank Gank —— `optional/aitankgank.sp`

- 源文件：`optional/aitankgank.sp`
- myinfo：version=0.3，author=Stabby
- myinfo description（源码原文）：Kills tanks on pass to AI.

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `tankgank_killoncrash` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 为 0 时若控制坦克的玩家崩溃则坦克不会被处死 | 服务器端本插件逻辑 | `aitankgank.sp:23` |

## L4D2 Auto-pause —— `optional/autopause.sp`

- 源文件：`optional/autopause.sp`
- myinfo：version=2.4.1，author=Darkid, Griffin, StarterX4, Forgetest, J.
- myinfo description（源码原文）：When a player disconnects due to crash, automatically pause the game. When they rejoin, give them a correct spawn timer.

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `autopause_enable` | `1` | 整数 | 无上下界 | 玩家崩溃时是否自动暂停 | 服务器端本插件逻辑 | `autopause.sp:64` |
| `autopause_force` | `0` | 整数 | 无上下界 | 玩家崩溃时是否强制暂停 | 服务器端本插件逻辑 | `autopause.sp:65` |
| `autopause_forceunpause` | `0` | 整数 | 无上下界 | 崩溃玩家重连后是否强制取消暂停 | 服务器端本插件逻辑 | `autopause.sp:66` |
| `autopause_apdebug` | `0` | 整数，源码写作浮点 | 0.0 ~ 3.0 | 调试等级：0 不调试，1 写 SourceMod 日志，2 聊天输出，3 两者 | 服务器端本插件逻辑 | `autopause.sp:67` |

## Blocks heatseeking chargers —— `optional/blockheatseekingchargers.sp`

- 源文件：`optional/blockheatseekingchargers.sp`
- myinfo：author=sheo, A1m`
- myinfo description（源码原文）：Blocks heatseeking chargers

### 指令

（本插件未注册指令）

### ConVar

（本插件未注册 ConVar）

## Block Trolls —— `optional/blocktrolls.sp`

- 源文件：`optional/blocktrolls.sp`
- myinfo：version=2.0.1.2，author=ProdigySim, CanadaRox, darkid
- myinfo description（源码原文）：Prevents calling votes while others are loading

### 指令

（本插件未注册指令）

### ConVar

（本插件未注册 ConVar）

## Boomer Horde Equalizer —— `optional/boomer_horde_equalizer.sp`

- 源文件：`optional/boomer_horde_equalizer.sp`
- myinfo：version=1.5，author=Visor, Jacob, A1m`
- myinfo description（源码原文）：Fixes boomer hordes being different sizes based on wandering commons.

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `boomer_horde_equalizer` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否修复 Boomer 尸潮规模因游荡普感而不同的问题：1 开，0 关 | 服务器端本插件逻辑 | `boomer_horde_equalizer.sp:31` |

## Boomer Horde Equalizer (Refactored) —— `optional/boomer_horde_equalizer_refactored.sp`

- 源文件：`optional/boomer_horde_equalizer_refactored.sp`
- myinfo：version=1.6，author=Visor, Jacob, A1m`, Sir
- myinfo description（源码原文）：Fixes boomer hordes being different sizes based on wandering commons (1.5) as well as adding zombies to the queue rather than relying on max_mob_size

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `boomer_horde_amount` | <被喷人数> <尸潮数量> | 服务器控制台命令，玩家无法使用 | 设置 Boomer 尸潮数量，服务器控制台命令 | `boomer_horde_equalizer_refactored.sp:92` |

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `boomer_horde_equalizer` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 同 1117，重构版插件中的同名 ConVar | 服务器端本插件逻辑 | `boomer_horde_equalizer_refactored.sp:87` |
| `boomer_horde_equalizer_events_default` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 活动尸潮期间是否使用默认 Boomer 行为：1 是，0 覆盖 | 服务器端本插件逻辑 | `boomer_horde_equalizer_refactored.sp:88` |

## Versus Boss Spawn Persuasion —— `optional/bossspawningfix.sp`

- 源文件：`optional/bossspawningfix.sp`
- myinfo：version=1.3，author=ProdigySim
- myinfo description（源码原文）：Makes Versus Boss Spawns obey cvars

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `l4d_obey_boss_spawn_cvars` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否强制 boss 刷新遵守相关 cvar | 服务器端本插件逻辑 | `bossspawningfix.sp:23` |
| `l4d_obey_boss_spawn_except_static` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 在静态坦克刷新地图上是否不覆盖 boss 刷新规则 | 服务器端本插件逻辑 | `bossspawningfix.sp:24` |

## Caster Assister —— `optional/caster_assister.sp`

- 源文件：`optional/caster_assister.sp`
- myinfo：version=2.3.1，author=CanadaRox, Sir, Forgetest
- myinfo description（源码原文）：Allows spectators to control their own specspeed and move vertically

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_set_specspeed_multi` | <倍数> | 无（任意玩家可用） | 设置旁观速度倍率，默认 1.0 | `caster_assister.sp:27` |
| `sm_set_specspeed_increment` | <增量> | 无（任意玩家可用） | 设置旁观速度的递增步长，默认 0.1 | `caster_assister.sp:28` |
| `sm_increase_specspeed` | 无参数 | 无（任意玩家可用） | 提高旁观速度 | `caster_assister.sp:29` |
| `sm_decrease_specspeed` | 无参数 | 无（任意玩家可用） | 降低旁观速度 | `caster_assister.sp:30` |
| `sm_set_vertical_increment` | <增量> | 无（任意玩家可用） | 设置垂直视角递增步长，默认 450.0 | `caster_assister.sp:31` |

### ConVar

（本插件未注册 ConVar）

## L4D2 Caster System (Original built in readyup) —— `optional/caster_system.sp`

- 源文件：`optional/caster_system.sp`
- myinfo：author=CanadaRox, Forgetest
- myinfo description（源码原文）：Standalone caster handler.

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_caster` | <玩家> | 需要 ADMFLAG_BAN（d，封禁） | 把指定玩家注册为解说 | `caster_system.sp:58` |
| `sm_resetcasters` | 无参数 | 需要 ADMFLAG_BAN（d，封禁） | 重置解说名单，适合放在 confogl_off.cfg 中 | `caster_system.sp:59` |
| `sm_add_caster_id` | <SteamID> | 需要 ADMFLAG_BAN（d，封禁） | 把解说加入白名单，即允许自助注册的名单 | `caster_system.sp:60` |
| `sm_remove_caster_id` | <SteamID> | 需要 ADMFLAG_BAN（d，封禁） | 把解说从白名单移除 | `caster_system.sp:61` |
| `sm_printcasters` | 无参数 | 需要 ADMFLAG_BAN（d，封禁） | 打印解说白名单 | `caster_system.sp:62` |
| `sm_cast` | 无参数 | 无（任意玩家可用） | 把调用者自己注册为解说 | `caster_system.sp:63` |
| `sm_notcasting` | [玩家] | 无（任意玩家可用） | 取消自己或指定玩家的解说身份 | `caster_system.sp:64` |
| `sm_uncast` | [玩家] | 无（任意玩家可用） | sm_notcasting 的别名 | `caster_system.sp:65` |
| `sm_kickspecs` | 无参数 | 无（任意玩家可用） | 发起投票踢出旁观的解说 | `caster_system.sp:68` |

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `caster_disable_addons` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否禁止解说员使用插件 | 服务器端本插件逻辑 | `caster_system.sp:54` |

## Config Description —— `optional/cfg_motd.sp`

- 源文件：`optional/cfg_motd.sp`
- myinfo：version=0.2.1，author=Visor
- myinfo description（源码原文）：Displays a descriptive MOTD on desire

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_changelog` | 无参数 | 无（任意玩家可用） | 显示描述当前配置的 MOTD 页面 | `cfg_motd.sp:23` |
| `sm_cfg` | 无参数 | 无（任意玩家可用） | sm_changelog 的别名 | `cfg_motd.sp:24` |

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `sm_cfgmotd_title` | `ZoneMod` | 字符串或表达式 | 无上下界 | 自定义 MOTD 标题 | 服务器端本插件逻辑 | `cfg_motd.sp:20` |
| `sm_cfgmotd_url` | `https://github.com/SirPlease/ZoneMod/blob/master/README.md` | 字符串或表达式 | 无上下界 | 自定义 MOTD 页面 URL | 服务器端本插件逻辑 | `cfg_motd.sp:21` |

## L4D2 Change Log Command —— `optional/changelog.sp`

- 源文件：`optional/changelog.sp`
- myinfo：version=3.0.6，author=Spoon
- myinfo description（源码原文）：Does things :)

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_changelog` | 无参数 | 无（任意玩家可用） | 显示更新日志页面，源码未给描述 | `changelog.sp:21` |

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `l4d2_cl_link` | `https://github.com/spoon-l4d2/NextMod` | 字符串或表达式 | 无上下界 | 更新日志页面的链接 | 服务器端本插件逻辑 | `changelog.sp:20` |

## Incapped Charger Damage —— `optional/charger_incap_damage.sp`

- 源文件：`optional/charger_incap_damage.sp`
- myinfo：version=2.1.0，author=Sir, A1m`
- myinfo description（源码原文）：Modify Charger pummel damage done to Survivors

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `charger_dmg_incapped` | `-1.0` | 整数，源码写作浮点 | 无上下界 | 对倒地生还者的冲锋伤害 | 服务器端本插件逻辑 | `charger_incap_damage.sp:39` |

## Checkpoint Rage Control —— `optional/checkpoint-rage-control.sp`

- 源文件：`optional/checkpoint-rage-control.sp`
- myinfo：version=0.3.2，author=ProdigySim, Visor
- myinfo description（源码原文）：Enable tank to lose rage while survivors are in saferoom

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `saferoom_frustration_tickdown` | <tick> | 服务器控制台命令，玩家无法使用 | 设置安全区挫败感倒计时，服务器控制台命令 | `checkpoint-rage-control.sp:74` |

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `crc_global` | `0` | 整数 | 无上下界 | 是否默认移除所有地图的安全区挫败感保留机制 | 服务器端本插件逻辑 | `checkpoint-rage-control.sp:70` |
| `crc_debug` | `0` | 整数，源码写作浮点 | 0.0 ~ 3.0 | 调试等级：0 关闭，1 启用，2 仅聊天，3 仅控制台 | 服务器端本插件逻辑 | `checkpoint-rage-control.sp:71` |

## Christmas Surprise —— `optional/christmas_surprise.sp`

- 源文件：`optional/christmas_surprise.sp`
- myinfo：version=1.1，author=Jacob
- myinfo description（源码原文）：Happy Holidays

### 指令

（本插件未注册指令）

### ConVar

（本插件未注册 ConVar）

## Code patcher —— `optional/code_patcher.sp`

- 源文件：`optional/code_patcher.sp`
- myinfo：version=1.1.1，author=Jahze?, A1m`
- myinfo description（源码原文）：Code patcher

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `codepatch_list` | 无参数 | 服务器控制台命令，玩家无法使用 | 列出已应用的代码补丁 | `code_patcher.sp:54` |
| `codepatch_patch` | <补丁名> <参数> | 服务器控制台命令，玩家无法使用 | 应用指定代码补丁，服务器控制台命令 | `code_patcher.sp:55` |
| `codepatch_unpatch` | <补丁名> <参数> | 服务器控制台命令，玩家无法使用 | 撤销指定代码补丁，服务器控制台命令 | `code_patcher.sp:56` |

### ConVar

（本插件未注册 ConVar）

## Coinflip —— `optional/coinflip.sp`

- 源文件：`optional/coinflip.sp`
- myinfo：version=1.0.2，author=purpletreefactory, epilimic
- myinfo description（源码原文）：purpletreefactory's version of coinflip

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_coinflip` | 无参数 | 无（任意玩家可用） | 投硬币并在聊天中公布结果 | `coinflip.sp:45` |
| `sm_cf` | 无参数 | 无（任意玩家可用） | sm_coinflip 的别名 | `coinflip.sp:46` |
| `sm_flip` | 无参数 | 无（任意玩家可用） | sm_coinflip 的别名 | `coinflip.sp:47` |
| `sm_roll` | <最小> <最大> | 无（任意玩家可用） | 在指定范围内随机掷点数 | `coinflip.sp:48` |
| `sm_picknumber` | <数字> <范围> | 无（任意玩家可用） | 猜数字玩法，随机开奖 | `coinflip.sp:49` |

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `coinflip_delay` | `-1` | 整数 | 无上下界 | 两次允许投硬币之间的延迟秒数，-1 表示无延迟 | 服务器端本插件逻辑 | `coinflip.sp:43` |

## L4D2 Survivor Progress —— `optional/current.sp`

- 源文件：`optional/current.sp`
- myinfo：version=2.0.7，author=CanadaRox, Visor
- myinfo description（源码原文）：Print survivor progress in flow percents

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_cur` | 无参数 | 无（任意玩家可用） | 显示当前地图的生还者进度 | `current.sp:26` |
| `sm_current` | 无参数 | 无（任意玩家可用） | sm_cur 的别名 | `current.sp:27` |

### ConVar

（本插件未注册 ConVar）

## Despawn Health —— `optional/despawn_health.sp`

- 源文件：`optional/despawn_health.sp`
- myinfo：version=1.3.2，author=Jacob
- myinfo description（源码原文）：Gives Special Infected health back when they despawn.

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `si_restore_ratio` | `0.5` | 浮点 | ≤ 1.0 | 应恢复玩家多少已损失生命，0 或负值表示关闭，1.0 表示补满 | 服务器端本插件逻辑 | `despawn_health.sp:22` |

## EQ2 Finale Tank Manager —— `optional/eq_finale_tanks.sp`

- 源文件：`optional/eq_finale_tanks.sp`
- myinfo：version=2.5.2，author=Visor, Electr0
- myinfo description（源码原文）：Either two event tanks or one flow and one (second) event tank

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `tank_map_flow_and_second_event` | <地图名> | 服务器控制台命令，玩家无法使用 | 标记该地图坦克出现于第二事件，服务器控制台命令 | `eq_finale_tanks.sp:71` |
| `tank_map_only_first_event` | <地图名> | 服务器控制台命令，玩家无法使用 | 标记该地图坦克只出现于第一事件，服务器控制台命令 | `eq_finale_tanks.sp:72` |

### ConVar

（本插件未注册 ConVar）

## Finale Even-Numbered Tank Blocker —— `optional/finale_tank_blocker.sp`

- 源文件：`optional/finale_tank_blocker.sp`
- myinfo：version=2.1，author=Stabby, Visor
- myinfo description（源码原文）：Blocks even-numbered non-flow finale tanks.

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `finale_tank_default` | <地图名> | 服务器控制台命令，玩家无法使用 | 设置该地图的终局坦克默认行为，服务器控制台命令 | `finale_tank_blocker.sp:27` |

### ConVar

（本插件未注册 ConVar）

## L4D2 Finale Incap Distance Fixifier —— `optional/finalefix.sp`

- 源文件：`optional/finalefix.sp`
- myinfo：version=1.0.2，author=CanadaRox
- myinfo description（源码原文）：Kills survivors before the score is calculated so you don't get full distance if you are incapped as the rescue vehicle leaves.

### 指令

（本插件未注册指令）

### ConVar

（本插件未注册 ConVar）

## [L4D & L4D2] Engine Fix —— `optional/fix_engine.sp`

- 源文件：`optional/fix_engine.sp`
- myinfo：author=raziEiL [disawar1]
- myinfo description（源码原文）：Blocking ladder speed glitch, no fall damage bug, health boost glitch.
- HookConVarChange：`hCvarDecayRate`→`OnConvarChange_DecayRate`、`hCvarWarnEnabled`→`OnConvarChange_WarnEnabled`、`hCvarEngineFlags`→`OnConvarChange_EngineFlags`

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `debug` | 无参数 | 无（任意玩家可用） | 显示加载提示并临时启用调试输出 | `fix_engine.sp:74` |

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `engine_fix_version` | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | 服务器端本插件逻辑；复制到客户端 | `fix_engine.sp:56` |
| `engine_warning` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否显示玩家正在使用漏洞的警告：1 启用，0 关闭 | 服务器端本插件逻辑 | `fix_engine.sp:58` |
| `engine_fix_flags` | `14` | 整数，源码写作浮点 | 0.0 ~ 30.0 | 要修复或阻止的漏洞类型位标志，可相加：0 关闭，2 梯子加速漏洞，4 无摔落伤害，其余被截断 | 服务器端本插件逻辑 | `fix_engine.sp:59` |

## Ghost Hurt Management —— `optional/ghost_hurt.sp`

- 源文件：`optional/ghost_hurt.sp`
- myinfo：version=1.1，author=Jacob
- myinfo description（源码原文）：Allows for modifications of trigger_hurt_ghost

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_reset_ghost_hurt` | 无参数 | 服务器控制台命令，玩家无法使用 | 重置 trigger_hurt_ghost，适合放在 confogl_off.cfg 中，服务器控制台命令 | `ghost_hurt.sp:30` |

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `ghost_hurt_type` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 何时启用 trigger_hurt_ghost：0 从不，1 回合开始时 | 服务器端本插件逻辑 | `ghost_hurt.sp:26` |

## Holdout Bonus —— `optional/holdout_bonus.sp`

- 源文件：`optional/holdout_bonus.sp`
- myinfo：version=0.1.1，author=Tabun
- myinfo description（源码原文）：Gives bonus for (partially) surviving holdout/camping events. (Requires penalty_bonus.)
- HookConVarChange：`g_hCvarKeyValuesPath`→`ConvarChange_KeyValuesPath`

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_hbonus` | 无参数 | 无（任意玩家可用） | 显示当前 Holdout 奖励 | `holdout_bonus.sp:147` |

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `sm_hbonus_report` | `2` | 整数，源码写作浮点 | ≥ 0.0 | 奖励播报方式；源码其实已给描述（`optional/holdout_bonus.sp:123-128` 第 3 参数）：「The way the bonus is reported. 0: no report; 1: report only on round end; 2: also report after event; 3: also report when event starts; 4: only report on event end」——0=不播报、1=仅回合结束时播报、2=事件结束后也播报、3=事件开始时也播报、4=仅在事件结束时播报；同处 :125 的内联注释写的是「0: disable; 1: leave distance unchanged; 2: substract points from distance」，与下一条 `sm_hbonus_pointsmode` 的注释重复，疑为复制残留，以正式描述为准（置信度：高） | 服务器端本插件逻辑 | `holdout_bonus.sp:123` |
| `sm_hbonus_pointsmode` | `2` | 整数，源码写作浮点 | ≥ 0.0 | 奖励计算方式；源码其实已给描述（`optional/holdout_bonus.sp:130-135` 第 3 参数）：「The way the holdout bonus is awarded. 0: disable; 1: leave distance unchanged; 2: substract points from distance.」——0=禁用、1=距离不变、2=从距离中扣除奖励分；:132 的内联注释与描述一致（置信度：高） | 服务器端本插件逻辑 | `holdout_bonus.sp:130` |
| `sm_hbonus_configpath` | `configs/holdoutmapinfo.txt` | 字符串或表达式 | 无上下界 | 每张地图 Holdout 奖励设置所用的 holdoutmapinfo.txt KeyValues 文件路径 | 服务器端本插件逻辑 | `holdout_bonus.sp:137` |

## L4D2 Antibaiter —— `optional/l4d2_antibaiter.sp`

- 源文件：`optional/l4d2_antibaiter.sp`
- myinfo：version=1.4.0，author=Visor, Sir (assisted by Devilesk), A1m`
- myinfo description（源码原文）：Makes you think twice before attempting to bait that shit

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_regsi` | 无参数 | 无（任意玩家可用） | 立即开始一回合 | `l4d2_antibaiter.sp:93` |

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `l4d2_antibaiter_delay` | `20` | 整数 | 无上下界 | 反挂机算法启动前的延迟秒数 | 服务器端本插件逻辑 | `l4d2_antibaiter.sp:80` |
| `l4d2_antibaiter_horde_timer` | `60` | 整数 | 无上下界 | 到尸潮的倒计时秒数 | 服务器端本插件逻辑 | `l4d2_antibaiter.sp:81` |
| `l4d2_antibaiter_progress` | `0.03` | 浮点 | 无上下界 | 生还者必须取得的最小进度才重置反挂机计时器 | 服务器端本插件逻辑 | `l4d2_antibaiter.sp:82` |
| `l4d2_antibaiter_bile_stop` | `0` | 整数 | 无上下界 | 玩家被 Boomer 喷吐时是否停止计时器 | 服务器端本插件逻辑 | `l4d2_antibaiter.sp:83` |

## Blind Infected —— `optional/l4d2_blind_infected.sp`

- 源文件：`optional/l4d2_blind_infected.sp`
- myinfo：version=1.2.2，author=CanadaRox, ProdigySim, A1m`
- myinfo description（源码原文）：Hides specified weapons from the infected team until they are (possibly) visible to one of the survivors to prevent SI scouting the map

### 指令

（本插件未注册指令）

### ConVar

（本插件未注册 ConVar）

## [L4D2] Block Autoaim —— `optional/l4d2_block_autoaim.sp`

- 源文件：`optional/l4d2_block_autoaim.sp`
- myinfo：author=Sir
- myinfo description（源码原文）：Strips Auto-Aim from the game entirely (disables controller aim-assist + patches an exploit)

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `l4d2_block_autoaim` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否禁用自动瞄准 | 服务器端本插件逻辑 | `l4d2_block_autoaim.sp:38` |

## [L4D2] Block Bot Pills —— `optional/l4d2_block_bot_pills.sp`

- 源文件：`optional/l4d2_block_bot_pills.sp`
- myinfo：version=1.0.1，author=B[R]UTUS
- myinfo description（源码原文）：Prohibits the use of pills to bots

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `l4d2_bbp_debug_enabled` | `0` | 整数 | 无上下界 | 是否开启调试模式 | 服务器端本插件逻辑 | `l4d2_block_bot_pills.sp:33` |

## L4D2 Black&White Rock Hit —— `optional/l4d2_bw_rock_hit.sp`

- 源文件：`optional/l4d2_bw_rock_hit.sp`
- myinfo：version=1.3，author=Visor, A1m`
- myinfo description（源码原文）：Stops rocks from passing through soon-to-be-dead Survivors

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_detonate_rock` | 无参数 | 无（任意玩家可用） | 立即引爆坦克投出的石头 | `l4d2_bw_rock_hit.sp:67` |

### ConVar

（本插件未注册 ConVar）

## Byebye Door —— `optional/l4d2_byedoor.sp`

- 源文件：`optional/l4d2_byedoor.sp`
- myinfo：version=1.2，author=Sir
- myinfo description（源码原文）：Time to kill Saferoom Doors.

### 指令

（本插件未注册指令）

### ConVar

（本插件未注册 ConVar）

## Character Fix —— `optional/l4d2_character_fix.sp`

- 源文件：`optional/l4d2_character_fix.sp`
- myinfo：version=0.2.1，author=someone
- myinfo description（源码原文）：Fixes character change exploit in 1v1, 2v2, 3v3

### 指令

（本插件未注册指令）

### ConVar

（本插件未注册 ConVar）

## L4D2 Collision Adjustments —— `optional/l4d2_collision_adjustments.sp`

- 源文件：`optional/l4d2_collision_adjustments.sp`
- myinfo：version=1.3，author=Sir
- myinfo description（源码原文）：Allows messing with pesky Collisions in Left 4 Dead 2

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `collision_tankrock_common` | `1` | 整数 | 无上下界 | 石头是否会穿过普通感染者并击杀它们，而不是被卡住 | 服务器端本插件逻辑 | `l4d2_collision_adjustments.sp:42` |
| `collision_smoker_common` | `0` | 整数 | 无上下界 | 被拉的生还者是否会穿过普通感染者 | 服务器端本插件逻辑 | `l4d2_collision_adjustments.sp:43` |
| `collision_tankrock_incap` | `0` | 整数 | 无上下界 | 石头是否会穿过倒地的生还者 | 服务器端本插件逻辑 | `l4d2_collision_adjustments.sp:44` |

## Director-scripted common limit blocker —— `optional/l4d2_director_commonlimit_block.sp`

- 源文件：`optional/l4d2_director_commonlimit_block.sp`
- myinfo：version=0.4，author=Tabun
- myinfo description（源码原文）：Prevents director scripted overrides of z_common_limit. Only affects scripted common limits higher than the cvar.
- HookConVarChange：`hCommonLimit`→`Cvar_CommonLimitChange`

### 指令

（本插件未注册指令）

### ConVar

（本插件未注册 ConVar）

## Dominators Control —— `optional/l4d2_dominatorscontrol.sp`

- 源文件：`optional/l4d2_dominatorscontrol.sp`
- myinfo：version=1.2，author=vintik, A1m`
- myinfo description（源码原文）：Changes bIsDominator flag for infected classes. Allows to have native-order quad-caps.

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `l4d2_dominators` | `53` | 整数，源码写作浮点 | 0.0 ~ 63.0 | 哪些特感被视为控制类，位掩码：1 smoker，2 boomer，4 hunter，8 spitter，16 jockey，其余被截断，范围 0~63 | 服务器端本插件逻辑 | `l4d2_dominatorscontrol.sp:38` |

## L4D2 Drop Secondary —— `optional/l4d2_drop_secondary.sp`

- 源文件：`optional/l4d2_drop_secondary.sp`
- myinfo：version=1.1，author=Sir, ProjectSky (Initial Plugin by Jahze & Visor)
- myinfo description（源码原文）：Testing Purposes

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `l4d2_drop_secondary_debug` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否开启调试输出 | 服务器端本插件逻辑 | `l4d2_drop_secondary.sp:48` |

## L4D2 Fireworks Noise Blocker —— `optional/l4d2_fireworks_noise_block.sp`

- 源文件：`optional/l4d2_fireworks_noise_block.sp`
- myinfo：version=0.4，author=Visor
- myinfo description（源码原文）：Focus on SI!

### 指令

（本插件未注册指令）

### ConVar

（本插件未注册 ConVar）

## [L4D2] Proper Sack Order —— `optional/l4d2_fix_spawn_order.sp`

- 源文件：`optional/l4d2_fix_spawn_order.sp`
- myinfo：author=Sir, Forgetest
- myinfo description（源码原文）：Finally fix that pesky spawn rotation not being reliable

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `sackorder_debug` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否开启调试 | 服务器端本插件逻辑；仅服务器；隐藏变量 | `l4d2_fix_spawn_order.sp:64` |

## [L4D2] Merged Get-Up Fixes —— `optional/l4d2_getup_fixes.sp`

- 源文件：`optional/l4d2_getup_fixes.sp`
- myinfo：author=Forgetest
- myinfo description（源码原文）：Fixes all double/missing get-up cases.

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `gfc_long_charger_duration` | `2.2` | 浮点 | 无上下界 | 长冲锋起身动画的神圣帧持续秒数 | 服务器端本插件逻辑 | `l4d2_getup_fixes.sp:156` |
| `charger_keep_wall_charge_animation` | `1` | 整数 | 无上下界 | 是否启用长距离撞墙动画及其神圣帧 | 服务器端本插件逻辑 | `l4d2_getup_fixes.sp:158` |
| `charger_keep_far_charge_animation` | `0` | 整数 | 无上下界 | 是否启用长距离远抛撞地动画及其神圣帧 | 服务器端本插件逻辑 | `l4d2_getup_fixes.sp:159` |

## Stagger Blocker —— `optional/l4d2_getup_slide_fix.sp`

- 源文件：`optional/l4d2_getup_slide_fix.sp`
- myinfo：version=1.4.3，author=Standalone (aka Manu), Visor, Sir, A1m`
- myinfo description（源码原文）：Block players from being staggered by Jockeys and Hunters for a time while getting up from a Hunter pounce & Charger pummel

### 指令

（本插件未注册指令）

### ConVar

（本插件未注册 ConVar）

## Infected Warp —— `optional/l4d2_ghost_warp.sp`

- 源文件：`optional/l4d2_ghost_warp.sp`
- myinfo：version=2.4.2，author=Confogl Team, CanadaRox, A1m`
- myinfo description（源码原文）：Allows infected to warp to survivors

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_warptosurvivor` | <参数> | 无（任意玩家可用） | 把自己传送到随机生还者处即幽灵传送，服务器上不可用 | `l4d2_ghost_warp.sp:74` |
| `sm_warpto` | <参数> | 无（任意玩家可用） | sm_warptosurvivor 的别名 | `l4d2_ghost_warp.sp:75` |
| `sm_warp` | <参数> | 无（任意玩家可用） | sm_warptosurvivor 的别名 | `l4d2_ghost_warp.sp:76` |

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `l4d2_ghost_warp_flag` | `3` | 整数，源码写作浮点 | 0.0 ~ float(eAllowAll) | 启用或禁用幽灵传送：0 禁用，1 通过 sm_warpto 命令启用，2 通过 IN_ATTACK2 按键启用 | 服务器端本插件逻辑 | `l4d2_ghost_warp.sp:56` |
| `l4d2_ghost_warp_delay` | `0.45` | 浮点 | 0.0 ~ 120.0 | 幽灵传送的重用间隔秒数，0.0 表示无延迟，最大 120 | 服务器端本插件逻辑 | `l4d2_ghost_warp.sp:63` |

## L4D2 Godframes Control combined with FF Plugins —— `optional/l4d2_godframes_control_merge.sp`

- 源文件：`optional/l4d2_godframes_control_merge.sp`
- myinfo：version=0.6.10，author=Stabby, CircleSquared, Tabun, Visor, dcx, Sir, Spoon, A1m`, Sir
- myinfo description（源码原文）：Allows for control of what gets godframed and what doesnt along with integrated FF Support from l4d2_survivor_ff (by dcx and Visor) and l4d2_shotgun_ff (by Visor)

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `gfc_godframe_glows` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否改变处于神圣帧的生还者渲染，红色或透明 | 服务器端本插件逻辑 | `l4d2_godframes_control_merge.sp:134` |
| `gfc_hittable_rage_override` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否允许坦克从可击打物命中获得怒气，0 阻止获得 | 服务器端本插件逻辑 | `l4d2_godframes_control_merge.sp:135` |
| `gfc_rock_rage_override` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否允许坦克从神圣帧内的命中获得怒气，0 阻止获得 | 服务器端本插件逻辑 | `l4d2_godframes_control_merge.sp:136` |
| `gfc_hittable_override` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否允许可击打物始终无视神圣帧 | 服务器端本插件逻辑 | `l4d2_godframes_control_merge.sp:137` |
| `gfc_rock_override` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否允许可击打物始终无视神圣帧，同一插件的另一开关 | 服务器端本插件逻辑 | `l4d2_godframes_control_merge.sp:138` |
| `gfc_witch_override` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否允许 Witch 始终无视神圣帧 | 服务器端本插件逻辑 | `l4d2_godframes_control_merge.sp:139` |
| `gfc_ff_min_time` | `0.3` | 浮点 | 0.0 ~ 3.0 | 允许友军伤害前的最短神圣帧时间，范围 0~3 | 服务器端本插件逻辑 | `l4d2_godframes_control_merge.sp:140` |
| `gfc_spit_extra_time` | `0.7` | 浮点 | 0.0 ~ 3.0 | 允许毒痰伤害前的额外神圣帧时间，范围 0~3 | 服务器端本插件逻辑 | `l4d2_godframes_control_merge.sp:141` |
| `gfc_common_extra_time` | `0.0` | 整数，源码写作浮点 | 0.0 ~ 3.0 | 允许普感伤害前的额外神圣帧时间，范围 0~3 | 服务器端本插件逻辑 | `l4d2_godframes_control_merge.sp:142` |
| `gfc_hunter_duration` | `2.1` | 浮点 | 0.0 ~ 3.0 | 飞扑后的神圣帧持续秒数，范围 0~3 | 服务器端本插件逻辑 | `l4d2_godframes_control_merge.sp:143` |
| `gfc_jockey_duration` | `0.0` | 整数，源码写作浮点 | 0.0 ~ 3.0 | 骑乘后的神圣帧持续秒数，范围 0~3 | 服务器端本插件逻辑 | `l4d2_godframes_control_merge.sp:144` |
| `gfc_smoker_duration` | `0.0` | 整数，源码写作浮点 | 0.0 ~ 3.0 | 拉拽或勒喉后的神圣帧持续秒数，范围 0~3 | 服务器端本插件逻辑 | `l4d2_godframes_control_merge.sp:145` |
| `gfc_charger_duration` | `2.1` | 浮点 | 0.0 ~ 3.0 | 冲锋压制后的神圣帧持续秒数，范围 0~3 | 服务器端本插件逻辑 | `l4d2_godframes_control_merge.sp:146` |
| `gfc_charger_stagger_extra_time` | `0.0` | 整数，源码写作浮点 | 0.0 ~ 3.0 | 来自 ChargerFlags 的伤害前的额外神圣帧时间，范围 0~3 | 服务器端本插件逻辑 | `l4d2_godframes_control_merge.sp:147` |
| `gfc_charger_stagger_flags` | `0` | 整数，源码写作浮点 | 0.0 ~ 3.0 | 受额外 Charger 硬直保护时间影响的类型：1 普感，2 毒痰，范围 0~3 | 服务器端本插件逻辑 | `l4d2_godframes_control_merge.sp:148` |
| `gfc_spit_zc_flags` | `6` | 整数，源码写作浮点 | 0.0 ~ 15.0 | 受额外毒痰保护时间影响的特感职业位域：1 Hunter，2 Smoker，4 Jockey，8 Charger，范围 0~15 | 服务器端本插件逻辑 | `l4d2_godframes_control_merge.sp:149` |
| `gfc_common_zc_flags` | `0` | 整数，源码写作浮点 | 0.0 ~ 15.0 | 受额外普感保护时间影响的特感职业位域：1 Hunter，2 Smoker，4 Jockey，8 Charger，范围 0~15 | 服务器端本插件逻辑 | `l4d2_godframes_control_merge.sp:150` |
| `l4d2_undoff_enable` | `7` | 整数 | 无上下界 | 位标志，可相加：1 距离过近，2 Charger 搬运，4 有罪 Bot，7 全部，0 关闭 | 服务器端本插件逻辑 | `l4d2_godframes_control_merge.sp:152` |
| `l4d2_undoff_blockzerodmg` | `7` | 整数 | 无上下界 | 位标志，可相加：屏蔽 0 伤害的友军伤害效果如后坐力与语音统计等，具体位含义源码描述被截断 | 服务器端本插件逻辑 | `l4d2_godframes_control_merge.sp:153` |
| `l4d2_undoff_permdmgfrac` | `1.0` | 开关 0 或 1 | 0.0 ~ 1.0 | 施加到永久生命的最小伤害比例，范围 0~1 | 服务器端本插件逻辑 | `l4d2_godframes_control_merge.sp:154` |
| `l4d2_shotgun_ff_enable` | `1` | 整数，源码写作浮点 | 0.0 ~ 5.0 | 是否启用霰弹枪友军伤害模块，范围 0~5 | 服务器端本插件逻辑 | `l4d2_godframes_control_merge.sp:156` |
| `l4d2_shotgun_ff_multi` | `0.5` | 浮点 | 0.0 ~ 5.0 | 霰弹枪友军伤害的伤害修正值，范围 0~5 | 服务器端本插件逻辑 | `l4d2_godframes_control_merge.sp:157` |
| `l4d2_shotgun_ff_min` | `1.0` | 整数，源码写作浮点 | ≥ 0.0 | 允许的霰弹枪友军伤害下限，0 表示不限制 | 服务器端本插件逻辑 | `l4d2_godframes_control_merge.sp:158` |
| `l4d2_shotgun_ff_max` | `6.0` | 整数，源码写作浮点 | ≥ 0.0 | 允许的霰弹枪友军伤害上限，0 表示不限制 | 服务器端本插件逻辑 | `l4d2_godframes_control_merge.sp:159` |

## L4D2 Hittable Control —— `optional/l4d2_hittable_control.sp`

- 源文件：`optional/l4d2_hittable_control.sp`
- myinfo：version=0.9.3，author=Stabby, Visor, Sir, Derpduck, Forgetest
- myinfo description（源码原文）：Allows for customisation of hittable damage values (and debugging)

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `hc_gauntlet_finale_multiplier` | `0.25` | 浮点 | 0.0 ~ 4.0 | 可击打物在 gauntlet 终局中的伤害倍率，范围 0~4 | 服务器端本插件逻辑 | `l4d2_hittable_control.sp:160` |
| `hc_sflog_standing_damage` | `48.0` | 整数，源码写作浮点 | ≥ -2.0 | swamp fever 原木对未倒地生还者的伤害 | 服务器端本插件逻辑 | `l4d2_hittable_control.sp:163` |
| `hc_bhlog_standing_damage` | `100.0` | 整数，源码写作浮点 | ≥ -2.0 | blood harvest 原木对未倒地生还者的伤害 | 服务器端本插件逻辑 | `l4d2_hittable_control.sp:166` |
| `hc_car_standing_damage` | `100.0` | 整数，源码写作浮点 | ≥ -2.0 | 汽车对未倒地生还者的伤害 | 服务器端本插件逻辑 | `l4d2_hittable_control.sp:169` |
| `hc_bumpercar_standing_damage` | `100.0` | 整数，源码写作浮点 | ≥ -2.0 | 碰碰车对未倒地生还者的伤害 | 服务器端本插件逻辑 | `l4d2_hittable_control.sp:172` |
| `hc_handtruck_standing_damage` | `8.0` | 整数，源码写作浮点 | ≥ -2.0 | 手推车对未倒地生还者的伤害 | 服务器端本插件逻辑 | `l4d2_hittable_control.sp:175` |
| `hc_forklift_standing_damage` | `100.0` | 整数，源码写作浮点 | ≥ -2.0 | 叉车对未倒地生还者的伤害 | 服务器端本插件逻辑 | `l4d2_hittable_control.sp:178` |
| `hc_broken_forklift_standing_damage` | `100.0` | 整数，源码写作浮点 | ≥ -2.0 | 损坏的叉车对未倒地生还者的伤害 | 服务器端本插件逻辑 | `l4d2_hittable_control.sp:181` |
| `hc_dumpster_standing_damage` | `100.0` | 整数，源码写作浮点 | ≥ -2.0 | 垃圾箱对未倒地生还者的伤害 | 服务器端本插件逻辑 | `l4d2_hittable_control.sp:184` |
| `hc_haybale_standing_damage` | `48.0` | 整数，源码写作浮点 | ≥ -2.0 | 干草捆对未倒地生还者的伤害 | 服务器端本插件逻辑 | `l4d2_hittable_control.sp:187` |
| `hc_baggage_standing_damage` | `48.0` | 整数，源码写作浮点 | ≥ -2.0 | 行李车对未倒地生还者的伤害 | 服务器端本插件逻辑 | `l4d2_hittable_control.sp:190` |
| `hc_generator_trailer_standing_damage` | `48.0` | 整数，源码写作浮点 | ≥ -2.0 | 发电拖车对未倒地生还者的伤害 | 服务器端本插件逻辑 | `l4d2_hittable_control.sp:193` |
| `hc_militia_rock_standing_damage` | `100.0` | 整数，源码写作浮点 | ≥ -2.0 | 民兵石头对未倒地生还者的伤害 | 服务器端本插件逻辑 | `l4d2_hittable_control.sp:196` |
| `hc_sofa_chair_standing_damage` | `100.0` | 整数，源码写作浮点 | ≥ -2.0 | Blood Harvest 终局沙发对未倒地生还者的伤害，仅对带目标跟踪的沙发生效 | 服务器端本插件逻辑 | `l4d2_hittable_control.sp:199` |
| `hc_atlas_ball_standing_damage` | `100.0` | 整数，源码写作浮点 | ≥ -2.0 | atlas 球对未倒地生还者的伤害 | 服务器端本插件逻辑 | `l4d2_hittable_control.sp:202` |
| `hc_ibeam_standing_damage` | `48.0` | 整数，源码写作浮点 | ≥ -2.0 | 工字钢对未倒地生还者的伤害 | 服务器端本插件逻辑 | `l4d2_hittable_control.sp:205` |
| `hc_brick_pallets_standing_damage` | `13.0` | 整数，源码写作浮点 | ≥ -2.0 | 砖托盘碎块对未倒地生还者的伤害 | 服务器端本插件逻辑 | `l4d2_hittable_control.sp:208` |
| `hc_boat_smash_standing_damage` | `23.0` | 整数，源码写作浮点 | ≥ -2.0 | 船只碎片对未倒地生还者的伤害 | 服务器端本插件逻辑 | `l4d2_hittable_control.sp:211` |
| `hc_concrete_piller_standing_damage` | `8.0` | 整数，源码写作浮点 | ≥ -2.0 | 混凝土柱碎片对未倒地生还者的伤害 | 服务器端本插件逻辑 | `l4d2_hittable_control.sp:214` |
| `hc_diescraper_ball_standing_damage` | `100.0` | 整数，源码写作浮点 | ≥ -2.0 | Diescraper 终局球形雕像对未倒地生还者的伤害 | 服务器端本插件逻辑 | `l4d2_hittable_control.sp:217` |
| `hc_van_standing_damage` | `100.0` | 整数，源码写作浮点 | ≥ -2.0 | Detour Ahead 第二关面包车对未倒地生还者的伤害 | 服务器端本插件逻辑 | `l4d2_hittable_control.sp:220` |
| `hc_incap_standard_damage` | `100` | 整数，源码写作浮点 | ≥ -2.0 | 所有可击打物对倒地玩家的伤害，-1 表示沿用 Valve 默认倒地伤害 | 服务器端本插件逻辑 | `l4d2_hittable_control.sp:223` |
| `hc_disable_self_damage` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 若开启，坦克不会用可击打物伤害自己 | 服务器端本插件逻辑 | `l4d2_hittable_control.sp:226` |
| `hc_overhit_time` | `1.2` | 浮点 | ≥ 0.0 | 允许同一可击打物连续命中前的等待秒数 | 服务器端本插件逻辑 | `l4d2_hittable_control.sp:229` |
| `hc_debug` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 0 关闭调试，1 启用调试 | 服务器端本插件逻辑 | `l4d2_hittable_control.sp:232` |
| `hc_unbreakable_forklifts` | `0` | 整数，源码写作浮点 | ≥ 0.0 | 是否阻止叉车被坦克击中后碎成多块 | 服务器端本插件逻辑 | `l4d2_hittable_control.sp:235` |
| `hc_phys_mass_incap_threshold` | `500.0` | 整数，源码写作浮点 | ≥ 0.0 | 重量超过该质量的可击打物命中时会击倒生还者，0.0 关闭 | 服务器端本插件逻辑 | `l4d2_hittable_control.sp:238` |

## L4D2 Horde —— `optional/l4d2_horde.sp`

- 源文件：`optional/l4d2_horde.sp`
- myinfo：version=1.3，author=Visor, Sir, A1m`
- myinfo description（源码原文）：Modifies Event Horde sizes and stops it completely during Tank

### 指令

（本插件未注册指令）

### ConVar

（本插件未注册 ConVar）

## L4D2 Horde Equaliser —— `optional/l4d2_horde_equaliser.sp`

- 源文件：`optional/l4d2_horde_equaliser.sp`
- myinfo：version=3.0.10，author=Visor (original idea by Sir), A1m`
- myinfo description（源码原文）：Make certain event hordes finite

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `l4d2_heq_no_tank_horde` | `0` | 整数 | 无上下界 | 是否在坦克战期间让无限尸潮保持暂停 | 服务器端本插件逻辑 | `l4d2_horde_equaliser.sp:76` |
| `l4d2_heq_checkpoint_sound` | `1` | 整数 | 无上下界 | 是否在检查点播放来袭尸潮声音，每击杀总量的四分之一触发一次，模拟 L4D1 行为 | 服务器端本插件逻辑 | `l4d2_horde_equaliser.sp:77` |

## [L4D2] No Hunter Deadstops —— `optional/l4d2_hunter_no_deadstops.sp`

- 源文件：`optional/l4d2_hunter_no_deadstops.sp`
- myinfo：version=1.0.3，author=Spoon, Luckylock, A1m`
- myinfo description（源码原文）：Prevents deadstops but allows m2s on standing hunters

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `hunter_ground_m2_godframes` | `0.75` | 浮点 | 0.0 ~ 1.0 | hunter 落地后的 m2 神圣帧秒数，范围 0~1 | 服务器端本插件逻辑 | `l4d2_hunter_no_deadstops.sp:35` |

## L4D2 Scoremod+ —— `optional/l4d2_hybrid_scoremod.sp`

- 源文件：`optional/l4d2_hybrid_scoremod.sp`
- myinfo：version=2.2.5，author=Visor
- myinfo description（源码原文）：The next generation scoring mod
- HookConVarChange：`hCvarBonusPerSurvivorMultiplier`→`CvarChanged`、`hCvarPermanentHealthProportion`→`CvarChanged`

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_health` | 无参数 | 无（任意玩家可用） | 源码未给描述；据 `optional/l4d2_hybrid_scoremod.sp:96-98` 三个命令全部注册到同一回调 `CmdBonus`（:209-246），该回调先取 `GetCmdArg(1)`，按 `full`（明细）/`lite`（单行）/无参（默认百分比）三种详略输出本队回血加成 HB、伤害加成 DB、药片加成 Pills 及其占上限百分比；回合结束后或由服务器控制台调用则不显示，推测为：“查看本队当前混合增分”的别名之一，与 `sm_damage`、`sm_bonus` 完全等价（命名分别对应加成的三个组成部分）（置信度：高） | `l4d2_hybrid_scoremod.sp:96` |
| `sm_damage` | 无参数 | 无（任意玩家可用） | 源码未给描述；据 `optional/l4d2_hybrid_scoremod.sp:96-98` 三个命令全部注册到同一回调 `CmdBonus`（:209-246），该回调先取 `GetCmdArg(1)`，按 `full`（明细）/`lite`（单行）/无参（默认百分比）三种详略输出本队回血加成 HB、伤害加成 DB、药片加成 Pills 及其占上限百分比；回合结束后或由服务器控制台调用则不显示，推测为：“查看本队当前混合增分”的别名之一，与 `sm_damage`、`sm_bonus` 完全等价（命名分别对应加成的三个组成部分）（置信度：高） | `l4d2_hybrid_scoremod.sp:97` |
| `sm_bonus` | 无参数 | 无（任意玩家可用） | 源码未给描述；据 `optional/l4d2_hybrid_scoremod.sp:96-98` 三个命令全部注册到同一回调 `CmdBonus`（:209-246），该回调先取 `GetCmdArg(1)`，按 `full`（明细）/`lite`（单行）/无参（默认百分比）三种详略输出本队回血加成 HB、伤害加成 DB、药片加成 Pills 及其占上限百分比；回合结束后或由服务器控制台调用则不显示，推测为：“查看本队当前混合增分”的别名之一，与 `sm_damage`、`sm_bonus` 完全等价（命名分别对应加成的三个组成部分）（置信度：高） | `l4d2_hybrid_scoremod.sp:98` |
| `sm_mapinfo` | 无参数 | 无（任意玩家可用） | 源码未给描述；据 `optional/l4d2_hybrid_scoremod.sp:99` 注册到 `CmdMapInfo`（:248-267，回调体逐条 `CPrintToChat`）：输出队伍规模、地图距离、总分（含药片上限）、生命加成/伤害加成/药片加成各自上限与占比、平局加分，数据来自 `OnConfigsExecuted`（:119-139）中按 `sm2_bonus_per_survivor_multiplier`×人数×地图距离算出的 `fMapBonus` 等全局量，推测为：显示本张地图 ScoreMod 2 增分配置的说明命令（置信度：高） | `l4d2_hybrid_scoremod.sp:99` |

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `sm2_bonus_per_survivor_multiplier` | `0.5` | 浮点 | 无上下界 | 生还者奖励总分等于该值乘以生还者人数再乘以地图距离 | 服务器端本插件逻辑 | `l4d2_hybrid_scoremod.sp:79` |
| `sm2_permament_health_proportion` | `0.75` | 浮点 | 无上下界 | 永久生命奖励比例，其余计入临时生命奖励 | 服务器端本插件逻辑 | `l4d2_hybrid_scoremod.sp:80` |
| `sm2_pills_hp_factor` | `6.0` | 整数，源码写作浮点 | 无上下界 | 未使用止痛药折算的血量等于地图奖励血量除以该值 | 服务器端本插件逻辑 | `l4d2_hybrid_scoremod.sp:81` |
| `sm2_pills_max_bonus` | `30` | 整数 | 无上下界 | 未使用止痛药折算奖励的上限 | 服务器端本插件逻辑 | `l4d2_hybrid_scoremod.sp:82` |

## L4D2 Scoremod+ —— `optional/l4d2_hybrid_scoremod_zone.sp`

- 源文件：`optional/l4d2_hybrid_scoremod_zone.sp`
- myinfo：version=2.2.6，author=Visor, Sir
- myinfo description（源码原文）：The next generation scoring mod
- HookConVarChange：`hCvarBonusPerSurvivorMultiplier`→`CvarChanged`、`hCvarPermanentHealthProportion`→`CvarChanged`

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_health` | 无参数 | 无（任意玩家可用） | 源码未给描述；据 `optional/l4d2_hybrid_scoremod_zone.sp:97-100` 三个命令全部注册到同一回调 `CmdBonus`（:210-247），该回调先取 `GetCmdArg(1)`，按 `full`（明细）/`lite`（单行）/无参（默认百分比）三种详略输出本队回血加成 HB、伤害加成 DB、药片加成 Pills 及其占上限百分比；回合结束后或由服务器控制台调用则不显示，推测为：“查看本队当前混合增分”的别名之一，与 `sm_damage`、`sm_bonus` 完全等价（命名分别对应加成的三个组成部分）（置信度：高） | `l4d2_hybrid_scoremod_zone.sp:97` |
| `sm_damage` | 无参数 | 无（任意玩家可用） | 源码未给描述；据 `optional/l4d2_hybrid_scoremod_zone.sp:97-100` 三个命令全部注册到同一回调 `CmdBonus`（:210-247），该回调先取 `GetCmdArg(1)`，按 `full`（明细）/`lite`（单行）/无参（默认百分比）三种详略输出本队回血加成 HB、伤害加成 DB、药片加成 Pills 及其占上限百分比；回合结束后或由服务器控制台调用则不显示，推测为：“查看本队当前混合增分”的别名之一，与 `sm_damage`、`sm_bonus` 完全等价（命名分别对应加成的三个组成部分）（置信度：高） | `l4d2_hybrid_scoremod_zone.sp:98` |
| `sm_bonus` | 无参数 | 无（任意玩家可用） | 源码未给描述；据 `optional/l4d2_hybrid_scoremod_zone.sp:97-100` 三个命令全部注册到同一回调 `CmdBonus`（:210-247），该回调先取 `GetCmdArg(1)`，按 `full`（明细）/`lite`（单行）/无参（默认百分比）三种详略输出本队回血加成 HB、伤害加成 DB、药片加成 Pills 及其占上限百分比；回合结束后或由服务器控制台调用则不显示，推测为：“查看本队当前混合增分”的别名之一，与 `sm_damage`、`sm_bonus` 完全等价（命名分别对应加成的三个组成部分）（置信度：高） | `l4d2_hybrid_scoremod_zone.sp:99` |
| `sm_mapinfo` | 无参数 | 无（任意玩家可用） | 源码未给描述；据 `optional/l4d2_hybrid_scoremod_zone.sp:100` 注册到 `CmdMapInfo`（:249-268，回调体逐条 `CPrintToChat`）：输出队伍规模、地图距离、总分（含药片上限）、生命加成/伤害加成/药片加成各自上限与占比、平局加分，数据来自 `OnConfigsExecuted`（:120-140）中按 `sm2_bonus_per_survivor_multiplier`×人数×地图距离算出的 `fMapBonus` 等全局量，推测为：显示本张地图 ScoreMod 2 增分配置的说明命令（置信度：高） | `l4d2_hybrid_scoremod_zone.sp:100` |

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `sm2_bonus_per_survivor_multiplier` | `0.5` | 浮点 | 无上下界 | 与 1209 同义，另一文件中的同名 ConVar | 服务器端本插件逻辑 | `l4d2_hybrid_scoremod_zone.sp:78` |
| `sm2_permament_health_proportion` | `0.75` | 浮点 | 无上下界 | 与 1210 同义，另一文件中的同名 ConVar | 服务器端本插件逻辑 | `l4d2_hybrid_scoremod_zone.sp:79` |
| `sm2_pills_hp_factor` | `6.0` | 整数，源码写作浮点 | 无上下界 | 与 1211 同义，另一文件中的同名 ConVar | 服务器端本插件逻辑 | `l4d2_hybrid_scoremod_zone.sp:80` |
| `sm2_pills_max_bonus` | `30` | 整数 | 无上下界 | 与 1212 同义，另一文件中的同名 ConVar | 服务器端本插件逻辑 | `l4d2_hybrid_scoremod_zone.sp:81` |

## L4D2 Jockey Skeet —— `optional/l4d2_jockey_skeet.sp`

- 源文件：`optional/l4d2_jockey_skeet.sp`
- myinfo：version=1.4，author=Visor, A1m`
- myinfo description（源码原文）：A dream come true

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `z_leap_damage_interrupt` | `195.0` | 整数，源码写作浮点 | 10.0 ~ 325.0 | 受到该伤害量会打断跳跃尝试，范围 10~325 | 服务器端本插件逻辑 | `l4d2_jockey_skeet.sp:44` |
| `jockey_skeet_report` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否在聊天中播报 Jockey 被击杀 | 服务器端本插件逻辑 | `l4d2_jockey_skeet.sp:45` |

## Ladder Rambos Dhooks [Merged] —— `optional/l4d2_ladder_rambos.sp`

- 源文件：`optional/l4d2_ladder_rambos.sp`
- myinfo：author=$atanic $pirit, Lux, Forgetest
- myinfo description（源码原文）：Allows players to shoot from Ladders

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `cssladders_enabled` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 生还者能否在梯子上射击：1 允许，0 禁止 | 服务器端本插件逻辑；仅服务器 | `l4d2_ladder_rambos.sp:112` |
| `cssladders_allow_m2` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否允许在梯子上推击：1 允许，0 禁止 | 服务器端本插件逻辑；仅服务器 | `l4d2_ladder_rambos.sp:119` |
| `cssladders_allow_reload` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否允许在梯子上换弹：1 允许，0 禁止 | 服务器端本插件逻辑；仅服务器 | `l4d2_ladder_rambos.sp:126` |
| `cssladders_allow_shotgun_reload` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否允许在梯子上给霰弹枪换弹：1 允许，0 禁止 | 服务器端本插件逻辑；仅服务器 | `l4d2_ladder_rambos.sp:133` |
| `cssladders_allow_switch` | `1` | 整数，源码写作浮点 | 0.0 ~ 2.0 | 是否允许在梯子上切换物品：2 全部允许，1 仅枪械间切换，0 禁止 | 服务器端本插件逻辑；仅服务器 | `l4d2_ladder_rambos.sp:140` |
| `cssladders_reduce_recoil` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否允许在梯子上射击时降低后坐力：1 允许，0 禁止 | 服务器端本插件逻辑；仅服务器 | `l4d2_ladder_rambos.sp:147` |

## L4D2 Ledge Blocker —— `optional/l4d2_ledgeblock.sp`

- 源文件：`optional/l4d2_ledgeblock.sp`
- myinfo：version=1.0，author=ProdigySim, Estoopi, Jacob, Visor, A1m`
- myinfo description（源码原文）：Blocks ledge hanging on various maps

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `ledge_block_square` | <参数> | 服务器控制台命令，玩家无法使用 | 设置边缘阻挡方块，服务器控制台命令 | `l4d2_ledgeblock.sp:34` |
| `ledge_remove_block_square` | <参数> | 服务器控制台命令，玩家无法使用 | 移除边缘阻挡方块，服务器控制台命令 | `l4d2_ledgeblock.sp:35` |

### ConVar

（本插件未注册 ConVar）

## L4D2 M2 Control —— `optional/l4d2_m2_control_eq.sp`

- 源文件：`optional/l4d2_m2_control_eq.sp`
- myinfo：version=1.16，author=Jahze, Visor, A1m`, Forgetest
- myinfo description（源码原文）：Blocks instant repounces and gives m2 penalty after a shove/deadstop

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `l4d2_m2_hunter_penalty` | `0` | 整数 | 无上下界 | 推 Hunter 时增加的惩罚值 | 服务器端本插件逻辑 | `l4d2_m2_control_eq.sp:51` |
| `l4d2_m2_jockey_penalty` | `0` | 整数 | 无上下界 | 推 Jockey 时增加的惩罚值 | 服务器端本插件逻辑 | `l4d2_m2_control_eq.sp:52` |
| `l4d2_m2_smoker_penalty` | `0` | 整数 | 无上下界 | 推 Smoker 时增加的惩罚值 | 服务器端本插件逻辑 | `l4d2_m2_control_eq.sp:53` |

## Magnum incap remover —— `optional/l4d2_magnum_incap.sp`

- 源文件：`optional/l4d2_magnum_incap.sp`
- myinfo：version=0.5.0，author=robex, Sir, Forgetest
- myinfo description（源码原文）：Replace magnum with regular pistols when incapped.

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `l4d2_replace_magnum_incap` | `1.0` | 整数，源码写作浮点 | 无上下界 | 倒地时把手枪替换为单持（1）或双持（2），0 关闭 | 服务器端本插件逻辑 | `l4d2_magnum_incap.sp:29` |

## Map Transitions —— `optional/l4d2_map_transitions.sp`

- 源文件：`optional/l4d2_map_transitions.sp`
- myinfo：version=3.3，author=Derpduck, Forgetest
- myinfo description（源码原文）：Define map transitions to combine campaigns

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_add_map_transition` | <起始地图> <结束地图> | 服务器控制台命令，玩家无法使用 | 添加地图过场映射，服务器控制台命令 | `l4d2_map_transitions.sp:45` |

### ConVar

（本插件未注册 ConVar）

## Shove Shenanigans - REVAMPED —— `optional/l4d2_melee_shenanigans.sp`

- 源文件：`optional/l4d2_melee_shenanigans.sp`
- myinfo：version=1.3，author=Sir
- myinfo description（源码原文）：Stops Shoves slowing the Tank and Charger, gives control over what happens when a Survivor is punched while having a melee out.

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `l4d2_melee_drop_method` | `2` | 整数 | 无上下界 | 坦克拳击手持近战武器的生还者时的处理：0 无动作，1 掉落近战武器，2 强制处理，源码描述在此被截断 | 服务器端本插件逻辑 | `l4d2_melee_shenanigans.sp:23` |

## l4d2 melee spawn control —— `optional/l4d2_melee_spawn_control.sp`

- 源文件：`optional/l4d2_melee_spawn_control.sp`
- myinfo：version=1.3，author=IA/NanaNana
- myinfo description（源码原文）：Unlock melee weapons

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `l4d2_melee_spawn` | `` | 字符串或表达式 | 无上下界 | 解锁的近战武器列表，逗号分隔，留空不作修改 | 服务器端本插件逻辑 | `l4d2_melee_spawn_control.sp:65` |
| `l4d2_add_melee` | `` | 字符串或表达式 | 无上下界 | 添加近战武器到地图基础刷新列表或 l4d2_melee_spawn，逗号分隔，留空不添加 | 服务器端本插件逻辑 | `l4d2_melee_spawn_control.sp:66` |

## l4d2_mixmap —— `optional/l4d2_mixmap.sp`

- 源文件：`optional/l4d2_mixmap.sp`
- myinfo：version=2.5，author=Bred, Hitomi
- myinfo description（源码原文）：Randomly select five maps for versus. Adding for fun and reference from CMT

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_addmap` | <地图名> <标签...> | 服务器控制台命令，玩家无法使用 | 把地图加入 mixmap 池并设置标签，服务器控制台命令 | `l4d2_mixmap.sp:127` |
| `sm_tagrank` | <标签> <地图编号> | 服务器控制台命令，玩家无法使用 | 设置标签对应的地图排序，服务器控制台命令 | `l4d2_mixmap.sp:128` |
| `sm_manualmixmap` | <地图顺序> | 需要 ADMFLAG_ROOT（z，最高权限） | 按指定顺序启用 mixmap | `l4d2_mixmap.sp:131` |
| `sm_fmixmap` | <cfg 名> | 需要 ADMFLAG_ROOT（z，最高权限） | 强制启用 mixmap，参数为空时使用默认地图池 | `l4d2_mixmap.sp:132` |
| `sm_mixmap` | <cfg 名> | 无（任意玩家可用） | 发起启用 mixmap 的投票 | `l4d2_mixmap.sp:133` |
| `sm_stopmixmap` | 无参数 | 无（任意玩家可用） | 发起中止 mixmap 的投票 | `l4d2_mixmap.sp:134` |
| `sm_fstopmixmap` | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 强制中止 mixmap | `l4d2_mixmap.sp:135` |
| `sm_maplist` | 无参数 | 无（任意玩家可用） | 显示 mixmap 最终抽取的地图列表 | `l4d2_mixmap.sp:138` |
| `sm_allmap` | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 显示所有官方地图的地图代码 | `l4d2_mixmap.sp:139` |
| `sm_allmaps` | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | sm_allmap 的别名 | `l4d2_mixmap.sp:140` |

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `l4d2mm_nextmap_print` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否显示下一张地图是什么 | 服务器端本插件逻辑 | `l4d2_mixmap.sp:122` |
| `l4d2mm_max_maps_num` | `2` | 整数，源码写作浮点 | 0.0 ~ 5.0 | 一个战役最多可选择多少张地图，0 表示不限制，范围 0~5 | 服务器端本插件逻辑 | `l4d2_mixmap.sp:123` |
| `l4d2mm_finale_end_start` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 终局结束后是否重新混合地图：0 禁用，1 启用 | 服务器端本插件逻辑 | `l4d2_mixmap.sp:124` |

## L4D2 No Hunter Deadstops —— `optional/l4d2_no_hunter_deadstops.sp`

- 源文件：`optional/l4d2_no_hunter_deadstops.sp`
- myinfo：version=3.4，author=Visor, A1m
- myinfo description（源码原文）：Self-descriptive

### 指令

（本插件未注册指令）

### ConVar

（本插件未注册 ConVar）

## L4D2 No M2 Reg on Hunters and Pulled Survivors —— `optional/l4d2_no_m2_pulled_and_hunters.sp`

- 源文件：`optional/l4d2_no_m2_pulled_and_hunters.sp`
- myinfo：version=3.4，author=Visor, Sir
- myinfo description（源码原文）：Self-descriptive

### 指令

（本插件未注册指令）

### ConVar

（本插件未注册 ConVar）

## L4D2 No Backjump —— `optional/l4d2_nobackjumps.sp`

- 源文件：`optional/l4d2_nobackjumps.sp`
- myinfo：version=1.4，author=Visor, A1m`, Forgetest
- myinfo description（源码原文）：Look at the title

### 指令

（本插件未注册指令）

### ConVar

（本插件未注册 ConVar）

## Simple Anti-Bunnyhop —— `optional/l4d2_nobhaps.sp`

- 源文件：`optional/l4d2_nobhaps.sp`
- myinfo：version=0.5.1，author=CanadaRox, ProdigySim, blodia, CircleSquared, robex, A1m`
- myinfo description（源码原文）：Stops bunnyhops by restricting speed when a player lands on the ground to their MaxSpeed

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_check_bhop` | 无参数 | 无（任意玩家可用） | 在聊天中报告当前是否允许连跳及豁免情况 | `l4d2_nobhaps.sp:85` |

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `simple_antibhop_enable` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用 Simple Anti-Bhop 插件：0 关，1 开 | 服务器端本插件逻辑 | `l4d2_nobhaps.sp:53` |
| `bhop_except_si_flags` | `0` | 整数，源码写作浮点 | 0.0 ~ 127.0 | 豁免禁跳的特感位掩码：1 smoker，2 boomer，4 hunter，8 spitter，16 jockey，其余被截断 | 服务器端本插件逻辑 | `l4d2_nobhaps.sp:64` |
| `bhop_allow_survivor` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否允许生还者连跳：1 允许，0 禁止 | 服务器端本插件逻辑 | `l4d2_nobhaps.sp:72` |

## L4D2 No Second Chances —— `optional/l4d2_nosecondchances.sp`

- 源文件：`optional/l4d2_nosecondchances.sp`
- myinfo：version=1.4，author=Visor, Jacob, A1m`
- myinfo description（源码原文）：Previously human-controlled SI bots with a cap won't die

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `bot_kick_delay` | `0` | 整数，源码写作浮点 | 0.0 ~ 30.0 | 踢出特感 Bot 前等待的秒数，范围 0~30 | 服务器端本插件逻辑 | `l4d2_nosecondchances.sp:54` |

## L4D2 Display Infected HP —— `optional/l4d2_nosey_parker.sp`

- 源文件：`optional/l4d2_nosey_parker.sp`
- myinfo：version=1.5.3，author=Visor, A1m`
- myinfo description（源码原文）：Survivors receive damage reports after they get capped

### 指令

（本插件未注册指令）

### ConVar

（本插件未注册 ConVar）

## No Spitter During Tank —— `optional/l4d2_nospitterduringtank.sp`

- 源文件：`optional/l4d2_nospitterduringtank.sp`
- myinfo：version=2.0，author=Don, epilimic, Griffin
- myinfo description（源码原文）：Prevents the director from giving the infected team a spitter while the tank is alive
- HookConVarChange：`g_hSpitterLimit`→`Cvar_SpitterLimit`

### 指令

（本插件未注册指令）

### ConVar

（本插件未注册 ConVar）

## L4D2 Tank Claw Fix —— `optional/l4d2_notankautoaim.sp`

- 源文件：`optional/l4d2_notankautoaim.sp`
- myinfo：version=0.5，author=Jahze(patch data), Visor(SM), A1m`
- myinfo description（源码原文）：Removes the Tank claw's undocumented auto-aiming ability

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `l4d2_notankautoaim` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否移除坦克爪击未公开的自动瞄准能力：1 启用，0 禁用 | 服务器端本插件逻辑 | `l4d2_notankautoaim.sp:28` |

## Penalty bonus system —— `optional/l4d2_penalty_bonus.sp`

- 源文件：`optional/l4d2_penalty_bonus.sp`
- myinfo：version=2.1，author=Tabun, A1m`
- myinfo description（源码原文）：Allows other plugins to set bonuses for a round that will be given even if the saferoom is not reached.

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_bonus` | 无参数 | 无（任意玩家可用） | 显示本回合当前的额外奖励 | `l4d2_penalty_bonus.sp:117` |

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `sm_pbonus_enable` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用扣分奖励系统 | 服务器端本插件逻辑 | `l4d2_penalty_bonus.sp:102` |
| `sm_pbonus_display` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否在回合结束与使用 !bonus 时显示奖励 | 服务器端本插件逻辑 | `l4d2_penalty_bonus.sp:103` |
| `sm_pbonus_reportchanges` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 当前奖励被修改时是否报告变更 | 服务器端本插件逻辑 | `l4d2_penalty_bonus.sp:104` |
| `sm_pbonus_tank` | `0` | 整数，源码写作浮点 | ≥ 0.0 | 击杀坦克给予的奖励值，0 完全关闭 | 服务器端本插件逻辑 | `l4d2_penalty_bonus.sp:105` |
| `sm_pbonus_witch` | `0` | 整数，源码写作浮点 | ≥ 0.0 | 击杀 Witch 给予的奖励值，0 完全关闭 | 服务器端本插件逻辑 | `l4d2_penalty_bonus.sp:106` |

## [L4D & 2] Pick-up Changes —— `optional/l4d2_pickup.sp`

- 源文件：`optional/l4d2_pickup.sp`
- myinfo：author=Sir, Forgetest
- myinfo description（源码原文）：Alters a few things regarding picking up/giving items and incapped Players.

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_primary` | 无参数 | 无（任意玩家可用） | 切换拾取物品后是否切换到主武器 | `l4d2_pickup.sp:219` |
| `sm_secondary` | 无参数 | 无（任意玩家可用） | 切换拾取物品后是否切换到副武器 | `l4d2_pickup.sp:220` |

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `l4d2_pickup_version` | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | 服务器端本插件逻辑；复制到客户端；仅服务器；不写入 cfg 存档 | `l4d2_pickup.sp:213` |
| `pickup_switch_flags` | `0` | 整数，源码写作浮点 | 0.0 ~ 7.0 | 切换所拾取物品的 flag，范围 0~7：1 默认切换到拾取的副武器，2 永不切换到被给予的止痛药或肾上腺素，其余被截断 | 服务器端本插件逻辑 | `l4d2_pickup.sp:215` |
| `pickup_incap_flags` | `7` | 整数，源码写作浮点 | 0.0 ~ 7.0 | 倒地生还者终止拾取进度的 flag，范围 0~7：1 毒痰伤害，2 坦克拳击，4 坦克石头 | 服务器端本插件逻辑 | `l4d2_pickup.sp:224` |

## Player Statistics —— `optional/l4d2_playstats.sp`

- 源文件：`optional/l4d2_playstats.sp`
- myinfo：version=1.1.3，author=Tabun, A1m`
- myinfo description（源码原文）：Tracks statistics, even when clients disconnect. MVP, Skills, Accuracy, etc.

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_stats` | [参数] | 无（任意玩家可用） | 输出生还者统计 | `l4d2_playstats.sp:569` |
| `sm_mvp` | [参数] | 无（任意玩家可用） | 输出生还者 MVP 统计 | `l4d2_playstats.sp:570` |
| `sm_skill` | [参数] | 无（任意玩家可用） | 输出生还者特殊技能统计 | `l4d2_playstats.sp:571` |
| `sm_ff` | [参数] | 无（任意玩家可用） | 输出友军伤害统计 | `l4d2_playstats.sp:572` |
| `sm_acc` | [参数] | 无（任意玩家可用） | 输出生还者命中率统计 | `l4d2_playstats.sp:573` |
| `sm_stats_auto` | <0 或 1> | 无（任意玩家可用） | 设置客户端是否在回合结束时自动打印统计 | `l4d2_playstats.sp:575` |
| `statsreset` | 无参数 | 需要 ADMFLAG_CHANGEMAP（g，换图） | 重置统计数据，仅管理员可用 | `l4d2_playstats.sp:577` |

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `sm_stats_debug` | `0` | 整数，源码写作浮点 | ≥ 0.0 | 调试模式，最小 0 | 服务器端本插件逻辑 | `l4d2_playstats.sp:512` |
| `sm_survivor_mvp_brevity_latest` | `4` | 整数，源码写作浮点 | ≥ 0.0 | MVP 聊天报告精简 flag：1 隐藏特感，2 隐藏普感，4 隐藏友伤，8 隐藏排名，32 隐藏百分比，64 隐藏绝对值 | 服务器端本插件逻辑 | `l4d2_playstats.sp:519` |
| `sm_stats_autoprint_vs_round` | `8325` | 整数，源码写作浮点 | ≥ 0.0 | 对抗回合自动播报位标志；源码其实已给描述（`optional/l4d2_playstats.sp:526-531` 第 3 参数）：「Flags for automatic print [versus round] (show 1,4:MVP-chat, 4,8,16:MVP-console, 32,64:FF, 128,256:special, 512,1024,2048,4096:accuracy).」，:528 的内联注释进一步说明默认值 `8325 = 1(mvpchat) + 4(mvpcon-round) + 128(special round) + 8192(funfact round)`，即按位控制对抗模式下回合结束时自动打印哪些统计表（置信度：高） | 服务器端本插件逻辑 | `l4d2_playstats.sp:526` |
| `sm_stats_autoprint_coop_round` | `1289` | 整数，源码写作浮点 | ≥ 0.0 | 合作(campaign)回合自动播报位标志；源码其实已给描述（`optional/l4d2_playstats.sp:533-538` 第 3 参数）：「Flags for automatic print [campaign round] (show 1,4:MVP-chat, 4,8,16:MVP-console, 32,64:FF, 128,256:special, 512,1024,2048,4096:accuracy).」，:535 的内联注释说明默认值 `1289 = 1(mvpchat) + 8(mvpcon-all) + 256(special all) + 1024(acc all)`，即按位控制合作模式下回合结束时自动打印哪些统计表（置信度：高） | 服务器端本插件逻辑 | `l4d2_playstats.sp:533` |
| `sm_stats_showbots` | `1` | 整数，源码写作浮点 | ≥ 0.0 | 是否在所有表格中显示 Bot，0 表示仅在 MVP 与友伤表中显示 | 服务器端本插件逻辑 | `l4d2_playstats.sp:540` |
| `sm_stats_percentdecimal` | `0` | 整数，源码写作浮点 | ≥ 0.0 | 是否在多数 MVP 百分比的控制台表格中显示一位小数 | 服务器端本插件逻辑 | `l4d2_playstats.sp:547` |
| `sm_stats_writestats` | `0` | 整数，源码写作浮点 | ≥ 0.0 | 是否把统计数据写入 logs/ 目录：1 写 csv，2 写 csv 与格式化表格，仅对战模式 | 服务器端本插件逻辑 | `l4d2_playstats.sp:554` |
| `sm_stats_resetnextmap` | `0` | 整数，源码写作浮点 | ≥ 0.0 | 是否忽略第一回合，配合 confogl 与比赛投票使用，换图后自动取消 | 服务器端本插件逻辑 | `l4d2_playstats.sp:561` |

## L4D2 Profitless AI Tank —— `optional/l4d2_profitless_ai_tank.sp`

- 源文件：`optional/l4d2_profitless_ai_tank.sp`
- myinfo：version=0.4，author=Visor, Forgetest
- myinfo description（源码原文）：Passing control to AI Tank will no longer be rewarded with an instant respawn

### 指令

（本插件未注册指令）

### ConVar

（本插件未注册 ConVar）

## [L4D2] No Reload Animation Fix —— `optional/l4d2_reload_fix.sp`

- 源文件：`optional/l4d2_reload_fix.sp`
- myinfo：author=SilverShot, HarryPotter
- myinfo description（源码原文）：Prevent filling the clip and skipping the reload animation when taking the same weapon.

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `l4d2_reload_fix_give` | `1` | 整数 | 无上下界 | 使用 give 命令替换同类型武器时是否把弹药转移给新武器：0 否，1 是 | 服务器端本插件逻辑 | `l4d2_reload_fix.sp:168` |
| `l4d2_reload_fix_version` | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | 服务器端本插件逻辑；不写入 cfg 存档 | `l4d2_reload_fix.sp:169` |

## L4D2 Riot Cops —— `optional/l4d2_riotcops.sp`

- 源文件：`optional/l4d2_riotcops.sp`
- myinfo：version=1.6.1，author=Jahze, Visor, A1m`
- myinfo description（源码原文）：Allow riot cops to be killed by a headshot

### 指令

（本插件未注册指令）

### ConVar

（本插件未注册 ConVar）

## Precise saferoom detection —— `optional/l4d2_saferoom_detect.sp`

- 源文件：`optional/l4d2_saferoom_detect.sp`
- myinfo：version=0.0.8，author=Tabun, devilesk
- myinfo description（源码原文）：Allows checks whether a coordinate/entity/player is in start or end saferoom (uses saferoominfo.txt).

### 指令

（本插件未注册指令）

### ConVar

（本插件未注册 ConVar）

## Saferoom Item Remover —— `optional/l4d2_saferoom_item_remove.sp`

- 源文件：`optional/l4d2_saferoom_item_remove.sp`
- myinfo：version=1.1.2，author=Tabun, Sir, A1m`
- myinfo description（源码原文）：Removes any saferoom item (start or end).

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `sm_safeitemkill_enable` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否移除终点安全室的物品 | 服务器端本插件逻辑 | `l4d2_saferoom_item_remove.sp:47` |
| `sm_safeitemkill_saferooms` | `1` | 整数，源码写作浮点 | 0.0 ~ 3.0 | 要清空的安全室 flag：1 终点安全室，2 起点安全室，3 两者 | 服务器端本插件逻辑 | `l4d2_saferoom_item_remove.sp:48` |
| `sm_safeitemkill_items` | `7` | 整数，源码写作浮点 | 0.0 ~ 15.0 | 要移除的物品类型 flag：1 治疗物品，2 枪械，4 近战，8 其他可用物品，范围 0~15 | 服务器端本插件逻辑 | `l4d2_saferoom_item_remove.sp:49` |

## L4D2 Scoremod —— `optional/l4d2_scoremod.sp`

- 源文件：`optional/l4d2_scoremod.sp`
- myinfo：version=1.1.2，author=CanadaRox, ProdigySim
- myinfo description（源码原文）：L4D2 Custom Scoring System (Health Bonus)
- HookConVarChange：`SM_hEnable`→`SM_ConVarChanged_Enable`、`SM_hHBRatio`→`SM_CVChanged_HealthBonusRatio`、`SM_hSurvivalBonusRatio`→`SM_CVChanged_SurvivalBonusRatio`、`SM_hTempMulti0`→`SM_ConVarChanged_TempMulti0`、`SM_hTempMulti1`→`SM_ConVarChanged_TempMulti1`、`SM_hTempMulti2`→`SM_ConVarChanged_TempMulti2`、`SM_hHealPercent`→`SM_ConVarChanged_Health`、`SM_hPillPercent`→`SM_ConVarChanged_Health`、`SM_hAdrenPercent`→`SM_ConVarChanged_Health`

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_health` | 无参数 | 无（任意玩家可用） | 显示当前生还者平均血量与回合奖励 | `l4d2_scoremod.sp:127` |
| `say` | 聊天文本 | 无（任意玩家可用） | 接管 say，识别 !health | `l4d2_scoremod.sp:232` |
| `say_team` | 聊天文本 | 无（任意玩家可用） | 接管 say_team，识别 !health | `l4d2_scoremod.sp:233` |

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `SM_enable` | `1` | 整数 | 无上下界 | L4D2 自定义计分系统开关 | 服务器端本插件逻辑 | `l4d2_scoremod.sp:80` |
| `SM_healthbonusratio` | `2.0` | 浮点 | 0.25 ~ 5.0 | 生命奖励倍率，范围 0.25~5 | 服务器端本插件逻辑 | `l4d2_scoremod.sp:83` |
| `SM_survivalbonusratio` | `0.0` | 整数，源码写作浮点 | 无上下界 | 用于按地图距离计算固定生存奖励的比例 | 服务器端本插件逻辑 | `l4d2_scoremod.sp:86` |
| `SM_tempmulti_incap_0` | `0.30625` | 浮点 | 0.0 ~ 1.0 | 未有倒地记录的生还者其临时生命的重要程度 | 服务器端本插件逻辑 | `l4d2_scoremod.sp:89` |
| `SM_tempmulti_incap_1` | `0.17500` | 浮点 | 0.0 ~ 1.0 | 倒地一次的生还者其临时生命的重要程度 | 服务器端本插件逻辑 | `l4d2_scoremod.sp:92` |
| `SM_tempmulti_incap_2` | `0.10000` | 浮点 | 0.0 ~ 1.0 | 倒地两次即黑白的生还者其临时生命的重要程度 | 服务器端本插件逻辑 | `l4d2_scoremod.sp:95` |
| `SM_first_aid_heal_percent` | `buf` | 字符串或表达式 | 无上下界 | 医疗包回复生命的百分比 | 服务器端本插件逻辑 | `l4d2_scoremod.sp:104` |
| `SM_pain_pills_health_value` | `buf` | 字符串或表达式 | 无上下界 | 止痛药增加的生命值 | 服务器端本插件逻辑 | `l4d2_scoremod.sp:106` |
| `SM_adrenaline_health_buffer` | `buf` | 字符串或表达式 | 无上下界 | 肾上腺素增加的生命缓冲值 | 服务器端本插件逻辑 | `l4d2_scoremod.sp:108` |
| `SM_mapmulti` | `1` | 整数 | 无上下界 | 是否把生命奖励上限提升到距离上限 | 服务器端本插件逻辑 | `l4d2_scoremod.sp:110` |
| `SM_custommaxdistance` | `0` | 整数 | 无上下界 | 是否使用配置中的自定义最大距离 | 服务器端本插件逻辑 | `l4d2_scoremod.sp:112` |

## SetScores —— `optional/l4d2_setscores.sp`

- 源文件：`optional/l4d2_setscores.sp`
- myinfo：author=vintik, Forgetest, A1m`
- myinfo description（源码原文）：Changes team scores.

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_setscores` | <生还者分数> <特感分数> | 无（任意玩家可用） | 修改分数，需处于准备阶段且回合尚未开始 | `l4d2_setscores.sp:62` |

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `setscore_player_limit` | `2` | 整数 | 无上下界 | 发起投票所需的最少在场玩家数 | 服务器端本插件逻辑 | `l4d2_setscores.sp:58` |
| `setscore_allow_player_vote` | `1` | 整数 | 无上下界 | 是否允许玩家发起投票：1 允许（默认），0 不允许 | 服务器端本插件逻辑 | `l4d2_setscores.sp:59` |
| `setscore_force_admin_vote` | `0` | 整数 | 无上下界 | 管理员修改分数是否需要投票：1 需要，0 不需要（默认） | 服务器端本插件逻辑 | `l4d2_setscores.sp:60` |

## L4D2 Infected Friendly Fire Disable —— `optional/l4d2_si_ffblock.sp`

- 源文件：`optional/l4d2_si_ffblock.sp`
- myinfo：version=2.1，author=ProdigySim, Don, Visor, A1m`
- myinfo description（源码原文）：Disables friendly fire between infected players.

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `l4d2_block_infected_ff` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否禁用特感之间的友军伤害 | 服务器端本插件逻辑 | `l4d2_si_ffblock.sp:41` |
| `l4d2_infected_ff_allow_tank` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否不禁用坦克对其他特感的友军伤害 | 服务器端本插件逻辑 | `l4d2_si_ffblock.sp:42` |
| `l4d2_infected_ff_block_witch` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否禁用对 Witch 的友军伤害 | 服务器端本插件逻辑 | `l4d2_si_ffblock.sp:43` |

## L4D2 No SI Friendly Staggers —— `optional/l4d2_si_staggers.sp`

- 源文件：`optional/l4d2_si_staggers.sp`
- myinfo：version=1.3，author=Visor, A1m`
- myinfo description（源码原文）：Removes SI staggers caused by other SI(Boomer, Charger, Witch)

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `l4d2_disable_si_friendly_staggers` | `0` | 整数，源码写作浮点 | 0.0 ~ 7.0 | 是否移除其他特感造成的特感硬直，位掩码：1 Boomer，2 Charger，4 Witch，范围 0~7 | 服务器端本插件逻辑 | `l4d2_si_staggers.sp:55` |

## Skill Detection (skeets, crowns, levels) —— `optional/l4d2_skill_detect.sp`

- 源文件：`optional/l4d2_skill_detect.sp`、`optional/l4d2_skill_detect/report.sp`、`optional/l4d2_skill_detect/tracking.sp`
- myinfo：author=Tabun
- myinfo description（源码原文）：Detects and reports skeets, crowns, levels, highpounces, etc.
- HookConVarChange：`g_cvarPounceInterrupt`→`CvarChange_PounceInterrupt`

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `sm_skill_detect_version` | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | 服务器端本插件逻辑；复制到客户端；不写入 cfg 存档 | `l4d2_skill_detect.sp:477` |
| `sm_skill_detect_debug` | `0` | 整数 | 无上下界 | 是否启用调试消息 | 服务器端本插件逻辑；复制到客户端；不写入 cfg 存档 | `l4d2_skill_detect.sp:478` |
| `sm_skill_report_enable` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否在聊天中播报 | 服务器端本插件逻辑 | `l4d2_skill_detect.sp:481` |
| `sm_skill_report_skeet` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用飞扑击杀播报 | 服务器端本插件逻辑 | `l4d2_skill_detect.sp:482` |
| `sm_skill_report_hurtskeet` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用受伤飞扑击杀播报 | 服务器端本插件逻辑 | `l4d2_skill_detect.sp:483` |
| `sm_skill_report_level` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用单杀播报 | 服务器端本插件逻辑 | `l4d2_skill_detect.sp:484` |
| `sm_skill_report_hurtlevel` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用受伤单杀播报 | 服务器端本插件逻辑 | `l4d2_skill_detect.sp:485` |
| `sm_skill_report_crow` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用爆头击杀 Witch 播报 | 服务器端本插件逻辑 | `l4d2_skill_detect.sp:486` |
| `sm_skill_report_drawcrow` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用互爆 Witch 播报 | 服务器端本插件逻辑 | `l4d2_skill_detect.sp:487` |
| `sm_skill_report_tonguecut` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用断舌播报 | 服务器端本插件逻辑 | `l4d2_skill_detect.sp:488` |
| `sm_skill_report_sc` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用自救播报 | 服务器端本插件逻辑 | `l4d2_skill_detect.sp:489` |
| `sm_skill_report_scs` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用推击自救播报 | 服务器端本插件逻辑 | `l4d2_skill_detect.sp:490` |
| `sm_skill_report_rockskeet` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用石头击杀播报 | 服务器端本插件逻辑 | `l4d2_skill_detect.sp:491` |
| `sm_skill_report_rockname` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用 Tank 名称播报 | 服务器端本插件逻辑 | `l4d2_skill_detect.sp:492` |
| `sm_skill_report_deadstop` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用死停播报 | 服务器端本插件逻辑 | `l4d2_skill_detect.sp:493` |
| `sm_skill_report_pop` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用弹跳击杀播报 | 服务器端本插件逻辑 | `l4d2_skill_detect.sp:494` |
| `sm_skill_report_shove` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用推击播报 | 服务器端本插件逻辑 | `l4d2_skill_detect.sp:495` |
| `sm_skill_report_hunterdp` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用 Hunter 高扑播报 | 服务器端本插件逻辑 | `l4d2_skill_detect.sp:496` |
| `sm_skill_report_jockeydp` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用 Jockey 高跳播报 | 服务器端本插件逻辑 | `l4d2_skill_detect.sp:497` |
| `sm_skill_report_deadcharger` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用 Charger 致死冲锋播报 | 服务器端本插件逻辑 | `l4d2_skill_detect.sp:498` |
| `sm_skill_report_instanclear` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用瞬间解救播报 | 服务器端本插件逻辑 | `l4d2_skill_detect.sp:499` |
| `sm_skill_report_bhop` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用连跳次数播报 | 服务器端本插件逻辑 | `l4d2_skill_detect.sp:500` |
| `sm_skill_report_caralarm` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用警报车播报 | 服务器端本插件逻辑 | `l4d2_skill_detect.sp:501` |
| `sm_skill_skeet_allowmelee` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否把近战击杀计入并转发飞扑击杀 | 服务器端本插件逻辑 | `l4d2_skill_detect.sp:503` |
| `sm_skill_skeet_allowsniper` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否把狙击与马格南爆头计入飞扑击杀 | 服务器端本插件逻辑 | `l4d2_skill_detect.sp:504` |
| `sm_skill_skeet_allowgl` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否把榴弹直击计入飞扑击杀 | 服务器端本插件逻辑 | `l4d2_skill_detect.sp:505` |
| `sm_skill_drawcrown_damage` | `500` | 整数，源码写作浮点 | ≥ 0.0 | 判定为互爆 Witch 所需的最后一击最低伤害 | 服务器端本插件逻辑 | `l4d2_skill_detect.sp:506` |
| `sm_skill_selfclear_damage` | `200` | 整数，源码写作浮点 | ≥ 0.0 | 判定为自救 Smoker 所需的最低伤害 | 服务器端本插件逻辑 | `l4d2_skill_detect.sp:507` |
| `sm_skill_hunterdp_height` | `400` | 整数，源码写作浮点 | ≥ 0.0 | 判定为 Hunter 高扑的最低高度 | 服务器端本插件逻辑 | `l4d2_skill_detect.sp:508` |
| `sm_skill_jockeydp_height` | `300` | 整数，源码写作浮点 | ≥ 0.0 | 判定为 Jockey 高跳所需的高度差 | 服务器端本插件逻辑 | `l4d2_skill_detect.sp:509` |
| `sm_skill_hidefakedamage` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 若开启，超过受害者生命值的伤害在报告中隐藏 | 服务器端本插件逻辑 | `l4d2_skill_detect.sp:510` |
| `sm_skill_deathcharge_height` | `400` | 整数，源码写作浮点 | ≥ 0.0 | 判定 Charger 致死冲锋所需带受害者移动的高度差 | 服务器端本插件逻辑 | `l4d2_skill_detect.sp:511` |
| `sm_skill_instaclear_time` | `0.75` | 浮点 | ≥ 0.0 | 在该秒数内完成解救计入瞬间解救 | 服务器端本插件逻辑 | `l4d2_skill_detect.sp:512` |
| `sm_skill_bhopstreak` | `3` | 整数，源码写作浮点 | ≥ 0.0 | 触发播报的最低连跳次数 | 服务器端本插件逻辑 | `l4d2_skill_detect.sp:513` |
| `sm_skill_bhopinitspeed` | `150` | 整数，源码写作浮点 | ≥ 0.0 | 连跳第一次起跳的最低速度，0 允许从静止起跳 | 服务器端本插件逻辑 | `l4d2_skill_detect.sp:514` |
| `sm_skill_bhopkeepspeed` | `300` | 整数，源码写作浮点 | ≥ 0.0 | 即使没有加速也视为成功连跳的最低速度 | 服务器端本插件逻辑 | `l4d2_skill_detect.sp:515` |
| `z_pounce_damage_range_max` | `1000.0` | 整数，源码写作浮点 | ≥ 0.0 | 源码说明：本服务器没有该 cvar，由 l4d2_skill_detect 添加 | 服务器端本插件逻辑 | `l4d2_skill_detect.sp:531` |
| `z_pounce_damage_range_min` | `300.0` | 整数，源码写作浮点 | ≥ 0.0 | 源码说明：本服务器没有该 cvar，由 l4d2_skill_detect 添加 | 服务器端本插件逻辑 | `l4d2_skill_detect.sp:533` |
| `z_hunter_max_pounce_bonus_damage` | `49` | 整数，源码写作浮点 | ≥ 0.0 | 源码说明：本服务器没有该 cvar，由 l4d2_skill_detect 添加 | 服务器端本插件逻辑 | `l4d2_skill_detect.sp:535` |

## L4D2 Slowdown Control —— `optional/l4d2_slowdown_control.sp`

- 源文件：`optional/l4d2_slowdown_control.sp`
- myinfo：version=2.7.1，author=Visor, Sir, darkid, Forgetest, A1m`, Derpduck
- myinfo description（源码原文）：Manages the water/gunfire slowdown for both teams

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `l4d2_slowdown_gunfire_si` | `0.0` | 整数，源码写作浮点 | -1.0 ~ 1.0 | 特感受枪击的最大减速：-1 原生减速，0.0 不减速，0.01-1.0 表示 1% 到 100% | 服务器端本插件逻辑 | `l4d2_slowdown_control.sp:87` |
| `l4d2_slowdown_gunfire_tank` | `0.2` | 浮点 | -1.0 ~ 1.0 | 坦克受枪击的最大减速：-1 原生减速，0.0 不减速，0.01-1.0 表示 1% 到 100% | 服务器端本插件逻辑 | `l4d2_slowdown_control.sp:88` |
| `l4d2_slowdown_water_tank` | `-1` | 整数，源码写作浮点 | ≥ -1.0 | 坦克在水中的最大速度：-1 忽略设置，0 默认，210 为默认坦克速度 | 服务器端本插件逻辑 | `l4d2_slowdown_control.sp:89` |
| `l4d2_slowdown_water_survivors` | `-1` | 整数，源码写作浮点 | ≥ -1.0 | 非坦克战期间生还者在水中的最大速度：-1 忽略设置，0 默认，220 为默认生还者速度 | 服务器端本插件逻辑 | `l4d2_slowdown_control.sp:90` |
| `l4d2_slowdown_water_survivors_during_tank` | `0` | 整数，源码写作浮点 | ≥ 0.0 | 坦克战期间生还者在水中的最大速度：0 忽略设置，220 为默认生还者速度 | 服务器端本插件逻辑 | `l4d2_slowdown_control.sp:91` |
| `l4d2_slowdown_crouch_speed_mod` | `1.0` | 整数，源码写作浮点 | ≥ 0.0 | 指定触发区内玩家蹲下速度的修正，75 为默认值，1 表示默认速度 | 服务器端本插件逻辑 | `l4d2_slowdown_control.sp:92` |
| `l4d2_slowdown_pistol_percent` | `0.0` | 整数，源码写作浮点 | 无上下界 | 手枪在最大伤害时造成的减速等于该值乘以 l4d2_slowdown_gunfire | 服务器端本插件逻辑 | `l4d2_slowdown_control.sp:94` |
| `l4d2_slowdown_deagle_percent` | `0.1` | 浮点 | 无上下界 | 沙鹰在最大伤害时造成的减速比例 | 服务器端本插件逻辑 | `l4d2_slowdown_control.sp:95` |
| `l4d2_slowdown_uzi_percent` | `0.8` | 浮点 | 无上下界 | 无消音乌兹在最大伤害时造成的减速比例 | 服务器端本插件逻辑 | `l4d2_slowdown_control.sp:96` |
| `l4d2_slowdown_mac_percent` | `0.8` | 浮点 | 无上下界 | 消音乌兹在最大伤害时造成的减速比例 | 服务器端本插件逻辑 | `l4d2_slowdown_control.sp:97` |
| `l4d2_slowdown_ak_percent` | `0.8` | 浮点 | 无上下界 | AK 在最大伤害时造成的减速比例 | 服务器端本插件逻辑 | `l4d2_slowdown_control.sp:98` |
| `l4d2_slowdown_m4_percent` | `0.8` | 浮点 | 无上下界 | M4 在最大伤害时造成的减速比例 | 服务器端本插件逻辑 | `l4d2_slowdown_control.sp:99` |
| `l4d2_slowdown_scar_percent` | `0.8` | 浮点 | 无上下界 | SCAR 在最大伤害时造成的减速比例 | 服务器端本插件逻辑 | `l4d2_slowdown_control.sp:100` |
| `l4d2_slowdown_pump_percent` | `0.5` | 浮点 | 无上下界 | 泵动霰弹枪在最大伤害时造成的减速比例 | 服务器端本插件逻辑 | `l4d2_slowdown_control.sp:101` |
| `l4d2_slowdown_chrome_percent` | `0.5` | 浮点 | 无上下界 | Chrome 霰弹枪在最大伤害时造成的减速比例 | 服务器端本插件逻辑 | `l4d2_slowdown_control.sp:102` |
| `l4d2_slowdown_auto_percent` | `0.5` | 浮点 | 无上下界 | 自动霰弹枪在最大伤害时造成的减速比例 | 服务器端本插件逻辑 | `l4d2_slowdown_control.sp:103` |
| `l4d2_slowdown_rifle_percent` | `0.1` | 浮点 | 无上下界 | 猎枪在最大伤害时造成的减速比例 | 服务器端本插件逻辑 | `l4d2_slowdown_control.sp:104` |
| `l4d2_slowdown_scout_percent` | `0.1` | 浮点 | 无上下界 | Scout 在最大伤害时造成的减速比例 | 服务器端本插件逻辑 | `l4d2_slowdown_control.sp:105` |
| `l4d2_slowdown_military_percent` | `0.1` | 浮点 | 无上下界 | 军用狙击枪在最大伤害时造成的减速比例 | 服务器端本插件逻辑 | `l4d2_slowdown_control.sp:106` |

## L4D2 smoker drag damage interval —— `optional/l4d2_smoker_drag_damage_interval.sp`

- 源文件：`optional/l4d2_smoker_drag_damage_interval.sp`
- myinfo：version=2.4，author=Visor, Sir, A1m`
- myinfo description（源码原文）：Implements a native-like cvar and functionality that should've been there out of the box

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `tongue_drag_damage_interval` | `sCvarVal` | 字符串或表达式 | 0.01 ~ 15.0 | 拖拽造成伤害的频率，允许值 0.01~15.0 | 服务器端本插件逻辑 | `l4d2_smoker_drag_damage_interval.sp:110` |
| `tongue_drag_first_damage_interval` | `-1.0` | 整数，源码写作浮点 | ≤ 15.0 | 首次伤害在多少秒后施加，0.0 关闭，最大 15.0 | 服务器端本插件逻辑 | `l4d2_smoker_drag_damage_interval.sp:111` |
| `tongue_drag_first_damage` | `-1.0` | 整数，源码写作浮点 | ≤ 100.0 | 舌头首次命中时施加的伤害，0.0 关闭，最大 100.0 | 服务器端本插件逻辑 | `l4d2_smoker_drag_damage_interval.sp:112` |
| `tongue_damage_continuity` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否在拖拽与勒喉切换之间保留伤害计时：0 原生行为，1 保留剩余时间 | 服务器端本插件逻辑 | `l4d2_smoker_drag_damage_interval.sp:113` |

## L4D2 Sniper Hunter Bodyshot —— `optional/l4d2_sniper_bodyshot.sp`

- 源文件：`optional/l4d2_sniper_bodyshot.sp`
- myinfo：version=2.2，author=Visor, A1m`
- myinfo description（源码原文）：Remove sniper weapons' stomach hitgroup damage multiplier against hunters

### 指令

（本插件未注册指令）

### ConVar

（本插件未注册 ConVar）

## L4D2 Sniper Precache —— `optional/l4d2_sniper_precache.sp`

- 源文件：`optional/l4d2_sniper_precache.sp`
- myinfo：version=2.2.1，author=Visor, A1m`
- myinfo description（源码原文）：Unlocks German sniper weapons

### 指令

（本插件未注册指令）

### ConVar

（本插件未注册 ConVar）

## Sound Manipulation: REWORK —— `optional/l4d2_sound_manipulation.sp`

- 源文件：`optional/l4d2_sound_manipulation.sp`
- myinfo：version=1.0，author=Sir
- myinfo description（源码原文）：Allows control over certain sounds
- HookConVarChange：`cvarSoundFlags`→`FlagsChanged`

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `sound_flags` | `0` | 整数 | 无上下界 | 屏蔽声音的位掩码：0 无，1 心跳，2 重型可击打物音效，4 倒地受伤音效，其余被截断 | 服务器端本插件逻辑 | `l4d2_sound_manipulation.sp:25` |

## L4D2 Various Sounds Blocker —— `optional/l4d2_sounds_blocker.sp`

- 源文件：`optional/l4d2_sounds_blocker.sp`
- myinfo：version=1.3.2，author=Spoon, A1m`
- myinfo description（源码原文）：Blocks out more annoying sounds and allows the option for blocking custom sounds. Designed for NextMod Config.

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `ssb_custom_path` | <路径> | 服务器控制台命令，玩家无法使用 | 设置自定义音效路径，服务器控制台命令 | `l4d2_sounds_blocker.sp:87` |
| `ssb_whitelist_path` | <路径> | 服务器控制台命令，玩家无法使用 | 设置白名单路径，服务器控制台命令 | `l4d2_sounds_blocker.sp:88` |

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `ssb_block_fireworks` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否屏蔽烟花音效 | 服务器端本插件逻辑 | `l4d2_sounds_blocker.sp:76` |
| `ssb_block_coaster` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否屏蔽过山车音效 | 服务器端本插件逻辑 | `l4d2_sounds_blocker.sp:77` |
| `ssb_block_car_alarms` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否屏蔽汽车警报音效 | 服务器端本插件逻辑 | `l4d2_sounds_blocker.sp:78` |
| `ssb_block_alarms` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否屏蔽警报音效 | 服务器端本插件逻辑 | `l4d2_sounds_blocker.sp:79` |
| `ssb_block_horde` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否屏蔽尸潮音效 | 服务器端本插件逻辑 | `l4d2_sounds_blocker.sp:80` |
| `ssb_block_misc_vehicles` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否屏蔽杂物车辆音效，如教区第四关拖拉机、死亡机场终局飞机等 | 服务器端本插件逻辑 | `l4d2_sounds_blocker.sp:81` |
| `ssb_block_generators` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否屏蔽发电机音效 | 服务器端本插件逻辑 | `l4d2_sounds_blocker.sp:82` |
| `ssb_block_ambient_explosions` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否屏蔽环境爆炸音效 | 服务器端本插件逻辑 | `l4d2_sounds_blocker.sp:83` |
| `ssb_block_lifts` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否屏蔽电梯音效 | 服务器端本插件逻辑 | `l4d2_sounds_blocker.sp:84` |
| `ssb_block_laughs` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否屏蔽笑声 | 服务器端本插件逻辑 | `l4d2_sounds_blocker.sp:85` |

## L4D2 Spit Blocker —— `optional/l4d2_spitblock.sp`

- 源文件：`optional/l4d2_spitblock.sp`
- myinfo：version=2.3，author=ProdigySim, Estoopi, Jacob, Visor, A1m`
- myinfo description（源码原文）：Blocks spit damage on various maps

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `spit_block_square` | <参数> | 服务器控制台命令，玩家无法使用 | 设置毒痰阻挡方块，服务器控制台命令 | `l4d2_spitblock.sp:46` |
| `spit_remove_block_square` | <参数> | 服务器控制台命令，玩家无法使用 | 移除毒痰阻挡方块，服务器控制台命令 | `l4d2_spitblock.sp:47` |

### ConVar

（本插件未注册 ConVar）

## L4D2 Static Shotgun Spread —— `optional/l4d2_static_shotgun_spread.sp`

- 源文件：`optional/l4d2_static_shotgun_spread.sp`
- myinfo：version=1.6.5，author=Jahze, Visor, A1m`, Rena
- myinfo description（源码原文）：Changes the values in the sgspread patch

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `sgspread_ring1_bullets` | `3` | 整数 | 无上下界 | 第一圈环的弹丸数量，其余弹丸进入第二圈 | 服务器端本插件逻辑 | `l4d2_static_shotgun_spread.sp:76` |
| `sgspread_ring1_factor` | `2` | 整数 | 无上下界 | 第一圈环的弹丸距中心的远近系数 | 服务器端本插件逻辑 | `l4d2_static_shotgun_spread.sp:77` |
| `sgspread_center_pellet` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 中心弹丸：0 关闭，1 开启 | 服务器端本插件逻辑 | `l4d2_static_shotgun_spread.sp:78` |

## L4D2 Realtime Stats —— `optional/l4d2_stats.sp`

- 源文件：`optional/l4d2_stats.sp`
- myinfo：version=1.2.3，author=Griffin, Philogl, Sir, A1m`
- myinfo description（源码原文）：Display Skeets/Etc to Chat to clients

### 指令

（本插件未注册指令）

### ConVar

（本插件未注册 ConVar）

## [L4D2] Steady Boost —— `optional/l4d2_steady_boost.sp`

- 源文件：`optional/l4d2_steady_boost.sp`
- myinfo：author=Forgetest
- myinfo description（源码原文）：Prevent forced sliding when landing at head of enemies.

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `l4d2_steady_boost_flags` | `3` | 整数，源码写作浮点 | 0.0 ~ 3.0 | 哪些队伍可以使用稳定加成：1 生还者，2 特感，3 全部，0 禁用 | 服务器端本插件逻辑；仅服务器 | `l4d2_steady_boost.sp:39` |

## L4D2 Tank Announcer —— `optional/l4d2_tank_announce.sp`

- 源文件：`optional/l4d2_tank_announce.sp`
- myinfo：author=Visor, Forgetest, xoxo
- myinfo description（源码原文）：Announce in chat and via a sound when a Tank has spawned

### 指令

（本插件未注册指令）

### ConVar

（本插件未注册 ConVar）

## Tank Attack Control —— `optional/l4d2_tank_attack_control.sp`

- 源文件：`optional/l4d2_tank_attack_control.sp`
- myinfo：version=0.8，author=vintik, CanadaRox, Jacob, Visor, Forgetest

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_underhand` | 无参数 | 无（任意玩家可用） | 切换坦克为下手投掷石头，仅坦克可用 | `l4d2_tank_attack_control.sp:60` |
| `sm_overhand` | 无参数 | 无（任意玩家可用） | 切换坦克为上手投掷石头，仅坦克可用 | `l4d2_tank_attack_control.sp:61` |
| `sm_overonehand` | 无参数 | 无（任意玩家可用） | 切换坦克为单手投掷石头，仅坦克可用 | `l4d2_tank_attack_control.sp:62` |

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `l4d2_block_punch_rock` | `1` | 整数 | 无上下界 | 是否阻止坦克同时拳击与投掷石头 | 服务器端本插件逻辑 | `l4d2_tank_attack_control.sp:50` |
| `l4d2_block_jump_rock` | `0` | 整数 | 无上下界 | 是否阻止坦克同时跳跃与投掷石头 | 服务器端本插件逻辑 | `l4d2_tank_attack_control.sp:51` |
| `tank_overhand_only` | `0` | 整数 | 无上下界 | 是否强制坦克只投掷上手石头 | 服务器端本插件逻辑 | `l4d2_tank_attack_control.sp:52` |

## L4D2 Tank & Charger M2 Fix —— `optional/l4d2_tank_charger_m2_fix.sp`

- 源文件：`optional/l4d2_tank_charger_m2_fix.sp`
- myinfo：version=1.0，author=Sir, Visor
- myinfo description（源码原文）：Stops Shoves slowing the Tank and Charger Down

### 指令

（本插件未注册指令）

### ConVar

（本插件未注册 ConVar）

## L4D2 Tank Damage Cvars —— `optional/l4d2_tank_damage_cvars.sp`

- 源文件：`optional/l4d2_tank_damage_cvars.sp`
- myinfo：version=2.1，author=Visor, A1m`
- myinfo description（源码原文）：Toggle Tank attack damage per type

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `vs_tank_pound_damage` | `24.0` | 整数，源码写作浮点 | 无上下界 | 对战模式坦克近战攻击对倒地生还者造成的伤害，0 或负值关闭 | 服务器端本插件逻辑 | `l4d2_tank_damage_cvars.sp:36` |
| `vs_tank_rock_damage` | `24.0` | 整数，源码写作浮点 | 无上下界 | 对战模式坦克石头造成的伤害，0 或负值关闭 | 服务器端本插件逻辑 | `l4d2_tank_damage_cvars.sp:37` |

## L4D2 Tank Horde Monitor —— `optional/l4d2_tank_horde_monitor.sp`

- 源文件：`optional/l4d2_tank_horde_monitor.sp`
- myinfo：version=1.3.1，author=Derpduck, Visor (l4d2_horde_equaliser)
- myinfo description（源码原文）：Monitors and changes state of infinite hordes during tanks

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `l4d2_tank_bypass_extra_flow` | `1500.0` | 整数，源码写作浮点 | ≥ 0.0 | 无限事件期间允许额外绕过坦克的流程距离，0 关闭 | 服务器端本插件逻辑 | `l4d2_tank_horde_monitor.sp:58` |

## L4D2 Tank Melee Fury —— `optional/l4d2_tank_melee_fury.sp`

- 源文件：`optional/l4d2_tank_melee_fury.sp`
- myinfo：version=1.0，author=Visor
- myinfo description（源码原文）：Aggressive melee Survivors are almost certain to get punished for excessively pushing the Tank

### 指令

（本插件未注册指令）

### ConVar

（本插件未注册 ConVar）

## L4D2 Tank Hittable Glow —— `optional/l4d2_tank_props_glow.sp`

- 源文件：`optional/l4d2_tank_props_glow.sp`
- myinfo：version=2.5.1，author=Harry Potter, Sir, A1m`, Derpduck
- myinfo description（源码原文）：Stop tank props from fading whilst the tank is alive + add Hittable Glow.

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `l4d_tank_props_glow` | `1` | 整数 | 无上下界 | 坦克存活期间是否向特感队伍显示可击打物轮廓 | 服务器端本插件逻辑 | `l4d2_tank_props_glow.sp:54` |
| `l4d2_tank_prop_glow_color` | `255 255 255` | 字符串或表达式 | 无上下界 | 可击打物轮廓颜色，三个 0-255 数值以空格分隔的 RGB | 服务器端本插件逻辑 | `l4d2_tank_props_glow.sp:55` |
| `l4d2_tank_prop_glow_range` | `4500` | 整数 | 无上下界 | 玩家需要靠近可击打物到多少距离才启用轮廓 | 服务器端本插件逻辑 | `l4d2_tank_props_glow.sp:56` |
| `l4d2_tank_prop_glow_range_min` | `256` | 整数 | 无上下界 | 玩家靠近到该距离以内则关闭轮廓 | 服务器端本插件逻辑 | `l4d2_tank_props_glow.sp:57` |
| `l4d2_tank_prop_glow_only` | `0` | 整数 | 无上下界 | 是否只有坦克能看到轮廓 | 服务器端本插件逻辑 | `l4d2_tank_props_glow.sp:58` |
| `l4d2_tank_prop_glow_spectators` | `1` | 整数 | 无上下界 | 旁观者是否也能看到轮廓 | 服务器端本插件逻辑 | `l4d2_tank_props_glow.sp:59` |
| `l4d2_tank_prop_dissapear_time` | `10.0` | 整数，源码写作浮点 | 无上下界 | 被坦克击打过的可击打物在坦克死亡后消失所需时间 | 服务器端本插件逻辑 | `l4d2_tank_props_glow.sp:60` |

## L4D2 Tank Rage —— `optional/l4d2_tankrage.sp`

- 源文件：`optional/l4d2_tankrage.sp`
- myinfo：version=1.0.3，author=Sir
- myinfo description（源码原文）：Manage Tank Rage when Survivors are running back.

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `l4d2_tankrage_flowpercent` | `7` | 整数 | 无上下界 | 生还者回跑到该流程百分比时给予挫败感冻结，以最远生还者计 | 服务器端本插件逻辑 | `l4d2_tankrage.sp:34` |
| `l4d2_tankrage_freezetime` | `4.0` | 整数，源码写作浮点 | 无上下界 | 生还者回跑达到该百分比后冻结坦克挫败感的秒数 | 服务器端本插件逻辑 | `l4d2_tankrage.sp:35` |
| `l4d2_tankrage_debug` | `0` | 整数 | 无上下界 | 是否输出调试信息 | 服务器端本插件逻辑 | `l4d2_tankrage.sp:36` |

## Tongue Timer —— `optional/l4d2_tongue_timer.sp`

- 源文件：`optional/l4d2_tongue_timer.sp`
- myinfo：version=1.3-anne.1，author=Sir
- myinfo description（源码原文）：Modify the Smoker's tongue ability timer in certain scenarios.

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `l4d2_tongue_delay_tank` | `8.0` | 整数，源码写作浮点 | 无上下界 | Smoker 被坦克拳击或石头快速解救后的冷却秒数，原生约 0.5 秒 | 服务器端本插件逻辑 | `l4d2_tongue_timer.sp:37` |
| `l4d2_tongue_delay_survivor` | `4.0` | 整数，源码写作浮点 | 无上下界 | Smoker 被生还者快速解救后的冷却秒数，原生约 0.5 秒 | 服务器端本插件逻辑 | `l4d2_tongue_timer.sp:38` |

## L4D2 Ultra Witch —— `optional/l4d2_ultra_witch.sp`

- 源文件：`optional/l4d2_ultra_witch.sp`
- myinfo：version=1.2.2，author=Visor, A1m`
- myinfo description（源码原文）：The Witch's hit deals a set amount of damage instead of instantly incapping, while also sending the survivor flying. Fixes convar z_witch_damage

### 指令

（本插件未注册指令）

### ConVar

（本插件未注册 ConVar）

## [L4D2] Uncommon Adjustment —— `optional/l4d2_uncommon_adjustment.sp`

- 源文件：`optional/l4d2_uncommon_adjustment.sp`
- myinfo：author=Forgetest
- myinfo description（源码原文）：Custom adjustments to uncommon infected.

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `l4d2_uncommon_attract` | `3` | 整数，源码写作浮点 | 0.0 ~ 3.0 | 小丑与 Jimmy Gibbs Jr. 能否吸引僵尸：0 都不能，1 小丑，2 Jimmy，3 两者 | 服务器端本插件逻辑；经 CreateConVarHook 包装并注册变更回调 | `l4d2_uncommon_adjustment.sp:81` |
| `l4d2_roadworker_sense_flag` | `0` | 开关 0 或 1 | 0.0 ~ 3.0 | 修路工能否听见或闻到吸引物：0 都不能，1 听管式炸弹与小丑，2 闻呕吐瓶，3 两者 | 服务器端本插件逻辑；经 CreateConVarHook 包装并注册变更回调 | `l4d2_uncommon_adjustment.sp:89` |
| `l4d2_jimmy_sense_flag` | `0` | 开关 0 或 1 | 0.0 ~ 3.0 | Jimmy Gibbs Jr. 能否听见或闻到吸引物：0 都不能，1 听，2 闻，3 两者 | 服务器端本插件逻辑；经 CreateConVarHook 包装并注册变更回调 | `l4d2_uncommon_adjustment.sp:97` |
| `l4d2_uncommon_health_multiplier` | `3.0` | 整数，源码写作浮点 | ≥ 0.0 | 非常见感染者的生命倍率，不适用于 Jimmy、fallen 与防暴警 | 服务器端本插件逻辑；经 CreateConVarHook 包装并注册变更回调 | `l4d2_uncommon_adjustment.sp:105` |
| `l4d2_jimmy_health_multiplier` | `20.0` | 整数，源码写作浮点 | ≥ 0.0 | Jimmy Gibbs Jr. 的生命倍率 | 服务器端本插件逻辑；经 CreateConVarHook 包装并注册变更回调 | `l4d2_uncommon_adjustment.sp:113` |
| `l4d2_fallen_equipments` | `15` | 整数，源码写作浮点 | 0.0 ~ 15.0 | fallen survivor 可携带的物品：1 燃烧瓶，2 管式炸弹，4 止痛药，8 医疗包，15 全部，0 无 | 服务器端本插件逻辑；经 CreateConVarHook 包装并注册变更回调 | `l4d2_uncommon_adjustment.sp:120` |
| `l4d2_riotcop_armor` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 防暴警是否拥有可抵挡正面伤害的护甲 | 服务器端本插件逻辑；经 CreateConVarHook 包装并注册变更回调 | `l4d2_uncommon_adjustment.sp:128` |
| `l4d2_mudman_crouch_run` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 泥人是否能边跑边蹲下 | 服务器端本插件逻辑；经 CreateConVarHook 包装并注册变更回调 | `l4d2_uncommon_adjustment.sp:135` |
| `l4d2_mudman_screen_splatter` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 泥人是否能遮蔽玩家屏幕 | 服务器端本插件逻辑；经 CreateConVarHook 包装并注册变更回调 | `l4d2_uncommon_adjustment.sp:142` |
| `l4d2_jimmy_screen_splatter` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | Jimmy Gibbs Jr. 是否能遮蔽玩家屏幕 | 服务器端本插件逻辑；经 CreateConVarHook 包装并注册变更回调 | `l4d2_uncommon_adjustment.sp:149` |
| `l4d2_ceda_fire_proof` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | CEDA 是否防火 | 服务器端本插件逻辑；经 CreateConVarHook 包装并注册变更回调 | `l4d2_uncommon_adjustment.sp:156` |

## Uncommon Infected Blocker —— `optional/l4d2_uncommon_blocker.sp`

- 源文件：`optional/l4d2_uncommon_blocker.sp`
- myinfo：version=2.3.1，author=Tabun, A1m`
- myinfo description（源码原文）：Blocks uncommon infected from ruining your day.

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_uncinfblock_check` | 无参数 | 需要 ADMFLAG_GENERIC（b，通用管理员） | 检查非常见感染者屏蔽状态 | `l4d2_uncommon_blocker.sp:121` |

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `sm_uncinfblock_enabled` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用非常见感染者屏蔽插件 | 服务器端本插件逻辑 | `l4d2_uncommon_blocker.sp:110` |
| `sm_uncinfblock_flags` | `55` | 整数，源码写作浮点 | 1.0 ~ 127.0 | 屏蔽哪些非常见感染者，位域 1~127：1 ceda，2 泥人，4 工兵，8 fallen，16 防暴警，32 小丑，其余被截断 | 服务器端本插件逻辑 | `l4d2_uncommon_blocker.sp:114` |

## L4D2 Uniform Spit —— `optional/l4d2_uniform_spit.sp`

- 源文件：`optional/l4d2_uniform_spit.sp`
- myinfo：version=1.5.1，author=Visor, Sir, A1m`
- myinfo description（源码原文）：Make the spit deal a set amount of DPS under all circumstances

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `l4d2_spit_dmg` | `-1.0` | 整数，源码写作浮点 | 无上下界 | 毒痰每跳造成的伤害，-1 表示不调整伤害 | 服务器端本插件逻辑 | `l4d2_uniform_spit.sp:68` |
| `l4d2_spit_alternate_dmg` | `-1.0` | 整数，源码写作浮点 | 无上下界 | 交替跳的伤害，-1 表示关闭 | 服务器端本插件逻辑 | `l4d2_uniform_spit.sp:69` |
| `l4d2_spit_max_ticks` | `28` | 整数 | 无上下界 | 酸液伤害跳数的上限 | 服务器端本插件逻辑 | `l4d2_uniform_spit.sp:70` |
| `l4d2_spit_godframe_ticks` | `4` | 整数 | 无上下界 | 初始处于神圣帧的酸液跳数 | 服务器端本插件逻辑 | `l4d2_uniform_spit.sp:71` |

## Unsilent Jockey —— `optional/l4d2_unsilent_jockey.sp`

- 源文件：`optional/l4d2_unsilent_jockey.sp`
- myinfo：version=0.8，author=Tabun, robex, Sir, A1m`
- myinfo description（源码原文）：Makes jockeys emit sound constantly.

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `sm_unsilentjockey_interval` | `2.0` | 整数，源码写作浮点 | 无上下界 | 强制播放 Jockey 音效的间隔秒数 | 服务器端本插件逻辑 | `l4d2_unsilent_jockey.sp:77` |

## L4D2 Weapon Attributes —— `optional/l4d2_weapon_attributes.sp`

- 源文件：`optional/l4d2_weapon_attributes.sp`
- myinfo：version=3.1.0，author=Jahze, A1m`, Forgetest
- myinfo description（源码原文）：Allowing tweaking of the attributes of all weapons

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_weapon` | <武器> <属性> <数值> | 服务器控制台命令，玩家无法使用 | 设置近战武器属性，服务器控制台命令 | `l4d2_weapon_attributes.sp:251` |
| `sm_weapon_attributes_reset` | 无参数 | 服务器控制台命令，玩家无法使用 | 重置所有近战武器属性，服务器控制台命令 | `l4d2_weapon_attributes.sp:252` |
| `sm_weaponstats` | [武器] | 无（任意玩家可用） | 显示武器统计 | `l4d2_weapon_attributes.sp:254` |
| `sm_weapon_attributes` | [武器] | 无（任意玩家可用） | 显示武器属性 | `l4d2_weapon_attributes.sp:255` |

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `sm_weapon_hide_attributes` | `2` | 整数，源码写作浮点 | 0.0 ~ 2.0 | 自定义 sm_weapon_attributes 命令：0 禁用命令，1 仅管理员可看武器属性，2 其余被截断 | 服务器端本插件逻辑 | `l4d2_weapon_attributes.sp:233` |

## L4D2 Weapon Rules —— `optional/l4d2_weaponrules.sp`

- 源文件：`optional/l4d2_weaponrules.sp`
- myinfo：version=1.0.2，author=ProdigySim
- myinfo description（源码原文）：^

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `l4d2_addweaponrule` | <匹配> <替换> | 服务器控制台命令，玩家无法使用 | 添加武器替换规则，服务器控制台命令 | `l4d2_weaponrules.sp:50` |
| `l4d2_resetweaponrules` | 无参数 | 服务器控制台命令，玩家无法使用 | 重置所有武器替换规则，服务器控制台命令 | `l4d2_weaponrules.sp:51` |

### ConVar

（本插件未注册 ConVar）

## L4D2Lib —— `optional/l4d2lib.sp`

- 源文件：`optional/l4d2lib/survivors.sp`、`optional/l4d2lib/rounds.sp`、`optional/l4d2lib.sp`、`optional/l4d2lib/tanks.sp`、`optional/l4d2lib/mapinfo.sp`
- myinfo：version=3.2.1，author=Confogl Team
- myinfo description（源码原文）：Useful natives and fowards for L4D2 Plugins

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `confogl_midata_save` | 无参数 | 需要 ADMFLAG_CONFIG（h，服务器配置） | 保存当前地图的物资点配置，L4D2Lib 版本 | `mapinfo.sp:54` |
| `confogl_save_location` | <点位类型> | 需要 ADMFLAG_CONFIG（h，服务器配置） | 把当前位置坐标保存到物资点配置，L4D2Lib 版本 | `mapinfo.sp:55` |

### ConVar

（本插件未注册 ConVar）

## L4D2 Bash Kills —— `optional/l4d_bash_kills.sp`

- 源文件：`optional/l4d_bash_kills.sp`
- myinfo：version=1.4，author=Jahze, A1m`
- myinfo description（源码原文）：Stop special infected getting bashed to death

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `l4d_no_bash_kills` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否阻止特感被推击致死 | 服务器端本插件逻辑 | `l4d_bash_kills.sp:37` |

## L4D Black and White Notifier —— `optional/l4d_blackandwhite.sp`

- 源文件：`optional/l4d_blackandwhite.sp`
- myinfo：author=DarkNoghri, madcap
- myinfo description（源码原文）：Notify people when player is black and white.
- HookConVarChange：`h_cvarNoticeType`→`ChangeVars`、`h_cvarPrintType`→`ChangeVars`、`h_cvarGlowEnable`→`ChangeVars`

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `l4d_blackandwhite_version` | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | 服务器端本插件逻辑；复制到客户端 | `l4d_blackandwhite.sp:37` |
| `l4d_bandw_notice` | `1` | 整数，源码写作浮点 | 0.0 ~ 3.0 | 0 关闭提示，1 通知生还者，2 通知所有人，3 通知特感 | 服务器端本插件逻辑 | `l4d_blackandwhite.sp:45` |
| `l4d_bandw_type` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 0 输出到聊天，1 显示提示框 | 服务器端本插件逻辑 | `l4d_blackandwhite.sp:46` |
| `l4d_bandw_glow` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 0 关闭发光，1 开启发光 | 服务器端本插件逻辑 | `l4d_blackandwhite.sp:47` |

## [L4D2] Boss Percents/Vote Boss Hybrid —— `optional/l4d_boss_percent.sp`

- 源文件：`optional/l4d_boss_percent.sp`
- myinfo：author=Spoon, Forgetest
- myinfo description（源码原文）：Displays Boss Flows on Ready-Up and via command. Remade for NextMod.

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_boss` | 无参数 | 无（任意玩家可用） | 显示 boss 刷新百分比 | `l4d_boss_percent.sp:105` |
| `sm_tank` | 无参数 | 无（任意玩家可用） | 显示 Tank 刷新百分比 | `l4d_boss_percent.sp:106` |
| `sm_witch` | 无参数 | 无（任意玩家可用） | 显示 Witch 刷新百分比 | `l4d_boss_percent.sp:107` |

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `l4d_global_percent` | `0` | 整数 | 无上下界 | 使用命令时是否向全队显示 boss 百分比 | 服务器端本插件逻辑 | `l4d_boss_percent.sp:94` |
| `l4d_tank_percent` | `1` | 整数 | 无上下界 | 是否在聊天中显示 Tank 流程百分比 | 服务器端本插件逻辑 | `l4d_boss_percent.sp:95` |
| `l4d_witch_percent` | `1` | 整数 | 无上下界 | 是否在聊天中显示 Witch 流程百分比 | 服务器端本插件逻辑 | `l4d_boss_percent.sp:96` |

## [L4D2] Vote Boss —— `optional/l4d_boss_vote.sp`

- 源文件：`optional/l4d_boss_vote.sp`
- myinfo：author=Spoon, Forgetest
- myinfo description（源码原文）：Votin for boss change.

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_voteboss` | <是否坦克 0或1> <百分比> | 无（任意玩家可用） | 发起坦克或女巫刷新百分比投票 | `l4d_boss_vote.sp:46` |
| `sm_bossvote` | <是否坦克 0或1> <百分比> | 无（任意玩家可用） | sm_voteboss 的别名 | `l4d_boss_vote.sp:47` |
| `sm_ftank` | <百分比> | 需要 ADMFLAG_BAN（d，封禁） | 管理员强制设置坦克刷新百分比 | `l4d_boss_vote.sp:49` |
| `sm_fwitch` | <百分比> | 需要 ADMFLAG_BAN（d，封禁） | 管理员强制设置女巫刷新百分比 | `l4d_boss_vote.sp:50` |

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `l4d_boss_vote` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用 boss 投票 | 服务器端本插件逻辑 | `l4d_boss_vote.sp:44` |

## SI - CI FF Block —— `optional/l4d_ci_ffblock.sp`

- 源文件：`optional/l4d_ci_ffblock.sp`
- myinfo：version=1.1，author=Sir
- myinfo description（源码原文）：Modifies FF from SI (Except Tank) to CI

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `common_hits` | `5` | 整数 | 无上下界 | 特感击杀普通感染者所需命中次数，0 屏蔽友军伤害，5 为 L4D1 风格 | 服务器端本插件逻辑 | `l4d_ci_ffblock.sp:35` |

## Common Ragdolls be gone —— `optional/l4d_common_ragdolls_be_gone.sp`

- 源文件：`optional/l4d_common_ragdolls_be_gone.sp`
- myinfo：version=1.1，author=Sir
- myinfo description（源码原文）：Make ragdolls for common infected vanish into thin air server-side on death.

### 指令

（本插件未注册指令）

### ConVar

（本插件未注册 ConVar）

## L4D2 Equalise Alarm Cars —— `optional/l4d_equalise_alarm_cars.sp`

- 源文件：`optional/l4d_equalise_alarm_cars.sp`
- myinfo：author=Jahze, Forgetest
- myinfo description（源码原文）：Make the alarmed car and its color spawns the same for each team in versus

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `l4d_equalise_alarm_start_disabled` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 游戏正式开始前警报车是否以禁用状态生成 | 服务器端本插件逻辑 | `l4d_equalise_alarm_cars.sp:93` |
| `l4d_equalise_alarm_debug` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否输出警报车调试信息 | 服务器端本插件逻辑；仅服务器；需 sv_cheats 方可修改；隐藏变量 | `l4d_equalise_alarm_cars.sp:94` |

## L4D2 Jockey Ledge Hang Recharge —— `optional/l4d_jockey_ledgehang.sp`

- 源文件：`optional/l4d_jockey_ledgehang.sp`
- myinfo：version=1.3，author=Jahze, A1m`
- myinfo description（源码原文）：Adds a cvar to adjust the recharge timer of a jockey after he ledge hangs a survivor.

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `z_leap_interval_post_ledge_hang` | `10` | 整数 | 无上下界 | Jockey 挂边后再次跳跃前的等待秒数 | 服务器端本插件逻辑 | `l4d_jockey_ledgehang.sp:25` |

## L4D(2) map-based convar loader. —— `optional/l4d_mapbased_cvars.sp`

- 源文件：`optional/l4d_mapbased_cvars.sp`
- myinfo：version=0.1b，author=Tabun
- myinfo description（源码原文）：Loads convars on map-load, based on currently active map and confogl config.
- HookConVarChange：`g_hUseConfigDir`→`CvarConfigChange`

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `l4d_mapcvars_configdir` | `` | 字符串或表达式 | 无上下界 | 使用哪个 cfgogl 配置 | 服务器端本插件逻辑 | `l4d_mapbased_cvars.sp:44` |

## L4D2 Remove Cans —— `optional/l4d_no_cans.sp`

- 源文件：`optional/l4d_no_cans.sp`
- myinfo：version=1.0，author=Jahze, Sir, A1m`
- myinfo description（源码原文）：Provides the ability to remove Gascans, Propane, Oxygen Tanks and Fireworks

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `l4d_no_cans` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否移除汽油桶 | 服务器端本插件逻辑 | `l4d_no_cans.sp:33` |
| `l4d_no_propane` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否移除丙烷罐 | 服务器端本插件逻辑 | `l4d_no_cans.sp:34` |
| `l4d_no_oxygen` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否移除氧气罐 | 服务器端本插件逻辑 | `l4d_no_cans.sp:35` |
| `l4d_no_fireworks` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否移除烟花 | 服务器端本插件逻辑 | `l4d_no_cans.sp:36` |

## L4D2 Pounce Protect —— `optional/l4d_pounceprotect.sp`

- 源文件：`optional/l4d_pounceprotect.sp`
- myinfo：version=1.1，author=ProdigySim
- myinfo description（源码原文）：Prevent damage from blocking a hunter's ability to pounce

### 指令

（本插件未注册指令）

### ConVar

（本插件未注册 ConVar）

## [L4D & 2] Return Thrown Items —— `optional/l4d_return_thrown_items.sp`

- 源文件：`optional/l4d_return_thrown_items.sp`
- myinfo：author=Forgetest
- myinfo description（源码原文）：Return pills/adrenalines thrown through shoving key if not successfully given.

### 指令

（本插件未注册指令）

### ConVar

（本插件未注册 ConVar）

## L4D(2) Tank Rock Lag Compensation —— `optional/l4d_rock_lagcomp.sp`

- 源文件：`optional/l4d_rock_lagcomp.sp`
- myinfo：version=1.13，author=Luckylockm,harry,Silvers
- myinfo description（源码原文）：Provides lag compensation for tank rock entities

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `sm_rock_print` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否打印石头伤害与距离数值 | 服务器端本插件逻辑 | `l4d_rock_lagcomp.sp:151` |
| `sm_rock_hitbox` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用石头自定义命中盒 | 服务器端本插件逻辑 | `l4d_rock_lagcomp.sp:152` |
| `sm_rock_lagcomp` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用延迟补偿 | 服务器端本插件逻辑 | `l4d_rock_lagcomp.sp:153` |
| `sm_rock_godframes` | `1.7` | 浮点 | 0.0 ~ 10.0 | 石头的神圣帧时间，单位秒 | 服务器端本插件逻辑 | `l4d_rock_lagcomp.sp:154` |
| `sm_rock_godframes_render` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否显示神圣帧视觉反馈 | 服务器端本插件逻辑 | `l4d_rock_lagcomp.sp:155` |
| `sm_rock_hitbox_radius` | `30` | 整数，源码写作浮点 | 0.0 ~ 10000.0 | 石头命中盒半径 | 服务器端本插件逻辑 | `l4d_rock_lagcomp.sp:156` |
| `sm_rock_damage_pistol` | `75` | 整数，源码写作浮点 | 0.0 ~ DAMAGE_MAX_ALL_ | 手枪类别的石头即时命中伤害，上限为 DAMAGE_MAX_ALL_ 宏 | 服务器端本插件逻辑 | `l4d_rock_lagcomp.sp:158` |
| `sm_rock_damage_magnum` | `1000` | 整数，源码写作浮点 | 0.0 ~ DAMAGE_MAX_ALL_ | 马格南类别的石头伤害 | 服务器端本插件逻辑 | `l4d_rock_lagcomp.sp:159` |
| `sm_rock_damage_shotgun` | `600` | 整数，源码写作浮点 | 0.0 ~ DAMAGE_MAX_ALL_ | 霰弹枪类别的石头伤害 | 服务器端本插件逻辑 | `l4d_rock_lagcomp.sp:160` |
| `sm_rock_damage_smg` | `75` | 整数，源码写作浮点 | 0.0 ~ DAMAGE_MAX_ALL_ | SMG 类别的石头伤害 | 服务器端本插件逻辑 | `l4d_rock_lagcomp.sp:161` |
| `sm_rock_damage_rifle` | `200` | 整数，源码写作浮点 | 0.0 ~ DAMAGE_MAX_ALL_ | 步枪类别的石头伤害 | 服务器端本插件逻辑 | `l4d_rock_lagcomp.sp:162` |
| `sm_rock_damage_melee` | `1000` | 整数，源码写作浮点 | 0.0 ~ DAMAGE_MAX_ALL_ | 近战类别的石头伤害 | 服务器端本插件逻辑 | `l4d_rock_lagcomp.sp:163` |
| `sm_rock_damage_sniper` | `10000` | 整数，源码写作浮点 | 0.0 ~ DAMAGE_MAX_ALL_ | 狙击类别的石头伤害 | 服务器端本插件逻辑 | `l4d_rock_lagcomp.sp:164` |
| `sm_rock_damage_minigun` | `300` | 整数，源码写作浮点 | 0.0 ~ DAMAGE_MAX_ALL_ | 转轮机枪类别的石头伤害 | 服务器端本插件逻辑 | `l4d_rock_lagcomp.sp:165` |
| `sm_rock_damage_mounted_machinegun` | `10000` | 整数，源码写作浮点 | 0.0 ~ DAMAGE_MAX_ALL_ | 固定机枪类别的石头伤害 | 服务器端本插件逻辑 | `l4d_rock_lagcomp.sp:166` |
| `sm_rock_range_min_all` | `1` | 整数，源码写作浮点 | 0.0 ~ RANGE_MAX_ALL_ | 石头伤害的全局最小距离，上限为 RANGE_MAX_ALL_ 宏 | 服务器端本插件逻辑 | `l4d_rock_lagcomp.sp:168` |
| `sm_rock_range_max_all` | `2000` | 整数，源码写作浮点 | 0.0 ~ RANGE_MAX_ALL_ | 石头伤害的全局最大距离 | 服务器端本插件逻辑 | `l4d_rock_lagcomp.sp:169` |
| `sm_rock_range_pistol` | `2000` | 整数，源码写作浮点 | 0.0 ~ RANGE_MAX_ALL_ | 手枪类别可造成石头伤害的最大距离 | 服务器端本插件逻辑 | `l4d_rock_lagcomp.sp:170` |
| `sm_rock_range_magnum` | `2000` | 整数，源码写作浮点 | 0.0 ~ RANGE_MAX_ALL_ | 马格南类别可造成石头伤害的最大距离 | 服务器端本插件逻辑 | `l4d_rock_lagcomp.sp:171` |
| `sm_rock_range_shotgun` | `1000` | 整数，源码写作浮点 | 0.0 ~ RANGE_MAX_ALL_ | 霰弹枪类别可造成石头伤害的最大距离 | 服务器端本插件逻辑 | `l4d_rock_lagcomp.sp:172` |
| `sm_rock_range_smg` | `2000` | 整数，源码写作浮点 | 0.0 ~ RANGE_MAX_ALL_ | SMG 类别可造成石头伤害的最大距离 | 服务器端本插件逻辑 | `l4d_rock_lagcomp.sp:173` |
| `sm_rock_range_rifle` | `2000` | 整数，源码写作浮点 | 0.0 ~ RANGE_MAX_ALL_ | 步枪类别可造成石头伤害的最大距离 | 服务器端本插件逻辑 | `l4d_rock_lagcomp.sp:174` |
| `sm_rock_range_melee` | `200` | 整数，源码写作浮点 | 0.0 ~ RANGE_MAX_ALL_ | 近战类别可造成石头伤害的最大距离 | 服务器端本插件逻辑 | `l4d_rock_lagcomp.sp:175` |
| `sm_rock_range_sniper` | `10000` | 整数，源码写作浮点 | 0.0 ~ RANGE_MAX_ALL_ | 狙击类别可造成石头伤害的最大距离 | 服务器端本插件逻辑 | `l4d_rock_lagcomp.sp:176` |
| `sm_rock_range_minigun` | `2000` | 整数，源码写作浮点 | 0.0 ~ RANGE_MAX_ALL_ | 转轮机枪类别可造成石头伤害的最大距离 | 服务器端本插件逻辑 | `l4d_rock_lagcomp.sp:177` |
| `sm_rock_range_mounted_machinegun` | `10000` | 整数，源码写作浮点 | 0.0 ~ RANGE_MAX_ALL_ | 固定机枪类别可造成石头伤害的最大距离 | 服务器端本插件逻辑 | `l4d_rock_lagcomp.sp:178` |

## L4D2 Tank Control —— `optional/l4d_tank_control_eq.sp`

- 源文件：`optional/l4d_tank_control_eq.sp`
- myinfo：version=0.0.29，author=arti, (Contributions by: Sheo, Sir, Altair-Sossai)
- myinfo description（源码原文）：Distributes the role of the tank evenly throughout the team, allows for overrides. (Includes forwards)

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_tankshuffle` | 无参数 | 需要 ADMFLAG_SLAY（f，处死） | 重新随机挑选一名玩家成为坦克 | `l4d_tank_control_eq.sp:80` |
| `sm_givetank` | <玩家> | 需要 ADMFLAG_SLAY（f，处死） | 把坦克交给指定玩家 | `l4d_tank_control_eq.sp:81` |
| `sm_tank` | 无参数 | 无（任意玩家可用） | 显示谁将成为坦克 | `l4d_tank_control_eq.sp:84` |
| `sm_boss` | 无参数 | 无（任意玩家可用） | sm_tank 的别名 | `l4d_tank_control_eq.sp:85` |
| `sm_witch` | 无参数 | 无（任意玩家可用） | sm_tank 的别名 | `l4d_tank_control_eq.sp:86` |

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `tankcontrol_print_all` | `0` | 整数 | 无上下界 | 谁能看到谁将成为坦克：0 特感，1 所有人 | 服务器端本插件逻辑 | `l4d_tank_control_eq.sp:89` |
| `tankcontrol_force_window` | `0.0` | 整数，源码写作浮点 | 无上下界 | 给原本要成为坦克或曾是坦克后掉线的玩家在该秒数后重新给予坦克 | 服务器端本插件逻辑 | `l4d_tank_control_eq.sp:90` |
| `tankcontrol_debug` | `0` | 整数 | 无上下界 | 是否输出调试信息到控制台 | 服务器端本插件逻辑 | `l4d_tank_control_eq.sp:91` |

## Tank Damage Announce L4D2 —— `optional/l4d_tank_damage_announce.sp`

- 源文件：`optional/l4d_tank_damage_announce.sp`
- myinfo：version=0.6.7，author=Griffin and Blade
- myinfo description（源码原文）：Announce damage dealt to tanks by survivors
- HookConVarChange：`g_cvarEnabled`→`Cvar_Enabled`、`g_cvarSurvivorLimit`→`Cvar_SurvivorLimit`、`g_cvarTankHealth`→`Cvar_TankHealth`、`g_cvarDifficulty`→`Cvar_TankHealth`、`FindConVar("mp_gamemode")`→`Cvar_TankHealth`

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `l4d_tankdamage_enabled` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否在启用时播报对坦克造成的伤害 | 服务器端本插件逻辑；仅服务器 | `l4d_tank_damage_announce.sp:71` |

## L4D Tank Pain Fade —— `optional/l4d_tank_painfade.sp`

- 源文件：`optional/l4d_tank_painfade.sp`
- myinfo：version=1.1，author=Visor
- myinfo description（源码原文）：Tank's screen fades into red when taking damage

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `l4d_tank_painfade` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用插件 | 服务器端本插件逻辑 | `l4d_tank_painfade.sp:58` |
| `l4d_tank_painfade_duration` | `150` | 整数 | 无上下界 | 淡化持续帧数 | 服务器端本插件逻辑 | `l4d_tank_painfade.sp:59` |
| `l4d_tank_painfade_flags` | `8` | 整数，源码写作浮点 | 1.0 ~ 15.0 | 哪些武器会造成淡化效果：1 乌兹，2 霰弹枪，4 狙击，8 近战 | 服务器端本插件逻辑 | `l4d_tank_painfade.sp:60` |

## L4D2 No Tank Rush —— `optional/l4d_tank_rush.sp`

- 源文件：`optional/l4d_tank_rush.sp`
- myinfo：version=1.1.4，author=Jahze, vintik, devilesk, Sir
- myinfo description（源码原文）：Stops distance points accumulating whilst the tank is alive, with the option of unfreezing distance on reaching the Saferoom

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `l4d_no_tank_rush` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否阻止生还者队伍在坦克存活期间累积分数 | 服务器端本插件逻辑 | `l4d_tank_rush.sp:39` |
| `l4d_no_tank_rush_unfreeze_saferoom` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 坦克仍存活时有生还者到达终点安全室是否解冻距离 | 服务器端本插件逻辑 | `l4d_tank_rush.sp:40` |
| `l4d_no_tank_rush_unfreeze_ai` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 坦克变为 AI 时是否解冻距离 | 服务器端本插件逻辑 | `l4d_tank_rush.sp:41` |
| `l4d_no_tank_rush_spawn_sound` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 生成坦克时是否播放声音 | 服务器端本插件逻辑 | `l4d_tank_rush.sp:42` |

## Tank Punch Ceiling Stuck Fix —— `optional/l4d_tankpunchstuckfix.sp`

- 源文件：`optional/l4d_tankpunchstuckfix.sp`
- myinfo：version=2.0，author=Tabun, Visor, A1m`, Forgetest
- myinfo description（源码原文）：Fixes the problem where tank-punches get a survivor stuck in the roof.

### 指令

（本插件未注册指令）

### ConVar

（本插件未注册 ConVar）

## Mathack Block —— `optional/l4d_texture_manager_block.sp`

- 源文件：`optional/l4d_texture_manager_block.sp`
- myinfo：version=1.1，author=Sir, Visor
- myinfo description（源码原文）：Kicks out clients who are potentially attempting to enable mathack

### 指令

（本插件未注册指令）

### ConVar

（本插件未注册 ConVar）

## Thirdpersonshoulder Block —— `optional/l4d_thirdpersonshoulderblock.sp`

- 源文件：`optional/l4d_thirdpersonshoulderblock.sp`
- myinfo：author=Don
- myinfo description（源码原文）：Kicks clients who enable the thirdpersonshoulder mode on L4D1/2 to prevent them from looking around corners, through walls etc.

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `l4d_tpsblock_version` | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | 服务器端本插件逻辑；不写入 cfg 存档 | `l4d_thirdpersonshoulderblock.sp:42` |
| `l4d_tpsblock_enabled` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用第三人称肩部视角屏蔽 | 服务器端本插件逻辑 | `l4d_thirdpersonshoulderblock.sp:44` |
| `l4d_tpsblock_action` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 玩家开启第三人称时的处理：1 移到旁观，0 踢出服务器 | 服务器端本插件逻辑 | `l4d_thirdpersonshoulderblock.sp:45` |

## L4D Weapon Limits —— `optional/l4d_weapon_limits.sp`

- 源文件：`optional/l4d_weapon_limits.sp`
- myinfo：version=2.2.4，author=CanadaRox, Stabby, Forgetest, A1m`, robex
- myinfo description（源码原文）：Restrict weapons individually or together

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `l4d_wlimits_add` | <武器> <数量> <掩码> | 服务器控制台命令，玩家无法使用 | 添加武器数量限制，服务器控制台命令 | `l4d_weapon_limits.sp:81` |
| `l4d_wlimits_lock` | 无参数 | 服务器控制台命令，玩家无法使用 | 锁定武器限制以加快查找速度，服务器控制台命令 | `l4d_weapon_limits.sp:82` |
| `l4d_wlimits_clear` | 无参数 | 服务器控制台命令，玩家无法使用 | 清空所有武器限制，需先锁定，服务器控制台命令 | `l4d_weapon_limits.sp:83` |

### ConVar

（本插件未注册 ConVar）

## Witch Damage Announce —— `optional/l4d_witch_damage_announce.sp`

- 源文件：`optional/l4d_witch_damage_announce.sp`
- myinfo：version=1.3，author=Sir
- myinfo description（源码原文）：Print Witch Damage to chat

### 指令

（本插件未注册指令）

### ConVar

（本插件未注册 ConVar）

## L4D HOTs —— `optional/l4dhots.sp`

- 源文件：`optional/l4dhots.sp`
- myinfo：author=ProdigySim, CircleSquared, Forgetest
- myinfo description（源码原文）：Pills and Adrenaline heal over time

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `l4d_pills_hot` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 止痛药是否随时间持续回血 | 服务器端本插件逻辑；仅服务器 | `l4dhots.sp:60` |
| `l4d_pills_hot_interval` | `1.0` | 浮点 | ≥ 0.00001 | 止痛药回血的间隔，最小 0.00001 | 服务器端本插件逻辑；仅服务器 | `l4dhots.sp:61` |
| `l4d_pills_hot_increment` | `10` | 整数，源码写作浮点 | ≥ 1.0 | 止痛药每次回血量，最小 1 | 服务器端本插件逻辑；仅服务器 | `l4dhots.sp:62` |
| `l4d_pills_hot_total` | `buffer` | 字符串或表达式 | ≥ 0.0 | 止痛药回血总量，最小 0 | 服务器端本插件逻辑；仅服务器 | `l4dhots.sp:63` |
| `l4d_adrenaline_hot` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 肾上腺素是否随时间持续回血 | 服务器端本插件逻辑；仅服务器 | `l4dhots.sp:74` |
| `l4d_adrenaline_hot_interval` | `1.0` | 浮点 | ≥ 0.00001 | 肾上腺素回血间隔，最小 0.00001 | 服务器端本插件逻辑；仅服务器 | `l4dhots.sp:75` |
| `l4d_adrenaline_hot_increment` | `15` | 整数，源码写作浮点 | ≥ 1.0 | 肾上腺素每次回血量，最小 1 | 服务器端本插件逻辑；仅服务器 | `l4dhots.sp:76` |
| `l4d_adrenaline_hot_total` | `buffer` | 字符串或表达式 | ≥ 0.0 | 肾上腺素回血总量，最小 0 | 服务器端本插件逻辑；仅服务器 | `l4dhots.sp:77` |

## LerpMonitor++ —— `optional/lerpmonitor.sp`

- 源文件：`optional/lerpmonitor.sp`
- myinfo：version=2.4.5，author=ProdigySim, Die Teetasse, vintik, A1m`, Modified by Gemini
- myinfo description（源码原文）：Keep track of players' lerp settings with 5s warning

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_lerps` | 无参数 | 无（任意玩家可用） | 列出所有在场玩家的 lerp 设置 | `lerpmonitor.sp:77` |

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `sm_allowed_lerp_changes` | `1` | 整数，源码写作浮点 | 0.0 ~ 20.0 | 半场内允许的 lerp 修改次数，范围 0~20 | 服务器端本插件逻辑 | `lerpmonitor.sp:69` |
| `sm_lerp_change_spec` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 超过 lerp 修改次数后是否移到旁观 | 服务器端本插件逻辑 | `lerpmonitor.sp:70` |
| `sm_bad_lerp_action` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 玩家 lerp 超出允许范围时的处理：1 移到旁观，0 踢出服务器 | 服务器端本插件逻辑 | `lerpmonitor.sp:71` |
| `sm_readyup_lerp_changes` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 准备阶段是否允许修改 lerp | 服务器端本插件逻辑 | `lerpmonitor.sp:72` |
| `sm_show_lerp_team_changes` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 玩家换队时是否提示其 lerp | 服务器端本插件逻辑 | `lerpmonitor.sp:73` |
| `sm_min_lerp` | `0.000` | 浮点 | 0.000 ~ 0.500 | 允许的最小 lerp 值，范围 0.000~0.500 | 服务器端本插件逻辑 | `lerpmonitor.sp:74` |
| `sm_max_lerp` | `0.067` | 浮点 | 0.000 ~ 0.500 | 允许的最大 lerp 值，范围 0.000~0.500 | 服务器端本插件逻辑 | `lerpmonitor.sp:75` |

## Network Quality Hint —— `optional/network_quality_hint.sp`

- 源文件：`optional/network_quality_hint.sp`
- myinfo：author=Anne
- myinfo description（源码原文）：Samples player network quality, reports summaries and incidents, and shows reconnect hints.
- HookConVarChange：`g_hEnable`→`OnCvarChanged`、`g_hCheckInterval`→`OnCvarChanged`、`g_hPingLimit`→`OnCvarChanged`、`g_hLossLimit`→`OnCvarChanged`、`g_hChokeLimit`→`OnCvarChanged`、`g_hBadSamples`→`OnCvarChanged`、`g_hWarnCooldown`→`OnCvarChanged`、`g_hIpPageUrl`→`OnCvarChanged`、`g_hIntroDelay`→`OnCvarChanged`、`g_hReportEnable`→`OnCvarChanged`、`g_hReportUrl`→`OnCvarChanged`、`g_hReportInterval`→`OnCvarChanged`、`g_hReportChokeLimit`→`OnCvarChanged`、`g_hReportBadSamples`→`OnCvarChanged`、`g_hReportRecoverySamples`→`OnCvarChanged`

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_net` | 无参数 | 无（任意玩家可用） | 显示自己的网络状态 | `network_quality_hint.sp:117` |
| `sm_ping` | 无参数 | 无（任意玩家可用） | sm_net 的别名 | `network_quality_hint.sp:118` |
| `sm_loss` | 无参数 | 无（任意玩家可用） | sm_net 的别名 | `network_quality_hint.sp:119` |

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `nqh_version` | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | 服务器端本插件逻辑；不写入 cfg 存档 | `network_quality_hint.sp:99` |
| `nqh_enable` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用网络质量检测 | 服务器端本插件逻辑 | `network_quality_hint.sp:101` |
| `nqh_check_interval` | `5.0` | 整数，源码写作浮点 | 5.0 ~ 300.0 | 本地网络采样间隔秒数，范围 5~300 | 服务器端本插件逻辑 | `network_quality_hint.sp:102` |
| `nqh_ping_limit` | `120` | 整数，源码写作浮点 | ≥ -1.0 | ping 高于该毫秒值时警告，-1 关闭 ping 检测 | 服务器端本插件逻辑 | `network_quality_hint.sp:103` |
| `nqh_loss_limit` | `2.0` | 整数，源码写作浮点 | ≥ -1.0 | 丢包率高于该百分比时警告，-1 关闭丢包检测 | 服务器端本插件逻辑 | `network_quality_hint.sp:104` |
| `nqh_choke_limit` | `5.0` | 整数，源码写作浮点 | ≥ -1.0 | choke 高于该百分比时警告，-1 关闭 choke 检测 | 服务器端本插件逻辑 | `network_quality_hint.sp:105` |
| `nqh_bad_samples` | `3` | 整数，源码写作浮点 | 1.0 ~ 20.0 | 警告玩家前需要的连续不良采样次数，范围 1~20 | 服务器端本插件逻辑 | `network_quality_hint.sp:106` |
| `nqh_warn_cooldown` | `180.0` | 整数，源码写作浮点 | 30.0 ~ 1800.0 | 同一玩家再次被警告前的间隔秒数，范围 30~1800 | 服务器端本插件逻辑 | `network_quality_hint.sp:107` |
| `nqh_ip_page_url` | `https://anne.trygek.com/ip.php` | 字符串或表达式 | 无上下界 | 列出服务器 IP 与可复制连接命令的网页地址 | 服务器端本插件逻辑 | `network_quality_hint.sp:108` |
| `nqh_intro_delay` | `25.0` | 整数，源码写作浮点 | 0.0 ~ 300.0 | 玩家加入后多少秒打印一次状态提示，0 关闭，范围 0~300 | 服务器端本插件逻辑 | `network_quality_hint.sp:109` |
| `nqh_report_enable` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 配置了 URL 时是否启用可选的 HTTPS 玩家连接质量上报 | 服务器端本插件逻辑 | `network_quality_hint.sp:110` |
| `nqh_report_url` | `http://anne.trygek.com/api/player/connection_quality.php` | 字符串或表达式 | 无上下界 | 玩家连接质量上报的 HTTP 端点 | 服务器端本插件逻辑 | `network_quality_hint.sp:111` |
| `nqh_report_interval` | `600` | 整数，源码写作浮点 | 60.0 ~ 3600.0 | 常规汇总上报的间隔秒数，范围 60~3600 | 服务器端本插件逻辑 | `network_quality_hint.sp:112` |
| `nqh_report_choke_limit` | `20.0` | 整数，源码写作浮点 | ≥ -1.0 | choke 高于该百分比时才上报事件，-1 关闭 choke 事件 | 服务器端本插件逻辑 | `network_quality_hint.sp:113` |
| `nqh_report_bad_samples` | `1` | 整数，源码写作浮点 | 1.0 ~ 20.0 | 上报事件所需的连续不良本地采样次数，范围 1~20 | 服务器端本插件逻辑 | `network_quality_hint.sp:114` |
| `nqh_report_recovery_samples` | `3` | 整数，源码写作浮点 | 1.0 ~ 20.0 | 上报恢复所需的连续良好本地采样次数，范围 1~20 | 服务器端本插件逻辑 | `network_quality_hint.sp:115` |

## No Mercy 3 Ladder Fix —— `optional/nm3_ladder_damage.sp`

- 源文件：`optional/nm3_ladder_damage.sp`
- myinfo：version=1.3，author=Jacob
- myinfo description（源码原文）：Blocks players getting incapped from full hp on the ladder.

### 指令

（本插件未注册指令）

### ConVar

（本插件未注册 ConVar）

## Death Cam Skip Fix —— `optional/nodeathcamskip.sp`

- 源文件：`optional/nodeathcamskip.sp`
- myinfo：author=Jacob, Sir, Forgetest
- myinfo description（源码原文）：Blocks players skipping their death time by going spec

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `deathcam_skip_announce` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 有人利用漏洞时是否打印提示 | 服务器端本插件逻辑；仅服务器 | `nodeathcamskip.sp:31` |

## No Safe Room Medkits —— `optional/nosaferoomkits.sp`

- 源文件：`optional/nosaferoomkits.sp`
- myinfo：author=Blade
- myinfo description（源码原文）：Removes Safe Room Medkits

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `nokits_version` | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | 服务器端本插件逻辑；复制到客户端；仅服务器 | `nosaferoomkits.sp:28` |

## [L4D/L4D2]noteam_nudging —— `optional/noteam_nudging.sp`

- 源文件：`optional/noteam_nudging.sp`
- myinfo：author=Lux
- myinfo description（源码原文）：Prevents small push effect between survior players, bots still get pushed.

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `noteam_nudging_version` | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | 服务器端本插件逻辑；不写入 cfg 存档 | `noteam_nudging.sp:23` |

## Add Text To Readyup Panel —— `optional/panel_text.sp`

- 源文件：`optional/panel_text.sp`
- myinfo：version=1.2，author=epilimic
- myinfo description（源码原文）：Displays custom text in the readyup panel. Spanks for the help CanadaRox!

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_addreadystring` | <文本> | 服务器控制台命令，玩家无法使用 | 设置要加入准备面板的文本，服务器控制台命令 | `panel_text.sp:31` |
| `sm_resetstringcount` | 无参数 | 服务器控制台命令，玩家无法使用 | 重置文本计数，服务器控制台命令 | `panel_text.sp:32` |
| `sm_lockstrings` | 无参数 | 服务器控制台命令，玩家无法使用 | 锁定准备面板文本，服务器控制台命令 | `panel_text.sp:33` |

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `sm_readypaneltextdelay` | `4.0` | 整数，源码写作浮点 | 0.0 ~ 10.0 | 把文本加入准备面板前的延迟秒数，范围 0~10 | 服务器端本插件逻辑 | `panel_text.sp:29` |

## Pause plugin —— `optional/pause.sp`

- 源文件：`optional/pause.sp`
- myinfo：version=6.9.0，author=CanadaRox, Sir, Forgetest, A1m`
- myinfo description（源码原文）：Adds pause functionality without breaking pauses, also prevents SI from spawning because of the Pause.

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_spectate` | 无参数 | 无（任意玩家可用） | 把自己移到旁观者队伍 | `pause.sp:124` |
| `sm_spec` | 无参数 | 无（任意玩家可用） | sm_spectate 的别名 | `pause.sp:125` |
| `sm_s` | 无参数 | 无（任意玩家可用） | sm_spectate 的别名 | `pause.sp:126` |
| `sm_pause` | 无参数 | 无（任意玩家可用） | 暂停游戏 | `pause.sp:128` |
| `sm_unpause` | 无参数 | 无（任意玩家可用） | 把自己队伍标记为准备以便取消暂停 | `pause.sp:129` |
| `sm_ready` | 无参数 | 无（任意玩家可用） | sm_unpause 的别名 | `pause.sp:130` |
| `sm_r` | 无参数 | 无（任意玩家可用） | sm_unpause 的别名 | `pause.sp:131` |
| `sm_unready` | 无参数 | 无（任意玩家可用） | 把自己队伍标记为未准备 | `pause.sp:132` |
| `sm_nr` | 无参数 | 无（任意玩家可用） | sm_unready 的别名 | `pause.sp:133` |
| `sm_toggleready` | 无参数 | 无（任意玩家可用） | 切换自己队伍的准备状态 | `pause.sp:134` |
| `sm_forcepause` | 无参数 | 需要 ADMFLAG_BAN（d，封禁） | 暂停游戏且只允许管理员取消暂停 | `pause.sp:136` |
| `sm_forceunpause` | 无参数 | 需要 ADMFLAG_BAN（d，封禁） | 无视队伍准备状态取消暂停，用于取消管理员发起的暂停 | `pause.sp:137` |
| `sm_forcestart` | 无参数 | 需要 ADMFLAG_BAN（d，封禁） | sm_forceunpause 的别名 | `pause.sp:138` |
| `sm_fs` | 无参数 | 需要 ADMFLAG_BAN（d，封禁） | sm_forceunpause 的短别名 | `pause.sp:139` |
| `sm_show` | 无参数 | 无（任意玩家可用） | 隐藏暂停面板以便查看其他菜单 | `pause.sp:141` |
| `sm_hide` | 无参数 | 无（任意玩家可用） | 显示被隐藏的暂停面板 | `pause.sp:142` |

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `sm_onlyforce` | `0` | 整数 | 无上下界 | 是否只允许强制暂停与取消暂停功能 | 服务器端本插件逻辑 | `pause.sp:111` |
| `sm_pausedelay` | `0` | 整数，源码写作浮点 | ≥ 0.0 | 暂停生效前的延迟秒数，可用于防止战术暂停 | 服务器端本插件逻辑 | `pause.sp:112` |
| `sm_unpausedelay` | `3` | 整数，源码写作浮点 | ≥ 0.0 | 取消暂停生效前的延迟秒数 | 服务器端本插件逻辑 | `pause.sp:113` |
| `sm_initiatorready` | `0` | 整数，源码写作浮点 | ≥ 0.0 | 是否要求暂停发起者准备后才取消暂停 | 服务器端本插件逻辑 | `pause.sp:114` |
| `sm_pauselimit` | `0` | 整数，源码写作浮点 | ≥ 0.0 | 单局内玩家可暂停的次数上限，0 表示不限制 | 服务器端本插件逻辑 | `pause.sp:115` |

## Easier Pill Passer —— `optional/pill_passer.sp`

- 源文件：`optional/pill_passer.sp`
- myinfo：version=1.6.3，author=CanadaRox, A1m`, Forgetest
- myinfo description（源码原文）：Lets players pass pills and adrenaline with +reload when they are holding one of those items

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `pill_passer_range` | `274.0` | 整数，源码写作浮点 | ≥ 0.0 | 玩家之间传递止痛药的最大距离 | 服务器端本插件逻辑；需 sv_cheats 方可修改；经 CreateConVarHook 包装并注册变更回调 | `pill_passer.sp:41` |
| `pill_passer_los_clear` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 传递止痛药时是否要求视线通畅 | 服务器端本插件逻辑；需 sv_cheats 方可修改；经 CreateConVarHook 包装并注册变更回调 | `pill_passer.sp:48` |
| `pill_passer_lag_compensate` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 传递止痛药时是否启用延迟补偿 | 服务器端本插件逻辑；需 sv_cheats 方可修改；经 CreateConVarHook 包装并注册变更回调 | `pill_passer.sp:55` |

## Player Management Plugin —— `optional/playermanagement.sp`

- 源文件：`optional/playermanagement.sp`
- myinfo：version=7.1.3，author=CanadaRox
- myinfo description（源码原文）：Player management!  Swap players/teams and spectate!

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_swap` | <玩家...> | 需要 ADMFLAG_KICK（c，踢出） | 把列出的玩家全部换到对面队伍 | `playermanagement.sp:67` |
| `sm_swapto` | <队伍号> <玩家...> | 需要 ADMFLAG_KICK（c，踢出） | 把列出的玩家换到指定队伍，队伍号见命令提示 | `playermanagement.sp:68` |
| `sm_swapteams` | 无参数 | 需要 ADMFLAG_KICK（c，踢出） | 交换两队玩家 | `playermanagement.sp:69` |
| `sm_fixbots` | 无参数 | 需要 ADMFLAG_BAN（d，封禁） | 补充生还者 Bot 到 survivor_limit | `playermanagement.sp:70` |
| `sm_spectate` | 无参数 | 无（任意玩家可用） | 把自己移到旁观者队伍 | `playermanagement.sp:71` |
| `sm_spec` | 无参数 | 无（任意玩家可用） | sm_spectate 的别名 | `playermanagement.sp:72` |
| `sm_s` | 无参数 | 无（任意玩家可用） | sm_spectate 的别名 | `playermanagement.sp:73` |

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `sm_allow_spectate_command` | `1` | 整数 | 无上下界 | 是否允许玩家使用 !spectate/!spec/!s 命令 | 服务器端本插件逻辑 | `playermanagement.sp:75` |
| `sm_blockspecintank` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否阻止生还者在扮演坦克时切换到旁观 | 服务器端本插件逻辑 | `playermanagement.sp:76` |
| `l4d_pm_supress_spectate` | `0` | 整数 | 无上下界 | 玩家转为旁观时是否不打印提示 | 服务器端本插件逻辑 | `playermanagement.sp:84` |

## Predictable Plugin Unloader —— `optional/predictable_unloader.sp`

- 源文件：`optional/predictable_unloader.sp`
- myinfo：version=1.2.2，author=Sir (heavily influenced by keyCat)
- myinfo description（源码原文）：Allows for unloading plugins from last to first.

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `pred_unload_plugins` | 无参数 | 服务器控制台命令，玩家无法使用 | 卸载插件，服务器控制台命令 | `predictable_unloader.sp:70` |

### ConVar

（本插件未注册 ConVar）

## RateMonitor —— `optional/ratemonitor.sp`

- 源文件：`optional/ratemonitor.sp`
- myinfo：version=2.6.1，author=Visor, Sir, A1m`
- myinfo description（源码原文）：Keep track of players' netsettings

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_rates` | 无参数 | 无（任意玩家可用） | 列出所有在场玩家的网络设置 | `ratemonitor.sp:87` |

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `rm_allowed_rate_changes` | `-1` | 整数 | 无上下界 | 单局内允许修改 rate 的次数，-1 表示不限 | 服务器端本插件逻辑 | `ratemonitor.sp:64` |
| `rm_public_notice` | `0` | 整数 | 无上下界 | 是否向公众打印 rate 变更提示 | 服务器端本插件逻辑 | `ratemonitor.sp:65` |
| `rm_min_rate` | `20000` | 整数 | 无上下界 | 允许的 rate 最小值，-1 表示不限 | 服务器端本插件逻辑 | `ratemonitor.sp:66` |
| `rm_min_upd` | `20` | 整数 | 无上下界 | 允许的 cl_updaterate 最小值，-1 表示不限 | 服务器端本插件逻辑 | `ratemonitor.sp:67` |
| `rm_min_cmd` | `20` | 整数 | 无上下界 | 允许的 cl_cmdrate 最小值，-1 表示不限 | 服务器端本插件逻辑 | `ratemonitor.sp:68` |
| `rm_no_fake_ping` | `0` | 整数 | 无上下界 | 是否允许在网络设置中使用 + - . 来隐藏真实 ping | 服务器端本插件逻辑 | `ratemonitor.sp:69` |
| `rm_countermeasure` | `2` | 整数，源码写作浮点 | 1.0 ~ 3.0 | 违规处理方式：1 聊天提醒，2 移到旁观，3 踢出 | 服务器端本插件逻辑 | `ratemonitor.sp:70` |

## L4D2 Ready-Up with convenience fixes —— `optional/readyup.sp`

- 源文件：`optional/readyup.sp`
- myinfo：author=CanadaRox, Target
- myinfo description（源码原文）：New and improved ready-up plugin with optimal for convenience.

### 指令

（本插件未注册指令）

### ConVar

（本插件未注册 ConVar）

## Tank Rock Stumble Block —— `optional/rock_stumble_block.sp`

- 源文件：`optional/rock_stumble_block.sp`
- myinfo：version=2.0，author=Jacob, Forgetest
- myinfo description（源码原文）：Fixes rocks disappearing if tank gets stumbled while throwing.

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `rock_stumble_throwing` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 被硬直时是否仍继续投掷石头，关闭后坦克被硬直会受到较大的移动惩罚 | 服务器端本插件逻辑；仅服务器；经 CreateConVarHook 包装并注册变更回调 | `rock_stumble_block.sp:39` |

## Server Clean Up —— `optional/servercleanup.sp`

- 源文件：`optional/servercleanup.sp`
- myinfo：author=Jamster
- myinfo description（源码原文）：Cleans up logs and demo files automatically

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_srvcln_now` | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 立即执行一次清理，源码未给描述 | `servercleanup.sp:174` |

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `sm_srvcln_version` | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 源码描述仅为 desc，未说明具体行为 | 服务器端本插件逻辑；复制到客户端；仅服务器；不写入 cfg 存档 | `servercleanup.sp:117` |
| `sm_srvcln_enable` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 源码描述仅为 desc，未说明具体行为 | 服务器端本插件逻辑 | `servercleanup.sp:120` |
| `sm_srvcln_logging_mode` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 源码描述仅为 desc，未说明具体行为 | 服务器端本插件逻辑 | `servercleanup.sp:123` |
| `sm_srvcln_logs` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 源码描述仅为 desc，未说明具体行为 | 服务器端本插件逻辑 | `servercleanup.sp:126` |
| `sm_srvcln_smlogs` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 源码描述仅为 desc，未说明具体行为 | 服务器端本插件逻辑 | `servercleanup.sp:129` |
| `sm_srvcln_demos` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 源码描述仅为 desc，未说明具体行为 | 服务器端本插件逻辑 | `servercleanup.sp:132` |
| `sm_srvcln_replays` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 源码描述仅为 desc，未说明具体行为 | 服务器端本插件逻辑 | `servercleanup.sp:135` |
| `sm_srvcln_replays_archives` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 源码描述仅为 desc，未说明具体行为 | 服务器端本插件逻辑 | `servercleanup.sp:138` |
| `sm_srvcln_roundbackups` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 源码描述仅为 desc，未说明具体行为 | 服务器端本插件逻辑 | `servercleanup.sp:141` |
| `sm_srvcln_demos_path` | `` | 字符串或表达式 | 无上下界 | 源码描述仅为 desc，未说明具体行为 | 服务器端本插件逻辑 | `servercleanup.sp:144` |
| `sm_srvcln_sprays` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 源码描述仅为 desc，未说明具体行为 | 服务器端本插件逻辑 | `servercleanup.sp:147` |
| `sm_srvcln_demos_archives` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 源码描述仅为 desc，未说明具体行为 | 服务器端本插件逻辑 | `servercleanup.sp:150` |
| `sm_srvcln_smlogs_type` | `0` | 整数，源码写作浮点 | 0.0 ~ 2.0 | 源码描述仅为 desc，未说明具体行为 | 服务器端本插件逻辑 | `servercleanup.sp:153` |
| `sm_srvcln_logs_time` | `168` | 整数，源码写作浮点 | ≥ -1.0 | 源码描述仅为 desc，未说明具体行为 | 服务器端本插件逻辑 | `servercleanup.sp:156` |
| `sm_srvcln_sprays_time` | `168` | 整数，源码写作浮点 | ≥ -1.0 | 源码描述仅为 desc，未说明具体行为 | 服务器端本插件逻辑 | `servercleanup.sp:159` |
| `sm_srvcln_smlogs_time` | `168` | 整数，源码写作浮点 | ≥ -1.0 | 源码描述仅为 desc，未说明具体行为 | 服务器端本插件逻辑 | `servercleanup.sp:162` |
| `sm_srvcln_demos_time` | `168` | 整数，源码写作浮点 | ≥ -1.0 | 源码描述仅为 desc，未说明具体行为 | 服务器端本插件逻辑 | `servercleanup.sp:165` |
| `sm_srvcln_replays_time` | `168` | 整数，源码写作浮点 | ≥ -1.0 | 源码描述仅为 desc，未说明具体行为 | 服务器端本插件逻辑 | `servercleanup.sp:168` |
| `sm_srvcln_roundbackups_time` | `168` | 整数，源码写作浮点 | ≥ -1.0 | 源码描述仅为 desc，未说明具体行为 | 服务器端本插件逻辑 | `servercleanup.sp:171` |

## Special Infected Class Announce —— `optional/si_class_announce.sp`

- 源文件：`optional/si_class_announce.sp`
- myinfo：author=Tabun, Forgetest
- myinfo description（源码原文）：Report what SI classes are up when the round starts.

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `si_announce_ready_footer` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否把特感职业字符串作为页脚加入准备面板 | 服务器端本插件逻辑 | `si_class_announce.sp:67` |
| `si_announce_print` | `1` | 整数，源码写作浮点 | 0.0 ~ 3.0 | 插件输出公告的位置：0 禁用，1 聊天，2 提示框，3 聊天与提示框 | 服务器端本插件逻辑 | `si_class_announce.sp:72` |

## SI Fire Immunity —— `optional/si_fire_immunity.sp`

- 源文件：`optional/si_fire_immunity.sp`
- myinfo：version=3.3，author=Jacob, darkid, A1m`
- myinfo description（源码原文）：Special Infected fire damage management.

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `infected_fire_immunity` | `3` | 整数，源码写作浮点 | 0.0 ~ 3.0 | 特感的火焰免疫类型：0 无，3 随时间自动熄灭，2 免疫燃烧，1 完全免疫 | 服务器端本插件逻辑 | `si_fire_immunity.sp:49` |
| `tank_fire_immunity` | `2` | 整数，源码写作浮点 | 0.0 ~ 3.0 | 坦克的火焰免疫类型：0 无，3 随时间自动熄灭，2 免疫燃烧，1 完全免疫 | 服务器端本插件逻辑 | `si_fire_immunity.sp:56` |
| `infected_extinguish_time` | `1.0` | 整数，源码写作浮点 | 0.0 ~ 999.0 | 特感玩家在多少秒后熄灭，仅当 infected_fire_immunity 为 3 时生效 | 服务器端本插件逻辑 | `si_fire_immunity.sp:63` |
| `tank_extinguish_time` | `1.0` | 整数，源码写作浮点 | 0.0 ~ 999.0 | 坦克玩家在多少秒后熄灭，仅当 tank_fire_immunity 为 3 时生效 | 服务器端本插件逻辑 | `si_fire_immunity.sp:70` |

## Simple Witch Kill Bonus —— `optional/simple_witch_bonus.sp`

- 源文件：`optional/simple_witch_bonus.sp`
- myinfo：version=0.9.3，author=Tabun
- myinfo description（源码原文）：Gives bonus for witches getting killed without doing damage to survivors (uses pbonus).

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `sm_simple_witch_bonus` | `25` | 整数，源码写作浮点 | ≥ 0.0 | 干净击杀 Witch 奖励的分数 | 服务器端本插件逻辑 | `simple_witch_bonus.sp:54` |
| `sm_witch_bonus_print` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 奖励 Witch 击杀分数时是否打印提示 | 服务器端本插件逻辑 | `simple_witch_bonus.sp:55` |
| `sm_witch_bonus_always` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 非生还者击杀 Witch 时是否也给予分数 | 服务器端本插件逻辑 | `simple_witch_bonus.sp:56` |

## Slots?! Voter —— `optional/slots_vote.sp`

- 源文件：`optional/slots_vote.sp`
- myinfo：version=1.0，author=Sir
- myinfo description（源码原文）：Slots Voter
- HookConVarChange：`hMaxSlots`→`CVarChanged`、`hNonAdminMinSlots`→`CVarChanged`

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_slots` | <槽位数> | 无（任意玩家可用） | 发起修改服务器槽位数的投票 | `slots_vote.sp:29` |

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `slots_max_slots` | `16` | 整数，源码写作浮点 | 1.0 ~ float(SLOT_HARD_MAX) | 玩家可投票的槽位上限，上限为 SLOT_HARD_MAX | 服务器端本插件逻辑 | `slots_vote.sp:30` |
| `slots_nonAdmin_min_slots` | `4` | 整数，源码写作浮点 | 1.0 ~ float(SLOT_HARD_MAX) | 非管理员玩家可投票的最小槽位，上限为 SLOT_HARD_MAX | 服务器端本插件逻辑 | `slots_vote.sp:31` |

## [L4D & 2] Smart AI Rock —— `optional/smart_ai_rock.sp`

- 源文件：`optional/smart_ai_rock.sp`
- myinfo：author=Forgetest, CanadaRox (Original author)
- myinfo description（源码原文）：Prevent underhand rocks and fix sticking aim after throws for AI Tanks.

### 指令

（本插件未注册指令）

### ConVar

（本插件未注册 ConVar）

## Hyper-V HUD Manager —— `optional/spechud.sp`

- 源文件：`optional/spechud.sp`
- myinfo：author=Visor, Forgetest
- myinfo description（源码原文）：Provides different HUDs for spectators

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_spechud` | 无参数 | 无（任意玩家可用） | 切换自己的 spechud | `spechud.sp:113` |
| `sm_tankhud` | 无参数 | 无（任意玩家可用） | 切换自己的坦克 HUD | `spechud.sp:114` |
| `tank_map_flow_and_second_event` | <地图名> | 服务器控制台命令，玩家无法使用 | 标记地图坦克出现于第二事件，服务器控制台命令 | `spechud.sp:293` |
| `tank_map_only_first_event` | <地图名> | 服务器控制台命令，玩家无法使用 | 标记地图坦克只出现于第一事件，服务器控制台命令 | `spechud.sp:294` |
| `finale_tank_default` | <地图名> | 服务器控制台命令，玩家无法使用 | 设置终局坦克默认行为，服务器控制台命令 | `spechud.sp:295` |

### ConVar

（本插件未注册 ConVar）

## Lightweight Spectating (merged+128+force-spec) —— `optional/specrates.sp`

- 源文件：`optional/specrates.sp`
- myinfo：version=1.6-merged-128，author=Visor, lechuga, 东(merge) + Simth req
- myinfo description（源码原文）：Spectate/Play rate policies with admin 128-tick in play, plus force spec limiter

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_specrates` | 无参数 | 无（任意玩家可用） | 分数达到 30 万时手动设置旁观为 60 tick | `specrates.sp:121` |
| `sm_adminrates` | 无参数 | 需要 ADMFLAG_GENERIC（b，通用管理员） | 管理员手动提升，对局 128 tick、旁观 100 tick | `specrates.sp:122` |

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `specrates_fulltickspecnum` | `4` | 整数 | 无上下界 | 旁观人数超过该值后，除管理员与解说外其余人限 30 tick，无视积分 | 服务器端本插件逻辑 | `specrates.sp:108` |
| `specrates_force_spec` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 开启后旁观一律 30 tick，管理员与解说旁观为 60 tick，对局内不受影响 | 服务器端本插件逻辑 | `specrates.sp:112` |

## Super Stagger Solver —— `optional/staggersolver.sp`

- 源文件：`optional/staggersolver.sp`
- myinfo：version=2.4，author=CanadaRox, A1m (fix), Sir (rework), Forgetest
- myinfo description（源码原文）：Blocks all button presses and restarts animations during stumbles

### 指令

（本插件未注册指令）

### ConVar

（本插件未注册 ConVar）

## Starting Items —— `optional/starting_items.sp`

- 源文件：`optional/starting_items.sp`
- myinfo：version=2.2.1，author=CircleSquared, Jacob, A1m`
- myinfo description（源码原文）：Gives health items and throwables to survivors at the start of each round

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_give_starting_items` | 无参数 | 需要 ADMFLAG_KICK（c，踢出） | 立即给生还者发放起始物品 | `starting_items.sp:56` |

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `starting_item_flags` | `0` | 整数，源码写作浮点 | ≤ 127.0 | 离开安全区时给予的物品 flag，0 禁用，1 急救包，2 除颤器，4 止痛药，8 肾上腺素，16 管式炸弹，32 燃烧瓶，其余被截断 | 服务器端本插件逻辑 | `starting_items.sp:47` |

## Survivor MVP notification —— `optional/survivor_mvp.sp`

- 源文件：`optional/survivor_mvp.sp`
- myinfo：version=0.3.4，author=Tabun, Artifacial
- myinfo description（源码原文）：Shows MVP for survivor team at end of round
- HookConVarChange：`hCountTankDamage`→`ConVarChange_CountTankDamage`、`hCountWitchDamage`→`ConVarChange_CountWitchDamage`、`hTrackFF`→`ConVarChange_TrackFF`、`hBrevityFlags`→`ConVarChange_BrevityFlags`

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_mvp` | 无参数 | 无（任意玩家可用） | 显示当前生还者队伍的 MVP | `survivor_mvp.sp:288` |
| `sm_mvpme` | 无参数 | 无（任意玩家可用） | 显示自己的 MVP 相关统计 | `survivor_mvp.sp:289` |
| `sm_kills` | 无参数 | 无（任意玩家可用） | 显示 AnneHappy 精简生还者统计 | `survivor_mvp.sp:290` |
| `say` | 聊天文本 | 无（任意玩家可用） | 接管 say，识别 !mvp | `survivor_mvp.sp:292` |
| `say_team` | 聊天文本 | 无（任意玩家可用） | 接管 say_team，识别 !mvp | `survivor_mvp.sp:293` |

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `sm_survivor_mvp_enabled` | `1` | 整数 | 无上下界 | 是否在回合结束时显示 MVP | 服务器端本插件逻辑 | `survivor_mvp.sp:264` |
| `sm_survivor_mvp_counttank` | `0` | 整数 | 无上下界 | 为 1 时对坦克的伤害计入 MVP 评选 | 服务器端本插件逻辑 | `survivor_mvp.sp:265` |
| `sm_survivor_mvp_countwitch` | `0` | 整数 | 无上下界 | 为 1 时对 Witch 的伤害计入 MVP 评选 | 服务器端本插件逻辑 | `survivor_mvp.sp:266` |
| `sm_survivor_mvp_showff` | `1` | 整数 | 无上下界 | 是否统计友军伤害 | 服务器端本插件逻辑 | `survivor_mvp.sp:267` |
| `sm_survivor_mvp_brevity` | `0` | 整数 | 无上下界 | MVP 报告精简 flag：1 隐藏特感，2 隐藏普感，4 隐藏友伤，8 隐藏排名，32 隐藏百分比，64 隐藏绝对值 | 服务器端本插件逻辑 | `survivor_mvp.sp:268` |

## Teamflip —— `optional/teamflip.sp`

- 源文件：`optional/teamflip.sp`
- myinfo：version=1.0.1.0.1.0.1.0，author=purpletreefactory, epilimic
- myinfo description（源码原文）：coinflip, but for teams!

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_teamflip` | 无参数 | 无（任意玩家可用） | 发起换边即两队互换 | `teamflip.sp:51` |
| `sm_tf` | 无参数 | 无（任意玩家可用） | sm_teamflip 的别名 | `teamflip.sp:52` |

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `teamflip_delay` | `-1` | 整数，源码写作浮点 | ≥ -1.0 | 两次允许换队之间的延迟秒数，-1 表示无延迟 | 服务器端本插件逻辑 | `teamflip.sp:49` |

## Temp Health Fixer —— `optional/temphealthfix.sp`

- 源文件：`optional/temphealthfix.sp`
- myinfo：version=2.1，author=CanadaRox, Sir
- myinfo description（源码原文）：Ensures that survivors that have been incapacitated with a hittable or ledged get their temp health set correctly

### 指令

（本插件未注册指令）

### ConVar

（本插件未注册 ConVar）

## Visualise impacts —— `optional/visualise_impacts.sp`

- 源文件：`optional/visualise_impacts.sp`
- myinfo：version=1.7，author=A1m`
- myinfo description（源码原文）：Shows bullet impacts (based on the original by Jahze, fully rewritten and improved)

### 指令

（本插件未注册指令）

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `l4d_remove_decals_time` | `20.0` | 整数，源码写作浮点 | 0.0 ~ 320.0 | 弹痕在多少秒后被移除，0 表示关闭，范围 0~320 | 服务器端本插件逻辑 | `visualise_impacts.sp:55` |

## Weapon Loadout —— `optional/weapon_loadout_vote.sp`

- 源文件：`optional/weapon_loadout_vote.sp`
- myinfo：version=2.4，author=Sir, A1m`
- myinfo description（源码原文）：Allows the Players to choose which weapons to play the mode in.

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_mode` | 无参数 | 无（任意玩家可用） | 打开武器配置投票菜单 | `weapon_loadout_vote.sp:84` |
| `sm_forcemode` | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 强制打开武器配置投票菜单 | `weapon_loadout_vote.sp:86` |

### ConVar

（本插件未注册 ConVar）

## Tank and Witch ifier! —— `optional/witch_and_tankifier.sp`

- 源文件：`optional/witch_and_tankifier.sp`
- myinfo：version=2.4.1，author=CanadaRox, Sir, devilesk, Derpduck, Forgetest
- myinfo description（源码原文）：Sets a tank spawn and has the option to remove the witch spawn point on every map

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `static_tank_map` | <地图名> | 服务器控制台命令，玩家无法使用 | 把地图加入静态坦克地图列表，服务器控制台命令 | `witch_and_tankifier.sp:87` |
| `static_witch_map` | <地图名> | 服务器控制台命令，玩家无法使用 | 把地图加入静态女巫地图列表，服务器控制台命令 | `witch_and_tankifier.sp:88` |
| `reset_static_maps` | 无参数 | 服务器控制台命令，玩家无法使用 | 重置静态地图列表，服务器控制台命令 | `witch_and_tankifier.sp:89` |
| `sm_tank_witch_debug_info` | 无参数 | 需要 ADMFLAG_KICK（c，踢出） | 输出刷新状态调试信息 | `witch_and_tankifier.sp:91` |
| `sm_tank_witch_debug_test` | 无参数 | 无（任意玩家可用） | 测试命令，启动 AdjustBossFlow 计时器 | `witch_and_tankifier.sp:94` |
| `sm_tank_witch_debug_profiler` | <次数> | 无（任意玩家可用） | 按指定次数运行 AdjustBossFlow 性能分析 | `witch_and_tankifier.sp:95` |

### ConVar

| ConVar | 默认值 | 类型 | 取值范围 | 作用 | 影响范围 | 来源 |
|---|---|---|---|---|---|---|
| `sm_tank_witch_debug` | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否开启调试模式 | 服务器端本插件逻辑；不写入 cfg 存档 | `witch_and_tankifier.sp:70` |
| `sm_tank_can_spawn` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否允许坦克生成 | 服务器端本插件逻辑 | `witch_and_tankifier.sp:71` |
| `sm_witch_can_spawn` | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否允许 Witch 生成 | 服务器端本插件逻辑 | `witch_and_tankifier.sp:72` |
| `sm_witch_avoid_tank_spawn` | `20` | 整数，源码写作浮点 | 0.0 ~ 100.0 | Witch 应避开坦克刷新点的最小流程距离，按给定值的一半在坦克刷新点两侧计算 | 服务器端本插件逻辑 | `witch_and_tankifier.sp:73` |

## Lightweight Spectating Test —— `specrates_test.sp`

- 源文件：`specrates_test.sp`
- myinfo：version=1.0.0，author=lechuga
- myinfo description（源码原文）：A simple plugin to test the spectating rates Natives.

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_changestatusrates` | <玩家> | 需要 ADMFLAG_GENERIC（b，通用管理员） | 修改指定玩家的旁观 tick 状态 | `specrates_test.sp:51` |
| `sm_getstatusrates` | <玩家> | 需要 ADMFLAG_GENERIC（b，通用管理员） | 查询指定玩家的旁观 tick 状态 | `specrates_test.sp:52` |

### ConVar

（本插件未注册 ConVar）

## Survivor MVP Test —— `survivor_mvp_test.sp`

- 源文件：`survivor_mvp_test.sp`
- myinfo：version=1.0，author=Lechuga
- myinfo description（源码原文）：Test MVP functions

### 指令

| 指令 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|
| `sm_mvptest` | 无参数 | 无（任意玩家可用） | 显示 Survivor MVP 的全部可用统计 | `survivor_mvp_test.sp:45` |

### ConVar

（本插件未注册 ConVar）

---

---

---






---


---

## 全局索引

下面两张表供**按名字反查**使用：`ConVar 名 → 所属插件 → 作用`、`指令名 → 所属插件 → 功能与权限`。
同一个名字被多个插件注册时会出现多行，请结合「来源」列与插件分节判断实际生效的那个。

### 全部 ConVar（按名称排序，共 1623 条）

| ConVar | 所属插件 | 默认值 | 类型 | 取值范围 | 作用 | 来源 |
|---|---|---|---|---|---|---|
| `_ai_charger3_airvec_modify_lerp` | Ai-Charger 3.0 | `0.3` | 浮点 | 0.0 ~ 1.0 | 空中速度方向修正的插值因子，0.1~1.0，越小越平滑但需要更多帧 | `ai_charger3.sp:92` |
| `_ai_charger3_melee_bait_maxrange` | Ai-Charger 3.0 | `50.0` | 整数，源码写作浮点 | ≥ 0.0 | 目标持有近战时近战博弈区的最大范围，等于 melee_range 加该值 | `ai_charger3.sp:84` |
| `_ai_charger3_melee_bait_minrange` | Ai-Charger 3.0 | `15.0` | 整数，源码写作浮点 | ≥ 0.0 | 目标持有近战时近战博弈区的最小范围，等于 melee_range 加该值 | `ai_charger3.sp:82` |
| `_ai_smoker3_bhop_nvis_maxang` | Ai-Smoker 3.0 | `75.0` | 整数，源码写作浮点 | ≥ 0.0 | 无生还者视野时速度向量与视角前向向量夹角在该范围内允许连跳 | `ai_smoker3.sp:146` |
| `_ai_tank3_bhop_nvis_maxang` | Ai-Tank 3 | `75.0` | 整数，源码写作浮点 | ≥ 0.0 | 无视野时速度向量与视角前向向量夹角阈值，单位度 | `ai_tank3.sp:74` |
| `_ai_tank3_direct_chase_max_angle` | Ai-Tank 3 | `45.0` | 整数，源码写作浮点 | 0.0 ~ 180.0 | 有视野时路径前瞻方向与目标方向夹角不超过该值才朝预测点直追，否则沿路径连跳，单位度 | `ai_tank3.sp:76` |
| `ah_ai_dynamic_announce` | AnneHappy Dynamic AI Difficulty | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 调档时是否在聊天框提示 | `annehappy_dynamic_ai_difficulty.sp:111` |
| `ah_ai_dynamic_check_interval` | AnneHappy Dynamic AI Difficulty | `5.0` | 整数，源码写作浮点 | 1.0 ~ 60.0 | 回合定档前每隔多少秒从 l4d_stats 重试检查一次平均 PPM，范围 1~60 | `annehappy_dynamic_ai_difficulty.sp:95` |
| `ah_ai_dynamic_config` | AnneHappy Dynamic AI Difficulty | `DEFAULT_CONFIG_PATH` | 字符串或表达式 | 无上下界 | 难度配置文件路径，相对 addons/sourcemod | `annehappy_dynamic_ai_difficulty.sp:101` |
| `ah_ai_dynamic_current_level` | AnneHappy Dynamic AI Difficulty | `0` | 整数，源码写作浮点 | 0.0 ~ 6.0 | 当前回合动态难度：0 未定档，1 简单，2 普通，3 困难，4 专家，5 极限，6 音理 | `annehappy_dynamic_ai_difficulty.sp:113` |
| `ah_ai_dynamic_current_locked` | AnneHappy Dynamic AI Difficulty | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 当前回合动态难度是否已锁定 | `annehappy_dynamic_ai_difficulty.sp:116` |
| `ah_ai_dynamic_current_mode` | AnneHappy Dynamic AI Difficulty | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 当前回合动态难度来源：0 自动，1 固定 | `annehappy_dynamic_ai_difficulty.sp:114` |
| `ah_ai_dynamic_current_ppm` | AnneHappy Dynamic AI Difficulty | `0.0` | 整数，源码写作浮点 | ≥ 0.0 | 当前回合自动定档使用的平均个人 PPM，固定模式为 0 | `annehappy_dynamic_ai_difficulty.sp:115` |
| `ah_ai_dynamic_debug` | AnneHappy Dynamic AI Difficulty | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否输出动态难度调试日志 | `annehappy_dynamic_ai_difficulty.sp:112` |
| `ah_ai_dynamic_enable` | AnneHappy Dynamic AI Difficulty | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用 AnneHappy 动态难度 | `annehappy_dynamic_ai_difficulty.sp:94` |
| `ah_ai_dynamic_enforce_interval` | AnneHappy Dynamic AI Difficulty | `0.0` | 整数，源码写作浮点 | 0.0 ~ 60.0 | 难度锁定后每隔多少秒重刷当前档位 cvar，0 关闭，范围 0~60 | `annehappy_dynamic_ai_difficulty.sp:108` |
| `ah_ai_dynamic_fixed_level` | AnneHappy Dynamic AI Difficulty | `0` | 整数，源码写作浮点 | 0.0 ~ 6.0 | 固定动态难度：0 自动，1 简单，2 普通，3 困难，4 专家，5 极限，6 音理 | `annehappy_dynamic_ai_difficulty.sp:100` |
| `ah_ai_dynamic_ppm_expert` | AnneHappy Dynamic AI Difficulty | `63.70` | 浮点 | ≥ 0.0 | 进入专家难度所需的 l4d_stats 平均 PPM 阈值 | `annehappy_dynamic_ai_difficulty.sp:98` |
| `ah_ai_dynamic_ppm_extreme` | AnneHappy Dynamic AI Difficulty | `77.57` | 浮点 | ≥ 0.0 | 进入极限难度所需的 l4d_stats 平均 PPM 阈值 | `annehappy_dynamic_ai_difficulty.sp:99` |
| `ah_ai_dynamic_ppm_hard` | AnneHappy Dynamic AI Difficulty | `43.23` | 浮点 | ≥ 0.0 | 进入困难难度所需的 l4d_stats 平均 PPM 阈值 | `annehappy_dynamic_ai_difficulty.sp:97` |
| `ah_ai_dynamic_ppm_normal` | AnneHappy Dynamic AI Difficulty | `30.89` | 浮点 | ≥ 0.0 | 进入普通难度所需的 l4d_stats 平均 PPM 阈值 | `annehappy_dynamic_ai_difficulty.sp:96` |
| `ah_ai_dynamic_quarter_min_minutes` | AnneHappy Dynamic AI Difficulty | `300` | 整数，源码写作浮点 | ≥ 0.0 | 玩家本季度样本低于该分钟数时回退使用总积分 PPM | `annehappy_dynamic_ai_difficulty.sp:103` |
| `ah_ai_dynamic_survivor_max_incaps` | AnneHappy Dynamic AI Difficulty | `2` | 整数，源码写作浮点 | -1.0 ~ 10.0 | 动态难度应用时强制恢复的生还者最大倒地次数，-1 不处理 | `annehappy_dynamic_ai_difficulty.sp:110` |
| `ah_ai_dynamic_tank_bhop_override` | AnneHappy Dynamic AI Difficulty | `-1` | 整数，源码写作浮点 | -1.0 ~ 1.0 | Tank 连跳覆盖：-1 跟随档位配置，0 强制关闭，1 强制开启 | `annehappy_dynamic_ai_difficulty.sp:109` |
| `ah_ai_dynamic_threshold_db_config` | AnneHappy Dynamic AI Difficulty | `DEFAULT_THRESHOLD_DB_CONFIG` | 字符串或表达式 | 无上下界 | 每日 PPM 阈值数据库配置名，对应 databases.cfg | `annehappy_dynamic_ai_difficulty.sp:105` |
| `ah_ai_dynamic_threshold_max_age` | AnneHappy Dynamic AI Difficulty | `172800` | 整数，源码写作浮点 | ≥ 0.0 | 数据库阈值的最大有效秒数，0 不检查过期，默认 2 天 | `annehappy_dynamic_ai_difficulty.sp:107` |
| `ah_ai_dynamic_threshold_mode` | AnneHappy Dynamic AI Difficulty | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | PPM 阈值来源：0 使用本 cfg 固定阈值，1 从数据库读取每日分位阈值 | `annehappy_dynamic_ai_difficulty.sp:104` |
| `ah_ai_dynamic_threshold_table` | AnneHappy Dynamic AI Difficulty | `DEFAULT_THRESHOLD_TABLE` | 字符串或表达式 | 无上下界 | 每日 PPM 阈值表名，只允许字母数字下划线 | `annehappy_dynamic_ai_difficulty.sp:106` |
| `ah_ai_dynamic_use_quarter_stats` | AnneHappy Dynamic AI Difficulty | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否优先使用季度积分与季度时间计算玩家 PPM，当前季度数据失真时应关闭 | `annehappy_dynamic_ai_difficulty.sp:102` |
| `ah_ai_dynamic_version` | AnneHappy Dynamic AI Difficulty | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | `annehappy_dynamic_ai_difficulty.sp:117` |
| `ai_aim_offset_sensitivity_hunter` | AI HUNTER | `180.0` | 整数，源码写作浮点 | 0.0 ~ 180.0 | 若目标水平瞄准在该半径内则 hunter 不会直扑，范围 0~180 | `ai_hunter_new.sp:52` |
| `ai_boomer3_path_bhop` | Ai Boomer 3.0 | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否优先使用 anne_nextbot 路径前视连跳 | `ai_boomer_3.sp:106` |
| `ai_boomer3_path_lane_offset` | Ai Boomer 3.0 | `10.0` | 整数，源码写作浮点 | 0.0 ~ 40.0 | 路径连跳的最大稳定侧向分流距离，范围 0~40 | `ai_boomer_3.sp:108` |
| `ai_boomer3_path_lookahead_depth` | Ai Boomer 3.0 | `5` | 整数，源码写作浮点 | 1.0 ~ 16.0 | 路径连跳最大前视节点数，范围 1~16 | `ai_boomer_3.sp:107` |
| `ai_BoomerAirAngles` | Ai-Boomer增强 | `60.0` | 整数，源码写作浮点 | ≥ 0.0 | 空中速度向量与到生还者方向向量夹角大于该值时停止连跳 | `ai_boomer_new.sp:43` |
| `ai_BoomerAutoFrame` | Ai Boomer 2.0 | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否按目标角度自动计算视野在下一目标的帧数 | `ai_boomer_2.sp:82` |
| `ai_BoomerAutoFrame` | Ai Boomer 3.0 | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否按目标角度自动计算视野在下一目标的帧数 | `ai_boomer_3.sp:119` |
| `ai_BoomerBhop` | Ai Boomer 2.0 | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否开启 Boomer 连跳 | `ai_boomer_2.sp:69` |
| `ai_BoomerBhop` | Ai Boomer 3.0 | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否开启 Boomer 连跳 | `ai_boomer_3.sp:99` |
| `ai_BoomerBhop` | Ai-Boomer增强 | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否开启 Boomer 连跳 | `ai_boomer_new.sp:41` |
| `ai_BoomerBhopFirstHopRatio` | Ai Boomer 3.0 | `0.8` | 浮点 | 0.0 ~ 1.0 | 从跑动直接起跳的第一跳可获得的推力比例，0.0 不加推力，1.0 与落地跳相同 | `ai_boomer_3.sp:101` |
| `ai_BoomerBhopMaxSpeed` | Ai Boomer 3.0 | `1000.0` | 整数，源码写作浮点 | ≥ 0.0 | Boomer 连跳的最大水平速度，0 不限制 | `ai_boomer_3.sp:103` |
| `ai_BoomerBhopSpeed` | Ai Boomer 2.0 | `150.0` | 整数，源码写作浮点 | ≥ 0.0 | Boomer 连跳速度 | `ai_boomer_2.sp:70` |
| `ai_BoomerBhopSpeed` | Ai Boomer 3.0 | `150.0` | 整数，源码写作浮点 | ≥ 0.0 | Boomer 连跳推力，落地后立即再起跳时追加，第一跳按 first_hop_ratio 折算 | `ai_boomer_3.sp:100` |
| `ai_BoomerBhopSpeed` | Ai-Boomer增强 | `150.0` | 整数，源码写作浮点 | ≥ 0.0 | Boomer 连跳的速度 | `ai_boomer_new.sp:42` |
| `ai_BoomerBhopStartDistance` | Ai Boomer 3.0 | `2500.0` | 整数，源码写作浮点 | ≥ 0.0 | Boomer 距最近生还者多远时开始连跳 | `ai_boomer_3.sp:104` |
| `ai_BoomerBileFindRange` | Ai Boomer 2.0 | `300` | 整数，源码写作浮点 | ≥ 0.0 | 该距离内有被控或倒地的生还者时 Boomer 优先攻击，0 禁用 | `ai_boomer_2.sp:78` |
| `ai_BoomerBileFindRange` | Ai Boomer 3.0 | `300` | 整数，源码写作浮点 | ≥ 0.0 | 该距离内有被控或倒地的生还者时 Boomer 优先攻击，0 禁用 | `ai_boomer_3.sp:115` |
| `ai_BoomerDegreeForceBile` | Ai Boomer 2.0 | `10` | 整数，源码写作浮点 | ≥ 0.0 | 目标与 Boomer 视角夹角在该值内且能看到目标头部时强制喷吐，0 禁用 | `ai_boomer_2.sp:81` |
| `ai_BoomerDegreeForceBile` | Ai Boomer 3.0 | `10` | 整数，源码写作浮点 | ≥ 0.0 | 目标与 Boomer 视角夹角在该值内且能看到目标头部时强制喷吐，0 禁用 | `ai_boomer_3.sp:118` |
| `ai_BoomerForceBile` | Ai Boomer 2.0 | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否开启生还者进入 Boomer 喷吐范围后强制被喷 | `ai_boomer_2.sp:76` |
| `ai_BoomerForceBile` | Ai Boomer 3.0 | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否开启生还者进入 Boomer 喷吐范围后强制被喷 | `ai_boomer_3.sp:113` |
| `ai_BoomerJumpVomit` | Ai Boomer 2.0 | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否允许 Boomer 已在空中时主动喷吐，不额外强制起跳 | `ai_boomer_2.sp:71` |
| `ai_BoomerJumpVomit` | Ai Boomer 3.0 | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否允许 Boomer 已在空中时主动喷吐 | `ai_boomer_3.sp:105` |
| `ai_BoomerTurnInterval` | Ai Boomer 2.0 | `15` | 整数，源码写作浮点 | ≥ 0.0 | Boomer 喷吐旋转视角时每隔多少帧切换目标 | `ai_boomer_2.sp:79` |
| `ai_BoomerTurnInterval` | Ai Boomer 3.0 | `15` | 整数，源码写作浮点 | ≥ 0.0 | Boomer 喷吐旋转视角时每隔多少帧切换目标 | `ai_boomer_3.sp:116` |
| `ai_BoomerTurnVision` | Ai Boomer 2.0 | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | Boomer 喷吐时是否旋转视角 | `ai_boomer_2.sp:73` |
| `ai_BoomerTurnVision` | Ai Boomer 3.0 | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | Boomer 喷吐时是否旋转视角 | `ai_boomer_3.sp:110` |
| `ai_BoomerUpVision` | Ai Boomer 2.0 | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | Boomer 喷吐时是否上抬视角 | `ai_boomer_2.sp:72` |
| `ai_BoomerUpVision` | Ai Boomer 3.0 | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | Boomer 喷吐时是否上抬视角 | `ai_boomer_3.sp:109` |
| `ai_ChagrerBhopSpeed` | Ai Charger 增强 2.0 版本 | `90.0` | 整数，源码写作浮点 | ≥ 0.0 | Charger 连跳速度 | `ai_charger_2.sp:39` |
| `ai_ChagrerBhopSpeed` | Ai_Jockey 2.0 版本 | `90.0` | 整数，源码写作浮点 | ≥ 0.0 | Charger 连跳速度与加速度的兼容值 | `ai_charger3.sp:116` |
| `ai_charger3_air_speed_floor_ratio` | Ai-Charger 3.0 | `0.50` | 浮点 | 0.0 ~ 1.0 | 空中方向修正使用的起跳保存速度下限比例 | `ai_charger3.sp:95` |
| `ai_charger3_air_turn_budget` | Ai-Charger 3.0 | `30.0` | 整数，源码写作浮点 | 0.0 ~ 89.0 | 每次离地后空中速度方向最多偏离起跳方向的角度，用完后按惯性落地，0 不限制 | `ai_charger3.sp:97` |
| `ai_charger3_air_turn_speed_loss` | Ai-Charger 3.0 | `0.12` | 浮点 | 0.0 ~ 0.5 | 空中实际转向 90 度时损失的水平速度比例，范围 0~0.5 | `ai_charger3.sp:94` |
| `ai_charger3_airvec_modify_interval` | Ai-Charger 3.0 | `0.3` | 浮点 | ≥ 0.0 | 空中平滑转向的基准时间，实际以 0.05 秒短步长执行 | `ai_charger3.sp:90` |
| `ai_charger3_airvec_modify_max_deg` | Ai-Charger 3.0 | `89.0` | 整数，源码写作浮点 | 0.0 ~ 89.0 | 空中转向允许的最大目标偏角，最大 89 度以禁止反向修正 | `ai_charger3.sp:88` |
| `ai_charger3_airvec_modify_min_deg` | Ai-Charger 3.0 | `45.0` | 整数，源码写作浮点 | ≥ 0.0 | 空中速度方向与自身到目标方向夹角超过该值时进行速度修正 | `ai_charger3.sp:86` |
| `ai_charger3_anti_retreat` | Ai-Charger 3.0 | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否禁止 Charger 逃跑：0 禁止，1 允许 | `ai_charger3.sp:108` |
| `ai_charger3_bait_max_duration` | Ai-Charger 3.0 | `7.0` | 整数，源码写作浮点 | ≥ 0.0 | Charger 进入博弈状态的最大允许时间 | `ai_charger3.sp:99` |
| `ai_charger3_bhop` | Ai-Charger 3.0 | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否允许 Charger 在接近状态时连跳：0 禁止，1 允许 | `ai_charger3.sp:51` |
| `ai_charger3_bhop_before_charge` | Ai-Charger 3.0 | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否允许在冲锋前连跳：0 禁止，1 允许 | `ai_charger3.sp:64` |
| `ai_charger3_bhop_direct_dist` | Ai-Charger 3.0 | `400.0` | 整数，源码写作浮点 | ≥ 0.0 | 接近状态中切换到朝目标方向直线连跳的距离阈值 | `ai_charger3.sp:55` |
| `ai_charger3_bhop_first_hop_ratio` | Ai-Charger 3.0 | `0.8` | 浮点 | 0.0 ~ 1.0 | 从跑动直接起跳的第一跳可获得的加速度比例，0.0 只改方向不加速，1.0 与落地跳相同 | `ai_charger3.sp:59` |
| `ai_charger3_bhop_impulse` | Ai-Charger 3.0 | `100.0` | 整数，源码写作浮点 | ≥ 0.0 | 连跳加速度，落地后立即再起跳时追加，第一跳按 first_hop_ratio 折算 | `ai_charger3.sp:57` |
| `ai_charger3_bhop_max_dist` | Ai-Charger 3.0 | `9999.0` | 整数，源码写作浮点 | ≥ 0.0 | 允许连跳的最大距离 | `ai_charger3.sp:54` |
| `ai_charger3_bhop_max_speed` | Ai-Charger 3.0 | `800` | 整数，源码写作浮点 | ≥ 0.0 | 连跳的最大限制速度 | `ai_charger3.sp:62` |
| `ai_charger3_bhop_min_dist` | Ai-Charger 3.0 | `100.0` | 整数，源码写作浮点 | ≥ 0.0 | 禁止连跳的最小距离，小于该距离转为博弈状态 | `ai_charger3.sp:53` |
| `ai_charger3_bhop_min_speed` | Ai-Charger 3.0 | `200` | 整数，源码写作浮点 | ≥ 0.0 | 允许连跳的最小速度 | `ai_charger3.sp:61` |
| `ai_charger3_bhop_no_vision` | Ai-Charger 3.0 | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否允许 Charger 无目标视野时连跳：0 禁止，1 允许 | `ai_charger3.sp:66` |
| `ai_charger3_bhop_nvis_maxang` | Ai-Charger 3.0 | `90.0` | 整数，源码写作浮点 | ≥ 0.0 | 无生还者视野时速度向量与视角前向向量夹角在该范围内允许连跳 | `ai_charger3.sp:68` |
| `ai_charger3_bhop_strafe_maxdeg` | Ai-Charger 3.0 | `55.0` | 整数，源码写作浮点 | 0.0 ~ 89.0 | 侧向连跳的最大角度，范围 0~89 | `ai_charger3.sp:76` |
| `ai_charger3_bhop_strafe_mindeg` | Ai-Charger 3.0 | `30.0` | 整数，源码写作浮点 | ≥ -1.0 | 侧向连跳的最小随机偏移角度，大于 0 启用，-1.0 禁用该功能 | `ai_charger3.sp:74` |
| `ai_charger3_bhop_strafe_mindist` | Ai-Charger 3.0 | `400.0` | 整数，源码写作浮点 | ≥ 0.0 | 与目标距离小于该值时禁止连跳方向左右偏移 | `ai_charger3.sp:72` |
| `ai_charger3_bhop_strafe_once_dist` | Ai-Charger 3.0 | `200.0` | 整数，源码写作浮点 | 无上下界 | 允许侧向连跳一次的最小距离，小于该距离不允许侧向连跳 | `ai_charger3.sp:78` |
| `ai_charger3_bhop_strafe_twice_dist` | Ai-Charger 3.0 | `400.0` | 整数，源码写作浮点 | 无上下界 | 允许侧向连跳两次的最小距离，大于该距离才允许侧向连跳两次 | `ai_charger3.sp:80` |
| `ai_charger3_evade_moveto_refresh_interval` | Ai-Charger 3.0 | `1.0` | 浮点 | 0.1 ~ 10.0 | ChargerEvade 追击时检查并刷新 BehaviorMoveTo 目标坐标的间隔，范围 0.1~10 | `ai_charger3.sp:110` |
| `ai_charger3_log_level` | Ai-Charger 3.0 | `1` | 整数 | 无上下界 | 日志记录级别：1 关闭，2 控制台输出，4 log 文件，8 聊天框，16 服务器控制台，32 error 文件，可相加 | `ai_charger3.sp:141` |
| `ai_charger3_melee_bait_blacklist_dur` | Ai-Charger 3.0 | `10.0` | 整数，源码写作浮点 | ≥ 0.0 | 因近战僵持放弃某目标后，该目标进入选目标黑名单的时长 | `ai_charger3.sp:102` |
| `ai_charger3_melee_bait_orbit` | Ai-Charger 3.0 | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 近战博弈处于博弈区内时是否沿目标切向绕圈代替原地急停：0 关闭则原地站定 | `ai_charger3.sp:100` |
| `ai_charger3_melee_bait_stalemate_switch` | Ai-Charger 3.0 | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 近战僵持升级到最后一级时是否允许改冲其他生还者 | `ai_charger3.sp:101` |
| `ai_charger3_path_lookahead_maxdepth` | Ai-Charger 3.0 | `10` | 整数，源码写作浮点 | ≥ 0.0 | 向前搜索一步可到达 PathSegment 的最大深度 | `ai_charger3.sp:112` |
| `ai_charger3_plugin_name` | Ai-Charger 3.0 | `ai_charger3` | 字符串或表达式 | 无上下界 | 插件名（非行为配置） | `ai_charger3.sp:135` |
| `ai_charger3_prob_charge_chk_dur` | Ai-Charger 3.0 | `0.5` | 浮点 | ≥ 0.0 | Charger 在博弈状态概率冲锋的检测间隔 | `ai_charger3.sp:104` |
| `ai_charger3_prob_charge_prob` | Ai-Charger 3.0 | `0.8` | 浮点 | ≥ 0.0 | Charger 在博弈状态概率冲锋的概率 | `ai_charger3.sp:106` |
| `ai_charger3_target_watch_maxdeg` | Ai-Charger 3.0 | `22.5` | 浮点 | 0.0 ~ 180.0 | 目标视角与到 Charger 位置向量夹角小于该值时认为目标正在看 Charger，范围 0~180 | `ai_charger3.sp:70` |
| `ai_ChargerAimOffset` | Ai Charger 增强 2.0 版本 | `30.0` | 整数，源码写作浮点 | ≥ 0.0 | 目标瞄准水平与 Charger 夹角处于该范围内时 Charger 不冲锋 | `ai_charger_2.sp:42` |
| `ai_ChargerAimOffset` | Ai-Charger增强 | `30` | 整数，源码写作浮点 | ≥ 0.0 | 目标瞄准角度与 Charger 处于该角度内时 Charger 不冲锋 | `ai_charger_new.sp:47` |
| `ai_ChargerAimOffset` | Ai_Jockey 2.0 版本 | `30.0` | 整数，源码写作浮点 | ≥ 0.0 | 目标视角与 Charger 的兼容判定角度 | `ai_charger3.sp:119` |
| `ai_ChargerAirAngles` | Ai-Charger增强 | `60.0` | 整数，源码写作浮点 | ≥ 0.0 | 空中速度向量与到生还者方向向量夹角大于该值时停止连跳 | `ai_charger_new.sp:49` |
| `ai_ChargerBhop` | Ai Charger 增强 2.0 版本 | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否开启 Charger 连跳 | `ai_charger_2.sp:38` |
| `ai_ChargerBhop` | Ai-Charger增强 | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否开启 Charger 连跳 | `ai_charger_new.sp:43` |
| `ai_ChargerBhop` | Ai_Jockey 2.0 版本 | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否开启 Charger 连跳，兼容旧插件名 | `ai_charger3.sp:115` |
| `ai_ChargerBhopSpeed` | Ai-Charger增强 | `90.0` | 整数，源码写作浮点 | ≥ 0.0 | Charger 连跳的速度 | `ai_charger_new.sp:44` |
| `ai_ChargerChargeDistance` | Ai Charger 增强 2.0 版本 | `250.0` | 整数，源码写作浮点 | ≥ 0.0 | Charger 只能在与目标距离小于该值时冲锋 | `ai_charger_2.sp:40` |
| `ai_ChargerChargeDistance` | Ai_Jockey 2.0 版本 | `250.0` | 整数，源码写作浮点 | ≥ 0.0 | Charger 旧版冲锋距离，转换为 3.0 的直线连跳阈值 | `ai_charger3.sp:117` |
| `ai_ChargerChargeHeightDiff` | Ai Charger 增强 2.0 版本 | `80.0` | 整数，源码写作浮点 | 无上下界 | 允许直接冲锋时目标高出自身的最大高度差，小于等于 0 关闭检测 | `ai_charger_2.sp:46` |
| `ai_ChargerChargeHeightDiff` | Ai_Jockey 2.0 版本 | `80.0` | 整数，源码写作浮点 | 无上下界 | 允许直接冲锋的最大高度差，小于等于 0 时使用默认值 | `ai_charger3.sp:123` |
| `ai_ChargerCoolTime` | Ai-Charger增强 | `12` | 整数，源码写作浮点 | 0.0 ~ 1.0 | Charger 多少秒后才能再次冲锋 | `ai_charger_new.sp:42` |
| `ai_ChargerExtraTargetDistance` | Ai Charger 增强 2.0 版本 | `0,350` | 字符串或表达式 | 无上下界 | Charger 在该范围内寻找其他有效目标，逗号分隔且无空格 | `ai_charger_2.sp:41` |
| `ai_ChargerExtraTargetDistance` | Ai_Jockey 2.0 版本 | `0,350` | 字符串或表达式 | 无上下界 | Charger 额外目标范围，格式为最小距离,最大距离 | `ai_charger3.sp:118` |
| `ai_ChargerMeleeAvoid` | Ai Charger 增强 2.0 版本 | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否开启 Charger 近战回避 | `ai_charger_2.sp:43` |
| `ai_ChargerMeleeAvoid` | Ai_Jockey 2.0 版本 | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用 Charger 近战回避 | `ai_charger3.sp:120` |
| `ai_ChargerMeleeDamage` | Ai Charger 增强 2.0 版本 | `350` | 整数，源码写作浮点 | ≥ 0.0 | Charger 血量小于该值时不会直接冲锋持有近战的生还者 | `ai_charger_2.sp:44` |
| `ai_ChargerMeleeDamage` | Ai_Jockey 2.0 版本 | `350` | 整数，源码写作浮点 | ≥ 0.0 | Charger 血量低于该值时避免近战目标 | `ai_charger3.sp:121` |
| `ai_ChargerStartChargeDistance` | Ai-Charger增强 | `300` | 整数，源码写作浮点 | ≥ 0.0 | Charger 只能在与目标距离小于该值时冲锋 | `ai_charger_new.sp:46` |
| `ai_ChargerStartChargeHealth` | Ai-Charger增强 | `350` | 整数，源码写作浮点 | ≥ 0.0 | Charger 生命值低于该值才会冲锋 | `ai_charger_new.sp:48` |
| `ai_ChargerTarget` | Ai Charger 增强 2.0 版本 | `1` | 整数，源码写作浮点 | 1.0 ~ 2.0 | Charger 目标选择：1 自然目标选择，2 优先取最近目标，3 优先撞人多处 | `ai_charger_2.sp:45` |
| `ai_ChargerTarget` | Ai-Charger增强 | `3` | 整数，源码写作浮点 | 1.0 ~ 2.0 | Charger 目标选择：1 自然目标选择，2 优先撞人多处，3 优先取最近目标 | `ai_charger_new.sp:45` |
| `ai_ChargerTarget` | Ai_Jockey 2.0 版本 | `1` | 开关 0 或 1 | 1.0 ~ 3.0 | Charger 目标选择：1 原生，2 最近，3 人群中心 | `ai_charger3.sp:122` |
| `ai_fast_pounce_proximity` | AI HUNTER | `1000.0` | 整数，源码写作浮点 | 无上下界 | 从多远开始快速飞扑 | `ai_hunter_new.sp:47` |
| `ai_hunter_aim_offset` | Ai Hunter 2.0 (fixed) | `360.0` | 整数，源码写作浮点 | 0.0 ~ 360.0 | 与目标水平角度在该范围内且在直扑范围外时 hunter 不直扑，范围 0~360 | `ai_hunter_2.sp:124` |
| `ai_hunter_angle_diff` | Ai Hunter 2.0 (fixed) | `3` | 整数，源码写作浮点 | ≥ 0.0 | 随机侧飞时左右累计次数差的上限 | `ai_hunter_2.sp:145` |
| `ai_hunter_angle_mean` | Ai Hunter 2.0 (fixed) | `10.0` | 整数，源码写作浮点 | ≥ 0.0 | 由随机数生成的基本角度 | `ai_hunter_2.sp:118` |
| `ai_hunter_angle_std` | Ai Hunter 2.0 (fixed) | `20.0` | 整数，源码写作浮点 | ≥ 0.0 | 与基本角度允许的偏差范围 | `ai_hunter_2.sp:120` |
| `ai_hunter_back_vision` | Ai Hunter 2.0 (fixed) | `25` | 整数，源码写作浮点 | 0.0 ~ 100.0 | hunter 在空中背对生还者视角的概率百分比，0 禁用，范围 0~100 | `ai_hunter_2.sp:132` |
| `ai_hunter_fast_pounce_distance` | Ai Hunter 2.0 (fixed) | `1000.0` | 整数，源码写作浮点 | ≥ 0.0 | hunter 开始快速突袭的距离 | `ai_hunter_2.sp:114` |
| `ai_hunter_high_pounce` | Ai Hunter 2.0 (fixed) | `400` | 整数，源码写作浮点 | ≥ 0.0 | 高度差超过该值时可直接高扑，单位为 Hammer 坐标 Z | `ai_hunter_2.sp:138` |
| `ai_hunter_melee_first` | Ai Hunter 2.0 (fixed) | `300.0,1000.0` | 字符串或表达式 | 无上下界 | 每次准备突袭时是否先按右键，格式为最小,最大距离，0 禁用 | `ai_hunter_2.sp:135` |
| `ai_hunter_no_sight_pounce_range` | Ai Hunter 2.0 (fixed) | `300.0,250.0` | 字符串或表达式 | 无上下界 | 不可见目标时允许飞扑的范围，格式为水平,垂直，0 表示该维度禁用 | `ai_hunter_2.sp:128` |
| `ai_hunter_straight_pounce_distance` | Ai Hunter 2.0 (fixed) | `200.0` | 整数，源码写作浮点 | ≥ 0.0 | hunter 允许直扑的范围 | `ai_hunter_2.sp:122` |
| `ai_hunter_vertical_angle` | Ai Hunter 2.0 (fixed) | `7.0` | 整数，源码写作浮点 | ≥ 0.0 | hunter 突袭的垂直角度上限，单位度 | `ai_hunter_2.sp:116` |
| `ai_hunter_wall_detect_distance` | Ai Hunter 2.0 (fixed) | `-1.0` | 整数，源码写作浮点 | 无上下界 | 视线前方墙体检测的射线长度，-1 关闭 | `ai_hunter_2.sp:142` |
| `ai_JockeyAirAngles` | Ai_Jockey增强 | `60.0` | 整数，源码写作浮点 | 0.0 ~ 180.0 | Jockey 速度方向与到目标向量方向夹角大于该角度时改变方向，范围 0~180 | `ai_jockey_new.sp:48` |
| `ai_JockeyAllowInterControl` | Ai_Jockey 2.0 版本 | `0` | 整数 | 无上下界 | Jockey 优先寻找被这些特感控制的生还者以抢控或补控，0 表示关闭该功能 | `ai_jockey_2.sp:67` |
| `ai_JockeyBackVision` | Ai_Jockey 2.0 版本 | `50` | 整数，源码写作浮点 | 0.0 ~ 100.0 | Jockey 在空中时以该概率向当前视角反方向看，范围 0~100 | `ai_jockey_2.sp:68` |
| `ai_JockeyBhopSpeed` | Ai_Jockey 2.0 版本 | `80.0` | 整数，源码写作浮点 | ≥ 0.0 | Jockey 连跳的速度 | `ai_jockey_2.sp:60` |
| `ai_JockeyBhopSpeed` | Ai_Jockey增强 | `80.0` | 整数，源码写作浮点 | ≥ 0.0 | Jockey 连跳的速度 | `ai_jockey_new.sp:45` |
| `ai_jockeyNoActionChance` | Ai_Jockey 2.0 版本 | `20,20,60` | 字符串或表达式 | 0.0 ~ 100.0 | Jockey 执行冻结行动、向后跳、高跳三种行为的概率，逗号分隔，范围 0~100 | `ai_jockey_2.sp:66` |
| `ai_JockeyShovedCooldown` | Ai_Jockey 2.0 版本 | `3.0` | 整数，源码写作浮点 | ≥ 0.0 | Jockey 被推后多少秒内禁止再次扑跳 | `ai_jockey_2.sp:69` |
| `ai_JockeySpecialJumpAngle` | Ai_Jockey 2.0 版本 | `60` | 整数，源码写作浮点 | 0.0 ~ 180.0 | 目标正看着 Jockey 且夹角在该范围内时 Jockey 尝试骗推，范围 0~180 | `ai_jockey_2.sp:64` |
| `ai_JockeySpecialJumpChance` | Ai_Jockey 2.0 版本 | `60` | 整数，源码写作浮点 | 0.0 ~ 100.0 | Jockey 执行骗推的概率百分比，范围 0~100 | `ai_jockey_2.sp:65` |
| `ai_JockeyStartHopDistance` | Ai_Jockey 2.0 版本 | `800` | 整数，源码写作浮点 | ≥ 0.0 | Jockey 距生还者多远开始主动连跳 | `ai_jockey_2.sp:61` |
| `ai_JockeyStartHopDistance` | Ai_Jockey增强 | `800.0` | 整数，源码写作浮点 | ≥ 0.0 | Jockey 距生还者多远开始主动连跳 | `ai_jockey_new.sp:46` |
| `ai_JockeyStumbleRadius` | Ai_Jockey 2.0 版本 | `50` | 整数，源码写作浮点 | ≥ 0.0 | Jockey 骑上人后对多少范围内的生还者产生硬直 | `ai_jockey_2.sp:62` |
| `ai_JockeyStumbleRadius` | Ai_Jockey增强 | `50.0` | 整数，源码写作浮点 | ≥ 0.0 | Jockey 骑上人后对多少范围内的生还者产生硬直 | `ai_jockey_new.sp:47` |
| `ai_pounce_angle_mean` | AI HUNTER | `10.0` | 整数，源码写作浮点 | 无上下界 | 高斯随机数生成的角度均值 | `ai_hunter_new.sp:49` |
| `ai_pounce_angle_std` | AI HUNTER | `20.0` | 整数，源码写作浮点 | 无上下界 | 高斯随机数生成的一个标准差 | `ai_hunter_new.sp:50` |
| `ai_pounce_vertical_angle` | AI HUNTER | `7.0` | 整数，源码写作浮点 | 无上下界 | AI hunter 飞扑的垂直角度限制 | `ai_hunter_new.sp:48` |
| `ai_smoker3_airvec_modify_degree` | Ai-Smoker 3.0 | `50.0` | 整数，源码写作浮点 | ≥ 0.0 | 空中速度方向与自身到目标方向夹角超过该值时进行速度修正 | `ai_smoker3.sp:148` |
| `ai_smoker3_airvec_modify_degree_max` | Ai-Smoker 3.0 | `105.0` | 整数，源码写作浮点 | ≥ 0.0 | 空中速度方向与自身到目标方向夹角超过该值时不进行速度修正 | `ai_smoker3.sp:149` |
| `ai_smoker3_airvec_modify_interval` | Ai-Smoker 3.0 | `0.3` | 浮点 | ≥ 0.1 | 空中速度修正间隔，最小 0.1 | `ai_smoker3.sp:150` |
| `ai_smoker3_anti_retreat` | Ai-Smoker 3.0 | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否防止 Smoker 无技能时逃跑，将其改为追击 | `ai_smoker3.sp:156` |
| `ai_smoker3_bhop` | Ai-Smoker 3.0 | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否允许 Smoker 连跳 | `ai_smoker3.sp:131` |
| `ai_smoker3_bhop_max_dist` | Ai-Smoker 3.0 | `9999` | 整数，源码写作浮点 | ≥ 0.0 | 允许连跳的最大距离，与目标距离大于该值不允许连跳 | `ai_smoker3.sp:141` |
| `ai_smoker3_bhop_max_speed` | Ai-Smoker 3.0 | `1000` | 整数，源码写作浮点 | ≥ 0.0 | 连跳时的最大速度 | `ai_smoker3.sp:138` |
| `ai_smoker3_bhop_min_dist` | Ai-Smoker 3.0 | `75` | 整数，源码写作浮点 | ≥ 0.0 | 允许连跳的最小距离，与目标距离小于该值不允许连跳 | `ai_smoker3.sp:140` |
| `ai_smoker3_bhop_min_speed` | Ai-Smoker 3.0 | `200` | 整数，源码写作浮点 | ≥ 0.0 | 允许连跳的最小速度 | `ai_smoker3.sp:137` |
| `ai_smoker3_bhop_no_vision` | Ai-Smoker 3.0 | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否允许无目标视野情况下连跳 | `ai_smoker3.sp:133` |
| `ai_smoker3_bhop_side_maxang` | Ai-Smoker 3.0 | `30.0` | 整数，源码写作浮点 | 0.0 ~ 180.0 | 连跳方向最大左右侧偏角度，范围 0~180 | `ai_smoker3.sp:144` |
| `ai_smoker3_bhop_side_minang` | Ai-Smoker 3.0 | `15.0` | 整数，源码写作浮点 | 0.0 ~ 180.0 | 连跳方向最小左右侧偏角度，范围 0~180 | `ai_smoker3.sp:143` |
| `ai_smoker3_jump_pull` | Ai-Smoker 3.0 | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否允许 Smoker 已在空中时主动吐舌 | `ai_smoker3.sp:151` |
| `ai_smoker3_log_level` | Ai-Smoker 3.0 | `32` | 整数 | 无上下界 | 日志记录级别：1 关闭，2 控制台，4 log 文件，8 聊天框，16 服务器控制台，32 error 文件，可相加 | `ai_smoker3.sp:169` |
| `ai_smoker3_move2_newtar_interval` | Ai-Smoker 3.0 | `1.0` | 整数，源码写作浮点 | ≥ 0.0 | 无技能追击时检测最近生还者并更换目标的间隔秒数 | `ai_smoker3.sp:158` |
| `ai_smoker3_plugin_name` | Ai-Smoker 3.0 | `ai_smoker3` | 字符串或表达式 | 无上下界 | 插件名（非行为配置） | `ai_smoker3.sp:164` |
| `ai_smoker3_pull_back_vision` | Ai-Smoker 3.0 | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否允许 Smoker 拉人时视角转向背后 | `ai_smoker3.sp:153` |
| `ai_smoker3_stop_warn_snd` | Ai-Smoker 3.0 | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否停止 Smoker 准备吐舌时的警告音效 | `ai_smoker3.sp:161` |
| `ai_SmokerBhop` | Ai_Smoker增强 | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否开启 Smoker 连跳 | `ai_smoker_new.sp:73` |
| `ai_SmokerBhopSpeed` | Ai-Smoker 3.0 | `120` | 整数，源码写作浮点 | ≥ 0.0 | 连跳加速度 | `ai_smoker3.sp:135` |
| `ai_SmokerBhopSpeed` | Ai_Smoker增强 | `90.0` | 整数，源码写作浮点 | ≥ 0.0 | Smoker 连跳的速度 | `ai_smoker_new.sp:74` |
| `ai_SmokerDistantPercent` | Ai_Smoker增强 | `0.80` | 浮点 | ≥ 0.0 | 舌头处于该系数乘以舌头长度的距离内时立刻拉人 | `ai_smoker_new.sp:80` |
| `ai_SmokerLeftBehindDistance` | Ai_Smoker增强 | `7.0` | 整数，源码写作浮点 | ≥ 0.0 | 玩家距离团队多远判定为落后或超前 | `ai_smoker_new.sp:79` |
| `ai_SmokerMeleeAvoid` | Ai_Smoker增强 | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | Smoker 的目标若手持近战则切换目标 | `ai_smoker_new.sp:76` |
| `ai_SmokerTarget` | Ai_Smoker增强 | `1` | 整数，源码写作浮点 | 1.0 ~ 4.0 | Smoker 优先目标：1 距离最近，2 手持霰弹枪者，3 落单或超前者，4 正在换弹者 | `ai_smoker_new.sp:75` |
| `ai_SpiiterDieAfterSpit` | Ai Spitter 3.0 | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否开启 Spitter 吐完痰后处死功能 | `ai_spitter_3.sp:77` |
| `ai_SpiiterDieAfterSpit` | Ai-Spitter-Enhance 2.0 | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否开启 Spitter 吐完痰后处死功能 | `ai_spitter_2.sp:54` |
| `ai_spitter3_air_spit` | Ai Spitter 3.0 | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否允许 AI Spitter 空中吐痰：0 禁止，1 允许 | `ai_spitter_3.sp:78` |
| `ai_spitter3_path_bhop` | Ai Spitter 3.0 | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否优先使用 anne_nextbot 路径前视连跳 | `ai_spitter_3.sp:72` |
| `ai_spitter3_path_lane_offset` | Ai Spitter 3.0 | `12.0` | 整数，源码写作浮点 | 0.0 ~ 40.0 | 路径连跳的最大稳定侧向分流距离，范围 0~40 | `ai_spitter_3.sp:74` |
| `ai_spitter3_path_lookahead_depth` | Ai Spitter 3.0 | `6` | 整数，源码写作浮点 | 1.0 ~ 16.0 | 路径连跳最大前视节点数，范围 1~16 | `ai_spitter_3.sp:73` |
| `ai_SpitterAirAngle` | Ai-Spitter增强 | `55.0` | 整数，源码写作浮点 | ≥ 0.0 | Spitter 连跳时速度与到目标生还者方向夹角超过该角度即停止连跳 | `ai_spitter_new.sp:40` |
| `ai_SpitterBhop` | Ai Spitter 3.0 | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否开启 Spitter 连跳功能 | `ai_spitter_3.sp:67` |
| `ai_SpitterBhop` | Ai-Spitter-Enhance 2.0 | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否开启 Spitter 连跳功能 | `ai_spitter_2.sp:50` |
| `ai_SpitterBhop` | Ai-Spitter增强 | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否开启 Spitter 连跳 | `ai_spitter_new.sp:35` |
| `ai_SpitterBhopFirstHopRatio` | Ai Spitter 3.0 | `0.8` | 浮点 | 0.0 ~ 1.0 | 从跑动直接起跳的第一跳可获得的推力比例，0.0 不加推力，1.0 与落地跳相同 | `ai_spitter_3.sp:69` |
| `ai_SpitterBhopMaxSpeed` | Ai Spitter 3.0 | `1000.0` | 整数，源码写作浮点 | ≥ 0.0 | Spitter 连跳的最大水平速度，0 不限制 | `ai_spitter_3.sp:70` |
| `ai_SpitterBhopSpeed` | Ai Spitter 3.0 | `100` | 整数，源码写作浮点 | ≥ 0.0 | Spitter 连跳推力，落地后立即再起跳时追加，第一跳按 first_hop_ratio 折算 | `ai_spitter_3.sp:68` |
| `ai_SpitterBhopSpeed` | Ai-Spitter-Enhance 2.0 | `100` | 整数，源码写作浮点 | ≥ 0.0 | Spitter 连跳的速度 | `ai_spitter_2.sp:51` |
| `ai_SpitterBhopSpeed` | Ai-Spitter增强 | `90.0` | 整数，源码写作浮点 | ≥ 0.0 | Spitter 连跳的速度 | `ai_spitter_new.sp:36` |
| `ai_SpitterBhopStartBhopDistance` | Ai-Spitter增强 | `2000.0` | 整数，源码写作浮点 | ≥ 0.0 | Spitter 在什么距离开始连跳 | `ai_spitter_new.sp:37` |
| `ai_SpitterBhopStartDistance` | Ai Spitter 3.0 | `2500.0` | 整数，源码写作浮点 | ≥ 0.0 | Spitter 距最近生还者多远开始连跳 | `ai_spitter_3.sp:71` |
| `ai_SpitterInstantKill` | Ai-Spitter增强 | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | Spitter 吐完痰之后是否处死 | `ai_spitter_new.sp:39` |
| `ai_SpitterPinnedPr` | Ai Spitter 3.0 | `6,3,1,5` | 字符串或表达式 | 无上下界 | 被控目标优先级，按被控特感编号逗号分隔 | `ai_spitter_3.sp:76` |
| `ai_SpitterPinnedPr` | Ai-Spitter-Enhance 2.0 | `6,3,1,5` | 字符串或表达式 | 无上下界 | 被控目标优先级，按被控特感编号逗号分隔 | `ai_spitter_2.sp:53` |
| `ai_SpitterTarget` | Ai Spitter 3.0 | `3` | 整数，源码写作浮点 | 1.0 ~ 4.0 | Spitter 目标选择：1 默认，2 最近，3 被控优先否则第一个生还，4 人多处 | `ai_spitter_3.sp:75` |
| `ai_SpitterTarget` | Ai-Spitter-Enhance 2.0 | `3` | 整数，源码写作浮点 | 1.0 ~ 4.0 | Spitter 目标选择：1 默认，2 最近，3 被控优先否则第一个生还，4 人多处 | `ai_spitter_2.sp:52` |
| `ai_SpitterTarget` | Ai-Spitter增强 | `3` | 整数，源码写作浮点 | 1.0 ~ 3.0 | Spitter 目标选择：1 默认，2 人多处优先，3 被扑撞拉者优先，无则取 2 | `ai_spitter_new.sp:38` |
| `ai_straight_pounce_proximity` | AI HUNTER | `200.0` | 整数，源码写作浮点 | 无上下界 | 距最近生还者多远时 hunter 考虑直扑 | `ai_hunter_new.sp:51` |
| `ai_tank3_airvec_modify_degree` | Ai-Tank 3 | `45.0` | 整数，源码写作浮点 | ≥ 0.0 | 追人时空中速度方向与目标方向夹角大于等于该值开始修正，单位度 | `ai_tank3.sp:83` |
| `ai_tank3_airvec_modify_degree_max` | Ai-Tank 3 | `135.0` | 整数，源码写作浮点 | ≥ 0.0 | 角度大于该值时不再修正，实际最大 89 度 | `ai_tank3.sp:84` |
| `ai_tank3_airvec_modify_interval` | Ai-Tank 3 | `0.3` | 浮点 | ≥ 0.1 | 空中转向平滑响应时间，单位秒，每 0.05 秒检查一次，最小 0.1 | `ai_tank3.sp:85` |
| `ai_tank3_back_fist` | Ai-Tank 3 | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否允许通背拳，即可拍到背后的人 | `ai_tank3.sp:92` |
| `ai_tank3_back_fist_max_spd` | Ai-Tank 3 | `50.0` | 整数，源码写作浮点 | ≥ -1.0 | 通背拳允许的最大移动速度，超过则禁用，-1 表示不限制 | `ai_tank3.sp:94` |
| `ai_tank3_back_fist_range` | Ai-Tank 3 | `128.0` | 整数，源码写作浮点 | ≥ -1.0 | 通背拳距离，-1 表示使用 tank_swing_range | `ai_tank3.sp:93` |
| `ai_tank3_back_fist_window` | Ai-Tank 3 | `3.0` | 整数，源码写作浮点 | ≥ 0.0 | 通背拳窗口秒数，Tank 爪击命中后开启或刷新 | `ai_tank3.sp:95` |
| `ai_tank3_bhop_first_hop_ratio` | Ai-Tank 3 | `1.0` | 开关 0 或 1 | 0.0 ~ 1.0 | 从跑动直接起跳的第一跳可获得的加速度比例，0.0 不加速只改方向，1.0 与落地跳相同 | `ai_tank3.sp:72` |
| `ai_tank3_bhop_impulse` | Ai-Tank 3 | `60` | 整数，源码写作浮点 | ≥ 0.0 | 连跳的加速度，落地后立即再起跳时追加，第一跳按 first_hop_ratio 折算 | `ai_tank3.sp:71` |
| `ai_tank3_bhop_max_dist` | Ai-Tank 3 | `9999` | 整数，源码写作浮点 | ≥ 0.0 | 开始连跳的最大距离 | `ai_tank3.sp:68` |
| `ai_tank3_bhop_max_speed` | Ai-Tank 3 | `1000` | 整数，源码写作浮点 | ≥ 0.0 | 连跳的最大速度 | `ai_tank3.sp:70` |
| `ai_tank3_bhop_min_speed` | Ai-Tank 3 | `200` | 整数，源码写作浮点 | ≥ 0.0 | 连跳的最小速度 | `ai_tank3.sp:69` |
| `ai_tank3_bhop_no_vision` | Ai-Tank 3 | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 无视野时是否允许连跳 | `ai_tank3.sp:73` |
| `ai_tank3_bhop_reverse_brake` | Ai-Tank 3 | `1500` | 整数，源码写作浮点 | ≥ 0.0 | 直追的一跳中目标跑到身后时空中每秒减掉的水平速度，最多减到跑速，0 不刹车 | `ai_tank3.sp:80` |
| `ai_tank3_bhop_reverse_hop` | Ai-Tank 3 | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 直追时目标跑到身后则落地改为掉头跳；0 为旧行为，会顺着原方向继续跳 | `ai_tank3.sp:79` |
| `ai_tank3_bhop_strafe_angle` | Ai-Tank 3 | `15.0` | 整数，源码写作浮点 | 0.0 ~ 35.0 | 远距离安全直追时逐跳左右交替的偏角，0 关闭，范围 0~35 | `ai_tank3.sp:77` |
| `ai_tank3_bhop_strafe_min_dist` | Ai-Tank 3 | `600.0` | 整数，源码写作浮点 | ≥ 0.0 | 距离目标超过该值才主动左右连跳 | `ai_tank3.sp:78` |
| `ai_tank3_enable` | Ai-Tank 3 | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用插件：0 禁用，1 启用 | `ai_tank3.sp:63` |
| `ai_tank3_head_block_enable` | Ai-Tank 3 | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用 Tank 反头顶卡逻辑 | `ai_tank3.sp:106` |
| `ai_tank3_head_block_force_rock_range` | Ai-Tank 3 | `250.0` | 整数，源码写作浮点 | ≥ 0.0 | 强制投石前 Tank 需与目标拉开的最小水平距离 | `ai_tank3.sp:112` |
| `ai_tank3_head_block_force_rock_release_h` | Ai-Tank 3 | `400` | 整数，源码写作浮点 | ≥ 0.0 | 强制投石期间目标水平离开多远即清除强制状态，小于等于 0 不检测 | `ai_tank3.sp:113` |
| `ai_tank3_head_block_force_rock_release_v` | Ai-Tank 3 | `250` | 整数，源码写作浮点 | ≥ 0.0 | 强制投石期间目标垂直离开多远即清除强制状态，小于等于 0 不检测 | `ai_tank3.sp:114` |
| `ai_tank3_head_block_force_rock_time` | Ai-Tank 3 | `20.0` | 整数，源码写作浮点 | ≥ 0.0 | 强制投石尝试的最长秒数 | `ai_tank3.sp:111` |
| `ai_tank3_head_block_horizontal` | Ai-Tank 3 | `65.0` | 整数，源码写作浮点 | ≥ 0.0 | 触发头顶卡判定的水平距离上限 | `ai_tank3.sp:109` |
| `ai_tank3_head_block_ignore_time` | Ai-Tank 3 | `10.0` | 整数，源码写作浮点 | ≥ 0.0 | 判定恶意卡位后屏蔽该生还者的秒数 | `ai_tank3.sp:110` |
| `ai_tank3_head_block_ride_enable` | Ai-Tank 3 | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否把 Tank 正上方的生还者按骑头处理，能命中则上挥否则原地投石 | `ai_tank3.sp:115` |
| `ai_tank3_head_block_ride_horizontal` | Ai-Tank 3 | `40.0` | 整数，源码写作浮点 | ≥ 0.0 | 骑头判定的水平距离上限 | `ai_tank3.sp:116` |
| `ai_tank3_head_block_ride_rock_time` | Ai-Tank 3 | `3.0` | 整数，源码写作浮点 | ≥ 0.0 | 拳打不到的骑头目标持续多久后开始原地投石 | `ai_tank3.sp:119` |
| `ai_tank3_head_block_ride_vertical_max` | Ai-Tank 3 | `100.0` | 整数，源码写作浮点 | ≥ 0.0 | 骑头判定的最大垂直差，超过视为高台 | `ai_tank3.sp:118` |
| `ai_tank3_head_block_ride_vertical_min` | Ai-Tank 3 | `40.0` | 整数，源码写作浮点 | ≥ 0.0 | 骑头判定的最小垂直差 | `ai_tank3.sp:117` |
| `ai_tank3_head_block_time` | Ai-Tank 3 | `2.0` | 整数，源码写作浮点 | ≥ 0.0 | Tank 位于目标脚下的持续时间阈值秒数，可经梯子走到目标时按 3 倍计算 | `ai_tank3.sp:107` |
| `ai_tank3_head_block_up_swing` | Ai-Tank 3 | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 挥拳时是否对骑头生还者额外向上 SweepFist | `ai_tank3.sp:120` |
| `ai_tank3_head_block_vertical` | Ai-Tank 3 | `80.0` | 整数，源码写作浮点 | ≥ 0.0 | 触发头顶卡判定需要的垂直距离 | `ai_tank3.sp:108` |
| `ai_tank3_jump_rock` | Ai-Tank 3 | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 扔石头起手时是否允许跳砖 | `ai_tank3.sp:91` |
| `ai_tank3_ladder_look_lock` | Ai-Tank 3 | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | Tank 在梯子上时是否把视角锁到梯子朝向 | `ai_tank3.sp:124` |
| `ai_tank3_ladder_nearby_cache` | Ai-Tank 3 | `0.20` | 浮点 | ≥ 0.0 | 路径快照不可用时梯子实体检测的缓存秒数 | `ai_tank3.sp:127` |
| `ai_tank3_ladder_nearby_disable` | Ai-Tank 3 | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | Tank 即将爬梯时是否暂停连跳与挥拳锁视角并压回跑速 | `ai_tank3.sp:125` |
| `ai_tank3_ladder_nearby_radius` | Ai-Tank 3 | `180.0` | 整数，源码写作浮点 | ≥ 0.0 | 沿路径距离梯子入口多近算即将爬梯 | `ai_tank3.sp:126` |
| `ai_tank3_log_level` | Ai-Tank 3 | `32` | 整数 | 无上下界 | 日志级别：1 关，2 控制台，4 log，8 聊天，16 服务器控制台，32 error 文件，可相加 | `ai_tank3.sp:134` |
| `ai_tank3_path_lookahead_maxdepth` | Ai-Tank 3 | `10` | 整数，源码写作浮点 | ≥ 1.0 | 沿路径连跳时向前搜索 PathSegment 的最大深度，最小 1 | `ai_tank3.sp:75` |
| `ai_tank3_plugin_name` | Ai-Tank 3 | `ai_tank3` | 字符串或表达式 | 无上下界 | 插件名（非行为配置） | `ai_tank3.sp:130` |
| `ai_tank3_punch_lock_vision` | Ai-Tank 3 | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 挥拳时是否把视角锁定到目标 | `ai_tank3.sp:96` |
| `ai_tank3_retreat_timeout` | Ai-Tank 3 | `3.0` | 浮点 | ≥ 0.5 | 反头顶卡撤离投石时单次 MOVE 命令的最长持续秒数，到时必定 RESET | `ai_tank3.sp:121` |
| `ai_tank3_rock_target_adjust` | Ai-Tank 3 | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 出手时改为瞄准最近的可视生还者，强制投石计划指定的目标除外 | `ai_tank3.sp:90` |
| `ai_tank3_target_commit_time` | Ai-Tank 3 | `1.0` | 整数，源码写作浮点 | ≥ 0.0 | 两次主动换目标之间的最短间隔秒数 | `ai_tank3.sp:102` |
| `ai_tank3_target_decisive_ratio` | Ai-Tank 3 | `0.5` | 浮点 | 0.0 ~ 1.0 | 新目标得分不超过当前目标的该比例时视为明显更好打，0 关闭 | `ai_tank3.sp:103` |
| `ai_tank3_target_select` | Ai-Tank 3 | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | Tank 选目标方式：0 交给 l4d_target_override 或原生排序，1 沿用其过滤口径但按寻路距离排序并带换目标粘滞 | `ai_tank3.sp:99` |
| `ai_tank3_target_switch_gain` | Ai-Tank 3 | `75` | 整数，源码写作浮点 | ≥ 0.0 | 同一层换目标时新目标得分至少要比当前目标低这么多，按寻路距离单位 | `ai_tank3.sp:101` |
| `ai_tank3_target_switch_ratio` | Ai-Tank 3 | `0.85` | 浮点 | 0.1 ~ 1.0 | 同一层换目标时新目标得分须不超过当前目标的该比例 | `ai_tank3.sp:100` |
| `ai_tank3_throw_max_dist` | Ai-Tank 3 | `800` | 整数，源码写作浮点 | ≥ 0.0 | 允许扔石头的最大距离 | `ai_tank3.sp:89` |
| `ai_tank3_throw_min_dist` | Ai-Tank 3 | `0` | 整数，源码写作浮点 | ≥ 0.0 | 允许扔石头的最小距离 | `ai_tank3.sp:88` |
| `ai_tank_bhop` | Ai-Tank 3 | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否允许坦克连跳 | `ai_tank3.sp:66` |
| `ai_Tank_Bhop` | Ai_Tank_Enhance | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否开启 Tank 连跳：0 关，1 开 | `ai_tank_new.sp:96` |
| `ai_Tank_Bhop` | Ai_Tank_Enhance2.0 | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否开启坦克连跳 | `ai_tank_2.sp:123` |
| `ai_Tank_BhopSpeed` | Ai_Tank_Enhance | `60.0` | 整数，源码写作浮点 | ≥ 0.0 | Tank 连跳的速度 | `ai_tank_new.sp:94` |
| `ai_Tank_StopDistance` | Ai-Tank 3 | `135` | 整数，源码写作浮点 | ≥ 0.0 | 停止连跳的最小距离 | `ai_tank3.sp:67` |
| `ai_Tank_StopDistance` | Ai_Tank_Enhance | `130` | 整数，源码写作浮点 | ≥ 0.0 | Tank 距目标多远时停下来 | `ai_tank_new.sp:95` |
| `ai_Tank_StopDistance` | Ai_Tank_Enhance2.0 | `135` | 整数，源码写作浮点 | ≥ 0.0 | Tank 距目标多远时停止连跳 | `ai_tank_2.sp:125` |
| `ai_Tank_Throw` | Ai_Tank_Enhance | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否允许 Tank 投掷石块：0 关，1 开 | `ai_tank_new.sp:97` |
| `ai_TankAirAngleRestrict` | Ai_Tank_Enhance2.0 | `57` | 整数，源码写作浮点 | 0.0 ~ 90.0 | 坦克当前速度与到目标向量夹角大于该角度即停止连跳，范围 0~90 | `ai_tank_2.sp:139` |
| `ai_TankAirAngles` | Ai_Tank_Enhance | `60.0` | 整数，源码写作浮点 | 0.0 ~ 90.0 | 空中速度向量与到生还者方向向量夹角大于该值即停止连跳，范围 0~90 | `ai_tank_new.sp:103` |
| `ai_TankAntiTreeMethod` | Ai_Tank_Enhance2.0 | `1` | 整数，源码写作浮点 | 1.0 ~ 2.0 | 防止绕树的方法：1 选择新目标，2 传送到绕树生还者位置 | `ai_tank_2.sp:144` |
| `ai_TankAttackVomitedNum` | Ai_Tank_Enhance | `1` | 整数，源码写作浮点 | ≥ 0.0 | 若有该数量的生还者被 Boomer 喷吐到，正在消耗的坦克将发动攻击 | `ai_tank_new.sp:108` |
| `ai_TankBhopSpeed` | Ai_Tank_Enhance2.0 | `60` | 整数，源码写作浮点 | ≥ 0.0 | 坦克连跳速度 | `ai_tank_2.sp:124` |
| `ai_TankBlockThrowDistance` | Ai_Tank_Enhance | `200` | 整数，源码写作浮点 | ≥ 0.0 | Tank 距目标多近时阻止投掷石块 | `ai_tank_new.sp:99` |
| `ai_TankConsume` | Ai_Tank_Enhance | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否开启 Tank 消耗功能 | `ai_tank_new.sp:104` |
| `ai_TankConsume` | Ai_Tank_Enhance2.0 | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否开启坦克消耗 | `ai_tank_2.sp:127` |
| `ai_TankConsumeAction` | Ai_Tank_Enhance | `2` | 整数，源码写作浮点 | 1.0 ~ 2.0 | Tank 在消耗范围内的行为：1 冰冻，2 可活动但不允许超出消耗范围 | `ai_tank_new.sp:115` |
| `ai_TankConsumeDamagePercent` | Ai_Tank_Enhance | `50` | 整数，源码写作浮点 | 0.0 ~ 100.0 | Tank 在消耗过程中只受到该百分比的伤害，范围 0~100 | `ai_tank_new.sp:116` |
| `ai_TankConsumeDistance` | Ai_Tank_Enhance | `1200` | 整数，源码写作浮点 | ≥ 0.0 | Tank 消耗找位的位置必须离生还者大于该距离 | `ai_tank_new.sp:122` |
| `ai_TankConsumeDistance` | Ai_Tank_Enhance2.0 | `1200` | 整数，源码写作浮点 | ≥ 0.0 | 射线找到的消耗位需要离生还者这么远 | `ai_tank_2.sp:131` |
| `ai_TankConsumeHealth` | Ai_Tank_Enhance2.0 | `2000` | 整数，源码写作浮点 | ≥ 0.0 | 坦克血量少于该值时强制压制 | `ai_tank_2.sp:136` |
| `ai_TankConsumeHealthLimit` | Ai_Tank_Enhance | `1200` | 整数，源码写作浮点 | ≥ 0.0 | Tank 血量少于该值时不会消耗 | `ai_tank_new.sp:120` |
| `ai_TankConsumeHeight` | Ai_Tank_Enhance | `100` | 整数，源码写作浮点 | ≥ 0.0 | 消耗时优先选择高于该高度的位置，无则随机选位 | `ai_tank_new.sp:105` |
| `ai_TankConsumeIncapNum` | Ai_Tank_Enhance2.0 | `1` | 整数，源码写作浮点 | ≥ 0.0 | 坦克强制压制时若令该数量的生还者倒地则允许时继续消耗 | `ai_tank_2.sp:138` |
| `ai_TankConsumeInfSub` | Ai_Tank_Enhance2.0 | `1` | 整数，源码写作浮点 | ≥ 0.0 | 当前特感数量小于等于特感上限减去该值时坦克可以消耗 | `ai_tank_2.sp:129` |
| `ai_TankConsumeLimit` | Ai_Tank_Enhance | `0` | 整数，源码写作浮点 | ≥ 0.0 | 感染者团队的特感数小于等于当前刷特数量减去该值时触发，源码描述不完整，未说明具体动作 | `ai_tank_new.sp:106` |
| `ai_TankConsumeLimitNum` | Ai_Tank_Enhance | `5` | 整数，源码写作浮点 | ≥ 0.0 | Tank 最多进行消耗的次数 | `ai_tank_new.sp:112` |
| `ai_TankConsumePosRaidus` | Ai_Tank_Enhance2.0 | `100` | 整数，源码写作浮点 | ≥ 0.0 | 坦克走出消耗位中心该半径的圆范围后强制重新进入 | `ai_tank_2.sp:135` |
| `ai_TankConsumeRaidus` | Ai_Tank_Enhance | `80.0` | 整数，源码写作浮点 | ≥ 0.0 | Tank 消耗位置的范围半径，以中心坐标画圆 | `ai_tank_new.sp:107` |
| `ai_TankConsumeRayRaidus` | Ai_Tank_Enhance2.0 | `1500` | 整数，源码写作浮点 | ≥ 0.0 | 射线寻找消耗位的范围，从坦克当前位置开始计算 | `ai_tank_2.sp:130` |
| `ai_TankConsumeRockInterval` | Ai_Tank_Enhance2.0 | `4` | 整数，源码写作浮点 | ≥ 0.0 | 坦克在消耗位上每多少秒扔一次石头 | `ai_tank_2.sp:140` |
| `ai_TankConsumeType` | Ai_Tank_Enhance | `6` | 整数，源码写作浮点 | 1.0 ~ 8.0 | Tank 按哪种特感类型找消耗位：1 Smoker，2 Boomer，3 Hunter，4 Spitter，5 Jockey，6 Charger，8 Tank | `ai_tank_new.sp:113` |
| `ai_TankConsumeValidRaidus` | Ai_Tank_Enhance | `1800` | 整数，源码写作浮点 | ≥ 0.0 | 当前消耗位不能直视生还时以该半径重新找位 | `ai_tank_new.sp:121` |
| `ai_TankFindNewConsumePosDistance` | Ai_Tank_Enhance2.0 | `750` | 整数，源码写作浮点 | ≥ 0.0 | 最近的生还者离坦克这么远时坦克重新找消耗位 | `ai_tank_2.sp:132` |
| `ai_TankForceAttackDist` | Ai_Tank_Enhance2.0 | `350` | 整数，源码写作浮点 | ≥ 0.0 | 生还者距离坦克这么近时坦克强制攻击 | `ai_tank_2.sp:133` |
| `ai_TankForceAttackDistance` | Ai_Tank_Enhance | `300` | 整数，源码写作浮点 | ≥ 0.0 | Tank 离最近生还者该距离时即使可消耗也会强制压制 | `ai_tank_new.sp:118` |
| `ai_TankForceAttackProgress` | Ai_Tank_Enhance2.0 | `10` | 整数，源码写作浮点 | ≥ 0.0 | 开始消耗时记录生还者路程，超过该值加上这个数后不允许消耗 | `ai_tank_2.sp:134` |
| `ai_TankIncappedCount` | Ai_Tank_Enhance | `1` | 整数，源码写作浮点 | ≥ 0.0 | 强制压制时需拍倒该数量的生还者才允许继续检测消耗 | `ai_tank_new.sp:119` |
| `ai_TankRetreatAirAngles` | Ai_Tank_Enhance | `75.0` | 整数，源码写作浮点 | ≥ 0.0 | Tank 在回避连跳过程中视角与速度夹角超过该值即停止连跳 | `ai_tank_new.sp:114` |
| `ai_TankSequencePlayBackRate` | Advance Special Infected AI | `3.0` | 整数，源码写作浮点 | ≥ 0.0 | 坦克攀爬动画加速速率 | `AI_HardSI_2.sp:80` |
| `ai_TankSequencePlayBackRate` | Advance Special Infected AI | `3.0` | 整数，源码写作浮点 | ≥ 0.0 | 坦克攀爬动画加速速率（同名 ConVar，来自 Ai Tank 增强变体） | `AI_HardSI_new.sp:81` |
| `ai_TankSneakTime` | Ai_Tank_Enhance2.0 | `0` | 整数，源码写作浮点 | 0.0 ~ 28.0 | tank 会消耗到下一波生成时间小于该值，0 为关闭 | `ai_tank_2.sp:128` |
| `ai_TankTarget` | Ai_Tank_Enhance | `1` | 整数，源码写作浮点 | 1.0 ~ 3.0 | Tank 目标选择：1 最近，2 血量最少，3 血量最多 | `ai_tank_new.sp:100` |
| `ai_TankTarget` | Ai_Tank_Enhance2.0 | `0` | 整数，源码写作浮点 | 0.0 ~ 3.0 | 坦克目标选择：0 自然选择，1 最近，2 血量最低，3 血量最高 | `ai_tank_2.sp:142` |
| `ai_TankTeleportForwardPercent` | Ai_Tank_Enhance | `10` | 整数，源码写作浮点 | ≥ 0.0 | 开始消耗时记录生还者行进距离 x，前压超过 x 加该值时 Tank 传送到生还者处压制 | `ai_tank_new.sp:111` |
| `ai_TankThow` | Ai_Tank_Enhance2.0 | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否允许坦克丢石头 | `ai_tank_2.sp:146` |
| `ai_TankThrowDistance` | Ai_Tank_Enhance | `450` | 整数，源码写作浮点 | ≥ 0.0 | Tank 距目标多近允许投掷石块 | `ai_tank_new.sp:98` |
| `ai_TankThrowRange` | Ai_Tank_Enhance2.0 | `250,500` | 字符串或表达式 | 无上下界 | 允许坦克丢石头的范围，逗号分隔且不能有空格 | `ai_tank_2.sp:147` |
| `ai_TankTreeDetect` | Ai_Tank_Enhance | `1` | 整数，源码写作浮点 | 0.0 ~ 2.0 | 生还者与 Tank 绕柱时的操作：0 关闭，1 切换目标，2 把 Tank 传送到绕树的生还者后 | `ai_tank_new.sp:101` |
| `ai_TankTreeDetect` | Ai_Tank_Enhance2.0 | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否开启防止绕树功能 | `ai_tank_2.sp:143` |
| `ai_TankTreeNewTargetDistance` | Ai_Tank_Enhance | `300` | 整数，源码写作浮点 | ≥ 0.0 | 记录绕树生还者并选择新目标后，距新目标多近重置绕树记录 | `ai_tank_new.sp:102` |
| `ai_TankVomitAttackInterval` | Ai_Tank_Enhance | `20.0` | 整数，源码写作浮点 | ≥ 0.0 | 从开始被喷且 Tank 允许攻击时起，该时间内 Tank 允许攻击 | `ai_tank_new.sp:110` |
| `ai_TankVomitAttackNum` | Ai_Tank_Enhance2.0 | `1` | 整数，源码写作浮点 | ≥ 0.0 | 有该数量的生还者被喷吐时正在消耗的坦克强制压制 | `ai_tank_2.sp:137` |
| `ai_TankVomitCanInstantAttack` | Ai_Tank_Enhance | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否开启固定数量生还者被喷吐后 Tank 立刻攻击 | `ai_tank_new.sp:109` |
| `ai_wall_detection_distance` | AI HUNTER | `-1.0` | 整数，源码写作浮点 | 无上下界 | 感染者 Bot 在前方多远检测墙壁，填 -1 关闭该功能 | `ai_hunter_new.sp:53` |
| `alonemode` | L4D2 Smoker Drag Damage Interval | `0` | 整数 | 无上下界 | 是否处于 alonemode | `l4d2_smoker_drag_damage_interval_zone.sp:48` |
| `anne_cvar_shield_debug` | anne_cvar_shield.sp（myinfo 缺 name，用文件名代替） | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否记录 Anne CVar Shield 的操作日志 | `anne_cvar_shield.sp:122` |
| `anne_cvar_shield_enable` | anne_cvar_shield.sp（myinfo 缺 name，用文件名代替） | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用 Anne 特感上限 cvar 保护 | `anne_cvar_shield.sp:117` |
| `anne_cvar_shield_sync_versus_limits` | anne_cvar_shield.sp（myinfo 缺 name，用文件名代替） | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否为每个特感职业同步 z_*_limit、z_versus_*_limit 与 infected_control 的 inf_*_limit 目标 | `anne_cvar_shield.sp:127` |
| `anne_db_charset` | Anne DB Connection Hub | `utf8mb4` | 字符串或表达式 | 无上下界 | 共享连接建立后统一设置的字符集，留空不设置 | `anne_db.sp:87` |
| `anne_db_keepalive` | Anne DB Connection Hub | `120` | 整数，源码写作浮点 | ≥ 0.0 | 共享连接保活间隔秒数，须小于 MySQL wait_timeout，0 关闭 | `anne_db.sp:88` |
| `anne_db_request_timeout` | Anne DB Connection Hub | `30` | 整数，源码写作浮点 | ≥ 1.0 | 异步连接请求最长等待秒数，超时后回调失败 | `anne_db.sp:89` |
| `anne_db_sync_cooldown` | Anne DB Connection Hub | `30` | 整数，源码写作浮点 | ≥ 0.0 | 连接失败后多少秒内同步获取直接返回失败，避免主线程反复阻塞 | `anne_db.sp:90` |
| `anne_db_version` | Anne DB Connection Hub | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | `anne_db.sp:91` |
| `anne_heal50_on_transition` | AnneServer Server Function (quiet minimal) | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 通关或切图时若生还者实血小于 50 则补至 50 并重置倒地次数，仅在 anne_reset_on_transition 为 0 时生效 | `server.sp:132` |
| `anne_reset_on_transition` | AnneServer Server Function (quiet minimal) | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 通关或切图时是否执行 RestoreHealth 与 ResetInventory：0 否，1 是 | `server.sp:128` |
| `anne_round_wipe_count` | AnneServer Server Function (quiet minimal) | `0` | 整数，源码写作浮点 | ≥ 0.0 | 当前地图已发生的团灭次数，只读状态 | `server.sp:138` |
| `anne_spawn_warp_to_start` | AnneServer Server Function (quiet minimal) | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 生还者 player_spawn 后是否在未离开安全区前传送到起始点：0 否，1 是 | `server.sp:136` |
| `AnnePluginVersion` | text.sp（myinfo 缺 name，用文件名代替） | `Latest` | 字符串或表达式 | 无上下界 | Anne 插件版本 | `text.sp:42` |
| `attachments_api_check` | [ANY] Attachments API | `0.1` | 浮点 | 无上下界 | 检查玩家模型是否变化的间隔，需 attachments_api_models 为 1 才启用 | `attachments_api.sp:188` |
| `attachments_api_equip` | [ANY] Attachments API | `0.1` | 浮点 | 无上下界 | 玩家模型变化时，为修复挂件先丢弃武器再重新装备的延迟秒数 | `attachments_api.sp:189` |
| `attachments_api_models` | [ANY] Attachments API | `sChk` | 字符串或表达式 | 无上下界 | 0 关闭；1 检测玩家模型变化以修复玩家身上的挂件 | `attachments_api.sp:190` |
| `attachments_api_version` | [ANY] Attachments API | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | `attachments_api.sp:192` |
| `attachments_api_weapons` | [ANY] Attachments API | `sChk` | 字符串或表达式 | 无上下界 | 0 关闭；1 检测武器模型变化以修复武器挂件 | `attachments_api.sp:191` |
| `autopause_apdebug` | L4D2 Auto-pause | `0` | 整数，源码写作浮点 | 0.0 ~ 3.0 | 调试等级：0 不调试，1 写 SourceMod 日志，2 聊天输出，3 两者 | `autopause.sp:67` |
| `autopause_enable` | L4D2 Auto-pause | `1` | 整数 | 无上下界 | 玩家崩溃时是否自动暂停 | `autopause.sp:64` |
| `autopause_force` | L4D2 Auto-pause | `0` | 整数 | 无上下界 | 玩家崩溃时是否强制暂停 | `autopause.sp:65` |
| `autopause_forceunpause` | L4D2 Auto-pause | `0` | 整数 | 无上下界 | 崩溃玩家重连后是否强制取消暂停 | `autopause.sp:66` |
| `bhop_allow_survivor` | Simple Anti-Bunnyhop | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否允许生还者连跳：1 允许，0 禁止 | `l4d2_nobhaps.sp:72` |
| `bhop_allow_survivor` | Simple Anti-Bunnyhop | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否允许生还者连跳：1 允许，0 禁止 | `l4d2_nobhaps.sp:72` |
| `bhop_except_si_flags` | Simple Anti-Bunnyhop | `0` | 整数，源码写作浮点 | 0.0 ~ 127.0 | 豁免禁跳的特感位掩码：1 smoker，2 boomer，4 hunter，8 spitter，16 jockey，32 charger，64 tank | `l4d2_nobhaps.sp:64` |
| `bhop_except_si_flags` | Simple Anti-Bunnyhop | `0` | 整数，源码写作浮点 | 0.0 ~ 127.0 | 豁免禁跳的特感位掩码：1 smoker，2 boomer，4 hunter，8 spitter，16 jockey，其余被截断 | `l4d2_nobhaps.sp:64` |
| `boomer_horde_equalizer` | Boomer Horde Equalizer | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否修复 Boomer 尸潮规模因游荡普感而不同的问题：1 开，0 关 | `boomer_horde_equalizer.sp:31` |
| `boomer_horde_equalizer` | Boomer Horde Equalizer (Refactored) | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 同 1117，重构版插件中的同名 ConVar | `boomer_horde_equalizer_refactored.sp:87` |
| `boomer_horde_equalizer_events_default` | Boomer Horde Equalizer (Refactored) | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 活动尸潮期间是否使用默认 Boomer 行为：1 是，0 覆盖 | `boomer_horde_equalizer_refactored.sp:88` |
| `bot_kick_delay` | L4D2 No Second Chances | `0` | 整数，源码写作浮点 | 0.0 ~ 30.0 | 踢出特感 Bot 前等待的秒数，范围 0~30 | `l4d2_nosecondchances.sp:54` |
| `bq_cvar_change_suppress` | AnneServer Server Function (quiet minimal) | `1` | 整数 | 无上下界 | 是否屏蔽服务器 cvar 变更提示，使聊天更干净 | `server.sp:155` |
| `bq_cvar_change_suppress` | BeQuiet | `1` | 整数 | 无上下界 | 是否屏蔽服务器 cvar 变更提示，使聊天更干净 | `bequiet.sp:30` |
| `bq_name_change_spec_suppress` | AnneServer Server Function (quiet minimal) | `1` | 整数 | 无上下界 | 是否屏蔽旁观玩家改名提示 | `server.sp:157` |
| `bq_name_change_spec_suppress` | BeQuiet | `1` | 整数 | 无上下界 | 是否屏蔽旁观玩家改名的提示 | `bequiet.sp:32` |
| `bq_name_change_suppress` | AnneServer Server Function (quiet minimal) | `1` | 整数 | 无上下界 | 是否屏蔽玩家改名提示 | `server.sp:156` |
| `bq_name_change_suppress` | BeQuiet | `1` | 整数 | 无上下界 | 是否屏蔽玩家改名提示 | `bequiet.sp:31` |
| `bq_show_player_team_chat_spec` | AnneServer Server Function (quiet minimal) | `1` | 整数 | 无上下界 | 是否向旁观者显示生还者与特感的团队聊天 | `server.sp:158` |
| `bq_show_player_team_chat_spec` | BeQuiet | `1` | 整数 | 无上下界 | 是否向旁观者显示生还者与特感的团队聊天 | `bequiet.sp:33` |
| `caster_disable_addons` | L4D2 Caster System (Original built in readyup) | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否禁止解说员使用插件 | `caster_system.sp:54` |
| `charger_collision_patch_version` | [L4D2]Charger_Collision_Patch | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | `Charger_Collision_patch.sp:56` |
| `charger_dmg_incapped` | Incapped Charger Damage | `-1.0` | 整数，源码写作浮点 | 无上下界 | 对倒地生还者的冲锋伤害 | `charger_incap_damage.sp:39` |
| `charger_keep_far_charge_animation` | [L4D2] Merged Get-Up Fixes | `0` | 整数 | 无上下界 | 是否启用长距离远抛撞地动画及其神圣帧 | `l4d2_getup_fixes.sp:159` |
| `charger_keep_wall_charge_animation` | [L4D2] Merged Get-Up Fixes | `1` | 整数 | 无上下界 | 是否启用长距离撞墙动画及其神圣帧 | `l4d2_getup_fixes.sp:158` |
| `charger_knockdown_getup_window` | [L4D2] Charger Target Fix | `0.1` | 浮点 | 0.0 ~ 4.0 | 击倒计时结束到起身完成之间的时长，值越大起身时越早变回可碰撞 | `l4d2_charge_target_fix.sp:96` |
| `cl_consistencycheck_interval` | sv_consistency fixes | `180.0` | 整数，源码写作浮点 | 无上下界 | 距上次一致性检查多少秒后再次执行检查 | `sv_consistency_fix.sp:41` |
| `coinflip_delay` | Coinflip | `-1` | 整数 | 无上下界 | 两次允许投硬币之间的延迟秒数，-1 表示无延迟 | `coinflip.sp:43` |
| `collision_smoker_common` | L4D2 Collision Adjustments | `0` | 整数 | 无上下界 | 被拉的生还者是否会穿过普通感染者 | `l4d2_collision_adjustments.sp:43` |
| `collision_tankrock_common` | L4D2 Collision Adjustments | `1` | 整数 | 无上下界 | 石头是否会穿过普通感染者并击杀它们，而不是被卡住 | `l4d2_collision_adjustments.sp:42` |
| `collision_tankrock_incap` | L4D2 Collision Adjustments | `0` | 整数 | 无上下界 | 石头是否会穿过倒地的生还者 | `l4d2_collision_adjustments.sp:44` |
| `command_buffer_version` | [ANY] Command and ConVar - Buffer Overflow Fixer | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | `command_buffer.sp:116` |
| `common_hits` | SI - CI FF Block | `5` | 整数 | 无上下界 | 特感击杀普通感染者所需命中次数，0 屏蔽友军伤害，5 为 L4D1 风格 | `l4d_ci_ffblock.sp:35` |
| `confogl_block_punch_rock` | Confogl's Competitive Mod | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否禁止坦克同时拳击与投掷石头 | `GhostTank.sp:47` |
| `confogl_boss_unprohibit` | Confogl's Competitive Mod | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否允许 boss 在所有地图刷新，即使该地图原本不允许 | `UnprohibitBosses.sp:16` |
| `confogl_customcfg` | Confogl's Competitive Mod | `` | 字符串或表达式 | 无上下界 | 内部使用，源码标注请勿修改 | `configs.sp:32` |
| `confogl_debug` | Confogl's Competitive Mod | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否开启所有 confogl 模块的调试日志 | `debug.sp:15` |
| `confogl_disable_tank_hordes` | Confogl's Competitive Mod | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 坦克在场时是否禁止自然尸潮 | `GhostTank.sp:46` |
| `confogl_enable_itemtracking` | Confogl's Competitive Mod | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用物品跟踪模块 | `ItemTracking.sp:164` |
| `confogl_ghost_warp` | Confogl's Competitive Mod | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 幽灵特感能否按右键传送到下一个生还者 | `GhostWarp.sp:23` |
| `confogl_ghost_warp_reload` | Confogl's Competitive Mod | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 幽灵传送使用鼠标右键还是换弹键 | `GhostWarp.sp:24` |
| `confogl_itemtracking_mapspecific` | Confogl's Competitive Mod | `0` | 开关 0 或 1 | 0.0 ~ 3.0 | mapinfo.txt 覆盖方式：0 忽略该文件，1 允许减少上限，2 允许提高上限 | `ItemTracking.sp:166` |
| `confogl_itemtracking_playeritems` | Confogl's Competitive Mod | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否忽略玩家出生自带的物品，非对战模式无影响 | `ItemTracking.sp:167` |
| `confogl_itemtracking_savespawns` | Confogl's Competitive Mod | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否让两回合的物品刷新保持一致 | `ItemTracking.sp:165` |
| `confogl_limit_sniper` | Confogl's Competitive Mod | `1` | 开关 0 或 1 | 0.0 ~ 4.0 | 同时最多允许的狙击步枪数量 | `WeaponCustomization.sp:30` |
| `confogl_limit_tier2` | Confogl's Competitive Mod | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否限制安全室外二级武器的数量，首次拾取时把二级武器堆替换为一级 | `WeaponInformation.sp:604` |
| `confogl_limit_tier2_saferoom` | Confogl's Competitive Mod | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否限制安全室内二级武器的数量，首次拾取时把二级武器堆替换为一级 | `WeaponInformation.sp:605` |
| `confogl_lock_boss_spawns` | Confogl's Competitive Mod | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 强制 Tank 与 Witch 在相同坐标刷新 | `BossSpawning.sp:36` |
| `confogl_match_autoconfig` | Confogl's Competitive Mod | `` | 字符串或表达式 | 无上下界 | 自动加载启用时加载哪个配置 | `ReqMatch.sp:55` |
| `confogl_match_autoload` | Confogl's Competitive Mod | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 玩家连接且服务器不在比赛模式时是否自动进入比赛模式 | `ReqMatch.sp:54` |
| `confogl_match_execcfg_off` | Confogl's Competitive Mod | `confogl_off.cfg` | 字符串或表达式 | 无上下界 | 比赛模式结束时执行的 cfg 文件 | `ReqMatch.sp:59` |
| `confogl_match_execcfg_on` | Confogl's Competitive Mod | `confogl.cfg` | 字符串或表达式 | 无上下界 | 比赛模式开始及之后每张图要执行的 cfg 文件 | `ReqMatch.sp:56` |
| `confogl_match_execcfg_plugins` | Confogl's Competitive Mod | `generalfixes.cfg;confogl_plugins.cfg;sharedplugins.cfg` | 字符串或表达式 | 无上下界 | 比赛模式开始时执行的插件 cfg，仅执行一次 | `ReqMatch.sp:58` |
| `confogl_match_killlobbyres` | Confogl's Competitive Mod | `1` | 开关 0 或 1 | 0.0 ~ 1.0 \ | 比赛开始后是否清除大厅预留 | `UnreserveLobby.sp:13` |
| `confogl_match_map` | Confogl's Competitive Mod | `` | 字符串或表达式 | 无上下界 | 内部使用，源码标注请勿修改，用于保存将要切换到的地图 | `ReqMatch.sp:81` |
| `confogl_match_reloaded` | Confogl's Competitive Mod | `0` | 开关 int 0 或 1 | 无上下界 | 内部使用，源码标注请勿修改，用于防止比赛模式反复循环 | `ReqMatch.sp:75` |
| `confogl_match_restart` | Confogl's Competitive Mod | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 强制或请求比赛模式时是否重开地图 | `ReqMatch.sp:52` |
| `confogl_match_untracked_cvar_reset_debug` | Confogl's Competitive Mod | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否记录 resetmatch 期间未跟踪 cvar 的恢复细节 | `ReqMatch.sp:61` |
| `confogl_match_untracked_cvar_reset_file` | Confogl's Competitive Mod | `RM_UNTRACKED_CVAR_RESET_PATH` | 字符串或表达式 | 无上下界 | addons/sourcemod 下列出在 resetmatch 时恢复的未跟踪 cvar 的文件 | `ReqMatch.sp:60` |
| `confogl_pills_flow_fill` | Confogl's Competitive Mod | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否在主线随机流程位置补齐缺失的止痛药，0 关闭 | `ItemTracking.sp:177` |
| `confogl_pills_flow_fill_max` | Confogl's Competitive Mod | `1.0` | 开关 0 或 1 | 0.0 ~ 1.0 | 随机补齐的止痛药所允许的最大地图流程比例，安全室区域被排除 | `ItemTracking.sp:179` |
| `confogl_pills_flow_fill_min` | Confogl's Competitive Mod | `0.3` | 浮点 | 0.0 ~ 1.0 | 随机补齐的止痛药所允许的最小地图流程比例 | `ItemTracking.sp:178` |
| `confogl_pills_flow_finale` | Confogl's Competitive Mod | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否在终局地图同样应用止痛药流程窗口，0 表示终局豁免 | `ItemTracking.sp:175` |
| `confogl_pills_flow_max` | Confogl's Competitive Mod | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 止痛药可能刷出的最大地图流程比例，晚于该值的会被移除 | `ItemTracking.sp:173` |
| `confogl_pills_flow_max_detour` | Confogl's Competitive Mod | `0` | 开关 0 或 1 | ≥ 0.0 | 止痛药刷新点偏离最短路线允许的最大绕行距离，单位为导航单位，0 关闭；偏离路线的会被移除 | `ItemTracking.sp:176` |
| `confogl_pills_flow_min` | Confogl's Competitive Mod | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 止痛药可能刷出的最小地图流程比例，早于该值的会被移除；0 且 max 为 1、separation 为 0 时关闭流程过滤 | `ItemTracking.sp:172` |
| `confogl_pills_flow_separation` | Confogl's Competitive Mod | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 两个保留的止痛药刷新点之间的最小流程比例间隔，0 表示不强制间隔，距离过近时保留较早的那个 | `ItemTracking.sp:174` |
| `confogl_pills_flow_visualize` | Confogl's Competitive Mod | `0` | 开关 0 或 1 | 0.0 ~ 2.0 | 调试用：1 不移除止痛药而是发光标注，2 正常移除后把保留的发光 | `ItemTracking.sp:180` |
| `confogl_reduce_finalespawnrange` | Confogl's Competitive Mod | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否把终局特感刷新范围调整为普通刷新范围 | `FinaleSpawn.sp:19` |
| `confogl_remove_chainsaw` | Confogl's Competitive Mod | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否移除所有电锯 | `WeaponInformation.sp:584` |
| `confogl_remove_defib` | Confogl's Competitive Mod | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否移除所有除颤器 | `WeaponInformation.sp:588` |
| `confogl_remove_escape_tank` | Confogl's Competitive Mod | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 终局救援载具到来时刷出的 Tank 是否移除 | `GhostTank.sp:45` |
| `confogl_remove_grenade` | Confogl's Competitive Mod | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否移除所有榴弹发射器 | `WeaponInformation.sp:583` |
| `confogl_remove_lasersight` | Confogl's Competitive Mod | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否移除所有激光瞄准升级 | `WeaponInformation.sp:608` |
| `confogl_remove_m60` | Confogl's Competitive Mod | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否移除所有 M60 | `WeaponInformation.sp:585` |
| `confogl_remove_parachutist` | Confogl's Competitive Mod | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否移除 c3m2 的跳伞员 | `EntityRemover.sp:37` |
| `confogl_remove_saferoomitems` | Confogl's Competitive Mod | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否移除安全室内除医疗包以外的额外物品 | `WeaponInformation.sp:609` |
| `confogl_remove_statickits` | Confogl's Competitive Mod | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否移除地图内置的静态医疗包 | `WeaponInformation.sp:587` |
| `confogl_remove_upg_explosive` | Confogl's Competitive Mod | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否移除所有高爆弹药包 | `WeaponInformation.sp:589` |
| `confogl_remove_upg_incendiary` | Confogl's Competitive Mod | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否移除所有燃烧弹药包 | `WeaponInformation.sp:590` |
| `confogl_replace_cssweapons` | Confogl's Competitive Mod | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否把 CSS 武器替换为普通 L4D2 武器 | `WeaponInformation.sp:577` |
| `confogl_replace_finalekits` | Confogl's Competitive Mod | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否把终局的医疗包替换为止痛药 | `WeaponInformation.sp:607` |
| `confogl_replace_startkits` | Confogl's Competitive Mod | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否把起点的医疗包替换为止痛药 | `WeaponInformation.sp:606` |
| `confogl_replace_tier2` | Confogl's Competitive Mod | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否把起点与终点安全室的二级武器替换为一级武器 | `WeaponInformation.sp:601` |
| `confogl_replace_tier2_all` | Confogl's Competitive Mod | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否在任何位置把所有二级武器替换为一级武器 | `WeaponInformation.sp:603` |
| `confogl_replace_tier2_finale` | Confogl's Competitive Mod | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 终局时是否把起点安全室的二级武器替换为一级武器 | `WeaponInformation.sp:602` |
| `confogl_slowdown_factor` | Confogl's Competitive Mod | `0.90` | 浮点 | 无上下界 | 水对生还者的减速程度，1.00 为原生值 | `WaterSlowdown.sp:23` |
| `confogl_SM_custommaxdistance` | Confogl's Competitive Mod | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否使用配置中的自定义最大距离 | `ScoreMod.sp:66` |
| `confogl_SM_enable` | Confogl's Competitive Mod | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | L4D2 自定义计分系统开关 | `ScoreMod.sp:59` |
| `confogl_SM_healthbonusratio` | Confogl's Competitive Mod | `2.0` | 浮点 | 0.25 ~ 5.0 | 生命奖励倍率，范围 0.25~5 | `ScoreMod.sp:60` |
| `confogl_SM_mapmulti` | Confogl's Competitive Mod | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否把生命奖励上限提升到距离上限 | `ScoreMod.sp:65` |
| `confogl_SM_survivalbonusratio` | Confogl's Competitive Mod | `0.0` | 开关 0 或 1 | 无上下界 | 按地图距离计算固定生存奖励的比例 | `ScoreMod.sp:61` |
| `confogl_SM_tempmulti_incap_0` | Confogl's Competitive Mod | `0.30625` | 浮点 | 0.0 ~ 1.0 | 未有倒地记录的生还者其临时生命的重要程度 | `ScoreMod.sp:62` |
| `confogl_SM_tempmulti_incap_1` | Confogl's Competitive Mod | `0.17500` | 浮点 | 0.0 ~ 1.0 | 倒地一次的生还者其临时生命的重要程度 | `ScoreMod.sp:63` |
| `confogl_SM_tempmulti_incap_2` | Confogl's Competitive Mod | `0.10000` | 浮点 | 0.0 ~ 1.0 | 倒地两次即黑白的生还者其临时生命的重要程度 | `ScoreMod.sp:64` |
| `confogl_waterslowdown` | Confogl's Competitive Mod | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用额外的水中减速 | `WaterSlowdown.sp:22` |
| `coop_round_restart_delay` | coop_round_delay.sp（myinfo 缺 name，用文件名代替） | `2.0` | 整数，源码写作浮点 | ≥ 0.0 | 战役模式回合重开延迟时间，最小 0 | `coop_round_delay.sp:31` |
| `coop_round_restart_delay_version` | coop_round_delay.sp（myinfo 缺 name，用文件名代替） | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | `coop_round_delay.sp:29` |
| `coopmode` | text.sp（myinfo 缺 name，用文件名代替） | `0` | 整数 | 无上下界 | 合作模式标记；据 `optional/AnneHappy/text.sp:59` 的 `CreateConVar("coopmode", "0")`（连描述参数都没有），全文唯一读取处在 :99-105 `Incap_Event`（钩子为 `player_incapacitated_start`/`player_incapacitated`，:60-61）：`if(GetConVarBool(g_hCvarCoop)) ForcePlayerSuicide(Incap);`，即被击倒的幸存者立即被处死，随后 :106-108 再按 `IsTeamImmobilised()` 决定是否全队处死，推测为：标记本局是否为合作(campaign)模式；1=倒地即死（跳过倒地挣扎/被救流程），0（默认，对抗）=保持原版倒地逻辑（置信度：中） | `text.sp:59` |
| `crc_debug` | Checkpoint Rage Control | `0` | 整数，源码写作浮点 | 0.0 ~ 3.0 | 调试等级：0 关闭，1 启用，2 仅聊天，3 仅控制台 | `checkpoint-rage-control.sp:71` |
| `crc_global` | Checkpoint Rage Control | `0` | 整数 | 无上下界 | 是否默认移除所有地图的安全区挫败感保留机制 | `checkpoint-rage-control.sp:70` |
| `cssladders_allow_m2` | Ladder Rambos Dhooks [Merged] | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否允许在梯子上推击：1 允许，0 禁止 | `l4d2_ladder_rambos.sp:119` |
| `cssladders_allow_reload` | Ladder Rambos Dhooks [Merged] | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否允许在梯子上换弹：1 允许，0 禁止 | `l4d2_ladder_rambos.sp:126` |
| `cssladders_allow_shotgun_reload` | Ladder Rambos Dhooks [Merged] | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否允许在梯子上给霰弹枪换弹：1 允许，0 禁止 | `l4d2_ladder_rambos.sp:133` |
| `cssladders_allow_switch` | Ladder Rambos Dhooks [Merged] | `1` | 整数，源码写作浮点 | 0.0 ~ 2.0 | 是否允许在梯子上切换物品：2 全部允许，1 仅枪械间切换，0 禁止 | `l4d2_ladder_rambos.sp:140` |
| `cssladders_enabled` | Ladder Rambos Dhooks [Merged] | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 生还者能否在梯子上射击：1 允许，0 禁止 | `l4d2_ladder_rambos.sp:112` |
| `cssladders_reduce_recoil` | Ladder Rambos Dhooks [Merged] | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否允许在梯子上射击时降低后坐力：1 允许，0 禁止 | `l4d2_ladder_rambos.sp:147` |
| `deathcam_skip_announce` | Death Cam Skip Fix | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 有人利用漏洞时是否打印提示 | `nodeathcamskip.sp:31` |
| `defib_fix_version` | [L4D2]Defib_Fix | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | `Defib_Fix.sp:98` |
| `dirspawn_active_challenge` | l4d2_dirspawn.sp（myinfo 缺 name，用文件名代替） | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否设置 ActiveChallenge、Aggressive、Assault 标志：0 否，1 是 | `l4d2_dirspawn.sp:1073` |
| `dirspawn_allow_si_with_tank` | l4d2_dirspawn.sp（myinfo 缺 name，用文件名代替） | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 坦克在场时是否允许刷特：0 禁刷，1 允许并存 | `l4d2_dirspawn.sp:1079` |
| `dirspawn_apply_delay` | l4d2_dirspawn.sp（myinfo 缺 name，用文件名代替） | `1.0` | 浮点 | 0.1 ~ 10.0 | 回合开始首次应用的延迟秒数，范围 0.1~10 | `l4d2_dirspawn.sp:1067` |
| `dirspawn_apply_on_roundstart` | l4d2_dirspawn.sp（myinfo 缺 name，用文件名代替） | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 回合开始是否自动应用：0 否，1 是 | `l4d2_dirspawn.sp:1066` |
| `dirspawn_auto_announce` | l4d2_dirspawn.sp（myinfo 缺 name，用文件名代替） | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 人数自适应变更时是否公告：0 否，1 是 | `l4d2_dirspawn.sp:1117` |
| `dirspawn_auto_base_count` | l4d2_dirspawn.sp（myinfo 缺 name，用文件名代替） | `6` | 整数，源码写作浮点 | 0.0 ~ 30.0 | 4 名真人时的基线总特数 | `l4d2_dirspawn.sp:1110` |
| `dirspawn_auto_base_interval` | l4d2_dirspawn.sp（myinfo 缺 name，用文件名代替） | `35` | 整数，源码写作浮点 | 0.0 ~ 120.0 | 4 名真人时的基线刷特间隔 | `l4d2_dirspawn.sp:1114` |
| `dirspawn_auto_count_mode` | l4d2_dirspawn.sp（myinfo 缺 name，用文件名代替） | `1` | 整数，源码写作浮点 | 0.0 ~ 2.0 | 计数模式：0 全部真人，1 仅生还，2 生还加感染但不含观察者 | `l4d2_dirspawn.sp:1109` |
| `dirspawn_auto_enable` | l4d2_dirspawn.sp（myinfo 缺 name，用文件名代替） | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用人数自适应，仅调整总特数量：0 否，1 是 | `l4d2_dirspawn.sp:1108` |
| `dirspawn_auto_max_count` | l4d2_dirspawn.sp（myinfo 缺 name，用文件名代替） | `30` | 整数，源码写作浮点 | 0.0 ~ 30.0 | 总特最大值 | `l4d2_dirspawn.sp:1113` |
| `dirspawn_auto_min_count` | l4d2_dirspawn.sp（myinfo 缺 name，用文件名代替） | `1` | 整数，源码写作浮点 | 0.0 ~ 30.0 | 总特最小值 | `l4d2_dirspawn.sp:1112` |
| `dirspawn_auto_min_interval` | l4d2_dirspawn.sp（myinfo 缺 name，用文件名代替） | `0` | 整数，源码写作浮点 | 0.0 ~ 120.0 | 人数自适应刷特间隔的下限 | `l4d2_dirspawn.sp:1116` |
| `dirspawn_auto_per_player_add` | l4d2_dirspawn.sp（myinfo 缺 name，用文件名代替） | `1` | 整数，源码写作浮点 | 0.0 ~ 6.0 | 每多 1 名真人增加的特感数 | `l4d2_dirspawn.sp:1111` |
| `dirspawn_auto_per_player_interval_sub` | l4d2_dirspawn.sp（myinfo 缺 name，用文件名代替） | `0` | 整数，源码写作浮点 | 0.0 ~ 30.0 | 每多 1 名真人减少的刷特间隔 | `l4d2_dirspawn.sp:1115` |
| `dirspawn_count` | l4d2_dirspawn.sp（myinfo 缺 name，用文件名代替） | `4` | 整数，源码写作浮点 | 0.0 ~ 30.0 | 并发特感总数，对应 cm_MaxSpecials，范围 0~30 | `l4d2_dirspawn.sp:1063` |
| `dirspawn_dominator_limit` | l4d2_dirspawn.sp（myinfo 缺 name，用文件名代替） | `-1` | 整数，源码写作浮点 | -1.0 ~ 30.0 | DominatorLimit，-1 表示自动取 dirspawn_count | `l4d2_dirspawn.sp:1065` |
| `dirspawn_dps_si_limit` | l4d2_dirspawn.sp（myinfo 缺 name，用文件名代替） | `10` | 整数，源码写作浮点 | 0.0 ~ 30.0 | Not0721 分配中 Spitter 与 Boomer 的数量限制，范围 0~30 | `l4d2_dirspawn.sp:1071` |
| `dirspawn_enable` | l4d2_dirspawn.sp（myinfo 缺 name，用文件名代替） | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 启用导演特感控制：0 关，1 开 | `l4d2_dirspawn.sp:1062` |
| `dirspawn_initial_auto` | l4d2_dirspawn.sp（myinfo 缺 name，用文件名代替） | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否根据 interval 自动调整首刷延迟：0 否，1 是 | `l4d2_dirspawn.sp:1103` |
| `dirspawn_initial_kmax` | l4d2_dirspawn.sp（myinfo 缺 name，用文件名代替） | `1.00` | 整数，源码写作浮点 | 无上下界 | 首刷最大延迟等于该系数乘以 interval | `l4d2_dirspawn.sp:1105` |
| `dirspawn_initial_kmin` | l4d2_dirspawn.sp（myinfo 缺 name，用文件名代替） | `0.80` | 浮点 | 无上下界 | 首刷最小延迟等于该系数乘以 interval | `l4d2_dirspawn.sp:1104` |
| `dirspawn_initial_max` | l4d2_dirspawn.sp（myinfo 缺 name，用文件名代替） | `60` | 整数，源码写作浮点 | 0.0 ~ 60.0 | 首次刷特的最大延迟秒数，范围 0~60 | `l4d2_dirspawn.sp:1093` |
| `dirspawn_initial_min` | l4d2_dirspawn.sp（myinfo 缺 name，用文件名代替） | `30` | 整数，源码写作浮点 | 0.0 ~ 60.0 | 首次刷特的最小延迟秒数，范围 0~60 | `l4d2_dirspawn.sp:1092` |
| `dirspawn_interval` | l4d2_dirspawn.sp（myinfo 缺 name，用文件名代替） | `35` | 整数，源码写作浮点 | 0.0 ~ 120.0 | 特感复活间隔，对应 cm_SpecialRespawnInterval，单位秒，范围 0~120 | `l4d2_dirspawn.sp:1064` |
| `dirspawn_kv_enable` | l4d2_dirspawn.sp（myinfo 缺 name，用文件名代替） | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否使用 KV 设置每类上限：0 否，1 是 | `l4d2_dirspawn.sp:1068` |
| `dirspawn_kv_path` | l4d2_dirspawn.sp（myinfo 缺 name，用文件名代替） | `cfg/sourcemod/dirspawn_si_limits.cfg` | 字符串或表达式 | 无上下界 | 每类上限 KV 文件路径 | `l4d2_dirspawn.sp:1069` |
| `dirspawn_limit_style` | l4d2_dirspawn.sp（myinfo 缺 name，用文件名代替） | `0` | 整数，源码写作浮点 | 0.0 ~ 2.0 | 每类上限分配方式：0 KV 或均衡，1 Not0721，2 Not0721 community2 | `l4d2_dirspawn.sp:1070` |
| `dirspawn_lock_tempo` | l4d2_dirspawn.sp（myinfo 缺 name，用文件名代替） | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否锁节奏：0 否，1 是 | `l4d2_dirspawn.sp:1083` |
| `dirspawn_lock_tempo_threshold` | l4d2_dirspawn.sp（myinfo 缺 name，用文件名代替） | `6` | 整数 | 无上下界 | interval 小于等于该阈值时自动把 LockTempo 设为 1，单位秒 | `l4d2_dirspawn.sp:1101` |
| `dirspawn_relax_auto` | l4d2_dirspawn.sp（myinfo 缺 name，用文件名代替） | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否根据 interval 自动调整 Relax 与 Lock：0 否，1 是 | `l4d2_dirspawn.sp:1096` |
| `dirspawn_relax_ceil` | l4d2_dirspawn.sp（myinfo 缺 name，用文件名代替） | `120` | 整数 | 无上下界 | RelaxMax 的上限秒数 | `l4d2_dirspawn.sp:1100` |
| `dirspawn_relax_enable` | l4d2_dirspawn.sp（myinfo 缺 name，用文件名代替） | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否保留导演 Relax 阶段：0 按 Not0721 源服方式压掉 Relax | `l4d2_dirspawn.sp:1080` |
| `dirspawn_relax_floor` | l4d2_dirspawn.sp（myinfo 缺 name，用文件名代替） | `0` | 整数 | 无上下界 | RelaxMin 的下限秒数 | `l4d2_dirspawn.sp:1099` |
| `dirspawn_relax_kmax` | l4d2_dirspawn.sp（myinfo 缺 name，用文件名代替） | `1.10` | 浮点 | 无上下界 | RelaxMax 等于该系数乘以 interval | `l4d2_dirspawn.sp:1098` |
| `dirspawn_relax_kmin` | l4d2_dirspawn.sp（myinfo 缺 name，用文件名代替） | `0.75` | 浮点 | 无上下界 | RelaxMin 等于该系数乘以 interval | `l4d2_dirspawn.sp:1097` |
| `dirspawn_relax_max` | l4d2_dirspawn.sp（myinfo 缺 name，用文件名代替） | `45` | 整数，源码写作浮点 | 0.0 ~ 180.0 | Relax 阶段最大秒数，范围 0~180 | `l4d2_dirspawn.sp:1082` |
| `dirspawn_relax_min` | l4d2_dirspawn.sp（myinfo 缺 name，用文件名代替） | `30` | 整数，源码写作浮点 | 0.0 ~ 120.0 | Relax 阶段最小秒数，范围 0~120 | `l4d2_dirspawn.sp:1081` |
| `dirspawn_relax_off_battlefield_respawn` | l4d2_dirspawn.sp（myinfo 缺 name，用文件名代替） | `2` | 整数，源码写作浮点 | 0.0 ~ 30.0 | Relax 关闭时 director_special_battlefield_respawn_interval 的取值 | `l4d2_dirspawn.sp:1084` |
| `dirspawn_relax_off_finale_offer` | l4d2_dirspawn.sp（myinfo 缺 name，用文件名代替） | `1` | 整数，源码写作浮点 | 0.0 ~ 30.0 | Relax 关闭时 director_special_finale_offer_length 的取值 | `l4d2_dirspawn.sp:1088` |
| `dirspawn_relax_off_initial_delay_max` | l4d2_dirspawn.sp（myinfo 缺 name，用文件名代替） | `1` | 整数，源码写作浮点 | 0.0 ~ 60.0 | Relax 关闭时 director_special_initial_spawn_delay_max 的取值 | `l4d2_dirspawn.sp:1085` |
| `dirspawn_relax_off_initial_delay_max_extra` | l4d2_dirspawn.sp（myinfo 缺 name，用文件名代替） | `2` | 整数，源码写作浮点 | 0.0 ~ 180.0 | Relax 关闭时 director_special_initial_spawn_delay_max_extra 的取值 | `l4d2_dirspawn.sp:1086` |
| `dirspawn_relax_off_initial_delay_min` | l4d2_dirspawn.sp（myinfo 缺 name，用文件名代替） | `0` | 整数，源码写作浮点 | 0.0 ~ 60.0 | Relax 关闭时 director_special_initial_spawn_delay_min 的取值 | `l4d2_dirspawn.sp:1087` |
| `dirspawn_relax_off_original_offer` | l4d2_dirspawn.sp（myinfo 缺 name，用文件名代替） | `1` | 整数，源码写作浮点 | 0.0 ~ 60.0 | Relax 关闭时 director_special_original_offer_length 的取值 | `l4d2_dirspawn.sp:1089` |
| `dirspawn_unlock_maxspecial` | l4d2_dirspawn.sp（myinfo 缺 name，用文件名代替） | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否解锁战役模式 3 特上限，需 sourcescramble 与 gamedata | `l4d2_dirspawn.sp:1076` |
| `dirspawn_verbose` | l4d2_dirspawn.sp（myinfo 缺 name，用文件名代替） | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否输出详细服务器日志：0 否，1 是 | `l4d2_dirspawn.sp:1072` |
| `engine_fix_flags` | [L4D & L4D2] Engine Fix | `14` | 整数，源码写作浮点 | 0.0 ~ 30.0 | 要修复或阻止的漏洞类型位标志，可相加：0 关闭，2 梯子加速漏洞，4 无摔落伤害，其余被截断 | `fix_engine.sp:59` |
| `engine_fix_version` | [L4D & L4D2] Engine Fix | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | `fix_engine.sp:56` |
| `engine_warning` | [L4D & L4D2] Engine Fix | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否显示玩家正在使用漏洞的警告：1 启用，0 关闭 | `fix_engine.sp:58` |
| `gfc_charger_duration` | L4D2 Godframes Control combined with FF Plugins | `2.1` | 浮点 | 0.0 ~ 3.0 | 冲锋压制后的神圣帧持续秒数，范围 0~3 | `l4d2_godframes_control_merge.sp:146` |
| `gfc_charger_stagger_extra_time` | L4D2 Godframes Control combined with FF Plugins | `0.0` | 整数，源码写作浮点 | 0.0 ~ 3.0 | 来自 ChargerFlags 的伤害前的额外神圣帧时间，范围 0~3 | `l4d2_godframes_control_merge.sp:147` |
| `gfc_charger_stagger_flags` | L4D2 Godframes Control combined with FF Plugins | `0` | 整数，源码写作浮点 | 0.0 ~ 3.0 | 受额外 Charger 硬直保护时间影响的类型：1 普感，2 毒痰，范围 0~3 | `l4d2_godframes_control_merge.sp:148` |
| `gfc_common_extra_time` | L4D2 Godframes Control combined with FF Plugins | `0.0` | 整数，源码写作浮点 | 0.0 ~ 3.0 | 允许普感伤害前的额外神圣帧时间，范围 0~3 | `l4d2_godframes_control_merge.sp:142` |
| `gfc_common_zc_flags` | L4D2 Godframes Control combined with FF Plugins | `0` | 整数，源码写作浮点 | 0.0 ~ 15.0 | 受额外普感保护时间影响的特感职业位域：1 Hunter，2 Smoker，4 Jockey，8 Charger，范围 0~15 | `l4d2_godframes_control_merge.sp:150` |
| `gfc_ff_min_time` | L4D2 Godframes Control combined with FF Plugins | `0.3` | 浮点 | 0.0 ~ 3.0 | 允许友军伤害前的最短神圣帧时间，范围 0~3 | `l4d2_godframes_control_merge.sp:140` |
| `gfc_godframe_glows` | L4D2 Godframes Control combined with FF Plugins | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否改变处于神圣帧的生还者渲染，红色或透明 | `l4d2_godframes_control_merge.sp:134` |
| `gfc_hittable_override` | L4D2 Godframes Control combined with FF Plugins | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否允许可击打物始终无视神圣帧 | `l4d2_godframes_control_merge.sp:137` |
| `gfc_hittable_rage_override` | L4D2 Godframes Control combined with FF Plugins | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否允许坦克从可击打物命中获得怒气，0 阻止获得 | `l4d2_godframes_control_merge.sp:135` |
| `gfc_hunter_duration` | L4D2 Godframes Control combined with FF Plugins | `2.1` | 浮点 | 0.0 ~ 3.0 | 飞扑后的神圣帧持续秒数，范围 0~3 | `l4d2_godframes_control_merge.sp:143` |
| `gfc_jockey_duration` | L4D2 Godframes Control combined with FF Plugins | `0.0` | 整数，源码写作浮点 | 0.0 ~ 3.0 | 骑乘后的神圣帧持续秒数，范围 0~3 | `l4d2_godframes_control_merge.sp:144` |
| `gfc_long_charger_duration` | [L4D2] Merged Get-Up Fixes | `2.2` | 浮点 | 无上下界 | 长冲锋起身动画的神圣帧持续秒数 | `l4d2_getup_fixes.sp:156` |
| `gfc_rock_override` | L4D2 Godframes Control combined with FF Plugins | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否允许可击打物始终无视神圣帧，同一插件的另一开关 | `l4d2_godframes_control_merge.sp:138` |
| `gfc_rock_rage_override` | L4D2 Godframes Control combined with FF Plugins | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否允许坦克从神圣帧内的命中获得怒气，0 阻止获得 | `l4d2_godframes_control_merge.sp:136` |
| `gfc_smoker_duration` | L4D2 Godframes Control combined with FF Plugins | `0.0` | 整数，源码写作浮点 | 0.0 ~ 3.0 | 拉拽或勒喉后的神圣帧持续秒数，范围 0~3 | `l4d2_godframes_control_merge.sp:145` |
| `gfc_spit_extra_time` | L4D2 Godframes Control combined with FF Plugins | `0.7` | 浮点 | 0.0 ~ 3.0 | 允许毒痰伤害前的额外神圣帧时间，范围 0~3 | `l4d2_godframes_control_merge.sp:141` |
| `gfc_spit_zc_flags` | L4D2 Godframes Control combined with FF Plugins | `6` | 整数，源码写作浮点 | 0.0 ~ 15.0 | 受额外毒痰保护时间影响的特感职业位域：1 Hunter，2 Smoker，4 Jockey，8 Charger，范围 0~15 | `l4d2_godframes_control_merge.sp:149` |
| `gfc_witch_override` | L4D2 Godframes Control combined with FF Plugins | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否允许 Witch 始终无视神圣帧 | `l4d2_godframes_control_merge.sp:139` |
| `ghost_hurt_type` | Ghost Hurt Management | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 何时启用 trigger_hurt_ghost：0 从不，1 回合开始时 | `ghost_hurt.sp:26` |
| `hc_atlas_ball_standing_damage` | L4D2 Hittable Control | `100.0` | 整数，源码写作浮点 | ≥ -2.0 | atlas 球对未倒地生还者的伤害 | `l4d2_hittable_control.sp:202` |
| `hc_baggage_standing_damage` | L4D2 Hittable Control | `48.0` | 整数，源码写作浮点 | ≥ -2.0 | 行李车对未倒地生还者的伤害 | `l4d2_hittable_control.sp:190` |
| `hc_bhlog_standing_damage` | L4D2 Hittable Control | `100.0` | 整数，源码写作浮点 | ≥ -2.0 | blood harvest 原木对未倒地生还者的伤害 | `l4d2_hittable_control.sp:166` |
| `hc_boat_smash_standing_damage` | L4D2 Hittable Control | `23.0` | 整数，源码写作浮点 | ≥ -2.0 | 船只碎片对未倒地生还者的伤害 | `l4d2_hittable_control.sp:211` |
| `hc_brick_pallets_standing_damage` | L4D2 Hittable Control | `13.0` | 整数，源码写作浮点 | ≥ -2.0 | 砖托盘碎块对未倒地生还者的伤害 | `l4d2_hittable_control.sp:208` |
| `hc_broken_forklift_standing_damage` | L4D2 Hittable Control | `100.0` | 整数，源码写作浮点 | ≥ -2.0 | 损坏的叉车对未倒地生还者的伤害 | `l4d2_hittable_control.sp:181` |
| `hc_bumpercar_standing_damage` | L4D2 Hittable Control | `100.0` | 整数，源码写作浮点 | ≥ -2.0 | 碰碰车对未倒地生还者的伤害 | `l4d2_hittable_control.sp:172` |
| `hc_car_standing_damage` | L4D2 Hittable Control | `100.0` | 整数，源码写作浮点 | ≥ -2.0 | 汽车对未倒地生还者的伤害 | `l4d2_hittable_control.sp:169` |
| `hc_concrete_piller_standing_damage` | L4D2 Hittable Control | `8.0` | 整数，源码写作浮点 | ≥ -2.0 | 混凝土柱碎片对未倒地生还者的伤害 | `l4d2_hittable_control.sp:214` |
| `hc_debug` | L4D2 Hittable Control | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 0 关闭调试，1 启用调试 | `l4d2_hittable_control.sp:232` |
| `hc_diescraper_ball_standing_damage` | L4D2 Hittable Control | `100.0` | 整数，源码写作浮点 | ≥ -2.0 | Diescraper 终局球形雕像对未倒地生还者的伤害 | `l4d2_hittable_control.sp:217` |
| `hc_disable_self_damage` | L4D2 Hittable Control | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 若开启，坦克不会用可击打物伤害自己 | `l4d2_hittable_control.sp:226` |
| `hc_dumpster_standing_damage` | L4D2 Hittable Control | `100.0` | 整数，源码写作浮点 | ≥ -2.0 | 垃圾箱对未倒地生还者的伤害 | `l4d2_hittable_control.sp:184` |
| `hc_forklift_standing_damage` | L4D2 Hittable Control | `100.0` | 整数，源码写作浮点 | ≥ -2.0 | 叉车对未倒地生还者的伤害 | `l4d2_hittable_control.sp:178` |
| `hc_gauntlet_finale_multiplier` | L4D2 Hittable Control | `0.25` | 浮点 | 0.0 ~ 4.0 | 可击打物在 gauntlet 终局中的伤害倍率，范围 0~4 | `l4d2_hittable_control.sp:160` |
| `hc_generator_trailer_standing_damage` | L4D2 Hittable Control | `48.0` | 整数，源码写作浮点 | ≥ -2.0 | 发电拖车对未倒地生还者的伤害 | `l4d2_hittable_control.sp:193` |
| `hc_handtruck_standing_damage` | L4D2 Hittable Control | `8.0` | 整数，源码写作浮点 | ≥ -2.0 | 手推车对未倒地生还者的伤害 | `l4d2_hittable_control.sp:175` |
| `hc_haybale_standing_damage` | L4D2 Hittable Control | `48.0` | 整数，源码写作浮点 | ≥ -2.0 | 干草捆对未倒地生还者的伤害 | `l4d2_hittable_control.sp:187` |
| `hc_ibeam_standing_damage` | L4D2 Hittable Control | `48.0` | 整数，源码写作浮点 | ≥ -2.0 | 工字钢对未倒地生还者的伤害 | `l4d2_hittable_control.sp:205` |
| `hc_incap_standard_damage` | L4D2 Hittable Control | `100` | 整数，源码写作浮点 | ≥ -2.0 | 所有可击打物对倒地玩家的伤害，-1 表示沿用 Valve 默认倒地伤害 | `l4d2_hittable_control.sp:223` |
| `hc_militia_rock_standing_damage` | L4D2 Hittable Control | `100.0` | 整数，源码写作浮点 | ≥ -2.0 | 民兵石头对未倒地生还者的伤害 | `l4d2_hittable_control.sp:196` |
| `hc_overhit_time` | L4D2 Hittable Control | `1.2` | 浮点 | ≥ 0.0 | 允许同一可击打物连续命中前的等待秒数 | `l4d2_hittable_control.sp:229` |
| `hc_phys_mass_incap_threshold` | L4D2 Hittable Control | `500.0` | 整数，源码写作浮点 | ≥ 0.0 | 重量超过该质量的可击打物命中时会击倒生还者，0.0 关闭 | `l4d2_hittable_control.sp:238` |
| `hc_sflog_standing_damage` | L4D2 Hittable Control | `48.0` | 整数，源码写作浮点 | ≥ -2.0 | swamp fever 原木对未倒地生还者的伤害 | `l4d2_hittable_control.sp:163` |
| `hc_sofa_chair_standing_damage` | L4D2 Hittable Control | `100.0` | 整数，源码写作浮点 | ≥ -2.0 | Blood Harvest 终局沙发对未倒地生还者的伤害，仅对带目标跟踪的沙发生效 | `l4d2_hittable_control.sp:199` |
| `hc_unbreakable_forklifts` | L4D2 Hittable Control | `0` | 整数，源码写作浮点 | ≥ 0.0 | 是否阻止叉车被坦克击中后碎成多块 | `l4d2_hittable_control.sp:235` |
| `hc_van_standing_damage` | L4D2 Hittable Control | `100.0` | 整数，源码写作浮点 | ≥ -2.0 | Detour Ahead 第二关面包车对未倒地生还者的伤害 | `l4d2_hittable_control.sp:220` |
| `hunter_ground_m2_godframes` | [L4D2] No Hunter Deadstops | `0.75` | 浮点 | 0.0 ~ 1.0 | hunter 落地后的 m2 神圣帧秒数，范围 0~1 | `l4d2_hunter_no_deadstops.sp:35` |
| `inf_traitor_daily_quota` | Anne Traitor Quota | `100` | 整数，源码写作浮点 | 0.0 ~ 10000.0 | 管理员的每日内鬼基础配额，sb_admins.expires 每剩满一年加 50 | `anne_traitor_quota.sp:23` |
| `inf_traitor_public_daily_quota` | Anne Traitor Quota | `20` | 整数，源码写作浮点 | 0.0 ~ 10000.0 | 合格非管理员玩家每日可用的内鬼配额 | `anne_traitor_quota.sp:27` |
| `inf_traitor_quota_db` | Anne Traitor Quota | `DEFAULT_TRAITOR_QUOTA_DB_CONFIG` | 字符串或表达式 | 无上下界 | 共享内鬼每日配额存储所用的 MySQL databases.cfg 区块，默认使用 l4d_stats | `anne_traitor_quota.sp:31` |
| `inf_traitor_quota_table` | Anne Traitor Quota | `infected_control_traitor_quota` | 字符串或表达式 | 无上下界 | 内鬼每日配额存储的 SQL 表名 | `anne_traitor_quota.sp:35` |
| `infected_extinguish_time` | SI Fire Immunity | `1.0` | 整数，源码写作浮点 | 0.0 ~ 999.0 | 特感玩家在多少秒后熄灭，仅当 infected_fire_immunity 为 3 时生效 | `si_fire_immunity.sp:63` |
| `infected_fire_immunity` | SI Fire Immunity | `3` | 整数，源码写作浮点 | 0.0 ~ 3.0 | 特感的火焰免疫类型：0 无，3 随时间自动熄灭，2 免疫燃烧，1 完全免疫 | `si_fire_immunity.sp:49` |
| `jockey_skeet_report` | L4D2 Jockey Skeet | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否在聊天中播报 Jockey 被击杀 | `l4d2_jockey_skeet.sp:45` |
| `join_autoupdate` | simple join | `0` | 整数，源码写作浮点 | 0.0 ~ 2.0 | 自动更新模式：0 关闭，1 无数据库插件，2 含数据库插件 | `join.sp:98` |
| `join_autoupdate_private_url` | simple join | `UPDATE_URL_PRIVATE` | 字符串或表达式 | 无上下界 | 自动更新为 2 时使用的数据库更新清单 URL | `join.sp:100` |
| `join_autoupdate_public_url` | simple join | `UPDATE_URL_PUBLIC` | 字符串或表达式 | 无上下界 | 自动更新为 1 时使用的无数据库更新清单 URL | `join.sp:99` |
| `join_enable_inf` | simple join | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否允许玩家加入特感 | `join.sp:96` |
| `join_enable_kickfamilyaccount` | simple join | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否踢出家庭共享账户 | `join.sp:97` |
| `l4d2_add_melee` | l4d2 melee spawn control | `` | 字符串或表达式 | 无上下界 | 添加近战武器到地图基础刷新列表或 l4d2_melee_spawn，逗号分隔，留空不添加 | `l4d2_melee_spawn_control.sp:66` |
| `l4d2_addons_eclipse` | [L4D & L4D2] Left 4 DHooks Direct | `-1` | 整数 | 无上下界 | Addons 管理器：-1 使用 addonconfig，0 禁用 addons，1 启用 addons | `left4dhooks.sp:740` |
| `l4d2_ai_ladder_boost` | L4D2 SI LADDER BOOSTER | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否允许 AI 特感在梯子上加速，范围 0~1 | `l4d2_si_ladder_booster.sp:30` |
| `l4d2_ai_ladder_boost` | [L4D2] Infected Ladder Speed Boost | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 兼容旧 l4d2_si_ladder_booster：AI 特感在梯子上固定加速 | `l4d2_ai_ladder_boost.sp:94` |
| `l4d2_Anne_stuck_tank_teleport` | Anne Stuck Tank Teleport System (ASTT) | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | `l4d2_Anne_stuck_tank_teleport.sp:104` |
| `l4d2_anne_thirdperson_fix_cfg_names` | Anne Thirdperson Shoulder Fix | `DEFAULT_CFG_NAMES` | 字符串或表达式 | 无上下界 | 逗号分隔的 l4d_ready_cfg_name 片段，匹配时启用该修复，留空为全部配置 | `l4d2_anne_thirdperson_fix.sp:38` |
| `l4d2_anne_thirdperson_fix_commands` | Anne Thirdperson Shoulder Fix | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 0 关闭，1 启用 !tp 与 !third 命令 | `l4d2_anne_thirdperson_fix.sp:35` |
| `l4d2_anne_thirdperson_fix_debug` | Anne Thirdperson Shoulder Fix | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 0 关闭，1 记录客户端伪造与恢复操作 | `l4d2_anne_thirdperson_fix.sp:39` |
| `l4d2_anne_thirdperson_fix_enabled` | Anne Thirdperson Shoulder Fix | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 0 关闭，1 在 l4d_ready_cfg_name 匹配时启用 Anne 第三人称修复 | `l4d2_anne_thirdperson_fix.sp:34` |
| `l4d2_anne_thirdperson_fix_fake_gamemode` | Anne Thirdperson Shoulder Fix | `coop` | 字符串或表达式 | 无上下界 | 仅发送给 !tp 客户端的 mp_gamemode 值 | `l4d2_anne_thirdperson_fix.sp:37` |
| `l4d2_anne_thirdperson_fix_spoof_gamemode` | Anne Thirdperson Shoulder Fix | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 0 关闭，1 为 !tp 在启用 thirdpersonshoulder 前伪造 mp_gamemode | `l4d2_anne_thirdperson_fix.sp:36` |
| `l4d2_anne_thirdperson_fix_version` | Anne Thirdperson Shoulder Fix | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | `l4d2_anne_thirdperson_fix.sp:33` |
| `l4d2_antibaiter_bile_stop` | L4D2 Antibaiter | `0` | 整数 | 无上下界 | 玩家被 Boomer 喷吐时是否停止计时器 | `l4d2_antibaiter.sp:83` |
| `l4d2_antibaiter_delay` | L4D2 Antibaiter | `20` | 整数 | 无上下界 | 反挂机算法启动前的延迟秒数 | `l4d2_antibaiter.sp:80` |
| `l4d2_antibaiter_horde_timer` | L4D2 Antibaiter | `60` | 整数 | 无上下界 | 到尸潮的倒计时秒数 | `l4d2_antibaiter.sp:81` |
| `l4d2_antibaiter_progress` | L4D2 Antibaiter | `0.03` | 浮点 | 无上下界 | 生还者必须取得的最小进度才重置反挂机计时器 | `l4d2_antibaiter.sp:82` |
| `l4d2_antirock_protect_time` | [L4D2] Tank Spawn Anti-Rock Protect | `1.5` | 浮点 | 无上下界 | 保护时间秒数，避免 Tank 意外投掷石头 | `l4d2_tank_spawn_antirock_protect.sp:21` |
| `l4d2_astt_block_checkpoint` | Anne Stuck Tank Teleport System (ASTT) | `1` | 整数 | 无上下界 | 是否禁止传送到 CHECKPOINT 导航区域：1 是，0 否 | `l4d2_Anne_stuck_tank_teleport.sp:116` |
| `l4d2_astt_debug` | Anne Stuck Tank Teleport System (ASTT) | `0` | 整数 | 无上下界 | 是否输出调试信息：1 开，0 关 | `l4d2_Anne_stuck_tank_teleport.sp:118` |
| `l4d2_astt_door_threshold` | Anne Stuck Tank Teleport System (ASTT) | `1000` | 整数 | 无上下界 | 生还者与安全室距离小于等于该值时传送到 DOOR，否则传送到 NEAR，单位为地图单位 | `l4d2_Anne_stuck_tank_teleport.sp:117` |
| `l4d2_astt_enable` | Anne Stuck Tank Teleport System (ASTT) | `1` | 整数 | 无上下界 | 是否启用插件：1 开，0 关 | `l4d2_Anne_stuck_tank_teleport.sp:106` |
| `l4d2_astt_non_stuck_radius` | Anne Stuck Tank Teleport System (ASTT) | `20` | 整数 | 无上下界 | 间隔内移动距离小于该值即判定为卡死 | `l4d2_Anne_stuck_tank_teleport.sp:108` |
| `l4d2_astt_rusher_checks` | Anne Stuck Tank Teleport System (ASTT) | `6` | 整数 | 无上下界 | 确认跑男前的检测次数 | `l4d2_Anne_stuck_tank_teleport.sp:112` |
| `l4d2_astt_rusher_dist` | Anne Stuck Tank Teleport System (ASTT) | `2800` | 整数 | 无上下界 | 距最近坦克小于该距离视为跑男 | `l4d2_Anne_stuck_tank_teleport.sp:111` |
| `l4d2_astt_rusher_interval` | Anne Stuck Tank Teleport System (ASTT) | `3` | 整数 | 无上下界 | 跑男检测间隔秒数 | `l4d2_Anne_stuck_tank_teleport.sp:113` |
| `l4d2_astt_rusher_minplayers` | Anne Stuck Tank Teleport System (ASTT) | `2` | 整数 | 无上下界 | 启用跑男规则所需的最少存活生还者数 | `l4d2_Anne_stuck_tank_teleport.sp:114` |
| `l4d2_astt_rusher_punish` | Anne Stuck Tank Teleport System (ASTT) | `1` | 整数 | 无上下界 | 是否通过传送坦克惩罚跑男：1 是，0 否 | `l4d2_Anne_stuck_tank_teleport.sp:110` |
| `l4d2_astt_stuck_check_interval` | Anne Stuck Tank Teleport System (ASTT) | `3` | 整数 | 无上下界 | Tank 卡死检测间隔秒数 | `l4d2_Anne_stuck_tank_teleport.sp:107` |
| `l4d2_auto_restart_delay` | L4D2 Auto restart | `30.0` | 整数，源码写作浮点 | 无上下界 | 重启宽限时间，单位秒 | `linux_auto_restart.sp:35` |
| `l4d2_auto_restart_version` | L4D2 Auto restart | `VERSION` | 字符串或表达式 | 无上下界 | 插件版本 | `linux_auto_restart.sp:34` |
| `l4d2_bbp_debug_enabled` | [L4D2] Block Bot Pills | `0` | 整数 | 无上下界 | 是否开启调试模式 | `l4d2_block_bot_pills.sp:33` |
| `l4d2_block_autoaim` | [L4D2] Block Autoaim | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否禁用自动瞄准 | `l4d2_block_autoaim.sp:38` |
| `l4d2_block_infected_ff` | L4D2 Infected Friendly Fire Disable | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否禁用特感之间的友军伤害 | `l4d2_si_ffblock.sp:41` |
| `l4d2_block_jump_rock` | Tank Attack Control | `0` | 整数 | 无上下界 | 是否阻止坦克同时跳跃与投掷石头 | `l4d2_tank_attack_control.sp:51` |
| `l4d2_block_no_steam_logon_check_timeout` | [L4D2] Block No Steam Logon | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 当客户端网络通道超时后是否断开该客户端 | `l4d2_block_no_steam_logon.sp:128` |
| `l4d2_block_no_steam_logon_enable` | [L4D2] Block No Steam Logon | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否阻止 Steam 鉴权响应 1 与 6 导致的客户端断线 | `l4d2_block_no_steam_logon.sp:116` |
| `l4d2_block_no_steam_logon_version` | [L4D2] Block No Steam Logon | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | `l4d2_block_no_steam_logon.sp:109` |
| `l4d2_block_punch_rock` | Tank Attack Control | `1` | 整数 | 无上下界 | 是否阻止坦克同时拳击与投掷石头 | `l4d2_tank_attack_control.sp:50` |
| `l4d2_boost_multiplier` | L4D2 SI LADDER BOOSTER | `3.2` | 浮点 | 0.0 ~ 10.0 | 爬梯加速倍数，范围 0~10 | `l4d2_si_ladder_booster.sp:32` |
| `l4d2_boost_multiplier` | [L4D2] Infected Ladder Speed Boost | `3.2` | 浮点 | 1.0 ~ 10.0 | 兼容旧 l4d2_si_ladder_booster：固定爬梯加速倍数，范围 1~10 | `l4d2_ai_ladder_boost.sp:96` |
| `l4d2_car_alarm_settings` | L4D2 Car Alarm Fixes | `3` | 整数 | 无上下界 | 位掩码：1 生还者触碰时触发警报，2 可击打物击中警报车时禁用警报 | `l4d2_car_alarm_hittable_fix.sp:80` |
| `l4d2_car_alarm_touch_ai` | L4D2 Car Alarm Fixes | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否统计 AI 生还者触碰警报车，需位掩码设置生效 | `l4d2_car_alarm_hittable_fix.sp:82` |
| `l4d2_car_alarm_touch_capped` | L4D2 Car Alarm Fixes | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 仅在生还者触碰车时被特感控制才追加警报触发，需位掩码设置生效 | `l4d2_car_alarm_hittable_fix.sp:81` |
| `l4d2_ceda_fire_proof` | [L4D2] Uncommon Adjustment | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | CEDA 是否防火 | `l4d2_uncommon_adjustment.sp:156` |
| `l4d2_character_manager_version` | [L4D2]Character_manager | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | `l4d2_character_manager.sp:105` |
| `l4d2_cl_link` | L4D2 Change Log Command | `https://github.com/spoon-l4d2/NextMod` | 字符串或表达式 | 无上下界 | 更新日志页面的链接 | `changelog.sp:20` |
| `l4d2_climb_anim_boost` | [L4D2] Infected Ladder Speed Boost | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否由该插件处理 AI Tank 翻越或爬小障碍的动画加速 | `l4d2_ai_ladder_boost.sp:98` |
| `l4d2_deathspit_trace_height` | [L4D2] Spit Spread Patch | `240.0` | 整数，源码写作浮点 | ≥ 0.0 | 死亡毒痰检测射线的长度，240.0 为默认长度 | `l4d2_spit_spread_patch.sp:169` |
| `l4d2_disable_si_friendly_staggers` | L4D2 No SI Friendly Staggers | `0` | 整数，源码写作浮点 | 0.0 ~ 7.0 | 是否移除其他特感造成的特感硬直，位掩码：1 Boomer，2 Charger，4 Witch，范围 0~7 | `l4d2_si_staggers.sp:55` |
| `l4d2_dominators` | Dominators Control | `53` | 整数，源码写作浮点 | 0.0 ~ 63.0 | 哪些特感被视为控制类，位掩码：1 smoker，2 boomer，4 hunter，8 spitter，16 jockey，其余被截断，范围 0~63 | `l4d2_dominatorscontrol.sp:38` |
| `l4d2_door_lock_version` | L4D2 Saferoom Locker | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | `l4d2_door_lock.sp:196` |
| `l4d2_doorlock_add_commands` | L4D2 Saferoom Locker | `!map,!buy,!shop` | 字符串或表达式 | 无上下界 | 需要屏蔽以免干扰 ReadyUp 面板的命令列表，无空格分隔 | `l4d2_door_lock.sp:211` |
| `l4d2_doorlock_countdown` | L4D2 Saferoom Locker | `0` | 整数 | 无上下界 | 解锁安全区门的倒计时秒数 | `l4d2_door_lock.sp:199` |
| `l4d2_doorlock_enable_ready_mode` | L4D2 Saferoom Locker | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 为 1 时启用准备（ReadyUp）功能 | `l4d2_door_lock.sp:205` |
| `l4d2_doorlock_freeze_survivor_bots` | L4D2 Saferoom Locker | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 为 1 时锁门期或第一关真人离开前只冻结生还者 Bot，真人接管后自动解冻；0 保留 nb_player_stop 停止全部 Bot | `l4d2_door_lock.sp:212` |
| `l4d2_doorlock_game_mode` | L4D2 Saferoom Locker | `versus,coop` | 字符串或表达式 | 无上下界 | 在这些模式中启用插件，逗号分隔，留空为全模式 | `l4d2_door_lock.sp:198` |
| `l4d2_doorlock_glow_enable` | L4D2 Saferoom Locker | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 为 1 时为安全室的门设置发光效果 | `l4d2_door_lock.sp:201` |
| `l4d2_doorlock_glow_range` | L4D2 Saferoom Locker | `500` | 整数 | 无上下界 | 安全门的发光范围 | `l4d2_door_lock.sp:202` |
| `l4d2_doorlock_leavers_notify` | L4D2 Saferoom Locker | `2` | 整数，源码写作浮点 | 0.0 ~ 1.0 | 被传送时给离开者的提示：0 禁用，1 聊天，2 中心文本 | `l4d2_door_lock.sp:210` |
| `l4d2_doorlock_loaders_time` | L4D2 Saferoom Locker | `40` | 整数 | 无上下界 | 等待玩家加载的最长秒数 | `l4d2_door_lock.sp:200` |
| `l4d2_doorlock_lock_glow_color` | L4D2 Saferoom Locker | `255 0 0` | 字符串或表达式 | 无上下界 | 上锁安全门的发光颜色，0-255 RGB 以空格分隔 | `l4d2_door_lock.sp:203` |
| `l4d2_doorlock_plugin_enable` | L4D2 Saferoom Locker | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 为 1 时启用插件 | `l4d2_door_lock.sp:197` |
| `l4d2_doorlock_readyup_notify` | L4D2 Saferoom Locker | `1` | 整数，源码写作浮点 | 0.0 ~ 3.0 | 团队准备就绪时的提示方式：0 禁用，1 聊天，2 中心文本，3 两者 | `l4d2_door_lock.sp:209` |
| `l4d2_doorlock_readyup_percent` | L4D2 Saferoom Locker | `75.0` | 整数，源码写作浮点 | 无上下界 | 一方开始游戏所需的最低准备百分比 | `l4d2_door_lock.sp:208` |
| `l4d2_doorlock_readyup_time` | L4D2 Saferoom Locker | `120` | 整数 | 无上下界 | 对未准备的团队等待多久后强制开始，单位秒 | `l4d2_door_lock.sp:207` |
| `l4d2_doorlock_unlock_glow_color` | L4D2 Saferoom Locker | `0 255 0` | 字符串或表达式 | 无上下界 | 解锁安全门的发光颜色，0-255 RGB 以空格分隔 | `l4d2_door_lock.sp:204` |
| `l4d2_doorlock_unready_counts` | L4D2 Saferoom Locker | `2` | 整数 | 无上下界 | 每轮允许玩家使用未准备的次数 | `l4d2_door_lock.sp:206` |
| `l4d2_drop_secondary_debug` | L4D2 Drop Secondary | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否开启调试输出 | `l4d2_drop_secondary.sp:48` |
| `l4d2_dynamic_ammo_allow_m60` | L4D2 Dynamic Ammo (dirspawn only) | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否处理 M60 弹药 | `l4d2_dynamic_ammo.sp:220` |
| `l4d2_dynamic_ammo_alpha` | L4D2 Dynamic Ammo (dirspawn only) | `1.0` | 整数，源码写作浮点 | ≥ 0.0 | SI 指数，最小 0 | `l4d2_dynamic_ammo.sp:215` |
| `l4d2_dynamic_ammo_base_auto` | L4D2 Dynamic Ammo (dirspawn only) | `90` | 整数 | 无上下界 | 连发霰弹枪基础预备弹数 | `l4d2_dynamic_ammo.sp:226` |
| `l4d2_dynamic_ammo_base_gl` | L4D2 Dynamic Ammo (dirspawn only) | `30` | 整数 | 无上下界 | 榴弹发射器基础预备弹数 | `l4d2_dynamic_ammo.sp:228` |
| `l4d2_dynamic_ammo_base_interval` | L4D2 Dynamic Ammo (dirspawn only) | `35.0` | 整数，源码写作浮点 | ≥ 1.0 | 基准刷特间隔秒数，最小 1 | `l4d2_dynamic_ammo.sp:214` |
| `l4d2_dynamic_ammo_base_pump` | L4D2 Dynamic Ammo (dirspawn only) | `72` | 整数 | 无上下界 | 泵动霰弹枪基础预备弹数 | `l4d2_dynamic_ammo.sp:225` |
| `l4d2_dynamic_ammo_base_rifle` | L4D2 Dynamic Ammo (dirspawn only) | `360` | 整数 | 无上下界 | 步枪基础预备弹数 | `l4d2_dynamic_ammo.sp:224` |
| `l4d2_dynamic_ammo_base_si` | L4D2 Dynamic Ammo (dirspawn only) | `4` | 整数，源码写作浮点 | ≥ 1.0 | 基准 SI 数量，最小 1 | `l4d2_dynamic_ammo.sp:213` |
| `l4d2_dynamic_ammo_base_smg` | L4D2 Dynamic Ammo (dirspawn only) | `650` | 整数 | 无上下界 | SMG 基础预备弹数 | `l4d2_dynamic_ammo.sp:223` |
| `l4d2_dynamic_ammo_base_sniper` | L4D2 Dynamic Ammo (dirspawn only) | `180` | 整数 | 无上下界 | 狙击枪基础预备弹数 | `l4d2_dynamic_ammo.sp:227` |
| `l4d2_dynamic_ammo_beta` | L4D2 Dynamic Ammo (dirspawn only) | `0.5` | 浮点 | ≥ 0.0 | 间隔指数，最小 0 | `l4d2_dynamic_ammo.sp:216` |
| `l4d2_dynamic_ammo_debug` | L4D2 Dynamic Ammo (dirspawn only) | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否输出调试信息 | `l4d2_dynamic_ammo.sp:221` |
| `l4d2_dynamic_ammo_enable` | L4D2 Dynamic Ammo (dirspawn only) | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用：1 开，0 关 | `l4d2_dynamic_ammo.sp:212` |
| `l4d2_dynamic_ammo_max_mult` | L4D2 Dynamic Ammo (dirspawn only) | `6.0` | 浮点 | ≥ 0.5 | 倍率上限，最小 0.5 | `l4d2_dynamic_ammo.sp:218` |
| `l4d2_dynamic_ammo_min_mult` | L4D2 Dynamic Ammo (dirspawn only) | `1.0` | 浮点 | ≥ 0.1 | 倍率下限，最小 0.1 | `l4d2_dynamic_ammo.sp:217` |
| `l4d2_dynamic_ammo_refill_mode` | L4D2 Dynamic Ammo (dirspawn only) | `1` | 整数，源码写作浮点 | 0.0 ~ 2.0 | 弹药补充模式：0 仅限上限，1 回满上限（默认），2 强制等于目标 | `l4d2_dynamic_ammo.sp:219` |
| `l4d2_dynamic_ammo_version` | L4D2 Dynamic Ammo (dirspawn only) | `1.0.1` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置，默认值为 1.0.1） | `l4d2_dynamic_ammo.sp:210` |
| `l4d2_fallen_equipments` | [L4D2] Uncommon Adjustment | `15` | 整数，源码写作浮点 | 0.0 ~ 15.0 | fallen survivor 可携带的物品：1 燃烧瓶，2 管式炸弹，4 止痛药，8 医疗包，15 全部，0 无 | `l4d2_uncommon_adjustment.sp:120` |
| `l4d2_fix_deathfall_cam_version` | [L4D2] Fix DeathFall Camera | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | `l4d_fix_deathfall_cam.sp:36` |
| `l4d2_ghost_warp_delay` | Infected Warp | `0.45` | 浮点 | 0.0 ~ 120.0 | 幽灵传送的重用间隔秒数，0.0 表示无延迟，最大 120 | `l4d2_ghost_warp.sp:63` |
| `l4d2_ghost_warp_flag` | Infected Warp | `3` | 整数，源码写作浮点 | 0.0 ~ float(eAllowAll) | 启用或禁用幽灵传送：0 禁用，1 通过 sm_warpto 命令启用，2 通过 IN_ATTACK2 按键启用 | `l4d2_ghost_warp.sp:56` |
| `l4d2_gnome_allow` | [L4D2] Healing Gnome | `1` | 整数 | 无上下界 | 0 关闭插件，1 开启插件 | `gnome.sp:183` |
| `l4d2_gnome_full` | [L4D2] Healing Gnome | `0` | 整数 | 无上下界 | 0 关闭；开启后在治疗到生命上限时移除黑白效果并补满生命，1 给临时生命，其余取值源码描述被截断 | `gnome.sp:186` |
| `l4d2_gnome_glow` | [L4D2] Healing Gnome | `200` | 整数 | 无上下界 | 地精发光的最远距离，0 关闭 | `gnome.sp:184` |
| `l4d2_gnome_glow_color` | [L4D2] Healing Gnome | `255 0 0` | 字符串或表达式 | 无上下界 | 地精发光颜色，三个 0~255 数值以空格分隔的 RGB，0 表示默认发光色 | `gnome.sp:185` |
| `l4d2_gnome_heal` | [L4D2] Healing Gnome | `1` | 整数 | 无上下界 | 是否用本插件的 cvar 来治疗手持地精的玩家，不影响治疗力场 | `gnome.sp:187` |
| `l4d2_gnome_healing_field` | [L4D2] Healing Gnome | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否治疗地精持有者周围的玩家：0 关，1 开 | `gnome.sp:196` |
| `l4d2_gnome_healing_field_amplitude` | [L4D2] Healing Gnome | `0.0` | 整数，源码写作浮点 | ≥ 0.0 | 治疗力场振幅 | `gnome.sp:207` |
| `l4d2_gnome_healing_field_color` | [L4D2] Healing Gnome | `0 255 0` | 字符串或表达式 | 无上下界 | 治疗力场颜色，三个 0~255 的 RGB 值以空格分隔，填 random 则随机生成 | `gnome.sp:202` |
| `l4d2_gnome_healing_field_duration` | [L4D2] Healing Gnome | `1.0` | 整数，源码写作浮点 | ≥ 0.0 | 治疗力场持续秒数 | `gnome.sp:205` |
| `l4d2_gnome_healing_field_end_radius` | [L4D2] Healing Gnome | `350.0` | 整数，源码写作浮点 | ≥ 0.0 | 治疗力场结束半径，同时决定治疗周围玩家的最大距离 | `gnome.sp:204` |
| `l4d2_gnome_healing_field_heal_amount` | [L4D2] Healing Gnome | `1` | 整数，源码写作浮点 | ≥ 0.0 | 在治疗力场内每次的治疗量 | `gnome.sp:198` |
| `l4d2_gnome_healing_field_heal_amount_incap` | [L4D2] Healing Gnome | `0` | 整数，源码写作浮点 | ≥ 0.0 | 倒地玩家在治疗力场内的治疗量 | `gnome.sp:199` |
| `l4d2_gnome_healing_field_heal_beacon` | [L4D2] Healing Gnome | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否生成治疗力场光柱：0 关，1 开 | `gnome.sp:201` |
| `l4d2_gnome_healing_field_refresh_time` | [L4D2] Healing Gnome | `2.0` | 整数，源码写作浮点 | ≥ 0.0 | 治疗力场重新触发治疗与光柱的间隔秒数，0 关闭 | `gnome.sp:197` |
| `l4d2_gnome_healing_field_self` | [L4D2] Healing Gnome | `1` | 整数 | 无上下界 | 0 只治疗他人，1 同时治疗自己与他人 | `gnome.sp:200` |
| `l4d2_gnome_healing_field_start_radius` | [L4D2] Healing Gnome | `100.0` | 整数，源码写作浮点 | ≥ 0.0 | 治疗力场起始半径 | `gnome.sp:203` |
| `l4d2_gnome_healing_field_width` | [L4D2] Healing Gnome | `3.0` | 整数，源码写作浮点 | ≥ 0.0 | 治疗力场宽度 | `gnome.sp:206` |
| `l4d2_gnome_max_main` | [L4D2] Healing Gnome | `100` | 整数 | 无上下界 | 治疗的主生命上限 | `gnome.sp:188` |
| `l4d2_gnome_max_temp` | [L4D2] Healing Gnome | `100.0` | 整数，源码写作浮点 | 无上下界 | 治疗的临时生命上限 | `gnome.sp:189` |
| `l4d2_gnome_modes` | [L4D2] Healing Gnome | `` | 字符串或表达式 | 无上下界 | 在这些游戏模式中启用插件，逗号分隔，留空为全部模式 | `gnome.sp:190` |
| `l4d2_gnome_modes_off` | [L4D2] Healing Gnome | `` | 字符串或表达式 | 无上下界 | 在这些游戏模式中关闭插件，逗号分隔，留空为不关闭 | `gnome.sp:191` |
| `l4d2_gnome_modes_tog` | [L4D2] Healing Gnome | `0` | 整数 | 无上下界 | 按模式位掩码启用插件：0 全部，1 战役，2 生存，4 对战，8 清道夫，可相加 | `gnome.sp:192` |
| `l4d2_gnome_random` | [L4D2] Healing Gnome | `0` | 整数 | 无上下界 | 从地图配置中随机刷出的地精数量：-1 全部，0 不刷 | `gnome.sp:193` |
| `l4d2_gnome_safe` | [L4D2] Healing Gnome | `0` | 整数 | 无上下界 | 回合开始刷出地精的位置：0 关，1 安全区内，2 装备给随机玩家 | `gnome.sp:194` |
| `l4d2_gnome_temp` | [L4D2] Healing Gnome | `-1` | 整数 | 无上下界 | -1 给临时生命，0 加到主生命；1~100 表示给临时生命的概率，否则加主生命 | `gnome.sp:195` |
| `l4d2_gnome_version` | [L4D2] Healing Gnome | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | `gnome.sp:208` |
| `l4d2_heq_checkpoint_sound` | L4D2 Horde Equaliser | `1` | 整数 | 无上下界 | 是否在检查点播放来袭尸潮声音，每击杀总量的四分之一触发一次，模拟 L4D1 行为 | `l4d2_horde_equaliser.sp:77` |
| `l4d2_heq_no_tank_horde` | L4D2 Horde Equaliser | `0` | 整数 | 无上下界 | 是否在坦克战期间让无限尸潮保持暂停 | `l4d2_horde_equaliser.sp:76` |
| `l4d2_hitsound_plus_ver` | L4D2 Hit/Kill Feedback Plus | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | `l4d2_hitsound.sp:714` |
| `l4d2_hunter_patch_bonus_damage` | L4D2 hunter patch | `0` | 整数 | 无上下界 | 是否启用飞扑附加伤害：0 游戏默认，1 总是，2 从不 | `l4d2_hunter_patch.sp:41` |
| `l4d2_hunter_patch_convert_leap` | L4D2 hunter patch | `0` | 整数 | 无上下界 | 是否把跳跃转为飞扑：0 游戏默认，1 总是，2 从不 | `l4d2_hunter_patch.sp:39` |
| `l4d2_hunter_patch_crouch_pounce` | L4D2 hunter patch | `0` | 整数 | 无上下界 | 在地面是否需要按蹲下键才能飞扑：0 游戏默认，1 总是，2 从不 | `l4d2_hunter_patch.sp:40` |
| `l4d2_hunter_patch_pounce_interrupt` | L4D2 hunter patch | `0` | 整数 | 无上下界 | 是否启用飞扑被打断：0 游戏默认，1 总是，2 从不 | `l4d2_hunter_patch.sp:42` |
| `l4d2_hunter_patch_version` | L4D2 hunter patch | `VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（默认值取自 VERSION 宏） | `l4d2_hunter_patch.sp:37` |
| `l4d2_identity_fix` | [L4D2]Character_manager | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否对真人玩家启用身份修复：0 关，1 开 | `l4d2_character_manager.sp:108` |
| `l4d2_infected_ff_allow_tank` | L4D2 Infected Friendly Fire Disable | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否不禁用坦克对其他特感的友军伤害 | `l4d2_si_ffblock.sp:42` |
| `l4d2_infected_ff_block_witch` | L4D2 Infected Friendly Fire Disable | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否禁用对 Witch 的友军伤害 | `l4d2_si_ffblock.sp:43` |
| `l4d2_infected_marker_announce_type` | L4D2 Item hint | `1` | 整数，源码写作浮点 | 0.0 ~ 3.0 | 特感标记提示显示方式：0 禁用，1 聊天，2 提示框，3 中心文本 | `l4d2_item_hint.sp:157` |
| `l4d2_infected_marker_cooldown_time` | L4D2 Item hint | `0.25` | 浮点 | ≥ 0.0 | 玩家再次使用 Look 特感标记的冷却秒数 | `l4d2_item_hint.sp:154` |
| `l4d2_infected_marker_glow_color` | L4D2 Item hint | `` | 字符串或表达式 | 无上下界 | 特感标记发光颜色，留空禁用特感标记 | `l4d2_item_hint.sp:160` |
| `l4d2_infected_marker_glow_range` | L4D2 Item hint | `2500` | 整数，源码写作浮点 | ≥ 0.0 | 特感标记发光范围 | `l4d2_item_hint.sp:159` |
| `l4d2_infected_marker_glow_timer` | L4D2 Item hint | `10.0` | 整数，源码写作浮点 | ≥ 0.0 | 特感标记发光持续时间 | `l4d2_item_hint.sp:158` |
| `l4d2_infected_marker_use_range` | L4D2 Item hint | `1800` | 整数，源码写作浮点 | ≥ 1.0 | 可使用 Look 特感标记的距离 | `l4d2_item_hint.sp:155` |
| `l4d2_infected_marker_use_sound` | L4D2 Item hint | `items/suitchargeok1.wav` | 字符串或表达式 | 无上下界 | 特感标记音效路径，留空为关闭 | `l4d2_item_hint.sp:156` |
| `l4d2_infected_marker_witch_enable` | L4D2 Item hint | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 为 1 时允许对 Witch 使用 Look 特感标记 | `l4d2_item_hint.sp:161` |
| `l4d2_item_hint_announce_type` | L4D2 Item hint | `0` | 整数，源码写作浮点 | 0.0 ~ 3.0 | 物品提示显示方式：0 禁用，1 聊天，2 提示框，3 中心文本 | `l4d2_item_hint.sp:135` |
| `l4d2_item_hint_cooldown_time` | L4D2 Item hint | `1.0` | 整数，源码写作浮点 | ≥ 0.0 | 玩家再次使用 Look 物品提示的冷却秒数 | `l4d2_item_hint.sp:132` |
| `l4d2_item_hint_glow_color` | L4D2 Item hint | `0 255 255` | 字符串或表达式 | 无上下界 | 物品发光颜色，0-255 RGB 以空格分隔，留空禁用物品发光 | `l4d2_item_hint.sp:138` |
| `l4d2_item_hint_glow_range` | L4D2 Item hint | `800` | 整数，源码写作浮点 | ≥ 0.0 | 物品发光范围 | `l4d2_item_hint.sp:137` |
| `l4d2_item_hint_glow_timer` | L4D2 Item hint | `30.0` | 整数，源码写作浮点 | ≥ 0.0 | 物品发光持续时间 | `l4d2_item_hint.sp:136` |
| `l4d2_item_hint_use_range` | L4D2 Item hint | `150` | 整数，源码写作浮点 | ≥ 1.0 | 玩家可使用 Look 物品提示的距离 | `l4d2_item_hint.sp:133` |
| `l4d2_item_hint_use_sound` | L4D2 Item hint | `` | 字符串或表达式 | 无上下界 | 物品提示音效路径，留空为关闭 | `l4d2_item_hint.sp:134` |
| `l4d2_item_instructorhint_color` | L4D2 Item hint | `0 255 255` | 字符串或表达式 | 无上下界 | 标记物品的教学提示颜色 | `l4d2_item_hint.sp:140` |
| `l4d2_item_instructorhint_enable` | L4D2 Item hint | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 为 1 时在标记的物品上创建教学提示 | `l4d2_item_hint.sp:139` |
| `l4d2_item_instructorhint_icon` | L4D2 Item hint | `icon_interact` | 字符串或表达式 | 无上下界 | 标记物品的教学提示图标名 | `l4d2_item_hint.sp:141` |
| `l4d2_jimmy_health_multiplier` | [L4D2] Uncommon Adjustment | `20.0` | 整数，源码写作浮点 | ≥ 0.0 | Jimmy Gibbs Jr. 的生命倍率 | `l4d2_uncommon_adjustment.sp:113` |
| `l4d2_jimmy_screen_splatter` | [L4D2] Uncommon Adjustment | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | Jimmy Gibbs Jr. 是否能遮蔽玩家屏幕 | `l4d2_uncommon_adjustment.sp:149` |
| `l4d2_jimmy_sense_flag` | [L4D2] Uncommon Adjustment | `0` | 开关 0 或 1 | 0.0 ~ 3.0 | Jimmy Gibbs Jr. 能否听见或闻到吸引物：0 都不能，1 听，2 闻，3 两者 | `l4d2_uncommon_adjustment.sp:97` |
| `l4d2_jumpcap_block_time` | L4D2 Jockey Jump-Cap Patch | `3.0` | 整数，源码写作浮点 | 1.0 ~ 10.0 | 设置 Jockey 跳扑被封锁的持续时间，单位秒，范围 1~10 | `l4d2_jockey_jumpcap_patch.sp:40` |
| `l4d2_ladder_boost_clamp_exit_speed` | [L4D2] Infected Ladder Speed Boost | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 特感离开梯子时是否把水平速度限制到当前走路速度，防止 10 倍速度带出 | `l4d2_ai_ladder_boost.sp:103` |
| `l4d2_ladder_boost_cooldown` | [L4D2] Infected Ladder Speed Boost | `3.0` | 整数，源码写作浮点 | 1.0 ~ 10.0 | 被看见后禁用未视野加速的冷却秒数，范围 1~10 | `l4d2_ai_ladder_boost.sp:90` |
| `l4d2_ladder_boost_debug` | [L4D2] Infected Ladder Speed Boost | `0` | 整数，源码写作浮点 | 0.0 ~ 2.0 | 调试模式：0 关闭，1 基本调试，2 详细调试 | `l4d2_ai_ladder_boost.sp:92` |
| `l4d2_ladder_boost_detection` | [L4D2] Infected Ladder Speed Boost | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 检测方法：0 威胁感知加射线，1 传统 FOV 加射线 | `l4d2_ai_ladder_boost.sp:89` |
| `l4d2_ladder_boost_enabled` | [L4D2] Infected Ladder Speed Boost | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用未被生还者看见时 AI 特感爬梯加速：1 开，0 关 | `l4d2_ai_ladder_boost.sp:87` |
| `l4d2_ladder_boost_multiplier` | [L4D2] Infected Ladder Speed Boost | `10.0` | 整数，源码写作浮点 | 1.0 ~ 20.0 | 未被看见时的爬梯速度倍数，范围 1~20 | `l4d2_ai_ladder_boost.sp:88` |
| `l4d2_ladder_boost_tank` | [L4D2] Infected Ladder Speed Boost | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否允许该通用插件加速 Tank 爬梯 | `l4d2_ai_ladder_boost.sp:97` |
| `l4d2_ladder_boost_use_sdkhook` | [L4D2] Infected Ladder Speed Boost | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 兼容旧配置：该项已停用，AI 特感固定使用 Timer 检测 | `l4d2_ai_ladder_boost.sp:91` |
| `l4d2_ladder_patch_version` | [L4D2] Ladder Server Crash - Patch Fix | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | `l4d2_ladder_patch.sp:85` |
| `l4d2_lmm_reservation_modify_flags` | L4D2 Lobby match manager | `7` | 整数 | 无上下界 | 修改客户端提交给服务器的 lobby 设置，见 RMFLAG_*，需 unreserve_type 不等于 1 | `l4d2_lobby_match_manager.sp:105` |
| `l4d2_lmm_unreserve_type` | L4D2 Lobby match manager | `0` | 整数 | 无上下界 | 直接加入不创建预留：0 保留原有预留，1 Anne 模式下保留原预留至其他情况，其余描述被截断 | `l4d2_lobby_match_manager.sp:104` |
| `l4d2_lobby_match_manager_version` | L4D2 Lobby match manager | `VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（默认值取自 VERSION 宏） | `l4d2_lobby_match_manager.sp:103` |
| `l4d2_m2_hunter_penalty` | L4D2 M2 Control | `0` | 整数 | 无上下界 | 推 Hunter 时增加的惩罚值 | `l4d2_m2_control_eq.sp:51` |
| `l4d2_m2_jockey_penalty` | L4D2 M2 Control | `0` | 整数 | 无上下界 | 推 Jockey 时增加的惩罚值 | `l4d2_m2_control_eq.sp:52` |
| `l4d2_m2_smoker_penalty` | L4D2 M2 Control | `0` | 整数 | 无上下界 | 推 Smoker 时增加的惩罚值 | `l4d2_m2_control_eq.sp:53` |
| `l4d2_manage_people` | [L4D2]Character_manager | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否同时管理真人：0 关，1 开，接管 Bot 时会覆盖身份修复 | `l4d2_character_manager.sp:110` |
| `l4d2_map_vote_version` | l4d2_map_vote.sp（myinfo 缺 name，用文件名代替） | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | `l4d2_map_vote.sp:109` |
| `l4d2_mapvote_autoreload` | l4d2_map_vote.sp（myinfo 缺 name，用文件名代替） | `2` | 整数 | 无上下界 | 输入 !mapvote 时是否自动刷新 VPK 与战役列表：0 关，1 仅管理员触发，2 所有人触发 | `l4d2_map_vote.sp:111` |
| `l4d2_mapvote_reload_cd` | l4d2_map_vote.sp（myinfo 缺 name，用文件名代替） | `10.0` | 整数，源码写作浮点 | 无上下界 | 自动刷新的冷却秒数，避免被频繁触发导致卡顿 | `l4d2_map_vote.sp:116` |
| `l4d2_mapvote_versus_from_coop` | l4d2_map_vote.sp（myinfo 缺 name，用文件名代替） | `1` | 整数 | 无上下界 | 为缺少 versus 的战役临时注入 modes/versus：0 关，1 仅 Anne 派生 cfg，2 所有 cfg | `l4d2_map_vote.sp:120` |
| `l4d2_melee_damage_charger` | L4D2 Melee Damage Fix&Control | `-1.0` | 整数，源码写作浮点 | 无上下界 | 每次挥击对 Charger 的近战伤害，0 或负值表示关闭 | `l4d2_melee_damage_control.sp:92` |
| `l4d2_melee_damage_fix` | L4D2 Melee Damage Fix&Control | `1.0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否修复近战武器对感染者伤害不正确的问题，即伤害不再取决于命中部位 | `l4d2_melee_damage_control.sp:77` |
| `l4d2_melee_damage_tank_nerf` | L4D2 Melee Damage Fix&Control | `-1.0` | 整数，源码写作浮点 | ≤ 100.0 | 近战对 Tank 伤害的削弱百分比，0 或负值表示关闭 | `l4d2_melee_damage_control.sp:85` |
| `l4d2_melee_drop_method` | Shove Shenanigans - REVAMPED | `2` | 整数 | 无上下界 | 坦克拳击手持近战武器的生还者时的处理：0 无动作，1 掉落近战武器，2 强制处理，源码描述在此被截断 | `l4d2_melee_shenanigans.sp:23` |
| `l4d2_melee_spawn` | l4d2 melee spawn control | `` | 字符串或表达式 | 无上下界 | 解锁的近战武器列表，逗号分隔，留空不作修改 | `l4d2_melee_spawn_control.sp:65` |
| `l4d2_MITSR_Amount` | Melee In The Saferoom | `8` | 整数 | 无上下界 | 当 Random 为 1 时刷出的武器数量 | `MeleeInTheSafeRoom.sp:75` |
| `l4d2_MITSR_BaseballBat` | Melee In The Saferoom | `1` | 整数 | 无上下界 | 刷出的棒球棍数量，需 Random 为 0 | `MeleeInTheSafeRoom.sp:76` |
| `l4d2_MITSR_CricketBat` | Melee In The Saferoom | `1` | 整数 | 无上下界 | 刷出的板球棍数量，需 Random 为 0 | `MeleeInTheSafeRoom.sp:77` |
| `l4d2_MITSR_Crowbar` | Melee In The Saferoom | `1` | 整数 | 无上下界 | 刷出的撬棍数量，需 Random 为 0 | `MeleeInTheSafeRoom.sp:78` |
| `l4d2_MITSR_ElecGuitar` | Melee In The Saferoom | `1` | 整数 | 无上下界 | 刷出的电吉他数量，需 Random 为 0 | `MeleeInTheSafeRoom.sp:79` |
| `l4d2_MITSR_Enabled` | Melee In The Saferoom | `1` | 整数 | 无上下界 | 是否启用插件 | `MeleeInTheSafeRoom.sp:73` |
| `l4d2_MITSR_FireAxe` | Melee In The Saferoom | `1` | 整数 | 无上下界 | 刷出的消防斧数量，需 Random 为 0 | `MeleeInTheSafeRoom.sp:80` |
| `l4d2_MITSR_FryingPan` | Melee In The Saferoom | `1` | 整数 | 无上下界 | 刷出的平底锅数量，需 Random 为 0 | `MeleeInTheSafeRoom.sp:81` |
| `l4d2_MITSR_GolfClub` | Melee In The Saferoom | `1` | 整数 | 无上下界 | 刷出的高尔夫球杆数量，需 Random 为 0 | `MeleeInTheSafeRoom.sp:82` |
| `l4d2_MITSR_Katana` | Melee In The Saferoom | `1` | 整数 | 无上下界 | 刷出的武士刀数量，需 Random 为 0 | `MeleeInTheSafeRoom.sp:84` |
| `l4d2_MITSR_Knife` | Melee In The Saferoom | `1` | 整数 | 无上下界 | 刷出的刀数量，需 Random 为 0 | `MeleeInTheSafeRoom.sp:83` |
| `l4d2_MITSR_Machete` | Melee In The Saferoom | `1` | 整数 | 无上下界 | 刷出的砍刀数量，需 Random 为 0 | `MeleeInTheSafeRoom.sp:85` |
| `l4d2_MITSR_Random` | Melee In The Saferoom | `1` | 整数 | 无上下界 | 随机刷武器为 1，使用自定义列表为 0 | `MeleeInTheSafeRoom.sp:74` |
| `l4d2_MITSR_RiotShield` | Melee In The Saferoom | `1` | 整数 | 无上下界 | 刷出的防暴盾数量，需 Random 为 0 | `MeleeInTheSafeRoom.sp:86` |
| `l4d2_MITSR_Tonfa` | Melee In The Saferoom | `1` | 整数 | 无上下界 | 刷出的拐棍数量，需 Random 为 0 | `MeleeInTheSafeRoom.sp:87` |
| `l4d2_MITSR_Version` | Melee In The Saferoom | `VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（默认值取自 VERSION 宏） | `MeleeInTheSafeRoom.sp:72` |
| `l4d2_mudman_crouch_run` | [L4D2] Uncommon Adjustment | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 泥人是否能边跑边蹲下 | `l4d2_uncommon_adjustment.sp:135` |
| `l4d2_mudman_screen_splatter` | [L4D2] Uncommon Adjustment | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 泥人是否能遮蔽玩家屏幕 | `l4d2_uncommon_adjustment.sp:142` |
| `l4d2_nativevote_initiator_auto_voteyes` | L4D2 Native vote | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 为 1 时发起者自动投赞成票 | `l4d2_nativevote.sp:75` |
| `l4d2_nativevote_initiator_auto_voteyes` | L4D2 Native vote | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 为 1 时发起者自动投赞成票 | `l4d2_nativevote.sp:75` |
| `l4d2_nativevote_version` | L4D2 Native vote | `VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | `l4d2_nativevote.sp:74` |
| `l4d2_nativevote_version` | L4D2 Native vote | `VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（默认值取自 VERSION 宏） | `l4d2_nativevote.sp:74` |
| `l4d2_nav_variant_config` | L4D2 Nav Variant Loader | `DEFAULT_CONFIG` | 字符串或表达式 | 无上下界 | 导航变体 KeyValues 配置位于 addons/sourcemod 下的路径 | `l4d2_nav_variant.sp:78` |
| `l4d2_nav_variant_debug` | L4D2 Nav Variant Loader | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否把导航变体决策打印到服务器日志 | `l4d2_nav_variant.sp:81` |
| `l4d2_nav_variant_enable` | L4D2 Nav Variant Loader | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用导航网格变体重定向 | `l4d2_nav_variant.sp:76` |
| `l4d2_nav_variant_name` | L4D2 Nav Variant Loader | `DEFAULT_VARIANT` | 字符串或表达式 | 无上下界 | 当前 stripper 路径符合条件时启用的导航变体名，留空使用地图默认 nav | `l4d2_nav_variant.sp:77` |
| `l4d2_nav_variant_required_cfg` | L4D2 Nav Variant Loader | `` | 字符串或表达式 | 无上下界 | 可选的 l4d_ready_cfg_name 子串守卫，留空则仅由 nav_variant_name 控制重定向 | `l4d2_nav_variant.sp:79` |
| `l4d2_nav_variant_stripper_path` | L4D2 Nav Variant Loader | `DEFAULT_STRIPPER_PATH` | 字符串或表达式 | 无上下界 | 要求 stripper_cfg_path 等于该路径，留空关闭该守卫 | `l4d2_nav_variant.sp:80` |
| `l4d2_nav_variant_version` | L4D2 Nav Variant Loader | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | `l4d2_nav_variant.sp:82` |
| `l4d2_notankautoaim` | L4D2 Tank Claw Fix | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否移除坦克爪击未公开的自动瞄准能力：1 启用，0 禁用 | `l4d2_notankautoaim.sp:28` |
| `l4d2_null_cusercmd_fix_version` | L4D2 Lag Compensation Null CUserCmd fix | `VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（默认值取自 VERSION 宏） | `l4d2_null_cusercmd_fix.sp:20` |
| `l4d2_pickup_version` | [L4D & 2] Pick-up Changes | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | `l4d2_pickup.sp:213` |
| `l4d2_playtime_apikey` | VeteransOnly | `C7B3FC46E6E6D5C87700963F0688FCB4` | 字符串或表达式 | 无上下界 | Steam 开发者 Web API key | `veterans.sp:112` |
| `l4d2_pz_ladder_boost` | L4D2 SI LADDER BOOSTER | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否允许真人特感在梯子上加速，范围 0~1 | `l4d2_si_ladder_booster.sp:31` |
| `l4d2_pz_ladder_boost` | [L4D2] Infected Ladder Speed Boost | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 兼容旧 l4d2_si_ladder_booster：已停用，真人特感不会获得加速 | `l4d2_ai_ladder_boost.sp:95` |
| `l4d2_reload_fix_give` | [L4D2] No Reload Animation Fix | `1` | 整数 | 无上下界 | 使用 give 命令替换同类型武器时是否把弹药转移给新武器：0 否，1 是 | `l4d2_reload_fix.sp:168` |
| `l4d2_reload_fix_version` | [L4D2] No Reload Animation Fix | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | `l4d2_reload_fix.sp:169` |
| `l4d2_remove_pillsLimit` | Remove Kits or replace kits and remove defib | `4` | 整数 | 无上下界 | 限制止痛药最多出现数量 | `remove.sp:29` |
| `l4d2_replace_magnum_incap` | Magnum incap remover | `1.0` | 整数，源码写作浮点 | 无上下界 | 倒地时把手枪替换为单持（1）或双持（2），0 关闭 | `l4d2_magnum_incap.sp:29` |
| `l4d2_riotcop_armor` | [L4D2] Uncommon Adjustment | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 防暴警是否拥有可抵挡正面伤害的护甲 | `l4d2_uncommon_adjustment.sp:128` |
| `l4d2_roadworker_sense_flag` | [L4D2] Uncommon Adjustment | `0` | 开关 0 或 1 | 0.0 ~ 3.0 | 修路工能否听见或闻到吸引物：0 都不能，1 听管式炸弹与小丑，2 闻呕吐瓶，3 两者 | `l4d2_uncommon_adjustment.sp:89` |
| `l4d2_rock_hurt_capper` | [L4D2] Rock Trace Unblock | `5` | 整数，源码写作浮点 | 0.0 ~ 7.0 | 是否在控制生效前先伤害控制者：1 Hunter，2 Jockey，4 Charger，7 全部，0 关闭 | `l4d2_rock_trace_unblock.sp:90` |
| `l4d2_rock_jockey_dismount` | [L4D2] Rock Trace Unblock | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否强制 Jockey 从被石头击中的生还者身上下来：1 开，0 关 | `l4d2_rock_trace_unblock.sp:82` |
| `l4d2_rock_trace_unblock_flag` | [L4D2] Rock Trace Unblock | `5` | 整数，源码写作浮点 | 0.0 ~ 31.0 | 位掩码：阻止特感挡住石头半径检测，1 所有站立的特感，2 扑中的，4 其余被截断 | `l4d2_rock_trace_unblock.sp:74` |
| `l4d2_script_cmd_swap_version` | [L4D2] Script Command Swap - Mem Leak Fix | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | `l4d2_script_cmd_swap.sp:34` |
| `l4d2_scripted_hud_enable` | l4d2_scripted_hud.sp（myinfo 缺 name，用文件名代替） | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用插件：0 关，1 开 | `l4d2_scripted_hud.sp:711` |
| `l4d2_scripted_hud_hud1_background` | l4d2_scripted_hud.sp（myinfo 缺 name，用文件名代替） | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 文本是否显示黑色半透明背景 | `l4d2_scripted_hud.sp:720` |
| `l4d2_scripted_hud_hud1_beep` | l4d2_scripted_hud.sp（myinfo 缺 name，用文件名代替） | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 闪烁时是否播放提示音，需闪烁开关为 1 | `l4d2_scripted_hud.sp:718` |
| `l4d2_scripted_hud_hud1_blink` | l4d2_scripted_hud.sp（myinfo 缺 name，用文件名代替） | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 文本是否在白色与红色间闪烁：0 关，1 开 | `l4d2_scripted_hud.sp:717` |
| `l4d2_scripted_hud_hud1_blink_tank` | l4d2_scripted_hud.sp（myinfo 缺 name，用文件名代替） | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 有 Tank 存活时文本是否在白色与红色间闪烁：0 关，1 开 | `l4d2_scripted_hud.sp:716` |
| `l4d2_scripted_hud_hud1_flag_debug` | l4d2_scripted_hud.sp（myinfo 缺 name，用文件名代替） | `0` | 整数，源码写作浮点 | 0.0 ~ 32767.0 | 覆盖 HUD flag，仅调试用，0 关闭 | `l4d2_scripted_hud.sp:722` |
| `l4d2_scripted_hud_hud1_height` | l4d2_scripted_hud.sp（myinfo 缺 name，用文件名代替） | `0.026` | 浮点 | 0.0 ~ 2.0 | 文本区域高度 | `l4d2_scripted_hud.sp:734` |
| `l4d2_scripted_hud_hud1_team` | l4d2_scripted_hud.sp（myinfo 缺 name，用文件名代替） | `0` | 整数，源码写作浮点 | 0.0 ~ 2.0 | 哪些队伍能看到文本：0 全部，1 生还者，2 特感 | `l4d2_scripted_hud.sp:721` |
| `l4d2_scripted_hud_hud1_text` | l4d2_scripted_hud.sp（myinfo 缺 name，用文件名代替） | `` | 字符串或表达式 | 无上下界 | 要在 HUD 上显示的文本，留空则使用插件内置的预定义文本 | `l4d2_scripted_hud.sp:714` |
| `l4d2_scripted_hud_hud1_text_align` | l4d2_scripted_hud.sp（myinfo 缺 name，用文件名代替） | `1` | 整数，源码写作浮点 | 1.0 ~ 3.0 | 文本水平对齐：1 左，2 中，3 右 | `l4d2_scripted_hud.sp:715` |
| `l4d2_scripted_hud_hud1_visible` | l4d2_scripted_hud.sp（myinfo 缺 name，用文件名代替） | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 文本是否可见：0 关，1 开 | `l4d2_scripted_hud.sp:719` |
| `l4d2_scripted_hud_hud1_width` | l4d2_scripted_hud.sp（myinfo 缺 name，用文件名代替） | `1.0` | 整数，源码写作浮点 | 0.0 ~ 2.0 | 文本区域宽度 | `l4d2_scripted_hud.sp:733` |
| `l4d2_scripted_hud_hud1_x` | l4d2_scripted_hud.sp（myinfo 缺 name，用文件名代替） | `0.05` | 浮点 | -1.0 ~ 1.0 | 文本水平位置 -1.0~1.0，小于 0 可能被屏幕裁切 | `l4d2_scripted_hud.sp:723` |
| `l4d2_scripted_hud_hud1_x_direction` | l4d2_scripted_hud.sp（myinfo 缺 name，用文件名代替） | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 文本水平移动方向：0 从右到左，1 从左到右 | `l4d2_scripted_hud.sp:727` |
| `l4d2_scripted_hud_hud1_x_max` | l4d2_scripted_hud.sp（myinfo 缺 name，用文件名代替） | `1.0` | 整数，源码写作浮点 | -1.0 ~ 1.0 | 动画时 HUD 可到达的水平最大位置 | `l4d2_scripted_hud.sp:731` |
| `l4d2_scripted_hud_hud1_x_min` | l4d2_scripted_hud.sp（myinfo 缺 name，用文件名代替） | `0.0` | 整数，源码写作浮点 | -1.0 ~ 1.0 | 动画时 HUD 可到达的水平最小位置 | `l4d2_scripted_hud.sp:729` |
| `l4d2_scripted_hud_hud1_x_speed` | l4d2_scripted_hud.sp（myinfo 缺 name，用文件名代替） | `0.0` | 整数，源码写作浮点 | 0.0 ~ 2.0 | 文本水平移动动画速度，0 关闭 | `l4d2_scripted_hud.sp:725` |
| `l4d2_scripted_hud_hud1_y` | l4d2_scripted_hud.sp（myinfo 缺 name，用文件名代替） | `0.0` | 整数，源码写作浮点 | -1.0 ~ 1.0 | 文本垂直位置 -1.0~1.0，小于 0 可能被屏幕裁切 | `l4d2_scripted_hud.sp:724` |
| `l4d2_scripted_hud_hud1_y_direction` | l4d2_scripted_hud.sp（myinfo 缺 name，用文件名代替） | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 文本垂直移动方向：0 从上到下，1 从下到上 | `l4d2_scripted_hud.sp:728` |
| `l4d2_scripted_hud_hud1_y_max` | l4d2_scripted_hud.sp（myinfo 缺 name，用文件名代替） | `1.0` | 整数，源码写作浮点 | -1.0 ~ 1.0 | 动画时 HUD 可到达的垂直最大位置 | `l4d2_scripted_hud.sp:732` |
| `l4d2_scripted_hud_hud1_y_min` | l4d2_scripted_hud.sp（myinfo 缺 name，用文件名代替） | `0.0` | 整数，源码写作浮点 | -1.0 ~ 1.0 | 动画时 HUD 可到达的垂直最小位置 | `l4d2_scripted_hud.sp:730` |
| `l4d2_scripted_hud_hud1_y_speed` | l4d2_scripted_hud.sp（myinfo 缺 name，用文件名代替） | `0.0` | 整数，源码写作浮点 | 0.0 ~ 2.0 | 文本垂直移动动画速度，0 关闭 | `l4d2_scripted_hud.sp:726` |
| `l4d2_scripted_hud_hud2_background` | l4d2_scripted_hud.sp（myinfo 缺 name，用文件名代替） | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | HUD 槽 2 文本是否显示黑色半透明背景 | `l4d2_scripted_hud.sp:741` |
| `l4d2_scripted_hud_hud2_beep` | l4d2_scripted_hud.sp（myinfo 缺 name，用文件名代替） | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | HUD 槽 2 闪烁时是否播放提示音，需闪烁开关为 1 | `l4d2_scripted_hud.sp:739` |
| `l4d2_scripted_hud_hud2_blink` | l4d2_scripted_hud.sp（myinfo 缺 name，用文件名代替） | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | HUD 槽 2 文本是否白红闪烁：0 关，1 开 | `l4d2_scripted_hud.sp:738` |
| `l4d2_scripted_hud_hud2_blink_tank` | l4d2_scripted_hud.sp（myinfo 缺 name，用文件名代替） | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | HUD 槽 2 在有 Tank 存活时是否白红闪烁：0 关，1 开 | `l4d2_scripted_hud.sp:737` |
| `l4d2_scripted_hud_hud2_flag_debug` | l4d2_scripted_hud.sp（myinfo 缺 name，用文件名代替） | `0` | 整数，源码写作浮点 | 0.0 ~ 32767.0 | 覆盖 HUD 槽 2 的 flag，仅调试用，0 关闭 | `l4d2_scripted_hud.sp:743` |
| `l4d2_scripted_hud_hud2_height` | l4d2_scripted_hud.sp（myinfo 缺 name，用文件名代替） | `0.026` | 浮点 | 0.0 ~ 2.0 | HUD 槽 2 文本区域高度 | `l4d2_scripted_hud.sp:755` |
| `l4d2_scripted_hud_hud2_team` | l4d2_scripted_hud.sp（myinfo 缺 name，用文件名代替） | `0` | 整数，源码写作浮点 | 0.0 ~ 2.0 | 哪些队伍能看到 HUD 槽 2：0 全部，1 生还者，2 特感 | `l4d2_scripted_hud.sp:742` |
| `l4d2_scripted_hud_hud2_text` | l4d2_scripted_hud.sp（myinfo 缺 name，用文件名代替） | `` | 字符串或表达式 | 无上下界 | HUD 槽 2 显示文本，留空则使用插件内置的预定义文本 | `l4d2_scripted_hud.sp:735` |
| `l4d2_scripted_hud_hud2_text_align` | l4d2_scripted_hud.sp（myinfo 缺 name，用文件名代替） | `1` | 整数，源码写作浮点 | 1.0 ~ 3.0 | HUD 槽 2 水平对齐：1 左，2 中，3 右 | `l4d2_scripted_hud.sp:736` |
| `l4d2_scripted_hud_hud2_visible` | l4d2_scripted_hud.sp（myinfo 缺 name，用文件名代替） | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | HUD 槽 2 文本是否可见：0 关，1 开 | `l4d2_scripted_hud.sp:740` |
| `l4d2_scripted_hud_hud2_width` | l4d2_scripted_hud.sp（myinfo 缺 name，用文件名代替） | `1.0` | 整数，源码写作浮点 | 0.0 ~ 2.0 | HUD 槽 2 文本区域宽度 | `l4d2_scripted_hud.sp:754` |
| `l4d2_scripted_hud_hud2_x` | l4d2_scripted_hud.sp（myinfo 缺 name，用文件名代替） | `0.65` | 浮点 | -1.0 ~ 1.0 | HUD 槽 2 文本水平位置 -1.0~1.0，小于 0 可能被裁切 | `l4d2_scripted_hud.sp:744` |
| `l4d2_scripted_hud_hud2_x_direction` | l4d2_scripted_hud.sp（myinfo 缺 name，用文件名代替） | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | HUD 槽 2 文本水平移动方向：0 从左到右，1 从右到左 | `l4d2_scripted_hud.sp:748` |
| `l4d2_scripted_hud_hud2_x_max` | l4d2_scripted_hud.sp（myinfo 缺 name，用文件名代替） | `1.0` | 整数，源码写作浮点 | -1.0 ~ 1.0 | HUD 槽 2 动画可到达的水平最大位置 | `l4d2_scripted_hud.sp:752` |
| `l4d2_scripted_hud_hud2_x_min` | l4d2_scripted_hud.sp（myinfo 缺 name，用文件名代替） | `0.0` | 整数，源码写作浮点 | -1.0 ~ 1.0 | HUD 槽 2 动画可到达的水平最小位置 | `l4d2_scripted_hud.sp:750` |
| `l4d2_scripted_hud_hud2_x_speed` | l4d2_scripted_hud.sp（myinfo 缺 name，用文件名代替） | `0.0` | 整数，源码写作浮点 | 0.0 ~ 2.0 | HUD 槽 2 文本水平移动动画速度，0 关闭 | `l4d2_scripted_hud.sp:746` |
| `l4d2_scripted_hud_hud2_y` | l4d2_scripted_hud.sp（myinfo 缺 name，用文件名代替） | `0.00` | 整数，源码写作浮点 | -1.0 ~ 1.0 | HUD 槽 2 文本垂直位置 -1.0~1.0，小于 0 可能被裁切 | `l4d2_scripted_hud.sp:745` |
| `l4d2_scripted_hud_hud2_y_direction` | l4d2_scripted_hud.sp（myinfo 缺 name，用文件名代替） | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | HUD 槽 2 文本垂直移动方向：0 从上到下，1 从下到上 | `l4d2_scripted_hud.sp:749` |
| `l4d2_scripted_hud_hud2_y_max` | l4d2_scripted_hud.sp（myinfo 缺 name，用文件名代替） | `1.0` | 整数，源码写作浮点 | -1.0 ~ 1.0 | HUD 槽 2 动画可到达的垂直最大位置 | `l4d2_scripted_hud.sp:753` |
| `l4d2_scripted_hud_hud2_y_min` | l4d2_scripted_hud.sp（myinfo 缺 name，用文件名代替） | `0.0` | 整数，源码写作浮点 | -1.0 ~ 1.0 | HUD 槽 2 动画可到达的垂直最小位置 | `l4d2_scripted_hud.sp:751` |
| `l4d2_scripted_hud_hud2_y_speed` | l4d2_scripted_hud.sp（myinfo 缺 name，用文件名代替） | `0.0` | 整数，源码写作浮点 | 0.0 ~ 2.0 | HUD 槽 2 文本垂直移动动画速度，0 关闭 | `l4d2_scripted_hud.sp:747` |
| `l4d2_scripted_hud_hud3_background` | l4d2_scripted_hud.sp（myinfo 缺 name，用文件名代替） | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | HUD 槽 3 文本是否显示黑色半透明背景 | `l4d2_scripted_hud.sp:762` |
| `l4d2_scripted_hud_hud3_beep` | l4d2_scripted_hud.sp（myinfo 缺 name，用文件名代替） | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | HUD 槽 3 闪烁时是否播放提示音，需闪烁开关为 1 | `l4d2_scripted_hud.sp:760` |
| `l4d2_scripted_hud_hud3_blink` | l4d2_scripted_hud.sp（myinfo 缺 name，用文件名代替） | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | HUD 槽 3 文本是否白红闪烁：0 关，1 开 | `l4d2_scripted_hud.sp:759` |
| `l4d2_scripted_hud_hud3_blink_tank` | l4d2_scripted_hud.sp（myinfo 缺 name，用文件名代替） | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | HUD 槽 3 在有 Tank 存活时是否白红闪烁：0 关，1 开 | `l4d2_scripted_hud.sp:758` |
| `l4d2_scripted_hud_hud3_flag_debug` | l4d2_scripted_hud.sp（myinfo 缺 name，用文件名代替） | `0` | 整数，源码写作浮点 | 0.0 ~ 32767.0 | 覆盖 HUD 槽 3 的 flag，仅调试用，0 关闭 | `l4d2_scripted_hud.sp:764` |
| `l4d2_scripted_hud_hud3_height` | l4d2_scripted_hud.sp（myinfo 缺 name，用文件名代替） | `0.026` | 浮点 | 0.0 ~ 2.0 | HUD 槽 3 文本区域高度 | `l4d2_scripted_hud.sp:776` |
| `l4d2_scripted_hud_hud3_team` | l4d2_scripted_hud.sp（myinfo 缺 name，用文件名代替） | `0` | 整数，源码写作浮点 | 0.0 ~ 2.0 | 哪些队伍能看到 HUD 槽 3：0 全部，1 生还者，2 特感 | `l4d2_scripted_hud.sp:763` |
| `l4d2_scripted_hud_hud3_text` | l4d2_scripted_hud.sp（myinfo 缺 name，用文件名代替） | `` | 字符串或表达式 | 无上下界 | HUD 槽 3 显示文本，留空则使用插件内置的预定义文本 | `l4d2_scripted_hud.sp:756` |
| `l4d2_scripted_hud_hud3_text_align` | l4d2_scripted_hud.sp（myinfo 缺 name，用文件名代替） | `1` | 整数，源码写作浮点 | 1.0 ~ 3.0 | HUD 槽 3 水平对齐：1 左，2 中，3 右 | `l4d2_scripted_hud.sp:757` |
| `l4d2_scripted_hud_hud3_visible` | l4d2_scripted_hud.sp（myinfo 缺 name，用文件名代替） | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | HUD 槽 3 是否允许玩家显示该槽位：0 全局禁用，1 按玩家偏好 | `l4d2_scripted_hud.sp:761` |
| `l4d2_scripted_hud_hud3_width` | l4d2_scripted_hud.sp（myinfo 缺 name，用文件名代替） | `1.5` | 浮点 | 0.0 ~ 2.0 | HUD 槽 3 文本区域宽度 | `l4d2_scripted_hud.sp:775` |
| `l4d2_scripted_hud_hud3_x` | l4d2_scripted_hud.sp（myinfo 缺 name，用文件名代替） | `0.8` | 浮点 | -1.0 ~ 1.0 | HUD 槽 3 文本水平位置 -1.0~1.0，小于 0 可能被裁切 | `l4d2_scripted_hud.sp:765` |
| `l4d2_scripted_hud_hud3_x_direction` | l4d2_scripted_hud.sp（myinfo 缺 name，用文件名代替） | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | HUD 槽 3 文本水平移动方向：0 从左到右，1 从右到左 | `l4d2_scripted_hud.sp:769` |
| `l4d2_scripted_hud_hud3_x_max` | l4d2_scripted_hud.sp（myinfo 缺 name，用文件名代替） | `1.0` | 整数，源码写作浮点 | -1.0 ~ 1.0 | HUD 槽 3 动画可到达的水平最大位置 | `l4d2_scripted_hud.sp:773` |
| `l4d2_scripted_hud_hud3_x_min` | l4d2_scripted_hud.sp（myinfo 缺 name，用文件名代替） | `0.0` | 整数，源码写作浮点 | -1.0 ~ 1.0 | HUD 槽 3 动画可到达的水平最小位置 | `l4d2_scripted_hud.sp:771` |
| `l4d2_scripted_hud_hud3_x_speed` | l4d2_scripted_hud.sp（myinfo 缺 name，用文件名代替） | `0.0` | 整数，源码写作浮点 | 0.0 ~ 2.0 | HUD 槽 3 文本水平移动动画速度，0 关闭 | `l4d2_scripted_hud.sp:767` |
| `l4d2_scripted_hud_hud3_y` | l4d2_scripted_hud.sp（myinfo 缺 name，用文件名代替） | `0.11` | 浮点 | -1.0 ~ 1.0 | HUD 槽 3 文本垂直位置 -1.0~1.0，小于 0 可能被裁切 | `l4d2_scripted_hud.sp:766` |
| `l4d2_scripted_hud_hud3_y_direction` | l4d2_scripted_hud.sp（myinfo 缺 name，用文件名代替） | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | HUD 槽 3 文本垂直移动方向：0 从上到下，1 从下到上 | `l4d2_scripted_hud.sp:770` |
| `l4d2_scripted_hud_hud3_y_max` | l4d2_scripted_hud.sp（myinfo 缺 name，用文件名代替） | `1.0` | 整数，源码写作浮点 | -1.0 ~ 1.0 | HUD 槽 3 动画可到达的垂直最大位置 | `l4d2_scripted_hud.sp:774` |
| `l4d2_scripted_hud_hud3_y_min` | l4d2_scripted_hud.sp（myinfo 缺 name，用文件名代替） | `0.0` | 整数，源码写作浮点 | -1.0 ~ 1.0 | HUD 槽 3 动画可到达的垂直最小位置 | `l4d2_scripted_hud.sp:772` |
| `l4d2_scripted_hud_hud3_y_speed` | l4d2_scripted_hud.sp（myinfo 缺 name，用文件名代替） | `0.0` | 整数，源码写作浮点 | 0.0 ~ 2.0 | HUD 槽 3 文本垂直移动动画速度，0 关闭 | `l4d2_scripted_hud.sp:768` |
| `l4d2_scripted_hud_hud4_background` | l4d2_scripted_hud.sp（myinfo 缺 name，用文件名代替） | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | HUD 槽 4 文本是否显示黑色半透明背景 | `l4d2_scripted_hud.sp:783` |
| `l4d2_scripted_hud_hud4_beep` | l4d2_scripted_hud.sp（myinfo 缺 name，用文件名代替） | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | HUD 槽 4 闪烁时是否播放提示音，需闪烁开关为 1 | `l4d2_scripted_hud.sp:781` |
| `l4d2_scripted_hud_hud4_blink` | l4d2_scripted_hud.sp（myinfo 缺 name，用文件名代替） | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | HUD 槽 4 文本是否白红闪烁：0 关，1 开 | `l4d2_scripted_hud.sp:780` |
| `l4d2_scripted_hud_hud4_blink_tank` | l4d2_scripted_hud.sp（myinfo 缺 name，用文件名代替） | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | HUD 槽 4 在有 Tank 存活时是否白红闪烁：0 关，1 开 | `l4d2_scripted_hud.sp:779` |
| `l4d2_scripted_hud_hud4_flag_debug` | l4d2_scripted_hud.sp（myinfo 缺 name，用文件名代替） | `0` | 整数，源码写作浮点 | 0.0 ~ 32767.0 | 覆盖 HUD 槽 4 的 flag，仅调试用，0 关闭 | `l4d2_scripted_hud.sp:785` |
| `l4d2_scripted_hud_hud4_height` | l4d2_scripted_hud.sp（myinfo 缺 name，用文件名代替） | `0.026` | 浮点 | 0.0 ~ 2.0 | HUD 槽 4 文本区域高度 | `l4d2_scripted_hud.sp:797` |
| `l4d2_scripted_hud_hud4_team` | l4d2_scripted_hud.sp（myinfo 缺 name，用文件名代替） | `0` | 整数，源码写作浮点 | 0.0 ~ 2.0 | 哪些队伍能看到 HUD 槽 4：0 全部，1 生还者，2 特感 | `l4d2_scripted_hud.sp:784` |
| `l4d2_scripted_hud_hud4_text` | l4d2_scripted_hud.sp（myinfo 缺 name，用文件名代替） | `` | 字符串或表达式 | 无上下界 | HUD 槽 4 显示文本，留空则使用插件内置的预定义文本 | `l4d2_scripted_hud.sp:777` |
| `l4d2_scripted_hud_hud4_text_align` | l4d2_scripted_hud.sp（myinfo 缺 name，用文件名代替） | `1` | 整数，源码写作浮点 | 1.0 ~ 3.0 | HUD 槽 4 水平对齐：1 左，2 中，3 右 | `l4d2_scripted_hud.sp:778` |
| `l4d2_scripted_hud_hud4_visible` | l4d2_scripted_hud.sp（myinfo 缺 name，用文件名代替） | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | HUD 槽 4 是否允许玩家显示该槽位：0 全局禁用，1 按玩家偏好 | `l4d2_scripted_hud.sp:782` |
| `l4d2_scripted_hud_hud4_width` | l4d2_scripted_hud.sp（myinfo 缺 name，用文件名代替） | `1.5` | 浮点 | 0.0 ~ 2.0 | HUD 槽 4 文本区域宽度 | `l4d2_scripted_hud.sp:796` |
| `l4d2_scripted_hud_hud4_x` | l4d2_scripted_hud.sp（myinfo 缺 name，用文件名代替） | `0.75` | 浮点 | -1.0 ~ 1.0 | HUD 槽 4 文本水平位置 -1.0~1.0，小于 0 可能被裁切 | `l4d2_scripted_hud.sp:786` |
| `l4d2_scripted_hud_hud4_x_direction` | l4d2_scripted_hud.sp（myinfo 缺 name，用文件名代替） | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | HUD 槽 4 文本水平移动方向：0 从左到右，1 从右到左 | `l4d2_scripted_hud.sp:790` |
| `l4d2_scripted_hud_hud4_x_max` | l4d2_scripted_hud.sp（myinfo 缺 name，用文件名代替） | `1.0` | 整数，源码写作浮点 | -1.0 ~ 1.0 | HUD 槽 4 动画可到达的水平最大位置 | `l4d2_scripted_hud.sp:794` |
| `l4d2_scripted_hud_hud4_x_min` | l4d2_scripted_hud.sp（myinfo 缺 name，用文件名代替） | `0.0` | 整数，源码写作浮点 | -1.0 ~ 1.0 | HUD 槽 4 动画可到达的水平最小位置 | `l4d2_scripted_hud.sp:792` |
| `l4d2_scripted_hud_hud4_x_speed` | l4d2_scripted_hud.sp（myinfo 缺 name，用文件名代替） | `0.0` | 整数，源码写作浮点 | 0.0 ~ 2.0 | HUD 槽 4 文本水平移动动画速度，0 关闭 | `l4d2_scripted_hud.sp:788` |
| `l4d2_scripted_hud_hud4_y` | l4d2_scripted_hud.sp（myinfo 缺 name，用文件名代替） | `0.35` | 浮点 | -1.0 ~ 1.0 | HUD 槽 4 文本垂直位置 -1.0~1.0，小于 0 可能被裁切 | `l4d2_scripted_hud.sp:787` |
| `l4d2_scripted_hud_hud4_y_direction` | l4d2_scripted_hud.sp（myinfo 缺 name，用文件名代替） | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | HUD 槽 4 文本垂直移动方向：0 从上到下，1 从下到上 | `l4d2_scripted_hud.sp:791` |
| `l4d2_scripted_hud_hud4_y_max` | l4d2_scripted_hud.sp（myinfo 缺 name，用文件名代替） | `1.0` | 整数，源码写作浮点 | -1.0 ~ 1.0 | HUD 槽 4 动画可到达的垂直最大位置 | `l4d2_scripted_hud.sp:795` |
| `l4d2_scripted_hud_hud4_y_min` | l4d2_scripted_hud.sp（myinfo 缺 name，用文件名代替） | `0.0` | 整数，源码写作浮点 | -1.0 ~ 1.0 | HUD 槽 4 动画可到达的垂直最小位置 | `l4d2_scripted_hud.sp:793` |
| `l4d2_scripted_hud_hud4_y_speed` | l4d2_scripted_hud.sp（myinfo 缺 name，用文件名代替） | `0.0` | 整数，源码写作浮点 | 0.0 ~ 2.0 | HUD 槽 4 文本垂直移动动画速度，0 关闭 | `l4d2_scripted_hud.sp:789` |
| `l4d2_scripted_hud_source_teams` | l4d2_scripted_hud.sp（myinfo 缺 name，用文件名代替） | `2:5,7:5,13:5` | 字符串或表达式 | 无上下界 | 按内容来源的队伍白名单，格式 source:teammask,...；掩码 1 旁观、2 生还、4 特感（其余被截断） | `l4d2_scripted_hud.sp:713` |
| `l4d2_scripted_hud_update_interval` | l4d2_scripted_hud.sp（myinfo 缺 name，用文件名代替） | `0.1` | 浮点 | ≥ 0.1 | HUD 刷新间隔秒数，最小 0.1 | `l4d2_scripted_hud.sp:712` |
| `l4d2_scripted_hud_version` | l4d2_scripted_hud.sp（myinfo 缺 name，用文件名代替） | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | `l4d2_scripted_hud.sp:710` |
| `l4d2_scvng_firsthit_shuffle` | [L4D2] Fix First-Hit | `0` | 整数，源码写作浮点 | 0.0 ~ 2.0 | 是否打乱首发特感职业，仅影响清道夫模式：1 每回合打乱，2 每场比赛打乱，其余被截断 | `l4d2_fix_firsthit.sp:39` |
| `l4d2_shotgun_ff_enable` | L4D2 Godframes Control combined with FF Plugins | `1` | 整数，源码写作浮点 | 0.0 ~ 5.0 | 是否启用霰弹枪友军伤害模块，范围 0~5 | `l4d2_godframes_control_merge.sp:156` |
| `l4d2_shotgun_ff_max` | L4D2 Godframes Control combined with FF Plugins | `6.0` | 整数，源码写作浮点 | ≥ 0.0 | 允许的霰弹枪友军伤害上限，0 表示不限制 | `l4d2_godframes_control_merge.sp:159` |
| `l4d2_shotgun_ff_min` | L4D2 Godframes Control combined with FF Plugins | `1.0` | 整数，源码写作浮点 | ≥ 0.0 | 允许的霰弹枪友军伤害下限，0 表示不限制 | `l4d2_godframes_control_merge.sp:158` |
| `l4d2_shotgun_ff_multi` | L4D2 Godframes Control combined with FF Plugins | `0.5` | 浮点 | 0.0 ~ 5.0 | 霰弹枪友军伤害的伤害修正值，范围 0~5 | `l4d2_godframes_control_merge.sp:157` |
| `l4d2_si_climb_anim_rate` | [L4D2] Infected Ladder Speed Boost | `3.2` | 浮点 | ≥ 0.0 | 兼容旧配置：普通 AI 特感仅做梯子移动加速，不再修改翻越动画速度 | `l4d2_ai_ladder_boost.sp:99` |
| `l4d2_si_push_allow_player` | L4d2-Si-Push-When-Spawn | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否允许真人特感刷出时推动 | `si_push_when_spawn.sp:36` |
| `l4d2_si_push_enable` | L4d2-Si-Push-When-Spawn | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否开启特感刷出时朝生还方向推动的效果 | `si_push_when_spawn.sp:34` |
| `l4d2_si_push_force` | L4d2-Si-Push-When-Spawn | `600` | 整数，源码写作浮点 | ≥ 0.0 | 特感刷出时的推动力度 | `si_push_when_spawn.sp:37` |
| `l4d2_si_push_height` | L4d2-Si-Push-When-Spawn | `200` | 整数，源码写作浮点 | ≥ 0.0 | 特感刷出位置高于目标生还者该高度即视为高处 | `si_push_when_spawn.sp:39` |
| `l4d2_si_push_infected` | L4d2-Si-Push-When-Spawn | `2,4,5,6` | 字符串或表达式 | 无上下界 | 允许刷出时推动的特感种类，逗号分隔 | `si_push_when_spawn.sp:35` |
| `l4d2_si_push_only_high` | L4d2-Si-Push-When-Spawn | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否只对在高处刷出的特感启用推动 | `si_push_when_spawn.sp:38` |
| `l4d2_slowdown_ak_percent` | L4D2 Slowdown Control | `0.8` | 浮点 | 无上下界 | AK 在最大伤害时造成的减速比例 | `l4d2_slowdown_control.sp:98` |
| `l4d2_slowdown_auto_percent` | L4D2 Slowdown Control | `0.5` | 浮点 | 无上下界 | 自动霰弹枪在最大伤害时造成的减速比例 | `l4d2_slowdown_control.sp:103` |
| `l4d2_slowdown_chrome_percent` | L4D2 Slowdown Control | `0.5` | 浮点 | 无上下界 | Chrome 霰弹枪在最大伤害时造成的减速比例 | `l4d2_slowdown_control.sp:102` |
| `l4d2_slowdown_crouch_speed_mod` | L4D2 Slowdown Control | `1.0` | 整数，源码写作浮点 | ≥ 0.0 | 指定触发区内玩家蹲下速度的修正，75 为默认值，1 表示默认速度 | `l4d2_slowdown_control.sp:92` |
| `l4d2_slowdown_deagle_percent` | L4D2 Slowdown Control | `0.1` | 浮点 | 无上下界 | 沙鹰在最大伤害时造成的减速比例 | `l4d2_slowdown_control.sp:95` |
| `l4d2_slowdown_gunfire_si` | L4D2 Slowdown Control | `0.0` | 整数，源码写作浮点 | -1.0 ~ 1.0 | 特感受枪击的最大减速：-1 原生减速，0.0 不减速，0.01-1.0 表示 1% 到 100% | `l4d2_slowdown_control.sp:87` |
| `l4d2_slowdown_gunfire_tank` | L4D2 Slowdown Control | `0.2` | 浮点 | -1.0 ~ 1.0 | 坦克受枪击的最大减速：-1 原生减速，0.0 不减速，0.01-1.0 表示 1% 到 100% | `l4d2_slowdown_control.sp:88` |
| `l4d2_slowdown_m4_percent` | L4D2 Slowdown Control | `0.8` | 浮点 | 无上下界 | M4 在最大伤害时造成的减速比例 | `l4d2_slowdown_control.sp:99` |
| `l4d2_slowdown_mac_percent` | L4D2 Slowdown Control | `0.8` | 浮点 | 无上下界 | 消音乌兹在最大伤害时造成的减速比例 | `l4d2_slowdown_control.sp:97` |
| `l4d2_slowdown_military_percent` | L4D2 Slowdown Control | `0.1` | 浮点 | 无上下界 | 军用狙击枪在最大伤害时造成的减速比例 | `l4d2_slowdown_control.sp:106` |
| `l4d2_slowdown_pistol_percent` | L4D2 Slowdown Control | `0.0` | 整数，源码写作浮点 | 无上下界 | 手枪在最大伤害时造成的减速等于该值乘以 l4d2_slowdown_gunfire | `l4d2_slowdown_control.sp:94` |
| `l4d2_slowdown_pump_percent` | L4D2 Slowdown Control | `0.5` | 浮点 | 无上下界 | 泵动霰弹枪在最大伤害时造成的减速比例 | `l4d2_slowdown_control.sp:101` |
| `l4d2_slowdown_rifle_percent` | L4D2 Slowdown Control | `0.1` | 浮点 | 无上下界 | 猎枪在最大伤害时造成的减速比例 | `l4d2_slowdown_control.sp:104` |
| `l4d2_slowdown_scar_percent` | L4D2 Slowdown Control | `0.8` | 浮点 | 无上下界 | SCAR 在最大伤害时造成的减速比例 | `l4d2_slowdown_control.sp:100` |
| `l4d2_slowdown_scout_percent` | L4D2 Slowdown Control | `0.1` | 浮点 | 无上下界 | Scout 在最大伤害时造成的减速比例 | `l4d2_slowdown_control.sp:105` |
| `l4d2_slowdown_uzi_percent` | L4D2 Slowdown Control | `0.8` | 浮点 | 无上下界 | 无消音乌兹在最大伤害时造成的减速比例 | `l4d2_slowdown_control.sp:96` |
| `l4d2_slowdown_water_survivors` | L4D2 Slowdown Control | `-1` | 整数，源码写作浮点 | ≥ -1.0 | 非坦克战期间生还者在水中的最大速度：-1 忽略设置，0 默认，220 为默认生还者速度 | `l4d2_slowdown_control.sp:90` |
| `l4d2_slowdown_water_survivors_during_tank` | L4D2 Slowdown Control | `0` | 整数，源码写作浮点 | ≥ 0.0 | 坦克战期间生还者在水中的最大速度：0 忽略设置，220 为默认生还者速度 | `l4d2_slowdown_control.sp:91` |
| `l4d2_slowdown_water_tank` | L4D2 Slowdown Control | `-1` | 整数，源码写作浮点 | ≥ -1.0 | 坦克在水中的最大速度：-1 忽略设置，0 默认，210 为默认坦克速度 | `l4d2_slowdown_control.sp:89` |
| `l4d2_source_keyvalues_version` | L4D2 Source KeyValues | `VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（默认值取自 VERSION 宏） | `l4d2_source_keyvalues.sp:78` |
| `l4d2_spit_alternate_dmg` | L4D2 Uniform Spit | `-1.0` | 整数，源码写作浮点 | 无上下界 | 交替跳的伤害，-1 表示关闭 | `l4d2_uniform_spit.sp:69` |
| `l4d2_spit_dmg` | L4D2 Uniform Spit | `-1.0` | 整数，源码写作浮点 | 无上下界 | 毒痰每跳造成的伤害，-1 表示不调整伤害 | `l4d2_uniform_spit.sp:68` |
| `l4d2_spit_godframe_ticks` | L4D2 Uniform Spit | `4` | 整数 | 无上下界 | 初始处于神圣帧的酸液跳数 | `l4d2_uniform_spit.sp:71` |
| `l4d2_spit_max_flames` | [L4D2] Spit Spread Patch | `10` | 整数，源码写作浮点 | ≥ 2.0 | 一次普通毒痰最多生成的痰池数量，最小 2，游戏默认 10 | `l4d2_spit_spread_patch.sp:177` |
| `l4d2_spit_max_ticks` | L4D2 Uniform Spit | `28` | 整数 | 无上下界 | 酸液伤害跳数的上限 | `l4d2_uniform_spit.sp:70` |
| `l4d2_spit_prop_damage` | [L4D2] Spit Spread Patch | `10.0` | 整数，源码写作浮点 | ≥ 0.0 | 投射物弹跳时对道具造成的伤害，0 表示不造成伤害 | `l4d2_spit_spread_patch.sp:193` |
| `l4d2_spit_spread_saferoom` | [L4D2] Spit Spread Patch | `0` | 开关 0 或 1 | 0.0 ~ 2.0 | 毒痰在安全室区域的扩散方式：0 不扩散，1 在开场起始区域扩散，2 每张地图都扩散 | `l4d2_spit_spread_patch.sp:161` |
| `l4d2_spit_water_collision` | [L4D2] Spit Spread Patch | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 毒痰投射物是否会与水碰撞：0 不碰撞，1 碰撞 | `l4d2_spit_spread_patch.sp:185` |
| `l4d2_spot_marker_announce_type` | L4D2 Item hint | `1` | 整数，源码写作浮点 | 0.0 ~ 3.0 | 标记点提示显示方式：0 禁用，1 聊天，2 提示框，3 中心文本 | `l4d2_item_hint.sp:146` |
| `l4d2_spot_marker_color` | L4D2 Item hint | `0 255 255` | 字符串或表达式 | 无上下界 | 标记点发光颜色，留空禁用标记点 | `l4d2_item_hint.sp:148` |
| `l4d2_spot_marker_cooldown_time` | L4D2 Item hint | `2.5` | 浮点 | ≥ 0.0 | 玩家再次使用 Look 标记点的冷却秒数 | `l4d2_item_hint.sp:143` |
| `l4d2_spot_marker_duration` | L4D2 Item hint | `15.0` | 整数，源码写作浮点 | ≥ 0.0 | 标记点持续时间 | `l4d2_item_hint.sp:147` |
| `l4d2_spot_marker_instructorhint_color` | L4D2 Item hint | `200 200 200` | 字符串或表达式 | 无上下界 | 标记点的教学提示颜色 | `l4d2_item_hint.sp:151` |
| `l4d2_spot_marker_instructorhint_enable` | L4D2 Item hint | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 为 1 时为标记点创建教学提示 | `l4d2_item_hint.sp:150` |
| `l4d2_spot_marker_instructorhint_icon` | L4D2 Item hint | `icon_info` | 字符串或表达式 | 无上下界 | 标记点的教学提示图标名 | `l4d2_item_hint.sp:152` |
| `l4d2_spot_marker_sprite_model` | L4D2 Item hint | `materials/vgui/icon_arrow_down.vmt` | 字符串或表达式 | 无上下界 | 标记点精灵模型，留空禁用 | `l4d2_item_hint.sp:149` |
| `l4d2_spot_marker_use_range` | L4D2 Item hint | `1800` | 整数，源码写作浮点 | ≥ 1.0 | 玩家可使用 Look 标记点的距离 | `l4d2_item_hint.sp:144` |
| `l4d2_spot_marker_use_sound` | L4D2 Item hint | `buttons/blip1.wav` | 字符串或表达式 | 无上下界 | 标记点音效路径，留空为关闭 | `l4d2_item_hint.sp:145` |
| `l4d2_steady_boost_flags` | [L4D2] Steady Boost | `3` | 整数，源码写作浮点 | 0.0 ~ 3.0 | 哪些队伍可以使用稳定加成：1 生还者，2 特感，3 全部，0 禁用 | `l4d2_steady_boost.sp:39` |
| `l4d2_survivor_set` | [L4D2]Character_manager | `3` | 整数，源码写作浮点 | 0.0 ~ 3.0 | 生还者角色集：0 用地图默认，1 用 L4D1，2 用 L4D2，3 两者都用 | `l4d2_character_manager.sp:106` |
| `l4d2_tank_bypass_extra_flow` | L4D2 Tank Horde Monitor | `1500.0` | 整数，源码写作浮点 | ≥ 0.0 | 无限事件期间允许额外绕过坦克的流程距离，0 关闭 | `l4d2_tank_horde_monitor.sp:58` |
| `l4d2_tank_climb_anim_rate` | [L4D2] Infected Ladder Speed Boost | `3.5` | 浮点 | ≥ 0.0 | Tank 高翻越动画播放倍速，1.0 为原速 | `l4d2_ai_ladder_boost.sp:100` |
| `l4d2_tank_flying_incap_anim_fix` | [L4D2] Flying Incap - Tank Punch | `0` | 整数 | 无上下界 | 是否移除飞翔结束时的起身动画，移除后生还者落地即可射击 | `l4d2_tank_flying_incap.sp:32` |
| `l4d2_tank_flying_incap_debug` | [L4D2] Flying Incap - Tank Punch | `0` | 整数 | 无上下界 | 是否输出调试信息 | `l4d2_tank_flying_incap.sp:31` |
| `l4d2_tank_ladder_anim_rate` | [L4D2] Infected Ladder Speed Boost | `1.0` | 整数，源码写作浮点 | ≥ 0.0 | Tank 梯子动画播放倍速，真实爬梯速度由 m_flLaggedMovementValue 控制 | `l4d2_ai_ladder_boost.sp:102` |
| `l4d2_tank_low_climb_anim_rate` | [L4D2] Infected Ladder Speed Boost | `2.5` | 浮点 | ≥ 0.0 | Tank 低翻越动画播放倍速，1.0 为原速 | `l4d2_ai_ladder_boost.sp:101` |
| `l4d2_tank_prop_dissapear_time` | L4D2 Tank Hittable Glow | `10.0` | 整数，源码写作浮点 | 无上下界 | 被坦克击打过的可击打物在坦克死亡后消失所需时间 | `l4d2_tank_props_glow.sp:60` |
| `l4d2_tank_prop_glow_color` | L4D2 Tank Hittable Glow | `255 255 255` | 字符串或表达式 | 无上下界 | 可击打物轮廓颜色，三个 0-255 数值以空格分隔的 RGB | `l4d2_tank_props_glow.sp:55` |
| `l4d2_tank_prop_glow_only` | L4D2 Tank Hittable Glow | `0` | 整数 | 无上下界 | 是否只有坦克能看到轮廓 | `l4d2_tank_props_glow.sp:58` |
| `l4d2_tank_prop_glow_range` | L4D2 Tank Hittable Glow | `4500` | 整数 | 无上下界 | 玩家需要靠近可击打物到多少距离才启用轮廓 | `l4d2_tank_props_glow.sp:56` |
| `l4d2_tank_prop_glow_range_min` | L4D2 Tank Hittable Glow | `256` | 整数 | 无上下界 | 玩家靠近到该距离以内则关闭轮廓 | `l4d2_tank_props_glow.sp:57` |
| `l4d2_tank_prop_glow_spectators` | L4D2 Tank Hittable Glow | `1` | 整数 | 无上下界 | 旁观者是否也能看到轮廓 | `l4d2_tank_props_glow.sp:59` |
| `l4d2_tankannounce_messagetype` | Tank刷新提示 | `1` | 整数，源码写作浮点 | 0.0 ~ 3.0 | Tank 生成提示类型：0 不提示，1 聊天框，2 中央提示框，3 中央文字 | `l4d2_tank_announce.sp:30` |
| `l4d2_tankannounce_playsound` | Tank刷新提示 | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否在 Tank 生成时播放声音 | `l4d2_tank_announce.sp:29` |
| `l4d2_tankrage_debug` | L4D2 Tank Rage | `0` | 整数 | 无上下界 | 是否输出调试信息 | `l4d2_tankrage.sp:36` |
| `l4d2_tankrage_flowpercent` | L4D2 Tank Rage | `7` | 整数 | 无上下界 | 生还者回跑到该流程百分比时给予挫败感冻结，以最远生还者计 | `l4d2_tankrage.sp:34` |
| `l4d2_tankrage_freezetime` | L4D2 Tank Rage | `4.0` | 整数，源码写作浮点 | 无上下界 | 生还者回跑达到该百分比后冻结坦克挫败感的秒数 | `l4d2_tankrage.sp:35` |
| `l4d2_tongue_delay_survivor` | Tongue Timer | `4.0` | 整数，源码写作浮点 | 无上下界 | Smoker 被生还者快速解救后的冷却秒数，原生约 0.5 秒 | `l4d2_tongue_timer.sp:38` |
| `l4d2_tongue_delay_tank` | Tongue Timer | `8.0` | 整数，源码写作浮点 | 无上下界 | Smoker 被坦克拳击或石头快速解救后的冷却秒数，原生约 0.5 秒 | `l4d2_tongue_timer.sp:37` |
| `l4d2_uncommon_attract` | [L4D2] Uncommon Adjustment | `3` | 整数，源码写作浮点 | 0.0 ~ 3.0 | 小丑与 Jimmy Gibbs Jr. 能否吸引僵尸：0 都不能，1 小丑，2 Jimmy，3 两者 | `l4d2_uncommon_adjustment.sp:81` |
| `l4d2_uncommon_health_multiplier` | [L4D2] Uncommon Adjustment | `3.0` | 整数，源码写作浮点 | ≥ 0.0 | 非常见感染者的生命倍率，不适用于 Jimmy、fallen 与防暴警 | `l4d2_uncommon_adjustment.sp:105` |
| `l4d2_undoff_blockzerodmg` | L4D2 Godframes Control combined with FF Plugins | `7` | 整数 | 无上下界 | 位标志，可相加：屏蔽 0 伤害的友军伤害效果如后坐力与语音统计等，具体位含义源码描述被截断 | `l4d2_godframes_control_merge.sp:153` |
| `l4d2_undoff_enable` | L4D2 Godframes Control combined with FF Plugins | `7` | 整数 | 无上下界 | 位标志，可相加：1 距离过近，2 Charger 搬运，4 有罪 Bot，7 全部，0 关闭 | `l4d2_godframes_control_merge.sp:152` |
| `l4d2_undoff_permdmgfrac` | L4D2 Godframes Control combined with FF Plugins | `1.0` | 开关 0 或 1 | 0.0 ~ 1.0 | 施加到永久生命的最小伤害比例，范围 0~1 | `l4d2_godframes_control_merge.sp:154` |
| `l4d2_vote_returnlobby_patch_version` | L4D2 Vote Return Lobby patch | `VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（默认值取自 VERSION 宏） | `l4d2_vote_returnlobby_patch.sp:22` |
| `l4d2_vscript_return` | [L4D & L4D2] Left 4 DHooks Direct | `` | 字符串或表达式 | 无上下界 | 用于返回 VScript 值的缓冲区，源码注明请勿使用 | `left4dhooks.sp:739` |
| `l4d2_witch_glow_max_range` | L4D2 Witch glow | `2000` | 整数 | 无上下界 | 发光最大距离 | `witch_glow.sp:26` |
| `l4d2_witch_glow_min_range` | L4D2 Witch glow | `500` | 整数 | 无上下界 | 发光最小距离 | `witch_glow.sp:25` |
| `l4d2_witch_glow_version` | L4D2 Witch glow | `VERSION` | 字符串或表达式 | 无上下界 | 插件版本 | `witch_glow.sp:23` |
| `l4d2mm_finale_end_start` | l4d2_mixmap | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 终局结束后是否重新混合地图：0 禁用，1 启用 | `l4d2_mixmap.sp:124` |
| `l4d2mm_max_maps_num` | l4d2_mixmap | `2` | 整数，源码写作浮点 | 0.0 ~ 5.0 | 一个战役最多可选择多少张地图，0 表示不限制，范围 0~5 | `l4d2_mixmap.sp:123` |
| `l4d2mm_nextmap_print` | l4d2_mixmap | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否显示下一张地图是什么 | `l4d2_mixmap.sp:122` |
| `l4d_adrenaline_hot` | L4D HOTs | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 肾上腺素是否随时间持续回血 | `l4dhots.sp:74` |
| `l4d_adrenaline_hot_increment` | L4D HOTs | `15` | 整数，源码写作浮点 | ≥ 1.0 | 肾上腺素每次回血量，最小 1 | `l4dhots.sp:76` |
| `l4d_adrenaline_hot_interval` | L4D HOTs | `1.0` | 浮点 | ≥ 0.00001 | 肾上腺素回血间隔，最小 0.00001 | `l4dhots.sp:75` |
| `l4d_adrenaline_hot_total` | L4D HOTs | `buffer` | 字符串或表达式 | ≥ 0.0 | 肾上腺素回血总量，最小 0 | `l4dhots.sp:77` |
| `l4d_air_abilities_patch_neri` | l4d_air_abilities_patch.sp（myinfo 缺 name，用文件名代替） | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用 Anne-Neri 的 smoker 与 boomer 空中技能补丁 | `l4d_air_abilities_patch.sp:22` |
| `l4d_automatic_pistol` | L4D2 pistol delay | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 按住攻击键时手枪能否连续射击 | `l4d2_pistol_delay.sp:92` |
| `l4d_bandw_glow` | L4D Black and White Notifier | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 0 关闭发光，1 开启发光 | `l4d_blackandwhite.sp:47` |
| `l4d_bandw_notice` | L4D Black and White Notifier | `1` | 整数，源码写作浮点 | 0.0 ~ 3.0 | 0 关闭提示，1 通知生还者，2 通知所有人，3 通知特感 | `l4d_blackandwhite.sp:45` |
| `l4d_bandw_type` | L4D Black and White Notifier | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 0 输出到聊天，1 显示提示框 | `l4d_blackandwhite.sp:46` |
| `l4d_blackandwhite_version` | L4D Black and White Notifier | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | `l4d_blackandwhite.sp:37` |
| `l4d_boss_vote` | [L4D2] Vote Boss | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用 boss 投票 | `l4d_boss_vote.sp:48` |
| `l4d_boss_vote` | [L4D2] Vote Boss | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用 boss 投票 | `l4d_boss_vote.sp:44` |
| `l4d_boss_vote_limit` | [L4D2] Vote Boss | `0` | 整数，源码写作浮点 | ≥ 0.0 | 第几回合之后才允许发起 boss 投票，最小 0 | `l4d_boss_vote.sp:49` |
| `l4d_bw_notify_enable` | AnneServer Server Function (quiet minimal) | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用生还者黑白提醒：0 否，1 是 | `server.sp:148` |
| `l4d_bw_notify_sound` | AnneServer Server Function (quiet minimal) | `ui/beep07.wav` | 字符串或表达式 | 无上下界 | 玩家变黑白时播放的音效，留空不播 | `server.sp:150` |
| `l4d_bw_notify_team` | AnneServer Server Function (quiet minimal) | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 0 只提醒本人，1 全队广播 | `server.sp:149` |
| `l4d_common_shove_flag` | [L4D & 2] Fix Common Shove | `15` | 整数，源码写作浮点 | ≥ 0.0 | 修复普通感染者推击的位标志：1 蹲下，2 下落，4 落地，8 攀爬 | `l4d_fix_common_shove.sp:140` |
| `l4d_console_spam_version` | [L4D & L4D2] Console Spam Patches | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | `l4d_console_spam.sp:90` |
| `l4d_csm_admins_only` | survivor_chat_select.sp（myinfo 缺 name，用文件名代替） | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否把 sm_csm 命令限制为管理员使用：1 仅管理员 | `survivor_chat_select.sp:103` |
| `l4d_equalise_alarm_debug` | L4D2 Equalise Alarm Cars | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否输出警报车调试信息 | `l4d_equalise_alarm_cars.sp:94` |
| `l4d_equalise_alarm_start_disabled` | L4D2 Equalise Alarm Cars | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 游戏正式开始前警报车是否以禁用状态生成 | `l4d_equalise_alarm_cars.sp:93` |
| `l4d_ff_announce_enable` | [L4D & 2] Survivor FF Announce | `2` | 整数，源码写作浮点 | 0.0 ~ 2.0 | 是否播报友军伤害：0 关闭，1 私下播报，2 额外播报给旁观者 | `l4d_ffannounce.sp:57` |
| `l4d_gear_transfer_allow` | [L4D & L4D2] Gear Transfer | `1` | 整数 | 无上下界 | 0 关闭插件，1 开启插件 | `l4d_gear_transfer.sp:697` |
| `l4d_gear_transfer_dist_give` | [L4D & L4D2] Gear Transfer | `150.0` | 整数，源码写作浮点 | 无上下界 | 转移物品所需的距离，同时影响 Bot 自动给予的范围 | `l4d_gear_transfer.sp:702` |
| `l4d_gear_transfer_dist_grab` | [L4D & L4D2] Gear Transfer | `150.0` | 整数，源码写作浮点 | 无上下界 | Bot 自动拾取物品所需的距离 | `l4d_gear_transfer.sp:703` |
| `l4d_gear_transfer_dying` | [L4D & L4D2] Gear Transfer | `0` | 整数 | 无上下界 | Bot 仅在接收方黑白时自动给予：0 忽略，1 急救包，2 止痛药或肾上腺素 | `l4d_gear_transfer.sp:704` |
| `l4d_gear_transfer_idle` | [L4D & L4D2] Gear Transfer | `0` | 整数 | 无上下界 | 是否允许与挂机玩家转移物品：0 否，1 是 | `l4d_gear_transfer.sp:705` |
| `l4d_gear_transfer_method` | [L4D & L4D2] Gear Transfer | `3` | 整数 | 无上下界 | 转移方式：0 关闭，1 仅推击，2 仅换弹键，3 推击与换弹键 | `l4d_gear_transfer.sp:706` |
| `l4d_gear_transfer_modes_bot` | [L4D & L4D2] Gear Transfer | `` | 字符串或表达式 | 无上下界 | 在这些游戏模式中禁止 Bot 自动给/取物品，逗号分隔，留空为不禁止 | `l4d_gear_transfer.sp:698` |
| `l4d_gear_transfer_modes_off` | [L4D & L4D2] Gear Transfer | `` | 字符串或表达式 | 无上下界 | 在这些游戏模式中关闭插件，逗号分隔，留空为不关闭 | `l4d_gear_transfer.sp:700` |
| `l4d_gear_transfer_modes_on` | [L4D & L4D2] Gear Transfer | `` | 字符串或表达式 | 无上下界 | 在这些游戏模式中启用插件，逗号分隔，留空为全部模式 | `l4d_gear_transfer.sp:699` |
| `l4d_gear_transfer_modes_tog` | [L4D & L4D2] Gear Transfer | `0` | 整数 | 无上下界 | 按模式位掩码启用插件：0 全部，1 战役，2 生存，4 对战，8 清道夫，可相加 | `l4d_gear_transfer.sp:701` |
| `l4d_gear_transfer_notifies` | [L4D & L4D2] Gear Transfer | `7` | 整数 | 无上下界 | 在哪些转移类型时提示：1 给予，2 拾取，4 交换，7 全部，可相加 | `l4d_gear_transfer.sp:707` |
| `l4d_gear_transfer_notify` | [L4D & L4D2] Gear Transfer | `1` | 整数 | 无上下界 | 转移提示方式：0 关闭，1 显示给所有人，2 额外显示游戏自带的药丸/肾上腺素转移，4 描述被截断 | `l4d_gear_transfer.sp:708` |
| `l4d_gear_transfer_sounds` | [L4D & L4D2] Gear Transfer | `1` | 整数 | 无上下界 | 0 关闭，1 给给予或接收物品的玩家播放音效 | `l4d_gear_transfer.sp:709` |
| `l4d_gear_transfer_start` | [L4D & L4D2] Gear Transfer | `0.0` | 整数，源码写作浮点 | 无上下界 | 回合开始后多少秒内禁止 Bot 自动给予与自动拾取 | `l4d_gear_transfer.sp:710` |
| `l4d_gear_transfer_timeout` | [L4D & L4D2] Gear Transfer | `5.0` | 整数，源码写作浮点 | ≥ 1.0 | Bot 与玩家交换后停止归还物品的超时秒数，最小 1 | `l4d_gear_transfer.sp:713` |
| `l4d_gear_transfer_timer_give` | [L4D & L4D2] Gear Transfer | `1.0` | 整数，源码写作浮点 | 0.0 ~ 10.0 | 检查生还者 Bot 与真人位置以自动给予的间隔秒数，0 关闭，范围 0~10 | `l4d_gear_transfer.sp:711` |
| `l4d_gear_transfer_timer_grab` | [L4D & L4D2] Gear Transfer | `0.5` | 浮点 | 0.0 ~ 10.0 | 检查生还者 Bot 与物品位置以自动拾取的间隔秒数，0 关闭，范围 0~10 | `l4d_gear_transfer.sp:712` |
| `l4d_gear_transfer_traces` | [L4D & L4D2] Gear Transfer | `15` | 整数，源码写作浮点 | 1.0 ~ 120.0 | 每帧用于自动给/取的最大射线检测次数，范围 1~120 | `l4d_gear_transfer.sp:714` |
| `l4d_gear_transfer_types_give` | [L4D & L4D2] Gear Transfer | `123456789` | 整数 | 无上下界 | Bot 可自动给予的物品类型位域：0 关闭，1 肾上腺素，2 止痛药，3 燃烧瓶，4 管式炸弹，5 呕吐瓶，6 急救包，其余截断 | `l4d_gear_transfer.sp:715` |
| `l4d_gear_transfer_types_grab` | [L4D & L4D2] Gear Transfer | `123456789` | 整数 | 无上下界 | Bot 可自动拾取的物品类型位域，取值含义同自动给予 | `l4d_gear_transfer.sp:716` |
| `l4d_gear_transfer_types_real` | [L4D & L4D2] Gear Transfer | `123456789` | 整数 | 无上下界 | 真人玩家可转移的物品类型位域，取值含义同上 | `l4d_gear_transfer.sp:717` |
| `l4d_gear_transfer_version` | [L4D & L4D2] Gear Transfer | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | `l4d_gear_transfer.sp:719` |
| `l4d_gear_transfer_vocalize` | [L4D & L4D2] Gear Transfer | `1` | 整数 | 无上下界 | 0 关闭，1 玩家转移物品时发出语音，新回合前 60 秒内禁止 | `l4d_gear_transfer.sp:718` |
| `l4d_global_percent` | [L4D2] Boss Percents/Vote Boss Hybrid | `0` | 整数 | 无上下界 | 使用命令时是否向全队显示 boss 百分比 | `l4d_boss_percent.sp:94` |
| `l4d_hats_allow` | [L4D & L4D2] Hats | `1` | 整数 | 无上下界 | 0 关闭插件，1 开启插件 | `l4d_hats.sp:816` |
| `l4d_hats_bots` | [L4D & L4D2] Hats | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 0 不允许 Bot 生成时戴帽子，1 允许 | `l4d_hats.sp:817` |
| `l4d_hats_change` | [L4D & L4D2] Hats | `1.3` | 浮点 | 无上下界 | 0 关闭；其它值表示选择帽子时让玩家进入第三人称的持续秒数 | `l4d_hats.sp:818` |
| `l4d_hats_detect` | [L4D & L4D2] Hats | `0.3` | 浮点 | 无上下界 | 0 关闭；检测第三人称视角的间隔，若存在 ThirdPersonShoulder_Detect 插件也会使用 | `l4d_hats.sp:819` |
| `l4d_hats_make` | [L4D & L4D2] Hats | `c` | 字符串或表达式 | 无上下界 | 允许随机戴帽子的管理员 flag，留空为所有玩家，需 l4d_hats_random 生效 | `l4d_hats.sp:820` |
| `l4d_hats_menu` | [L4D & L4D2] Hats | `c` | 字符串或表达式 | 无上下界 | 允许访问帽子菜单的管理员 flag，留空为所有玩家 | `l4d_hats.sp:821` |
| `l4d_hats_modes` | [L4D & L4D2] Hats | `` | 字符串或表达式 | 无上下界 | 在这些游戏模式中启用插件，逗号分隔，留空为全部模式 | `l4d_hats.sp:822` |
| `l4d_hats_modes_off` | [L4D & L4D2] Hats | `` | 字符串或表达式 | 无上下界 | 在这些游戏模式中关闭插件，逗号分隔，留空为不关闭 | `l4d_hats.sp:823` |
| `l4d_hats_modes_tog` | [L4D & L4D2] Hats | `` | 字符串或表达式 | 无上下界 | 按模式位掩码启用插件：0 全部，1 战役，2 生存，4 对战，8 清道夫，可相加 | `l4d_hats.sp:824` |
| `l4d_hats_opaque` | [L4D & L4D2] Hats | `255` | 整数，源码写作浮点 | 0.0 ~ 255.0 | 帽子的不透明度：0 半透明，255 完全不透明 | `l4d_hats.sp:825` |
| `l4d_hats_precache` | [L4D & L4D2] Hats | `` | 字符串或表达式 | 无上下界 | 在这些地图上禁止预缓存模型，逗号分隔，源码警告在这些地图启用会导致服务器崩溃 | `l4d_hats.sp:826` |
| `l4d_hats_random` | [L4D & L4D2] Hats | `0` | 整数，源码写作浮点 | 0.0 ~ 3.0 | 生还者生成时是否随机戴帽子：0 从不，1 每回合开始，2 仅首次生成（下回合保持同一顶） | `l4d_hats.sp:827` |
| `l4d_hats_save` | [L4D & L4D2] Hats | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 0 关闭，1 保存玩家选定的帽子并在其生成或重连时戴上，覆盖随机设置 | `l4d_hats.sp:828` |
| `l4d_hats_third` | [L4D & L4D2] Hats | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 0 关闭，1 玩家处于第三人称时显示帽子，第一人称时隐藏 | `l4d_hats.sp:829` |
| `l4d_hats_version` | [L4D & L4D2] Hats | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | `l4d_hats.sp:831` |
| `l4d_hats_wall` | [L4D & L4D2] Hats | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 0 显示穿墙的帽子发光，1 隐藏墙后的帽子发光（每个帽子多消耗一个实体） | `l4d_hats.sp:830` |
| `l4d_infected_movement_allow` | [L4D & L4D2] Special Infected Ability Movement | `3` | 整数 | 无上下界 | 0 关闭插件，1 仅真人可用，2 仅 Bot 可用，3 两者都可用 | `l4d_infected_movement.sp:121` |
| `l4d_infected_movement_bots` | [L4D & L4D2] Special Infected Ability Movement | `2` | 整数 | 无上下界 | 哪些 AI 特感可使用技能期间移动：1 Smoker，2 Spitter，4 Tank，7 全部，可相加 | `l4d_infected_movement.sp:125` |
| `l4d_infected_movement_modes` | [L4D & L4D2] Special Infected Ability Movement | `` | 字符串或表达式 | 无上下界 | 在这些游戏模式中启用插件，逗号分隔，留空为全部模式 | `l4d_infected_movement.sp:122` |
| `l4d_infected_movement_modes_off` | [L4D & L4D2] Special Infected Ability Movement | `` | 字符串或表达式 | 无上下界 | 在这些游戏模式中关闭插件，逗号分隔，留空为不关闭 | `l4d_infected_movement.sp:123` |
| `l4d_infected_movement_modes_tog` | [L4D & L4D2] Special Infected Ability Movement | `0` | 整数 | 无上下界 | 按模式位掩码启用插件：0 全部，1 战役，2 生存，4 对战，8 清道夫，可相加 | `l4d_infected_movement.sp:124` |
| `l4d_infected_movement_smoker` | [L4D & L4D2] Special Infected Ability Movement | `2` | 整数 | 无上下界 | 0 仅吐舌时；1 Smoker 拉人时可移动；2 舌头挂住目标时也可移动 | `l4d_infected_movement.sp:126` |
| `l4d_infected_movement_speed_smoker` | [L4D & L4D2] Special Infected Ability Movement | `250` | 整数 | 无上下界 | Smoker 使用技能时的移动速度 | `l4d_infected_movement.sp:127` |
| `l4d_infected_movement_speed_spitter` | [L4D & L4D2] Special Infected Ability Movement | `250` | 整数 | 无上下界 | Spitter 使用技能时的移动速度 | `l4d_infected_movement.sp:130` |
| `l4d_infected_movement_speed_tank` | [L4D & L4D2] Special Infected Ability Movement | `210` | 整数 | 无上下界 | Tank 使用技能时的移动速度 | `l4d_infected_movement.sp:128` |
| `l4d_infected_movement_type` | [L4D & L4D2] Special Infected Ability Movement | `2` | 整数 | 无上下界 | 哪些真人特感可使用技能期间移动：1 Smoker，2 Spitter，4 Tank，7 全部，可相加 | `l4d_infected_movement.sp:131` |
| `l4d_infected_movement_version` | [L4D & L4D2] Special Infected Ability Movement | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | `l4d_infected_movement.sp:132` |
| `l4d_mapcvars_configdir` | L4D(2) map-based convar loader. | `` | 字符串或表达式 | 无上下界 | 使用哪个 cfgogl 配置 | `l4d_mapbased_cvars.sp:44` |
| `l4d_multi_witches_version` | [L4D1 & L4D2] Multi witches | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | `l4d_multi_witches.sp:71` |
| `l4d_multislots_autokicktank` | AnneServer Server Function (quiet minimal) | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 当场上 Tank 大于 1 时是否自动踢到只剩 1 只：0 否，1 是 | `server.sp:124` |
| `l4d_multislots_max_survivors` | AnneServer Server Function (quiet minimal) | `4` | 整数，源码写作浮点 | 4.0 ~ 8.0 | 生还者最大人数，仅踢 Bot 不踢真人，范围 4~8 | `server.sp:122` |
| `l4d_multislots_survivors_manager_enable` | AnneServer Server Function (quiet minimal) | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用生还者数量管理：0 否，1 是 | `server.sp:120` |
| `l4d_no_bash_kills` | L4D2 Bash Kills | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否阻止特感被推击致死 | `l4d_bash_kills.sp:37` |
| `l4d_no_cans` | L4D2 Remove Cans | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否移除汽油桶 | `l4d_no_cans.sp:33` |
| `l4d_no_fireworks` | L4D2 Remove Cans | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否移除烟花 | `l4d_no_cans.sp:36` |
| `l4d_no_oxygen` | L4D2 Remove Cans | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否移除氧气罐 | `l4d_no_cans.sp:35` |
| `l4d_no_propane` | L4D2 Remove Cans | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否移除丙烷罐 | `l4d_no_cans.sp:34` |
| `l4d_no_tank_rush` | L4D2 No Tank Rush | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否阻止生还者队伍在坦克存活期间累积分数 | `l4d_tank_rush.sp:39` |
| `l4d_no_tank_rush_spawn_sound` | L4D2 No Tank Rush | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 生成坦克时是否播放声音 | `l4d_tank_rush.sp:42` |
| `l4d_no_tank_rush_unfreeze_ai` | L4D2 No Tank Rush | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 坦克变为 AI 时是否解冻距离 | `l4d_tank_rush.sp:41` |
| `l4d_no_tank_rush_unfreeze_saferoom` | L4D2 No Tank Rush | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 坦克仍存活时有生还者到达终点安全室是否解冻距离 | `l4d_tank_rush.sp:40` |
| `l4d_obey_boss_spawn_cvars` | Versus Boss Spawn Persuasion | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否强制 boss 刷新遵守相关 cvar | `bossspawningfix.sp:23` |
| `l4d_obey_boss_spawn_except_static` | Versus Boss Spawn Persuasion | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 在静态坦克刷新地图上是否不覆盖 boss 刷新规则 | `bossspawningfix.sp:24` |
| `l4d_pills_hot` | L4D HOTs | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 止痛药是否随时间持续回血 | `l4dhots.sp:60` |
| `l4d_pills_hot_increment` | L4D HOTs | `10` | 整数，源码写作浮点 | ≥ 1.0 | 止痛药每次回血量，最小 1 | `l4dhots.sp:62` |
| `l4d_pills_hot_interval` | L4D HOTs | `1.0` | 浮点 | ≥ 0.00001 | 止痛药回血的间隔，最小 0.00001 | `l4dhots.sp:61` |
| `l4d_pills_hot_total` | L4D HOTs | `buffer` | 字符串或表达式 | ≥ 0.0 | 止痛药回血总量，最小 0 | `l4dhots.sp:63` |
| `l4d_pistol_delay_dualies` | L4D2 pistol delay | `0.1` | 浮点 | MIN_RATE_OF_FIRE ~ MAX_RATE_OF_FIRE | 双持手枪两次射击之间的最短秒数 | `l4d2_pistol_delay.sp:71` |
| `l4d_pistol_delay_single` | L4D2 pistol delay | `sDefValue` | 字符串或表达式 | MIN_RATE_OF_FIRE ~ MAX_RATE_OF_FIRE | 单持手枪两次射击之间的最短秒数 | `l4d2_pistol_delay.sp:81` |
| `l4d_player_count_unload_mode_count` | High Perk Server Status Control[Work in ZM] | `3` | 整数，源码写作浮点 | 1.0 ~ 32.0 | 当 survivor_limit 与 infected 空位之和小于等于该值时强制 sm_resetmatch 并卸载模式，范围 1~32 | `l4d_player_count_unload_mode.sp:123` |
| `l4d_player_count_unload_mode_db_config` | High Perk Server Status Control[Work in ZM] | `l4dstats` | 字符串或表达式 | 无上下界 | peak_mode 为 1 时使用的 databases.cfg 区块名 | `l4d_player_count_unload_mode.sp:129` |
| `l4d_player_count_unload_mode_delay` | High Perk Server Status Control[Work in ZM] | `60.0` | 整数，源码写作浮点 | ≥ 0.0 | 地图加载后经过这么多秒才开始检测时间与人数 | `l4d_player_count_unload_mode.sp:125` |
| `l4d_player_count_unload_mode_enable` | High Perk Server Status Control[Work in ZM] | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 0 关闭插件，1 开启插件 | `l4d_player_count_unload_mode.sp:121` |
| `l4d_player_count_unload_mode_flag` | High Perk Server Status Control[Work in ZM] | `b` | 字符串或表达式 | 无上下界 | 拥有该权限的管理员在场时不会被强制卸载模式 | `l4d_player_count_unload_mode.sp:124` |
| `l4d_player_count_unload_mode_peak_hold_time` | High Perk Server Status Control[Work in ZM] | `3600` | 整数，源码写作浮点 | ≥ 0.0 | peak_mode 为 1 时，进入高峰期后至少持续限制的秒数，0 表示不保持 | `l4d_player_count_unload_mode.sp:128` |
| `l4d_player_count_unload_mode_peak_mode` | High Perk Server Status Control[Work in ZM] | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 高峰期判定方式：0 按时间段，1 按共享数据库中所有服务器的有玩家比例 | `l4d_player_count_unload_mode.sp:126` |
| `l4d_player_count_unload_mode_peak_ratio` | High Perk Server Status Control[Work in ZM] | `0.70` | 浮点 | 0.0 ~ 1.0 | peak_mode 为 1 时，有玩家的服务器数占有效服务器数的比例达到该值即视为高峰期 | `l4d_player_count_unload_mode.sp:127` |
| `l4d_player_count_unload_mode_server_id` | High Perk Server Status Control[Work in ZM] | `` | 字符串或表达式 | 无上下界 | 本服务器唯一 ID，留空时优先从 hostname 提取 #编号，失败则用 hostname:hostport | `l4d_player_count_unload_mode.sp:130` |
| `l4d_player_count_unload_mode_server_ip` | High Perk Server Status Control[Work in ZM] | `` | 字符串或表达式 | 无上下界 | 写入网页状态表的服务器公网 IP 或域名，可填 host 或 host:port，留空则自动读取 | `l4d_player_count_unload_mode.sp:131` |
| `l4d_player_count_unload_mode_server_port` | High Perk Server Status Control[Work in ZM] | `0` | 整数，源码写作浮点 | 0.0 ~ 65535.0 | 写入网页状态表的服务器外网端口，0 表示自动读取 | `l4d_player_count_unload_mode.sp:132` |
| `l4d_player_count_unload_mode_server_utc_offset` | High Perk Server Status Control[Work in ZM] | `480` | 整数 | 无上下界 | 备用值：服务器时区相对 UTC 的偏移分钟数，仅在 %z 格式不受支持时使用 | `l4d_player_count_unload_mode.sp:137` |
| `l4d_player_count_unload_mode_status_interval` | High Perk Server Status Control[Work in ZM] | `180.0` | 整数，源码写作浮点 | ≥ 5.0 | peak_mode 为 1 时本服人数写入数据库的心跳间隔秒数，最小 5 | `l4d_player_count_unload_mode.sp:134` |
| `l4d_player_count_unload_mode_status_max_age` | High Perk Server Status Control[Work in ZM] | `540` | 整数，源码写作浮点 | ≥ 10.0 | peak_mode 为 1 时只统计多少秒内更新过的服务器，最小 10 | `l4d_player_count_unload_mode.sp:135` |
| `l4d_player_count_unload_mode_status_table` | High Perk Server Status Control[Work in ZM] | `l4d_server_status` | 字符串或表达式 | 无上下界 | peak_mode 为 1 时使用的服务器状态表名 | `l4d_player_count_unload_mode.sp:133` |
| `l4d_player_count_unload_mode_time` | High Perk Server Status Control[Work in ZM] | `18:00~22:59` | 字符串或表达式 | 无上下界 | 检测的时间段，格式 xx:xx~xx:xx 二十四小时制，多段用逗号分隔 | `l4d_player_count_unload_mode.sp:122` |
| `l4d_player_count_unload_mode_version` | High Perk Server Status Control[Work in ZM] | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | `l4d_player_count_unload_mode.sp:136` |
| `l4d_pm_supress_spectate` | Player Management Plugin | `0` | 整数 | 无上下界 | 玩家转为旁观时是否不打印提示 | `playermanagement.sp:84` |
| `l4d_ragdoll_fader` | [L4D & L4D2] Ragdoll Fader | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | `l4d_ragdoll_fader.sp:82` |
| `l4d_random_beam_item_enable` | l4d_random_beam_item.sp（myinfo 缺 name，用文件名代替） | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用插件：0 关，1 开 | `l4d_random_beam_item.sp:346` |
| `l4d_random_beam_item_min_brightness` | l4d_random_beam_item.sp（myinfo 缺 name，用文件名代替） | `0.5` | 浮点 | 0.0 ~ 1.0 | 判断随机颜色光束最低亮度的算法阈值，源码注明该值不精确 | `l4d_random_beam_item.sp:348` |
| `l4d_random_beam_item_player_edict_limit` | l4d_random_beam_item.sp（myinfo 缺 name，用文件名代替） | `1900` | 整数，源码写作浮点 | 0.0 ~ 2048.0 | 当服务器已用实体数达到该值时，玩家为默认隐藏物品开启的光束不再创建，范围 0~2048 | `l4d_random_beam_item.sp:351` |
| `l4d_random_beam_item_remove_spawner` | l4d_random_beam_item.sp（myinfo 缺 name，用文件名代替） | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 当 *_spawn 实体计数归零时是否删除该实体：0 关，1 开 | `l4d_random_beam_item.sp:347` |
| `l4d_random_beam_item_use_glow_color` | l4d_random_beam_item.sp（myinfo 缺 name，用文件名代替） | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 仅 L4D2：是否沿用发光颜色：0 关，1 开 | `l4d_random_beam_item.sp:350` |
| `l4d_random_beam_item_version` | l4d_random_beam_item.sp（myinfo 缺 name，用文件名代替） | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | `l4d_random_beam_item.sp:345` |
| `l4d_remove_decals_time` | Visualise impacts | `20.0` | 整数，源码写作浮点 | 0.0 ~ 320.0 | 弹痕在多少秒后被移除，0 表示关闭，范围 0~320 | `visualise_impacts.sp:55` |
| `l4d_safe_spam_allow` | [L4D & L4D2] Saferoom Door Spam Protection | `1` | 整数 | 无上下界 | 0 关闭插件，1 开启插件 | `l4d_safe_door_spam.sp:252` |
| `l4d_safe_spam_fall_time` | [L4D & L4D2] Saferoom Door Spam Protection | `0.0` | 整数，源码写作浮点 | 无上下界 | 0 关闭；回合开始或被锁定类 cvar 解锁后多少秒内不得使用安全门 | `l4d_safe_door_spam.sp:256` |
| `l4d_safe_spam_hint` | [L4D & L4D2] Saferoom Door Spam Protection | `0` | 整数 | 无上下界 | 0 关闭；1 显示谁开关了安全门，2 在安全门自动……时显示（描述被截断） | `l4d_safe_door_spam.sp:257` |
| `l4d_safe_spam_last` | [L4D & L4D2] Saferoom Door Spam Protection | `0` | 整数 | 无上下界 | 回合开始时最后一扇安全门的最终状态：0 用地图默认，1 关闭，2 打开 | `l4d_safe_door_spam.sp:258` |
| `l4d_safe_spam_lock` | [L4D & L4D2] Saferoom Door Spam Protection | `0.0` | 整数，源码写作浮点 | 无上下界 | 0 关闭；回合开始后安全门保持上锁的秒数 | `l4d_safe_door_spam.sp:259` |
| `l4d_safe_spam_lock_2` | [L4D & L4D2] Saferoom Door Spam Protection | `0.0` | 整数，源码写作浮点 | 无上下界 | 同 lock，但作用于地图第二回合及之后 | `l4d_safe_door_spam.sp:260` |
| `l4d_safe_spam_modes` | [L4D & L4D2] Saferoom Door Spam Protection | `` | 字符串或表达式 | 无上下界 | 在这些游戏模式中启用插件，逗号分隔，留空为全部模式 | `l4d_safe_door_spam.sp:253` |
| `l4d_safe_spam_modes_off` | [L4D & L4D2] Saferoom Door Spam Protection | `` | 字符串或表达式 | 无上下界 | 在这些游戏模式中关闭插件，逗号分隔，留空为不关闭 | `l4d_safe_door_spam.sp:254` |
| `l4d_safe_spam_modes_tog` | [L4D & L4D2] Saferoom Door Spam Protection | `0` | 整数 | 无上下界 | 按模式位掩码启用插件：0 全部，1 战役，2 生存，4 对战，8 清道夫，可相加 | `l4d_safe_door_spam.sp:255` |
| `l4d_safe_spam_open` | [L4D & L4D2] Saferoom Door Spam Protection | `2` | 整数 | 无上下界 | 0 关闭，1 第一扇安全门打开后保持打开，2 打开后让它倒下 | `l4d_safe_door_spam.sp:263` |
| `l4d_safe_spam_physics` | [L4D & L4D2] Saferoom Door Spam Protection | `3.0` | 整数，源码写作浮点 | 无上下界 | 0 表示始终保留物理；倒下的门在多长时间后禁用物理 | `l4d_safe_door_spam.sp:264` |
| `l4d_safe_spam_skin` | [L4D & L4D2] Saferoom Door Spam Protection | `0` | 整数 | 无上下界 | 第一与最后一扇安全门使用哪种模型：0 地图默认，1 经典，2 The Last Stand | `l4d_safe_door_spam.sp:266` |
| `l4d_safe_spam_time_close` | [L4D & L4D2] Saferoom Door Spam Protection | `0.0` | 整数，源码写作浮点 | 无上下界 | 关闭最后一扇安全门后禁止操作的秒数 | `l4d_safe_door_spam.sp:267` |
| `l4d_safe_spam_time_open` | [L4D & L4D2] Saferoom Door Spam Protection | `0.0` | 整数，源码写作浮点 | 无上下界 | 打开最后一扇安全门后禁止操作的秒数 | `l4d_safe_door_spam.sp:268` |
| `l4d_safe_spam_touch` | [L4D & L4D2] Saferoom Door Spam Protection | `0.0` | 整数，源码写作浮点 | 无上下界 | 0 关闭；尝试打开上锁安全门后多少秒解锁，覆盖 _time 类 cvar | `l4d_safe_door_spam.sp:261` |
| `l4d_safe_spam_touch_2` | [L4D & L4D2] Saferoom Door Spam Protection | `0.0` | 整数，源码写作浮点 | 无上下界 | 同 touch，但作用于地图第二回合及之后（描述被截断） | `l4d_safe_door_spam.sp:262` |
| `l4d_safe_spam_type` | [L4D & L4D2] Saferoom Door Spam Protection | `3` | 整数 | 无上下界 | 0 关闭；最后一扇安全门被使用时启用超时限制的方向：1 打开，2 关闭，3 两者 | `l4d_safe_door_spam.sp:269` |
| `l4d_safe_spam_version` | [L4D & L4D2] Saferoom Door Spam Protection | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | `l4d_safe_door_spam.sp:271` |
| `l4d_scs_botschange` | survivor_chat_select.sp（myinfo 缺 name，用文件名代替） | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否把新出现的 Bot 换成当前数量最少的生还者：1 开，0 关 | `survivor_chat_select.sp:105` |
| `l4d_scs_cookies` | survivor_chat_select.sp（myinfo 缺 name，用文件名代替） | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否记住玩家上次使用的生还者：1 开，0 关 | `survivor_chat_select.sp:106` |
| `l4d_scs_zoey` | survivor_chat_select.sp（myinfo 缺 name，用文件名代替） | `1` | 整数，源码写作浮点 | 0.0 ~ 2.0 | Zoey 的模型：0 用 Rochelle，1 用 Zoey，2 用 Nick | `survivor_chat_select.sp:104` |
| `l4d_skip_intro_allow` | [L4D & L4D2] First Map - Skip Intro Cutscenes | `1` | 整数 | 无上下界 | 0 关闭插件，1 开启插件 | `l4d_skip_intro.sp:133` |
| `l4d_skip_intro_modes` | [L4D & L4D2] First Map - Skip Intro Cutscenes | `` | 字符串或表达式 | 无上下界 | 在这些游戏模式中启用插件，逗号分隔，留空为全部模式 | `l4d_skip_intro.sp:134` |
| `l4d_skip_intro_modes_off` | [L4D & L4D2] First Map - Skip Intro Cutscenes | `` | 字符串或表达式 | 无上下界 | 在这些游戏模式中关闭插件，逗号分隔，留空为不关闭 | `l4d_skip_intro.sp:135` |
| `l4d_skip_intro_modes_tog` | [L4D & L4D2] First Map - Skip Intro Cutscenes | `0` | 整数 | 无上下界 | 按模式位掩码启用插件：0 全部，1 战役，2 生存，4 对战，8 清道夫，可相加 | `l4d_skip_intro.sp:136` |
| `l4d_skip_intro_version` | [L4D & L4D2] First Map - Skip Intro Cutscenes | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | `l4d_skip_intro.sp:137` |
| `l4d_tank_painfade` | L4D Tank Pain Fade | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用插件 | `l4d_tank_painfade.sp:58` |
| `l4d_tank_painfade_duration` | L4D Tank Pain Fade | `150` | 整数 | 无上下界 | 淡化持续帧数 | `l4d_tank_painfade.sp:59` |
| `l4d_tank_painfade_flags` | L4D Tank Pain Fade | `8` | 整数，源码写作浮点 | 1.0 ~ 15.0 | 哪些武器会造成淡化效果：1 乌兹，2 霰弹枪，4 狙击，8 近战 | `l4d_tank_painfade.sp:60` |
| `l4d_tank_percent` | [L4D2] Boss Percents/Vote Boss Hybrid | `1` | 整数 | 无上下界 | 是否在聊天中显示 Tank 流程百分比 | `l4d_boss_percent.sp:95` |
| `l4d_tank_props_glow` | L4D2 Tank Hittable Glow | `1` | 整数 | 无上下界 | 坦克存活期间是否向特感队伍显示可击打物轮廓 | `l4d2_tank_props_glow.sp:54` |
| `l4d_tankdamage_enabled` | Tank Damage Announce L4D2 | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否在启用时播报对坦克造成的伤害 | `l4d_tank_damage_announce.sp:71` |
| `l4d_target_override_allow` | [L4D & L4D2] Target Override | `1` | 整数 | 无上下界 | 0 关闭插件，1 开启插件 | `l4d_target_override.sp:457` |
| `l4d_target_override_forward` | [L4D & L4D2] Target Override | `0` | 整数 | 无上下界 | 0 关闭；1 为其他插件提供每帧左右触发的被瞄准前向回调 | `l4d_target_override.sp:461` |
| `l4d_target_override_modes` | [L4D & L4D2] Target Override | `` | 字符串或表达式 | 无上下界 | 在这些游戏模式中启用插件，逗号分隔，留空为全部模式 | `l4d_target_override.sp:458` |
| `l4d_target_override_modes_off` | [L4D & L4D2] Target Override | `` | 字符串或表达式 | 无上下界 | 在这些游戏模式中关闭插件，逗号分隔，留空为不关闭 | `l4d_target_override.sp:459` |
| `l4d_target_override_modes_tog` | [L4D & L4D2] Target Override | `0` | 整数 | 无上下界 | 按模式位掩码启用插件：0 全部，1 战役，2 生存，4 对战，8 清道夫，可相加 | `l4d_target_override.sp:460` |
| `l4d_target_override_specials` | [L4D & L4D2] Target Override | `127` | 整数 | 无上下界 | 覆盖哪些特感的目标函数：1 Smoker，2 Boomer，4 Hunter，8 Spitter，16 Jockey，32 Charger，64 Tank，127 为全部 | `l4d_target_override.sp:463` |
| `l4d_target_override_specials` | [L4D & L4D2] Target Override | `15` | 整数 | 无上下界 | 覆盖哪些特感的目标函数：1 Smoker，2 Boomer，4 Hunter，8 Tank，15 为全部，可相加 | `l4d_target_override.sp:465` |
| `l4d_target_override_team` | [L4D & L4D2] Target Override | `2` | 整数 | 无上下界 | 应对哪些生还者队伍生效：2 默认生还者，4 Holdout 与 Passing 的 Bot，6 两者 | `l4d_target_override.sp:466` |
| `l4d_target_override_type` | [L4D & L4D2] Target Override | `1` | 整数 | 无上下界 | 插件搜索生还者的方式：1 最近的可见目标，2 全部生还者，其余源码描述被截断 | `l4d_target_override.sp:467` |
| `l4d_target_override_version` | [L4D & L4D2] Target Override | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | `l4d_target_override.sp:468` |
| `l4d_tpsblock_action` | Thirdpersonshoulder Block | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 玩家开启第三人称时的处理：1 移到旁观，0 踢出服务器 | `l4d_thirdpersonshoulderblock.sp:45` |
| `l4d_tpsblock_enabled` | Thirdpersonshoulder Block | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用第三人称肩部视角屏蔽 | `l4d_thirdpersonshoulderblock.sp:44` |
| `l4d_tpsblock_version` | Thirdpersonshoulder Block | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | `l4d_thirdpersonshoulderblock.sp:42` |
| `l4d_use_priority_version` | [L4D & L4D2] Use Priority Patch | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | `l4d_use_priority.sp:135` |
| `l4d_votepoll_fix_version` | [L4D & L4D2] Vote Poll Fix | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | `l4d_votepoll_fix.sp:30` |
| `l4d_witch_percent` | [L4D2] Boss Percents/Vote Boss Hybrid | `1` | 整数 | 无上下界 | 是否在聊天中显示 Witch 流程百分比 | `l4d_boss_percent.sp:96` |
| `l4d_witches_director_witch` | [L4D1 & L4D2] Multi witches | `1` | 整数 | 无上下界 | 1 启用导演 Witch，0 禁用导演 Witch | `l4d_multi_witches.sp:78` |
| `l4d_witches_distance` | [L4D1 & L4D2] Multi witches | `1600.0` | 整数，源码写作浮点 | 无上下界 | 距生还者多远以外的 Witch 会被移除，0 表示不移除 | `l4d_multi_witches.sp:77` |
| `l4d_witches_limit` | [L4D1 & L4D2] Multi witches | `20` | 整数 | 无上下界 | 允许刷出的 Witch 数量上限，0 表示不检查数量 | `l4d_multi_witches.sp:73` |
| `l4d_witches_limit_alive` | [L4D1 & L4D2] Multi witches | `3` | 整数 | 无上下界 | 允许同时存活的 Witch 上限，0 表示不检查存活数量 | `l4d_multi_witches.sp:74` |
| `l4d_witches_spawn_time_max` | [L4D1 & L4D2] Multi witches | `35.0` | 整数，源码写作浮点 | 无上下界 | 插件刷 Witch 的最大间隔秒数 | `l4d_multi_witches.sp:76` |
| `l4d_witches_spawn_time_min` | [L4D1 & L4D2] Multi witches | `20.0` | 整数，源码写作浮点 | 无上下界 | 插件刷 Witch 的最小间隔秒数 | `l4d_multi_witches.sp:75` |
| `left4dhooks_version` | [L4D & L4D2] Left 4 DHooks Direct | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | `left4dhooks.sp:734` |
| `map_changer_version` | map_changer.sp（myinfo 缺 name，用文件名代替） | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | `map_changer.sp:137` |
| `mapchanger_finale_change_type` | map_changer.sp（myinfo 缺 name，用文件名代替） | `8` | 整数 | 无上下界 | 终局换图时机：0 不换图返回大厅，1 救援载具离开时，2 终局获胜时，4 统计屏幕出现时，8 统计屏幕结束时，可相加 | `map_changer.sp:139` |
| `mapchanger_finale_failure_count` | map_changer.sp（myinfo 缺 name，用文件名代替） | `3` | 整数 | 无上下界 | 终局团灭几次后自动换到下一张图 | `map_changer.sp:140` |
| `mapchanger_finale_failure_vote` | map_changer.sp（myinfo 缺 name，用文件名代替） | `1` | 整数 | 无上下界 | 救援关第一回合是否投票决定启用终局团灭自动换图 | `map_changer.sp:142` |
| `mapchanger_finale_failure_vote_default` | map_changer.sp（myinfo 缺 name，用文件名代替） | `1` | 整数 | 无上下界 | 投票关闭或无法发起时是否默认启用终局团灭自动换图 | `map_changer.sp:143` |
| `mapchanger_finale_failure_vote_delay` | map_changer.sp（myinfo 缺 name，用文件名代替） | `30.0` | 整数，源码写作浮点 | 1.0 ~ 120.0 | 救援关第一回合开始后延迟多少秒发起该投票，范围 1~120 | `map_changer.sp:144` |
| `mapchanger_finale_random_nextmap` | map_changer.sp（myinfo 缺 name，用文件名代替） | `1` | 整数 | 无上下界 | 终局是否启用随机下一关地图 | `map_changer.sp:141` |
| `mv_maxplayers` | Match Vote | `30` | 整数，源码写作浮点 | 1.0 ~ 32.0 | 配置加载或卸载时服务器应有的槽位数，范围 1~32 | `match_vote.sp:67` |
| `nb_uf_Boomer` | l4d2_npc_manager | `0.05` | 浮点 | 0.0 ~ 1.0 | Boomer 的 update frequency 更新频率，范围 0~1 | `l4d2_npc_manager.sp:113` |
| `nb_uf_Charger` | l4d2_npc_manager | `0.05` | 浮点 | 0.0 ~ 1.0 | Charger 的 update frequency 更新频率，范围 0~1 | `l4d2_npc_manager.sp:117` |
| `nb_uf_Common` | l4d2_npc_manager | `0.02` | 浮点 | 0.0 ~ 1.0 | Common 的 update frequency 更新频率，范围 0~1 | `l4d2_npc_manager.sp:111` |
| `nb_uf_Hunter` | l4d2_npc_manager | `0.02` | 浮点 | 0.0 ~ 1.0 | Hunter 的 update frequency 更新频率，范围 0~1 | `l4d2_npc_manager.sp:114` |
| `nb_uf_Jockey` | l4d2_npc_manager | `0.05` | 浮点 | 0.0 ~ 1.0 | Jockey 的 update frequency 更新频率，范围 0~1 | `l4d2_npc_manager.sp:116` |
| `nb_uf_onoff` | l4d2_npc_manager | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 插件是否接管 update frequency：1 接管，0 不接管 | `l4d2_npc_manager.sp:110` |
| `nb_uf_sb` | l4d2_npc_manager | `0.1` | 浮点 | 0.0 ~ 1.0 | Survivor Bot 的 update frequency 更新频率，范围 0~1 | `l4d2_npc_manager.sp:120` |
| `nb_uf_Smoker` | l4d2_npc_manager | `0.05` | 浮点 | 0.0 ~ 1.0 | Smoker 的 update frequency 更新频率，范围 0~1 | `l4d2_npc_manager.sp:112` |
| `nb_uf_Spitter` | l4d2_npc_manager | `0.05` | 浮点 | 0.0 ~ 1.0 | Spitter 的 update frequency 更新频率，范围 0~1 | `l4d2_npc_manager.sp:115` |
| `nb_uf_Tank` | l4d2_npc_manager | `0.02` | 浮点 | 0.0 ~ 1.0 | Tank 的 update frequency 更新频率，范围 0~1 | `l4d2_npc_manager.sp:119` |
| `nb_uf_Witch` | l4d2_npc_manager | `0.02` | 浮点 | 0.0 ~ 1.0 | Witch 的 update frequency 更新频率，范围 0~1 | `l4d2_npc_manager.sp:118` |
| `nokits_version` | No Safe Room Medkits | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | `nosaferoomkits.sp:28` |
| `noteam_nudging_version` | [L4D/L4D2]noteam_nudging | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | `noteam_nudging.sp:23` |
| `notify_map_next` | l4d2_map_vote.sp（myinfo 缺 name，用文件名代替） | `1` | 整数 | 无上下界 | 终局开始后提示投票下一张地图的方式：0 不提示，1 聊天栏，2 屏幕中央，4 弹出菜单，可相加 | `l4d2_map_vote.sp:110` |
| `nqh_bad_samples` | Network Quality Hint | `3` | 整数，源码写作浮点 | 1.0 ~ 20.0 | 警告玩家前需要的连续不良采样次数，范围 1~20 | `network_quality_hint.sp:106` |
| `nqh_check_interval` | Network Quality Hint | `5.0` | 整数，源码写作浮点 | 5.0 ~ 300.0 | 本地网络采样间隔秒数，范围 5~300 | `network_quality_hint.sp:102` |
| `nqh_choke_limit` | Network Quality Hint | `5.0` | 整数，源码写作浮点 | ≥ -1.0 | choke 高于该百分比时警告，-1 关闭 choke 检测 | `network_quality_hint.sp:105` |
| `nqh_enable` | Network Quality Hint | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用网络质量检测 | `network_quality_hint.sp:101` |
| `nqh_intro_delay` | Network Quality Hint | `25.0` | 整数，源码写作浮点 | 0.0 ~ 300.0 | 玩家加入后多少秒打印一次状态提示，0 关闭，范围 0~300 | `network_quality_hint.sp:109` |
| `nqh_ip_page_url` | Network Quality Hint | `https://anne.trygek.com/ip.php` | 字符串或表达式 | 无上下界 | 列出服务器 IP 与可复制连接命令的网页地址 | `network_quality_hint.sp:108` |
| `nqh_loss_limit` | Network Quality Hint | `2.0` | 整数，源码写作浮点 | ≥ -1.0 | 丢包率高于该百分比时警告，-1 关闭丢包检测 | `network_quality_hint.sp:104` |
| `nqh_ping_limit` | Network Quality Hint | `120` | 整数，源码写作浮点 | ≥ -1.0 | ping 高于该毫秒值时警告，-1 关闭 ping 检测 | `network_quality_hint.sp:103` |
| `nqh_report_bad_samples` | Network Quality Hint | `1` | 整数，源码写作浮点 | 1.0 ~ 20.0 | 上报事件所需的连续不良本地采样次数，范围 1~20 | `network_quality_hint.sp:114` |
| `nqh_report_choke_limit` | Network Quality Hint | `20.0` | 整数，源码写作浮点 | ≥ -1.0 | choke 高于该百分比时才上报事件，-1 关闭 choke 事件 | `network_quality_hint.sp:113` |
| `nqh_report_enable` | Network Quality Hint | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 配置了 URL 时是否启用可选的 HTTPS 玩家连接质量上报 | `network_quality_hint.sp:110` |
| `nqh_report_interval` | Network Quality Hint | `600` | 整数，源码写作浮点 | 60.0 ~ 3600.0 | 常规汇总上报的间隔秒数，范围 60~3600 | `network_quality_hint.sp:112` |
| `nqh_report_recovery_samples` | Network Quality Hint | `3` | 整数，源码写作浮点 | 1.0 ~ 20.0 | 上报恢复所需的连续良好本地采样次数，范围 1~20 | `network_quality_hint.sp:115` |
| `nqh_report_url` | Network Quality Hint | `http://anne.trygek.com/api/player/connection_quality.php` | 字符串或表达式 | 无上下界 | 玩家连接质量上报的 HTTP 端点 | `network_quality_hint.sp:111` |
| `nqh_version` | Network Quality Hint | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | `network_quality_hint.sp:99` |
| `nqh_warn_cooldown` | Network Quality Hint | `180.0` | 整数，源码写作浮点 | 30.0 ~ 1800.0 | 同一玩家再次被警告前的间隔秒数，范围 30~1800 | `network_quality_hint.sp:107` |
| `pickup_incap_flags` | [L4D & 2] Pick-up Changes | `7` | 整数，源码写作浮点 | 0.0 ~ 7.0 | 倒地生还者终止拾取进度的 flag，范围 0~7：1 毒痰伤害，2 坦克拳击，4 坦克石头 | `l4d2_pickup.sp:224` |
| `pickup_switch_flags` | [L4D & 2] Pick-up Changes | `0` | 整数，源码写作浮点 | 0.0 ~ 7.0 | 切换所拾取物品的 flag，范围 0~7：1 默认切换到拾取的副武器，2 永不切换到被给予的止痛药或肾上腺素，其余被截断 | `l4d2_pickup.sp:215` |
| `pill_passer_lag_compensate` | Easier Pill Passer | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 传递止痛药时是否启用延迟补偿 | `pill_passer.sp:55` |
| `pill_passer_los_clear` | Easier Pill Passer | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 传递止痛药时是否要求视线通畅 | `pill_passer.sp:48` |
| `pill_passer_range` | Easier Pill Passer | `274.0` | 整数，源码写作浮点 | ≥ 0.0 | 玩家之间传递止痛药的最大距离 | `pill_passer.sp:41` |
| `prop_heavy_touching_move_above` | [L4D & 2] Prop Touching Rules | `0` | 开关 0 或 1 | ≥ 0.0 | 是否阻止玩家被推上重型道具上方：0 关闭，1 生还者，2 除坦克外的特感，4 坦克，7 全部 | `l4d_prop_touching_rules.sp:78` |
| `prop_moveaway_mass_thres` | [L4D & 2] Prop Touching Rules | `900.0` | 整数，源码写作浮点 | ≥ 0.0 | 允许被推开的中等重量道具的最大质量，仅当 prop_touching_moveaway 开启时有效 | `l4d_prop_touching_rules.sp:66` |
| `prop_touching_moveaway` | [L4D & 2] Prop Touching Rules | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否在触碰时推开中等重量道具 | `l4d_prop_touching_rules.sp:72` |
| `punch_angle_toggle` | [L4D2] Punch Angle (RPG-aware, recoil command) | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否开启后坐力 | `punch_angle.sp:71` |
| `punch_angle_version` | [L4D2] Punch Angle (RPG-aware, recoil command) | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | `punch_angle.sp:61` |
| `ReturnBlood` | 商店插件 | `0` | 整数 | 无上下界 | 击杀特感回血（源码描述只有“回血模式”四字）；据 `extend/rpg.sp:1120` 的 `CreateConVar("ReturnBlood", "0", "回血模式")`、:1079 把 `EventReturnBlood` 挂在 `player_death`（EventHookMode_Pre），:1303-1334 中当死者为特感(team 3)、攻击者为幸存者且 `GetConVarBool(ReturnBlood)` 为真时，把攻击者永久血量写回为「当前永久血量（`player[attacker].ClientBlood>0` 时再 +2）」并以 `m_iMaxHealth` 封顶（:1319-1330），:2477 为真时购买菜单追加“回血技能”项，`optional/AnneHappy/text.sp:256-261` 也按“>0”显示回血已开启，推测为：纯开关，0=关闭、非 0=开启（源码用 `GetConVarBool` 读取，不存在多档取值语义）（置信度：高） | `rpg.sp:1120` |
| `rm_allowed_rate_changes` | RateMonitor | `-1` | 整数 | 无上下界 | 单局内允许修改 rate 的次数，-1 表示不限 | `ratemonitor.sp:64` |
| `rm_countermeasure` | RateMonitor | `2` | 整数，源码写作浮点 | 1.0 ~ 3.0 | 违规处理方式：1 聊天提醒，2 移到旁观，3 踢出 | `ratemonitor.sp:70` |
| `rm_min_cmd` | RateMonitor | `20` | 整数 | 无上下界 | 允许的 cl_cmdrate 最小值，-1 表示不限 | `ratemonitor.sp:68` |
| `rm_min_rate` | RateMonitor | `20000` | 整数 | 无上下界 | 允许的 rate 最小值，-1 表示不限 | `ratemonitor.sp:66` |
| `rm_min_upd` | RateMonitor | `20` | 整数 | 无上下界 | 允许的 cl_updaterate 最小值，-1 表示不限 | `ratemonitor.sp:67` |
| `rm_no_fake_ping` | RateMonitor | `0` | 整数 | 无上下界 | 是否允许在网络设置中使用 + - . 来隐藏真实 ping | `ratemonitor.sp:69` |
| `rm_public_notice` | RateMonitor | `0` | 整数 | 无上下界 | 是否向公众打印 rate 变更提示 | `ratemonitor.sp:65` |
| `rock_stumble_throwing` | Tank Rock Stumble Block | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 被硬直时是否仍继续投掷石头，关闭后坦克被硬直会受到较大的移动惩罚 | `rock_stumble_block.sp:39` |
| `rpg_allow_biggun` | 商店插件 | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 商店是否允许购买大枪 | `rpg.sp:1086` |
| `rpg_allow_glow` | 商店插件 | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 商店是否打开轮廓 | `rpg.sp:1087` |
| `rpg_allow_UseB` | 商店插件 | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否允许消费 B 数，即价格大于 0 的商品：1 允许，0 仅允许 0B 商品 | `rpg.sp:1094` |
| `rpg_antikick_block_cmdkick` | 商店插件 | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否禁止低级或同级管理员用 sm_kick 踢受保护管理员 | `rpg.sp:1091` |
| `rpg_antikick_block_votekick` | 商店插件 | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否禁止对受保护管理员发起投票踢 | `rpg.sp:1090` |
| `rpg_antikick_enable` | 商店插件 | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用管理员防踢（投票与命令） | `rpg.sp:1089` |
| `rpg_antikick_equal_block` | 商店插件 | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 同级免疫是否禁止互踢，仅对 sm_kick 生效 | `rpg.sp:1093` |
| `rpg_antikick_min_immunity` | 商店插件 | `0` | 整数，源码写作浮点 | 0.0 ~ 100.0 | 受保护阈值：管理员免疫等级大于等于该值即受保护，0 表示任意管理员都受保护 | `rpg.sp:1092` |
| `rygive_version` | rygive.sp（myinfo 缺 name，用文件名代替） | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | `rygive.sp:250` |
| `sackorder_debug` | [L4D2] Proper Sack Order | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否开启调试 | `l4d2_fix_spawn_order.sp:64` |
| `sam_vs_force_kick` | [L4D & L4D2] Simple AFK Manager VS | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 换图时是否踢出所有闲置的旁观玩家，用于与其他插件的兼容 | `sam_vs.sp:54` |
| `sam_vs_kick_time` | [L4D & L4D2] Simple AFK Manager VS | `120` | 整数，源码写作浮点 | ≥ 0.0 | 闲置旁观玩家被踢出前的等待秒数，0 表示永不踢出 | `sam_vs.sp:50` |
| `sam_vs_respect_admins` | [L4D & L4D2] Simple AFK Manager VS | `k` | 字符串或表达式 | 无上下界 | 管理员免疫 AFK 管理器的 flag 值，留空表示不保护管理员 | `sam_vs.sp:53` |
| `sam_vs_respect_spec` | [L4D & L4D2] Simple AFK Manager VS | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 已不再挂机的旁观玩家是否不踢出 | `sam_vs.sp:51` |
| `sam_vs_respect_tank` | [L4D & L4D2] Simple AFK Manager VS | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 正在扮演 Tank 的挂机玩家是否不移到旁观 | `sam_vs.sp:52` |
| `sam_vs_spec_time` | [L4D & L4D2] Simple AFK Manager VS | `35` | 整数，源码写作浮点 | ≥ 10.0 | 闲置玩家被移到旁观者前的等待秒数，最小 10 | `sam_vs.sp:49` |
| `sam_vs_version` | [L4D & L4D2] Simple AFK Manager VS | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | `sam_vs.sp:47` |
| `sb_fix_bash_enabled` | L4D2 Survivor Bot Fix | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否推击飞扑中的 Hunter 或 Jockey：0 关，1 开 | `l4d2_sb_fix.sp:287` |
| `sb_fix_bash_hunter_chance` | L4D2 Survivor Bot Fix | `100` | 整数，源码写作浮点 | 0.0 ~ 100.0 | 推击飞扑中 Hunter 的概率百分比，1~100 | `l4d2_sb_fix.sp:288` |
| `sb_fix_bash_hunter_range` | L4D2 Survivor Bot Fix | `145` | 整数，源码写作浮点 | 1.0 ~ 500.0 | 推击或搜索飞扑中 Hunter 的距离范围，1~500 | `l4d2_sb_fix.sp:289` |
| `sb_fix_bash_jockey_chance` | L4D2 Survivor Bot Fix | `100` | 整数，源码写作浮点 | 0.0 ~ 100.0 | 推击飞扑中 Jockey 的概率百分比，1~100 | `l4d2_sb_fix.sp:290` |
| `sb_fix_bash_jockey_range` | L4D2 Survivor Bot Fix | `125` | 整数，源码写作浮点 | 1.0 ~ 500.0 | 推击或搜索飞扑中 Jockey 的距离范围，1~500 | `l4d2_sb_fix.sp:291` |
| `sb_fix_ci_enabled` | L4D2 Survivor Bot Fix | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否处理普通感染者：0 关，1 开 | `l4d2_sb_fix.sp:272` |
| `sb_fix_ci_melee_allow` | L4D2 Survivor Bot Fix | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否允许用近战武器处理：0 关，1 开 | `l4d2_sb_fix.sp:274` |
| `sb_fix_ci_melee_range` | L4D2 Survivor Bot Fix | `160` | 整数，源码写作浮点 | 1.0 ~ 500.0 | 近战处理普通感染者的距离范围，1~500 | `l4d2_sb_fix.sp:275` |
| `sb_fix_ci_range` | L4D2 Survivor Bot Fix | `500` | 整数，源码写作浮点 | 1.0 ~ 2000.0 | 搜索或射击普通感染者的距离范围，1~2000 | `l4d2_sb_fix.sp:273` |
| `sb_fix_debug` | L4D2 Survivor Bot Fix | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 调试用：是否打印行动状态 | `l4d2_sb_fix.sp:308` |
| `sb_fix_dont_switch_secondary` | L4D2 Survivor Bot Fix | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 在副武器弹药耗尽前不允许切换到副武器：0 否，1 是 | `l4d2_sb_fix.sp:265` |
| `sb_fix_enabled` | L4D2 Survivor Bot Fix | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用插件：0 关，1 开 | `l4d2_sb_fix.sp:259` |
| `sb_fix_help_enabled` | L4D2 Survivor Bot Fix | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否帮助被控制的生还者：0 关，1 开 | `l4d2_sb_fix.sp:267` |
| `sb_fix_help_range` | L4D2 Survivor Bot Fix | `1200` | 整数，源码写作浮点 | 1.0 ~ 3000.0 | 搜索或射击被控制生还者的距离范围，1~3000 | `l4d2_sb_fix.sp:268` |
| `sb_fix_help_shove_reloading` | L4D2 Survivor Bot Fix | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 当 help_shove_type 为 2 及以上时是否仅在换弹时推击：0 否，1 是 | `l4d2_sb_fix.sp:270` |
| `sb_fix_help_shove_type` | L4D2 Survivor Bot Fix | `2` | 整数，源码写作浮点 | 0.0 ~ 3.0 | 是否用推击帮助：0 不推，1 仅 Smoker，2 Smoker 与 Jockey，3 再加 Hunter | `l4d2_sb_fix.sp:269` |
| `sb_fix_incapacitated_enabled` | L4D2 Survivor Bot Fix | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用倒地相关命令：0 关，1 开 | `l4d2_sb_fix.sp:306` |
| `sb_fix_prioritize_ownersmoker` | L4D2 Survivor Bot Fix | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否优先处理正在控制自己的 Smoker：0 否，1 是 | `l4d2_sb_fix.sp:304` |
| `sb_fix_rock_enabled` | L4D2 Survivor Bot Fix | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否射击 Tank 投掷的石头：0 关，1 开 | `l4d2_sb_fix.sp:293` |
| `sb_fix_rock_range` | L4D2 Survivor Bot Fix | `700` | 整数，源码写作浮点 | 1.0 ~ 2000.0 | 搜索或射击 Tank 石头的距离范围，1~2000 | `l4d2_sb_fix.sp:294` |
| `sb_fix_select_character_name` | L4D2 Survivor Bot Fix | `` | 字符串或表达式 | 无上下界 | 当 sb_fix_select_type 为 4 时，指定要优化的角色名，空格分隔 | `l4d2_sb_fix.sp:263` |
| `sb_fix_select_number` | L4D2 Survivor Bot Fix | `1` | 整数，源码写作浮点 | ≥ 0.0 | 当 sb_fix_select_type 为 1 时，随机优化的 Bot 数量，0~4 | `l4d2_sb_fix.sp:262` |
| `sb_fix_select_type` | L4D2 Survivor Bot Fix | `0` | 整数，源码写作浮点 | 0.0 ~ 2.0 | 优化哪些生还者 Bot：0 全部，1 离开安全区时随机选若干名，2 按指定角色名 | `l4d2_sb_fix.sp:261` |
| `sb_fix_si_enabled` | L4D2 Survivor Bot Fix | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否处理特殊感染者：0 关，1 开 | `l4d2_sb_fix.sp:277` |
| `sb_fix_si_ignore_boomer` | L4D2 Survivor Bot Fix | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否忽略生还者附近的 Boomer 并推开它：0 否，1 是 | `l4d2_sb_fix.sp:279` |
| `sb_fix_si_ignore_boomer_range` | L4D2 Survivor Bot Fix | `200` | 整数，源码写作浮点 | 1.0 ~ 500.0 | 忽略 Boomer 的距离范围，源码正文注明 1~900 | `l4d2_sb_fix.sp:280` |
| `sb_fix_si_range` | L4D2 Survivor Bot Fix | `500` | 整数，源码写作浮点 | 1.0 ~ 3000.0 | 搜索或射击特殊感染者的距离范围，1~3000 | `l4d2_sb_fix.sp:278` |
| `sb_fix_si_tank_priority_type` | L4D2 Survivor Bot Fix | `0` | 整数，源码写作浮点 | 0.0 ~ 2.0 | 范围内同时有特感与 Tank 时优先目标：0 最近的，1 特感优先，其余源码描述被截断 | `l4d2_sb_fix.sp:285` |
| `sb_fix_tank_enabled` | L4D2 Survivor Bot Fix | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否处理 Tank：0 关，1 开 | `l4d2_sb_fix.sp:282` |
| `sb_fix_tank_range` | L4D2 Survivor Bot Fix | `1200` | 整数，源码写作浮点 | 1.0 ~ 3000.0 | 搜索或射击 Tank 的距离范围，1~3000 | `l4d2_sb_fix.sp:283` |
| `sb_fix_witch_enabled` | L4D2 Survivor Bot Fix | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否射击被激怒的 Witch：0 关，1 开 | `l4d2_sb_fix.sp:296` |
| `sb_fix_witch_range` | L4D2 Survivor Bot Fix | `1500` | 整数，源码写作浮点 | 1.0 ~ 2000.0 | 搜索或射击被激怒 Witch 的距离范围，1~2000 | `l4d2_sb_fix.sp:297` |
| `sb_fix_witch_range_incapacitated` | L4D2 Survivor Bot Fix | `1000` | 整数，源码写作浮点 | 0.0 ~ 2000.0 | 搜索或射击使生还者倒地的 Witch 的距离范围，0~2000 | `l4d2_sb_fix.sp:298` |
| `sb_fix_witch_range_killed` | L4D2 Survivor Bot Fix | `0` | 整数，源码写作浮点 | 0.0 ~ 2000.0 | 搜索或射击杀死生还者的 Witch 的距离范围，0 表示不处理，0~2000 | `l4d2_sb_fix.sp:299` |
| `sb_fix_witch_shotgun_control` | L4D2 Survivor Bot Fix | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 若持有霰弹枪，是否由插件控制对 Witch 的开火时机：0 关，1 开 | `l4d2_sb_fix.sp:300` |
| `sb_fix_witch_shotgun_range_max` | L4D2 Survivor Bot Fix | `300` | 整数，源码写作浮点 | 1.0 ~ 1000.0 | Witch 距离在该值以内则停止攻击，1~1000 | `l4d2_sb_fix.sp:301` |
| `sb_fix_witch_shotgun_range_min` | L4D2 Survivor Bot Fix | `70` | 整数，源码写作浮点 | 1.0 ~ 500.0 | Witch 距离在该值及以上则停止攻击，1~500 | `l4d2_sb_fix.sp:302` |
| `sb_version` | SourceBans++: Main Plugin | `SB_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（默认值取自 SB_VERSION 宏） | `sbpp_main.sp:170` |
| `sbchecker_version` | SourceBans++: Bans Checker | `VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（默认值取自 VERSION 宏） | `sbpp_checker.sp:55` |
| `sbpp_report_cooldown` | SourceBans++ Report Plugin | `60.0` | 整数，源码写作浮点 | ≥ 0.0 | 同一玩家两次举报之间的冷却秒数 | `sbpp_report.sp:43` |
| `sbpp_report_minlen` | SourceBans++ Report Plugin | `10` | 整数，源码写作浮点 | ≥ 0.0 | 举报理由的最小长度 | `sbpp_report.sp:44` |
| `sbpp_report_version` | SourceBans++ Report Plugin | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | `sbpp_report.sp:41` |
| `sbr_version` | SourceBans++: Main Plugin | `SBR_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（默认值取自 SBR_VERSION 宏） | `sbpp_main.sp:171` |
| `sceneprocessor_jailbreak_vocalize` | Scene Processor | `1` | 整数 | 无上下界 | 内鬼模式发泄命令的 ConVar 名，源码中由宏 CONVAR_JAILBREAK_VOCALIZE_COMMAND_NAME 定义 | `sceneprocessor.sp:186` |
| `sceneprocessor_version` | Scene Processor | `1.0.1` | 字符串 | 无上下界 | 插件版本号，源码中由宏 CONVAR_VERSION_NAME 定义 | `sceneprocessor.sp:192` |
| `setscore_allow_player_vote` | SetScores | `1` | 整数 | 无上下界 | 是否允许玩家发起投票：1 允许（默认），0 不允许 | `l4d2_setscores.sp:59` |
| `setscore_force_admin_vote` | SetScores | `0` | 整数 | 无上下界 | 管理员修改分数是否需要投票：1 需要，0 不需要（默认） | `l4d2_setscores.sp:60` |
| `setscore_player_limit` | SetScores | `2` | 整数 | 无上下界 | 发起投票所需的最少在场玩家数 | `l4d2_setscores.sp:58` |
| `sgspread_center_pellet` | L4D2 Static Shotgun Spread | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 中心弹丸：0 关闭，1 开启 | `l4d2_static_shotgun_spread.sp:78` |
| `sgspread_ring1_bullets` | L4D2 Static Shotgun Spread | `3` | 整数 | 无上下界 | 第一圈环的弹丸数量，其余弹丸进入第二圈 | `l4d2_static_shotgun_spread.sp:76` |
| `sgspread_ring1_factor` | L4D2 Static Shotgun Spread | `2` | 整数 | 无上下界 | 第一圈环的弹丸距中心的远近系数 | `l4d2_static_shotgun_spread.sp:77` |
| `shop_enable` | 商店插件 | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否打开商店购买 | `rpg.sp:1085` |
| `show_mic_center_hat_enable` | [L4D2] Voice Announce + Show MIC Hat. | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 为 1 时在说话玩家的头顶显示帽子 | `show_mic.sp:54` |
| `show_mic_center_text_enable` | [L4D2] Voice Announce + Show MIC Hat. | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 为 1 时在中心文本显示玩家说话提示 | `show_mic.sp:55` |
| `show_mic_version` | [L4D2] Voice Announce + Show MIC Hat. | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | `show_mic.sp:56` |
| `si_announce_print` | Special Infected Class Announce | `1` | 整数，源码写作浮点 | 0.0 ~ 3.0 | 插件输出公告的位置：0 禁用，1 聊天，2 提示框，3 聊天与提示框 | `si_class_announce.sp:72` |
| `si_announce_ready_footer` | Special Infected Class Announce | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否把特感职业字符串作为页脚加入准备面板 | `si_class_announce.sp:67` |
| `SI_enable_option` | SI target limit | `53` | 整数，源码写作浮点 | 0.0 ~ 127.0 | 控制类特感掩码：1 舌头，2 Boomer，4 猎人，8 Spitter，16 猴子，32 牛，64 Tank；默认 53 为舌头加猎人加猴子加牛共用控制上限 | `SI_Target_limit.sp:133` |
| `si_restore_ratio` | Despawn Health | `0.5` | 浮点 | ≤ 1.0 | 应恢复玩家多少已损失生命，0 或负值表示关闭，1.0 表示补满 | `despawn_health.sp:22` |
| `SI_target_enable` | SI target limit | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否开启插件 | `SI_Target_limit.sp:132` |
| `SI_target_limit_auto` | SI target limit | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 自动上限：已启用控制类职业预算除以可动生还者再加 1，预算为对应 z_* 与 inf_* 上限之和并钳在 l4d_infected_limit 内 | `SI_Target_limit.sp:134` |
| `SI_target_limit_manual` | SI target limit | `3` | 整数 | 无上下界 | 服务器不自动限制时手动设置的最大目标数 | `SI_Target_limit.sp:135` |
| `SI_target_rushman_scope` | SI target limit | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | infected_control 检测到跑男时放开上限的范围：0 全体生还者抬到特感总上限，1 仅跑男本人放开 | `SI_Target_limit.sp:136` |
| `simple_antibhop_enable` | Simple Anti-Bunnyhop | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 总开关：0 关闭 Simple Anti-Bhop，1 启用 | `l4d2_nobhaps.sp:53` |
| `simple_antibhop_enable` | Simple Anti-Bunnyhop | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用 Simple Anti-Bhop 插件：0 关，1 开 | `l4d2_nobhaps.sp:53` |
| `slots_max_slots` | Slots?! Voter | `16` | 整数，源码写作浮点 | 1.0 ~ float(SLOT_HARD_MAX) | 玩家可投票的槽位上限，上限为 SLOT_HARD_MAX | `slots_vote.sp:30` |
| `slots_nonAdmin_min_slots` | Slots?! Voter | `4` | 整数，源码写作浮点 | 1.0 ~ float(SLOT_HARD_MAX) | 非管理员玩家可投票的最小槽位，上限为 SLOT_HARD_MAX | `slots_vote.sp:31` |
| `sm2_bonus_per_survivor_multiplier` | L4D2 Scoremod+ | `0.5` | 浮点 | 无上下界 | 生还者奖励总分等于该值乘以生还者人数再乘以地图距离 | `l4d2_hybrid_scoremod.sp:79` |
| `sm2_bonus_per_survivor_multiplier` | L4D2 Scoremod+ | `0.5` | 浮点 | 无上下界 | 与 1209 同义，另一文件中的同名 ConVar | `l4d2_hybrid_scoremod_zone.sp:78` |
| `sm2_permament_health_proportion` | L4D2 Scoremod+ | `0.75` | 浮点 | 无上下界 | 永久生命奖励比例，其余计入临时生命奖励 | `l4d2_hybrid_scoremod.sp:80` |
| `sm2_permament_health_proportion` | L4D2 Scoremod+ | `0.75` | 浮点 | 无上下界 | 与 1210 同义，另一文件中的同名 ConVar | `l4d2_hybrid_scoremod_zone.sp:79` |
| `sm2_pills_hp_factor` | L4D2 Scoremod+ | `6.0` | 整数，源码写作浮点 | 无上下界 | 未使用止痛药折算的血量等于地图奖励血量除以该值 | `l4d2_hybrid_scoremod.sp:81` |
| `sm2_pills_hp_factor` | L4D2 Scoremod+ | `6.0` | 整数，源码写作浮点 | 无上下界 | 与 1211 同义，另一文件中的同名 ConVar | `l4d2_hybrid_scoremod_zone.sp:80` |
| `sm2_pills_max_bonus` | L4D2 Scoremod+ | `30` | 整数 | 无上下界 | 未使用止痛药折算奖励的上限 | `l4d2_hybrid_scoremod.sp:82` |
| `sm2_pills_max_bonus` | L4D2 Scoremod+ | `30` | 整数 | 无上下界 | 与 1212 同义，另一文件中的同名 ConVar | `l4d2_hybrid_scoremod_zone.sp:81` |
| `sm_1v1_dmgthreshold` | 1v1 EQ | `24` | 整数，源码写作浮点 | ≥ 1.0 | 特感自杀前一次性受到的伤害量，最小 1 | `1v1.sp:45` |
| `SM_adrenaline_health_buffer` | L4D2 Scoremod | `buf` | 字符串或表达式 | 无上下界 | 肾上腺素增加的生命缓冲值 | `l4d2_scoremod.sp:108` |
| `sm_advertisements_enabled` | Advertisements | `1` | 整数 | 无上下界 | 是否启用广告显示 | `advertisements.sp:64` |
| `sm_advertisements_file` | Advertisements | `advertisements.txt` | 字符串或表达式 | 无上下界 | 读取广告的文本文件名 | `advertisements.sp:65` |
| `sm_advertisements_interval` | Advertisements | `30` | 整数 | 无上下界 | 两条广告之间的间隔秒数 | `advertisements.sp:66` |
| `sm_advertisements_random` | Advertisements | `0` | 整数 | 无上下界 | 是否随机播放广告 | `advertisements.sp:67` |
| `sm_advertisements_version` | Advertisements | `PL_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | `advertisements.sp:63` |
| `sm_aidmgfix_enable` | Bot SI skeet/level damage fix | `3` | 整数 | 无上下界 | 位标志：1 修复飞扑中的 AI 被击杀判定，2 削弱冲锋中的 AI，3 全部启用，0 关闭 | `l4d2_ai_damagefix.sp:99` |
| `sm_allow_spectate_command` | Player Management Plugin | `1` | 整数 | 无上下界 | 是否允许玩家使用 !spectate/!spec/!s 命令 | `playermanagement.sp:75` |
| `sm_allowed_lerp_changes` | LerpMonitor++ | `1` | 整数，源码写作浮点 | 0.0 ~ 20.0 | 半场内允许的 lerp 修改次数，范围 0~20 | `lerpmonitor.sp:69` |
| `sm_anne_mode_guide_enable` | Anne Telecom Server Mode Guide | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用自动的 Anne 模式引导提示 | `new_player_guide.sp:233` |
| `sm_anne_mode_guide_initial_delay` | Anne Telecom Server Mode Guide | `1.0` | 整数，源码写作浮点 | 1.0 ~ 120.0 | 首次自动模式引导检查前的延迟秒数，范围 1~120 | `new_player_guide.sp:234` |
| `sm_anne_mode_guide_max_retries` | Anne Telecom Server Mode Guide | `10` | 整数，源码写作浮点 | 0.0 ~ 30.0 | 等待游戏时长与偏好数据的最大重试次数，范围 0~30 | `new_player_guide.sp:236` |
| `sm_anne_mode_guide_permanent_disable_minutes` | Anne Telecom Server Mode Guide | `600` | 整数，源码写作浮点 | 0.0 ~ 100000.0 | 出现永久关闭提示开关前所需的本服游戏时长分钟数 | `new_player_guide.sp:238` |
| `sm_anne_mode_guide_retry_delay` | Anne Telecom Server Mode Guide | `5.0` | 整数，源码写作浮点 | 1.0 ~ 60.0 | 游戏时长与偏好数据重试之间的延迟秒数，范围 1~60 | `new_player_guide.sp:235` |
| `sm_anne_mode_guide_server_stop_minutes` | Anne Telecom Server Mode Guide | `1800` | 整数，源码写作浮点 | 0.0 ~ 100000.0 | 本服累计游戏时长超过该分钟数后停止自动提示，范围 0~100000 | `new_player_guide.sp:237` |
| `sm_anne_mode_guide_suppress_mode_loaded` | Anne Telecom Server Mode Guide | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 加载 Confogl 比赛模式时是否抑制自动提示 | `new_player_guide.sp:239` |
| `sm_bad_lerp_action` | LerpMonitor++ | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 玩家 lerp 超出允许范围时的处理：1 移到旁观，0 踢出服务器 | `lerpmonitor.sp:71` |
| `sm_bl_admin_flag` | l4d2_blacklist.sp（myinfo 缺 name，用文件名代替） | `b` | 字符串或表达式 | 无上下界 | 享管理员上限所需的权限 flag，留空表示无 | `l4d2_blacklist.sp:248` |
| `sm_bl_consider_teams` | l4d2_blacklist.sp（myinfo 缺 name，用文件名代替） | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否仅检查队伍 2/3：1 是，0 否 | `l4d2_blacklist.sp:249` |
| `sm_bl_db_section` | l4d2_blacklist.sp（myinfo 缺 name，用文件名代替） | `g_sDBSection` | 字符串或表达式 | 无上下界 | databases.cfg 区块名 | `l4d2_blacklist.sp:243` |
| `sm_bl_enable` | l4d2_blacklist.sp（myinfo 缺 name，用文件名代替） | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用插件：1 开，0 关 | `l4d2_blacklist.sp:242` |
| `sm_bl_expose_blocker` | l4d2_blacklist.sp（myinfo 缺 name，用文件名代替） | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 踢出提示是否包含屏蔽者信息：1 是，0 否 | `l4d2_blacklist.sp:255` |
| `sm_bl_immune_flag` | l4d2_blacklist.sp（myinfo 缺 name，用文件名代替） | `z` | 字符串或表达式 | 无上下界 | 免疫 flag，拥有该 flag 的玩家被忽略屏蔽检查，留空则关闭 | `l4d2_blacklist.sp:251` |
| `sm_bl_kick_msg` | l4d2_blacklist.sp（myinfo 缺 name，用文件名代替） | `g_sKickMsg` | 字符串或表达式 | 无上下界 | 踢出提示，留空则使用翻译短语 | `l4d2_blacklist.sp:245` |
| `sm_bl_limit_admin` | l4d2_blacklist.sp（myinfo 缺 name，用文件名代替） | `10` | 整数，源码写作浮点 | ≥ 0.0 | 管理员屏蔽上限 | `l4d2_blacklist.sp:247` |
| `sm_bl_limit_user` | l4d2_blacklist.sp（myinfo 缺 name，用文件名代替） | `3` | 整数，源码写作浮点 | ≥ 0.0 | 普通玩家屏蔽上限 | `l4d2_blacklist.sp:246` |
| `sm_bl_log` | l4d2_blacklist.sp（myinfo 缺 name，用文件名代替） | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用日志记录：1 开，0 关 | `l4d2_blacklist.sp:253` |
| `sm_bl_log_file` | l4d2_blacklist.sp（myinfo 缺 name，用文件名代替） | `g_sLogFile` | 字符串或表达式 | 无上下界 | 日志文件名，位于 logs/ 下 | `l4d2_blacklist.sp:254` |
| `sm_bl_mutual` | l4d2_blacklist.sp（myinfo 缺 name，用文件名代替） | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用双向不共存：加入者若屏蔽了在场玩家也禁止加入 | `l4d2_blacklist.sp:259` |
| `sm_bl_quiet` | l4d2_blacklist.sp（myinfo 缺 name，用文件名代替） | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 静默模式：1 不通知触发屏蔽的玩家，0 通知 | `l4d2_blacklist.sp:252` |
| `sm_bl_table` | l4d2_blacklist.sp（myinfo 缺 name，用文件名代替） | `g_sTable` | 字符串或表达式 | 无上下界 | 数据库表名 | `l4d2_blacklist.sp:244` |
| `sm_bl_use_i18n_kick` | l4d2_blacklist.sp（myinfo 缺 name，用文件名代替） | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 踢出提示是否使用翻译短语：1 是，0 则优先用 cvar 文本 | `l4d2_blacklist.sp:256` |
| `sm_blast_damage_enable` | L4D2 Hit/Kill Feedback Plus | `0` | 整数 | 无上下界 | 是否开启爆炸反馈提示：0 关，1 开，源码建议关闭 | `l4d2_hitsound.sp:719` |
| `sm_blockspecintank` | Player Management Plugin | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否阻止生还者在扮演坦克时切换到旁观 | `playermanagement.sp:76` |
| `sm_cfgip_url` | simple join | `http://anne.trygek.com/ip.php` | 字符串或表达式 | 无上下界 | 查询 IP 的页面 URL | `join.sp:110` |
| `sm_cfgmotd_language_redirect` | simple join | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否按客户端语言为 !web/MOTD 添加 lang 参数或使用语言专用 URL | `join.sp:103` |
| `sm_cfgmotd_title` | Config Description | `ZoneMod` | 字符串或表达式 | 无上下界 | 自定义 MOTD 标题 | `cfg_motd.sp:20` |
| `sm_cfgmotd_title` | simple join | `AnneHappy电信服` | 字符串或表达式 | 无上下界 | MOTD 标题 | `join.sp:101` |
| `sm_cfgmotd_url` | Config Description | `https://github.com/SirPlease/ZoneMod/blob/master/README.md` | 字符串或表达式 | 无上下界 | 自定义 MOTD 页面 URL | `cfg_motd.sp:21` |
| `sm_cfgmotd_url` | simple join | `http://anne.trygek.com/l4d2/` | 字符串或表达式 | 无上下界 | MOTD 页面 URL | `join.sp:102` |
| `sm_cfgmotd_url_chi` | simple join | `` | 字符串或表达式 | 无上下界 | 简体中文客户端专用 MOTD URL，留空则用 sm_cfgmotd_url | `join.sp:105` |
| `sm_cfgmotd_url_en` | simple join | `` | 字符串或表达式 | 无上下界 | 英语客户端专用 MOTD URL，留空则用 sm_cfgmotd_url | `join.sp:104` |
| `sm_cfgmotd_url_jp` | simple join | `` | 字符串或表达式 | 无上下界 | 日语客户端专用 MOTD URL，留空则用 sm_cfgmotd_url | `join.sp:107` |
| `sm_cfgmotd_url_ko` | simple join | `` | 字符串或表达式 | 无上下界 | 韩语客户端专用 MOTD URL，留空则用 sm_cfgmotd_url | `join.sp:108` |
| `sm_cfgmotd_url_zho` | simple join | `` | 字符串或表达式 | 无上下界 | 繁体中文客户端专用 MOTD URL，留空则用 sm_cfgmotd_url | `join.sp:106` |
| `sm_chatlog_cleartable_duration` | Chat Log | `12 MONTH` | 字符串或表达式 | 无上下界 | 聊天记录表重置的周期 | `chatlog.sp:30` |
| `sm_chatlog_cleartable_enabled` | Chat Log | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用清空聊天记录表：1 开，0 关 | `chatlog.sp:29` |
| `sm_chatprocessor_addgotv` | chat-processor.sp（myinfo 缺 name，用文件名代替） | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否把 GOTV 客户端加入接收列表，仅对有 GOTV/SourceTV 的游戏生效 | `chat-processor.sp:125` |
| `sm_chatprocessor_allchat` | chat-processor.sp（myinfo 缺 name，用文件名代替） | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否允许双方通过团队聊天互相交流：0 关，1 开 | `chat-processor.sp:123` |
| `sm_chatprocessor_colors_flag` | chat-processor.sp（myinfo 缺 name，用文件名代替） | `b` | 字符串或表达式 | 无上下界 | 使用彩色名字与消息所需的权限 flag，需 strip_colors 为 1 | `chat-processor.sp:121` |
| `sm_chatprocessor_config` | chat-processor.sp（myinfo 缺 name，用文件名代替） | `configs/chat_processor.cfg` | 字符串或表达式 | 无上下界 | 消息格式配置文件路径 | `chat-processor.sp:117` |
| `sm_chatprocessor_deadchat` | chat-processor.sp（myinfo 缺 name，用文件名代替） | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 死者聊天开关：0 关，1 开 | `chat-processor.sp:122` |
| `sm_chatprocessor_process_colors_default` | chat-processor.sp（myinfo 缺 name，用文件名代替） | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否默认给转发处理颜色 | `chat-processor.sp:118` |
| `sm_chatprocessor_remove_colors_default` | chat-processor.sp（myinfo 缺 name，用文件名代替） | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否默认给转发移除颜色 | `chat-processor.sp:119` |
| `sm_chatprocessor_restrictdeadchat` | chat-processor.sp（myinfo 缺 name，用文件名代替） | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否完全禁止死者的所有聊天：0 关，1 开 | `chat-processor.sp:124` |
| `sm_chatprocessor_status` | chat-processor.sp（myinfo 缺 name，用文件名代替） | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 插件状态显示开关 | `chat-processor.sp:116` |
| `sm_chatprocessor_strip_colors` | chat-processor.sp（myinfo 缺 name，用文件名代替） | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 输出前是否先移除名字与消息中的颜色标签 | `chat-processor.sp:120` |
| `sm_chatprocessor_version` | chat-processor.sp（myinfo 缺 name，用文件名代替） | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | `chat-processor.sp:114` |
| `sm_clientcommandlogging_version` | SM Client Command Logging | `DATA` | 字符串或表达式 | 无上下界 | 插件版本号（默认值取自 DATA 宏，非行为配置） | `logcommands.sp:54` |
| `SM_custommaxdistance` | L4D2 Scoremod | `0` | 整数 | 无上下界 | 是否使用配置中的自定义最大距离 | `l4d2_scoremod.sp:112` |
| `sm_cvar_test_N` | [ANY] Command and ConVar - Buffer Overflow Fixer | `0` | 整数 | 无上下界 | 仅当源码 DEBUGGING 宏为 1 时创建；名称为 sm_cvar_test_0 到 MAX_CVARS-1，默认值全部为 0 | `command_buffer.sp:150` |
| `sm_dances_admin_flag_menu` | SM Fortnite Emotes Extended | `` | 字符串或表达式 | 无上下界 | 舞蹈管理菜单所需权限 flag，留空表示所有玩家可用 | `fornite_l4d.sp:124` |
| `sm_dmg_allowed_flags` | [L4D2] Damage HUD (MySQL+Cookie, DB-first, fixes) | `` | 字符串或表达式 | 无上下界 | 允许使用本插件的管理员 flag，留空为所有人，例 bc | `l4d2_damage_show.sp:1092` |
| `sm_emotes_add_downloads` | SM Fortnite Emotes Extended | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否由舞蹈插件一次性注册全部模型与音乐下载：1 是，0 否 | `fornite_l4d.sp:129` |
| `sm_emotes_admin_flag_menu` | SM Fortnite Emotes Extended | `` | 字符串或表达式 | 无上下界 | 表情管理菜单所需权限 flag，留空表示所有玩家可用 | `fornite_l4d.sp:123` |
| `sm_emotes_cooldown` | SM Fortnite Emotes Extended | `2.0` | 整数，源码写作浮点 | 无上下界 | 表情冷却秒数，-1 或 0 表示无冷却 | `fornite_l4d.sp:121` |
| `sm_emotes_hide_enemies` | SM Fortnite Emotes Extended | `0` | 整数 | 无上下界 | 跳舞时是否隐藏敌方玩家：1 是，0 否 | `fornite_l4d.sp:126` |
| `sm_emotes_hide_weapons` | SM Fortnite Emotes Extended | `0` | 整数 | 无上下界 | 跳舞时是否隐藏武器：1 是，0 否 | `fornite_l4d.sp:125` |
| `sm_emotes_sounds` | SM Fortnite Emotes Extended | `1` | 整数 | 无上下界 | 是否启用表情声音 | `fornite_l4d.sp:120` |
| `sm_emotes_soundvolume` | SM Fortnite Emotes Extended | `1.0` | 整数，源码写作浮点 | 无上下界 | 舞蹈音量 | `fornite_l4d.sp:122` |
| `sm_emotes_speed` | SM Fortnite Emotes Extended | `0.80` | 浮点 | 无上下界 | 动画播放速度，源码注明默认值为 1.0 | `fornite_l4d.sp:128` |
| `sm_emotes_teleportonend` | SM Fortnite Emotes Extended | `0` | 整数 | 无上下界 | 跳舞开始时是否传送回原地，部分地图需要以此触发传送 | `fornite_l4d.sp:127` |
| `SM_enable` | L4D2 Scoremod | `1` | 整数 | 无上下界 | L4D2 自定义计分系统开关 | `l4d2_scoremod.sp:80` |
| `SM_first_aid_heal_percent` | L4D2 Scoremod | `buf` | 字符串或表达式 | 无上下界 | 医疗包回复生命的百分比 | `l4d2_scoremod.sp:104` |
| `sm_fixscreen_deferred_group_count` | [L4D & L4D2] Additive Staged FastDL | `8` | 整数，源码写作浮点 | 0.0 ~ float(MAX_DEFERRED_GROUPS) | 用于拆分延迟下载文件的分组数量，每次真实换图加入一组，0 表示关闭延迟下载 | `l4d2_blackscreen_fix.sp:52` |
| `sm_hbonus_configpath` | Holdout Bonus | `configs/holdoutmapinfo.txt` | 字符串或表达式 | 无上下界 | 每张地图 Holdout 奖励设置所用的 holdoutmapinfo.txt KeyValues 文件路径 | `holdout_bonus.sp:137` |
| `sm_hbonus_pointsmode` | Holdout Bonus | `2` | 整数，源码写作浮点 | ≥ 0.0 | 奖励计算方式；源码其实已给描述（`optional/holdout_bonus.sp:130-135` 第 3 参数）：「The way the holdout bonus is awarded. 0: disable; 1: leave distance unchanged; 2: substract points from distance.」——0=禁用、1=距离不变、2=从距离中扣除奖励分；:132 的内联注释与描述一致（置信度：高） | `holdout_bonus.sp:130` |
| `sm_hbonus_report` | Holdout Bonus | `2` | 整数，源码写作浮点 | ≥ 0.0 | 奖励播报方式；源码其实已给描述（`optional/holdout_bonus.sp:123-128` 第 3 参数）：「The way the bonus is reported. 0: no report; 1: report only on round end; 2: also report after event; 3: also report when event starts; 4: only report on event end」——0=不播报、1=仅回合结束时播报、2=事件结束后也播报、3=事件开始时也播报、4=仅在事件结束时播报；同处 :125 的内联注释写的是「0: disable; 1: leave distance unchanged; 2: substract points from distance」，与下一条 `sm_hbonus_pointsmode` 的注释重复，疑为复制残留，以正式描述为准（置信度：高） | `holdout_bonus.sp:123` |
| `SM_healthbonusratio` | L4D2 Scoremod | `2.0` | 浮点 | 0.25 ~ 5.0 | 生命奖励倍率，范围 0.25~5 | `l4d2_scoremod.sp:83` |
| `sm_hextags_enable_tagslist` | hextags | `1` | 整数 | 无上下界 | 为 1 时启用 sm_tagslist 命令 | `hextags.sp:141` |
| `sm_hextags_roundend` | hextags | `0` | 整数 | 无上下界 | 为 1 时回合结束也重新加载称号 | `hextags.sp:140` |
| `sm_hextags_version` | hextags | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | `hextags.sp:139` |
| `sm_hitsound_db_conf` | L4D2 Hit/Kill Feedback Plus | `rpg` | 字符串或表达式 | 无上下界 | databases.cfg 中的连接名，运行时修改需重载插件 | `l4d2_hitsound.sp:724` |
| `sm_hitsound_db_enable` | L4D2 Hit/Kill Feedback Plus | `1` | 整数 | 无上下界 | 是否启用 RPG 表存储，运行时修改需重载插件 | `l4d2_hitsound.sp:723` |
| `sm_hitsound_db_table` | L4D2 Hit/Kill Feedback Plus | `RPG` | 字符串或表达式 | 无上下界 | 存储表名，运行时修改需重载插件 | `l4d2_hitsound.sp:725` |
| `sm_hitsound_debug` | L4D2 Hit/Kill Feedback Plus | `0` | 整数 | 无上下界 | 调试输出：0 关，1 开 | `l4d2_hitsound.sp:726` |
| `sm_hitsound_enable` | L4D2 Hit/Kill Feedback Plus | `1` | 整数 | 无上下界 | 是否开启本插件：0 关，1 开 | `l4d2_hitsound.sp:716` |
| `sm_hitsound_overlay_default` | L4D2 Hit/Kill Feedback Plus | `1` | 整数 | 无上下界 | 新玩家默认是否启用覆盖图：1 给套装 1，0 禁用 | `l4d2_hitsound.sp:721` |
| `sm_hitsound_pic_enable` | L4D2 Hit/Kill Feedback Plus | `1` | 整数 | 无上下界 | 是否开启覆盖图标总开关：0 关，1 开 | `l4d2_hitsound.sp:718` |
| `sm_hitsound_showtime` | L4D2 Hit/Kill Feedback Plus | `0.3` | 浮点 | 无上下界 | 覆盖图标显示时长秒数 | `l4d2_hitsound.sp:720` |
| `sm_hitsound_sound_enable` | L4D2 Hit/Kill Feedback Plus | `1` | 整数 | 无上下界 | 是否开启音效：0 关，1 开 | `l4d2_hitsound.sp:717` |
| `sm_initiatorready` | Pause plugin | `0` | 整数，源码写作浮点 | ≥ 0.0 | 是否要求暂停发起者准备后才取消暂停 | `pause.sp:114` |
| `sm_lerp_change_spec` | LerpMonitor++ | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 超过 lerp 修改次数后是否移到旁观 | `lerpmonitor.sp:70` |
| `SM_mapmulti` | L4D2 Scoremod | `1` | 整数 | 无上下界 | 是否把生命奖励上限提升到距离上限 | `l4d2_scoremod.sp:110` |
| `sm_match_player_limit` | Match Vote | `1` | 整数，源码写作浮点 | 1.0 ~ 32.0 | 发起投票所需的最少在场玩家数，范围 1~32 | `match_vote.sp:68` |
| `sm_match_vote_enabled` | Match Vote | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用插件 | `match_vote.sp:66` |
| `sm_max_lerp` | LerpMonitor++ | `0.067` | 浮点 | 0.000 ~ 0.500 | 允许的最大 lerp 值，范围 0.000~0.500 | `lerpmonitor.sp:75` |
| `sm_min_lerp` | LerpMonitor++ | `0.000` | 浮点 | 0.000 ~ 0.500 | 允许的最小 lerp 值，范围 0.000~0.500 | `lerpmonitor.sp:74` |
| `sm_new_player_guide_url` | simple join | `http://anne.trygek.com/l4d2/guide` | 字符串或表达式 | 无上下界 | 新玩家进服自动打开的玩法指南 URL，留空则用 sm_cfgmotd_url | `join.sp:109` |
| `sm_onlyforce` | Pause plugin | `0` | 整数 | 无上下界 | 是否只允许强制暂停与取消暂停功能 | `pause.sp:111` |
| `SM_pain_pills_health_value` | L4D2 Scoremod | `buf` | 字符串或表达式 | 无上下界 | 止痛药增加的生命值 | `l4d2_scoremod.sp:106` |
| `sm_pausedelay` | Pause plugin | `0` | 整数，源码写作浮点 | ≥ 0.0 | 暂停生效前的延迟秒数，可用于防止战术暂停 | `pause.sp:112` |
| `sm_pauselimit` | Pause plugin | `0` | 整数，源码写作浮点 | ≥ 0.0 | 单局内玩家可暂停的次数上限，0 表示不限制 | `pause.sp:115` |
| `sm_pbonus_display` | Penalty bonus system | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否在回合结束与使用 !bonus 时显示奖励 | `l4d2_penalty_bonus.sp:103` |
| `sm_pbonus_enable` | Penalty bonus system | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用扣分奖励系统 | `l4d2_penalty_bonus.sp:102` |
| `sm_pbonus_reportchanges` | Penalty bonus system | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 当前奖励被修改时是否报告变更 | `l4d2_penalty_bonus.sp:104` |
| `sm_pbonus_tank` | Penalty bonus system | `0` | 整数，源码写作浮点 | ≥ 0.0 | 击杀坦克给予的奖励值，0 完全关闭 | `l4d2_penalty_bonus.sp:105` |
| `sm_pbonus_witch` | Penalty bonus system | `0` | 整数，源码写作浮点 | ≥ 0.0 | 击杀 Witch 给予的奖励值，0 完全关闭 | `l4d2_penalty_bonus.sp:106` |
| `sm_qf_blacklist_database` | Anne Global Chat | `l4dstats` | 字符串或表达式 | 无上下界 | l4d2_blacklist 使用的 databases.cfg 配置名 | `global_chat.sp:170` |
| `sm_qf_blacklist_filter` | Anne Global Chat | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否按 l4d2_blacklist 屏蔽全服聊天显示 | `global_chat.sp:169` |
| `sm_qf_blacklist_refresh_interval` | Anne Global Chat | `60.0` | 整数，源码写作浮点 | 10.0 ~ 600.0 | 刷新在线玩家屏蔽缓存的间隔秒数，范围 10~600 | `global_chat.sp:172` |
| `sm_qf_blacklist_table` | Anne Global Chat | `player_blocks` | 字符串或表达式 | 无上下界 | l4d2_blacklist 使用的数据库表名 | `global_chat.sp:171` |
| `sm_qf_cleanup_interval` | Anne Global Chat | `21600` | 整数，源码写作浮点 | ≥ 300.0 | 清理旧全服聊天记录的间隔秒数，最小 300 | `global_chat.sp:166` |
| `sm_qf_database` | Anne Global Chat | `globalchat` | 字符串或表达式 | 无上下界 | databases.cfg 中的数据库配置名 | `global_chat.sp:162` |
| `sm_qf_enabled` | Anne Global Chat | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用全服聊天 | `global_chat.sp:161` |
| `sm_qf_idle_poll_interval` | Anne Global Chat | `30.0` | 整数，源码写作浮点 | 5.0 ~ 300.0 | 无真人玩家时的全服聊天轮询间隔秒数，范围 5~300 | `global_chat.sp:164` |
| `sm_qf_limit_10m` | Anne Global Chat | `20` | 整数，源码写作浮点 | ≥ 0.0 | 积分 1000 万以上玩家每日全服聊天次数上限 | `global_chat.sp:176` |
| `sm_qf_limit_1m` | Anne Global Chat | `5` | 整数，源码写作浮点 | ≥ 0.0 | 积分 100 万以上玩家每日全服聊天次数上限 | `global_chat.sp:174` |
| `sm_qf_limit_20m` | Anne Global Chat | `30` | 整数，源码写作浮点 | ≥ 0.0 | 积分 2000 万以上玩家每日全服聊天次数上限 | `global_chat.sp:177` |
| `sm_qf_limit_5m` | Anne Global Chat | `10` | 整数，源码写作浮点 | ≥ 0.0 | 积分 500 万以上玩家每日全服聊天次数上限 | `global_chat.sp:175` |
| `sm_qf_limit_default` | Anne Global Chat | `3` | 整数，源码写作浮点 | ≥ 0.0 | 普通玩家每日全服聊天次数上限 | `global_chat.sp:173` |
| `sm_qf_limit_kick_admin` | Anne Global Chat | `50` | 整数，源码写作浮点 | ≥ 0.0 | 有 kick 权限但无 z 权限的管理员每日全服聊天次数上限 | `global_chat.sp:178` |
| `sm_qf_poll_batch` | Anne Global Chat | `30` | 整数，源码写作浮点 | 1.0 ~ 200.0 | 每次最多拉取的全服聊天消息条数，范围 1~200 | `global_chat.sp:165` |
| `sm_qf_poll_interval` | Anne Global Chat | `5.0` | 整数，源码写作浮点 | 2.0 ~ 30.0 | 全服聊天轮询间隔秒数，范围 2~30 | `global_chat.sp:163` |
| `sm_qf_prefix` | Anne Global Chat | `[全服]` | 字符串或表达式 | 无上下界 | 全服聊天前缀 | `global_chat.sp:168` |
| `sm_qf_retention_days` | Anne Global Chat | `7` | 整数，源码写作浮点 | ≥ 0.0 | 全服聊天记录保留天数，0 表示不清理 | `global_chat.sp:167` |
| `sm_qf_show_global` | Anne Global Chat | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 本服玩家能否看到普通全服聊天与上线提示 | `global_chat.sp:179` |
| `sm_readypaneltextdelay` | Add Text To Readyup Panel | `4.0` | 整数，源码写作浮点 | 0.0 ~ 10.0 | 把文本加入准备面板前的延迟秒数，范围 0~10 | `panel_text.sp:29` |
| `sm_readyup_lerp_changes` | LerpMonitor++ | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 准备阶段是否允许修改 lerp | `lerpmonitor.sp:72` |
| `sm_rock_damage_magnum` | L4D(2) Tank Rock Lag Compensation | `1000` | 整数，源码写作浮点 | 0.0 ~ DAMAGE_MAX_ALL_ | 马格南类别的石头伤害 | `l4d_rock_lagcomp.sp:159` |
| `sm_rock_damage_melee` | L4D(2) Tank Rock Lag Compensation | `1000` | 整数，源码写作浮点 | 0.0 ~ DAMAGE_MAX_ALL_ | 近战类别的石头伤害 | `l4d_rock_lagcomp.sp:163` |
| `sm_rock_damage_minigun` | L4D(2) Tank Rock Lag Compensation | `300` | 整数，源码写作浮点 | 0.0 ~ DAMAGE_MAX_ALL_ | 转轮机枪类别的石头伤害 | `l4d_rock_lagcomp.sp:165` |
| `sm_rock_damage_mounted_machinegun` | L4D(2) Tank Rock Lag Compensation | `10000` | 整数，源码写作浮点 | 0.0 ~ DAMAGE_MAX_ALL_ | 固定机枪类别的石头伤害 | `l4d_rock_lagcomp.sp:166` |
| `sm_rock_damage_pistol` | L4D(2) Tank Rock Lag Compensation | `75` | 整数，源码写作浮点 | 0.0 ~ DAMAGE_MAX_ALL_ | 手枪类别的石头即时命中伤害，上限为 DAMAGE_MAX_ALL_ 宏 | `l4d_rock_lagcomp.sp:158` |
| `sm_rock_damage_rifle` | L4D(2) Tank Rock Lag Compensation | `200` | 整数，源码写作浮点 | 0.0 ~ DAMAGE_MAX_ALL_ | 步枪类别的石头伤害 | `l4d_rock_lagcomp.sp:162` |
| `sm_rock_damage_shotgun` | L4D(2) Tank Rock Lag Compensation | `600` | 整数，源码写作浮点 | 0.0 ~ DAMAGE_MAX_ALL_ | 霰弹枪类别的石头伤害 | `l4d_rock_lagcomp.sp:160` |
| `sm_rock_damage_smg` | L4D(2) Tank Rock Lag Compensation | `75` | 整数，源码写作浮点 | 0.0 ~ DAMAGE_MAX_ALL_ | SMG 类别的石头伤害 | `l4d_rock_lagcomp.sp:161` |
| `sm_rock_damage_sniper` | L4D(2) Tank Rock Lag Compensation | `10000` | 整数，源码写作浮点 | 0.0 ~ DAMAGE_MAX_ALL_ | 狙击类别的石头伤害 | `l4d_rock_lagcomp.sp:164` |
| `sm_rock_godframes` | L4D(2) Tank Rock Lag Compensation | `1.7` | 浮点 | 0.0 ~ 10.0 | 未检测到石头投掷时的兜底保护秒数，从石头创建算起，范围 0~10 | `l4d_rock_lagcomp.sp:56` |
| `sm_rock_godframes` | L4D(2) Tank Rock Lag Compensation | `1.7` | 浮点 | 0.0 ~ 10.0 | 石头的神圣帧时间，单位秒 | `l4d_rock_lagcomp.sp:154` |
| `sm_rock_godframes_render` | L4D(2) Tank Rock Lag Compensation | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 石头受保护期间是否显示视觉反馈 | `l4d_rock_lagcomp.sp:58` |
| `sm_rock_godframes_render` | L4D(2) Tank Rock Lag Compensation | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否显示神圣帧视觉反馈 | `l4d_rock_lagcomp.sp:155` |
| `sm_rock_hitbox` | L4D(2) Tank Rock Lag Compensation | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用自定义石头命中盒与伤害处理 | `l4d_rock_lagcomp.sp:54` |
| `sm_rock_hitbox` | L4D(2) Tank Rock Lag Compensation | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用石头自定义命中盒 | `l4d_rock_lagcomp.sp:152` |
| `sm_rock_hitbox_radius` | L4D(2) Tank Rock Lag Compensation | `30` | 整数，源码写作浮点 | 0.0 ~ 10000.0 | 自定义石头命中盒半径，范围 0~10000 | `l4d_rock_lagcomp.sp:59` |
| `sm_rock_hitbox_radius` | L4D(2) Tank Rock Lag Compensation | `30` | 整数，源码写作浮点 | 0.0 ~ 10000.0 | 石头命中盒半径 | `l4d_rock_lagcomp.sp:156` |
| `sm_rock_lagcomp` | L4D(2) Tank Rock Lag Compensation | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否对即时命中的石头射击启用延迟补偿 | `l4d_rock_lagcomp.sp:55` |
| `sm_rock_lagcomp` | L4D(2) Tank Rock Lag Compensation | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用延迟补偿 | `l4d_rock_lagcomp.sp:153` |
| `sm_rock_print` | L4D(2) Tank Rock Lag Compensation | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否打印石头伤害与距离数值 | `l4d_rock_lagcomp.sp:53` |
| `sm_rock_print` | L4D(2) Tank Rock Lag Compensation | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否打印石头伤害与距离数值 | `l4d_rock_lagcomp.sp:151` |
| `sm_rock_range_magnum` | L4D(2) Tank Rock Lag Compensation | `2000` | 整数，源码写作浮点 | 0.0 ~ RANGE_MAX_ALL_ | 马格南类别可造成石头伤害的最大距离 | `l4d_rock_lagcomp.sp:171` |
| `sm_rock_range_max_all` | L4D(2) Tank Rock Lag Compensation | `2000` | 整数，源码写作浮点 | 0.0 ~ 10000.0 | 石头伤害的全局最大距离，0 表示不限制，范围 0~10000 | `l4d_rock_lagcomp.sp:61` |
| `sm_rock_range_max_all` | L4D(2) Tank Rock Lag Compensation | `2000` | 整数，源码写作浮点 | 0.0 ~ RANGE_MAX_ALL_ | 石头伤害的全局最大距离 | `l4d_rock_lagcomp.sp:169` |
| `sm_rock_range_melee` | L4D(2) Tank Rock Lag Compensation | `200` | 整数，源码写作浮点 | 0.0 ~ RANGE_MAX_ALL_ | 近战类别可造成石头伤害的最大距离 | `l4d_rock_lagcomp.sp:175` |
| `sm_rock_range_min_all` | L4D(2) Tank Rock Lag Compensation | `1` | 整数，源码写作浮点 | 0.0 ~ 10000.0 | 即时命中石头伤害的全局最小距离，范围 0~10000 | `l4d_rock_lagcomp.sp:60` |
| `sm_rock_range_min_all` | L4D(2) Tank Rock Lag Compensation | `1` | 整数，源码写作浮点 | 0.0 ~ RANGE_MAX_ALL_ | 石头伤害的全局最小距离，上限为 RANGE_MAX_ALL_ 宏 | `l4d_rock_lagcomp.sp:168` |
| `sm_rock_range_minigun` | L4D(2) Tank Rock Lag Compensation | `2000` | 整数，源码写作浮点 | 0.0 ~ RANGE_MAX_ALL_ | 转轮机枪类别可造成石头伤害的最大距离 | `l4d_rock_lagcomp.sp:177` |
| `sm_rock_range_mounted_machinegun` | L4D(2) Tank Rock Lag Compensation | `10000` | 整数，源码写作浮点 | 0.0 ~ RANGE_MAX_ALL_ | 固定机枪类别可造成石头伤害的最大距离 | `l4d_rock_lagcomp.sp:178` |
| `sm_rock_range_pistol` | L4D(2) Tank Rock Lag Compensation | `2000` | 整数，源码写作浮点 | 0.0 ~ RANGE_MAX_ALL_ | 手枪类别可造成石头伤害的最大距离 | `l4d_rock_lagcomp.sp:170` |
| `sm_rock_range_rifle` | L4D(2) Tank Rock Lag Compensation | `2000` | 整数，源码写作浮点 | 0.0 ~ RANGE_MAX_ALL_ | 步枪类别可造成石头伤害的最大距离 | `l4d_rock_lagcomp.sp:174` |
| `sm_rock_range_shotgun` | L4D(2) Tank Rock Lag Compensation | `1000` | 整数，源码写作浮点 | 0.0 ~ RANGE_MAX_ALL_ | 霰弹枪类别可造成石头伤害的最大距离 | `l4d_rock_lagcomp.sp:172` |
| `sm_rock_range_smg` | L4D(2) Tank Rock Lag Compensation | `2000` | 整数，源码写作浮点 | 0.0 ~ RANGE_MAX_ALL_ | SMG 类别可造成石头伤害的最大距离 | `l4d_rock_lagcomp.sp:173` |
| `sm_rock_range_sniper` | L4D(2) Tank Rock Lag Compensation | `10000` | 整数，源码写作浮点 | 0.0 ~ RANGE_MAX_ALL_ | 狙击类别可造成石头伤害的最大距离 | `l4d_rock_lagcomp.sp:176` |
| `sm_rock_release_godframes` | L4D(2) Tank Rock Lag Compensation | `0.15` | 浮点 | 0.0 ~ 10.0 | 实际投出后石头的保护秒数，0 表示可立即造成伤害，范围 0~10 | `l4d_rock_lagcomp.sp:57` |
| `sm_safeitemkill_enable` | Saferoom Item Remover | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否移除终点安全室的物品 | `l4d2_saferoom_item_remove.sp:47` |
| `sm_safeitemkill_items` | Saferoom Item Remover | `7` | 整数，源码写作浮点 | 0.0 ~ 15.0 | 要移除的物品类型 flag：1 治疗物品，2 枪械，4 近战，8 其他可用物品，范围 0~15 | `l4d2_saferoom_item_remove.sp:49` |
| `sm_safeitemkill_saferooms` | Saferoom Item Remover | `1` | 整数，源码写作浮点 | 0.0 ~ 3.0 | 要清空的安全室 flag：1 终点安全室，2 起点安全室，3 两者 | `l4d2_saferoom_item_remove.sp:48` |
| `sm_show_lerp_team_changes` | LerpMonitor++ | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 玩家换队时是否提示其 lerp | `lerpmonitor.sp:73` |
| `sm_simple_witch_bonus` | Simple Witch Kill Bonus | `25` | 整数，源码写作浮点 | ≥ 0.0 | 干净击杀 Witch 奖励的分数 | `simple_witch_bonus.sp:54` |
| `sm_skeetstat_brevity` | 1v1 SkeetStats | `32` | 整数，源码写作浮点 | ≥ 0.0 | 报告精简位标志：1 隐藏特感，2 隐藏普感，4 隐藏命中率，8 隐藏击杀与死停，32 近战命中率，64 伤害统计，可相加 | `1v1_skeetstats.sp:240` |
| `sm_skeetstat_counttank` | 1v1 SkeetStats | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 为 1 时对 Tank 的伤害计入统计 | `1v1_skeetstats.sp:238` |
| `sm_skeetstat_countwitch` | 1v1 SkeetStats | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 为 1 时对 Witch 的伤害计入统计 | `1v1_skeetstats.sp:239` |
| `sm_skill_bhopinitspeed` | Skill Detection (skeets, crowns, levels) | `150` | 整数，源码写作浮点 | ≥ 0.0 | 连跳第一次起跳的最低速度，0 允许从静止起跳 | `l4d2_skill_detect.sp:514` |
| `sm_skill_bhopkeepspeed` | Skill Detection (skeets, crowns, levels) | `300` | 整数，源码写作浮点 | ≥ 0.0 | 即使没有加速也视为成功连跳的最低速度 | `l4d2_skill_detect.sp:515` |
| `sm_skill_bhopstreak` | Skill Detection (skeets, crowns, levels) | `3` | 整数，源码写作浮点 | ≥ 0.0 | 触发播报的最低连跳次数 | `l4d2_skill_detect.sp:513` |
| `sm_skill_deathcharge_height` | Skill Detection (skeets, crowns, levels) | `400` | 整数，源码写作浮点 | ≥ 0.0 | 判定 Charger 致死冲锋所需带受害者移动的高度差 | `l4d2_skill_detect.sp:511` |
| `sm_skill_detect_debug` | Skill Detection (skeets, crowns, levels) | `0` | 整数 | 无上下界 | 是否启用调试消息 | `l4d2_skill_detect.sp:478` |
| `sm_skill_detect_version` | Skill Detection (skeets, crowns, levels) | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | `l4d2_skill_detect.sp:477` |
| `sm_skill_drawcrown_damage` | Skill Detection (skeets, crowns, levels) | `500` | 整数，源码写作浮点 | ≥ 0.0 | 判定为互爆 Witch 所需的最后一击最低伤害 | `l4d2_skill_detect.sp:506` |
| `sm_skill_hidefakedamage` | Skill Detection (skeets, crowns, levels) | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 若开启，超过受害者生命值的伤害在报告中隐藏 | `l4d2_skill_detect.sp:510` |
| `sm_skill_hunterdp_height` | Skill Detection (skeets, crowns, levels) | `400` | 整数，源码写作浮点 | ≥ 0.0 | 判定为 Hunter 高扑的最低高度 | `l4d2_skill_detect.sp:508` |
| `sm_skill_instaclear_time` | Skill Detection (skeets, crowns, levels) | `0.75` | 浮点 | ≥ 0.0 | 在该秒数内完成解救计入瞬间解救 | `l4d2_skill_detect.sp:512` |
| `sm_skill_jockeydp_height` | Skill Detection (skeets, crowns, levels) | `300` | 整数，源码写作浮点 | ≥ 0.0 | 判定为 Jockey 高跳所需的高度差 | `l4d2_skill_detect.sp:509` |
| `sm_skill_report_bhop` | Skill Detection (skeets, crowns, levels) | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用连跳次数播报 | `l4d2_skill_detect.sp:500` |
| `sm_skill_report_caralarm` | Skill Detection (skeets, crowns, levels) | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用警报车播报 | `l4d2_skill_detect.sp:501` |
| `sm_skill_report_crow` | Skill Detection (skeets, crowns, levels) | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用爆头击杀 Witch 播报 | `l4d2_skill_detect.sp:486` |
| `sm_skill_report_deadcharger` | Skill Detection (skeets, crowns, levels) | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用 Charger 致死冲锋播报 | `l4d2_skill_detect.sp:498` |
| `sm_skill_report_deadstop` | Skill Detection (skeets, crowns, levels) | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用死停播报 | `l4d2_skill_detect.sp:493` |
| `sm_skill_report_drawcrow` | Skill Detection (skeets, crowns, levels) | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用互爆 Witch 播报 | `l4d2_skill_detect.sp:487` |
| `sm_skill_report_enable` | Skill Detection (skeets, crowns, levels) | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否在聊天中播报 | `l4d2_skill_detect.sp:481` |
| `sm_skill_report_hunterdp` | Skill Detection (skeets, crowns, levels) | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用 Hunter 高扑播报 | `l4d2_skill_detect.sp:496` |
| `sm_skill_report_hurtlevel` | Skill Detection (skeets, crowns, levels) | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用受伤单杀播报 | `l4d2_skill_detect.sp:485` |
| `sm_skill_report_hurtskeet` | Skill Detection (skeets, crowns, levels) | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用受伤飞扑击杀播报 | `l4d2_skill_detect.sp:483` |
| `sm_skill_report_instanclear` | Skill Detection (skeets, crowns, levels) | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用瞬间解救播报 | `l4d2_skill_detect.sp:499` |
| `sm_skill_report_jockeydp` | Skill Detection (skeets, crowns, levels) | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用 Jockey 高跳播报 | `l4d2_skill_detect.sp:497` |
| `sm_skill_report_level` | Skill Detection (skeets, crowns, levels) | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用单杀播报 | `l4d2_skill_detect.sp:484` |
| `sm_skill_report_pop` | Skill Detection (skeets, crowns, levels) | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用弹跳击杀播报 | `l4d2_skill_detect.sp:494` |
| `sm_skill_report_rockname` | Skill Detection (skeets, crowns, levels) | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用 Tank 名称播报 | `l4d2_skill_detect.sp:492` |
| `sm_skill_report_rockskeet` | Skill Detection (skeets, crowns, levels) | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用石头击杀播报 | `l4d2_skill_detect.sp:491` |
| `sm_skill_report_sc` | Skill Detection (skeets, crowns, levels) | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用自救播报 | `l4d2_skill_detect.sp:489` |
| `sm_skill_report_scs` | Skill Detection (skeets, crowns, levels) | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用推击自救播报 | `l4d2_skill_detect.sp:490` |
| `sm_skill_report_shove` | Skill Detection (skeets, crowns, levels) | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用推击播报 | `l4d2_skill_detect.sp:495` |
| `sm_skill_report_skeet` | Skill Detection (skeets, crowns, levels) | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用飞扑击杀播报 | `l4d2_skill_detect.sp:482` |
| `sm_skill_report_tonguecut` | Skill Detection (skeets, crowns, levels) | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用断舌播报 | `l4d2_skill_detect.sp:488` |
| `sm_skill_selfclear_damage` | Skill Detection (skeets, crowns, levels) | `200` | 整数，源码写作浮点 | ≥ 0.0 | 判定为自救 Smoker 所需的最低伤害 | `l4d2_skill_detect.sp:507` |
| `sm_skill_skeet_allowgl` | Skill Detection (skeets, crowns, levels) | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否把榴弹直击计入飞扑击杀 | `l4d2_skill_detect.sp:505` |
| `sm_skill_skeet_allowmelee` | Skill Detection (skeets, crowns, levels) | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否把近战击杀计入并转发飞扑击杀 | `l4d2_skill_detect.sp:503` |
| `sm_skill_skeet_allowsniper` | Skill Detection (skeets, crowns, levels) | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否把狙击与马格南爆头计入飞扑击杀 | `l4d2_skill_detect.sp:504` |
| `sm_sleuth_actions` | SourceBans++: SourceSleuth | `3` | 整数，源码写作浮点 | 1.0 ~ 4.0 | SourceSleuth 的封禁方式：1 原时长，2 自定义时长，3 双倍时长，4 仅通知管理员 | `sbpp_sleuth.sp:72` |
| `sm_sleuth_adminbypass` | SourceBans++: SourceSleuth | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 0 不启用，1 允许所有有封禁 flag 的管理员跳过该检查 | `sbpp_sleuth.sp:77` |
| `sm_sleuth_bansallowed` | SourceBans++: SourceSleuth | `0` | 整数 | 无上下界 | 采取行动前允许的生效封禁数量 | `sbpp_sleuth.sp:75` |
| `sm_sleuth_bantype` | SourceBans++: SourceSleuth | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 0 对所有时长类型的封禁都处理，1 仅处理永久封禁 | `sbpp_sleuth.sp:76` |
| `sm_sleuth_duration` | SourceBans++: SourceSleuth | `0` | 整数 | 无上下界 | 当 sm_sleuth_actions 为 1 时的封禁时长，0 表示永久 | `sbpp_sleuth.sp:73` |
| `sm_sleuth_excludeold` | SourceBans++: SourceSleuth | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 0 不启用，1 允许把旧封禁排除在检查之外 | `sbpp_sleuth.sp:78` |
| `sm_sleuth_excludetime` | SourceBans++: SourceSleuth | `31536000` | 整数，源码写作浮点 | ≥ 1.0 | 可排除在检查之外的旧封禁时长阈值，单位秒，最小 1 | `sbpp_sleuth.sp:79` |
| `sm_sleuth_prefix` | SourceBans++: SourceSleuth | `sb` | 字符串或表达式 | 无上下界 | SourceBans 数据库表前缀，默认 sb | `sbpp_sleuth.sp:74` |
| `sm_sourcesleuth_version` | SourceBans++: SourceSleuth | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | `sbpp_sleuth.sp:70` |
| `sm_spawnvote_preset_db` | Anne Spawn Vote Menu | `storage-local` | 字符串或表达式 | 无上下界 | 刷特预设数据库配置名，支持 MySQL 或 SQLite | `spawn_vote_menu.sp:114` |
| `sm_spawnvote_preset_table` | Anne Spawn Vote Menu | `spawn_vote_presets` | 字符串或表达式 | 无上下界 | 刷特预设数据库表名 | `spawn_vote_menu.sp:115` |
| `sm_srvcln_demos` | Server Clean Up | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 源码描述仅为 desc，未说明具体行为 | `servercleanup.sp:132` |
| `sm_srvcln_demos_archives` | Server Clean Up | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 源码描述仅为 desc，未说明具体行为 | `servercleanup.sp:150` |
| `sm_srvcln_demos_path` | Server Clean Up | `` | 字符串或表达式 | 无上下界 | 源码描述仅为 desc，未说明具体行为 | `servercleanup.sp:144` |
| `sm_srvcln_demos_time` | Server Clean Up | `168` | 整数，源码写作浮点 | ≥ -1.0 | 源码描述仅为 desc，未说明具体行为 | `servercleanup.sp:165` |
| `sm_srvcln_enable` | Server Clean Up | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 源码描述仅为 desc，未说明具体行为 | `servercleanup.sp:120` |
| `sm_srvcln_logging_mode` | Server Clean Up | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 源码描述仅为 desc，未说明具体行为 | `servercleanup.sp:123` |
| `sm_srvcln_logs` | Server Clean Up | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 源码描述仅为 desc，未说明具体行为 | `servercleanup.sp:126` |
| `sm_srvcln_logs_time` | Server Clean Up | `168` | 整数，源码写作浮点 | ≥ -1.0 | 源码描述仅为 desc，未说明具体行为 | `servercleanup.sp:156` |
| `sm_srvcln_replays` | Server Clean Up | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 源码描述仅为 desc，未说明具体行为 | `servercleanup.sp:135` |
| `sm_srvcln_replays_archives` | Server Clean Up | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 源码描述仅为 desc，未说明具体行为 | `servercleanup.sp:138` |
| `sm_srvcln_replays_time` | Server Clean Up | `168` | 整数，源码写作浮点 | ≥ -1.0 | 源码描述仅为 desc，未说明具体行为 | `servercleanup.sp:168` |
| `sm_srvcln_roundbackups` | Server Clean Up | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 源码描述仅为 desc，未说明具体行为 | `servercleanup.sp:141` |
| `sm_srvcln_roundbackups_time` | Server Clean Up | `168` | 整数，源码写作浮点 | ≥ -1.0 | 源码描述仅为 desc，未说明具体行为 | `servercleanup.sp:171` |
| `sm_srvcln_smlogs` | Server Clean Up | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 源码描述仅为 desc，未说明具体行为 | `servercleanup.sp:129` |
| `sm_srvcln_smlogs_time` | Server Clean Up | `168` | 整数，源码写作浮点 | ≥ -1.0 | 源码描述仅为 desc，未说明具体行为 | `servercleanup.sp:162` |
| `sm_srvcln_smlogs_type` | Server Clean Up | `0` | 整数，源码写作浮点 | 0.0 ~ 2.0 | 源码描述仅为 desc，未说明具体行为 | `servercleanup.sp:153` |
| `sm_srvcln_sprays` | Server Clean Up | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 源码描述仅为 desc，未说明具体行为 | `servercleanup.sp:147` |
| `sm_srvcln_sprays_time` | Server Clean Up | `168` | 整数，源码写作浮点 | ≥ -1.0 | 源码描述仅为 desc，未说明具体行为 | `servercleanup.sp:159` |
| `sm_srvcln_version` | Server Clean Up | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 源码描述仅为 desc，未说明具体行为 | `servercleanup.sp:117` |
| `sm_stats_autoprint_coop_round` | Player Statistics | `1289` | 整数，源码写作浮点 | ≥ 0.0 | 合作(campaign)回合自动播报位标志；源码其实已给描述（`optional/l4d2_playstats.sp:533-538` 第 3 参数）：「Flags for automatic print [campaign round] (show 1,4:MVP-chat, 4,8,16:MVP-console, 32,64:FF, 128,256:special, 512,1024,2048,4096:accuracy).」，:535 的内联注释说明默认值 `1289 = 1(mvpchat) + 8(mvpcon-all) + 256(special all) + 1024(acc all)`，即按位控制合作模式下回合结束时自动打印哪些统计表（置信度：高） | `l4d2_playstats.sp:533` |
| `sm_stats_autoprint_vs_round` | Player Statistics | `8325` | 整数，源码写作浮点 | ≥ 0.0 | 对抗回合自动播报位标志；源码其实已给描述（`optional/l4d2_playstats.sp:526-531` 第 3 参数）：「Flags for automatic print [versus round] (show 1,4:MVP-chat, 4,8,16:MVP-console, 32,64:FF, 128,256:special, 512,1024,2048,4096:accuracy).」，:528 的内联注释进一步说明默认值 `8325 = 1(mvpchat) + 4(mvpcon-round) + 128(special round) + 8192(funfact round)`，即按位控制对抗模式下回合结束时自动打印哪些统计表（置信度：高） | `l4d2_playstats.sp:526` |
| `sm_stats_debug` | Player Statistics | `0` | 整数，源码写作浮点 | ≥ 0.0 | 调试模式，最小 0 | `l4d2_playstats.sp:512` |
| `sm_stats_percentdecimal` | Player Statistics | `0` | 整数，源码写作浮点 | ≥ 0.0 | 是否在多数 MVP 百分比的控制台表格中显示一位小数 | `l4d2_playstats.sp:547` |
| `sm_stats_resetnextmap` | Player Statistics | `0` | 整数，源码写作浮点 | ≥ 0.0 | 是否忽略第一回合，配合 confogl 与比赛投票使用，换图后自动取消 | `l4d2_playstats.sp:561` |
| `sm_stats_showbots` | Player Statistics | `1` | 整数，源码写作浮点 | ≥ 0.0 | 是否在所有表格中显示 Bot，0 表示仅在 MVP 与友伤表中显示 | `l4d2_playstats.sp:540` |
| `sm_stats_writestats` | Player Statistics | `0` | 整数，源码写作浮点 | ≥ 0.0 | 是否把统计数据写入 logs/ 目录：1 写 csv，2 写 csv 与格式化表格，仅对战模式 | `l4d2_playstats.sp:554` |
| `SM_survivalbonusratio` | L4D2 Scoremod | `0.0` | 整数，源码写作浮点 | 无上下界 | 用于按地图距离计算固定生存奖励的比例 | `l4d2_scoremod.sp:86` |
| `sm_survivor_mvp_brevity` | Survivor MVP notification | `0` | 整数 | 无上下界 | MVP 报告精简 flag：1 隐藏特感，2 隐藏普感，4 隐藏友伤，8 隐藏排名，32 隐藏百分比，64 隐藏绝对值 | `survivor_mvp.sp:268` |
| `sm_survivor_mvp_brevity_latest` | Player Statistics | `4` | 整数，源码写作浮点 | ≥ 0.0 | MVP 聊天报告精简 flag：1 隐藏特感，2 隐藏普感，4 隐藏友伤，8 隐藏排名，32 隐藏百分比，64 隐藏绝对值 | `l4d2_playstats.sp:519` |
| `sm_survivor_mvp_counttank` | Survivor MVP notification | `0` | 整数 | 无上下界 | 为 1 时对坦克的伤害计入 MVP 评选 | `survivor_mvp.sp:265` |
| `sm_survivor_mvp_countwitch` | Survivor MVP notification | `0` | 整数 | 无上下界 | 为 1 时对 Witch 的伤害计入 MVP 评选 | `survivor_mvp.sp:266` |
| `sm_survivor_mvp_enabled` | Survivor MVP notification | `1` | 整数 | 无上下界 | 是否在回合结束时显示 MVP | `survivor_mvp.sp:264` |
| `sm_survivor_mvp_showff` | Survivor MVP notification | `1` | 整数 | 无上下界 | 是否统计友军伤害 | `survivor_mvp.sp:267` |
| `sm_tank_can_spawn` | Tank and Witch ifier! | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否允许坦克生成 | `witch_and_tankifier.sp:71` |
| `sm_tank_witch_debug` | Tank and Witch ifier! | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否开启调试模式 | `witch_and_tankifier.sp:70` |
| `SM_tempmulti_incap_0` | L4D2 Scoremod | `0.30625` | 浮点 | 0.0 ~ 1.0 | 未有倒地记录的生还者其临时生命的重要程度 | `l4d2_scoremod.sp:89` |
| `SM_tempmulti_incap_1` | L4D2 Scoremod | `0.17500` | 浮点 | 0.0 ~ 1.0 | 倒地一次的生还者其临时生命的重要程度 | `l4d2_scoremod.sp:92` |
| `SM_tempmulti_incap_2` | L4D2 Scoremod | `0.10000` | 浮点 | 0.0 ~ 1.0 | 倒地两次即黑白的生还者其临时生命的重要程度 | `l4d2_scoremod.sp:95` |
| `sm_uncinfblock_enabled` | Uncommon Infected Blocker | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用非常见感染者屏蔽插件 | `l4d2_uncommon_blocker.sp:110` |
| `sm_uncinfblock_flags` | Uncommon Infected Blocker | `55` | 整数，源码写作浮点 | 1.0 ~ 127.0 | 屏蔽哪些非常见感染者，位域 1~127：1 ceda，2 泥人，4 工兵，8 fallen，16 防暴警，32 小丑，其余被截断 | `l4d2_uncommon_blocker.sp:114` |
| `sm_unpausedelay` | Pause plugin | `3` | 整数，源码写作浮点 | ≥ 0.0 | 取消暂停生效前的延迟秒数 | `pause.sp:113` |
| `sm_unsilentjockey_interval` | Unsilent Jockey | `2.0` | 整数，源码写作浮点 | 无上下界 | 强制播放 Jockey 音效的间隔秒数 | `l4d2_unsilent_jockey.sp:77` |
| `sm_updater` | Updater | `2` | 整数，源码写作浮点 | 1.0 ~ 3.0 | 更新功能模式：1 仅通知，2 下载，3 同时包含源码 | `updater.sp:108` |
| `sm_updater_version` | Updater | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | `updater.sp:105` |
| `sm_veterans_bantime` | VeteransOnly | `10` | 整数，源码写作浮点 | 0.0 ~ MAX_FLOAT | 是否改为封禁而非踢出，以及封禁分钟数，0 表示不封禁 | `veterans.sp:172` |
| `sm_veterans_cachetime` | VeteransOnly | `14400` | 整数，源码写作浮点 | 0.0 ~ MAX_FLOAT | 对同一查询不重复发送请求的缓存秒数 | `veterans.sp:197` |
| `sm_veterans_enable` | VeteransOnly | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用 VeteransOnly 插件 | `veterans.sp:121` |
| `sm_veterans_excludegroupmember` | VeteransOnly | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否把 Steam 组成员排除在惩罚之外 | `veterans.sp:151` |
| `sm_veterans_excludegroupmembercount` | VeteransOnly | `2` | 整数，源码写作浮点 | 0.0 ~ MAX_FLOAT | 应从前多少个 sv_steamgroup 组中排除玩家，0 检查所有配置的组 | `veterans.sp:157` |
| `sm_veterans_excludegroupmemberplay` | VeteransOnly | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否让 Steam 组成员免除时长门槛；据 `extend/veterans.sp:133-138` 的英文描述原文「Should we let exclude group member but not rechach mititaion to play」（语法不通），配合 :242 `if(!HasEnoughPlaytime(player[Player].servertime) && player[Player].isGroupMember && !GetConVarBool(cvar_excludeGroupMemberPlay))` 命中时提示 `Veterans_PlayerDurationDetectionNotMeet` 并 `return Plugin_Stop` 拦截入队，以及 :404-415 两个分支的提示文本（`Veterans_PlayerDurationDetectionPlayerGame`=“…may play normally” 与 `Veterans_PlayerTimeDetectionPlayerGame`=“…may only spectate”，见 `translations/veterans.phrases.txt:46-52`），推测为：1=组成员即使时长不达标也可正常游玩（豁免时长限制），0=时长不达标的组成员只能旁观并被拒绝加入队伍（置信度：中） | `veterans.sp:133` |
| `sm_veterans_excludeprivileged` | VeteransOnly | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否把有权限的玩家排除在惩罚之外 | `veterans.sp:145` |
| `sm_veterans_excludereservedslots` | VeteransOnly | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否把拥有预留槽位的玩家排除在惩罚之外 | `veterans.sp:139` |
| `sm_veterans_gameid` | VeteransOnly | `550` | 整数，源码写作浮点 | 0.0 ~ MAX_FLOAT | 要检查玩家游戏时长的游戏的 Steam 商店 id | `veterans.sp:127` |
| `sm_veterans_minServertotal` | VeteransOnly | `0` | 整数，源码写作浮点 | 0.0 ~ MAX_FLOAT | 玩家需要的最低本服总游戏时长，单位分钟 | `veterans.sp:184` |
| `sm_veterans_mintotal` | VeteransOnly | `0` | 整数，源码写作浮点 | 0.0 ~ MAX_FLOAT | 玩家需要的最低总游戏时长，单位分钟 | `veterans.sp:178` |
| `sm_veterans_mintotalminuslastweeks` | VeteransOnly | `0` | 整数，源码写作浮点 | 0.0 ~ MAX_FLOAT | 玩家需要的最低总游戏时长（不含最近两周），单位分钟 | `veterans.sp:190` |
| `sm_veterans_version` | VeteransOnly | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | `veterans.sp:111` |
| `sm_weapon_hide_attributes` | L4D2 Weapon Attributes | `2` | 整数，源码写作浮点 | 0.0 ~ 2.0 | 自定义 sm_weapon_attributes 命令：0 禁用命令，1 仅管理员可看武器属性，2 其余被截断 | `l4d2_weapon_attributes.sp:233` |
| `sm_witch_avoid_tank_spawn` | Tank and Witch ifier! | `20` | 整数，源码写作浮点 | 0.0 ~ 100.0 | Witch 应避开坦克刷新点的最小流程距离，按给定值的一半在坦克刷新点两侧计算 | `witch_and_tankifier.sp:73` |
| `sm_witch_bonus_always` | Simple Witch Kill Bonus | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 非生还者击杀 Witch 时是否也给予分数 | `simple_witch_bonus.sp:56` |
| `sm_witch_bonus_print` | Simple Witch Kill Bonus | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 奖励 Witch 击杀分数时是否打印提示 | `simple_witch_bonus.sp:55` |
| `sm_witch_can_spawn` | Tank and Witch ifier! | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否允许 Witch 生成 | `witch_and_tankifier.sp:72` |
| `sm_zd_limit_10m` | Anne Global Chat | `20` | 整数，源码写作浮点 | ≥ 0.0 | 积分 1000 万以上玩家每日找队友次数上限 | `global_chat.sp:185` |
| `sm_zd_limit_1m` | Anne Global Chat | `5` | 整数，源码写作浮点 | ≥ 0.0 | 积分 100 万以上玩家每日找队友次数上限 | `global_chat.sp:183` |
| `sm_zd_limit_20m` | Anne Global Chat | `30` | 整数，源码写作浮点 | ≥ 0.0 | 积分 2000 万以上玩家每日找队友次数上限 | `global_chat.sp:186` |
| `sm_zd_limit_5m` | Anne Global Chat | `10` | 整数，源码写作浮点 | ≥ 0.0 | 积分 500 万以上玩家每日找队友次数上限 | `global_chat.sp:184` |
| `sm_zd_limit_default` | Anne Global Chat | `3` | 整数，源码写作浮点 | ≥ 0.0 | 普通玩家每日找队友次数上限 | `global_chat.sp:182` |
| `sm_zd_limit_kick_admin` | Anne Global Chat | `50` | 整数，源码写作浮点 | ≥ 0.0 | 有 kick 权限但无 z 权限的管理员每日找队友次数上限 | `global_chat.sp:187` |
| `sm_zd_show_global` | Anne Global Chat | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 本服旁观玩家能否看到找队友全服提示 | `global_chat.sp:180` |
| `smoker_anim_fix_enabled` | Smoker Animation Fix (windowed & safe reset, end-on-grab option) | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用 Smoker 动画修复 | `smoker_anim_fix.sp:83` |
| `smoker_anim_fix_end_on_grab` | Smoker Animation Fix (windowed & safe reset, end-on-grab option) | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 舌头抓住目标时是否立即结束保持窗口 | `smoker_anim_fix.sp:92` |
| `smoker_anim_fix_hold_window` | Smoker Animation Fix (windowed & safe reset, end-on-grab option) | `0.6` | 浮点 | 0.0 ~ 2.0 | 技能开始后强制舌头动画的秒数，范围 0~2 | `smoker_anim_fix.sp:86` |
| `smoker_anim_fix_safety_timer` | Smoker Animation Fix (windowed & safe reset, end-on-grab option) | `2.0` | 浮点 | 0.5 ~ 5.0 | 技能开始后重置舌头 flag 的安全计时，范围 0.5~5 | `smoker_anim_fix.sp:89` |
| `sms_cvar_change_notify_block` | ShieldTips.sp（myinfo 缺 name，用文件名代替） | `1` | 整数 | 无上下界 | 是否屏蔽游戏自带的 ConVar 更改提示 | `ShieldTips.sp:36` |
| `sms_game_disconnect_notify_block` | ShieldTips.sp（myinfo 缺 name，用文件名代替） | `1` | 整数 | 无上下界 | 是否屏蔽游戏自带的玩家离开提示 | `ShieldTips.sp:38` |
| `sms_game_idle_notify_block` | ShieldTips.sp（myinfo 缺 name，用文件名代替） | `1` | 整数 | 无上下界 | 是否屏蔽游戏自带的玩家闲置提示 | `ShieldTips.sp:35` |
| `sms_sourcemod_sm_notify_admin` | ShieldTips.sp（myinfo 缺 name，用文件名代替） | `0` | 整数 | 无上下界 | 是否屏蔽 SourceMod 自带的 SM 提示：1 只向管理员显示，0 对所有人屏蔽 | `ShieldTips.sp:37` |
| `sn_hostname_format` | Anne ServerName | `{hostname}{gamemode}` | 字符串或表达式 | 无上下界 | hostname 格式模板（源码未给描述） | `server_name.sp:79` |
| `sn_hostname_format1` | Anne ServerName | `{Confogl}{AIDifficulty}{Full}{MOD}{AnneHappy}` | 字符串或表达式 | 无上下界 | 备用 hostname 格式模板（源码未给描述） | `server_name.sp:82` |
| `sn_main_name` | Anne ServerName | `电信服` | 字符串或表达式 | 无上下界 | 服务器主名称（源码未给描述） | `server_name.sp:76` |
| `sound_flags` | Sound Manipulation: REWORK | `0` | 整数 | 无上下界 | 屏蔽声音的位掩码：0 无，1 心跳，2 重型可击打物音效，4 倒地受伤音效，其余被截断 | `l4d2_sound_manipulation.sp:25` |
| `sourcecomms_version` | SourceBans++: SourceComms | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | `sbpp_comms.sp:196` |
| `specrates_force_spec` | Lightweight Spectating (merged+128+force-spec) | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 开启后旁观一律 30 tick，管理员与解说旁观为 60 tick，对局内不受影响 | `specrates.sp:112` |
| `specrates_fulltickspecnum` | Lightweight Spectating (merged+128+force-spec) | `4` | 整数 | 无上下界 | 旁观人数超过该值后，除管理员与解说外其余人限 30 tick，无视积分 | `specrates.sp:108` |
| `sr_medkit_debug` | l4d2_med_dynamic.sp（myinfo 缺 name，用文件名代替） | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否输出调试日志 | `l4d2_med_dynamic.sp:77` |
| `sr_medkit_enable` | l4d2_med_dynamic.sp（myinfo 缺 name，用文件名代替） | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否启用安全室医疗包控制：1 开，0 关 | `l4d2_med_dynamic.sp:75` |
| `sr_medkit_scan_delay` | l4d2_med_dynamic.sp（myinfo 缺 name，用文件名代替） | `1.5` | 浮点 | 0.0 ~ 10.0 | round_start 后剥离安全室医疗包的延迟秒数，范围 0~10 | `l4d2_med_dynamic.sp:76` |
| `ssb_block_alarms` | L4D2 Various Sounds Blocker | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否屏蔽警报音效 | `l4d2_sounds_blocker.sp:79` |
| `ssb_block_ambient_explosions` | L4D2 Various Sounds Blocker | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否屏蔽环境爆炸音效 | `l4d2_sounds_blocker.sp:83` |
| `ssb_block_car_alarms` | L4D2 Various Sounds Blocker | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否屏蔽汽车警报音效 | `l4d2_sounds_blocker.sp:78` |
| `ssb_block_coaster` | L4D2 Various Sounds Blocker | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否屏蔽过山车音效 | `l4d2_sounds_blocker.sp:77` |
| `ssb_block_fireworks` | L4D2 Various Sounds Blocker | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否屏蔽烟花音效 | `l4d2_sounds_blocker.sp:76` |
| `ssb_block_generators` | L4D2 Various Sounds Blocker | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否屏蔽发电机音效 | `l4d2_sounds_blocker.sp:82` |
| `ssb_block_horde` | L4D2 Various Sounds Blocker | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否屏蔽尸潮音效 | `l4d2_sounds_blocker.sp:80` |
| `ssb_block_laughs` | L4D2 Various Sounds Blocker | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否屏蔽笑声 | `l4d2_sounds_blocker.sp:85` |
| `ssb_block_lifts` | L4D2 Various Sounds Blocker | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否屏蔽电梯音效 | `l4d2_sounds_blocker.sp:84` |
| `ssb_block_misc_vehicles` | L4D2 Various Sounds Blocker | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否屏蔽杂物车辆音效，如教区第四关拖拉机、死亡机场终局飞机等 | `l4d2_sounds_blocker.sp:81` |
| `starting_item_flags` | Starting Items | `0` | 整数，源码写作浮点 | ≤ 127.0 | 离开安全区时给予的物品 flag，0 禁用，1 急救包，2 除颤器，4 止痛药，8 肾上腺素，16 管式炸弹，32 燃烧瓶，其余被截断 | `starting_items.sp:47` |
| `stop_trolls_flags` | StopTrolls | `862` | 整数 | 无上下界 | 谁可以在爬梯时推动巨魔：0 关闭，2 Smoker，4 Boomer，8 Hunter，16 Spitter，64 Charger，256 Tank，可相加 | `l4d2_ladderblock.sp:74` |
| `stop_trolls_immune` | StopTrolls | `256` | 整数 | 无上下界 | 什么职业免疫：0 关闭，2 Smoker，4 Boomer，8 Hunter，16 Spitter，32 Jockey，64 Charger，256 Tank，512 Survivor，可相加 | `l4d2_ladderblock.sp:75` |
| `stop_trolls_version` | StopTrolls | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | `l4d2_ladderblock.sp:72` |
| `survivor_afk_fix_ver` | [L4D2]Survivor_AFK_Fix | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | `survivor_afk_fix.sp:66` |
| `survivor_legs_version` | [L4D2]Survivor_Legs_Restore | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | `Survivor_Legs.sp:87` |
| `sv_vote_returnlobby_allowed` | L4D2 Vote Return Lobby patch | `0` | 整数 | 无上下界 | 玩家能否投票返回大厅 | `l4d2_vote_returnlobby_patch.sp:23` |
| `svctyfix_message_enable` | sv_consistency fixes | `1.0` | 开关 0 或 1 | 0.0 ~ 1.0 | 玩家加入时是否在控制台打印提示信息 | `sv_consistency_fix.sp:28` |
| `svctyfix_welcome_message` | sv_consistency fixes | `a SoundM Protected Server` | 字符串或表达式 | 无上下界 | 在控制台向玩家显示的消息 | `sv_consistency_fix.sp:35` |
| `tank_damage_enable` | Tank Damage Announce 2.0 | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否允许在 Tank 死亡后输出生还者对 Tank 的伤害统计 | `l4d_tank_damage_announce.sp:66` |
| `tank_damage_enable_healthset` | Tank Damage Announce 2.0 | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否把坦克生命值设置为 z_tank_health 的数值 | `l4d_tank_damage_announce.sp:71` |
| `tank_damage_failed_announce` | Tank Damage Announce 2.0 | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 生还者团灭而场上仍有 Tank 时是否显示伤害统计 | `l4d_tank_damage_announce.sp:69` |
| `tank_damage_force_kill_announce` | Tank Damage Announce 2.0 | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | Tank 被强制处死或自杀时是否输出生还者对 Tank 的伤害统计 | `l4d_tank_damage_announce.sp:67` |
| `tank_damage_print_livetime` | Tank Damage Announce 2.0 | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否显示 Tank 的存活时间 | `l4d_tank_damage_announce.sp:68` |
| `tank_damage_print_zero` | Tank Damage Announce 2.0 | `1` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否显示对 Tank 零伤害的玩家 | `l4d_tank_damage_announce.sp:70` |
| `tank_extinguish_time` | SI Fire Immunity | `1.0` | 整数，源码写作浮点 | 0.0 ~ 999.0 | 坦克玩家在多少秒后熄灭，仅当 tank_fire_immunity 为 3 时生效 | `si_fire_immunity.sp:70` |
| `tank_fire_immunity` | SI Fire Immunity | `2` | 整数，源码写作浮点 | 0.0 ~ 3.0 | 坦克的火焰免疫类型：0 无，3 随时间自动熄灭，2 免疫燃烧，1 完全免疫 | `si_fire_immunity.sp:56` |
| `tank_overhand_only` | Tank Attack Control | `0` | 整数 | 无上下界 | 是否强制坦克只投掷上手石头 | `l4d2_tank_attack_control.sp:52` |
| `tank_punch_getup_scale` | [L4D & 2] Static Punch Get-up | `0.5` | 浮点 | 0.01 ~ 0.99 | Tank 拳击落地起身动画长度的缩放比例，范围 0.01~0.99 | `l4d_static_punch_getup.sp:52` |
| `tankcontrol_debug` | L4D2 Tank Control | `0` | 整数 | 无上下界 | 是否输出调试信息到控制台 | `l4d_tank_control_eq.sp:91` |
| `tankcontrol_force_window` | L4D2 Tank Control | `0.0` | 整数，源码写作浮点 | 无上下界 | 给原本要成为坦克或曾是坦克后掉线的玩家在该秒数后重新给予坦克 | `l4d_tank_control_eq.sp:90` |
| `tankcontrol_print_all` | L4D2 Tank Control | `0` | 整数 | 无上下界 | 谁能看到谁将成为坦克：0 特感，1 所有人 | `l4d_tank_control_eq.sp:89` |
| `tankgank_killoncrash` | AI Tank Gank | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 为 0 时若控制坦克的玩家崩溃则坦克不会被处死 | `aitankgank.sp:23` |
| `teamflip_delay` | Teamflip | `-1` | 整数，源码写作浮点 | ≥ -1.0 | 两次允许换队之间的延迟秒数，-1 表示无延迟 | `teamflip.sp:49` |
| `thirdpersonshoulder_detect_allow_versus` | ThirdPersonShoulder_Detect | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 0 在对战类配置中强制第一人称，1 检测客户端真实的第三人称状态 | `ThirdPersonShoulder_Detect.sp:38` |
| `ThirdPersonShoulder_Detect_Version` | ThirdPersonShoulder_Detect | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | `ThirdPersonShoulder_Detect.sp:37` |
| `tick_door_speed` | Tickrate Fixes | `1.3` | 浮点 | 无上下界 | 设置地图上所有 prop_door 实体的速度，1.05 表示 105% 速度 | `TickrateFixes.sp:71` |
| `tongue_bend_exception_flag` | [L4D & 2] Tongue Bend Fix | `1` | 整数，源码写作浮点 | ≥ 0.0 | 允许舌头抓取特定类型实体的 flag：1 门，2 可搬运物，3 全部，0 关闭 | `l4d_tongue_bend_fix.sp:42` |
| `tongue_damage_continuity` | L4D2 smoker drag damage interval | `0` | 开关 0 或 1 | 0.0 ~ 1.0 | 是否在拖拽与勒喉切换之间保留伤害计时：0 原生行为，1 保留剩余时间 | `l4d2_smoker_drag_damage_interval.sp:113` |
| `tongue_drag_damage_interval` | L4D2 Smoker Drag Damage Interval | `value` | 字符串或表达式 | 无上下界 | 拖拽造成伤害的频率 | `l4d2_smoker_drag_damage_interval_zone.sp:45` |
| `tongue_drag_damage_interval` | L4D2 smoker drag damage interval | `sCvarVal` | 字符串或表达式 | 0.01 ~ 15.0 | 拖拽造成伤害的频率，允许值 0.01~15.0 | `l4d2_smoker_drag_damage_interval.sp:110` |
| `tongue_drag_first_damage` | L4D2 Smoker Drag Damage Interval | `3.0` | 整数，源码写作浮点 | 无上下界 | 舌头首次命中时施加的伤害，仅在启用 first_damage_interval 时生效 | `l4d2_smoker_drag_damage_interval_zone.sp:47` |
| `tongue_drag_first_damage` | L4D2 smoker drag damage interval | `-1.0` | 整数，源码写作浮点 | ≤ 100.0 | 舌头首次命中时施加的伤害，0.0 关闭，最大 100.0 | `l4d2_smoker_drag_damage_interval.sp:112` |
| `tongue_drag_first_damage_interval` | L4D2 Smoker Drag Damage Interval | `-1.0` | 整数，源码写作浮点 | 无上下界 | 首次伤害在多少秒后施加，0.0 表示关闭 | `l4d2_smoker_drag_damage_interval_zone.sp:46` |
| `tongue_drag_first_damage_interval` | L4D2 smoker drag damage interval | `-1.0` | 整数，源码写作浮点 | ≤ 15.0 | 首次伤害在多少秒后施加，0.0 关闭，最大 15.0 | `l4d2_smoker_drag_damage_interval.sp:111` |
| `tongue_fly_through_teammate` | [L4D & 2] Tongue Block Fix | `1` | 整数，源码写作浮点 | 0.0 ~ 3.0 | 舌头射出后能否穿过队友：1 普通特感，2 Tank，3 全部，0 关闭 | `l4d_tongue_block_fix.sp:110` |
| `tongue_tip_through_teammate` | [L4D & 2] Tongue Block Fix | `0` | 整数，源码写作浮点 | 0.0 ~ 3.0 | Smoker 能否透过队友吐舌：1 透过普通特感，2 透过 Tank，3 全部，0 关闭 | `l4d_tongue_block_fix.sp:101` |
| `versus_coop_mode_version` | versus_coop_mode.sp（myinfo 缺 name，用文件名代替） | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | `versus_coop_mode.sp:56` |
| `votecfgfile` | Vote for run command or cfg file | `VOTE_DEFAULT_CONFIG` | 字符串或表达式 | 无上下界 | 投票文件的位置，位于 sourcemod/ 文件夹下 | `vote.sp:77` |
| `vs_tank_pound_damage` | L4D2 Tank Damage Cvars | `24.0` | 整数，源码写作浮点 | 无上下界 | 对战模式坦克近战攻击对倒地生还者造成的伤害，0 或负值关闭 | `l4d2_tank_damage_cvars.sp:36` |
| `vs_tank_rock_damage` | L4D2 Tank Damage Cvars | `24.0` | 整数，源码写作浮点 | 无上下界 | 对战模式坦克石头造成的伤害，0 或负值关闭 | `l4d2_tank_damage_cvars.sp:37` |
| `witch_double_start_fix` | [L4D2]Witch_Double_Start_Fix | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | `Witch_Double_Startle_Fix.sp:50` |
| `witch_prevent_target_loss` | [L4D1/2]witch_prevent_target_loss | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | `witch_prevent_target_loss.sp:61` |
| `witch_show_true_damage` | Witch Damage Announce | `0` | 整数 | 无上下界 | 是否显示伤害输出而非 Witch 实际受到的伤害，0 显示实际生命伤害 | `witch_announce.sp:93` |
| `witch_target_patch_version` | [L4D1/2]Witch_Target_Patch | `PLUGIN_VERSION` | 字符串或表达式 | 无上下界 | 插件版本号（非行为配置） | `Witch_Target_patch.sp:70` |
| `z_charge_pinned_collision` | [L4D2] Charger Target Fix | `3` | 整数，源码写作浮点 | 0.0 ~ 3.0 | 被 Charger 控制的生还者能否与特感碰撞：1 压制期间，2 起身期间，3 两者，0 完全不碰撞 | `l4d2_charge_target_fix.sp:88` |
| `z_hunter_max_pounce_bonus_damage` | Skill Detection (skeets, crowns, levels) | `49` | 整数，源码写作浮点 | ≥ 0.0 | 源码说明：本服务器没有该 cvar，由 l4d2_skill_detect 添加 | `l4d2_skill_detect.sp:535` |
| `z_leap_damage_interrupt` | L4D2 Jockey Skeet | `195.0` | 整数，源码写作浮点 | 10.0 ~ 325.0 | 受到该伤害量会打断跳跃尝试，范围 10~325 | `l4d2_jockey_skeet.sp:44` |
| `z_leap_interval_post_ledge_hang` | L4D2 Jockey Ledge Hang Recharge | `10` | 整数 | 无上下界 | Jockey 挂边后再次跳跃前的等待秒数 | `l4d_jockey_ledgehang.sp:25` |
| `z_pounce_damage_range_max` | Skill Detection (skeets, crowns, levels) | `1000.0` | 整数，源码写作浮点 | ≥ 0.0 | 源码说明：本服务器没有该 cvar，由 l4d2_skill_detect 添加 | `l4d2_skill_detect.sp:531` |
| `z_pounce_damage_range_min` | Skill Detection (skeets, crowns, levels) | `300.0` | 整数，源码写作浮点 | ≥ 0.0 | 源码说明：本服务器没有该 cvar，由 l4d2_skill_detect 添加 | `l4d2_skill_detect.sp:533` |
| `ZonemodWeapon` | text.sp（myinfo 缺 name，用文件名代替） | `0` | 整数 | 无上下界 | 整服配置档位开关；据 `optional/AnneHappy/text.sp:41` 的 `CreateConVar("ZonemodWeapon", "0", "", 0, false, 0.0, false, 0.0)`（描述为空字符串），:150-177 `CvarWeapon` 按值执行 `ServerCommand("exec vote/weapon/...cfg")`：1→`zonemod.cfg`、0→`AnneHappy.cfg`、2→特感上限≥10 或配置名含 Alone/1vHunters 时执行 `AnneHappyPlus.cfg`，否则回退成 0（:158-176），:226 又把它映射为显示名 `Weapon>1?"Anne+":(Weapon>0?"Zone":"Anne")`，推测为：切换整服武器/玩法配置风格，0=AnneHappy、1=Zonemod、2=AnneHappyPlus（置信度：高） | `text.sp:41` |

### 全部指令（按名称排序，共 485 条）

| 指令 | 所属插件 | 语法/参数 | 权限 | 功能 | 来源 |
|---|---|---|---|---|---|
| `anne_cvar_shield_capture` | anne_cvar_shield.sp（myinfo 缺 name，用文件名代替） | 无参数 | 服务器控制台命令，玩家无法使用 | 把当前 Anne 特感上限 cvar 捕获为保护目标，服务器控制台命令 | `anne_cvar_shield.sp:137` |
| `boomer_horde_amount` | Boomer Horde Equalizer (Refactored) | <被喷人数> <尸潮数量> | 服务器控制台命令，玩家无法使用 | 设置 Boomer 尸潮数量，服务器控制台命令 | `boomer_horde_equalizer_refactored.sp:92` |
| `codepatch_list` | Code patcher | 无参数 | 服务器控制台命令，玩家无法使用 | 列出已应用的代码补丁 | `code_patcher.sp:54` |
| `codepatch_patch` | Code patcher | <补丁名> <参数> | 服务器控制台命令，玩家无法使用 | 应用指定代码补丁，服务器控制台命令 | `code_patcher.sp:55` |
| `codepatch_unpatch` | Code patcher | <补丁名> <参数> | 服务器控制台命令，玩家无法使用 | 撤销指定代码补丁，服务器控制台命令 | `code_patcher.sp:56` |
| `confogl_addcvar` | Confogl's Competitive Mod | <cvar> <newValue> | 服务器控制台命令，玩家无法使用 | 把某个 ConVar 加入由 Confogl 设置与强制的列表，服务器控制台命令 | `CvarSettings.sp:43` |
| `confogl_clientsettings` | Confogl's Competitive Mod | 无参数 | 无（任意玩家可用） | 列出 confogl 强制跟踪的客户端 ConVar | `ClientSettings.sp:39` |
| `confogl_cvardiff` | Confogl's Competitive Mod | [0 或 1] | 无（任意玩家可用） | 列出被改动过初始值的 ConVar | `CvarSettings.sp:41` |
| `confogl_cvarsettings` | Confogl's Competitive Mod | [0 或 1] | 无（任意玩家可用） | 列出 confogl 正在强制的所有 ConVar | `CvarSettings.sp:40` |
| `confogl_erdata_reload` | Confogl's Competitive Mod | 无参数 | 需要 ADMFLAG_CONFIG（h，服务器配置） | 重新加载 EntityRemoveData 配置 | `EntityRemover.sp:53` |
| `confogl_midata_save` | Confogl's Competitive Mod | 无参数 | 需要 ADMFLAG_CONFIG（h，服务器配置） | 把当前地图的 EntityRemoveData 配置保存下来 | `MapInfo.sp:43` |
| `confogl_midata_save` | L4D2Lib | 无参数 | 需要 ADMFLAG_CONFIG（h，服务器配置） | 保存当前地图的物资点配置，L4D2Lib 版本 | `mapinfo.sp:54` |
| `confogl_resetclientcvars` | Confogl's Competitive Mod | 无参数 | 服务器控制台命令，玩家无法使用 | 清除所有已跟踪的客户端 ConVar，比赛中使用会被拒绝，服务器控制台命令 | `ClientSettings.sp:43` |
| `confogl_resetcvars` | Confogl's Competitive Mod | 无参数 | 服务器控制台命令，玩家无法使用 | 重置被强制的 ConVar，比赛中不可用，服务器控制台命令 | `CvarSettings.sp:45` |
| `confogl_save_location` | Confogl's Competitive Mod | <位置名> | 需要 ADMFLAG_CONFIG（h，服务器配置） | 把当前所在坐标保存到地图配置 | `MapInfo.sp:44` |
| `confogl_save_location` | L4D2Lib | <点位类型> | 需要 ADMFLAG_CONFIG（h，服务器配置） | 把当前位置坐标保存到物资点配置，L4D2Lib 版本 | `mapinfo.sp:55` |
| `confogl_setcvars` | Confogl's Competitive Mod | 无参数 | 服务器控制台命令，玩家无法使用 | 开始强制已添加的 ConVar，服务器控制台命令 | `CvarSettings.sp:44` |
| `confogl_startclientchecking` | Confogl's Competitive Mod | 无参数 | 服务器控制台命令，玩家无法使用 | 开始检查并强制已跟踪的客户端 ConVar，服务器控制台命令 | `ClientSettings.sp:44` |
| `confogl_trackclientcvar` | Confogl's Competitive Mod | <cvar> <hasMin> <min> [<hasMax> <max> [...]] | 服务器控制台命令，玩家无法使用 | 把某个客户端 ConVar 加入 confogl 的跟踪与强制列表，服务器控制台命令 | `ClientSettings.sp:42` |
| `debug` | [L4D & L4D2] Engine Fix | 无参数 | 无（任意玩家可用） | 显示加载提示并临时启用调试输出 | `fix_engine.sp:74` |
| `finale_tank_default` | Finale Even-Numbered Tank Blocker | <地图名> | 服务器控制台命令，玩家无法使用 | 设置该地图的终局坦克默认行为，服务器控制台命令 | `finale_tank_blocker.sp:27` |
| `finale_tank_default` | Hyper-V HUD Manager | <地图名> | 服务器控制台命令，玩家无法使用 | 设置终局坦克默认行为，服务器控制台命令 | `spechud.sp:295` |
| `l4d2_addweaponrule` | L4D2 Weapon Rules | <匹配> <替换> | 服务器控制台命令，玩家无法使用 | 添加武器替换规则，服务器控制台命令 | `l4d2_weaponrules.sp:50` |
| `l4d2_resetweaponrules` | L4D2 Weapon Rules | 无参数 | 服务器控制台命令，玩家无法使用 | 重置所有武器替换规则，服务器控制台命令 | `l4d2_weaponrules.sp:51` |
| `l4d_wlimits_add` | L4D Weapon Limits | <武器> <数量> <掩码> | 服务器控制台命令，玩家无法使用 | 添加武器数量限制，服务器控制台命令 | `l4d_weapon_limits.sp:81` |
| `l4d_wlimits_clear` | L4D Weapon Limits | 无参数 | 服务器控制台命令，玩家无法使用 | 清空所有武器限制，需先锁定，服务器控制台命令 | `l4d_weapon_limits.sp:83` |
| `l4d_wlimits_lock` | L4D Weapon Limits | 无参数 | 服务器控制台命令，玩家无法使用 | 锁定武器限制以加快查找速度，服务器控制台命令 | `l4d_weapon_limits.sp:82` |
| `ledge_block_square` | L4D2 Ledge Blocker | <参数> | 服务器控制台命令，玩家无法使用 | 设置边缘阻挡方块，服务器控制台命令 | `l4d2_ledgeblock.sp:34` |
| `ledge_remove_block_square` | L4D2 Ledge Blocker | <参数> | 服务器控制台命令，玩家无法使用 | 移除边缘阻挡方块，服务器控制台命令 | `l4d2_ledgeblock.sp:35` |
| `pred_unload_plugins` | Predictable Plugin Unloader | 无参数 | 服务器控制台命令，玩家无法使用 | 卸载插件，服务器控制台命令 | `predictable_unloader.sp:70` |
| `reset_static_maps` | Tank and Witch ifier! | 无参数 | 服务器控制台命令，玩家无法使用 | 重置静态地图列表，服务器控制台命令 | `witch_and_tankifier.sp:89` |
| `saferoom_frustration_tickdown` | Checkpoint Rage Control | <tick> | 服务器控制台命令，玩家无法使用 | 设置安全区挫败感倒计时，服务器控制台命令 | `checkpoint-rage-control.sp:74` |
| `say` | 1v1 SkeetStats | 聊天文本 | 无（任意玩家可用） | 接管 say，识别 !skeets 并输出 skeetstats | `1v1_skeetstats.sp:271` |
| `say` | L4D2 Scoremod | 聊天文本 | 无（任意玩家可用） | 接管 say，识别 !health | `l4d2_scoremod.sp:232` |
| `say` | SourceBans++: Main Plugin | 聊天文本 | 无（任意玩家可用） | 接管 say 命令，识别 !noreason 等内容以发起封禁 | `sbpp_main.sp:183` |
| `say` | Survivor MVP notification | 聊天文本 | 无（任意玩家可用） | 接管 say，识别 !mvp | `survivor_mvp.sp:292` |
| `say_team` | 1v1 SkeetStats | 聊天文本 | 无（任意玩家可用） | 接管 say_team，识别 !skeets 并输出 skeetstats | `1v1_skeetstats.sp:272` |
| `say_team` | L4D2 Scoremod | 聊天文本 | 无（任意玩家可用） | 接管 say_team，识别 !health | `l4d2_scoremod.sp:233` |
| `say_team` | SourceBans++: Main Plugin | 聊天文本 | 无（任意玩家可用） | 接管 say_team 命令，识别 !noreason 等内容以发起封禁 | `sbpp_main.sp:184` |
| `say_team` | Survivor MVP notification | 聊天文本 | 无（任意玩家可用） | 接管 say_team，识别 !mvp | `survivor_mvp.sp:293` |
| `sb_reload` | SourceBans++: Bans Checker | 无参数 | 需要 ADMFLAG_RCON（i，RCON 命令） | 重新加载 SourceBans 配置与封禁原因菜单项 | `sbpp_checker.sp:58` |
| `sb_reload` | SourceBans++: Main Plugin | 无参数 | 需要 ADMFLAG_RCON（i，RCON 命令） | 重新加载 SourceBans 配置与封禁原因菜单项 | `sbpp_main.sp:177` |
| `sc_fw_block` | SourceBans++: SourceComms | 无参数 | 服务器控制台命令，玩家无法使用 | 由 SourceBans 网站调用，用于封禁玩家通信，服务器控制台命令 | `sbpp_comms.sp:203` |
| `sc_fw_ungag` | SourceBans++: SourceComms | 无参数 | 服务器控制台命令，玩家无法使用 | 由 SourceBans 网站调用，用于解除禁言，服务器控制台命令 | `sbpp_comms.sp:204` |
| `sc_fw_unmute` | SourceBans++: SourceComms | 无参数 | 服务器控制台命令，玩家无法使用 | 由 SourceBans 网站调用，用于解除禁声，服务器控制台命令 | `sbpp_comms.sp:205` |
| `sm_3rd` | Anne Thirdperson Shoulder Fix | 无参数 | 无（任意玩家可用） | sm_tp 的别名 | `l4d2_anne_thirdperson_fix.sp:70` |
| `sm_3rdoff` | Anne Thirdperson Shoulder Fix | 无参数 | 无（任意玩家可用） | 关闭 Anne 第三人称肩部视角 | `l4d2_anne_thirdperson_fix.sp:72` |
| `sm_3rdon` | Anne Thirdperson Shoulder Fix | 无参数 | 无（任意玩家可用） | 启用 Anne 第三人称肩部视角 | `l4d2_anne_thirdperson_fix.sp:71` |
| `sm_8ball` | 8Ball | <问题文本> | 无（任意玩家可用） | 8ball 随机回答一个问题 | `8ball.sp:19` |
| `sm_acc` | Player Statistics | [参数] | 无（任意玩家可用） | 输出生还者命中率统计 | `l4d2_playstats.sp:573` |
| `sm_add_caster_id` | L4D2 Caster System (Original built in readyup) | <SteamID> | 需要 ADMFLAG_BAN（d，封禁） | 把解说加入白名单，即允许自助注册的名单 | `caster_system.sp:60` |
| `sm_add_map_transition` | Map Transitions | <起始地图> <结束地图> | 服务器控制台命令，玩家无法使用 | 添加地图过场映射，服务器控制台命令 | `l4d2_map_transitions.sp:45` |
| `sm_addban` | SourceBans++: Main Plugin | <时间> <steamid> [原因] | 需要 ADMFLAG_RCON（i，RCON 命令） | 按 SteamID 添加封禁 | `sbpp_main.sp:175` |
| `sm_addbot` | AnneServer Server Function (quiet minimal) | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 添加一个不会被本插件踢出的生还者 Bot | `server.sp:114` |
| `sm_addmap` | l4d2_mixmap | <地图名> <标签...> | 服务器控制台命令，玩家无法使用 | 把地图加入 mixmap 池并设置标签，服务器控制台命令 | `l4d2_mixmap.sp:127` |
| `sm_addreadystring` | Add Text To Readyup Panel | <文本> | 服务器控制台命令，玩家无法使用 | 设置要加入准备面板的文本，服务器控制台命令 | `panel_text.sp:31` |
| `sm_adminmap_gentrans` | L4D2 Admin Mission Menu | 无参数 | 需要 ADMFLAG_RCON（i，RCON 命令） | 强制重新生成地图翻译文件 | `adminmenu_mission_list.sp:113` |
| `sm_adminrates` | Lightweight Spectating (merged+128+force-spec) | 无参数 | 需要 ADMFLAG_GENERIC（b，通用管理员） | 管理员手动提升，对局 128 tick、旁观 100 tick | `specrates.sp:122` |
| `sm_advertisements_reload` | Advertisements | 无参数 | 服务器控制台命令，玩家无法使用 | 重新加载广告配置，服务器控制台命令 | `advertisements.sp:76` |
| `sm_afk` | simple join | 无参数 | 无（任意玩家可用） | sm_away 的别名 | `join.sp:116` |
| `sm_afktest` | [L4D2]Survivor_AFK_Fix | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 源码未给描述；据 `duoren/survivor_afk_fix.sp:108-117` 的 `PrepSDKCall_SetFromConf(hGamedata, SDKConf_Signature, "CTerrorPlayer::GoAwayFromKeyboard")` 与 :123-131 回调 `AFKTEST` 中的 `SDKCall(hAFKSDKCall, client)`，推测为：调试命令，对指定玩家强制触发 `CTerrorPlayer::GoAwayFromKeyboard`（令其进入挂机/AFK 状态），用于验证本插件的 AFK 修复逻辑；另注意 :29 为 `#define DEBUG 0`，默认编译时该命令不会注册（置信度：高） | `survivor_afk_fix.sp:117` |
| `sm_aidiff` | AnneHappy Dynamic AI Difficulty | <0 至 6> | 需要 ADMFLAG_CONFIG（h，服务器配置） | 设置动态难度：0 自动，1 简单，2 普通，3 困难，4 专家，5 极限，6 音理 | `annehappy_dynamic_ai_difficulty.sp:125` |
| `sm_aidiff_reload` | AnneHappy Dynamic AI Difficulty | 无参数 | 需要 ADMFLAG_CONFIG（h，服务器配置） | 重新读取难度配置并应用当前难度 | `annehappy_dynamic_ai_difficulty.sp:126` |
| `sm_aippm` | AnneHappy Dynamic AI Difficulty | 无参数 | 无（任意玩家可用） | 显示当前 AnneHappy 动态难度与 PPM | `annehappy_dynamic_ai_difficulty.sp:124` |
| `sm_allmap` | l4d2_mixmap | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 显示所有官方地图的地图代码 | `l4d2_mixmap.sp:139` |
| `sm_allmaps` | l4d2_mixmap | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | sm_allmap 的别名 | `l4d2_mixmap.sp:140` |
| `sm_ammo` | 商店插件 | 无参数 | 无（任意玩家可用） | 快速购买子弹 | `rpg.sp:1122` |
| `sm_anne_thirdperson_status` | Anne Thirdperson Shoulder Fix | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 显示 Anne 第三人称修复的状态 | `l4d2_anne_thirdperson_fix.sp:73` |
| `sm_annedb_status` | Anne DB Connection Hub | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 在控制台输出共享数据库连接状态、目标数与待处理数 | `anne_db.sp:97` |
| `sm_anneguide` | Anne Telecom Server Mode Guide | 无参数 | 无（任意玩家可用） | 打开模式引导主菜单，sm_guide 的别名 | `new_player_guide.sp:244` |
| `sm_anonymous` | hextags | 无参数 | 需要 ADMFLAG_GENERIC（b，通用管理员） | 切换匿名模式，忽略 SteamID 与管理员组、权限相关的称号 | `hextags.sp:148` |
| `sm_applytags` | 商店插件 | 无参数 | 无（任意玩家可用） | 佩戴自己的自定义称号 | `rpg.sp:1131` |
| `sm_attachment_qc` | [ANY] Attachments API | <qc 文件夹路径> | 需要 ADMFLAG_ROOT（z，最高权限） | 解析 .qc 文件以获取模型挂点名称 | `attachments_api.sp:203` |
| `sm_attachment_reload` | [ANY] Attachments API | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 重新加载 attachments_api 数据配置 | `attachments_api.sp:204` |
| `sm_away` | simple join | 无参数 | 无（任意玩家可用） | 把自己转为旁观挂机状态，被控制时会被拒绝 | `join.sp:115` |
| `sm_b` | survivor_chat_select.sp（myinfo 缺 name，用文件名代替） | 无参数 | 无（任意玩家可用） | 把自己切换为 Bill，短别名 | `survivor_chat_select.sp:93` |
| `sm_ban` | SourceBans++: Main Plugin | <目标玩家> <分钟数或 0> [原因] | 需要 ADMFLAG_BAN（d，封禁） | 封禁玩家，时间单位为分钟，0 表示永久 | `sbpp_main.sp:173` |
| `sm_banip` | SourceBans++: Main Plugin | <IP 或目标玩家> <时间> [原因] | 需要 ADMFLAG_BAN（d，封禁） | 按 IP 或玩家封禁 | `sbpp_main.sp:174` |
| `sm_beam` | l4d_random_beam_item.sp（myinfo 缺 name，用文件名代替） | [bright 或 subtle 或 off 或 default 或 reset] | 无（任意玩家可用） | 设置自己的物品光束样式 | `l4d_random_beam_item.sp:373` |
| `sm_beamadd` | l4d_random_beam_item.sp（myinfo 缺 name，用文件名代替） | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 给准星指向的实体添加一条默认配置的光束 | `l4d_random_beam_item.sp:369` |
| `sm_beaminfo` | l4d_random_beam_item.sp（myinfo 缺 name，用文件名代替） | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 在聊天中输出准星指向实体的光束信息 | `l4d_random_beam_item.sp:365` |
| `sm_beamreload` | l4d_random_beam_item.sp（myinfo 缺 name，用文件名代替） | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 重新加载光束配置 | `l4d_random_beam_item.sp:366` |
| `sm_beamremove` | l4d_random_beam_item.sp（myinfo 缺 name，用文件名代替） | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 移除准星指向实体上的插件光束 | `l4d_random_beam_item.sp:367` |
| `sm_beamremoveall` | l4d_random_beam_item.sp（myinfo 缺 name，用文件名代替） | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 移除插件创建的所有光束 | `l4d_random_beam_item.sp:368` |
| `sm_bill` | survivor_chat_select.sp（myinfo 缺 name，用文件名代替） | 无参数 | 无（任意玩家可用） | 把自己使用的生还者切换为 Bill | `survivor_chat_select.sp:84` |
| `sm_blmenu` | l4d2_blacklist.sp（myinfo 缺 name，用文件名代替） | 无参数 | 无（任意玩家可用） | 打开黑名单主菜单 | `l4d2_blacklist.sp:240` |
| `sm_block` | l4d2_blacklist.sp（myinfo 缺 name，用文件名代替） | <目标玩家或 steam64 或名字> | 无（任意玩家可用） | 把目标玩家加入自己的黑名单 | `l4d2_blacklist.sp:236` |
| `sm_blocklimit` | l4d2_blacklist.sp（myinfo 缺 name，用文件名代替） | 无参数 | 无（任意玩家可用） | 查看自己的屏蔽上限 | `l4d2_blacklist.sp:239` |
| `sm_blocklist` | l4d2_blacklist.sp（myinfo 缺 name，用文件名代替） | [目标玩家或 steam64 或名字] | 无（任意玩家可用） | 查看自己或指定玩家的屏蔽列表 | `l4d2_blacklist.sp:238` |
| `sm_bonus` | Confogl's Competitive Mod | 无参数 | 无（任意玩家可用） | 在聊天中报告当前回合奖励，sm_health 的别名 | `ScoreMod.sp:96` |
| `sm_bonus` | L4D2 Scoremod+ | 无参数 | 无（任意玩家可用） | 源码未给描述；据 `optional/l4d2_hybrid_scoremod.sp:96-98` 三个命令全部注册到同一回调 `CmdBonus`（:209-246），该回调先取 `GetCmdArg(1)`，按 `full`（明细）/`lite`（单行）/无参（默认百分比）三种详略输出本队回血加成 HB、伤害加成 DB、药片加成 Pills 及其占上限百分比；回合结束后或由服务器控制台调用则不显示，推测为：“查看本队当前混合增分”的别名之一，与 `sm_damage`、`sm_bonus` 完全等价（命名分别对应加成的三个组成部分）（置信度：高） | `l4d2_hybrid_scoremod.sp:98` |
| `sm_bonus` | L4D2 Scoremod+ | 无参数 | 无（任意玩家可用） | 源码未给描述；据 `optional/l4d2_hybrid_scoremod_zone.sp:97-100` 三个命令全部注册到同一回调 `CmdBonus`（:210-247），该回调先取 `GetCmdArg(1)`，按 `full`（明细）/`lite`（单行）/无参（默认百分比）三种详略输出本队回血加成 HB、伤害加成 DB、药片加成 Pills 及其占上限百分比；回合结束后或由服务器控制台调用则不显示，推测为：“查看本队当前混合增分”的别名之一，与 `sm_damage`、`sm_bonus` 完全等价（命名分别对应加成的三个组成部分）（置信度：高） | `l4d2_hybrid_scoremod_zone.sp:99` |
| `sm_bonus` | Penalty bonus system | 无参数 | 无（任意玩家可用） | 显示本回合当前的额外奖励 | `l4d2_penalty_bonus.sp:117` |
| `sm_boss` | L4D2 Tank Control | 无参数 | 无（任意玩家可用） | sm_tank 的别名 | `l4d_tank_control_eq.sp:85` |
| `sm_boss` | [L4D2] Boss Percents/Vote Boss Hybrid | 无参数 | 无（任意玩家可用） | 显示 boss 刷新百分比 | `l4d_boss_percent.sp:105` |
| `sm_bossvote` | [L4D2] Vote Boss | <是否坦克 0或1> <百分比> | 无（任意玩家可用） | sm_voteboss 的别名 | `l4d_boss_vote.sp:54` |
| `sm_bossvote` | [L4D2] Vote Boss | <是否坦克 0或1> <百分比> | 无（任意玩家可用） | sm_voteboss 的别名 | `l4d_boss_vote.sp:47` |
| `sm_buy` | 商店插件 | 无参数 | 无（任意玩家可用） | 打开购买菜单，仅限游戏内的存活玩家 | `rpg.sp:1121` |
| `sm_c` | survivor_chat_select.sp（myinfo 缺 name，用文件名代替） | 无参数 | 无（任意玩家可用） | 把自己切换为 Coach，短别名 | `survivor_chat_select.sp:91` |
| `sm_cancelvote` | Vote for run command or cfg file | 无参数 | 需要 ADMFLAG_GENERIC（b，通用管理员） | 终止当前正在进行的投票 | `vote.sp:83` |
| `sm_cast` | L4D2 Caster System (Original built in readyup) | 无参数 | 无（任意玩家可用） | 把调用者自己注册为解说 | `caster_system.sp:63` |
| `sm_caster` | L4D2 Caster System (Original built in readyup) | <玩家> | 需要 ADMFLAG_BAN（d，封禁） | 把指定玩家注册为解说 | `caster_system.sp:58` |
| `sm_cf` | Coinflip | 无参数 | 无（任意玩家可用） | sm_coinflip 的别名 | `coinflip.sp:46` |
| `sm_cfg` | Config Description | 无参数 | 无（任意玩家可用） | sm_changelog 的别名 | `cfg_motd.sp:24` |
| `sm_ch` | HexTags Lite | 无参数 | 无（任意玩家可用） | 打开称号选择菜单，短别名 | `hextags_lite.sp:62` |
| `sm_ch` | hextags | 无参数 | 无（任意玩家可用） | 打开称号选择菜单，短别名 | `hextags.sp:152` |
| `sm_changelog` | Config Description | 无参数 | 无（任意玩家可用） | 显示描述当前配置的 MOTD 页面 | `cfg_motd.sp:23` |
| `sm_changelog` | L4D2 Change Log Command | 无参数 | 无（任意玩家可用） | 显示更新日志页面，源码未给描述 | `changelog.sp:21` |
| `sm_changestatusrates` | Lightweight Spectating Test | <玩家> | 需要 ADMFLAG_GENERIC（b，通用管理员） | 修改指定玩家的旁观 tick 状态 | `specrates_test.sp:51` |
| `sm_check_bhop` | Simple Anti-Bunnyhop | 无参数 | 无（任意玩家可用） | 在聊天中报告当前是否允许连跳以及按特感职业的豁免情况 | `l4d2_nobhaps.sp:85` |
| `sm_check_bhop` | Simple Anti-Bunnyhop | 无参数 | 无（任意玩家可用） | 在聊天中报告当前是否允许连跳及豁免情况 | `l4d2_nobhaps.sp:85` |
| `sm_checkladder` | Ai_Tank_Enhance2.0 | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 输出当前地图的梯子数量 | `ai_tank_2.sp:167` |
| `sm_chenghao` | HexTags Lite | 无参数 | 无（任意玩家可用） | 打开称号选择菜单，sm_tagslist 的别名 | `hextags_lite.sp:61` |
| `sm_chenghao` | hextags | 无参数 | 无（任意玩家可用） | 打开称号选择菜单，sm_tagslist 的别名 | `hextags.sp:151` |
| `sm_chmap` | l4d2_map_vote.sp（myinfo 缺 name，用文件名代替） | 无参数 | 无（任意玩家可用） | 打开换图菜单，sm_v3 的别名 | `l4d2_map_vote.sp:137` |
| `sm_chmatch` | Match Vote | <cfg 名> | 无（任意玩家可用） | 发起比赛模式变更请求 | `match_vote.sp:71` |
| `sm_chr` | 商店插件 | 无参数 | 无（任意玩家可用） | 快速购买一把二代单发霰弹枪 Chrome | `rpg.sp:1124` |
| `sm_clear` | VeteransOnly | 无参数 | 需要 ADMFLAG_GENERIC（b，通用管理员） | 清空缓存 | `veterans.sp:209` |
| `sm_coach` | survivor_chat_select.sp（myinfo 缺 name，用文件名代替） | 无参数 | 无（任意玩家可用） | 把自己使用的生还者切换为 Coach | `survivor_chat_select.sp:82` |
| `sm_coinflip` | Coinflip | 无参数 | 无（任意玩家可用） | 投硬币并在聊天中公布结果 | `coinflip.sp:45` |
| `sm_comms` | SourceBans++: SourceComms | 无参数 | 无（任意玩家可用） | 显示当前玩家的通信状态 | `sbpp_comms.sp:206` |
| `sm_con` | Ai_Tank_Enhance | 无参数 | 需要 ADMFLAG_BAN（d，封禁） | 让 AI Tank 立即执行消耗行为，测试用 | `ai_tank_new.sp:178` |
| `sm_consistencycheck` | sv_consistency fixes | [目标] | 需要 ADMFLAG_RCON（i，RCON 命令） | 对所有玩家或指定目标执行一致性检查 | `sv_consistency_fix.sp:53` |
| `sm_crash` | L4D2 Auto restart | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 手动重启服务器，与 sm_restart 等价 | `linux_auto_restart.sp:41` |
| `sm_csc` | survivor_chat_select.sp（myinfo 缺 name，用文件名代替） | 无参数 | 需要 ADMFLAG_GENERIC（b，通用管理员） | 打开菜单选择指定玩家的生还者角色 | `survivor_chat_select.sp:97` |
| `sm_csm` | survivor_chat_select.sp（myinfo 缺 name，用文件名代替） | 无参数 | 无（任意玩家可用） | 打开生还者角色选择菜单，是否仅管理员可用由 l4d_csm_admins_only 控制 | `survivor_chat_select.sp:98` |
| `sm_cur` | L4D2 Survivor Progress | 无参数 | 无（任意玩家可用） | 显示当前地图的生还者进度 | `current.sp:26` |
| `sm_current` | L4D2 Survivor Progress | 无参数 | 无（任意玩家可用） | sm_cur 的别名 | `current.sp:27` |
| `sm_cvar_test` | [ANY] Command and ConVar - Buffer Overflow Fixer | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 创建一个测试 ConVar 并输出其结果，用于缓冲区溢出修复验证 | `command_buffer.sp:153` |
| `sm_cz` | hextags | 无参数 | 无（任意玩家可用） | 重新加载 HexTags 配置，sm_reloadtags 的别名 | `hextags.sp:149` |
| `sm_da_recalc` | L4D2 Dynamic Ammo (dirspawn only) | 无参数 | 需要 ADMFLAG_GENERIC（b，通用管理员） | 手动重算並应用动态弹药倍率 | `l4d2_dynamic_ammo.sp:235` |
| `sm_damage` | L4D2 Scoremod+ | 无参数 | 无（任意玩家可用） | 源码未给描述；据 `optional/l4d2_hybrid_scoremod.sp:96-98` 三个命令全部注册到同一回调 `CmdBonus`（:209-246），该回调先取 `GetCmdArg(1)`，按 `full`（明细）/`lite`（单行）/无参（默认百分比）三种详略输出本队回血加成 HB、伤害加成 DB、药片加成 Pills 及其占上限百分比；回合结束后或由服务器控制台调用则不显示，推测为：“查看本队当前混合增分”的别名之一，与 `sm_damage`、`sm_bonus` 完全等价（命名分别对应加成的三个组成部分）（置信度：高） | `l4d2_hybrid_scoremod.sp:97` |
| `sm_damage` | L4D2 Scoremod+ | 无参数 | 无（任意玩家可用） | 源码未给描述；据 `optional/l4d2_hybrid_scoremod_zone.sp:97-100` 三个命令全部注册到同一回调 `CmdBonus`（:210-247），该回调先取 `GetCmdArg(1)`，按 `full`（明细）/`lite`（单行）/无参（默认百分比）三种详略输出本队回血加成 HB、伤害加成 DB、药片加成 Pills 及其占上限百分比；回合结束后或由服务器控制台调用则不显示，推测为：“查看本队当前混合增分”的别名之一，与 `sm_damage`、`sm_bonus` 完全等价（命名分别对应加成的三个组成部分）（置信度：高） | `l4d2_hybrid_scoremod_zone.sp:98` |
| `sm_dance` | SM Fortnite Emotes Extended | 无参数 | 无（任意玩家可用） | 打开舞蹈菜单，sm_dances 的别名 | `fornite_l4d.sp:96` |
| `sm_dances` | SM Fortnite Emotes Extended | 无参数 | 无（任意玩家可用） | 打开舞蹈菜单，是否可用由 sm_dances_admin_flag_menu 控制 | `fornite_l4d.sp:95` |
| `sm_decrease_specspeed` | Caster Assister | 无参数 | 无（任意玩家可用） | 降低旁观速度 | `caster_assister.sp:30` |
| `sm_delbot` | AnneServer Server Function (quiet minimal) | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 删除一个未被接管的生还者 Bot | `server.sp:115` |
| `sm_detonate_rock` | L4D2 Black&White Rock Hit | 无参数 | 无（任意玩家可用） | 立即引爆坦克投出的石头 | `l4d2_bw_rock_hit.sp:67` |
| `sm_dirspawn_apply` | l4d2_dirspawn.sp（myinfo 缺 name，用文件名代替） | [总特数] [间隔] | 需要 ADMFLAG_GENERIC（b，通用管理员） | 立即应用导演刷特设置 | `l4d2_dirspawn.sp:1149` |
| `sm_dirspawn_genkv` | l4d2_dirspawn.sp（myinfo 缺 name，用文件名代替） | [min] [max] | 需要 ADMFLAG_ROOT（z，最高权限） | 生成均衡的每类上限 KV 文件到 dirspawn_kv_path | `l4d2_dirspawn.sp:1150` |
| `sm_dmgcookie` | [L4D2] Damage HUD (MySQL+Cookie, DB-first, fixes) | 无参数 | 需要 Admin_Generic | 把玩家的 Cookie 与当前设置打印到控制台 | `l4d2_damage_show.sp:1087` |
| `sm_dmgdbprobe` | [L4D2] Damage HUD (MySQL+Cookie, DB-first, fixes) | 无参数 | 需要 Admin_Generic | 执行一次同步数据库连接探测 | `l4d2_damage_show.sp:1090` |
| `sm_dmgdbstat` | [L4D2] Damage HUD (MySQL+Cookie, DB-first, fixes) | 无参数 | 需要 Admin_Generic | 在控制台显示数据库状态与会话字符集 | `l4d2_damage_show.sp:1088` |
| `sm_dmgforcesavecookie` | [L4D2] Damage HUD (MySQL+Cookie, DB-first, fixes) | 无参数 | 需要 Admin_Generic | 强制把当前设置保存到 Cookie | `l4d2_damage_show.sp:1089` |
| `sm_dmgmenu` | [L4D2] Damage HUD (MySQL+Cookie, DB-first, fixes) | 无参数 | 无（任意玩家可用） | 打开伤害数字设置菜单 | `l4d2_damage_show.sp:1086` |
| `sm_door_drop` | [L4D & L4D2] Saferoom Door Spam Protection | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 测试命令，让准星指向的门倒下 | `l4d_safe_door_spam.sp:295` |
| `sm_door_fall` | [L4D & L4D2] Saferoom Door Spam Protection | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 测试命令，让第一扇上锁的安全门倒下 | `l4d_safe_door_spam.sp:296` |
| `sm_e` | survivor_chat_select.sp（myinfo 缺 name，用文件名代替） | 无参数 | 无（任意玩家可用） | 把自己切换为 Ellis，短别名 | `survivor_chat_select.sp:90` |
| `sm_ellis` | survivor_chat_select.sp（myinfo 缺 name，用文件名代替） | 无参数 | 无（任意玩家可用） | 把自己使用的生还者切换为 Ellis | `survivor_chat_select.sp:81` |
| `sm_emote` | SM Fortnite Emotes Extended | 无参数 | 无（任意玩家可用） | 打开表情菜单，sm_emotes 的别名 | `fornite_l4d.sp:94` |
| `sm_emotes` | SM Fortnite Emotes Extended | 无参数 | 无（任意玩家可用） | 打开表情菜单，是否可用由 sm_emotes_admin_flag_menu 控制 | `fornite_l4d.sp:93` |
| `sm_f` | survivor_chat_select.sp（myinfo 缺 name，用文件名代替） | 无参数 | 无（任意玩家可用） | 把自己切换为 Francis，短别名 | `survivor_chat_select.sp:94` |
| `sm_fchmatch` | Confogl's Competitive Mod | <cfg> | 需要 ADMFLAG_CONFIG（h，服务器配置） | 强制切换比赛配置，sm_forcechangematch 的短别名 | `ReqMatch.sp:68` |
| `sm_ff` | Player Statistics | [参数] | 无（任意玩家可用） | 输出友军伤害统计 | `l4d2_playstats.sp:572` |
| `sm_firesel` | hextags | 无参数 | 无（任意玩家可用） | 触发称号选择相关前向，源码仅给出触发参数 thistoggle | `hextags.sp:169` |
| `sm_fixbots` | Player Management Plugin | 无参数 | 需要 ADMFLAG_BAN（d，封禁） | 补充生还者 Bot 到 survivor_limit | `playermanagement.sp:70` |
| `sm_flip` | Coinflip | 无参数 | 无（任意玩家可用） | sm_coinflip 的别名 | `coinflip.sp:47` |
| `sm_fm` | Confogl's Competitive Mod | <cfg> [mode] | 需要 ADMFLAG_CONFIG（h，服务器配置） | 强制加载比赛模式，sm_forcematch 的短别名 | `ReqMatch.sp:65` |
| `sm_fmixmap` | l4d2_mixmap | <cfg 名> | 需要 ADMFLAG_ROOT（z，最高权限） | 强制启用 mixmap，参数为空时使用默认地图池 | `l4d2_mixmap.sp:132` |
| `sm_forcechangematch` | Confogl's Competitive Mod | <cfg> | 需要 ADMFLAG_CONFIG（h，服务器配置） | 强制切换比赛配置 | `ReqMatch.sp:67` |
| `sm_forcematch` | Confogl's Competitive Mod | <cfg> [mode] | 需要 ADMFLAG_CONFIG（h，服务器配置） | 强制加载比赛模式 | `ReqMatch.sp:64` |
| `sm_forcemode` | Weapon Loadout | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 强制打开武器配置投票菜单 | `weapon_loadout_vote.sp:86` |
| `sm_forcepause` | Pause plugin | 无参数 | 需要 ADMFLAG_BAN（d，封禁） | 暂停游戏且只允许管理员取消暂停 | `pause.sp:136` |
| `sm_forcestart` | Pause plugin | 无参数 | 需要 ADMFLAG_BAN（d，封禁） | sm_forceunpause 的别名 | `pause.sp:138` |
| `sm_forceunpause` | Pause plugin | 无参数 | 需要 ADMFLAG_BAN（d，封禁） | 无视队伍准备状态取消暂停，用于取消管理员发起的暂停 | `pause.sp:137` |
| `sm_francis` | survivor_chat_select.sp（myinfo 缺 name，用文件名代替） | 无参数 | 无（任意玩家可用） | 把自己使用的生还者切换为 Francis | `survivor_chat_select.sp:85` |
| `sm_fs` | Pause plugin | 无参数 | 需要 ADMFLAG_BAN（d，封禁） | sm_forceunpause 的短别名 | `pause.sp:139` |
| `sm_fstopmixmap` | l4d2_mixmap | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 强制中止 mixmap | `l4d2_mixmap.sp:135` |
| `sm_ftank` | [L4D2] Vote Boss | <百分比> | 需要 ADMFLAG_BAN（d，封禁） | 管理员强制设置坦克刷新百分比 | `l4d_boss_vote.sp:56` |
| `sm_ftank` | [L4D2] Vote Boss | <百分比> | 需要 ADMFLAG_BAN（d，封禁） | 管理员强制设置坦克刷新百分比 | `l4d_boss_vote.sp:49` |
| `sm_fwitch` | [L4D2] Vote Boss | <百分比> | 需要 ADMFLAG_BAN（d，封禁） | 管理员强制设置女巫刷新百分比 | `l4d_boss_vote.sp:57` |
| `sm_fwitch` | [L4D2] Vote Boss | <百分比> | 需要 ADMFLAG_BAN（d，封禁） | 管理员强制设置女巫刷新百分比 | `l4d_boss_vote.sp:50` |
| `sm_get_restricted_strings` | [L4D & L4D2] Additive Staged FastDL | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 列出玩家首次连接时会加入的受限文件清单 | `l4d2_blackscreen_fix.sp:63` |
| `sm_getstatusrates` | Lightweight Spectating Test | <玩家> | 需要 ADMFLAG_GENERIC（b，通用管理员） | 查询指定玩家的旁观 tick 状态 | `specrates_test.sp:52` |
| `sm_gettagvars` | hextags | 无参数 | 无（任意玩家可用） | 源码未给描述；据 `extend/hextags.sp:167-170` 该命令注册在 `#if defined DEBUG` 块内，回调 `Cmd_GetVars`（:460-467）连续四次 `ReplyToCommand` 输出 `selectedTags[client]` 的 `ScoreTag`、`ChatTag`、`ChatColor`、`NameColor`，推测为：调试命令，把调用者当前选中的计分板标签、聊天前缀、聊天颜色、名字颜色四个变量值回显到控制台，用于排查标签为何不生效；:22 的 `//#define DEBUG 0` 仍处于被注释状态，故默认编译不会注册（置信度：高） | `hextags.sp:168` |
| `sm_getteam` | hextags | 无参数 | 无（任意玩家可用） | 显示当前队伍名称 | `hextags.sp:153` |
| `sm_give_starting_items` | Starting Items | 无参数 | 需要 ADMFLAG_KICK（c，踢出） | 立即给生还者发放起始物品 | `starting_items.sp:56` |
| `sm_givetank` | L4D2 Tank Control | <玩家> | 需要 ADMFLAG_SLAY（f，处死） | 把坦克交给指定玩家 | `l4d_tank_control_eq.sp:81` |
| `sm_gnome` | [L4D2] Healing Gnome | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 在准星位置生成一只临时地精，仅可用于服务器端 | `gnome.sp:245` |
| `sm_gnomeang` | [L4D2] Healing Gnome | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 打开菜单调整准星所指地精的角度 | `gnome.sp:252` |
| `sm_gnomedel` | [L4D2] Healing Gnome | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 删除准星指向的地精，若已保存则同时从配置中删除 | `gnome.sp:247` |
| `sm_gnomeglow` | [L4D2] Healing Gnome | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 切换是否让所有地精发光，便于查看摆放位置 | `gnome.sp:249` |
| `sm_gnomekill` | [L4D2] Healing Gnome | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 删除当前地图所有地精并从配置中删除 | `gnome.sp:248` |
| `sm_gnomelist` | [L4D2] Healing Gnome | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 列出地精坐标与总数 | `gnome.sp:250` |
| `sm_gnomepos` | [L4D2] Healing Gnome | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 打开菜单调整准星所指地精的坐标 | `gnome.sp:253` |
| `sm_gnomesave` | [L4D2] Healing Gnome | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 在准星位置生成地精并把位置保存到配置 | `gnome.sp:246` |
| `sm_gnometele` | [L4D2] Healing Gnome | <序号 1 至 MAX_GNOMES> | 需要 ADMFLAG_ROOT（z，最高权限） | 传送到指定序号的地精 | `gnome.sp:251` |
| `sm_guide` | Anne Telecom Server Mode Guide | 无参数 | 无（任意玩家可用） | 打开模式引导主菜单 | `new_player_guide.sp:241` |
| `sm_hat` | [L4D & L4D2] Hats | [帽子编号] | 无（任意玩家可用） | 打开帽子列表菜单以更换自己佩戴的帽子 | `l4d_hats.sp:855` |
| `sm_hatadd` | [L4D & L4D2] Hats | <完整模型路径> | 需要 ADMFLAG_ROOT（z，最高权限） | 把指定模型加入帽子配置 | `l4d_hats.sp:868` |
| `sm_hatall` | [L4D & L4D2] Hats | 无参数 | 无（任意玩家可用） | 切换所有人帽子的可见性 | `l4d_hats.sp:861` |
| `sm_hatallc` | [L4D & L4D2] Hats | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 打开玩家列表以切换指定玩家帽子是否可见 | `l4d_hats.sp:864` |
| `sm_hatang` | [L4D & L4D2] Hats | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 打开菜单调整帽子角度，影响所有帽子与玩家 | `l4d_hats.sp:873` |
| `sm_hatc` | [L4D & L4D2] Hats | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 打开玩家列表，选择一名玩家修改其帽子 | `l4d_hats.sp:865` |
| `sm_hatclient` | [L4D & L4D2] Hats | <目标玩家> [帽子名或索引 0 至 128] | 需要 ADMFLAG_ROOT（z，最高权限） | 给目标玩家设置帽子 | `l4d_hats.sp:862` |
| `sm_hatdel` | [L4D & L4D2] Hats | <索引或部分名称> | 需要 ADMFLAG_ROOT（z，最高权限） | 从帽子配置中删除模型 | `l4d_hats.sp:869` |
| `sm_hatlist` | [L4D & L4D2] Hats | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 列出所有帽子模型，供 sm_hatdel 使用 | `l4d_hats.sp:870` |
| `sm_hatload` | [L4D & L4D2] Hats | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 把所有玩家的帽子改成自己当前佩戴的帽子 | `l4d_hats.sp:872` |
| `sm_hatoff` | [L4D & L4D2] Hats | 无参数 | 无（任意玩家可用） | 切换是否佩戴帽子 | `l4d_hats.sp:856` |
| `sm_hatoffc` | [L4D & L4D2] Hats | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 打开玩家列表以切换指定玩家能否佩戴帽子 | `l4d_hats.sp:863` |
| `sm_hatpos` | [L4D & L4D2] Hats | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 打开菜单调整帽子位置，影响所有帽子与玩家 | `l4d_hats.sp:874` |
| `sm_hatrand` | [L4D & L4D2] Hats | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 随机化所有玩家的帽子，sm_hatrandom 的别名 | `l4d_hats.sp:867` |
| `sm_hatrandom` | [L4D & L4D2] Hats | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 随机化所有玩家的帽子 | `l4d_hats.sp:866` |
| `sm_hats` | [L4D & L4D2] Hats | 无参数 | 无（任意玩家可用） | 打开帽子自定义设置菜单 | `l4d_hats.sp:854` |
| `sm_hatsave` | [L4D & L4D2] Hats | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 保存当前帽子的位置与角度到帽子配置 | `l4d_hats.sp:871` |
| `sm_hatshow` | [L4D & L4D2] Hats | [开或关] | 无（任意玩家可用） | 切换是否显示自己的帽子 | `l4d_hats.sp:857` |
| `sm_hatshowoff` | [L4D & L4D2] Hats | 无参数 | 无（任意玩家可用） | 隐藏自己的帽子 | `l4d_hats.sp:860` |
| `sm_hatshowon` | [L4D & L4D2] Hats | 无参数 | 无（任意玩家可用） | 显示自己的帽子 | `l4d_hats.sp:859` |
| `sm_hatsize` | [L4D & L4D2] Hats | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 打开菜单调整帽子大小，影响所有帽子与玩家 | `l4d_hats.sp:875` |
| `sm_hatview` | [L4D & L4D2] Hats | [开或关] | 无（任意玩家可用） | 切换是否显示自己的帽子，sm_hatshow 的别名 | `l4d_hats.sp:858` |
| `sm_hbonus` | Holdout Bonus | 无参数 | 无（任意玩家可用） | 显示当前 Holdout 奖励 | `holdout_bonus.sp:147` |
| `sm_health` | Confogl's Competitive Mod | 无参数 | 无（任意玩家可用） | 在聊天中报告当前生还者平均血量与回合奖励 | `ScoreMod.sp:95` |
| `sm_health` | L4D2 Scoremod | 无参数 | 无（任意玩家可用） | 显示当前生还者平均血量与回合奖励 | `l4d2_scoremod.sp:127` |
| `sm_health` | L4D2 Scoremod+ | 无参数 | 无（任意玩家可用） | 源码未给描述；据 `optional/l4d2_hybrid_scoremod.sp:96-98` 三个命令全部注册到同一回调 `CmdBonus`（:209-246），该回调先取 `GetCmdArg(1)`，按 `full`（明细）/`lite`（单行）/无参（默认百分比）三种详略输出本队回血加成 HB、伤害加成 DB、药片加成 Pills 及其占上限百分比；回合结束后或由服务器控制台调用则不显示，推测为：“查看本队当前混合增分”的别名之一，与 `sm_damage`、`sm_bonus` 完全等价（命名分别对应加成的三个组成部分）（置信度：高） | `l4d2_hybrid_scoremod.sp:96` |
| `sm_health` | L4D2 Scoremod+ | 无参数 | 无（任意玩家可用） | 源码未给描述；据 `optional/l4d2_hybrid_scoremod_zone.sp:97-100` 三个命令全部注册到同一回调 `CmdBonus`（:210-247），该回调先取 `GetCmdArg(1)`，按 `full`（明细）/`lite`（单行）/无参（默认百分比）三种详略输出本队回血加成 HB、伤害加成 DB、药片加成 Pills 及其占上限百分比；回合结束后或由服务器控制台调用则不显示，推测为：“查看本队当前混合增分”的别名之一，与 `sm_damage`、`sm_bonus` 完全等价（命名分别对应加成的三个组成部分）（置信度：高） | `l4d2_hybrid_scoremod_zone.sp:97` |
| `sm_hide` | Pause plugin | 无参数 | 无（任意玩家可用） | 显示被隐藏的暂停面板 | `pause.sp:142` |
| `sm_hitsound_reload` | L4D2 Hit/Kill Feedback Plus | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 重新从数据库与 KV 读取所有在线玩家的偏好 | `l4d2_hitsound.sp:774` |
| `sm_hud` | l4d2_scripted_hud.sp（myinfo 缺 name，用文件名代替） | 无参数 | 无（任意玩家可用） | 打开脚本化 HUD 设置菜单，sm_hudmenu 的别名 | `l4d2_scripted_hud.sp:901` |
| `sm_hudlang` | l4d2_scripted_hud.sp（myinfo 缺 name，用文件名代替） | 无参数 | 无（任意玩家可用） | 打开显示语言菜单 | `l4d2_scripted_hud.sp:902` |
| `sm_hudmenu` | l4d2_scripted_hud.sp（myinfo 缺 name，用文件名代替） | 无参数 | 无（任意玩家可用） | 打开脚本化 HUD 设置菜单 | `l4d2_scripted_hud.sp:900` |
| `sm_hunter_patch_print_cvars` | L4D2 hunter patch | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 输出 l4d2_hunter_patch 相关 cvar 的当前取值 | `l4d2_hunter_patch.sp:46` |
| `sm_increase_specspeed` | Caster Assister | 无参数 | 无（任意玩家可用） | 提高旁观速度 | `caster_assister.sp:29` |
| `sm_inf` | simple join | 无参数 | 无（任意玩家可用） | 加入特感队伍，短别名 | `join.sp:121` |
| `sm_infected` | simple join | 无参数 | 无（任意玩家可用） | 加入特感队伍，sm_joininfected 的别名 | `join.sp:122` |
| `sm_ip` | simple join | 无参数 | 无（任意玩家可用） | 打开服务器 IP 与 MOTD 页面 | `join.sp:132` |
| `sm_it` | Direct InfectedSpawn (directed-nav + maxdist-fallback) | [特感职业] | 无（任意玩家可用） | sm_neigui 的别名 | `infected_control.sp:520` |
| `sm_it` | Direct InfectedSpawn (fdxx-nav + buckets + maxdist-fallback) | [特感职业] | 无（任意玩家可用） | sm_neigui 的别名 | `infected_control26-07.sp:326` |
| `sm_itcancel` | Direct InfectedSpawn (directed-nav + maxdist-fallback) | 无参数 | 无（任意玩家可用） | sm_neiguicancel 的别名 | `infected_control.sp:522` |
| `sm_jg` | simple join | 无参数 | 无（任意玩家可用） | 加入生还者队伍，短别名 | `join.sp:125` |
| `sm_join` | simple join | 无参数 | 无（任意玩家可用） | 加入生还者队伍，受人数上限限制 | `join.sp:124` |
| `sm_joingame` | simple join | 无参数 | 无（任意玩家可用） | 加入生还者队伍，sm_join 的别名 | `join.sp:127` |
| `sm_joininfected` | simple join | 无参数 | 无（任意玩家可用） | 加入特感队伍，受内鬼模式与人数上限限制 | `join.sp:119` |
| `sm_kickspecs` | L4D2 Caster System (Original built in readyup) | 无参数 | 无（任意玩家可用） | 发起投票踢出旁观的解说 | `caster_system.sp:68` |
| `sm_kicktank` | AnneServer Server Function (quiet minimal) | 无参数 | 需要 ADMFLAG_KICK（c，踢出） | 有多只 Tank 时随机踢到只剩一只 | `server.sp:112` |
| `sm_kill` | AnneServer Server Function (quiet minimal) | 无参数 | 无（任意玩家可用） | 自杀，sm_zs 的别名 | `server.sp:117` |
| `sm_killall` | text.sp（myinfo 缺 name，用文件名代替） | 无参数 | 需要 ADMFLAG_BAN（d，封禁） | 处死所有玩家 | `text.sp:64` |
| `sm_killlobbyres` | Confogl's Competitive Mod | 无参数 | 需要 ADMFLAG_BAN（d，封禁） | 清除大厅预留，使更多玩家可以加入 | `UnreserveLobby.sp:18` |
| `sm_kills` | Kills Statistic | 无参数 | 无（任意玩家可用） | 查看击杀统计，即 MVP 统计 | `HitStatistics.sp:46` |
| `sm_kills` | Survivor MVP notification | 无参数 | 无（任意玩家可用） | 显示 AnneHappy 精简生还者统计 | `survivor_mvp.sp:290` |
| `sm_killsme` | Kills Statistic | 无参数 | 无（任意玩家可用） | 查看自己的击杀统计 | `HitStatistics.sp:47` |
| `sm_l` | survivor_chat_select.sp（myinfo 缺 name，用文件名代替） | 无参数 | 无（任意玩家可用） | 把自己切换为 Louis，短别名 | `survivor_chat_select.sp:95` |
| `sm_l4d2_scripted_hud_reload_data` | l4d2_scripted_hud.sp（myinfo 缺 name，用文件名代替） | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 重新加载 HUD 文本数据文件 | `l4d2_scripted_hud.sp:896` |
| `sm_l4dd` | [L4D & L4D2] Left 4 DHooks Direct - TESTER | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 输出区域与距离等调试信息，来自测试插件 | `left4dhooks_test.sp:112` |
| `sm_l4dd_calls` | [L4D & L4D2] Left 4 DHooks Direct - TESTER | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 把前向允许记录的调用次数加 1，来自测试插件 | `left4dhooks_test.sp:113` |
| `sm_l4dd_detours` | [L4D & L4D2] Left 4 DHooks Direct | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 列出当前生效的前向及其使用插件 | `left4dhooks.sp:725` |
| `sm_l4dd_reload` | [L4D & L4D2] Left 4 DHooks Direct | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 重新加载 detour 钩子，按需启用或禁用 | `left4dhooks.sp:724` |
| `sm_l4dd_unreserve` | [L4D & L4D2] Left 4 DHooks Direct | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 移除大厅预留 | `left4dhooks.sp:723` |
| `sm_l4df` | [L4D & L4D2] Left 4 DHooks Direct - TESTER | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 输出已触发前向的数量统计，来自测试插件 | `left4dhooks_test.sp:111` |
| `sm_l4dhooks_detours` | [L4D & L4D2] Left 4 DHooks Direct | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 列出当前生效的前向及其使用插件，sm_l4dd_detours 的别名 | `left4dhooks.sp:727` |
| `sm_l4dhooks_reload` | [L4D & L4D2] Left 4 DHooks Direct | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 重新加载 detour 钩子，sm_l4dd_reload 的别名 | `left4dhooks.sp:726` |
| `sm_ladder_debug` | [L4D2] Infected Ladder Speed Boost | 无参数 | 需要 ADMFLAG_GENERIC（b，通用管理员） | 在控制台输出爬梯加速调试信息 | `l4d2_ai_ladder_boost.sp:109` |
| `sm_ladder_status` | [L4D2] Infected Ladder Speed Boost | 无参数 | 需要 ADMFLAG_GENERIC（b，通用管理员） | 显示当前获得爬梯加速的特感数量 | `l4d2_ai_ladder_boost.sp:110` |
| `sm_lang` | l4d2_scripted_hud.sp（myinfo 缺 name，用文件名代替） | 无参数 | 无（任意玩家可用） | 打开显示语言菜单，sm_hudlang 的别名 | `l4d2_scripted_hud.sp:903` |
| `sm_lerps` | LerpMonitor++ | 无参数 | 无（任意玩家可用） | 列出所有在场玩家的 lerp 设置 | `lerpmonitor.sp:77` |
| `sm_listbans` | SourceBans++: Bans Checker | 无参数 | 需要 ADMFLAG_BAN（d，封禁） | 列出封禁记录 | `sbpp_checker.sp:56` |
| `sm_listcomms` | SourceBans++: Bans Checker | 无参数 | 需要 ADMFLAG_BAN（d，封禁） | 列出禁言与禁声记录 | `sbpp_checker.sp:57` |
| `sm_listen` | SpecListener | <菜单项> | 无（任意玩家可用） | 打开旁观者监听菜单，包含准备与暂停面板 | `SpecListener.sp:86` |
| `sm_lobby_set` | L4D2 Lobby match manager | <cookie> <bAllowLobbyConnectOnly> <bHostingLobby> <bUpdateGameType> | 需要 ADMFLAG_ROOT（z，最高权限） | 直接设置大厅预留 Cookie 的各字段 | `l4d2_lobby_match_manager.sp:118` |
| `sm_lobby_status` | L4D2 Lobby match manager | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 显示大厅预留状态与槽位信息 | `l4d2_lobby_match_manager.sp:117` |
| `sm_lobby_unreserve` | L4D2 Lobby match manager | 无参数 | 服务器控制台命令，玩家无法使用 | 移除大厅预留使更多玩家可以加入，服务器控制台命令 | `l4d2_lobby_match_manager.sp:119` |
| `sm_lock` | L4D2 Saferoom Locker | 无参数 | 需要 ADMFLAG_GENERIC（b，通用管理员） | 强制上锁安全室的门 | `l4d2_door_lock.sp:219` |
| `sm_lockstrings` | Add Text To Readyup Panel | 无参数 | 服务器控制台命令，玩家无法使用 | 锁定准备面板文本，服务器控制台命令 | `panel_text.sp:33` |
| `sm_loss` | Network Quality Hint | 无参数 | 无（任意玩家可用） | sm_net 的别名 | `network_quality_hint.sp:119` |
| `sm_louis` | survivor_chat_select.sp（myinfo 缺 name，用文件名代替） | 无参数 | 无（任意玩家可用） | 把自己使用的生还者切换为 Louis | `survivor_chat_select.sp:86` |
| `sm_manualmixmap` | l4d2_mixmap | <地图顺序> | 需要 ADMFLAG_ROOT（z，最高权限） | 按指定顺序启用 mixmap | `l4d2_mixmap.sp:131` |
| `sm_mapinfo` | L4D2 Scoremod+ | 无参数 | 无（任意玩家可用） | 源码未给描述；据 `optional/l4d2_hybrid_scoremod.sp:99` 注册到 `CmdMapInfo`（:248-267，回调体逐条 `CPrintToChat`）：输出队伍规模、地图距离、总分（含药片上限）、生命加成/伤害加成/药片加成各自上限与占比、平局加分，数据来自 `OnConfigsExecuted`（:119-139）中按 `sm2_bonus_per_survivor_multiplier`×人数×地图距离算出的 `fMapBonus` 等全局量，推测为：显示本张地图 ScoreMod 2 增分配置的说明命令（置信度：高） | `l4d2_hybrid_scoremod.sp:99` |
| `sm_mapinfo` | L4D2 Scoremod+ | 无参数 | 无（任意玩家可用） | 源码未给描述；据 `optional/l4d2_hybrid_scoremod_zone.sp:100` 注册到 `CmdMapInfo`（:249-268，回调体逐条 `CPrintToChat`）：输出队伍规模、地图距离、总分（含药片上限）、生命加成/伤害加成/药片加成各自上限与占比、平局加分，数据来自 `OnConfigsExecuted`（:120-140）中按 `sm2_bonus_per_survivor_multiplier`×人数×地图距离算出的 `fMapBonus` 等全局量，推测为：显示本张地图 ScoreMod 2 增分配置的说明命令（置信度：高） | `l4d2_hybrid_scoremod_zone.sp:100` |
| `sm_maplist` | l4d2_mixmap | 无参数 | 无（任意玩家可用） | 显示 mixmap 最终抽取的地图列表 | `l4d2_mixmap.sp:138` |
| `sm_mapnext` | l4d2_map_vote.sp（myinfo 缺 name，用文件名代替） | 无参数 | 无（任意玩家可用） | 对下一张地图发起投票，终局地图需前置插件支持 | `l4d2_map_vote.sp:142` |
| `sm_maps` | l4d2_map_vote.sp（myinfo 缺 name，用文件名代替） | 无参数 | 无（任意玩家可用） | 打开地图列表菜单，sm_v3 的别名 | `l4d2_map_vote.sp:136` |
| `sm_mapvote` | l4d2_map_vote.sp（myinfo 缺 name，用文件名代替） | 无参数 | 无（任意玩家可用） | 打开换图投票菜单，sm_v3 的别名 | `l4d2_map_vote.sp:138` |
| `sm_match` | Match Vote | <cfg 名> | 无（任意玩家可用） | 发起比赛模式切换请求 | `match_vote.sp:70` |
| `sm_melee` | Melee In The Saferoom | 无参数 | 需要 ADMFLAG_KICK（c，踢出） | 列出当前战役可刷出的所有近战武器 | `MeleeInTheSafeRoom.sp:92` |
| `sm_missions_export` | l4d2_map_vote.sp（myinfo 缺 name，用文件名代替） | <文件名> | 需要 ADMFLAG_ROOT（z，最高权限） | 把任务 KV 导出为文件 | `l4d2_map_vote.sp:148` |
| `sm_mixmap` | l4d2_mixmap | <cfg 名> | 无（任意玩家可用） | 发起启用 mixmap 的投票 | `l4d2_mixmap.sp:133` |
| `sm_mode` | Weapon Loadout | 无参数 | 无（任意玩家可用） | 打开武器配置投票菜单 | `weapon_loadout_vote.sp:84` |
| `sm_modeguide` | Anne Telecom Server Mode Guide | 无参数 | 无（任意玩家可用） | 打开模式引导主菜单，sm_guide 的别名 | `new_player_guide.sp:243` |
| `sm_modes` | Anne Telecom Server Mode Guide | 无参数 | 无（任意玩家可用） | 打开模式引导主菜单，sm_guide 的别名 | `new_player_guide.sp:242` |
| `sm_mvp` | Player Statistics | [参数] | 无（任意玩家可用） | 输出生还者 MVP 统计 | `l4d2_playstats.sp:570` |
| `sm_mvp` | Survivor MVP notification | 无参数 | 无（任意玩家可用） | 显示当前生还者队伍的 MVP | `survivor_mvp.sp:288` |
| `sm_mvpme` | Survivor MVP notification | 无参数 | 无（任意玩家可用） | 显示自己的 MVP 相关统计 | `survivor_mvp.sp:289` |
| `sm_mvptest` | Survivor MVP Test | 无参数 | 无（任意玩家可用） | 显示 Survivor MVP 的全部可用统计 | `survivor_mvp_test.sp:45` |
| `sm_n` | survivor_chat_select.sp（myinfo 缺 name，用文件名代替） | 无参数 | 无（任意玩家可用） | 把自己切换为 Nick，短别名 | `survivor_chat_select.sp:89` |
| `sm_nav_variant_clearcache` | L4D2 Nav Variant Loader | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 清除已缓存的 nav 文件数据，供下次加载使用 | `l4d2_nav_variant.sp:91` |
| `sm_nav_variant_reload` | L4D2 Nav Variant Loader | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 重新加载导航变体配置 | `l4d2_nav_variant.sp:90` |
| `sm_nav_variant_status` | L4D2 Nav Variant Loader | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 显示导航变体状态与最近一次重定向读取结果 | `l4d2_nav_variant.sp:92` |
| `sm_navgraph_export` | Anne NavGraph Export | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 把当前地图 Nav 图导出为 JSON 到 data/anne_navgraph/，请在非对局时执行 | `anne_navgraph_export.sp:46` |
| `sm_navpeek` | Direct InfectedSpawn (directed-nav + maxdist-fallback) | 无参数 | 需要 ADMFLAG_GENERIC（b，通用管理员） | 查看准星 Nav 的分桶与属性 | `infected_control.sp:514` |
| `sm_navpeek` | Direct InfectedSpawn (fdxx-nav + buckets + maxdist-fallback) | 无参数 | 需要 ADMFLAG_GENERIC（b，通用管理员） | 查看准星 Nav 的分桶与属性 | `infected_control26-07.sp:320` |
| `sm_navtest` | Direct InfectedSpawn (directed-nav + maxdist-fallback) | 无参数 | 需要 ADMFLAG_GENERIC（b，通用管理员） | 测试准星 Nav 能否生成特感及评分 | `infected_control.sp:516` |
| `sm_navtest` | Direct InfectedSpawn (fdxx-nav + buckets + maxdist-fallback) | 无参数 | 需要 ADMFLAG_GENERIC（b，通用管理员） | 测试准星 Nav 能否生成特感及评分 | `infected_control26-07.sp:322` |
| `sm_neigui` | Direct InfectedSpawn (directed-nav + maxdist-fallback) | [特感职业] | 无（任意玩家可用） | 进入内鬼刷特队列 | `infected_control.sp:519` |
| `sm_neigui` | Direct InfectedSpawn (fdxx-nav + buckets + maxdist-fallback) | [特感职业] | 无（任意玩家可用） | 进入内鬼刷特队列 | `infected_control26-07.sp:325` |
| `sm_neiguicancel` | Direct InfectedSpawn (directed-nav + maxdist-fallback) | 无参数 | 无（任意玩家可用） | 取消内鬼刷特队列 | `infected_control.sp:521` |
| `sm_neiguicancel` | Direct InfectedSpawn (fdxx-nav + buckets + maxdist-fallback) | 无参数 | 无（任意玩家可用） | 取消内鬼刷特队列 | `infected_control26-07.sp:327` |
| `sm_net` | Network Quality Hint | 无参数 | 无（任意玩家可用） | 显示自己的网络状态 | `network_quality_hint.sp:117` |
| `sm_nick` | survivor_chat_select.sp（myinfo 缺 name，用文件名代替） | 无参数 | 无（任意玩家可用） | 把自己使用的生还者切换为 Nick | `survivor_chat_select.sp:80` |
| `sm_notcasting` | L4D2 Caster System (Original built in readyup) | [玩家] | 无（任意玩家可用） | 取消自己或指定玩家的解说身份 | `caster_system.sp:64` |
| `sm_np` | Direct InfectedSpawn (directed-nav + maxdist-fallback) | 无参数 | 需要 ADMFLAG_GENERIC（b，通用管理员） | sm_navpeek 的别名 | `infected_control.sp:515` |
| `sm_np` | Direct InfectedSpawn (fdxx-nav + buckets + maxdist-fallback) | 无参数 | 需要 ADMFLAG_GENERIC（b，通用管理员） | sm_navpeek 的别名 | `infected_control26-07.sp:321` |
| `sm_nr` | Pause plugin | 无参数 | 无（任意玩家可用） | sm_unready 的别名 | `pause.sp:133` |
| `sm_nt` | Direct InfectedSpawn (directed-nav + maxdist-fallback) | 无参数 | 需要 ADMFLAG_GENERIC（b，通用管理员） | sm_navtest 的别名 | `infected_control.sp:517` |
| `sm_nt` | Direct InfectedSpawn (fdxx-nav + buckets + maxdist-fallback) | 无参数 | 需要 ADMFLAG_GENERIC（b，通用管理员） | sm_navtest 的别名 | `infected_control26-07.sp:323` |
| `sm_overhand` | Tank Attack Control | 无参数 | 无（任意玩家可用） | 切换坦克为上手投掷石头，仅坦克可用 | `l4d2_tank_attack_control.sp:61` |
| `sm_overonehand` | Tank Attack Control | 无参数 | 无（任意玩家可用） | 切换坦克为单手投掷石头，仅坦克可用 | `l4d2_tank_attack_control.sp:62` |
| `sm_pause` | Pause plugin | 无参数 | 无（任意玩家可用） | 暂停游戏 | `pause.sp:128` |
| `sm_peakstatus` | High Perk Server Status Control[Work in ZM] | 无参数 | 需要 ADMFLAG_GENERIC（b，通用管理员） | 查看当前全服高峰期的判定状态 | `l4d_player_count_unload_mode.sp:170` |
| `sm_pen` | 商店插件 | 无参数 | 无（任意玩家可用） | 快速随机购买一把单发霰弹枪 | `rpg.sp:1123` |
| `sm_picknumber` | Coinflip | <数字> <范围> | 无（任意玩家可用） | 猜数字玩法，随机开奖 | `coinflip.sp:49` |
| `sm_pill` | 商店插件 | 无参数 | 无（任意玩家可用） | 快速购买止痛药 | `rpg.sp:1128` |
| `sm_ping` | Network Quality Hint | 无参数 | 无（任意玩家可用） | sm_net 的别名 | `network_quality_hint.sp:118` |
| `sm_primary` | [L4D & 2] Pick-up Changes | 无参数 | 无（任意玩家可用） | 切换拾取物品后是否切换到主武器 | `l4d2_pickup.sp:219` |
| `sm_print_cvars_l4d2_scripted_hud` | l4d2_scripted_hud.sp（myinfo 缺 name，用文件名代替） | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 把本插件的 ConVar 及其取值打印到控制台 | `l4d2_scripted_hud.sp:897` |
| `sm_print_cvars_l4d_random_beam_item` | l4d_random_beam_item.sp（myinfo 缺 name，用文件名代替） | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 把本插件的 ConVar 及其取值打印到控制台 | `l4d_random_beam_item.sp:370` |
| `sm_printcasters` | L4D2 Caster System (Original built in readyup) | 无参数 | 需要 ADMFLAG_BAN（d，封禁） | 打印解说白名单 | `caster_system.sp:62` |
| `sm_pum` | 商店插件 | 无参数 | 无（任意玩家可用） | 快速购买一把一代单发霰弹枪 Pump | `rpg.sp:1125` |
| `sm_punch` | [L4D2] Punch Angle (RPG-aware, recoil command) | [0 或 1] | 无（任意玩家可用） | 切换自己的后坐力设置，sm_recoil 的别名 | `punch_angle.sp:78` |
| `sm_qf` | Anne Global Chat | <内容> | 无（任意玩家可用） | 发送一条全服聊天消息 | `global_chat.sp:189` |
| `sm_qfadmin` | Anne Global Chat | 无参数 | 无（任意玩家可用） | 打开全服聊天接收设置菜单，sm_qfmenu 的别名 | `global_chat.sp:196` |
| `sm_qfmenu` | Anne Global Chat | 无参数 | 无（任意玩家可用） | 打开全服聊天接收设置菜单 | `global_chat.sp:195` |
| `sm_quanfu` | Anne Global Chat | <内容> | 无（任意玩家可用） | 发送一条全服聊天消息，sm_qf 的别名 | `global_chat.sp:190` |
| `sm_r` | Pause plugin | 无参数 | 无（任意玩家可用） | sm_unpause 的别名 | `pause.sp:131` |
| `sm_r` | survivor_chat_select.sp（myinfo 缺 name，用文件名代替） | 无参数 | 无（任意玩家可用） | 把自己切换为 Rochelle，短别名 | `survivor_chat_select.sp:92` |
| `sm_rates` | RateMonitor | 无参数 | 无（任意玩家可用） | 列出所有在场玩家的网络设置 | `ratemonitor.sp:87` |
| `sm_ready` | L4D2 Saferoom Locker | 无参数 | 无（任意玩家可用） | 把自己设为已准备状态 | `l4d2_door_lock.sp:221` |
| `sm_ready` | Pause plugin | 无参数 | 无（任意玩家可用） | sm_unpause 的别名 | `pause.sp:130` |
| `sm_rebuildnavcache` | Direct InfectedSpawn (directed-nav + maxdist-fallback) | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 重建当前地图的 Nav 图与进度缓存 | `infected_control.sp:513` |
| `sm_rebuildnavcache` | Direct InfectedSpawn (fdxx-nav + buckets + maxdist-fallback) | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 强制重建当前地图的 Nav 分桶缓存 | `infected_control26-07.sp:319` |
| `sm_recoil` | [L4D2] Punch Angle (RPG-aware, recoil command) | [0 或 1] | 无（任意玩家可用） | 切换自己的后坐力设置 | `punch_angle.sp:77` |
| `sm_regsi` | L4D2 Antibaiter | 无参数 | 无（任意玩家可用） | 立即开始一回合 | `l4d2_antibaiter.sp:93` |
| `sm_rehash` | SourceBans++: Main Plugin | 无参数 | 服务器控制台命令，玩家无法使用 | 重新加载 SQL 管理员，服务器控制台命令 | `sbpp_main.sp:172` |
| `sm_reload_vpk` | l4d2_map_vote.sp（myinfo 缺 name，用文件名代替） | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 刷新 VPK 与战役列表 | `l4d2_map_vote.sp:145` |
| `sm_reloadtags` | HexTags Lite | 无参数 | 需要 ADMFLAG_GENERIC（b，通用管理员） | 重新加载称号配置，HexTags Lite 版本 | `hextags_lite.sp:59` |
| `sm_reloadtags` | hextags | 无参数 | 需要 ADMFLAG_GENERIC（b，通用管理员） | 重新加载 HexTags 配置 | `hextags.sp:146` |
| `sm_remove_caster_id` | L4D2 Caster System (Original built in readyup) | <SteamID> | 需要 ADMFLAG_BAN（d，封禁） | 把解说从白名单移除 | `caster_system.sp:61` |
| `sm_report` | SourceBans++ Report Plugin | 无参数 | 无（任意玩家可用） | 打开玩家举报菜单 | `sbpp_report.sp:48` |
| `sm_reset_ghost_hurt` | Ghost Hurt Management | 无参数 | 服务器控制台命令，玩家无法使用 | 重置 trigger_hurt_ghost，适合放在 confogl_off.cfg 中，服务器控制台命令 | `ghost_hurt.sp:30` |
| `sm_resetcasters` | L4D2 Caster System (Original built in readyup) | 无参数 | 需要 ADMFLAG_BAN（d，封禁） | 重置解说名单，适合放在 confogl_off.cfg 中 | `caster_system.sp:59` |
| `sm_resetmatch` | Confogl's Competitive Mod | 无参数 | 需要 ADMFLAG_CONFIG（h，服务器配置） | 强制关闭比赛模式，无论此前是强制还是常开 | `ReqMatch.sp:66` |
| `sm_resetstringcount` | Add Text To Readyup Panel | 无参数 | 服务器控制台命令，玩家无法使用 | 重置文本计数，服务器控制台命令 | `panel_text.sp:32` |
| `sm_restart` | L4D2 Auto restart | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 手动重启服务器 | `linux_auto_restart.sp:40` |
| `sm_restartmap` | simple join | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 立即重启地图 | `join.sp:135` |
| `sm_rmatch` | Match Vote | 无参数 | 无（任意玩家可用） | 发起重置比赛模式的投票 | `match_vote.sp:72` |
| `sm_rochelle` | survivor_chat_select.sp（myinfo 缺 name，用文件名代替） | 无参数 | 无（任意玩家可用） | 把自己使用的生还者切换为 Rochelle | `survivor_chat_select.sp:83` |
| `sm_roll` | Coinflip | <最小> <最大> | 无（任意玩家可用） | 在指定范围内随机掷点数 | `coinflip.sp:48` |
| `sm_rpg` | 商店插件 | 无参数 | 无（任意玩家可用） | 打开购买菜单，sm_buy 的别名 | `rpg.sp:1132` |
| `sm_rpgglowdebug` | 商店插件 | <玩家> | 需要 ADMFLAG_ROOT（z，最高权限） | 输出指定玩家的 RPG 轮廓权限调试信息 | `rpg.sp:1134` |
| `sm_rpginfo` | 商店插件 | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 把 RPG 人物信息输出到控制台，含近战、血包、轮廓、帽子、皮肤与后坐力 | `rpg.sp:1133` |
| `sm_rygive` | rygive.sp（myinfo 缺 name，用文件名代替） | 无参数 | 需要 ADMFLAG_CHAT（j，聊天管理） | 打开 rygive 给予物品菜单 | `rygive.sp:254` |
| `sm_s` | Pause plugin | 无参数 | 无（任意玩家可用） | sm_spectate 的别名 | `pause.sp:126` |
| `sm_s` | Player Management Plugin | 无参数 | 无（任意玩家可用） | sm_spectate 的别名 | `playermanagement.sp:73` |
| `sm_s` | simple join | 无参数 | 无（任意玩家可用） | sm_away 的短别名 | `join.sp:118` |
| `sm_secondary` | [L4D & 2] Pick-up Changes | 无参数 | 无（任意玩家可用） | 切换拾取物品后是否切换到副武器 | `l4d2_pickup.sp:220` |
| `sm_set_specspeed_increment` | Caster Assister | <增量> | 无（任意玩家可用） | 设置旁观速度的递增步长，默认 0.1 | `caster_assister.sp:28` |
| `sm_set_specspeed_multi` | Caster Assister | <倍数> | 无（任意玩家可用） | 设置旁观速度倍率，默认 1.0 | `caster_assister.sp:27` |
| `sm_set_vertical_increment` | Caster Assister | <增量> | 无（任意玩家可用） | 设置垂直视角递增步长，默认 450.0 | `caster_assister.sp:31` |
| `sm_setbot` | AnneServer Server Function (quiet minimal) | 无参数 | 无（任意玩家可用） | 在随机生还者附近生成一个生还者 Bot，源码未给描述 | `server.sp:111` |
| `sm_setch` | 商店插件 | <称号文本> | 无（任意玩家可用） | 设置自定义称号，需积分不低于 50 万 | `rpg.sp:1129` |
| `sm_setdance` | SM Fortnite Emotes Extended | <目标玩家> [舞蹈 ID] | 需要 ADMFLAG_GENERIC（b，通用管理员） | 给目标玩家设置舞蹈，sm_setdances 的别名 | `fornite_l4d.sp:100` |
| `sm_setdances` | SM Fortnite Emotes Extended | <目标玩家> [舞蹈 ID] | 需要 ADMFLAG_GENERIC（b，通用管理员） | 给目标玩家设置舞蹈 | `fornite_l4d.sp:99` |
| `sm_setemote` | SM Fortnite Emotes Extended | <目标玩家> [表情 ID] | 需要 ADMFLAG_GENERIC（b，通用管理员） | 给目标玩家设置表情，sm_setemotes 的别名 | `fornite_l4d.sp:98` |
| `sm_setemotes` | SM Fortnite Emotes Extended | <目标玩家> [表情 ID] | 需要 ADMFLAG_GENERIC（b，通用管理员） | 给目标玩家设置表情，用法 sm_setemotes <目标玩家> [表情 ID] | `fornite_l4d.sp:97` |
| `sm_setnext` | map_changer.sp（myinfo 缺 name，用文件名代替） | <章节地图代码> | 需要 ADMFLAG_RCON（i，RCON 命令） | 设置下一张地图 | `map_changer.sp:159` |
| `sm_setscores` | SetScores | <生还者分数> <特感分数> | 无（任意玩家可用） | 修改分数，需处于准备阶段且回合尚未开始 | `l4d2_setscores.sp:62` |
| `sm_show` | Pause plugin | 无参数 | 无（任意玩家可用） | 隐藏暂停面板以便查看其他菜单 | `pause.sp:141` |
| `sm_show_lagcomp_list` | L4D2 Lag Compensation Manager | 无参数 | 无（任意玩家可用） | 把延迟补偿数组打印到服务器控制台 | `l4d2_lagcomp_manager.sp:173` |
| `sm_sivote` | Anne Spawn Vote Menu | 无参数 | 无（任意玩家可用） | 打开刷特投票菜单，sm_spawnvote 的别名 | `spawn_vote_menu.sp:108` |
| `sm_skeets` | 1v1 SkeetStats | 无参数 | 无（任意玩家可用） | 输出当前 skeetstats | `1v1_skeetstats.sp:269` |
| `sm_skill` | Player Statistics | [参数] | 无（任意玩家可用） | 输出生还者特殊技能统计 | `l4d2_playstats.sp:571` |
| `sm_sleuth_reloadlist` | SourceBans++: SourceSleuth | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 重新加载 sleuth 排除列表，源码未给描述 | `sbpp_sleuth.sp:85` |
| `sm_slots` | Slots?! Voter | <槽位数> | 无（任意玩家可用） | 发起修改服务器槽位数的投票 | `slots_vote.sp:29` |
| `sm_smg` | 商店插件 | 无参数 | 无（任意玩家可用） | 快速购买 SMG | `rpg.sp:1126` |
| `sm_snd` | L4D2 Hit/Kill Feedback Plus | 无参数 | 无（任意玩家可用） | 打开命中与击杀反馈主菜单，含音效、图标与范围设置 | `l4d2_hitsound.sp:773` |
| `sm_spawnpreset_delete` | Anne Spawn Vote Menu | <名字> | 需要 ADMFLAG_CONFIG（h，服务器配置） | 删除指定的刷特预设 | `spawn_vote_menu.sp:111` |
| `sm_spawnpreset_list` | Anne Spawn Vote Menu | 无参数 | 无（任意玩家可用） | 列出当前模式的刷特预设 | `spawn_vote_menu.sp:112` |
| `sm_spawnpreset_save` | Anne Spawn Vote Menu | <名字> | 需要 ADMFLAG_CONFIG（h，服务器配置） | 把当前刷特设置保存为预设 | `spawn_vote_menu.sp:110` |
| `sm_spawnvote` | Anne Spawn Vote Menu | 无参数 | 无（任意玩家可用） | 打开刷特投票菜单 | `spawn_vote_menu.sp:107` |
| `sm_spec` | Pause plugin | 无参数 | 无（任意玩家可用） | sm_spectate 的别名 | `pause.sp:125` |
| `sm_spec` | Player Management Plugin | 无参数 | 无（任意玩家可用） | sm_spectate 的别名 | `playermanagement.sp:72` |
| `sm_spec` | simple join | 无参数 | 无（任意玩家可用） | sm_away 的别名 | `join.sp:117` |
| `sm_spechud` | Hyper-V HUD Manager | 无参数 | 无（任意玩家可用） | 切换自己的 spechud | `spechud.sp:113` |
| `sm_spechudoff` | l4d2_scripted_hud.sp（myinfo 缺 name，用文件名代替） | 无参数 | 无（任意玩家可用） | 关闭 spechud | `l4d2_scripted_hud.sp:899` |
| `sm_spechudon` | l4d2_scripted_hud.sp（myinfo 缺 name，用文件名代替） | 无参数 | 无（任意玩家可用） | 打开 spechud | `l4d2_scripted_hud.sp:898` |
| `sm_specrates` | Lightweight Spectating (merged+128+force-spec) | 无参数 | 无（任意玩家可用） | 分数达到 30 万时手动设置旁观为 60 tick | `specrates.sp:121` |
| `sm_spectate` | Pause plugin | 无参数 | 无（任意玩家可用） | 把自己移到旁观者队伍 | `pause.sp:124` |
| `sm_spectate` | Player Management Plugin | 无参数 | 无（任意玩家可用） | 把自己移到旁观者队伍 | `playermanagement.sp:71` |
| `sm_srmedkit_apply` | l4d2_med_dynamic.sp（myinfo 缺 name，用文件名代替） | 无参数 | 需要 ADMFLAG_GENERIC（b，通用管理员） | 立即移除安全室医疗包并按生还者人数分配 | `l4d2_med_dynamic.sp:88` |
| `sm_srvcln_now` | Server Clean Up | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 立即执行一次清理，源码未给描述 | `servercleanup.sp:174` |
| `sm_startspawn` | Direct InfectedSpawn (directed-nav + maxdist-fallback) | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 管理员重置刷特时钟 | `infected_control.sp:511` |
| `sm_startspawn` | Direct InfectedSpawn (fdxx-nav + buckets + maxdist-fallback) | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 管理员重置刷特时钟 | `infected_control26-07.sp:317` |
| `sm_stats` | Player Statistics | [参数] | 无（任意玩家可用） | 输出生还者统计 | `l4d2_playstats.sp:569` |
| `sm_stats_auto` | Player Statistics | <0 或 1> | 无（任意玩家可用） | 设置客户端是否在回合结束时自动打印统计 | `l4d2_playstats.sp:575` |
| `sm_stopmixmap` | l4d2_mixmap | 无参数 | 无（任意玩家可用） | 发起中止 mixmap 的投票 | `l4d2_mixmap.sp:134` |
| `sm_stopspawn` | Direct InfectedSpawn (directed-nav + maxdist-fallback) | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 管理员停止刷特 | `infected_control.sp:512` |
| `sm_stopspawn` | Direct InfectedSpawn (fdxx-nav + buckets + maxdist-fallback) | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 管理员停止刷特 | `infected_control26-07.sp:318` |
| `sm_survivor` | simple join | 无参数 | 无（任意玩家可用） | 加入生还者队伍，sm_join 的别名 | `join.sp:128` |
| `sm_swap` | Player Management Plugin | <玩家...> | 需要 ADMFLAG_KICK（c，踢出） | 把列出的玩家全部换到对面队伍 | `playermanagement.sp:67` |
| `sm_swapteams` | Player Management Plugin | 无参数 | 需要 ADMFLAG_KICK（c，踢出） | 交换两队玩家 | `playermanagement.sp:69` |
| `sm_swapto` | Player Management Plugin | <队伍号> <玩家...> | 需要 ADMFLAG_KICK（c，踢出） | 把列出的玩家换到指定队伍，队伍号见命令提示 | `playermanagement.sp:68` |
| `sm_tagrank` | l4d2_mixmap | <标签> <地图编号> | 服务器控制台命令，玩家无法使用 | 设置标签对应的地图排序，服务器控制台命令 | `l4d2_mixmap.sp:128` |
| `sm_tagslist` | HexTags Lite | 无参数 | 无（任意玩家可用） | 打开称号选择菜单 | `hextags_lite.sp:60` |
| `sm_tagslist` | hextags | 无参数 | 无（任意玩家可用） | 打开称号选择菜单 | `hextags.sp:150` |
| `sm_tank` | L4D2 Tank Control | 无参数 | 无（任意玩家可用） | 显示谁将成为坦克 | `l4d_tank_control_eq.sp:84` |
| `sm_tank` | [L4D2] Boss Percents/Vote Boss Hybrid | 无参数 | 无（任意玩家可用） | 显示 Tank 刷新百分比 | `l4d_boss_percent.sp:106` |
| `sm_tank_witch_debug_info` | Tank and Witch ifier! | 无参数 | 需要 ADMFLAG_KICK（c，踢出） | 输出刷新状态调试信息 | `witch_and_tankifier.sp:91` |
| `sm_tank_witch_debug_profiler` | Tank and Witch ifier! | <次数> | 无（任意玩家可用） | 按指定次数运行 AdjustBossFlow 性能分析 | `witch_and_tankifier.sp:95` |
| `sm_tank_witch_debug_test` | Tank and Witch ifier! | 无参数 | 无（任意玩家可用） | 测试命令，启动 AdjustBossFlow 计时器 | `witch_and_tankifier.sp:94` |
| `sm_tankhud` | Hyper-V HUD Manager | 无参数 | 无（任意玩家可用） | 切换自己的坦克 HUD | `spechud.sp:114` |
| `sm_tankshuffle` | L4D2 Tank Control | 无参数 | 需要 ADMFLAG_SLAY（f，处死） | 重新随机挑选一名玩家成为坦克 | `l4d_tank_control_eq.sp:80` |
| `sm_targetlimit_status` | SI target limit | 无参数 | 需要 ADMFLAG_GENERIC（b，通用管理员） | 显示每名生还者的特感目标数量与上限 | `SI_Target_limit.sp:162` |
| `sm_team2` | simple join | 无参数 | 无（任意玩家可用） | 加入生还者队伍，sm_join 的别名 | `join.sp:126` |
| `sm_team3` | simple join | 无参数 | 无（任意玩家可用） | 加入特感队伍，sm_joininfected 的别名 | `join.sp:120` |
| `sm_teamflip` | Teamflip | 无参数 | 无（任意玩家可用） | 发起换边即两队互换 | `teamflip.sp:51` |
| `sm_tf` | Teamflip | 无参数 | 无（任意玩家可用） | sm_teamflip 的别名 | `teamflip.sp:52` |
| `sm_third` | Anne Thirdperson Shoulder Fix | 无参数 | 无（任意玩家可用） | sm_tp 的别名 | `l4d2_anne_thirdperson_fix.sp:68` |
| `sm_thirdperson` | Anne Thirdperson Shoulder Fix | 无参数 | 无（任意玩家可用） | sm_tp 的别名 | `l4d2_anne_thirdperson_fix.sp:69` |
| `sm_time` | VeteransOnly | 无参数 | 无（任意玩家可用） | 显示自己的游戏时长 | `veterans.sp:211` |
| `sm_timeall` | VeteransOnly | 无参数 | 需要 ADMFLAG_GENERIC（b，通用管理员） | 显示所有玩家的游戏时长 | `veterans.sp:210` |
| `sm_to_reload` | [L4D & L4D2] Target Override | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 重新加载数据配置 | `l4d_target_override.sp:491` |
| `sm_to_stats` | [L4D & L4D2] Target Override | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 显示目标选择函数的性能统计，含最小、平均与最大耗时 | `l4d_target_override.sp:494` |
| `sm_toggleready` | Pause plugin | 无参数 | 无（任意玩家可用） | 切换自己队伍的准备状态 | `pause.sp:134` |
| `sm_toggletags` | HexTags Lite | 无参数 | 无（任意玩家可用） | 切换自己的称号显示 | `hextags_lite.sp:63` |
| `sm_toggletags` | hextags | 无参数 | 需要 ADMFLAG_GENERIC（b，通用管理员） | 切换自己的称号是否可见 | `hextags.sp:147` |
| `sm_tp` | Anne Thirdperson Shoulder Fix | 无参数 | 无（任意玩家可用） | 切换自己的 Anne 第三人称肩部视角 | `l4d2_anne_thirdperson_fix.sp:67` |
| `sm_unban` | SourceBans++: Main Plugin | <steamid 或 IP> [原因] | 需要 ADMFLAG_UNBAN（e，解封） | 解除封禁 | `sbpp_main.sp:176` |
| `sm_unblock` | l4d2_blacklist.sp（myinfo 缺 name，用文件名代替） | <目标玩家或 steam64 或名字> | 无（任意玩家可用） | 把目标玩家从黑名单移除 | `l4d2_blacklist.sp:237` |
| `sm_uncast` | L4D2 Caster System (Original built in readyup) | [玩家] | 无（任意玩家可用） | sm_notcasting 的别名 | `caster_system.sp:65` |
| `sm_uncinfblock_check` | Uncommon Infected Blocker | 无参数 | 需要 ADMFLAG_GENERIC（b，通用管理员） | 检查非常见感染者屏蔽状态 | `l4d2_uncommon_blocker.sp:121` |
| `sm_underhand` | Tank Attack Control | 无参数 | 无（任意玩家可用） | 切换坦克为下手投掷石头，仅坦克可用 | `l4d2_tank_attack_control.sp:60` |
| `sm_unlock` | L4D2 Saferoom Locker | 无参数 | 需要 ADMFLAG_GENERIC（b，通用管理员） | 强制解锁安全室的门 | `l4d2_door_lock.sp:220` |
| `sm_unpause` | Pause plugin | 无参数 | 无（任意玩家可用） | 把自己队伍标记为准备以便取消暂停 | `pause.sp:129` |
| `sm_unready` | L4D2 Saferoom Locker | 无参数 | 无（任意玩家可用） | 把自己设为未准备状态 | `l4d2_door_lock.sp:222` |
| `sm_unready` | Pause plugin | 无参数 | 无（任意玩家可用） | 把自己队伍标记为未准备 | `pause.sp:132` |
| `sm_unsetch` | 商店插件 | 无参数 | 无（任意玩家可用） | 取消自定义称号，需积分不低于 50 万 | `rpg.sp:1130` |
| `sm_update_vpk` | l4d2_map_vote.sp（myinfo 缺 name，用文件名代替） | 无参数 | 需要 ADMFLAG_ROOT（z，最高权限） | 刷新 VPK 与战役列表，sm_reload_vpk 的别名 | `l4d2_map_vote.sp:146` |
| `sm_updater_check` | Updater | 无参数 | 需要 ADMFLAG_RCON（i，RCON 命令） | 强制检查更新，每小时只能检查一次 | `updater.sp:112` |
| `sm_updater_forcecheck` | Updater | 无参数 | 需要 ADMFLAG_RCON（i，RCON 命令） | 无限制地强制检查更新 | `updater.sp:113` |
| `sm_updater_status` | Updater | 无参数 | 需要 ADMFLAG_RCON（i，RCON 命令） | 查看 Updater 的状态 | `updater.sp:114` |
| `sm_uzi` | 商店插件 | 无参数 | 无（任意玩家可用） | 快速购买乌兹 | `rpg.sp:1127` |
| `sm_v3` | l4d2_map_vote.sp（myinfo 缺 name，用文件名代替） | 无参数 | 无（任意玩家可用） | 打开换图投票菜单，旁观者不可投票 | `l4d2_map_vote.sp:135` |
| `sm_veterans_exclude` | VeteransOnly | 无参数 | 需要 ADMFLAG_GENERIC（b，通用管理员） | 把指定用户排除出 VeteransOnly 检查 | `veterans.sp:207` |
| `sm_veterans_include` | VeteransOnly | 无参数 | 需要 ADMFLAG_GENERIC（b，通用管理员） | 把已被排除的用户重新纳入 VeteransOnly 检查 | `veterans.sp:208` |
| `sm_vote` | Vote for run command or cfg file | <cfg 名> | 无（任意玩家可用） | 发起投票执行指定的 cfg 或命令 | `vote.sp:80` |
| `sm_voteban` | Vote for run command or cfg file | 无参数 | 无（任意玩家可用） | 打开投票封禁菜单 | `vote.sp:82` |
| `sm_voteboss` | [L4D2] Vote Boss | <是否坦克 0或1> <百分比> | 无（任意玩家可用） | 发起坦克或女巫刷新百分比投票 | `l4d_boss_vote.sp:53` |
| `sm_voteboss` | [L4D2] Vote Boss | <是否坦克 0或1> <百分比> | 无（任意玩家可用） | 发起坦克或女巫刷新百分比投票 | `l4d_boss_vote.sp:46` |
| `sm_votekick` | Vote for run command or cfg file | 无参数 | 无（任意玩家可用） | 打开投票踢人菜单 | `vote.sp:81` |
| `sm_votemap` | l4d2_map_vote.sp（myinfo 缺 name，用文件名代替） | 无参数 | 无（任意玩家可用） | 打开换图投票菜单，sm_v3 的别名 | `l4d2_map_vote.sp:139` |
| `sm_votenext` | l4d2_map_vote.sp（myinfo 缺 name，用文件名代替） | 无参数 | 无（任意玩家可用） | 对下一张地图发起投票，sm_mapnext 的别名 | `l4d2_map_vote.sp:143` |
| `sm_vpk_reload` | L4D2 Admin Mission Menu | 无参数 | 需要 ADMFLAG_RCON（i，RCON 命令） | 重新加载 VPK 与任务列表，并执行 update_addon_paths 与 mission_reload | `adminmenu_mission_list.sp:112` |
| `sm_warp` | Infected Warp | <参数> | 无（任意玩家可用） | sm_warptosurvivor 的别名 | `l4d2_ghost_warp.sp:76` |
| `sm_warpto` | Infected Warp | <参数> | 无（任意玩家可用） | sm_warptosurvivor 的别名 | `l4d2_ghost_warp.sp:75` |
| `sm_warptosurvivor` | Confogl's Competitive Mod | <生还者序号> | 无（任意玩家可用） | 把自己从幽灵状态传送到指定序号的生还者 | `GhostWarp.sp:33` |
| `sm_warptosurvivor` | Infected Warp | <参数> | 无（任意玩家可用） | 把自己传送到随机生还者处即幽灵传送，服务器上不可用 | `l4d2_ghost_warp.sp:74` |
| `sm_wavestatus` | Direct InfectedSpawn (directed-nav + maxdist-fallback) | 无参数 | 需要 ADMFLAG_GENERIC（b，通用管理员） | 查看当前波决策器状态 | `infected_control.sp:518` |
| `sm_wavestatus` | Direct InfectedSpawn (fdxx-nav + buckets + maxdist-fallback) | 无参数 | 需要 ADMFLAG_GENERIC（b，通用管理员） | 查看当前波决策器状态 | `infected_control26-07.sp:324` |
| `sm_weapon` | L4D2 Weapon Attributes | <武器> <属性> <数值> | 服务器控制台命令，玩家无法使用 | 设置近战武器属性，服务器控制台命令 | `l4d2_weapon_attributes.sp:251` |
| `sm_weapon_attributes` | L4D2 Weapon Attributes | [武器] | 无（任意玩家可用） | 显示武器属性 | `l4d2_weapon_attributes.sp:255` |
| `sm_weapon_attributes_reset` | L4D2 Weapon Attributes | 无参数 | 服务器控制台命令，玩家无法使用 | 重置所有近战武器属性，服务器控制台命令 | `l4d2_weapon_attributes.sp:252` |
| `sm_weaponstats` | L4D2 Weapon Attributes | [武器] | 无（任意玩家可用） | 显示武器统计 | `l4d2_weapon_attributes.sp:254` |
| `sm_web` | simple join | 无参数 | 无（任意玩家可用） | 打开按客户端语言本地化的 MOTD 页面 | `join.sp:133` |
| `sm_witch` | L4D2 Tank Control | 无参数 | 无（任意玩家可用） | sm_tank 的别名 | `l4d_tank_control_eq.sp:86` |
| `sm_witch` | [L4D2] Boss Percents/Vote Boss Hybrid | 无参数 | 无（任意玩家可用） | 显示 Witch 刷新百分比 | `l4d_boss_percent.sp:107` |
| `sm_xx` | text.sp（myinfo 缺 name，用文件名代替） | 无参数 | 无（任意玩家可用） | 源码未给描述；据 `optional/AnneHappy/text.sp:396-400` 回调 `InfectedStatus` 只调用 `printinfo(Client)`，:180-195 的 `printinfo` 再对该玩家调用 `PrintInfoToClient`，后者（:207-280）逐行输出坦克 Bhop 开关（:223-224）、当前武器配置档位（:226-228）、AI 难度、特感上限与刷新间隔（:240）以及刷怪距离/传送检查/回血/坦克消耗等 ConVar（:242-274），推测为：查询本局特感与插件配置状态的玩家命令（`sm_xx` 命名随意，实质等同 `sm_info`；`event_RoundStart` 开局用同一函数全服播报，:401-404）（置信度：高） | `text.sp:58` |
| `sm_z` | survivor_chat_select.sp（myinfo 缺 name，用文件名代替） | 无参数 | 无（任意玩家可用） | 把自己切换为 Zoey，sm_zoey 的短别名 | `survivor_chat_select.sp:88` |
| `sm_zd` | Anne Global Chat | <内容> | 无（任意玩家可用） | 发送找队友信息到全服，受每日次数限制 | `global_chat.sp:192` |
| `sm_zdmenu` | Anne Global Chat | 无参数 | 无（任意玩家可用） | 打开找队友提示接收设置菜单 | `global_chat.sp:197` |
| `sm_zdy` | Anne Global Chat | <内容> | 无（任意玩家可用） | 发送找队友信息，sm_zd 的别名 | `global_chat.sp:194` |
| `sm_zoey` | survivor_chat_select.sp（myinfo 缺 name，用文件名代替） | 无参数 | 无（任意玩家可用） | 把自己使用的生还者切换为 Zoey | `survivor_chat_select.sp:79` |
| `sm_zombie` | simple join | 无参数 | 无（任意玩家可用） | 加入特感队伍，sm_joininfected 的别名 | `join.sp:123` |
| `sm_zs` | AnneServer Server Function (quiet minimal) | 无参数 | 无（任意玩家可用） | 自杀，源码未给描述 | `server.sp:116` |
| `sm_zudui` | Anne Global Chat | <内容> | 无（任意玩家可用） | 发送找队友信息，sm_zd 的别名 | `global_chat.sp:193` |
| `sm刷特` | Anne Spawn Vote Menu | 无参数 | 无（任意玩家可用） | 打开刷特投票菜单，中文别名 | `spawn_vote_menu.sp:109` |
| `spit_block_square` | L4D2 Spit Blocker | <参数> | 服务器控制台命令，玩家无法使用 | 设置毒痰阻挡方块，服务器控制台命令 | `l4d2_spitblock.sp:46` |
| `spit_remove_block_square` | L4D2 Spit Blocker | <参数> | 服务器控制台命令，玩家无法使用 | 移除毒痰阻挡方块，服务器控制台命令 | `l4d2_spitblock.sp:47` |
| `spit_spread_saferoom_except` | [L4D2] Spit Spread Patch | <地图名> | 服务器控制台命令，玩家无法使用 | 把指定地图排除在毒痰扩散修正之外，服务器控制台命令 | `l4d2_spit_spread_patch.sp:202` |
| `ssb_custom_path` | L4D2 Various Sounds Blocker | <路径> | 服务器控制台命令，玩家无法使用 | 设置自定义音效路径，服务器控制台命令 | `l4d2_sounds_blocker.sp:87` |
| `ssb_whitelist_path` | L4D2 Various Sounds Blocker | <路径> | 服务器控制台命令，玩家无法使用 | 设置白名单路径，服务器控制台命令 | `l4d2_sounds_blocker.sp:88` |
| `static_tank_map` | Tank and Witch ifier! | <地图名> | 服务器控制台命令，玩家无法使用 | 把地图加入静态坦克地图列表，服务器控制台命令 | `witch_and_tankifier.sp:87` |
| `static_witch_map` | Tank and Witch ifier! | <地图名> | 服务器控制台命令，玩家无法使用 | 把地图加入静态女巫地图列表，服务器控制台命令 | `witch_and_tankifier.sp:88` |
| `statsreset` | Player Statistics | 无参数 | 需要 ADMFLAG_CHANGEMAP（g，换图） | 重置统计数据，仅管理员可用 | `l4d2_playstats.sp:577` |
| `tank_map_flow_and_second_event` | EQ2 Finale Tank Manager | <地图名> | 服务器控制台命令，玩家无法使用 | 标记该地图坦克出现于第二事件，服务器控制台命令 | `eq_finale_tanks.sp:71` |
| `tank_map_flow_and_second_event` | Hyper-V HUD Manager | <地图名> | 服务器控制台命令，玩家无法使用 | 标记地图坦克出现于第二事件，服务器控制台命令 | `spechud.sp:293` |
| `tank_map_only_first_event` | EQ2 Finale Tank Manager | <地图名> | 服务器控制台命令，玩家无法使用 | 标记该地图坦克只出现于第一事件，服务器控制台命令 | `eq_finale_tanks.sp:72` |
| `tank_map_only_first_event` | Hyper-V HUD Manager | <地图名> | 服务器控制台命令，玩家无法使用 | 标记地图坦克只出现于第一事件，服务器控制台命令 | `spechud.sp:294` |
