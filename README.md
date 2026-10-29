# DevOps Sentinel · V1.0 终极商业版

> 全终端极客运维中台：资产、SSH、终端、告警、脚本、AI 排障、局域网大屏，全部本地闭环。

- Flutter 3.35 / Dart 3.9
- 纯本地存储：SQLite + Keychain / Keystore
- SSH 直连：dartssh2 + xterm
- 零中转服务器，不上传业务数据

## 功能总览

### 资产与会话
- 宿主机 / 虚拟机 / 云服务器三分类，设备卡片展示 CPU / 内存 / 磁盘。
- 长按、左滑或右上角「更多」打开设备操作菜单：重新连接、修改分组、打开终端、查看详情、删除设备。
- 删除设备会同步清理 SQLite 设备记录、该设备的命令历史 / 告警，以及 Keychain / Keystore 凭证。
- 红灯离线设备同样可以删除，不再受 SSH 连通性影响。
- 顶部自动检测离线设备，一键清空（含凭证与本地记录）。

### 告警中心
- 命令失败、连接中断自动写入，正文 AES-256-GCM 加密。
- 一键已读、左滑删除单条、清空全部告警。
- 告警详情支持复制日志与「呼叫 AI 专家」。

### 脚本库
- 内置 **159 条 root 级指令**，15 个收纳分组，点开分组展开。
- 覆盖系统巡检、磁盘日志、网络、防火墙、端口映射、Nginx、Docker、数据库、SSH 加固、用户权限、服务、内核性能、备份、证书、K8s、监控与一键排障。
- 防火墙命令自动识别 ufw / firewalld / iptables；端口映射提供 iptables / firewalld / nftables / socat / rinetd 多套方案。
- 支持搜索、全部展开 / 收起、自定义加密脚本、GitHub Raw 导入、离线社区精选包。

### Sentinel Pro
- 支付选择面板：微信支付、支付宝、PayPal、输入激活码。
- 微信 / 支付宝会提示「请添加客服微信购买激活码」并提供激活码输入框。
- 激活码本机离线校验，**永久主码：`SENTINEL-PRO-2026`**（另有 `SENTINEL-PRO-LIFETIME`、`SENTINEL-VIP-2026`、`SNTL-2026-PRO`）。
- 还支持 `SNTL-XXXX-XXXX-CCCC` 规则分发码。
- 授权写入系统安全区，重装后与本地账号自动对齐。

### 终端与 HoloLink
- 真实 SSH PTY + xterm 渲染。
- 点击命令输入框自动弹出 **HoloLink 极客键盘**：Ctrl / Alt / Shift、ESC / TAB、方向键、符号层、整行发送。
- 可一键切回系统输入法。
- 「我的 → HoloLink 局域网大屏」启动真实 HTTP 服务（默认 8080），生成二维码与局域网地址，同网段设备可打开只读仪表盘。
- 服务支持端口修改、访问统计、在线设备数，并内置防火墙 / AP 隔离排查指引。

### AI 排障
- 兼容 OpenAI Chat Completions 协议（OpenAI / DeepSeek / Moonshot / Ollama / 自建网关）。
- API Key 只进系统安全区；日志发送前对口令、Token、私钥做脱敏。
- 支持连接测试与自定义系统提示词。

### 设置与安全
- 设置中心：账号、AI、外观、通知、安全锁、数据导出、关于。
- 通知开关真实生效：关闭后不再写对应告警，命令历史仍保留。
- 终端字号、减少动效、触感反馈实时生效。
- 安全锁：4 位 PIN（PBKDF2 哈希）+ 生物识别，冷启动覆盖式锁屏。
- 数据导出：设备、告警、脚本、设置的 JSON 快照，可复制或保存到文件。
- 修改昵称、查看永久授权、注销账号并级联删除本地数据。

### 视觉
- 深色霓虹绿终端图标（`>_` + 网格 + 状态灯），Android 自适应图标 + iOS 全套尺寸。
- Android / iOS 深色启动图与 Flutter 启动页。
- 统一 320ms 淡入 + 右滑转场；「减少动效」可关闭。
- 中文 Material 本地化。

## 快速开始

```bash
flutter pub get
flutter analyze          # 0 issue
flutter test             # all passed
flutter run
```

Android 生物识别需要 `FlutterFragmentActivity`（已配置）；iOS 已添加 Face ID 与本地网络权限说明。

## Windows 一键运维工具箱

项目根目录：

```bat
一键运维工具箱.bat
```

- 自动 UAC 提权。
- 156 条命令，10 个分类收纳，高危操作输入 `YES` 确认。
- 专治：防火墙放行、portproxy 端口映射、IP Helper、IP 转发、WSL 转发、端口占用、DNS 异常、RDP / SSH 连不上、HoloLink 8080 打不开。
- 详见 [`tools/README.md`](tools/README.md)。

## 目录

```
lib/
├── core/            # 数据库、路由、主题、设置、安全
├── shared/widgets/  # 玻璃面板、表单、菜单、页面骨架
└── features/
    ├── assets/      # 资产、设备操作菜单、绑定、详情
    ├── alerts/      # 告警中心
    ├── scripts/     # 159 条指令库、社区仓库
    ├── terminal/    # xterm + HoloLink 键盘
    ├── hololink/    # 局域网 HTTP 大屏
    ├── ai/          # AI 排障
    ├── membership/  # 付费墙、激活码
    ├── security/    # 应用锁、锁屏
    ├── profile/     # 我的、设置中心及各子页
    ├── ssh/         # 连接池、重连、遥测
    ├── auth/        # 登录注册
    └── ...
tools/               # 图标生成器、Windows 工具箱
```

更详细的技术说明见 [`ARCHITECTURE.md`](ARCHITECTURE.md)。

## 安全边界

- SSH 密码 / 私钥只写 Keychain / Keystore，不落 SQLite、不上传。
- 命令、告警、脚本正文以 AES-256-GCM 加密落盘。
- 卸载 App 会清除主密钥，数据库密文随之不可解。
- HoloLink 只读、不含凭证；**不要直接把 8080 映射到公网**（当前版本未内置鉴权）。
- 数据导出包含明文命令与设备地址，请只保存在可信位置。
