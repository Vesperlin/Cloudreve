# Cloudreve
自建多网盘中枢 / 聚合网盘方案（AList + Cloudreve + rclone）

目标：搭一个“统一中枢”，绑定多个网盘（百度、阿里、夸克等），用一个自建网盘作为最终存储，一键克隆其它网盘内容，并通过 WebDAV 让 iPhone / Windows 像正常网盘一样使用。

⸻

0. 项目目标与功能概览

核心目标
	1.	自建一个主网盘（Cloudreve），所有重要文件最终都落到这里。
	2.	使用 AList 绑定多个第三方网盘账号（百度网盘、阿里云盘、夸克网盘、123 云盘等）。
	3.	支持通过分享链接（含提取码）挂载第三方网盘内容，不打开原网盘客户端即可浏览/下载。
	4.	使用 rclone 作为“搬运工”，定时把 AList 挂载的各网盘内容克隆到 Cloudreve。
	5.	给 iPhone / Windows 暴露统一的 WebDAV 接口，实现“一个入口访问所有自建内容”。

功能清单
	•	多网盘统一聚合：
	•	挂载：百度网盘账号 / 百度分享链接
	•	挂载：阿里云盘账号 / 分享链接
	•	挂载：夸克网盘账号 / 分享链接
	•	挂载：123 云盘账号 / 分享链接
	•	统一 Web 访问：
	•	通过浏览器访问 AList 界面
	•	通过浏览器访问 Cloudreve 界面
	•	自动克隆：
	•	rclone 从 AList 暴露的 WebDAV 读取文件
	•	rclone 写入 Cloudreve 的主存储目录 / 对象存储
	•	支持按目录、按规则过滤、增量同步
	•	多终端访问：
	•	iPhone / iPad 通过 WebDAV 客户端访问（Documents / FE File Explorer / 原生“文件 App”）
	•	Windows 通过 WebDAV 映射网络驱动器
	•	安全：
	•	全站 HTTPS（反向代理 / Caddy / nginx / Traefik）
	•	独立账号体系 + 强密码
	•	网页管理后台仅管理员可访问

⸻

1. 整体架构设计

1.1 架构示意

                ┌─────────────────────────┐
                │       iPhone / iPad     │
                │  Web / WebDAV / Apps    │
                └────────────┬────────────┘
                             │ HTTPS / WebDAV
                ┌────────────▼────────────┐
                │   反向代理 (Caddy/Nginx) │
                │  - alist.example.com    │
                │  - cloud.example.com    │
                └────────────┬────────────┘
           HTTP/WS           │              HTTP/WS
        ┌────────────────────┴──────────────────────┐
        │                                           │
┌───────▼────────┐                         ┌────────▼─────────┐
│      AList     │                         │    Cloudreve     │
│ 多网盘聚合器   │                         │ 自建主网盘系统   │
│ - 挂载各网盘   │ <───────────────┐       │ - Web / WebDAV   │
│ - 暴露 WebDAV  │                 │       │ - 存储后端       │
└───────┬────────┘                 │       └────────┬─────────┘
        │ WebDAV                   │                │
        │                          │                │
        │        ┌─────────────────▼────────────────▼─────────┐
        │        │                 rclone                      │
        └────────►  从 AList 拉取 → 写入 Cloudreve 存储后端   │
                 └─────────────────────────────────────────────┘


第三方网盘：
Baidu / AliyunDrive / Quark / 123 / OneDrive / Google Drive ...
通过 AList 驱动挂载，统一进 AList。

1.2 组件职责
	•	AList
	•	负责“连接所有外部网盘”和“解析分享链接”
	•	提供 Web 管理界面和统一 WebDAV
	•	Cloudreve
	•	作为“最终自建网盘”
	•	提供 Web 网盘界面和 WebDAV
	•	rclone
	•	作为“同步 / 搬运引擎”
	•	从 AList WebDAV 拉取文件，写入 Cloudreve 后端
	•	反向代理（Caddy / Nginx / Traefik 等）
	•	提供域名和 HTTPS
	•	转发流量到 AList / Cloudreve

⸻

2. 环境与前置条件

2.1 硬件 / 网络
	•	一台公网 VPS 或家用服务器：
	•	推荐：2 核 CPU、4GB 内存起步（视并发情况可调）
	•	硬盘：按你计划的存储量预留（至少几百 GB 起步）
	•	公网 IP + 域名：
	•	示例：alist.example.com / cloud.example.com
	•	域名支持添加 A 记录

2.2 软件 / 系统
	•	操作系统：Linux（推荐 Ubuntu / Debian）
	•	Docker：已安装
	•	docker-compose：已安装（v2+）

（Codex 根据 README 自动生成安装脚本时，可包含一键安装 Docker / docker-compose 的脚本）

⸻

3. 项目目录结构约定

项目放在一个独立目录，比如 /opt/multi-cloud-hub，结构约定如下（供 Codex 生成代码用）：

multi-cloud-hub/
├─ docker-compose.yml          # 主编排文件
├─ .env                        # 环境变量配置（域名、路径、账号等）
├─ reverse-proxy/              # 反向代理配置（Caddyfile/nginx.conf）
├─ alist/                      # AList 数据 & 配置卷
│  └─ data/
├─ cloudreve/                  # Cloudreve 数据 & 配置卷
│  └─ data/
├─ storage/                    # Cloudreve 实际存储目录（本地盘方案）
│  ├─ cloudreve-main/          # Cloudreve 主存储根目录
│  └─ rclone-tmp/              # rclone 临时目录
├─ rclone/
│  ├─ rclone.conf              # rclone 配置文件
│  ├─ scripts/
│  │  ├─ sync_baidu_to_cloudreve.sh
│  │  ├─ sync_quark_to_cloudreve.sh
│  │  └─ sync_all.sh
│  └─ cron/
│     └─ crontab               # 定时任务配置
└─ docs/
   └─ README.md                # 本文档

说明：
	•	Codex 需要按上述结构生成对应的 Docker 配置、脚本、示例文件。
	•	storage/cloudreve-main/ 可替换为挂载对象存储（S3 / OSS）的方式，由 Cloudreve 进行管理。

⸻

4. 部署步骤总览
	1.	准备基础环境（Docker / docker-compose，配置域名解析）。
	2.	克隆本仓库到服务器。
	3.	编辑 .env 文件，填入基本配置。
	4.	编写并确认 docker-compose.yml。
	5.	运行 docker compose up -d 启动所有服务。
	6.	首次登录 AList：
	•	设置管理员账户
	•	挂载本地存储（Cloudreve 主存储目录，只读）
	•	挂载百度 / 阿里 / 夸克 / 123 等网盘
	•	配置分享链接驱动（BaiduYun Share 等）
	7.	首次登录 Cloudreve：
	•	设置管理员账户
	•	配置存储策略（本地目录 / 对象存储）
	•	启用 WebDAV
	8.	配置 rclone：
	•	配置 AList WebDAV 作为“源”
	•	配置 Cloudreve 主存储作为“目标”（本地目录或对象存储）
	•	编写同步脚本 & cron 定时任务
	9.	在 iPhone / Windows 上配置 WebDAV 客户端，接入 Cloudreve。
	10.	测试从“分享链接 → AList → rclone → Cloudreve → iPhone/Windows”全链路是否正常。

⸻

5. .env 配置说明（示例变量）

建议在项目根目录创建 .env，作为 docker-compose 和脚本的统一配置来源。以下为建议字段（由 Codex 在实际代码中引用）：

# 通用
TZ=Asia/Shanghai

# 域名配置
ALIST_DOMAIN=alist.example.com
CLOUDREVE_DOMAIN=cloud.example.com

# 反向代理使用的 Email（申请 HTTPS 证书用）
ACME_EMAIL=your-email@example.com

# AList 管理相关（可选，首次运行也可以用默认再改）
ALIST_ADMIN_USER=admin
ALIST_ADMIN_PASS=强密码

# Cloudreve 管理相关
CLOUDREVE_ADMIN_USER=admin
CLOUDREVE_ADMIN_PASS=强密码

# 存储路径（容器内挂载到 /data 等）
DATA_ROOT=/opt/multi-cloud-hub
STORAGE_ROOT=/opt/multi-cloud-hub/storage

Codex 需要：在 docker-compose.yml 中，使用 ${DATA_ROOT}、${STORAGE_ROOT} 等变量挂载目录。

⸻

6. docker-compose 编排设计要点

docker-compose.yml 至少包含以下服务：
	1.	reverse-proxy（Caddy / Nginx / Traefik，推荐 Caddy 简化 HTTPS）
	2.	alist（多网盘聚合器）
	3.	cloudreve（主网盘）
	4.	rclone-cron（可选：基于轻量容器 + cron 的同步服务）

6.1 关键点约定（供 Codex 实现）
	•	所有服务放在同一个 docker 网络 multi-cloud-hub。
	•	AList 容器内：
	•	监听端口：5244（默认）
	•	数据目录：/opt/alist/data（容器内路径，可自定义）
	•	Cloudreve 容器内：
	•	监听端口：5212（示例）
	•	数据目录：/cloudreve/uploads（上传目录）
	•	配置目录：/cloudreve/config
	•	反向代理：
	•	将 https://ALIST_DOMAIN 转发到 alist:5244
	•	将 https://CLOUDREVE_DOMAIN 转发到 cloudreve:5212
	•	rclone 容器：
	•	挂载 ./rclone/rclone.conf 到 /config/rclone/rclone.conf
	•	挂载 ./storage/cloudreve-main 到 /mnt/cloudreve-main
	•	挂载 AList 的 WebDAV 作为远端（通过 https://ALIST_DOMAIN/dav/...）

注意：具体容器镜像名、标签和参数由 Codex 按当前主流版本选择，例如：
	•	xhofe/alist:latest
	•	cloudreve/cloudreve:latest
	•	rclone/rclone:latest
	•	caddy:latest 或 nginx:alpine

⸻

7. AList 配置流程

7.1 首次登陆与管理员设置
	1.	访问：https://ALIST_DOMAIN
	2.	使用默认账号登录（视 AList 版本而定，一般在容器日志中会打印初始账号密码）。
	3.	登录后立刻修改：
	•	管理员用户名
	•	管理员密码
	•	面板访问路径（可选，提升安全性）

7.2 挂载 Cloudreve 主存储（只读）

目的：在 AList 里看到 Cloudreve 已有文件，方便统一浏览（可选）。
	•	挂载类型：本地目录或 WebDAV（视实现方式）
	•	路径指向：/storage/cloudreve-main（对应宿主机挂载到容器内部）
	•	挂载选项：只读（避免 AList 和 Cloudreve 并发写入同一目录结构）

7.3 挂载第三方网盘账号

按 AList 文档说明，为以下网盘分别创建挂载：
	•	BaiduNetdisk（账号驱动）
	•	AliyunDrive（阿里云盘）
	•	Quark（夸克网盘）
	•	123Pan（123 云盘）
	•	其他如 OneDrive / Google Drive 等（视需要选择）

配置方式由 AList 提供（通常为 token / refresh token / cookies 等），在 README 中不写具体隐私字段，交给用户在面板里填。

7.4 挂载“分享链接”驱动

示例：百度分享链接挂载流程（逻辑说明，具体字段由 AList 文档决定）：
	1.	新建存储：
	•	类型：BaiduYun Share Link（名称以 AList 实际驱动为准）
	•	参数：
	•	分享链接 URL
	•	提取码（若有）
	2.	保存后，该分享即作为一个只读存储出现在 AList 的目录树下。
	3.	通过 WebDAV 或 Web 界面即可看到分享里的真实文件。

对阿里/夸克/123 等分享链接同理，选择对应的“Share / Link”驱动类型。

⸻

8. Cloudreve 配置流程

8.1 首次登录
	1.	访问：https://CLOUDREVE_DOMAIN
	2.	根据 Cloudreve 初始化流程设置管理员账号和密码（如需，也可通过环境变量覆盖）。

8.2 配置存储策略

有两种典型方案：

方案 A：本地存储
	1.	在 Cloudreve 后台“存储策略”中新增本地存储：
	•	根目录指向：容器内的 /cloudreve/uploads（实际映射到宿主机 storage/cloudreve-main）。
	•	上传限制与目录结构按需配置。
	2.	将该存储策略设为默认。

方案 B：对象存储（OSS / S3 等）
	1.	在 Cloudreve 后台设置对应的 S3 / OSS 存储：
	•	AccessKey / SecretKey / Endpoint / Bucket 等信息。
	2.	上传路径、绑定域名按需求配置。
	3.	如选此方案，需要在 rclone 与 Cloudreve 之间多考虑同步规则（可以直接让 rclone 写入对象存储，也可以写入 Cloudreve 本地再由 Cloudreve 迁移）。

初版推荐使用方案 A（本地存储），架构简单清晰，之后根据实际情况再扩展。

8.3 启用 WebDAV 接口

在 Cloudreve 控制台中：
	1.	打开 WebDAV 功能（如果有开关）。
	2.	设置 WebDAV 连接路径、端口（通常与主服务共用）。
	3.	记录：
	•	WebDAV 地址（如 https://CLOUDREVE_DOMAIN/dav/）
	•	WebDAV 用户名 / 密码（可以直接使用 Cloudreve 账号）。

⸻

9. rclone 同步与克隆策略

9.1 rclone remote 命名约定

建议在 rclone.conf 中定义以下 remote（示例）：

[alist-baidu-share]
type = webdav
url = https://ALIST_DOMAIN/dav/baidu-share
vendor = other
user = <如需认证则填写>
pass = <如需认证则填写>

[alist-quark]
type = webdav
url = https://ALIST_DOMAIN/dav/quark
vendor = other

[cloudreve-local]
type = local
nounc = true
shadow_copy = false
# 对应 /mnt/cloudreve-main 挂载点

实际路径和认证方式由 Codex 根据 AList WebDAV 路径和安全配置自动填入。

9.2 同步脚本设计（供 Codex 实现）

目标：自动把 AList 某个路径下的内容同步到 Cloudreve 主存储的某个子目录。

示例脚本：rclone/scripts/sync_baidu_to_cloudreve.sh
	•	核心命令大致为：

#!/usr/bin/env bash
set -e

# 从 AList 中的某个路径同步到 Cloudreve 本地目录
rclone copy \
  alist-baidu-share:/ \
  cloudreve-local:/baidu-from-share \
  --ignore-existing \
  --transfers=8 \
  --checkers=8 \
  --progress

脚本要求：
	•	不删除 Cloudreve 端已有文件（默认只追加，不做双向同步）。
	•	可接受多个源 remote（百度、阿里、夸克等），分别对应不同脚本或统一脚本。
	•	返回码非 0 时输出错误日志。

9.3 定时任务（cron）设计
	•	在 rclone/cron/crontab 中定义，例如：

# 每小时同步一次百度分享
0 * * * * /rclone/scripts/sync_baidu_to_cloudreve.sh >> /var/log/rclone_baidu.log 2>&1

# 每天同步一次夸克
30 3 * * * /rclone/scripts/sync_quark_to_cloudreve.sh >> /var/log/rclone_quark.log 2>&1

	•	rclone-cron 容器启动时，将该 crontab 安装到系统 cron，常驻运行。

⸻

10. 客户端接入：iPhone / Windows

10.1 iPhone / iPad 接入（推荐 WebDAV）

方式 A：使用第三方 App（推荐）
以 FE File Explorer / Documents 为例：
	1.	安装 App。
	2.	添加新连接：
	•	类型：WebDAV
	•	服务器：https://CLOUDREVE_DOMAIN/dav/
	•	用户名：Cloudreve 用户名
	•	密码：Cloudreve 密码
	3.	保存后即可浏览、上传、下载 Cloudreve 中的文件。

方式 B：使用“文件 App”连接服务器
	1.	打开 iOS“文件”App。
	2.	点击“连接到服务器”。
	3.	输入：
	•	https://CLOUDREVE_DOMAIN/dav/
	4.	按提示输入用户名、密码。
	5.	成功后 Cloudreve 会出现在“位置”列表中。

10.2 Windows 接入

方式 A：映射 WebDAV 为网络驱动器
	1.	打开“此电脑”。
	2.	点击“映射网络驱动器”。
	3.	选择一个盘符（例如 Z:）。
	4.	文件夹填入：https://CLOUDREVE_DOMAIN/dav/
	5.	勾选“使用其他凭据连接”。
	6.	输入 Cloudreve 账号和密码。
	7.	完成后，Cloudreve 将作为一个盘符出现在“此电脑”中。

方式 B：使用 rclone mount
	1.	在 Windows 上安装 rclone。
	2.	配置 cloudreve-webdav remote。
	3.	使用 rclone mount cloudreve-webdav: Z: 挂载为盘符（需要 WinFsp 或 Dokany）。
	4.	可在资源管理器中当本地盘使用。

⸻

11. 安全与合规注意事项
	1.	权限范围
	•	所有同步、克隆行为必须在你对文件有合法访问权限的前提下执行（自己的网盘账号 / 合法授权的分享链接）。
	•	不要利用 AList / rclone 去绕过服务条款、权限控制、DRM 或其他安全机制。
	2.	账号安全
	•	第三方网盘的 token / cookies 等敏感信息不要写死在仓库里。
	•	统一写在 AList 面板中或在服务器环境变量中管理，避免 commit 到 GitHub。
	3.	网络安全
	•	反向代理必须启用 HTTPS。
	•	考虑只允许特定 IP/地区访问管理面板。
	•	建议开启 Fail2ban / WAF 等防护措施。
	4.	数据安全
	•	为 Cloudreve 主存储配置定期备份（本地/异地都可）。
	•	确保 rclone 操作始终是“单向复制”，避免误删数据。

⸻

12. 后续可扩展方向

本 README 定义的是“第一版完全体”方案，未来可以扩展：
	•	增加：
	•	OCR、缩略图、媒体转码等后台任务。
	•	自动化规则（按文件类型自动分类目录）。
	•	集成：
	•	Telegram / Discord / 企业微信 Bot，通知同步结果。
	•	WebUI 中增加“任务状态面板”，展示 rclone 同步日志。
	•	拆分：
	•	将存储后端从本地迁移到 S3 / OSS，对接 CDN 加速访问。

⸻

13. 给 Codex / 自动化生成器的实现提示

为方便自动生成配置和代码，约定如下关键点：
	1.	所有路径、域名、账号、密码从 .env 中读取。
	2.	docker-compose.yml 中：
	•	使用外部网络 multi-cloud-hub（如不存在则自动创建）。
	•	反向代理、AList、Cloudreve、rclone-cron 均加入该网络。
	3.	生成的脚本必须：
	•	默认使用 bash，兼容常见 Linux 发行版。
	•	对关键步骤加上 set -e，防止静默失败。
	4.	所有敏感信息（token / cookies / 密码）不得写入 Git 仓库示例文件，统一留出占位符让用户自行填入。

⸻

以上即为整个“多网盘中枢 + 自建主网盘 + 自动克隆”的完整设计文档。
让 Codex 据此自动生成：
	•	docker-compose.yml
	•	reverse-proxy 配置
	•	各类 rclone 脚本和 cron 配置
	•	一键部署脚本等。
