# 公司内部管理系统：开发与联调说明

## 1. 仓库与分支

本项目采用一个后端主仓库 + 两个前端子仓库：

- 后端主仓库：`lei56602-glitch/ruoyi-vue-pro`
- 电脑管理端：`lei56602-glitch/yudao-ui-admin-vue3`
- 手机员工端：`lei56602-glitch/yudao-ui-admin-uniapp`

三个仓库统一在 `development` 分支开发。后端主仓库通过 Git Submodule 固定前端提交，路径：

- `frontend/admin-web`
- `frontend/mobile-app`

首次完整克隆：

```bash
git clone --branch development --recurse-submodules https://github.com/lei56602-glitch/ruoyi-vue-pro.git
cd ruoyi-vue-pro
```

已克隆但未拉取子模块时：

```bash
git submodule update --init --recursive
```

需要同步前端子仓库 development 分支最新提交时：

```bash
git submodule update --remote --merge
git add frontend/admin-web frontend/mobile-app
git commit -m "chore: update frontend submodules"
git push origin development
```

## 2. 当前后端技术基线

- JDK 17
- Spring Boot 3.5.x
- Maven 多模块
- MySQL（默认开发配置）
- Redis
- Spring Security + Token
- Flowable 工作流

旧 JDK 8 代码保留在后端仓库的 `legacy-jdk8` 分支；当前业务开发只使用 `development`。

## 3. 已启用的公司核心业务模块

以下模块已同时加入根 Maven Reactor，并由 `yudao-server` 引入：

| 模块 | 用途 |
| --- | --- |
| system | 用户、部门、岗位、角色、菜单、权限 |
| infra | 文件、字典、参数、定时任务等基础设施 |
| bpm | 流程定义、流程表单、申请、待办、已办 |
| oa | 请假、报销、公告、任务、工作汇报、会议等办公协同 |
| hrm | 员工档案、考勤、假期、请假、薪资、招聘、绩效 |
| im | 单聊、群聊、消息及即时通讯 |

CRM、ERP、商城、支付、MES、WMS、IoT、AI 等模块目前没有启用，避免首期引入不必要的业务复杂度。后续按实际业务再逐个评估。

注意：启用模块只代表后端依赖已接入，不代表所有流程、菜单、审批模板和业务规则已经按本公司制度配置完成。

## 4. 本地开发环境

### 后端

准备：

- JDK 17
- Maven 3.9+
- MySQL 8.x
- Redis 6/7

后端本地服务默认端口为 `48080`，API 前缀为 `/admin-api`。

数据库初始化脚本位于 `sql/mysql/`。按项目初始化说明导入 MySQL 主脚本；如启用 Quartz 定时任务，再按需要处理 `quartz.sql`。Flowable 的表结构由当前配置自动检查/创建。

构建后端：

```bash
mvn -B -pl yudao-server -am package -DskipTests
```

启动可在 IDE 中运行 `yudao-server` 模块的 Spring Boot 启动类，或运行构建生成的可执行 JAR。

### 电脑管理端

要求 Node.js 20.19+、pnpm 10.28+。

```bash
cd frontend/admin-web
pnpm install
pnpm dev-server
```

` .env.dev ` 已配置为访问本机后端：

- 后端根地址：`http://localhost:48080`
- API 前缀：`/admin-api`

本机浏览器访问前端开发服务器输出的地址即可。

### 手机员工端

要求 Node.js 20+、pnpm 10+，并按目标平台安装 HBuilderX / Android 或 iOS 构建环境。

```bash
cd frontend/mobile-app
pnpm install
pnpm dev:h5
```

开发环境 API 地址配置文件：

`env/.env.development`

当前默认地址为 `http://127.0.0.1:48080`，适用于在同一台电脑上进行 H5 本地调试。

**真机调试必须将该地址改为运行后端电脑的局域网 IP**，例如 `http://192.168.1.10:48080`；手机和电脑需处于可互通网络，且电脑防火墙允许 48080 端口访问。手机上的 `127.0.0.1` 指手机自身，不能指向开发电脑。

## 5. 前后端接口约定

- 电脑端请求基础地址由 `VITE_BASE_URL + VITE_API_URL` 组成。
- 电脑端 API 前缀：`/admin-api`。
- 手机端通过 `VITE_SERVER_BASEURL` 设置后端根地址，业务请求继续使用项目原有 API 路径。
- 登录、Token 刷新、租户标识、权限菜单沿用芋道现有认证与权限实现；不要在前端自行绕过后端权限校验。

本阶段只完成开发环境地址对齐，尚未完成真实账号登录、跨端权限和全部业务 API 的端到端联调。

## 6. 安全与上线约束

当前 `application-local.yaml` 是上游本地开发示例，含本机数据库地址及示例账号密码。它不能直接作为生产配置。

正式部署前必须：

1. 使用独立数据库和 Redis 账号，设置强密码，不复用示例凭据。
2. 将密钥、数据库密码、短信/邮件/对象存储凭据放入服务器环境变量或安全配置中心，不提交到 Git。
3. 通过 HTTPS 域名访问，配置反向代理与跨域策略。
4. 限制数据库、Redis 仅允许应用服务器访问，不向公网开放。
5. 设置数据库备份、文件备份、日志留存和恢复演练。
6. 关闭或限制 Swagger、Druid、Actuator 等运维入口的公网访问。
7. 完成员工数据、考勤、薪资、审批等敏感信息的角色权限和数据权限测试。

## 7. 当前阶段边界

本次已完成：开发分支对齐、前端 Submodule 纳入、BPM/HRM/OA/IM 后端模块启用、前端开发 API 地址配置、Maven CI 触发分支配置。

本次尚未完成：完整 Maven 构建结果确认、数据库实际初始化、前后端真实登录联调、移动端真机网络验证、公司组织架构和审批制度配置。后续必须以 CI 构建和实际运行测试结果为准，不能仅凭源码存在就认定业务已可用。
