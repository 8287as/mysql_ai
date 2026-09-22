# QueryCraft · MySQL AI 学习工作台

> 用自然语言描述需求，DeepSeek 自动建库建表、出题、判分和讲解的交互式 SQL 学习工具。

QueryCraft 是一个连接**真实 MySQL**、由 **DeepSeek** 驱动的在线 SQL 学习工作台。你只需要用一句话描述想练什么（比如"电商订单，练多表 JOIN 和按月统计"），它就会自动设计 2-4 张关联表、填充练习数据、生成题目，并在你写完 SQL 后给出运行结果、AI 判分和逐项讲解。

## 功能亮点

- **AI 生成练习**：自然语言描述学习需求，自动设计关联表、填充数据、生成 1-50 套题目（数量、难度可选）
- **AI 判分讲解**：根据题意、参考 SQL 和实际执行结果自动判分，逐项解释错误
- **AI 问答助手**：左侧抽屉带入需求、题目、SQL 和执行结果，支持连续追问
- **SQL 智能补全**：轻量 tokenizer + 上下文状态机，根据 SELECT/FROM/JOIN/WHERE/ON 位置提示字段、表/视图和别名；字符串/注释内自动关闭提示
- **SQLGlot 增强解析**：可选 Python 侧车服务，CTE / JOIN / 别名场景走 AST 解析，服务不可用时自动回退前端补全
- **真实 MySQL 环境**：在真实 MySQL 中创建独立 `sql_lab_*` 练习库，逐题预执行参考 SQL，答案不可运行则放弃该练习库
- **完整数据库浏览器**：数据库树、表结构、字段类型/主键/注释、分页查看数据、右键菜单（创建/删除数据库）
- **多 SQL 控制台**：多个独立控制台标签，各自保存 SQL；支持选中执行、格式化、快捷键
- **安全设计**：拦截 `DROP DATABASE`、权限管理、文件导出等高危操作；凭据只保存在浏览器会话，不落盘
- **本地持久化**：练习需求、题目、进度和 SQL 草稿存在浏览器本地，重开自动恢复

## 技术栈

| 层 | 技术 |
| --- | --- |
| 前端 | 原生 HTML/CSS/JS（无框架），Vant 风格手写 UI |
| 后端 | Node.js 20+ · Express 5 · mysql2 · sql-formatter |
| 解析侧车 | Python 3.11+ · FastAPI · SQLGlot（可选） |
| AI | DeepSeek API（OpenAI 兼容协议） |
| 数据库 | MySQL 8 |

## 快速开始

### 环境要求

- Node.js 20+
- MySQL 8（用户需拥有建库、建表、插入和查询权限）
- Python 3.11+（可选，启用 SQLGlot 增强解析时）

### 安装与启动

```bash
npm install
npm start
```

可选：启动 SQLGlot 解析服务（仅监听 `127.0.0.1:8765`）：

```bash
# Windows
.\start-parser.ps1

# 或同时启动 Node 和 SQLGlot
.\start-all.ps1
```

打开 <http://127.0.0.1:3030>，进入"设置"：

1. 填写 MySQL 主机、端口、用户名和密码，测试连接
2. 填写 DeepSeek API Key，测试 AI 连接
3. 保存设置，在顶部输入学习需求，点击"生成练习"

### 使用示例

在顶部输入：

> 创建一个电商订单数据库，我想练习多表 JOIN、按月统计销售额，并找出复购率最高的客户

QueryCraft 会创建独立的 `sql_lab_*` 数据库，设计订单、客户等关联表并填充数据，生成一套带难度分级的练习题。你可以在编辑器里写 SQL、运行，然后查看 AI 老师的判分和讲解。

## 架构

```
┌─────────────────────────────────────────────┐
│  浏览器（原生前端，无框架）                     │
│  SQL 编辑器 · 结果区 · 数据库树 · AI 问答抽屉   │
└──────────────┬──────────────────────────────┘
               │ HTTP
┌──────────────▼──────────────────────────────┐
│  Node.js 主服务 (Express, :3030)             │
│  SQL 校验/安全拦截 · 语句拆分 · 批量/事务执行   │
│  DeepSeek 代理（生成/判分/讲解/问答）          │
└───────┬───────────────────────────┬─────────┘
        │ mysql2                    │ HTTP (:8765, 可选)
┌───────▼─────────┐      ┌──────────▼──────────┐
│  MySQL 8         │      │ SQLGlot 解析侧车     │
│  sql_lab_* 练习库 │      │ tokenize / parse    │
└─────────────────┘      └─────────────────────┘
```

## API 一览

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| GET | `/api/health` | 服务健康检查 |
| POST | `/api/sql/format` | SQL 格式化 |
| GET | `/api/sqlglot/health` | SQLGlot 侧车健康检查 |
| POST | `/api/sqlglot/tokenize` | SQL 分词 |
| POST | `/api/sqlglot/parse` | SQL AST 解析 |
| POST | `/api/mysql/test` | 测试 MySQL 连接 |
| POST | `/api/mysql/schema` | 获取数据库/表结构 |
| POST | `/api/mysql/table` | 分页查看表数据 |
| POST | `/api/mysql/query` | 执行 SQL（支持批量/事务/存储过程） |
| POST | `/api/mysql/drop` | 删除库/表（需名称二次确认） |
| POST | `/api/mysql/create-database` | 创建数据库 |
| POST | `/api/lab/materialize` | 物化 AI 生成的练习库 |
| POST | `/api/ai/test` | 测试 DeepSeek 连接 |
| POST | `/api/ai/generate` | 生成练习 |
| POST | `/api/ai/answer` | AI 判分 |
| POST | `/api/ai/explain` | AI 讲解 |
| POST | `/api/ai/chat` | AI 问答（连续追问） |

## 安全设计

- **高危 SQL 拦截**：阻止 `DROP DATABASE`、权限管理、全局设置、`LOAD_FILE()` 文件导出等操作
- **系统库保护**：`information_schema`、`mysql`、`performance_schema`、`sys` 禁止删除
- **删除二次确认**：删除数据库/表必须输入完整名称确认
- **凭据不落盘**：MySQL 密码和 DeepSeek Key 只保存在浏览器会话中，不写入项目文件或服务端日志
- **练习库隔离**：AI 生成的数据写入独立的 `sql_lab_*` 数据库，不触碰已有业务表

## 目录结构

```
mysql-ai-studio/
├── server.js              # Node 主服务（约 1900 行）
├── package.json
├── public/                # 前端
│   ├── index.html         # 页面结构
│   ├── styles.css         # 样式
│   └── app.js             # 前端逻辑（约 2700 行）
├── parser_service/        # SQLGlot 解析侧车（可选）
│   ├── app.py
│   └── requirements.txt
├── start-all.ps1          # 一键启动 Node + SQLGlot
├── start-parser.ps1       # 仅启动 SQLGlot
└── .gitignore
```

## 屏幕截图

> TODO：在下方替换为你的实际运行截图。
> 建议：首页（需求输入）、SQL 编辑器 + 补全提示、AI 讲解面板、数据库浏览器。

![image-20260922222708203](C:\Users\Lan\AppData\Roaming\Typora\typora-user-images\image-20260922222708203.png)

## 开源协议

本项目采用 [MIT License](LICENSE)。

## 致谢

- [DeepSeek API](https://api-docs.deepseek.com/zh-cn/) — AI 生成、判分、讲解
- [SQLGlot](https://github.com/tobymao/sqlglot) — SQL 解析增强
- [Express](https://expressjs.com/) · [mysql2](https://github.com/sidorares/node-mysql2) · [sql-formatter](https://github.com/sql-formatter-org/sql-formatter)

---

> 本项目为个人学习工具，连接的是你自己的 MySQL 实例和 DeepSeek 账号，使用前请确保你有相应权限并注意数据安全。
