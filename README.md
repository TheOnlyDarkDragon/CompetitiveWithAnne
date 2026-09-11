# **L4D2 Competitive Rework**

> [!IMPORTANT]
> It is recommended to host servers on Linux, but Windows is supported.  
> When running Linux ensure that your setup is running a minimum of **`GLIBC 2.35`** (Ubuntu 22.04 or higher) or you will run into issues loading certain extensions.  
> This repository only supports Sourcemod **1.12** and up (which comes with the repository for ease of use)

> [!重要]
> 推荐使用Linux服务器来运行，尽管Windows也是可以的。
> 如果你使用Linux运行服务器，确认你的GLIBC库版本不低于 GLIBC **`2.35`** (Ubuntu 22.04 或以上版本) ，否则你将一定会遇到 **`extensions`** 文件夹里面的拓展加载失败问题
> 这个仓库只支持 Sourcemod **`1.12`** 或以上版本(该工具随仓库提供，使用方便便捷)
---

> [!NOTE]
> ConVar **`mv_maxplayers`** was added which replaces **`sv_maxplayers`** in **`cfg/server.cfg`**, this is used to prevent it from being overwritten every map change.  
> On config unload, the value will be reset to the value used in the **`cfg/server.cfg`**.

> [!通知]
> 使用了ConVar **`mv_maxplayers`** 来替换 **`cfg/server.cfg`** 中的 ConVar **`sv_maxplayers`**，这是为了防止每次地图切换时最大人数被错误修改
> 在卸载比赛模式配置时，它的值被重设为 **`cfg/server.cfg`** 中设定的值

> [!NOTE]
> Every confogl matchmode will now execute 2 additional files; **`cfg/sharedplugins.cfg`** and **`cfg/generalfixes.cfg`**.  
> **`generalfixes.cfg`** contains all the crucial fixes that will be loaded in every matchmode.  
> **`sharedplugins.cfg`** is for you, the server owner. You can load any custom plugin that you want to be loaded in every matchmode here.

> [!通知]
> 现在每个比赛模式会运行两个额外的文件: **`cfg/sharedplugins.cfg`** 和 **`cfg/generalfixes.cfg`**
> **`generalfixes.cfg`** 包含了所有的关键性修复，并且会在所有的比赛模式中加载
> **`sharedplugins.cfg`** 是为了你 服务器的所有者 而创建的。你可以在里面加载自定义的插件，这个文件会在所有的比赛模式中加载

> [!CAUTION]
> Plugin load locking and unlocking is no longer handled by the configs themselves, refrain from doing it manually or you can run into issues.

> [!注意]
> 插件加载的锁定与解锁功能不再由配置文件本身来处理，请不要手动进行此类操作，否则可能会引发问题

## **Download & Installation:**

> [!IMPORTANT]
> Pick the archive that matches your **Server OS**:
> * **Linux:** `L4D2-Competitive-Rework-<version>-linux.tar.gz`
> * **Windows:** `L4D2-Competitive-Rework-<version>-windows.zip`

1. Download the latest archive from the [**Releases**](../../releases/latest) page.
2. Extract it directly into your server's **`left4dead2/`** directory.
3. For first-time server setup on dedicated servers, the [Dedicated Server Install Guide](Dedicated%20Server%20Install%20Guide/README.md) might be of use to you!

> [!NOTE]
> Releases only include what the servers need, **no** SourcePawn sources or compiler.  
> To modify or recompile plugins, clone the repository instead.

### Crash reporting (Accelerator)

> [!IMPORTANT]
> The checked-in [`addons/sourcemod/configs/core.cfg`](addons/sourcemod/configs/core.cfg) is configured for Anne's private servers. Its crash upload endpoint only accepts public IPs from Anne's private SourceBans-derived allowlist. The Anne URLs intentionally use HTTP because the bundled Accelerator/libcurl build cannot use HTTPS; do not reuse those private URLs.
>
> If you are deploying this repository on your own server, do **not** use Anne's private upload endpoints. Use Accelerator's official crash, symbol, and binary upload endpoints:
>
> ```text
> "MinidumpUrl" "http://crash.limetech.org/submit"
> "MinidumpSymbolUrl" "http://crash.limetech.org/symbols/submit"
> "MinidumpBinaryUrl" "http://crash.limetech.org/binary/submit"
> ```
>
> Configure `MinidumpAccount` for the account that should own the reports. The Anne-specific configuration uses the maintainer's SteamID64 because AnneWeb displays reports to the matching Steam account.

> [!重要]
> 已配置过的 [`addons/sourcemod/configs/core.cfg`](addons/sourcemod/configs/core.cfg) 文件适用于Anne的私有服务器。其崩溃报告上传端点仅接受来自Anne的私有SourceBans允许列表中的公共IP。Anne的URL故意使用HTTP协议，因为随仓库附带的Accelerator/libcurl构建版本无法使用HTTPS协议；请勿使用这些私有URL。
>
> 如果您要在自己的服务器上部署此仓库，请不要使用Anne的私有上传端点。请使用Accelerator的官方崩溃报告、符号文件和二进制文件上传端点：
> ```text
> "MinidumpUrl" "http://crash.limetech.org/submit"
> "MinidumpSymbolUrl" "http://crash.limetech.org/symbols/submit"
> "MinidumpBinaryUrl" "http://crash.limetech.org/binary/submit"
> ```
>
> 请为负责处理报告的账户配置MinidumpAccount。与Anne相关的配置需要使用维护者的SteamID64，因为AnneWeb会将报告显示给对应的Steam账户。


### Network quality reports

`network_quality_hint` samples ping/loss/choke locally. HTTPS reporting to NewAnneWeb is optional: the plugin ConVar default is **`nqh_report_enable "0"`**, so community copies keep local hints without uploading.

This repository sets **`nqh_report_enable "1"`** in [`cfg/sourcemod/network_quality_hint.cfg`](cfg/sourcemod/network_quality_hint.cfg) for Anne's website-whitelisted game servers. Reporting requires:

* `network_quality_hint` 1.1.5 or newer
* SteamWorks (`addons/sourcemod/extensions/SteamWorks.ext.so`)
* an explicit `nqh_report_enable "1"`

Website incidents focus on packet loss. Choke below `nqh_report_choke_limit` (default 20%) is not reported as an incident. Local chat warnings still use `nqh_choke_limit` (default 5%).

If you deploy this pack on a server that is **not** on the website whitelist, set `nqh_report_enable "0"`. Local detection still works; the website will not accept reports from unknown servers.

## **Credits:**

> **Foundation/Advanced Work:**

* A1m`
* AlliedModders LLC.
* "Confogl Team"
* Dr!fter
* Forgetest
* Jahze
* Lux
* Prodigysim
* Silvers
* XutaxKamay
* Visor

> **Additional Plugins/Extensions:**

* Accelerator74
* Arti
* AtomicStryker
* Backwards
* BHaType
* Blade
* Buster
* Canadarox
* CircleSquared
* Darkid
* DarkNoghri
* Dcx
* Devilesk
* Die Teetasse
* Disawar1
* Don
* Dragokas
* Dr. Gregory House
* Epilimic
* Estoopi
* Griffin
* Harry Potter
* Jacob
* Luckylock
* Madcap
* Mr. Zero
* Nielsen
* Powerlord
* Rena
* Sheo
* Sir
* Spoon
* Stabby
* Step
* Tabun
* Target
* TheTrick
* V10
* Vintik
* VoiDeD
* xoxo
* $atanic $pirit

> **Competitive Mapping Rework:**

* Aiden
* Derpduck
* Mart

> [!NOTE]
> If your work is being used and I forgot to credit you, don't hesitate to contact me on Discord (user: `sirplease`)
