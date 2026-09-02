# Spring Boot 单体、微服务、Spring AI 与安全合规开发规范 (java-springboot-standards-skill)

> **基于大型生产级微服务与 Spring AI 智能体工程实践提炼的跨平台通用 Agent Skill**  
> 遵循开放的 [Agent Skills Open Standard (SKILL.md)](https://github.com/agent-skills/standard) 规范。

---

## 📖 规范体系与文件索引

| 规范模块 | 路径 | 核心内容 |
| :--- | :--- | :--- |
| **主入口定义** | [`SKILL.md`](SKILL.md) | 遵循开放标准的 Frontmatter、双模触发机制、反过度设计铁律与质量自检清单。 |
| **架构选型与形态** | [`references/architecture_modes.md`](references/architecture_modes.md) | 单体架构 vs 微服务架构 vs Spring AI 智能体体系全维度对照表及平滑演进路线。 |
| **分层与包结构** | [`references/layering_and_packages.md`](references/layering_and_packages.md) | 标准包结构定义、六端 Controller 物理隔离、Domain/DTO/VO 生命周期流转与 `@NoWrapper`。 |
| **编码与注释规范** | [`references/coding_and_comment_standards.md`](references/coding_and_comment_standards.md) | `UserContext` 规范、统一异常与 600+ 错误码体系、Javadoc 与序号化步骤业务注释标准。 |
| **安全防护与合规** | [`references/security_and_compliance.md`](references/security_and_compliance.md) | JWT 认证与网关协同、`@PreAuthorize` 细粒度权限、SQL 注入/XSS/CSRF 防御、PII 数据脱敏、安全头与上线审查清单。 |
| **中间件与设计模式** | [`references/patterns_and_middleware.md`](references/patterns_and_middleware.md) | Spring AI Multi-Agent 工具链、Redisson 分布式锁、Seata 事务、状态机、Redis+Lua、Canal+ES、XXL-Job。 |

---

## 🚀 跨平台安装与接入指南

本 Skill 不依赖特定平台特有协议，可在当前工作区使用，也可随时分发安装至各大主流 Agent 工具：

### 1. Claude Code
将本目录复制或软链接至全局或项目目录：
```bash
cp -r java-springboot-standards-skill ~/.claude/skills/
```

### 2. OpenAI Codex Desktop
放置于全局 Skills 根目录：
```bash
cp -r java-springboot-standards-skill ~/.codex/skills/
```

### 3. Cursor / Windsurf / Cline / OpenCode
直接在项目的 `.cursor/rules/` 或提示词库中引入：
```markdown
# 引用本规范
@java-springboot-standards-skill/SKILL.md
```

---

## 🛡️ 开源协议
MIT License
