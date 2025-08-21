# 在线视频课程学习平台（Python + MySQL）开发需求文档

> 目标：在一台 Ubuntu 服务器（已具备 MySQL 数据库）上快速部署一套“可公开浏览所有课程、部分课程需登录授权播放”的视频学习网站；视频来源支持**本地上传**与**哔哩哔哩嵌入**；后台可精细指定“哪些用户可观看哪些课程”；对**下载与录屏**做“尽最大努力”的防护。

---

## 1. 技术栈与总体架构

* **后端框架**：Python 3.11 + **Flask**（轻量、易上手；若偏好全家桶，可替换为 Django，本方案以 Flask 为主）
* **数据库**：**MySQL 8.x**（已具备），ORM 使用 **SQLAlchemy** + **Alembic**（迁移）
* **鉴权**：Flask-Login（会话）+ itsdangerous（签名）/PyJWT（短期令牌）
* **缓存/队列**：Redis（可选，用于登录限流、HLS 签名缓存、后台任务队列）
* **任务与转码**：Celery（可选）+ ffmpeg（本地视频转 HLS）
* **前端**：Jinja2 模板 + TailwindCSS（或简单 Bootstrap）
* **视频分发**：

  * 本地视频：**HLS（m3u8 + TS/fragmented MP4）**，AES-128/ SAMPLE-AES 加密、Nginx `secure_link` 短期签名
  * 哔哩哔哩：`<iframe>` 嵌入（或按 BVID/嵌入代码存储）
* **Web 服务器**：Nginx（反向代理、静态/HLS 分发、HTTPS、CSP、安全头）
* **Wsgi**：Gunicorn（Flask 生产部署）
* **操作系统**：Ubuntu 20.04/22.04/24.04（均可）

> 说明：**无法绝对防录屏**。行业常见做法是**动态可见/半透明水印**（用户ID/时间/IP）、**HLS 分段 + 短期签名**、**禁止右键/拖拽/控制台下载**、**CSP**、**禁用缓存**等综合措施，降低大规模盗链与“简单下载”的可能性。

---

## 2. 角色与权限

* **访客（未登录）**：可浏览所有课程目录；可播放“公开课程/公开视频”
* **普通用户（登录）**：在被授权范围内播放“受限课程/受限视频”
* **编辑/运营**：创建课程、上传视频、设置可见性、指派可观看用户或用户组、查看播放日志
* **管理员**：一切权限 + 账号管理（创建/禁用/重置密码）、系统配置项

> 注册策略：**不开放自助注册**，账号由管理员**统一分配**（用户名/初始密码），首次登录强制修改密码。

---

## 3. 核心业务需求

### 3.1 课程/视频展示

* 主页展示**全部课程卡片**（缩略图、标题、标签、公开/受限徽标）
* 课程详情页：展示课程简介、章节/视频列表
* 视频来源两类：

  1. **本地上传视频**：上传后存储到服务器指定目录；后台触发转码（HLS）并生成封面
  2. **哔哩哔哩视频**：后台保存 BVID 或嵌入代码；前端以 `<iframe>` 嵌入播放
* 点击视频：

  * 若为“公开视频”：直接播放
  * 若为“受限视频”：未登录则弹出登录框；登录后若**有权限**则播放；无权限则提示“无观看权限”

### 3.2 授权模型

* 视频/课程可设置三种可见性：

  * `public`：任何人可播放
  * `login_required`：登录即可播放
  * `whitelist`：仅被授权的**用户或用户组**可播放
* 后台界面支持：

  * 按**用户**或**用户组**授权到**课程/视频**（两级都可以，视频级授权优先于课程级）
  * 批量导入用户、批量授权
  * 账号禁用/启用

### 3.3 内容管理

* 课程：标题、封面、简介、标签、排序、状态（草稿/发布）
* 视频：标题、来源（本地/哔哩哔哩）、时长、封面、清晰度/码率（本地HLS）
* 本地视频上传：

  * 支持 MP4/MOV 等常见格式
  * 上传后入库 → 触发 ffmpeg 转码为 HLS（多码率自适应可选） → 生成封面图
  * 生成的 m3u8/分片放置在仅 Nginx 可直接访问的目录（受 `secure_link` 控制）
* 哔哩哔哩视频：

  * 存 BVID 或完整嵌入代码（进行安全校验与过滤）

### 3.4 反下载/反盗链/防录屏（“尽力而为”）

* **HLS 加密**：AES-128，密钥文件通过后端短期授权 URL 动态返回（校验用户会话/权限/签名）
* **短期签名 URL**：Nginx `secure_link` + 服务端签名（每个会话/每段有效期 60–300s）
* **域名与 Referer 校验**、**CORS/CSP/Referrer-Policy** 加固
* **水印**：播放器层叠 **动态水印**（用户ID、时间戳、IP 尾段）随机位置/缓慢移动
* **禁用简单下载手段**：隐藏真实文件路径、分片名不可预测、禁止右键/拖拽、`Content-Disposition: inline;`、禁止 `Range` 跨域滥用
* **限频与并发控制**：同一用户/课程同时播放流个数限制；异常请求封禁IP（fail2ban/iptables/Nginx limit\_req）
* **风控与审计**：异常分片请求模式、重复密钥试探、分享链接访问地理异常告警
* **重要提醒**：任何前端方案无法彻底阻止“屏幕录制设备”，故需依赖**合规与水印追责**。

### 3.5 观看时长统计与看板（MVP）

* 指标定义（后台看板可筛选按时间范围：今日/近7天/近30天、按课程/视频/用户）：

  * 总观看时长（秒/分钟/小时）、唯一观看用户数（UV）、观看次数（play starts）
  * 按课程/视频的 Top N 排行（观看总时长、观看次数）
  * 用户维度：每个用户的观看总时长、最近观看、观看的课程/视频清单
  * 粗粒度完成度（近似）：观看时长 ÷ 视频时长（按用户-视频去重后限最大为100%）
  * 在线并发（近实时）：过去5分钟内仍在心跳的会话数
* 统计采集方式（尽力准确）：

  * 播放会话：前端在开始播放时创建会话（`/api/playback/start`），服务端返回 `session_id`
  * 心跳：前端每 10–15 秒向 `/api/playback/heartbeat` 上报当前位置、是否在前台、播放速率
  * 结束：暂停/结束/页面卸载时调用 `/api/playback/stop`；服务端也会基于心跳超时（例如 30–60 秒）自动结束
  * 数据修正：单会话累计时长上限（例如视频时长的 120%）、同一用户同一视频同时会话合并或取一
* 权限与隐私：仅管理员/编辑可访问看板；用户只能查看自己的学习记录（可选）

> 说明：心跳法对“切到后台/静音卡滞”有一定鲁棒性，但并非绝对精确；用于运营看板足够。

---

## 4. 数据库设计（MySQL）

> 实体与关系（简化）

* `users`（用户）
* `roles`（角色，Admin/Editor/User）
* `user_roles`（多对多）
* `groups`（用户组）
* `user_groups`（多对多）
* `courses`（课程）
* `videos`（视频，关联课程或可多对多）
* `course_videos`（课程-视频 多对多/排序）
* `permissions`（授权策略：对象类型 course/video + 可见性）
* `acl_user_video`、`acl_group_video`（白名单明细）
* `acl_user_course`、`acl_group_course`（白名单明细）
* `uploads`（上传/资产表：原始文件、转码产物路径、封面）
* `playback_log`（播放日志：user、video、时间、IP、UA、分辨率）
* `audit_log`（后台操作审计）
* `config`（站点配置项）

**建议字段（DDL 草案，节选）**：

```sql
CREATE TABLE users (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  username VARCHAR(64) UNIQUE NOT NULL,
  password_hash VARCHAR(255) NOT NULL,
  full_name VARCHAR(128),
  email VARCHAR(128),
  is_active TINYINT(1) DEFAULT 1,
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE roles (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  name VARCHAR(32) UNIQUE NOT NULL
);

CREATE TABLE user_roles (
  user_id BIGINT NOT NULL,
  role_id BIGINT NOT NULL,
  PRIMARY KEY (user_id, role_id)
);

CREATE TABLE groups (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  name VARCHAR(64) UNIQUE NOT NULL,
  description VARCHAR(255)
);

CREATE TABLE user_groups (
  user_id BIGINT NOT NULL,
  group_id BIGINT NOT NULL,
  PRIMARY KEY (user_id, group_id)
);

CREATE TABLE courses (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  title VARCHAR(200) NOT NULL,
  slug VARCHAR(200) UNIQUE,
  cover VARCHAR(255),
  summary TEXT,
  visibility ENUM('public','login_required','whitelist') DEFAULT 'public',
  status ENUM('draft','published') DEFAULT 'published',
  created_by BIGINT,
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE videos (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  title VARCHAR(200) NOT NULL,
  source ENUM('local','bilibili') NOT NULL,
  bilibili_bvid VARCHAR(64),
  bilibili_embed TEXT,
  duration_sec INT,
  cover VARCHAR(255),
  visibility ENUM('public','login_required','whitelist') DEFAULT 'public',
  hls_master VARCHAR(255),     -- m3u8 相对路径（本地）
  hls_key_id VARCHAR(128),     -- 对应密钥标识（本地）
  created_by BIGINT,
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE course_videos (
  course_id BIGINT NOT NULL,
  video_id BIGINT NOT NULL,
  sort_order INT DEFAULT 0,
  PRIMARY KEY (course_id, video_id)
);

-- 课程 ACL
CREATE TABLE acl_user_course (user_id BIGINT, course_id BIGINT, PRIMARY KEY(user_id, course_id));
CREATE TABLE acl_group_course (group_id BIGINT, course_id BIGINT, PRIMARY KEY(group_id, course_id));

-- 视频 ACL
CREATE TABLE acl_user_video (user_id BIGINT, video_id BIGINT, PRIMARY KEY(user_id, video_id));
CREATE TABLE acl_group_video (group_id BIGINT, video_id BIGINT, PRIMARY KEY(group_id, video_id));

CREATE TABLE uploads (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  owner_id BIGINT,
  kind ENUM('raw_video','hls_master','hls_segment','cover') NOT NULL,
  path VARCHAR(255) NOT NULL,
  sha256 CHAR(64),
  size_bytes BIGINT,
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE playback_log (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  user_id BIGINT,
  video_id BIGINT NOT NULL,
  started_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  ip VARCHAR(45),
  user_agent VARCHAR(255),
  resolution VARCHAR(32),
  session_id VARCHAR(64),
  ended_at DATETIME NULL,
  watched_seconds INT DEFAULT 0,
  last_position_sec INT DEFAULT 0
);

CREATE TABLE playback_sessions (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  session_uid CHAR(36) UNIQUE NOT NULL, -- UUID，返回给前端
  user_id BIGINT,
  video_id BIGINT NOT NULL,
  started_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  last_heartbeat_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  accumulated_seconds INT DEFAULT 0,
  last_position_sec INT DEFAULT 0,
  ip VARCHAR(45),
  user_agent VARCHAR(255),
  is_active TINYINT(1) DEFAULT 1
);

CREATE TABLE video_progress (
  user_id BIGINT NOT NULL,
  video_id BIGINT NOT NULL,
  total_watched_seconds INT DEFAULT 0,
  last_position_sec INT DEFAULT 0,
  last_seen_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (user_id, video_id)
);

CREATE TABLE audit_log (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  actor_id BIGINT,
  action VARCHAR(64),
  object_type VARCHAR(32),
  object_id BIGINT,
  detail JSON,
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

---

## 5. 主要接口与页面

### 5.1 页面

* `/` 主页：课程列表（筛选/搜索/标签）
* `/course/<slug|id>` 课程详情 + 视频目录
* `/video/<id>` 视频播放页

  * 本地源：前端 HLS 播放器（hls.js），向 `/api/hls/<video_id>/master.m3u8?sig=...` 请求
  * 哔哩哔哩：iframe 嵌入
* `/login` 登录弹框（也支持独立页）、`/logout`
* `/admin/*` 后台（课程、视频、用户、授权、日志、系统配置）

### 5.2 API（示例）

* `GET /api/courses?query=&tag=&page=` — 课程列表
* `GET /api/courses/:id` — 课程详情（含视频清单，过滤未授权的视频）
* `POST /api/login` — 登录（返回会话或JWT）
* `GET /api/videos/:id` — 视频元数据（前端决定 HLS 或 iframe）
* `GET /api/hls/:video_id/master.m3u8?sig=...&exp=...` — 返回主 m3u8（服务端校验登录/权限/签名/有效期）
* `GET /api/hls/:video_id/key?kid=...&token=...` — 返回 AES-128 Key（短期、与用户/会话绑定）
* `POST /api/admin/video/upload` — 分片/直传，返回 upload id
* `POST /api/admin/video/ingest` — 将上传转为视频资源（触发 ffmpeg 转码）
* `POST /api/admin/acl` — 绑定用户/组与课程/视频
* `GET /api/admin/logs/playback` — 播放日志查询

* 观看统计/会话（MVP）：

* `POST /api/playback/start` — 开始播放，返回 `session_id`（UUID）
* `POST /api/playback/heartbeat` — 心跳（每 10–15s），上报 `session_id`、`position_sec`、`playing`、`visibility_state`
* `POST /api/playback/stop` — 主动结束播放，会话归档
* `GET /api/admin/dashboard/overview?from=&to=` — 概览指标（总时长、UV、播放次数、并发）
* `GET /api/admin/dashboard/top-videos?from=&to=&limit=10` — Top 视频
* `GET /api/admin/dashboard/user-activity?from=&to=&user_id=` — 用户观看明细
* `GET /api/progress?user_id=&video_id=` — 返回用户-视频的累计进度（用于前端显示“继续观看”）

> 所有 `/api/hls/*` 路由需**严格鉴权**且配合 **Nginx secure\_link** 二次校验，防盗链。

---

## 6. 播放与安全实现要点

### 6.1 本地视频 → HLS 流程

1. 用户在后台上传原始视频，落地到：`/var/www/app_data/uploads/raw/`
2. 触发任务（Celery 或同步）：`ffmpeg` 转码多码率（可选）
   输出到：`/var/www/app_data/hls/<video_id>/`

   * `master.m3u8`、`index_720p.m3u8`、`seg_720p_0001.ts` ...
3. 生成 AES-128 `key`（随机 16 字节），存储到受控目录，不直接暴露；在 DB 记录 `hls_key_id`
4. Nginx 仅负责**分片与 m3u8**静态托管，但访问路径带\*\*`secure_link` 签名\*\*（referer + ip + 到期时间）
5. 当播放器请求 `master.m3u8` 时，后端检查：登录态、ACL、签名有效期；动态返回注入了**Key URI** 的 m3u8，Key URI 指向 `/api/hls/:video_id/key?...`（再次鉴权）

### 6.2 Nginx 安全配置（关键点）

* `secure_link` / `secure_link_md5`（带过期时间戳）
* 仅允许 m3u8/ts/fmp4 的 GET 请求
* 严格 `CORS`、`Referrer-Policy: same-origin`、`Content-Security-Policy`（禁止跨域脚本）
* `X-Frame-Options: SAMEORIGIN`（除哔哩哔哩 iframe 容器页）
* `limit_req`/`limit_conn`（限频/并发）
* `gzip off` 对视频流
* 响应头禁止缓存或设置短缓存；Key 接口**不缓存**

### 6.3 前端播放器与水印

* 使用 `hls.js` 播放（Safari 可走原生 HLS）
* 叠加绝对定位 DIV 实时显示 **用户ID + 时间戳**（每 3–5 秒轻微移动位置、透明度 0.3–0.4）
* 禁用右键、拷贝、选择、拖拽；监听 `visibilitychange`、`devtools` 打开等做弱提示/埋点
* 在 DOM 中避免出现真实分片 URL（仅通过内部 blob 播放）

---

## 7. 后台功能详述

* 用户管理：新增/禁用/重置密码、批量导入（CSV）
* 用户组管理：新增组、分配成员
* 课程管理：CRUD、封面上传、可见性设置（public/login\_required/whitelist）
* 视频管理：

  * 新增（本地上传/哔哩哔哩）
  * 本地：原始文件上传、触发/查看转码状态、查看生成的多码率
  * 哔哩哔哩：填写 BVID 或粘贴嵌入代码（做 XSS 过滤）
  * 绑定到课程、排序
* 授权管理：为课程/视频指派可访问的用户或用户组；批量授权
* 日志与审计：用户登录记录、播放日志、管理操作日志
* 系统设置：站点名、Logo、SMTP（找回密码邮件）、HLS 签名秘钥轮换、最大并发、签名有效期

---

## 8. 部署方案（快速落地）

### 8.1 目录规划（示意）

```
/opt/learnsite/
  app/                     # Flask 项目
  venv/
  .env                     # 环境变量（数据库、Redis、密钥）
/var/www/app_data/
  uploads/raw/             # 原始上传
  hls/                     # 转码产物（m3u8/ts）
  covers/                  # 封面
/etc/nginx/sites-available/learnsite.conf
```

### 8.2 环境变量（.env 示例）

```
FLASK_ENV=production
SECRET_KEY=随机生成
DATABASE_URL=mysql+pymysql://user:pass@127.0.0.1:3306/myschool
REDIS_URL=redis://127.0.0.1:6379/0
HLS_SIGN_SECRET=随机生成
HLS_URL_BASE=https://your-domain/hls/
```

### 8.3 安装步骤（命令要点）

1. 系统依赖

   ```bash
   sudo apt update && sudo apt install -y python3.11-venv ffmpeg nginx mysql-client redis-server
   ```
2. Python 依赖

   ```bash
   python3 -m venv /opt/learnsite/venv
   source /opt/learnsite/venv/bin/activate
   pip install flask flask-login flask-wtf sqlalchemy alembic pymysql redis celery itsdangerous passlib[bcrypt] hlsjs-python  #（hlsjs-python可省）
   ```
3. 数据库迁移

   ```bash
   alembic upgrade head
   ```
4. Gunicorn 启动（systemd）

   ```bash
   pip install gunicorn
   # /etc/systemd/system/learnsite.service
   # ExecStart=/opt/learnsite/venv/bin/gunicorn -b 127.0.0.1:8000 "app:create_app()"
   sudo systemctl enable --now learnsite
   ```
5. Nginx 配置（要点）

   * 反向代理到 127.0.0.1:8000
   * 配置 `/hls/` 静态目录，启用 `secure_link` 校验
   * 添加安全响应头（CSP/Frame-Options/Referrer-Policy）
   * `client_max_body_size 2g;`（视上传需求）
6. HTTPS

   ```bash
   sudo apt install certbot python3-certbot-nginx
   sudo certbot --nginx -d your-domain
   ```
7. 防火墙

   ```bash
   sudo ufw allow 'Nginx Full'
   sudo ufw enable
   ```

> 可选：一键安装脚本把上述步骤写入 `install.sh`，执行后自动配置服务。

---

## 9. 关键模块实现思路（代码级要点）

* **密码安全**：`passlib[bcrypt]` 或 `argon2-cffi`；强制首次改密
* **CSRF**：Flask-WTF 全局启用
* **登录限流**：Redis + “用户名/IP 失败计数”
* **短期签名**：`itsdangerous.TimestampSigner` 生成 `sig` + `exp`，Nginx `secure_link` 再校验
* **HLS Key 发放**：后端校验会话与 ACL，无缓存，返回 `application/octet-stream` 16 bytes
* **ffmpeg 转码示例**：

  ```bash
  ffmpeg -i input.mp4 -profile:v main -level 3.1 -start_number 0 \
         -hls_time 4 -hls_list_size 0 -f hls -hls_segment_filename "seg_%04d.ts" master.m3u8
  ```

  生产中建议生成多码率与独立 `key`，m3u8 中使用 `#EXT-X-KEY:METHOD=AES-128,URI=".../key?kid=...",IV=...`

---

## 10. 安全与合规

* 全站 HTTPS，强 CSP
* 禁用目录索引，隐藏服务器版本
* 管理后台双因素（可选，TOTP）
* 日志留存与隐私合规（告知用户水印/日志用途）
* 定期轮换 `HLS_SIGN_SECRET`、数据库备份（mysqldump + 加密）
* 最小权限：Nginx 对 HLS 目录只读，应用写入受控

---

## 11. 验收标准（Definition of Done）

* [ ] 主页可浏览全部课程，公开/受限标识清晰
* [ ] 受限视频：未登录弹框，登录后根据 ACL 正确放行/拦截
* [ ] 本地视频上传 → 自动转码 → HLS 播放正常（含手机端）
* [ ] 哔哩哔哩视频嵌入可播放（PC/移动端主流浏览器）
* [ ] 播放请求具备短期签名，过期拉流失败
* [ ] 水印显示正确，信息动态变换
* [ ] 异常并发/盗链访问能被限流/拦截
* [ ] 后台可批量授权用户/组到课程/视频
* [ ] 播放与管理操作有日志可查
* [ ] 一键部署脚本可在新机器 30 分钟内完成安装与上线（示意）
* [ ] 看板展示：按时间范围能查看总观看时长/UV/播放次数；能列出 Top 视频与用户观看明细；单用户可查询自己的观看进度（可选）

---

## 12. 后续迭代路线（可选）

* 学习进度与完成度（观看时长、章节进度、合格线）
* 试题/测验/证书（题库、判分、证书 PDF）
* 点播 + 防录屏增强（更强 DRM 需引入商业方案：Widevine/FairPlay/PlayReady）
* 消息通知（邮件/短信/企业微信/钉钉）
* CDN 加速（HLS 边缘缓存 + 源站签名校验）

---

## 13. 简化时间计划（建议）

* 第 1 周：项目脚手架、数据模型、后台 CRUD、登录
* 第 2 周：上传与转码、HLS 播放、哔哩哔哩嵌入
* 第 3 周：ACL/授权、Nginx 安全、签名与限流、水印
* 第 4 周：日志与审计、部署脚本、联调与验收

---

## 14. 清单：第三方与依赖（建议）

* Python：Flask、Flask-Login、Flask-WTF、SQLAlchemy、Alembic、passlib\[bcrypt]、PyJWT/itsdangerous、redis、celery（可选）
* 系统：Nginx、ffmpeg、Redis、MySQL 客户端、certbot
* 前端：hls.js、Tailwind/Bootstrap、Alpine.js（少量交互）

---

## 15. MVP 范围与优先级

* 必做（首版落地）：

  * 课程/视频展示、登录、ACL（public/login_required/whitelist）
  * 本地视频上传 → 转码为 HLS（单码率起步）+ Nginx secure_link + Key 发放接口
  * 哔哩哔哩 iframe 嵌入播放（后台按 BVID/嵌入代码录入）
  * 播放会话与心跳、累计观看时长（用户-视频维度）
  * 后台看板基本指标与 Top 列表

* 次优先：

  * 多码率自适应、动态水印、限频/并发控制
  * 批量导入用户/授权、播放日志高级筛选
  * UI 细节优化（标签/搜索/排序、响应式栅格）

* 可延期：

  * 学习进度与完成度的严格定义、测验/证书、消息通知、CDN 加速
