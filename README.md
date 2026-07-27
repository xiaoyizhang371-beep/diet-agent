# Diet Agent

一个基于 Spring Boot + AgentScope 的智能饮食推荐助手。项目通过多 Agent 编排完成意图识别、信息澄清、餐食检索、候选重排、推荐话术生成、健康风险守卫和链路追踪，并内置一个可直接访问的 Web 页面用于本地演示、餐食维护和评测调试。

## 功能特性

- 自然语言饮食推荐：用户可以用“今晚想吃清淡一点”“帮我安排三餐”这类表达发起请求。
- 多轮会话状态：保存 session、历史槽位、最近推荐结果，支持继续追问和“换一批”。
- 个人餐食库 / 公共餐食库：支持个人餐食增删改查，也可以从系统预置公共餐食中推荐。
- 槽位澄清：当餐次、口味、场景、健康目标等信息不足时，系统会先追问。
- 多餐规划：支持早餐、午餐、晚餐等多餐次组合推荐。
- 健康风险守卫：对医疗诊断、治疗承诺、极端节食等高风险内容走保守回复。
- Trace 与评测后台：记录每轮请求的 Agent 执行链路，支持人工标注和批量评估。
- 内置静态前端：无需单独启动前端工程，应用启动后即可访问页面体验。

## 技术栈

- Java 21
- Spring Boot 3.3.13
- MyBatis 3.0.4
- MySQL 8.x
- AgentScope Spring Boot Starter 1.0.11
- DashScope / Qwen 模型
- Lombok
- 原生 HTML / CSS / JavaScript 静态前端

## 项目结构

```text
diet-agent
├── pom.xml
├── diet_db.sql                         # 数据库初始化脚本，根目录副本
├── src/main/java/com/diet
│   ├── DietApplication.java             # Spring Boot 启动类
│   ├── agent                            # Agent 构建、Prompt 加载、Agent 工厂
│   ├── config                           # AgentScope / 模型配置
│   ├── controller                       # HTTP 接口
│   ├── enums                            # 意图、会话阶段、来源模式等枚举
│   ├── mapper                           # MyBatis Mapper
│   ├── model                            # 请求、响应、领域模型
│   ├── service                          # 核心业务服务与编排逻辑
│   └── util                             # JSON / LLM 输出解析工具
└── src/main/resources
    ├── application.yml                  # 应用配置
    ├── db/diet_db.sql                   # 数据库初始化脚本
    ├── diet/prompts                     # 各 Agent Prompt
    ├── mapper                           # MyBatis XML
    └── static                           # 内置 Web 页面
```

## 核心流程

```text
用户输入
  ↓
Session 加载 / 创建
  ↓
IntentAgent 意图识别
  ↓
槽位合并与意图修正
  ↓
ClarifyAgent 判断是否需要追问
  ↓
餐食检索 + 重排
  ↓
RecommendResponseAgent / PlanResponseAgent 生成回复
  ↓
RiskGuard 健康风险检查
  ↓
保存会话、消息、Trace 并返回结果
```

## 快速开始

### 1. 准备环境

请先确保本地安装：

- JDK 21+
- Maven 3.9+
- MySQL 8.x

### 2. 初始化数据库

创建并导入数据库：

```sql
CREATE DATABASE IF NOT EXISTS diet_db DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

然后导入 SQL 脚本：

```bash
mysql -u root -p diet_db < src/main/resources/db/diet_db.sql
```

如果你在 Windows PowerShell 中执行，也可以使用：

```powershell
mysql -u root -p diet_db < .\src\main\resources\db\diet_db.sql
```

### 3. 配置数据库和模型 Key

默认配置文件位于 `src/main/resources/application.yml`：

```yaml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/diet_db?useUnicode=true&characterEncoding=utf8&serverTimezone=Asia/Shanghai&useSSL=false&allowPublicKeyRetrieval=true&createDatabaseIfNotExist=true
    username: root
    password: root

agentscope:
  dashscope:
    api-key: ${DASHSCOPE_API_KEY:}

diet:
  llm:
    main-model: qwen-max
    light-model: qwen-turbo
```

建议通过环境变量提供 DashScope API Key：

```bash
export DASHSCOPE_API_KEY=your-dashscope-api-key
```

PowerShell：

```powershell
$env:DASHSCOPE_API_KEY="your-dashscope-api-key"
```

> 提交到 GitHub 前，请不要把真实 API Key、数据库密码等敏感信息写死在配置文件里。

### 4. 启动项目

```bash
mvn spring-boot:run
```

启动成功后访问：

```text
http://localhost:8080
```

## API 速览

所有接口默认使用请求头 `X-User-Id` 标识用户，不传时默认用户 ID 为 `1`。

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| POST | `/api/v1/diet/sessions` | 创建会话 |
| POST | `/api/v1/diet/chat` | 饮食推荐聊天接口 |
| GET | `/api/v1/diet/meals/personal` | 查询个人餐食 |
| POST | `/api/v1/diet/meals/personal` | 新增个人餐食 |
| PUT | `/api/v1/diet/meals/personal/{mealId}` | 修改个人餐食 |
| DELETE | `/api/v1/diet/meals/personal/{mealId}` | 删除个人餐食 |
| GET | `/api/v1/diet/meals/public` | 查询公共餐食 |
| GET | `/api/v1/diet/slot-options` | 查询可选槽位 |
| POST | `/api/v1/diet/feedback` | 保存推荐反馈 |
| GET | `/api/v1/diet/debug/traces` | 按时间范围查询 Trace |
| GET | `/api/v1/diet/debug/traces/{traceId}` | 查询单条 Trace |
| GET | `/api/v1/diet/debug/sessions/{sessionId}/traces` | 查询会话 Trace |
| PUT | `/api/v1/diet/debug/traces/{traceId}/label` | 标注 Trace |
| POST | `/api/v1/diet/evaluations` | 生成评测报告 |

### 聊天接口示例

```bash
curl -X POST http://localhost:8080/api/v1/diet/chat \
  -H "Content-Type: application/json" \
  -H "X-User-Id: 1" \
  -d '{
    "sessionId": null,
    "message": "今晚想吃清淡一点，最好快一点",
    "sourceMode": "PUBLIC",
    "context": {}
  }'
```

响应字段示例：

```json
{
  "sessionId": "session_xxx",
  "traceId": "trace_xxx",
  "responseType": "ANSWER",
  "speechText": "推荐理由文本",
  "displayBlocks": [],
  "nextAction": "WAIT_USER",
  "clarifyQuestion": null,
  "missingSlots": []
}
```

当信息不足时，`responseType` 会返回 `CLARIFY`，并通过 `clarifyQuestion` 和 `missingSlots` 告诉前端需要继续追问哪些信息。

## 数据库表

初始化脚本包含以下主要表：

- `diet_sessions`：会话状态，包含当前阶段、槽位、最近推荐结果。
- `diet_messages`：会话消息记录。
- `meal_item`：餐食数据，区分 `PUBLIC` 和 `PERSONAL`。
- `diet_slot_option`：餐次、心情、场景、健康目标、菜系、口味、便利性等可选项。
- `diet_request_trace`：每轮请求的执行链路、标注信息和耗时。
- `recommend_feedback`：用户对推荐结果的反馈。

## 前端页面

项目内置静态页面，启动后访问 `http://localhost:8080` 即可使用：

- 首页：查看个人餐食、公共餐食统计。
- 聊天推荐：选择个人库或公共库，与助手对话获取推荐。
- 个人餐食：维护自己的常吃餐食与标签。
- 公共餐食：查看系统预置餐食。
- Trace：查看每轮请求的 Agent 执行链路。
- 评估：按时间范围生成推荐链路评测报告。

## 开发说明

- Prompt 位于 `src/main/resources/diet/prompts`，分别对应意图识别、澄清、推荐回复、多餐规划回复和评测裁判。
- 核心编排入口是 `DietOrchestratorService`，一轮对话会在这里完成状态机流转。
- 模型 Bean 在 `DietAgentScopeConfig` 中配置，主模型默认用于最终推荐回复，轻量模型默认用于意图识别和澄清。
- Mapper XML 位于 `src/main/resources/mapper`，实体模型位于 `com.diet.model`。

## 提交 GitHub 前检查

- 移除真实 API Key、数据库密码等敏感信息。
- 确认 `target/`、IDE 配置、本地临时文件没有被提交。
- 确认数据库脚本中的示例数据可以公开。
- 如果需要公开部署，建议新增独立的 `application-example.yml`，把本地私密配置放入未提交的 profile 文件或环境变量。

## License

当前仓库尚未声明开源许可证。如需开源发布，请根据项目用途选择合适的 License，例如 MIT、Apache-2.0 或 GPL。

