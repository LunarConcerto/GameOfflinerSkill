# AI Skills

个人 Agent Skills 集合。每个子目录是一个独立 Skill，入口为 `SKILL.md`。

## 索引

| Skill | 说明 |
|---|---|
| [offline-login-revival](offline-login-revival/) | 关服 / 离线网游登录复原方法论：战略选型、静态侦察、注入与重定向、协议还原、时序契约、多版本适配 |

## 使用（opencode）

在本机全局配置 `~/.config/opencode/opencode.json` 中注册本目录：

```json
{ "$schema": "https://opencode.ai/config.json", "skills": { "paths": ["E:/AI/Skills"] } }
```

保存后退出并重启 opencode 生效。

## Skill 格式

每个 Skill 为一个目录，含：

```
<skill-name>/
├── SKILL.md        # 必需，YAML frontmatter（name + description）+ 正文
├── README.md       # 可选，面向人的介绍（GitHub 展示）
└── reference/       # 可选，按需加载的参考文档
```

`SKILL.md` frontmatter 的 `name` 必须与目录名一致（小写连字符）；`description` 需同时说明"做什么"和"何时触发"，并前置触发关键词。
