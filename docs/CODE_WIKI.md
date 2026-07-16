# JumpServer Code Wiki

> 本文档为 JumpServer 仓库（`/workspace`，对应 `pyproject.toml` 中 `version = "v4.0"`）的结构化代码 Wiki，涵盖项目整体架构、主要模块职责、关键类与函数说明、依赖关系以及项目运行方式等关键信息。

---

## 目录

1. [项目概述](#1-项目概述)
2. [技术栈](#2-技术栈)
3. [目录结构](#3-目录结构)
4. [整体架构](#4-整体架构)
5. [配置系统](#5-配置系统)
6. [应用模块详解](#6-应用模块详解)
   - 6.1 [common — 共享框架层](#61-common--共享框架层)
   - 6.2 [orgs — 多租户核心](#62-orgs--多租户核心)
   - 6.3 [users — 用户与组](#63-users--用户与组)
   - 6.4 [assets — 资产盘点](#64-assets--资产盘点)
   - 6.5 [accounts — 特权账号管理](#65-accounts--特权账号管理)
   - 6.6 [perms — 资产访问授权](#66-perms--资产访问授权)
   - 6.7 [authentication — 认证与连接授权](#67-authentication--认证与连接授权)
   - 6.8 [terminal — 终端组件与会话生命周期](#68-terminal--终端组件与会话生命周期)
   - 6.9 [audits — 审计日志](#69-audits--审计日志)
   - 6.10 [ops — 自动化执行引擎](#610-ops--自动化执行引擎)
   - 6.11 [rbac — 角色权限控制](#611-rbac--角色权限控制)
   - 6.12 [tickets — 工单审批](#612-tickets--工单审批)
   - 6.13 [acls — 访问控制列表](#613-acls--访问控制列表)
   - 6.14 [notifications — 站内信与通知](#614-notifications--站内信与通知)
   - 6.15 [settings — 系统设置](#615-settings--系统设置)
   - 6.16 [labels / reports — 标签与报表](#616-labels--reports--标签与报表)
7. [跨模块关键机制](#7-跨模块关键机制)
8. [依赖关系](#8-依赖关系)
9. [项目运行方式](#9-项目运行方式)
10. [CI/CD 与工具链](#10-cicd-与工具链)

---

## 1. 项目概述

**JumpServer** 是一款开源的 **Privileged Access Management（PAM，特权访问管理）/ 堡垒机平台**。它通过 Web 浏览器为 DevOps 与 IT 团队提供对 SSH、RDP、Kubernetes、数据库、RemoteApp 等终端的按需、安全访问。

- **协议**：GPLv3（Copyright 2014-2026 FIT2CLOUD）
- **版本**：`v4.0`（见 [pyproject.toml](file:///workspace/pyproject.toml)）；运行时常量 `const.VERSION` 在 Docker 构建期由 `sed` 注入（见 [Dockerfile](file:///workspace/Dockerfile)）
- **Python 要求**：`>=3.14`（见 [pyproject.toml](file:///workspace/pyproject.toml)）
- **核心定位**：作为整个堡垒机体系的 **Core（控制中心）**，与一组独立的 **Connector（连接器组件）** 协作，完成资产纳管、身份认证、授权、会话代理、命令审计、回放存储等能力。

### 组件全景

JumpServer 由多个独立项目组合而成，本仓库是其中的 **Core**：

| 项目 | 角色 | 说明 |
|------|------|------|
| **jumpserver（本仓库）** | Core | Django 后端，提供 API、权限、审计、调度 |
| [Lina](https://github.com/jumpserver/lina) | Web UI | 后台管理界面（Vue） |
| [Luna](https://github.com/jumpserver/luna) | Web Terminal | 浏览器内终端 |
| [KoKo](https://github.com/jumpserver/koko) | 字符协议连接器 | SSH/Telnet |
| [Lion](https://github.com/jumpserver/lion) | 图形协议连接器 | RDP/VNC（Guacamole） |
| [Chen](https://github.com/jumpserver/chen) | Web DB | 数据库 Web 客户端 |
| Magnus / Razor / Panda / Nec / Tinker / Facelive | EE 连接器 | DB 代理、RDP 代理、远程应用、人脸识别（部分私有） |
| Client | 桌面客户端 | 客户端程序 |

---

## 2. 技术栈

| 层 | 技术 |
|----|------|
| 语言 | Python 3.14 |
| Web 框架 | Django 5.2 + Django REST Framework 3.16 |
| ASGI/WSGI | Uvicorn / Gunicorn / Daphne（dev）+ Channels（WebSocket） |
| 异步任务 | Celery 5.6 + Kombu，Flower 监控，django-celery-beat 调度 |
| 数据库 | MySQL / PostgreSQL / SQLite（dev）/ VastBase（信创） |
| 缓存/消息 | Redis（缓存、Session、Celery broker、Channels layer、PubSub） |
| 自动化 | ansible-core（fork 版本）+ ansible-runner（fork 版本）+ ansible-builder |
| 搜索引擎 | Elasticsearch 7/8（命令/操作日志可选后端） |
| API 文档 | drf-spectacular + drf-spectacular-sidecar（Swagger/ReDoc） |
| 认证 | python-ldap / ldap3 / django-auth-ldap / python3-saml / fido2（Passkey）/ django-cas-ng / django-radius / django-oauth-toolkit |
| 密码学 | cryptography / pycryptodome / gmssl（国密 SM3/SM4）/ bcrypt / paramiko |
| 云/对象存储 | boto3(S3) / oss2(阿里云) / esdk-obs(华为云) / azure-storage-blob / qingcloud-sdk / 各种云 SDK（xpack 组） |
| Vault | hvac（HashiCorp）/ Azure Key Vault / AWS Secrets Manager / 本地 |
| AI | openai（Chat AI 可选） |
| 其他 | geoip2 / ipip-ipdb（IP 库）、polib（i18n）、pydantic、httpx、premailer 等 |

> 完整依赖见 [pyproject.toml](file:///workspace/pyproject.toml) 与 [uv.lock](file:///workspace/uv.lock)。依赖分组：`default`（核心）、`xpack`（企业版/云 SDK）、`dev`（开发工具）。`ansible-core`、`ansible-runner`、`django-cas-ng`、`django-radius`、`redis` 均使用 JumpServer fork 版本（见 `[tool.uv.sources]`）。

---

## 3. 目录结构

```
/workspace
├── apps/                       # Django 项目根（所有应用）
│   ├── jumpserver/             # 项目主配置（settings/urls/wsgi/asgi）
│   ├── common/                 # 共享框架层（base models/api/fields/utils）
│   ├── orgs/                   # 多租户
│   ├── users/                  # 用户与组
│   ├── assets/                 # 资产
│   ├── accounts/               # 特权账号
│   ├── perms/                  # 资产授权
│   ├── authentication/         # 认证与连接令牌
│   ├── terminal/               # 终端组件/会话/命令/回放
│   ├── audits/                 # 审计日志
│   ├── ops/                    # Ansible/Celery 自动化
│   ├── rbac/                   # 角色权限
│   ├── tickets/                # 工单审批
│   ├── acls/                   # 访问控制列表
│   ├── notifications/          # 通知系统
│   ├── settings/               # 系统设置
│   ├── labels/                 # 通用标签
│   ├── reports/                # 报表
│   ├── templates/ static/      # 模板与静态资源
│   └── manage.py               # Django 入口
├── docs/                       # 文档（本文件所在）
├── readmes/                    # 多语言 README
├── requirements/               # 系统/Python 依赖打包脚本
├── ui/                         # 前端占位（实际前端在独立仓库 Lina/Luna）
├── utils/                      # 运维脚本（迁移、假数据、playbooks 等）
├── .github/                    # CI 工作流、Issue 模板
├── jms                         # 服务控制脚本（start/stop/restart/status）
├── entrypoint.sh               # Docker 入口
├── Dockerfile / Dockerfile-base / Dockerfile-ee
├── config_example.yml          # 配置文件示例
├── pyproject.toml / uv.lock    # 依赖管理（uv）
└── README.md
```

每个 `apps/<app>/` 内部的典型布局：

```
apps/<app>/
├── apps.py                 # AppConfig，ready() 中挂载 signal_handlers
├── models.py 或 models/    # 数据模型
├── api.py 或 api/          # ViewSet
├── serializers.py 或 serializers/
├── urls.py 或 urls/        # 路由（api_urls.py / view_urls.py / ws_urls.py）
├── signal_handlers.py 或 signal_handlers/
├── tasks.py                # Celery 任务
├── const.py                # 常量与枚举
├── filters.py              # django-filter
├── notifications.py        # 通知消息类型
├── migrations/             # 数据库迁移
└── tests.py 或 tests/
```

---

## 4. 整体架构

### 4.1 分层架构

```
┌──────────────────────────────────────────────────────────────┐
│                    前端（Lina / Luna / Client）               │
└──────────────┬─────────────────────────────────┬─────────────┘
               │ REST API (/api/v1/*)            │ WebSocket
┌──────────────▼─────────────────────────────────▼─────────────┐
│                    JumpServer Core（本仓库）                  │
│  ┌────────────────────────────────────────────────────────┐  │
│  │  表现层：DRF ViewSet + Swagger + Channels WS            │  │
│  ├────────────────────────────────────────────────────────┤  │
│  │  业务层：users / assets / accounts / perms / tickets /  │  │
│  │          authentication / rbac / acls / audits / ops    │  │
│  ├────────────────────────────────────────────────────────┤  │
│  │  框架层：common（base models/fields/api mixin/perms/    │  │
│  │          tree/cache/signals）+ orgs（多租户隔离）        │  │
│  ├────────────────────────────────────────────────────────┤  │
│  │  基础设施：Django ORM + Celery + Redis + Ansible        │  │
│  └────────────────────────────────────────────────────────┘  │
└──┬──────────────────┬───────────────────┬────────────────────┘
   │ Bootstrap Token  │ AccessKey 签名     │
┌──▼────────┐  ┌──────▼──────┐  ┌────────▼─────────┐
│  KoKo     │  │  Lion       │  │  Chen / Magnus   │  ... 连接器组件
│ (SSH)     │  │  (RDP/VNC)  │  │  (DB)            │
└───────────┘  └─────────────┘  └──────────────────┘
```

### 4.2 Core 与 Connector 的协作模型

JumpServer Core 是中心权威；连接器（koko/lion/chen/magnus/razor 等）是独立进程/服务。它们通过 **Bootstrap Token 注册 → AccessKey 签名认证 → 心跳拉取任务 → 上报会话/命令/回放** 的流程协作（详见 [§7.1](#71-连接器注册与心跳)）。

### 4.3 多租户隔离

几乎所有业务模型混入 `OrgModelMixin`（来自 [apps/orgs/mixins/models.py](file:///workspace/apps/orgs/mixins/models.py)），带 `org_id` 字段；`OrgMiddleware` 在每请求设置当前组织上下文；查询时 `OrgManager` 自动按 `org_id` 过滤。内置组织：`GLOBAL`、`DEFAULT`、`SYSTEM`。

### 4.4 请求-响应管道

中间件链见 [apps/jumpserver/settings/base.py](file:///workspace/apps/jumpserver/settings/base.py) `MIDDLEWARE`，关键自定义中间件：

- `StartMiddleware` / `EndMiddleware` — 管道边界
- `CsrfCheckMiddleware` — 替代 Django 默认 CSRF，支持 SSO 等场景
- `TimezoneMiddleware` — 按用户设置时区
- `DemoMiddleware` — 演示模式拦截
- `RequestMiddleware` — 将 request 注入 thread_local（供 `created_by` 自动填充）
- `RefererCheckMiddleware` — Referer 校验
- `SQLCountMiddleware` — SQL 计数
- `orgs.middleware.OrgMiddleware` — 组织上下文
- `authentication.middleware.MFAMiddleware` / `ThirdPartyLoginMiddleware` / `SessionCookieMiddleware` — 认证相关
- `simple_history.middleware.HistoryRequestMiddleware` — 历史记录
- `SafeRedirectMiddleware` — 安全跳转

---

## 5. 配置系统

JumpServer 的配置分三类（见 [apps/jumpserver/conf.py](file:///workspace/apps/jumpserver/conf.py) 顶部注释）：

1. **Django settings**：纯框架配置，写在 [apps/jumpserver/settings/](file:///workspace/apps/jumpserver/settings/) 下。
2. **程序内部常量**：用户不需要改的，写在 settings 中。
3. **用户可配置项**：写在 `config.yml`（或 `config.py`）中，由 `ConfigManager` 加载。

### 5.1 配置加载流程

入口 [apps/jumpserver/const.py](file:///workspace/apps/jumpserver/const.py)：

```python
CONFIG = ConfigManager.load_user_config()
```

`ConfigManager.load_user_config()`（[conf.py:1139](file:///workspace/apps/jumpserver/conf.py)）按优先级加载：

1. `config.py`（`from config import config as c`）— `load_from_object()`
2. `config.yml` / `config.yaml` — `load_from_yaml()`
3. 都没有则抛 `ImportError`，提示 `cp config_example.yml config.yml`

加载后调用 `config.compatible()` 做兼容处理（OpenID 旧 Keycloak 配置转换、Redis 密码 URL 编码）。

### 5.2 配置取值优先级

`Config.get(item)`（[conf.py:961](file:///workspace/apps/jumpserver/conf.py)）：

```
config 文件值  >  环境变量  >  defaults 默认值  >  old_config_map 兼容映射
```

- 支持环境变量覆盖（`get_from_env`）
- 支持 SM4 加密敏感字段（`SECRET_KEY`/`DB_PASSWORD`/`REDIS_PASSWORD`），由环境变量 `SECRET_ENCRYPT_KEY` 触发（`ConfigCrypto`）
- 自动类型转换：按 `defaults` 中默认值的类型转换（bool/list/dict/str/int）

### 5.3 settings 模块组成

[apps/jumpserver/settings/__init__.py](file:///workspace/apps/jumpserver/settings/__init__.py) 依次聚合：

| 文件 | 内容 |
|------|------|
| [base.py](file:///workspace/apps/jumpserver/settings/base.py) | INSTALLED_APPS、MIDDLEWARE、DATABASES、CACHES(Redis)、STATIC/MEDIA、SESSION、AUTH_USER_MODEL 等 |
| [logging.py](file:///workspace/apps/jumpserver/settings/logging.py) | 日志配置 |
| [libs.py](file:///workspace/apps/jumpserver/settings/libs.py) | 第三方库配置（DRF、channels、oauth2_provider、simple_history 等） |
| [auth.py](file:///workspace/apps/jumpserver/settings/auth.py) | AUTHENTICATION_BACKENDS、LDAP/CAS/SAML/OIDC 等 |
| [custom.py](file:///workspace/apps/jumpserver/settings/custom.py) | 自定义业务配置项（安全策略、Terminal 配置等） |
| [_xpack.py](file:///workspace/apps/jumpserver/settings/_xpack.py) | 企业版（XPACK）配置，存在 `apps/xpack` 目录时启用 |

### 5.4 关键配置项示例

见 [config_example.yml](file:///workspace/config_example.yml)。核心项：

- `SECRET_KEY`、`BOOTSTRAP_TOKEN` — 加密与组件注册
- `DB_ENGINE`（mysql/postgresql/sqlite3/vastbase）、`DB_HOST/PORT/USER/PASSWORD/NAME`
- `REDIS_HOST/PORT/PASSWORD`，支持 Sentinel（`REDIS_SENTINEL_HOSTS`）与 SSL
- `HTTP_LISTEN_PORT`（8080）、`WS_LISTEN_PORT`（8070）
- `AUTH_LDAP/AUTH_OPENID/AUTH_CAS/AUTH_SAML2/AUTH_OAUTH2/AUTH_RADIUS/AUTH_WECOM/AUTH_DINGTALK/AUTH_FEISHU/AUTH_LARK/AUTH_SLACK/AUTH_PASSKEY` — 第三方认证
- `SECURITY_*` — 安全策略（MFA、密码强度、登录限制、IP 黑白名单、水印等）
- `TERMINAL_*` — 连接器配置
- `VAULT_*` — 凭据 Vault 后端
- `MCP_ENABLED`、`CHAT_AI_ENABLED`、`FACE_RECOGNITION_ENABLED` — 可选特性

---

## 6. 应用模块详解

`INSTALLED_APPS` 顺序见 [base.py:126](file:///workspace/apps/jumpserver/settings/base.py)。注意：`common.apps.CommonConfig` 必须放在内置 app 最后（其 `ready()` 发送 `django_ready` 信号）；`simple_history` 必须放最后。

### 6.1 common — 共享框架层

> 路径：[apps/common](file:///workspace/apps/common)。所有其他 app 的基础设施。

#### 6.1.1 基础模型（[db/models.py](file:///workspace/apps/common/db/models.py)、[db/mixins.py](file:///workspace/apps/common/db/mixins.py)）

| 类 | 作用 |
|----|------|
| `ChoicesMixin` | 给枚举类增加 `get_label(value)` |
| `BaseCreateUpdateModel` | 审计字段：`created_by`/`updated_by`/`date_created`/`date_updated`/`comment` |
| `JMSBaseModel` | `BaseCreateUpdateModel` + UUID 主键（`id = UUIDField`），非组织级模型的基类 |
| `NoDeleteModelMixin` + `NoDeleteManager` + `NoDeleteQuerySet` | 软删除（`is_discard`/`discard_time`），`delete()` 改为 update |
| `MultiTableChildQueryset` | 多表继承的 `bulk_create` 支持 |
| `CASCADE_SIGNAL_SKIP` | 级联删除时标记 `OP_LOG_SKIP_SIGNAL`，避免操作日志噪声 |

> 注：`OrgModelMixin` / `JMSOrgBaseModel` 在 [apps/orgs/mixins/models.py](file:///workspace/apps/orgs/mixins/models.py)，组织级模型混入。

#### 6.1.2 自定义字段（[db/fields.py](file:///workspace/apps/common/db/fields.py)）

- **JSON 字段族**：`JsonCharField`/`JsonTextField`/`JsonListCharField`/`JsonDictCharField` 等，Char/Text 列透明序列化 JSON
- **加密字段**：`EncryptCharField`/`EncryptTextField`/`EncryptJsonDictTextField`，写入加密、读取解密（`Encryptor`）
- `PortField`（0–65535）、`PortRangeField`
- `TreeChoices` / `BitChoices` — 可渲染为树/位掩码的选择类（权限/选项树用，12 位 max 4095）
- **`JSONManyToManyField`** — 以 JSON 存储的虚拟 M2M，值形如 `{'type':'all'|'ids'|'attrs', ...}`，`RelatedManager` 解析为 queryset（支持 `exact`/`contains`/`regex`/`ip_in`/`m2m_all` 等匹配）。广泛用于动态权限范围。

#### 6.1.3 基础 API ViewSet（[api/mixin.py](file:///workspace/apps/common/api/mixin.py)、[api/generic.py](file:///workspace/apps/common/api/generic.py)）

| 类 | 作用 |
|----|------|
| `SerializerMixin` | 按 action 选择 serializer（`serializer_classes = {'list':..., 'retrieve':..., 'display':..., 'default':...}`） |
| `QuerySetMixin` | 支持 UUID/数字/slug 多种 lookup；`limit_queryset_if_no_page` 限制无分页 list；`setup_eager_loading` 避免 N+1；分页后重取 queryset 保持组织作用域 |
| `ExtraFilterFieldsMixin` | 追加自定义 filter backend（`CustomFilterBackend`/`IDSpmFilterBackend`/`LabelFilterBackend` 等） |
| `OrderingFielderFieldsMixin` | 从模型字段自动推导 `ordering_fields` |
| `RelationMixin` | M2M through 表端点，触发 `m2m_changed` 信号 |
| `CommonApiMixin` | 聚合上述全部 + `RenderToJsonMixin` + `PaginatedResponseMixin` |
| `JMSGenericViewSet` | `CommonApiMixin` + `GenericViewSet` |
| `JMSModelViewSet` | `CommonApiMixin` + `ModelViewSet`（CRUD 默认选择） |
| `JMSBulkModelViewSet` | + `AllowBulkDestroyMixin` + `BulkModelViewSet`（批量增删改） |
| `JMSBulkRelationModelViewSet` | + `RelationMixin` |
| `AsyncApiMixin`（[api/patch.py](file:///workspace/apps/common/api/patch.py)） | 昂贵 GET 异步化：首次返回 `{status:'running'}`，后台线程计算并缓存 600s，`?refresh=1` 失效 |

#### 6.1.4 权限基类（[permissions.py](file:///workspace/apps/common/permissions.py)）

| 类 | 作用 |
|----|------|
| `IsValidUser` | `IsAuthenticated` + `user.is_valid`（active + 未过期），多数端点基线 |
| `OnlySuperUser` / `OnlyAdminSuperUser` | 超级管理员 / 内置 admin |
| `IsServiceAccount` | 服务账号（连接器） |
| `WithBootstrapToken` | 组件注册：`hmac.compare_digest` 校验 `BOOTSTRAP_TOKEN`，受 `SECURITY_SERVICE_ACCOUNT_REGISTRATION` 控制 |
| `ServiceAccountSignaturePermission` | 校验 `X-JMS-SVC: Sign <ak_id>:<encrypted_timestamp>`（AES-ECB，30s 容差） |
| `IsValidLicense` / `IsValidLicenseForWriteAction` | 企业版 License 校验 |
| `IsOwnerOrAdminWritable` | 对象级：超管全权，写操作限创建者 |
| `AllowBulkDestroyMixin` | 仅当带 `id`/`ptr_id` 过滤条件才允许批量删除（安全护栏） |

#### 6.1.5 树工具（[tree.py](file:///workspace/apps/common/tree.py)）

- `TreeNode` — 节点对象（`id`/`name`/`title`/`pId`/`isParent`/`open`/`iconSkin`/`meta`/`checked`），`root()` 类方法返回根（`id='#'`），支持父子遍历与排序
- `Tree` — 节点容器，`add_node(node, parent)` 防环校验，`get_nodes()` 排序输出
- `TreeNodeSerializer` — DRF 序列化器，匹配前端 z-tree 载荷

被 `orgs/models.py`、`rbac/tree.py`、`assets/api/tree.py`、`assets/models/node.py` 等使用。

#### 6.1.6 信号与启动钩子（[signals.py](file:///workspace/apps/common/signals.py)、[signal_handlers.py](file:///workspace/apps/common/signal_handlers.py)、[apps.py](file:///workspace/apps/common/apps.py)）

- **`django_ready = Signal()`** — JumpServer 主启动钩子，由 `CommonConfig.ready()` 发送（但 `migrate`/`makemigrations`/`compilemessages`/`check`/`upgrade_db`/`collect_static` 等管理命令跳过）
- `on_create_set_created_by` / `on_update_set_updated_by`（`pre_save`）— 自动填充审计字段（从 thread_local 取 request.user，否则 `'System'`）
- `on_request_finished_release_local` — 请求结束清理 thread_local
- `check_migrations_file_prefix_conflict`（`django_ready`，dev）— 检查迁移文件前缀冲突
- `clear_response_cache`（`django_ready`）— 清理 `views.decorators.cache.cache_page:*` 缓存

#### 6.1.7 缓存框架（[cache.py](file:////workspace/apps/common/cache.py)）

声明式 Redis 计算缓存：

- `CacheFieldBase` / `CharField` / `IntegerField` — 声明可缓存字段（queryset 计数或 `compute_<field>` 方法）
- `CacheType` 元类 — 扫描类属性，替换为 `CacheValueDesc` 描述符
- `Cache` 基类 — Redis hash 存储（key `cache.<module>.<ClassName>.<key_suffix>`），`data` 懒加载，`ComputeLock` 防雪崩，`refresh`/`expire`/`reload`
- `RedisChannelLayer` — 覆写 `channels_redis`，用 Lua 脚本处理云 Redis 限制

#### 6.1.8 通用 Celery 任务（[tasks.py](file:///workspace/apps/common/tasks.py)）

- `send_mail_async` — 异步发邮件，按收件人语言激活并发送
- `send_mail_attachment_async` — 带附件邮件，发送后删除临时文件
- `upload_backup_to_obj_storage` — 上传备份到对象存储
- `get_email_connection` — SMTP/Exchange 后端选择

#### 6.1.9 其他工具

- [utils/](file:///workspace/apps/common/utils)：`common.py`/`django.py`/`file.py`/`http.py`/`encode.py`/`lock.py`（`DistributedLock`）/`random.py`/`timezone.py`/`verify_code.py`/`yml.py`/`zip.py`/`ip/`（GeoIP/IPIP）/`crypto/` 等
- [storage/](file:///workspace/apps/common/storage)：`base.py`/`ftp_file.py`/`replay.py` — 文件存储后端
- [plugins/es.py](file:///workspace/apps/common/plugins/es.py) — ES 客户端
- [hashers/sm3.py](file:///workspace/apps/common/hashers/sm3.py) — 国密 SM3 密码哈希（`PBKDF2SM3PasswordHasher`）
- [drf/](file:///workspace/apps/common/drf)：`exc_handlers.py`/`filters.py`/`metadata.py`/`routers.py`/`throttling.py` — DRF 增强
- [auth/signature.py](file:///workspace/apps/common/auth/signature.py) — AccessKey 签名认证

---

### 6.2 orgs — 多租户核心

> 路径：[apps/orgs](file:///workspace/apps/orgs)

**职责**：多租户隔离核心。拥有 `Organization` 模型（内置 `GLOBAL`/`DEFAULT`/`SYSTEM`），通过中间件与 context processor 实现组织上下文，维护组织映射缓存与组织资源统计缓存。

- **AppConfig**：`OrgsConfig`
- **关键模型**：`Organization`（[models.py](file:///workspace/apps/orgs/models.py)，含 `OrgRoleMixin`）
- **URL 命名空间**：`orgs`（[urls/api_urls.py](file:///workspace/apps/orgs/urls)）
- **关键机制**：
  - `OrgMiddleware` 每请求设置当前组织；`OrgManager`/`OrgModelMixin` 自动按 `org_id` 过滤
  - `OrgsConfig.ready()` 导入 `signal_handlers.common` 与 `signal_handlers.cache`
    - `common.py`：组织增删改 → 失效组织映射（Redis PubSub `fm.orgs_mapping`）+ 同步组织根 `Node`；`User` 创建 → 加入默认组织与 Default 组；用户离开组织 → 清理其在该组织的 `UserGroup`/`AssetPermission`
    - `cache.py`：监听 `User`/`UserGroup`/`Node`/`Zone`/`Account`/`Asset`/`AssetPermission`/`RoleBinding`/`Session` 的 `post_save`/`post_delete` → 刷新 `OrgResourceStatisticsCache`
- **关键类**：`Organization`、`OrgRoleMixin`、`OrgModelMixin`、`JMSOrgBaseModel`、`OrgManager`、`OrgMiddleware`、`OrgResourceStatisticsCache`

---

### 6.3 users — 用户与组

> 路径：[apps/users](file:///workspace/apps/users)

**职责**：用户身份与组管理、用户来源（source）、密码历史、偏好设置；外部认证（LDAP/CAS/SAML2/OAuth2/OIDC/RADIUS）用户供给。

- **AppConfig**：`UsersConfig`
- **关键模型**：`User`、`UserGroup`、`UserPasswordHistory`、`Preference`（[models/](file:///workspace/apps/users/models) 包：`group`/`preference`/`user`/`utils`）
- **`AUTH_USER_MODEL = 'users.User'`**（[base.py:372](file:///workspace/apps/jumpserver/settings/base.py)）
- **URL 命名空间**：`users`
- **signal_handlers.py**：
  - `User.post_save` → 设置 `email_lookup`（HMAC）、记录 `UserPasswordHistory`、分配默认系统角色 / 同步超管角色
  - `post_user_create`/`post_user_update` → email lookup
  - CAS/SAML2/OAuth2/OIDC/RADIUS/LDAP 认证信号 → 设置用户 source、绑定组织角色与组
  - `user_logged_out` → 移除 session + 删除 `UserSession`
  - `setting_changed`（AUTH_*）与 `django_ready` → 重置 `User._source_choices`
- **关键类**：`User`、`UserGroup`、`UserPasswordHistory`、`Preference`

---

### 6.4 assets — 资产盘点

> 路径：[apps/assets](file:///workspace/apps/assets)

**职责**：资产清单管理。按类别纳管资产（Host/Database/Device/Web/Cloud/GPT/DS/Custom）、节点树、平台与协议、网关、区域、标签、收藏、资产自动化。

- **AppConfig**：`AssetsConfig`
- **关键模型**：`Asset`、`Node`、`Platform`、`PlatformProtocol`、`PlatformAutomation`、`Label`、`Gateway`、`Zone`、`CommandFilter`/`CommandFilterRule`、`FavoriteAsset`/`FavoriteFolder`、`MyAsset`、`AbsConnectivity`（[models/](file:///workspace/apps/assets/models) 包：`base`/`platform`/`asset`/`label`/`gateway`/`zone`/`node`/`favorite_asset`/`automations`/`my_asset`/`cmd_filter`）
- **URL 命名空间**：`assets`（[urls/api_urls.py](file:///workspace/apps/assets/urls/api_urls.py)、`views_urls.py`）
- **signal_handlers/**（`asset.py`/`node_assets_amount.py`/`node_assets_mapping.py`）：
  - `Node.pre_save` → 计算 `parent_key`
  - `Asset.post_save`(created) → 确保归属节点、测试连通性、采集 facts（按组织延迟合并）
  - `Asset.pre_delete`/`post_delete` → 重发节点关系 `m2m_changed`
  - 重发 `Host`/`Database`/`Device`/`Web`/`Cloud` 的 save 信号到 `Asset`
  - 维护节点资产数量与节点↔资产映射
- **tasks/**：`automation.py`（自动化执行）、`ping.py`（连通性）、`ping_gateway.py`、`gather_facts.py`、`nodes_amount.py`、`common.py`、`utils.py`
- **const/**：按类别拆分常量（`host.py`/`database.py`/`device.py`/`web.py`/`cloud.py`/`gpt.py`/`ds.py`/`custom.py`/`platform.py`/`protocol.py`/`automation.py`/`types.py`）

---

### 6.5 accounts — 特权账号管理

> 路径：[apps/accounts](file:///workspace/apps/accounts)

**职责**：PAM 凭据管理 — 资产上的账号、账号模板、虚拟/采集账号、自动化（推送/改密/采集/校验/备份）、Vault 集成、账号风险。

- **AppConfig**：`AccountsConfig`
- **关键模型**：`BaseAccount`（抽象，`VaultModelMixin` + `JMSOrgBaseModel`）、`Account`（`BaseAccount` + `AbsConnectivity` + `LabeledMixin` + `JSONFilterMixin`）、`AccountTemplate`、`VirtualAccount`、`IntegrationApplication`、自动化模型（`BackupAccountAutomation`/`ChangeSecretAutomation` 等）、`AccountRisk`、`ChangeSecretRecord`
- **URL 命名空间**：`accounts`
- **关键设计**：
  - `BaseAccount`：`secret_type`（password/ssh_key/access_key/token/api_key），密钥用 `EncryptTextField`，`SecretWithRandomMixin` 提供随机生成策略
  - `Account`：唯一约束 `(username, asset, secret_type)` 与 `(name, asset)`；`AccountHistoricalRecords`（simple_history）在 `id/_secret/secret_type/version` 变更时升 `version`；`alias` 属性返回 `@` 前缀用户名（虚拟账号）或 id —— 这正是 `AssetPermission.accounts` 引用的 token
  - `VirtualAccount`：代表 `@INPUT`/`@USER`/`@ANON` 别名（非 `@ALL`）；`get_special_account(alias, user, asset, ...)` 工厂在权限展开时构造临时 Account
  - Vault 后端：`accounts/backends/`（local/aws/azure/hcp），`VaultManagerMixin`/`VaultQuerySetMixin`
- **signal_handlers.py**：`Account.post_save`(created) → 模板来源则推送账号 + 创建活动；`VaultSignalHandler` 在 `Account`/`AccountTemplate`/`Account.history` 的 save/delete 时同步到 Vault；`django_ready` 订阅 `refresh_vault` PubSub
- **const**：`AliasAccount`（`@ALL`/`@INPUT`/`@USER`/`@ANON`/`@SPEC`）、`SecretType`、`Source`（LOCAL/DISCOVERY/TEMPLATE）

---

### 6.6 perms — 资产访问授权

> 路径：[apps/perms](file:///workspace/apps/perms)

**职责**：资产访问权限 — `AssetPermission` 授权用户/组访问资产/节点/账号；权限过期检查；用户授权树缓存与失效。

- **AppConfig**：`PermsConfig`
- **关键模型**：`AssetPermission`、`PermNode`（`Node` 只读代理）、`PermedAsset`、`PermedAccount`、`UserAssetGrantedTreeNodeRelation`（[models/](file:///workspace/apps/perms/models) 包：`asset_permission`/`perm_node`）
- **URL 命名空间**：`perms`（含 `asset_permission.py`、`user_permission.py`）
- **核心模型 `AssetPermission`**（`JMSOrgBaseModel` + `LabeledMixin`）：
  - M2M：`users`→`User`、`user_groups`→`UserGroup`、`assets`→`Asset`、`nodes`→`Node`
  - `accounts`：JSON 列表，支持别名 token（`@ALL`/`@INPUT`/`@USER`/`@ANON`/`@SPEC`，`!account` 排除）
  - `actions`：`ActionChoices` 位掩码（`connect=1`/`upload=2`/`download=3`/`copy=4`/`paste=5`/`delete=6`/`share=7`）
  - `date_start`/`date_expired`/`is_active`、`from_ticket`
  - `AssetPermissionQuerySet`：`.valid()`/`.active()`/`.invalid()`/`.get_expired_permissions()`
- **权限计算**（[utils/permission.py](file:///workspace/apps/perms/utils/permission.py) `AssetPermissionUtil`）：
  - `get_permissions_for_user`、`get_permissions_for_user_asset`（用户×资产交集，核心查询）
  - `PermAssetDetailUtil`（[utils/asset_perm.py](file:///workspace/apps/perms/utils/asset_perm.py)）：展开每个 (user,asset) 的账号，按别名合并 actions（位或），展开 `@ALL`，应用 `!account` 排除
- **授权树缓存**（[utils/user_perm_tree.py](file:///workspace/apps/perms/utils/user_perm_tree.py)）：
  - `UserAssetGrantedTreeNodeRelation`：每用户每节点一行，`node_from`（`granted`/`asset`/`child`）
  - `UserPermTreeBuildUtil.rebuild_user_perm_tree()` — 计算直接授权节点、含授权资产的节点、祖先补全节点
  - `UserPermTreeRefreshUtil` — 按组织比对 Redis SET 决定是否重建，`UserGrantedTreeRebuildLock` 防并发，小资产量同步/否则异步
  - `UserPermTreeExpireUtil` — 失效（`expire_perm_tree_for_perms`/`for_users_orgs`/`for_nodes_assets`/`for_all_user`）
  - `UserPermAssetUtil`/`UserPermNodeUtil`（[utils/user_perm.py](file:///workspace/apps/perms/utils/user_perm.py)）— 读取物化树服务"我的资产"视图
- **signal_handlers/asset_permission.py**：`AssetPermission` pre/post save/delete + M2M（nodes/assets/users/user_groups）变化 → `UserPermTreeExpireUtil` 失效；反向 M2M 抛 `M2MReverseNotAllowed`
- **tasks.py**：`check_asset_permission_expired`（按 `PERM_EXPIRED_CHECK_PERIODIC`）、`check_asset_permission_will_expired`（每日 10:00 通知）

> 详见 [§7.3](#73-权限树缓存机制)。

---

### 6.7 authentication — 认证与连接授权

> 路径：[apps/authentication](file:///workspace/apps/authentication)

**职责**：认证与连接授权 — 认证后端（token/ldap/pubkey/sso/oauth2/oidc/saml2/cas/radius/passkey）、MFA、资产访问连接令牌、AccessKey/SSHKey、临时令牌。

- **AppConfig**：`AuthenticationConfig`
- **关键模型**：`AccessKey`、`SSHKey`、`ConnectionToken`、`SuperConnectionToken`、`AdminConnectionToken`、`TempToken`、`PrivateToken`、`SSOToken`、`Passkey`（[models/](file:///workspace/apps/authentication/models) 包：`access_key`/`connection_token`/`private_token`/`ssh_key`/`sso_token`/`temp_token`，passkey 在 `backends/passkey/models`）
- **URL 命名空间**：`authentication`（`api_urls.py`、`view_urls.py`），子命名空间 `ukey`
- **认证后端**（`AUTHENTICATION_BACKENDS` 见 [settings/auth.py](file:///workspace/apps/jumpserver/settings/auth.py)）：支持密码、LDAP、PublicKey(SSO)、OAuth2、OIDC、SAML2、CAS、RADIUS、Passkey、临时 Token 等
- **signal_handlers.py**：`user_logged_in` → 失效 RBAC 权限缓存、设置 MFA/第三方必填 session 标志、单机登录踢出其他 session、更新 `UserSession`、设置用户语言；`backend_auth_failed` → 转发 `post_auth_failed`；导入 `backends.oauth2_provider.signal_handlers`
- **关键类**：各类 Token 模型、`AccessKey`（连接器服务账号凭据）、`ConnectionToken`（用户访问资产的连接令牌）

---

### 6.8 terminal — 终端组件与会话生命周期

> 路径：[apps/terminal](file:///workspace/apps/terminal)

**职责**：终端/组件管理与会话生命周期 — 终端（koko 等）、会话、命令、回放、命令/回放存储、端点、Applet、虚拟应用、会话共享。

- **AppConfig**：`TerminalConfig`
- **关键模型**（[models/](file:///workspace/apps/terminal/models) 包，按子包组织）：
  - `component/`：`Terminal`、`Status`、`Task`、`CommandStorage`、`ReplayStorage`、`Endpoint`、`EndpointRule`
  - `session/`：`Session`、`Command`、`SessionReplay`、`SessionSharing`、`SessionJoinRecord`
  - `applet/`：`Applet`、`AppletPublication`、`AppletHost`、`AppletHostDeployment`
  - `virtualapp/`：`VirtualApp`、`VirtualAppPublication`、`AppProvider`
- **URL 命名空间**：`terminal`（`api_urls.py`、`ws_urls.py`）
- **`Terminal` 模型**（[models/component/terminal.py](file:///workspace/apps/terminal/models/component/terminal.py)）：`name`/`type`/`remote_addr`/`command_storage`/`replay_storage`/`user`(OneToOne 服务账号)/`is_deleted`。`is_active`/`is_alive`（缓存 `TERMINAL_ALIVE_{id}`，3 分钟 TTL）；`config` 属性聚合所有 `TERMINAL*` 设置返回给连接器
- **`TerminalType`**（[const.py](file:///workspace/apps/terminal/const.py)）：`koko`/`guacamole`/`omnidb`/`xrdp`/`lion`/`core`/`celery`/`magnus`/`razor`/`tinker`/`video_worker`/`chen`/`kael`/`panda`/`nec`/`facelive`
- **session_lifecycle.py**：定义生命周期事件类（`AssetConnectSuccess`/`AssetConnectFinished`/`UserCreateShareLink`/`UserJoinSession`/.../`ReplayUploadSuccess`），每个产生 `audits.ActivityLog`（type `session_log`）
- **tasks.py**：`clean_orphan_session`（每 10 分钟清理孤儿会话）、`delete_terminal_status_period`（每小时清 7 天前状态）、`check_command_replay_storage_connectivity`（每日校验存储）、`upload_session_replay_to_external_storage`（回放外存）
- **signal_handlers/**：`session.py`（`Session.pre_save` 计算 cmd_amount + 更新账号 last_login；`post_save` 完成时解锁）、`terminal.py`（`Task.post_save` 经 Redis PubSub `fm.component_event_chan` 推送；任务完成时锁/解锁 Session）、`applet.py`/`session_sharing.py`/`virtualapp.py`
- **关键类**：`Terminal`、`Session`、`Command`、`SessionReplay`、`Task`、`Status`、`CommandStorage`、`ReplayStorage`、`Endpoint`

> 连接器注册、心跳、会话上报、命令/回放存储详见 [§7.1](#71-连接器注册与心跳)、[§7.2](#72-会话生命周期)。

---

### 6.9 audits — 审计日志

> 路径：[apps/audits](file:///workspace/apps/audits)

**职责**：审计日志 — 操作日志（监控模型的 CRUD）、登录日志、活动日志、FTP 日志、改密日志、用户会话、作业日志；可选 syslog 转发；ES 后端支持。

- **AppConfig**：`AuditsConfig`
- **关键模型**（[models.py](file:///workspace/apps/audits/models.py)）：`OperateLog`、`UserLoginLog`、`ActivityLog`、`FTPLog`、`PasswordChangeLog`、`UserSession`、`JobLog`、`IntegrationApplicationLog`
- **URL 命名空间**：`audits`
- **signal_handlers/**（`activity_log`/`login_log`/`operate_log`/`other`）：
  - `operate_log.py`：`pre_save`/`pre_delete` 缓存前态；`post_save` 创建/更新 `OperateLog`；`post_delete` 删除日志；`m2m_changed` 记录 M2M 变更；`django_ready` 填充 `MODELS_NEED_RECORD`
  - `login_log.py`/`activity_log.py`/`other.py` — 登录/活动/其他审计
- **特别**：`AuditsConfig.ready()` 是唯一在 `ready()` 中显式调用 `post_save.connect(...)` 的 app（条件：`settings.SYSLOG_ENABLE` → `on_audits_log_create`）
- **backends/**：`db.py`/`es.py` — 镜像 terminal 命令存储后端模式
- **`ActivityLog`** 桥接 terminal 会话生命周期事件；`OperateLog` 记录回放查看/下载等操作

---

### 6.10 ops — 自动化执行引擎

> 路径：[apps/ops](file:///workspace/apps/ops)

**职责**：运维自动化执行 — AdHoc 任务、Playbook、Ansible Job、Celery 任务注册与执行追踪、变量、定时任务。

- **AppConfig**：`OpsConfig`（`ready()` 先设置当前组织为 `Organization.root()`，再导入 celery.signal_handler/signal_handlers/notifications/tasks）
- **关键模型**（[models/](file:///workspace/apps/ops/models) 包）：`AdHoc`、`Playbook`、`Job`、`JobExecution`、`BaseAnsibleJob`、`BaseAnsibleExecution`、`CeleryTask`、`CeleryTaskExecution`、`Variable`、`JMSPermedInventory`
- **URL 命名空间**：`ops`（`api_urls.py`/`view_urls.py`/`ws_urls.py`）
- **ansible/**（[apps/ops/ansible](file:///workspace/apps/ops/ansible)）：`runner.py`（执行器）、`inventory.py`（动态主机清单）、`callback.py`（回调）、`docker.py`（容器执行）、`interface.py`、`cleaner.py`、`utils.py`
- **celery/**（[apps/ops/celery](file:///workspace/apps/ops/celery)）：`decorator.py`、`heatbeat.py`、`logger.py`、`signal_handler.py`、`utils.py`、`const.py`
- **signal_handlers.py**：`Job.pre_save` → 升版本；celery `worker_ready` → 同步已注册 `CeleryTask` 到 DB；`before_task_publish` → 注入语言 + org_id；`task_prerun` → 标记 `CeleryTaskExecution` RUNNING；`task_postrun` → 标记完成；`after_task_publish` → 创建 `CeleryTaskExecution`；`django_ready` → 检查注册任务 + 订阅 `fm.job_execution_stop` PubSub 以 kill 进程
- **关键类**：`AdHoc`、`Playbook`、`Job`/`JobExecution`、`CeleryTask`/`CeleryTaskExecution`、Ansible Runner 系列

---

### 6.11 rbac — 角色权限控制

> 路径：[apps/rbac](file:///workspace/apps/rbac)

**职责**：RBAC — 权限、角色（系统/组织）、角色绑定、菜单、内置角色、RBAC 权限后端。

- **AppConfig**：`RBACConfig`
- **关键模型**（[models/](file:///workspace/apps/rbac/models) 包）：`Permission`、`ContentType`（自定义）、`Role`、`SystemRole`、`OrgRole`、`RoleBinding`、`OrgRoleBinding`、`SystemRoleBinding`、`MenuPermission`
- **URL 命名空间**：`rbac`
- **其他**：[backends.py](file:///workspace/apps/rbac/backends.py)（RBAC 权限后端）、[builtin.py](file:///workspace/apps/rbac/builtin.py)（内置角色定义）、[tree.py](file:///workspace/apps/rbac/tree.py)（权限树）、[utils.py](file:///workspace/apps/rbac/utils.py)
- **signal_handlers.py**：`post_migrate`（最后一个 app）→ `BuiltinRole.sync_to_db()`；`SystemRole`/`OrgRole` post_save + permissions M2M 变化 + `SystemRoleBinding`/`OrgRoleBinding` post_save/post_delete → `User.expire_users_rbac_perms_cache()`
- **关键类**：`Permission`、`Role`、`SystemRole`/`OrgRole`、`RoleBinding`、`MenuPermission`

---

### 6.12 tickets — 工单审批

> 路径：[apps/tickets](file:///workspace/apps/tickets)

**职责**：工单/工作流系统 — 工单、工单流程（审批规则）、评论、工单↔会话关联；apply-asset / login-confirm / command-confirm / login-asset-confirm 工单处理器。

- **AppConfig**：`TicketsConfig`
- **关键模型**（[models/](file:///workspace/apps/tickets/models) 包）：
  - `flow.py`：`TicketFlow`（按 `type` 键，`approval_level` 1/2，M2M `rules`）、`ApprovalRule`（`level` + `JSONManyToManyField users` 过滤规格）
  - `ticket/general.py`：`Ticket`（`StatusMixin` + `JMSBaseModel`）、`TicketStep`、`TicketAssignee`、`SuperTicket`；子类 `ApplyAssetTicket`/`ApplyLoginTicket`/`ApplyLoginAssetTicket`/`ApplyCommandTicket`
  - `comment.py`、`relation.py`（`TicketSession` 关联 `terminal.Session`）
- **URL 命名空间**：`tickets`
- **常量**（[const.py](file:///workspace/apps/tickets/const.py)）：`TicketType`（general/apply_asset/login_confirm/command_confirm/login_asset_confirm）、`TicketState`（all/pending/closed/approved/rejected）、`TicketStatus`（open/closed）、`StepState`/`StepStatus`、`TicketAction`、`TicketLevel`、`TicketApplyAssetScope`
- **状态机 `StatusMixin`**：`open()` → 按 flow 创建 `TicketStep`/`TicketAssignee` → state=pending；`approve(processor)`/`reject(processor)` → 当前 step change_state → 有 next 则激活下一级（`approval_step++`），否则终结；`close()` → 申请人关闭
- **Handler 分发**（[handlers/__init__.py](file:///workspace/apps/tickets/handlers/__init__.py) `get_ticket_handler`）：动态导入 `tickets.handlers.<ticket.type>.Handler`
- **`BaseHandler`**（[handlers/base.py](file:///workspace/apps/tickets/handlers/base.py)）：`on_change_state`/`on_step_state_change` 钩子 + 评论 + 邮件通知
- **apply-asset 授权**（[handlers/apply_asset.py](file:///workspace/apps/tickets/handlers/apply_asset.py)）：最终审批通过 → `_create_asset_permission()` 创建 `AssetPermission`（`from_ticket=True`，PK 复用工单 PK，授权申请人请求的 nodes/assets/accounts/actions/有效期）
- **API**（[api/ticket.py](file:///workspace/apps/tickets/api/ticket.py) `TicketViewSet`）：`open`/`approve`/`reject`/`close`/`bulk` 自定义动作（普通 create/update/destroy 为 `MethodNotAllowed`）
- **关键类**：`Ticket`/`StatusMixin`、`TicketFlow`/`ApprovalRule`、`TicketStep`/`TicketAssignee`、`BaseHandler`、各类型 `Handler`

> 工单授权链路详见 [§7.4](#74-工单审批与授权链路)。

---

### 6.13 acls — 访问控制列表

> 路径：[apps/acls](file:///workspace/apps/acls)

**职责**：访问控制列表 — 登录 ACL、登录资产 ACL、命令过滤 ACL、连接方式 ACL、数据脱敏规则。

- **AppConfig**：`AclsConfig`（**无 `ready()`**，无 signal_handlers）
- **关键模型**（[models/](file:///workspace/apps/acls/models) 包）：`BaseACL`、`UserBaseACL`、`UserAssetAccountBaseACL`、`LoginACL`、`LoginAssetACL`、`CommandFilterACL`、`CommandGroup`、`ConnectMethodACL`、`DataMaskingRule`
- **URL 命名空间**：`acls`
- **api/**：`command_acl.py`/`connect_method.py`/`data_masking.py`/`login_acl.py`/`login_asset_acl.py`/`login_asset_check.py`/`common.py`
- **关键类**：`BaseACL` 及其子类、`CommandGroup`、`DataMaskingRule`

---

### 6.14 notifications — 站内信与通知

> 路径：[apps/notifications](file:///workspace/apps/notifications)

**职责**：通知系统 — 站内信、系统/用户消息订阅、通知后端、WebSocket 推送。

- **AppConfig**：`NotificationsConfig`
- **关键模型**（[models/](file:///workspace/apps/notifications/models) 包）：`MessageContent`、`SiteMessage`、`SystemMsgSubscription`、`UserMsgSubscription`
- **URL 命名空间**：`notifications`（`api_urls.py`/`ws_urls.py`）
- **signal_handlers.py**：`MessageContent.post_save`(created) → 经 Redis PubSub `notifications.SiteMessageCome` 发布站内信；`post_migrate` → 扫描各 app 的 `notifications` 模块找 `SystemMessage` 子类自动创建 `SystemMsgSubscription`；`User.post_save`(created) → 按用户接收后端创建 `UserMsgSubscription`
- **关键类**：`MessageContent`、`SiteMessage`、`SystemMsgSubscription`、`UserMsgSubscription`、[site_msg.py](file:///workspace/apps/notifications/site_msg.py)、[ws.py](file:///workspace/apps/notifications/ws.py)

---

### 6.15 settings — 系统设置

> 路径：[apps/settings](file:///workspace/apps/settings)

**职责**：系统设置持久化（`Setting`）、聊天提示、泄漏密码 sqlite 库、LDAP TLS；设置变更 PubSub 与刷新；终端 host-key 生成。

- **AppConfig**：`SettingsConfig`
- **关键模型**（[models.py](file:///workspace/apps/settings/models.py)）：`Setting`、`ChatPrompt`、`LeakPasswords`
- **URL 命名空间**：`common`（注意：settings app 的 urls 声明 `app_name='common'`，与 common app 共享命名空间）
- **signal_handlers.py**：`Setting.post_save` → PubSub 发布变更；若 `PERM_SINGLE_ASSET_TO_UNGROUP_NODE` 变更 → 失效所有用户权限树；`django_ready` → `refresh_all_settings()`、自动生成 `TERMINAL_HOST_KEY`、订阅设置变更 PubSub、monkey-patch `LazySettings.__getattr__` 以评估 `jumpserver.conf` 中的可调用默认值；`post_migrate` → 更新 site URL + 初始化/复制泄漏密码 sqlite 库；`django_ready` → 注册 sqlite 连接
- **关键类**：`Setting`、`ChatPrompt`、`LeakPasswords`

---

### 6.16 labels / reports — 标签与报表

**labels**（[apps/labels](file:///workspace/apps/labels)）：通用标签 — `Label` + `LabeledResource` 关联，可附加到组织级资源。`LabelsConfig`，URL 命名空间 `labels`，无 `ready()`。

**reports**（[apps/reports](file:///workspace/apps/reports)）：报表 — `Report`（组织级）模型 + 报表 mixin/views，用于生成与导出报表。`ReportsConfig`，URL 命名空间 `reports`，无 `ready()`。

---

## 7. 跨模块关键机制

### 7.1 连接器注册与心跳

**注册流程**（BOOTSTRAP_TOKEN 一次性注册，之后用服务账号 AccessKey）：

1. **配置**：`BOOTSTRAP_TOKEN`（[conf.py](file:///workspace/apps/jumpserver/conf.py) defaults、[base.py:44](file:///workspace/apps/jumpserver/settings/base.py)）
2. **注册端点**：`POST /api/v1/terminal/terminal-registrations/`（或 `registration/`）→ `TerminalRegistrationApi`（[api/component/terminal.py](file:///workspace/apps/terminal/api/component/terminal.py)），`permission_classes = [WithBootstrapToken]`
3. **`WithBootstrapToken`**（[common/permissions.py](file:///workspace/apps/common/permissions.py)）：`hmac.compare_digest` 校验 token；`check_can_register()` 受 `SECURITY_SERVICE_ACCOUNT_REGISTRATION`（`auto`/true/false）控制
4. **`TerminalRegistrationSerializer`**（[serializers/terminal.py](file:///workspace/apps/terminal/serializers/terminal.py)）：创建 `Terminal` + 服务账号 `User`（赋予 system-component 角色）+ 默认存储
5. **内置组件自注册**：[startup.py](file:///workspace/apps/terminal/startup.py) 的 `CoreTerminal`/`CeleryTerminal`（Singleton）在进程内通过 `TerminalRegistrationSerializer` 直接注册（绕过 HTTP/BOOTSTRAP_TOKEN），并起 30s 心跳守护线程

**心跳与任务分发**：

- 连接器每 ~30s `POST /status/`（`StatusViewSet`，[api/component/status.py](file:///workspace/apps/terminal/api/component/status.py)），上报 `StatSerializer`（cpu/mem/disk/sessions）
- `Status` 模型不落库，`save()` 调 `terminal.set_alive(ttl=3min)` + `save_to_cache()`（`TERMINAL_STATUS_{id}`，3 分钟 TTL）
- 心跳响应返回该连接器未完成的 `Task`（10 分钟内）—— **拉取通道**
- `Task.post_save` 经 Redis PubSub `fm.component_event_chan` 推送 —— **推送通道**

**控制任务**（kill/lock/unlock session）：

- `Task` 模型（[models/component/task.py](file:///workspace/apps/terminal/models/component/task.py)）：`name`（`kill_session`/`lock_session`/`unlock_session`）、`args`(session id)、`terminal`
- `KillSessionAPI`、`TaskViewSet.create_toggle_task` → `create_sessions_tasks()`
- 任务完成时信号处理器更新 Session 锁缓存

**配置下发**：连接器以服务账号认证后 `GET /api/v1/terminal/terminals/config/`（`TerminalConfig`）获取 `terminal.config`（所有 `TERMINAL*` 设置 + 存储配置 + 安全限制 + Chat AI + License）。`EncryptedTerminalConfig` 解密敏感值。

### 7.2 会话生命周期

- **创建**：连接器 `POST sessions`（`SessionViewSet`，[api/session/session.py](file:///workspace/apps/terminal/api/session/session.py)），`perform_create` 隐式设 `terminal = request.user.terminal`
- **`Session` 模型**（[models/session/session.py](file:///workspace/apps/terminal/models/session/session.py)）：UUID PK、denormalized user/asset/account、`protocol`、`login_from`、`type`（normal/tunnel/command/sftp）、`is_success`/`is_finished`、`has_replay`/`has_command`、`date_start`/`date_end`、`cmd_amount`、锁缓存 `TOGGLE_LOCKED_SESSION_{id}`、活跃缓存 `SESSION_ACTIVE_{id}`
- **生命周期事件**：连接器 `POST sessions/<id>/lifecycle_log/`（`IsServiceAccount`），core 记录为 `audits.ActivityLog`（[session_lifecycle.py](file:///workspace/apps/terminal/session_lifecycle.py)）
- **命令上报**：连接器批量 `POST commands`（`CommandViewSet`，[api/session/command.py](file:///workspace/apps/terminal/api/session/command.py)）→ `command_store.bulk_save()`；高危命令走 `commands/insecure-command/` 告警
- **命令存储后端**：`CommandStorage`（type `null`/`server`/`es`），DB 或 ES（`common.plugins.es`）；`Command` 模型 `bulk_create` 重发 `post_save` 以触发信号
- **回放上报**：连接器 `POST sessions/<id>/replay/`（`SessionReplayViewSet`）→ `session.save_replay_to_storage_with_version()`；v≤4 按 `SUFFIX_MAP` 后缀（`.replay.gz`/`.cast.gz`/`.replay.mp4`/`.replay.json`），v≥5 按会话目录 part 文件；可选外存（`SERVER_REPLAY_STORAGE`）
- **孤儿会话清理**：`clean_orphan_session`（每 10 分钟）—— 无 `SESSION_ACTIVE_` 缓存则标记 `is_finished`

### 7.3 权限树缓存机制

由于授权资产树计算昂贵，按用户物化（`UserAssetGrantedTreeNodeRelation`）：

1. **构建**（`UserPermTreeBuildUtil.rebuild_user_perm_tree`）：直接授权节点 → `granted`；含授权资产的节点 → `asset`；祖先补全 → `child`；计算资产数量并 bulk_create
2. **刷新**（`UserPermTreeRefreshUtil`）：按组织比对 Redis SET `perms.user.node_tree.built_orgs.user_id:{user_id}` 决定是否重建；`UserGrantedTreeRebuildLock` 防并发；小资产量同步、否则异步 Celery
3. **失效**（`UserPermTreeExpireUtil`）：`AssetPermission` save/delete/M2M 变化、`User.groups` 变化、`UserGroup` 删除、`Asset.nodes` 变化 → 失效 + 刷新 type-nodes-tree 缓存
4. **读取**（`UserPermAssetUtil`/`UserPermNodeUtil`）：服务"我的资产"树视图（顶级节点、节点子项、节点资产、收藏/未分组、按类型分组）

### 7.4 工单审批与授权链路

1. 申请人 `open` 一个 `ApplyAssetTicket` → `StatusMixin.open()` 按 `TicketFlow` 的 `ApprovalRule` 创建 `TicketStep`/`TicketAssignee`，state=pending
2. 审批人 `approve` → 每个 step `_on_step_approved` 邮件通知 + 激活下一级（若有）
3. **最终审批通过** → `ApplyAssetTicket.Handler._create_asset_permission()` 创建 `AssetPermission`（`from_ticket=True`，PK = 工单 PK，授权申请人请求的 nodes/assets/accounts/actions/有效期）
4. `AssetPermission` save + M2M `.set()` 触发 `perms/signal_handlers/asset_permission.py` → `UserPermTreeExpireUtil.expire_perm_tree_for_perms([ticket.id])` → 申请人授权树失效，下次访问懒重建

> 工单不绕过权限模型，而是创建真实的 `AssetPermission`（标记 `from_ticket`），复用正常权限/缓存机制。

### 7.5 启动钩子 `django_ready`

`CommonConfig.ready()` 在非维护命令时发送 `django_ready` 信号，众多 app 连接该信号完成启动初始化：检查迁移冲突、清响应缓存、刷新设置、生成 host key、订阅各类 PubSub（`fm.orgs_mapping`/`refresh_vault`/`fm.component_event_chan`/`fm.job_execution_stop`/`notifications.SiteMessageCome`/`settings`）、同步内置角色、初始化系统消息订阅等。

---

## 8. 依赖关系

### 8.1 内部依赖（app 间）

```
                      ┌──────────┐
                      │  orgs    │ ← 多租户基础（OrgModelMixin 被几乎所有业务模型混入）
                      └────┬─────┘
                           │
   ┌───────────┬───────────┼───────────┬────────────┐
   ▼           ▼           ▼           ▼            ▼
 users      assets      accounts     rbac       authentication
   │           │           │           │            │
   │           │           │           │            │
   └─────┬─────┘     ┌─────┘           │            │
         ▼           ▼                 │            │
      perms ←──── tickets ─────────────┘            │
         │                                          │
         ▼                                          ▼
      terminal  ←─────── 连接器（koko/lion/...）  连接令牌
         │
         ▼
      audits（记录所有活动）/ ops（执行自动化）

   common 为所有 app 提供基类；settings/labels/notifications/acls/reports 为横切服务
```

- **orgs** 是多租户基石，几乎所有业务模型依赖 `OrgModelMixin`
- **perms** 依赖 `users`/`assets`/`accounts`；**tickets** 通过创建 `AssetPermission` 依赖 `perms`
- **terminal** 与 **audits** 通过 `ActivityLog` 桥接会话生命周期
- **authentication** 的 `ConnectionToken` 依赖 `perms` 校验与 `terminal` 会话
- **ops** 提供自动化能力，被 `accounts`/`assets` 等调用执行 Ansible
- 为打破循环依赖，多处使用 `hands.py`（如 [apps/perms/hands.py](file:///workspace/apps/perms/hands.py)、[apps/assets/hands.py](file:///workspace/apps/assets/hands.py)、[apps/terminal/hands.py](file:///workspace/apps/terminal/hands.py)）做导入再导出

### 8.2 外部依赖（关键）

| 用途 | 依赖 |
|------|------|
| Web 框架 | django==5.2.13、djangorestframework==3.16.1、drf-spectacular==0.29.0、channels、drf-nested-routers、drf-writable-nested、rest-condition、djangorestframework-bulk |
| 任务队列 | celery==5.6.0、kombu==5.6.0、flower==2.0.1、django-celery-beat==2.8.1、eventlet、greenlet |
| 数据库驱动 | mysqlclient==2.2.4、psycopg2-binary==2.9.12、pymssql==2.3.10、oracledb（xpack） |
| Redis | redis（fork）、django-redis==5.4.0、python-redis-lock==4.0.0、channels-redis==4.1.0 |
| Ansible | ansible-core（fork v2.16.18.3）、ansible==9.13.0、ansible-runner（fork v2.4.0.2）、ansible-builder |
| 认证 | python-ldap==3.4.5、ldap3==2.9.1、django-auth-ldap==4.4.0、python3-saml==1.16.0、django-cas-ng（fork）、django-radius（fork）、django-oauth-toolkit==2.4.0、fido2（Passkey） |
| 密码学 | cryptography==46.0.7、pycryptodome==3.23.0、gmssl==3.2.2（国密）、bcrypt==4.0.1、paramiko==3.5.1、pyopenssl==26.0.0 |
| 云 SDK（xpack） | boto3、oss2、esdk-obs-python、azure-*、tencentcloud-sdk-python、huaweicloudsdk*、alibabacloud-*、qingcloud-sdk、bce-python-sdk、ucloud-sdk-python3、volcengine-python-sdk、google-cloud-compute、python-novaclient/keystoneclient |
| Vault | hvac==1.1.1、azure-keyvault-secrets、azure-identity |
| ES | elasticsearch7==7.17.13、elasticsearch8==8.13.2 |
| 其他 | openai（AI）、pydantic、httpx、requests、jinja2、openpyxl/xlsxwriter、polib（i18n）、geoip2/ipip-ipdb、premailer、mistune、exchangelib（邮件） |

> 依赖由 **uv** 管理（[pyproject.toml](file:///workspace/pyproject.toml) + [uv.lock](file:///workspace/uv.lock)）。Docker 构建用 `uv pip install -r pyproject.toml`。

---

## 9. 项目运行方式

### 9.1 快速开始（生产，单机）

```sh
curl -sSL https://github.com/jumpserver/jumpserver/releases/latest/download/quick_start.sh | bash
```

浏览器访问 `http://your-jumpserver-ip/`，默认账号 `admin` / `ChangeMe`。

### 9.2 Docker 部署

镜像构建见 [Dockerfile](file:///workspace/Dockerfile)（多阶段：`stage-build` 安装依赖 + 编译 i18n；运行阶段 `python:3.14-slim-trixie`）。

```sh
docker build -t jumpserver/core .
docker run -d -p 8080:8080 -v $PWD/data:/opt/jumpserver/data jumpserver/core
```

- 入口 [entrypoint.sh](file:///workspace/entrypoint.sh)：`cron` 启动后执行 `python jms "${action}" "${service}"`
- 默认 `CMD ["start", "all"]`，暴露 8080，`STOPSIGNAL SIGQUIT`，数据卷 `/opt/jumpserver/data`
- 需要 MySQL/PostgreSQL + Redis（通过 `config.yml` 配置）

### 9.3 源码开发运行

```sh
# 1. 安装依赖（uv）
uv pip install -r pyproject.toml        # 或 uv sync

# 2. 准备配置
cp config_example.yml config.yml        # 编辑 DB/REDIS/SECRET_KEY/BOOTSTRAP_TOKEN

# 3. 初始化
cd apps
python manage.py migrate
python manage.py compilemessages         # 编译 i18n
python manage.py collectstatic --no-input

# 4. 启动（推荐用 jms 脚本）
cd ..
python jms start all                     # 或 -d 守护进程，-w 指定 worker 数
```

开发模式（DEBUG）可用 `python manage.py runserver 0.0.0.0:8080`（[manage.py](file:///workspace/apps/manage.py)）。

### 9.4 `jms` 服务控制脚本

[jms](file:///workspace/jms) 是主控制脚本，支持：

| 命令 | 说明 |
|------|------|
| `start all` | 启动全部（web + task），先执行 `prepare()` |
| `start web` | 仅启动 Web 服务 |
| `start task` | 仅启动 Celery 任务 |
| `stop` / `restart` / `status` | 停止/重启/查看状态 |
| `upgrade_db` | 收集静态文件 + 执行 migrate |
| `collect_static` | 仅收集静态文件 |

参数：`-d/--daemon`（守护）、`-w/--worker <n>`（worker 数，或环境变量 `CORE_WORKER`）、`-f/--force`。

**启动前 `prepare()` 流程**：

1. `check_database_connection()` — 最多重试 60 次等 DB 就绪
2. `upgrade_db()` — `collect_static()` + `perform_db_migrate()`
3. `expire_caches()` — 清理过期缓存
4. `download_ip_db()` — 下载 GeoLite2/ipip IP 库到 `apps/common/utils/ip/`
5. `install_builtin_applets()` — 安装内置 Applet
6. `init_oauth2_provider()` — 初始化 OAuth2 Provider

### 9.5 配置文件

复制 [config_example.yml](file:///workspace/config_example.yml) 为 `config.yml`，配置：

- `SECRET_KEY` / `BOOTSTRAP_TOKEN`（必填）
- 数据库（默认 PostgreSQL：`DB_ENGINE: postgresql`）
- Redis
- HTTP/WS 端口（8080/8070）
- 认证方式（LDAP/OIDC/CAS/SAML2/OAuth2/RADIUS/Passkey/企业微信/钉钉/飞书/Lark/Slack）
- 安全策略、Terminal、Vault 等

也支持 `config.py`（Python 形式）或环境变量覆盖。

### 9.6 常用管理命令

```sh
python manage.py createsuperuser        # 创建超管
python manage.py changepassword admin   # 改密
python manage.py makemigrations         # 生成迁移
python manage.py migrate                # 应用迁移
python manage.py compilemessages        # 编译 i18n
python manage.py collectstatic          # 收集静态
python manage.py init_oauth2_provider   # 初始化 OAuth2
python manage.py install_builtin_applets
python manage.py expire_caches
python manage.py check --database default
```

[utils/](file:///workspace/utils) 下有大量运维脚本：`create_test_data.py`、`backup_db.sh`、`clean_migrations.sh`、`sync_builtin_role.py`、`change_user_password.py`、`disable_user_mfa.sh`、各迁移脚本等。

---

## 10. CI/CD 与工具链

### 10.1 GitHub Actions 工作流

见 [.github/workflows/](file:///workspace/.github/workflows)：

- `build-base-image.yml` / `build-python-image.yml` / `build-ansible-executor.yml` — 构建基础镜像
- `jms-generic-action-handler.yml` — 通用 action 处理
- `check-compilemessages.yml` — 检查 i18n 编译
- `i18n-auto-translate.yml` — 自动翻译 i18n
- `translate-readme.yml` — 翻译 README
- `docs-release.yml` / `discord-release.yml` / `release-drafter.yml` — 文档/通知/发布
- `sync-gitee.yml` — 同步到 Gitee
- `issue-*.yml` — Issue 管理（关闭、评论、提醒、垃圾过滤）
- `cleanup-branches.yml` — 分支清理

### 10.2 代码规范

- [.isort.cfg](file:///workspace/.isort.cfg) — import 排序
- [.pylintrc](file:///workspace/.pylintrc) — Python lint
- [.prettierrc](file:///workspace/.prettierrc) — 前端格式化
- [CONTRIBUTING.md](file:///workspace/CONTRIBUTING.md) — 贡献指南

### 10.3 依赖管理

- **uv**（[pyproject.toml](file:///workspace/pyproject.toml) + [uv.lock](file:///workspace/uv.lock)）
- 依赖分组：`default`（核心）、`xpack`（企业版/云 SDK，默认启用）、`dev`（开发工具，默认启用）
- 部分 fork 依赖通过 `[tool.uv.sources]` 指定 URL：`ansible-core`、`ansible-runner`、`django-cas-ng`、`django-radius`、`redis`

### 10.4 多语言

- 支持 8 种语言（英/中简/中繁/日/葡/西/俄/韩），见 [readmes/](file:///workspace/readmes)
- i18n 资源在 [apps/i18n/](file:///workspace/apps/i18n)（`translate.py`/`sort_json.py`/`ci_translate.py`），编译输出 `LOCALE_PATHS = [BASE_DIR/i18n/core]`

---

## 附录：关键文件速查

| 主题 | 文件 |
|------|------|
| 项目主配置 | [apps/jumpserver/settings/__init__.py](file:///workspace/apps/jumpserver/settings/__init__.py) |
| 基础 settings | [apps/jumpserver/settings/base.py](file:///workspace/apps/jumpserver/settings/base.py) |
| 配置加载 | [apps/jumpserver/conf.py](file:///workspace/apps/jumpserver/conf.py) |
| 根 URL | [apps/jumpserver/urls.py](file:///workspace/apps/jumpserver/urls.py) |
| 服务控制 | [jms](file:///workspace/jms) |
| Docker 入口 | [entrypoint.sh](file:///workspace/entrypoint.sh) |
| 依赖声明 | [pyproject.toml](file:///workspace/pyproject.toml) |
| 配置示例 | [config_example.yml](file:///workspace/config_example.yml) |
| 基础模型 | [apps/common/db/models.py](file:///workspace/apps/common/db/models.py) |
| 基础 API | [apps/common/api/generic.py](file:///workspace/apps/common/api/generic.py) |
| 权限基类 | [apps/common/permissions.py](file:///workspace/apps/common/permissions.py) |
| 多租户 | [apps/orgs/mixins/models.py](file:///workspace/apps/orgs/mixins/models.py) |
| 连接器注册 | [apps/terminal/api/component/terminal.py](file:///workspace/apps/terminal/api/component/terminal.py) |
| 会话模型 | [apps/terminal/models/session/session.py](file:///workspace/apps/terminal/models/session/session.py) |
| 授权模型 | [apps/perms/models/asset_permission.py](file:///workspace/apps/perms/models/asset_permission.py) |
| 工单状态机 | [apps/tickets/models/ticket/general.py](file:///workspace/apps/tickets/models/ticket/general.py) |
| 工单授权 | [apps/tickets/handlers/apply_asset.py](file:///workspace/apps/tickets/handlers/apply_asset.py) |
| 启动钩子 | [apps/common/apps.py](file:///workspace/apps/common/apps.py) |

---

*本文档基于仓库当前状态生成，用于代码理解与导航。如需运行部署，请以 [README.md](file:///workspace/README.md) 与官方文档为准。*
