# 登录持久化修复 & acelite 轻应用体系（2026-10-28）

本文记录两件事的排查与实现依据：

1. 「集成频繁掉登录」的根因与修复
2. acelite 轻应用（插件）体系的完整逆向与实现

所有结论均来自**插件反编译 + 本机（AS6020-02）真机实测**。

---

## 一、登录持久化：两个互相掩盖的 bug

### Bug A（运行期）—— 集成每分钟自我拆装一次

链条（每一环都在代码里核实过）：

```
① 每分钟轮询 → async_ensure_access_token()
      账号密码模式：无条件调 token_provider()      ← 没有缓存（设备码模式有）
② → async_refresh_session()  没有过期判断，每次都真发续期请求
③ → 续期成功 → 写 store + hass.config_entries.async_update_entry(entry.data)
④ → HA：entry 任何字段变化都触发 add_update_listener
⑤ → _async_update_listener → async_reload(entry.entry_id)   ← 无条件重载
⑥ → 集成被拆掉重装 → 回到 ①
```

后果：实体反复 unavailable、MQTT 反复断连；重载期间设备侧容易 401，
`coordinator.py` 把 `HuaweiDeviceAuthError` 转成 `ConfigEntryAuthFailed`，
**HA 弹出「需要重新认证」**。

### Bug B（重启后）—— 从不主动续期

`rebuild_session()` 里 `oauth_expires_at` 被硬编码成 `None`：

```python
oauth_expires_at=None,   # ← 每次重启都这样
```

而 `AuthSession.is_oauth_valid()` 对 `None` 的处理是「视为有效」：

```python
if self.oauth_expires_at is None:
    return True          # ← 于是一辈子不续期
```

结果：重启后令牌真过期时直接 401 → 同样弹「重新认证」。

> 补充：`is_oauth_valid()` 这个方法**在修复前是全项目 0 处调用的死代码** ——
> 能力早就写好了，只是没接进调用链。

### 修复（4 处）

| 文件 | 改动 |
|---|---|
| `api/huawei_account.py` | `session_to_dict()` 持久化 `oauth_expires_at`（ISO 字符串）|
| `api/huawei_account.py` | `rebuild_session()` 还原它（不再写死 None）|
| `api/huawei_account.py` | `async_refresh_session()` 加 `is_oauth_valid(300)` 判断 |
| `api/huawei_cloud.py` | 账号密码模式加 30 分钟 TTL 缓存 + 锁；取凭据失败清缓存重试一次 |
| `__init__.py` | update listener 只对 `options` 变化 reload |

迁移期保护：旧快照里 `expires_at` 为 `None` 时**不当有效**，先续期一次拿到准确时间。

### 实测对比

| 配置 | 令牌续期频率 | reload |
|---|---|---|
| 修复前（TTL=0 模拟）| **每分钟 1 次** | — |
| 修复后（TTL=1800）| **0 次 / 2 分钟**（命中缓存）| **0** |

---

## 二、acelite 轻应用体系

此前只用了 `/acelite/installed_infos`（已装列表）。插件 `PluginApi.smali`
里其实有 **12 个 acelite 端点**。

### 端点与请求体

| 路径 | 方法 | 请求体 |
|---|---|---|
| `/acelite/market_infos` | GET | —（返回**压缩**，见下）|
| `/acelite/installed_infos` | GET | — |
| `/acelite/all_infos` | GET | —（已装 + 安装中 + status）|
| `/acelite/xunlei_infos` | GET | —（nasId + JWT）|
| `/acelite/hap_add` | POST | `PluginAddRequestBean` |
| `/acelite/hap_confirm` | POST | `PluginConfirmPermRequestBean` |
| `/acelite/hap_delete` / `hap_cancel` / `hap_pause` | POST | `PluginDeleteRequestBean` |
| `/acelite/hap_resume` | POST | `PluginAddRequestBean` |
| `/acelite/autoupg` | GET/POST | `PluginAutogradeRquesetBean` |

```
PluginAddRequestBean:         appId, downloadURL, iconURL, name, pkgName,
                              uiURL, versionCode(int), versionName
PluginConfirmPermRequestBean: appId, confirm(int), perms, pkgName, version
PluginDeleteRequestBean:      appId, pkgName
PluginAutogradeRquesetBean:   hdcPlugin(int)
```

### 关键坑 1：`market_infos` 返回压缩串

设备返回 `{"data": {"hapInfos": "eNrd..."}}` —— `eNrd` 是
**base64 → zlib → JSON** 的特征。解压后：

```json
{"sign": "...", "allList": [
  {"appId": "C979782581979458432", "pkgName": "com.xunlei.download",
   "name": "迅雷", "versionName": "3.4.2", "versionCode": 3004002,
   "size": 16442285, "iconUrl": "...", "downloadUrl": "..."}
]}
```

### 关键坑 2：字段名大小写不一致

市场列表用 `iconUrl` / `downloadUrl`（小写 url），而 `hap_add` 的
`PluginAddRequestBean` 用 `iconURL` / `downloadURL`（大写 URL）。
不做映射，安装请求会缺字段。

### 本机实测

| 能力 | 结果 |
|---|---|
| `market_infos` | ✅ 解出 3 个：迅雷 3.4.2 / 玩物下载 2.3.11 / 百度网盘 3.03.18 |
| `all_infos` | ✅ 3 个已装（`presetStatus=0`、`status=1`）|
| `xunlei_infos` | ✅ nasId + JWT |
| **卸载 → 安装 全流程** | ✅ **真机验证通过** |
| `retryTask` | ✅ 端点+报文验证（详见下）|

#### 卸载 → 安装验证细节

```
基线          3 个：迅雷 / 百度网盘 / 玩物下载
hap_delete    卸载玩物下载 → code=0
复查          installedHap 真的只剩 2 个
hap_add       仅给 app_id → code=0
复查          玩物下载回到列表，status=1，installTime 更新为当时时间
```

→ `plugin_install` / `plugin_uninstall` 是**真实可用**的，不是只回成功码。

### 能否安装「自己开发的」程序？—— 不行

4 组对照实验（`hap_add` 各字段分别替换）：

| # | appId / pkgName | downloadURL | uiURL | 结果 |
|---|---|---|---|---|
| 1 | 假 `com.test.probe` | 自建 HTTP | — | `code=5` |
| 2 | 假 | **官方 URL** | — | `code=5` |
| 3 | 官方玩物下载 | **自建 HTTP** | — | `ok:true`，但设备**未发起下载**（自建服务日志 0 请求）|
| 4 | 官方 | 自建 HTTP | **自建页面** | `ok:true`，但 **uiURL 未变更** |

**结论**：

1. 唯一硬校验是 **`appId` + `pkgName` 在华为侧注册**。不在注册表 → `code=5`，
   与 URL 域名无关（实验 2 用官方 URL 配假 appId 同样失败）。
2. 注册通过后，服务端**忽略客户端传入的 `downloadURL` / `uiURL` / `name`**，
   按 `appId` 从云端下发真实配置。
3. 因此无法通过冒用已注册 appId 的方式安装自定义内容。

**为什么这样设计（合理）**：这些「插件」不是设备上的本地程序，而是
**webview 入口**（`uiURL` 实测：迅雷 `pan.xunlei.com/yc/huawei/`、
百度网盘 `smarthome.hicloud.com/device/guide/plugins/nas_netdisk/`、
玩物下载 `yc.ionewu.com/wap/`）。若客户端能指定 `uiURL`，任何人都能把钓鱼页
伪装成「百度网盘」骗取账号 —— 服务端锁死是必要的安全设计。

**要装自己的程序需要**：华为分配的 `appId` + 绑定的 `pkgName` + 云端可索引的包 +
https 网页入口，即通过华为智慧生活/家庭存储插件开放平台注册。没有个人开发者
自签名安装通道（不像 Android `adb install`）。

---

## 三、新增的服务（7 个）

```
retry_task        重试失败的传输任务（POST /{business}/retryTask）
dynamic_info      动态/时光轴（GET /gallery/getDynamicInfo）
plugin_market     市场可安装列表（自动解压，返回 {sign, allList}）
plugin_all        已装 + 安装中 + status
plugin_xunlei     迅雷插件 token
plugin_install    安装（只给 app_id，其余字段自动补齐；也可给 download_url 自定义来源）
plugin_uninstall  卸载/取消/暂停（action=delete/cancel/pause）
```

### `retryTask` 报文（插件反编译）

```java
@POST("/{business}/retryTask")
Call retryTask(@Path("business") String b, @Body TaskRetryRequestBean r);
```

`business` 取值：`filesvc` / `gallery` / `trans` / `safebox`。
`transId` / `isSupportSubTask=1` 在特定任务类型下才带（服务层会自动从
`task_status` 查 `transId` 补上）。

实测：taskId 不存在时返回 `code=1101` —— **不是**参数错误码 `1100`，
说明端点在、报文被接受，只是目标任务不存在/不可重试。

---

## 四、盘点了但本机不支持的端点

| 端点 | 返回 | 判定 |
|---|---|---|
| `/devmanage/diskBasicInfo` | `code=1`（GET 与 POST 都试）| 固件不支持 |
| `/devmanage/diskSmartInfo` | `code=1` | **App 根本不调用它**（只有路径常量，无 Retrofit 方法）|
| `/raid/getraidstatus` | `code=22` | 固件不支持 |
| `/gallery/getDynamicInfo` | `30101`（缺参）→ `code=1`（全参）| 固件不支持 |

> `diskBasicInfo` 已按 App 真实调用改成 **POST + DiskParam body**
> （`HardDriveManageActivity.H3()`：`new DiskParam(null)`），但设备仍返回 `code=1`。
