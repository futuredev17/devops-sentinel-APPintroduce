# DevOps Sentinel · 架构与实现说明

> 版本：v1.0.0+10（V1.0 终极商业版）
> 状态：账号 / 本地库 / 设备 / 真实 SSH / 终端 / 告警 / 脚本 / 支付授权 / AI 排障 / 局域网大屏 / 应用锁 全部打通。

---

## 1. 技术选型

| 关注点 | 选型 |
| --- | --- |
| 状态管理 | flutter_riverpod 3.3（Notifier / AsyncNotifier，无 codegen） |
| 路由 | go_router 17（StatefulShellRoute + CustomTransitionPage + 登录守卫） |
| 本地库 | sqflite 2.4（SQLite，WAL，外键级联，schema v2） |
| 加密 | cryptography 2.9（AES-256-GCM 列级加密 + PBKDF2-HMAC-SHA256 口令哈希） |
| 凭证 | flutter_secure_storage 10（Keychain / Keystore） |
| SSH | dartssh2 4.1（密码 / 私钥、exec、PTY shell） |
| 终端 | xterm 4.0（真实 ANSI 渲染 + 自绘 HoloLink 键盘） |
| 二维码 | qr_flutter 4.1 |
| 生物识别 | local_auth 2.3（PIN 仍是主路径，生物识别为快捷替代） |
| 本地服务 | dart:io HttpServer（HoloLink，绑定 0.0.0.0:8080） |
| 导出 | path_provider + JSON |
| 测试 | flutter_test + sqflite_common_ffi（内存库） |

---

## 2. 目录结构

```
lib/
├── main.dart                          # 打开 DB → 取主密钥 → ProviderScope
├── app.dart                           # MaterialApp.router + 应用锁闸门 + 中文本地化
├── core/
│   ├── app_info.dart                  # 版本 / 发布名
│   ├── database/app_database.dart     # 建表、v1→v2 迁移、WAL
│   ├── providers/app_providers.dart   # 组合根（DB/加密/仓储/授权）
│   ├── router/                        # 路由与统一转场
│   ├── security/                      # 主密钥、AES-GCM、PBKDF2、会话
│   ├── settings/app_settings.dart     # 按账号隔离的键值设置
│   └── theme/                         # 设计令牌 / 排版 / ThemeData
├── shared/widgets/                    # GlassPanel / SentinelPage / SentinelMenu / 表单
├── features/
│   ├── auth/                          # 登录注册、会话恢复、改名/注销
│   ├── shell/                         # 底部导航
│   ├── assets/                        # 设备列表、操作菜单、分组、详情、绑定
│   ├── ssh/                           # 连接池、重连、遥测、告警写入
│   ├── terminal/                      # xterm + PTY + HoloLink 键盘
│   ├── alerts/                         # 告警中心（已读/删除/清空）
│   ├── history/                       # 命令历史（加密）
│   ├── scripts/                       # 159 条指令库、社区仓库、自定义脚本
│   ├── membership/                    # 付费墙、激活码、永久授权
│   ├── ai/                            # OpenAI 兼容客户端、脱敏、排障面板
│   ├── hololink/                      # 局域网 HTTP 服务与大屏页面
│   ├── security/                      # 应用锁、PIN、锁屏
│   └── profile/                       # 我的、设置中心及 8 个子页
tools/
├── SentinelToolkit.ps1                # Windows 一键运维工具箱（自动提权）
├── 一键运维工具箱.bat
├── generate_icons.ps1                 # 图标 / 启动图生成
└── README.md
```

---

## 3. 数据层（schema v2）

| 表 | 说明 | 加密字段 |
| --- | --- | --- |
| `users` | 账号、PBKDF2 哈希、会员等级、昵称 | 无（口令已哈希） |
| `devices` | 设备元数据、分组、最近遥测、状态 | 无 |
| `command_logs` | 命令 + 输出 | `payload` |
| `alerts` | 命令失败 / 连接中断 | `payload` |
| `scripts` | 自定义脚本正文 | `payload` |
| `settings` | 通知、外观、HoloLink 端口等键值 | 无 |

- v1 → v2 迁移只新增 `settings` 表，不破坏旧数据。
- 所有查询按 `user_id` 隔离；删除账号级联清空全部本地数据。
- 删除设备使用事务：`command_logs` → `alerts` → `devices`，随后清 Keychain。
- 命令历史保留最近 200 条，告警保留最近 300 条。

```
Keychain/Keystore ──32B 主密钥──► CryptoBox(AES-GCM)
                                      │
       命令正文 / 日志正文 / 脚本正文 ──┴──► base64(nonce|cipher|mac) ──► SQLite
```

---

## 4. 资产与告警交互

### 设备操作菜单

三种入口，全部走同一个 `showDeviceActionSheet`：

1. 长按设备卡片；
2. 左滑卡片（`Dismissible.confirmDismiss` 打开菜单后回弹，不会误删）；
3. 卡片右上角「更多」。

菜单动作：重新连接、修改分组、打开终端、查看详情、删除设备。

**红灯设备可删**：入口不读取链路状态；删除流程先 `forget` 链路，再事务删除本地数据，最后清凭证。顶部离线检测同时判断 `DeviceStatus.offline` 与 `LinkPhase.failed`，一键清空会批量走同一套级联逻辑。

### 告警

- 打开详情自动标记已读。
- `Dismissible` 左滑删除单条。
- 顶部「一键已读」与「清空全部」。
- `SshController._record` 写告警前读取通知设置，关闭后只保留历史、不产生告警。

---

## 5. 商业授权

```
点击立即解锁
  └─ 支付方式面板
      ├─ 微信支付 / 支付宝 → 客服微信提示 + 激活码输入
      ├─ PayPal → 收款账号 + 激活码输入
      └─ 输入激活码 → 直接校验
```

- `LicenseService.validate` 离线校验：
  - 永久主码：`SENTINEL-PRO-2026`、`SENTINEL-PRO-LIFETIME`、`SENTINEL-VIP-2026`、`SNTL-2026-PRO`；
  - 规则分发码：`SNTL-XXXX-XXXX-CCCC`，`CCCC` 为前 8 位 Base36 加权校验。
- 成功后写 `users.membership='pro'`，并把授权 JSON 写入安全区。
- 冷启动时若数据库为 free 但安全区有授权，自动恢复为 Pro。

---

## 6. AI 排障

```
告警详情 → 呼叫 AI 专家（Pro 校验）
  → AiService._chat()
      ├─ 读取安全区配置（baseUrl / model / apiKey）
      ├─ _redact：口令、Token、私钥、Bearer 头脱敏
      ├─ POST {baseUrl}/chat/completions
      └─ 解析 choices[0].message.content
```

- 兼容 OpenAI / DeepSeek / Moonshot / Ollama / 自建网关。
- 失败信息中文化（超时、TLS、非 JSON、HTTP 错误）。
- 未配置时提供「去配置」入口。

---

## 7. HoloLink

### 终端键盘

`HoloKeyboard` 是自绘输入层，不替代系统输入法：

- 字母层 / 符号层；
- Ctrl 组合（a-z → 0x01-0x1A）、Alt 前缀 ESC；
- ESC / TAB / 方向键 / Home / End / 删除；
- 全部输入先进本地缓冲，Enter 整行发送；
- 触感反馈读取用户设置。

终端页命令栏默认 `readOnly`，点击即展开键盘；右上角可切换系统输入法。

### 局域网大屏

`HoloLinkController` 直接 `HttpServer.bind(InternetAddress.anyIPv4, port)`：

| 路由 | 内容 |
| --- | --- |
| `/` | 深色霓虹大屏 HTML，5 秒自动刷新 |
| `/api/status` | 运行状态、端口、运行时长、访问次数 |
| `/api/devices` | 设备快照（名称、地址、状态、CPU/内存/磁盘） |
| `/health` | `ok` |

- 枚举 `NetworkInterface` 获取局域网 IPv4，优先 192.168 / 10 / 172 网段。
- 二维码内容为 `http://<局域网IP>:<端口>`。
- 客户端连接按 2 分钟窗口去重统计。
- 页面内置防火墙 / AP 隔离 / 端口占用排查提示。
- **只读、无凭证；未内置鉴权，禁止映射公网。**

---

## 8. 安全锁

```
AppLockGate（MaterialApp.builder 内）
  ├─ 未启用 → 正常路由
  ├─ 已启用且锁定 → LockScreen 覆盖
  └─ 解锁 → PIN（PBKDF2）或生物识别
```

- PIN 仅存哈希 + 盐，不存明文。
- 生物识别能力在进入锁屏 / 安全页时才探测，避免冷启动被平台通道拖慢。
- 忘记 PIN 只能清除应用数据，UI 已明示。

---

## 9. 设置与导出

- `settings` 表按账号保存：告警开关、每日汇总、触感、终端字号、减少动效、HoloLink 端口。
- 减少动效直接短路路由转场。
- 数据导出 JSON 包含：账号、设备、告警、脚本（含明文命令）、设置；不含密码 / 私钥。
- 支持复制到剪贴板或写入应用文档目录。

---

## 10. 内置指令库

`features/scripts/domain/builtin_script_catalog.dart`：

- 15 个分类、159 条命令；
- 每条命令自带 `sudo`；
- 防火墙命令做发行版自适应：

```sh
if command -v ufw >/dev/null; then sudo ufw allow 8080/tcp
elif command -v firewall-cmd >/dev/null; then sudo firewall-cmd --permanent --add-port=8080/tcp && sudo firewall-cmd --reload
else sudo iptables -I INPUT -p tcp --dport 8080 -j ACCEPT && (sudo netfilter-persistent save || sudo iptables-save > /etc/iptables/rules.v4); fi
```

- 端口映射覆盖：iptables REDIRECT / DNAT、firewalld forward-port / rich rule、nftables、socat、rinetd、SSH 隧道。
- 分类级 Pro 门槛；高危命令单独标记。

---

## 11. 视觉与转场

- 图标：深色 `#05070A` + 霓虹绿 `#00F5A0`，`>_` 终端提示符 + 网格 + 状态灯；Android 自适应图标，iOS 24bpp 无 alpha。
- 启动图：Android 12 SplashScreen + 传统 layer-list；iOS LaunchScreen 深色背景 + LaunchImage。
- 转场：`CustomTransitionPage` 320ms 淡入 + 4.5% 右滑；Material 内置路由使用 Zoom / Cupertino。
- 中文本地化：`flutter_localizations` + `GlobalMaterialLocalizations`。

---

## 12. 运行与验证

```bash
flutter pub get
flutter analyze          # 0 issue
flutter test             # 18 passed
flutter run
flutter build apk --debug
```

真机自测建议：

1. 注册 → 退出 → 重新登录，确认设备仍在。
2. 绑定设备，长按 / 左滑 / 更多，分别测试重连、改分组、删除。
3. 断开网络执行命令，确认告警产生；左滑删除单条、清空全部。
4. 打开终端，点击命令栏验证 HoloLink 键盘；测试 Ctrl-C、方向键、整行发送。
5. 我的 → HoloLink，启动服务，扫二维码从另一台设备访问。
6. 付费墙 → 微信支付 → 输入 `SENTINEL-PRO-2026` → 永久 Pro。
7. 设置 → 安全锁，设置 PIN 后「立即锁定」，验证冷启动锁屏。
8. 设置 → 数据导出，生成并保存 JSON。

### 构建环境注意

本机若为 **JDK 25 + Gradle 8.12**，Gradle 8.12 不支持 JDK 25，`flutter build apk` 会报 `What went wrong: 25.0.3`。
解决方式（二选一）：

- 安装 JDK 17 / 21，`flutter config --jdk-dir <路径>`；
- 或把 `android/gradle/wrapper/gradle-wrapper.properties` 升到 Gradle 9.x 并同步升级 AGP。

Dart 层（analyze / test）不依赖这两者。
