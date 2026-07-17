# Taven's Claude Code Skills

> Taven 的 Claude Code 技能库。
> 工程、思维、创作、决策——把踩过的坑和想明白的事变成可复用的 AI 工作流。

## Skills

### Thinking 思维工具
| Skill | Description |
|-------|-------------|
| [polanyi](thinking/polanyi/) | 隐性知识思维教练 — 帮你把"说不出来但知道"的东西挖掘、外化、整合 |

### Engineering 全栈开发
| Skill | Description |
|-------|-------------|
| [senior-engineer](engineering/senior-engineer/) | 资深工程师思维 — 代码质量标准、架构判断力、审查检查点 |
| [code-review](engineering/code-review/) | 代码审查 — 安全→正确性→性能→可维护性，任何语言通用 |
| [project-takeover](engineering/project-takeover/) | 接手/二开项目 — 5 分钟摸清陌生代码库的系统方法 |
| [project-bootstrap](engineering/project-bootstrap/) | 新建项目 — 技术选型、项目结构、从零到部署的完整流程 |
| [frontend-dev](engineering/frontend-dev/) | 前端开发 — 数据加载三层架构、状态管理、组件设计、性能优化 |
| [backend-dev](engineering/backend-dev/) | 后端开发 — API 设计、认证、数据库交互、后台任务、错误处理 |
| [database-ops](engineering/database-ops/) | 数据库 — 表设计、索引策略、迁移管理、连接池调优、归档 |
| [system-design](engineering/system-design/) | 系统架构 — 缓存/解耦/降级/背压的通用设计模式 |
| [deploy-safety](engineering/deploy-safety/) | 安全部署 — 部署前/中/后检查、健康检查、回滚策略 |
| [debug-method](engineering/debug-method/) | 系统排错 — 四步定位法、常见 bug 模式速查 |

### Decision 决策系统
| Skill | Description |
|-------|-------------|
| [advisor](decision/advisor/) | 多顾问决策系统总调度器 — 描述问题，自动按会诊顺序调度顾问团队 |
| [invest](decision/invest/) | 投资将军 — 调度 6 位投资专家进行多维度投资分析 |
| [munger](decision/munger/) | 查理·芒格 — 多维决策、风险识别、跨学科分析 |
| [buffett](decision/buffett/) | 沃伦·巴菲特 — 护城河、聚焦、拒绝决策 |
| [drucker](decision/drucker/) | 彼得·德鲁克 — 商务增长、价值定义、用户需求 |
| [jobs](decision/jobs/) | 史蒂夫·乔布斯 — 产品体验、内容设计、用户交互 |
| [musk](decision/musk/) | 埃隆·马斯克 — 执行推动、第一性原理、打破惯性 |
| [hara](decision/hara/) | 原研哉 — 系统极简、结构审查、流程优化 |
| [wittgenstein](decision/wittgenstein/) | 维特根斯坦 — 概念澄清、语言分析、问题本质 |
| [freud](decision/freud/) | 弗洛伊德 — 深层动机、情绪化决策识别 |
| [dalio](decision/dalio/) | 瑞·达利欧 — 宏观经济、经济周期、全天候策略 |
| [graham](decision/graham/) | 本杰明·格雷厄姆 — 价值投资、安全边际 |
| [marks](decision/marks/) | 霍华德·马克斯 — 逆向投资、市场周期、风险控制 |
| [thiel](decision/thiel/) | 彼得·蒂尔 — 创新投资、垄断思维、从 0 到 1 |
| [fisher](decision/fisher/) | 菲利普·费雪 — 成长股投资、长期持有 |
| [lynch](decision/lynch/) | 彼得·林奇 — 个股分析、生活中发现机会 |

## Install

```bash
./install.sh                           # Install all
./install.sh engineering/frontend-dev   # Install one
```

## Usage

```bash
# 思维挖掘
/polanyi 我对 AI 时代的内容创作有一种矛盾感
/polanyi --mode create 关于隐性知识的文章

# 接手一个陌生项目
/project-takeover /path/to/project

# 从零开始新项目
/project-bootstrap 我要做一个社交 app

# 写前端功能
/frontend-dev 做一个带筛选和分页的用户列表

# 写后端 API
/backend-dev 设计一个用户认证系统

# 数据库设计
/database-ops 设计用户表和关注关系表

# 检查代码质量
/senior-engineer 审查这个 PR

# 排查线上问题
/debug-method 页面打开后所有请求都显示"已取消"

# 多顾问决策
/advisor 要不要把现有产品从 Web 迁移到 App

# 单独请教某位顾问
/munger 分析一下知识付费这个商业模式
/jobs 审视一下这个产品的用户体验
```

## About

Built by [Taven](https://github.com/wenttt)。

## License

MIT
