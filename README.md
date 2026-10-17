# wechat-exporter

面向 Claude Code、Codex 等 coding agent 的技能与实现规范，用于在授权范围内将 macOS 微信 4.x 的本地 SQLCipher 数据库恢复为标准 SQLite 数据库。

本项目不再依赖下载旧上游的扫描与解密脚本。即使原依赖不可用，Agent 也可依据仓库内的完整指南自行实现、测试和验收。当前仓库提供文档，不附带可直接运行的解密器或 MCP Server。

## 项目内容

| 文件 | 用途 |
| --- | --- |
| [SKILL.md](SKILL.md) | Agent 入口：授权、实现顺序、测试、导出与交付要求 |
| [references/agent-decrypt-guide.md](references/agent-decrypt-guide.md) | 完整实现规范：密钥形态、页面认证与解密、一致快照、WAL、验收、测试和实现骨架 |
| [references/database-schema.md](references/database-schema.md) | 历史数据库结构与查询参考，使用前须核验当前版本 |

实现指南由 robbin 编写，从 [原始 Gist](https://gist.github.com/robbin/db022e901b0e44690606aa3ffaf7d0e8) 完整收录，来源版本为 `9b3be014245c156856db9f501877da54c50dea07`，文件名为 `agent-decrypt-guide.md`。仓库副本无需在线访问 Gist 即可阅读；后续更新需要显式同步和审阅。

## 能力与边界

- 已取得密钥后，指南完整描述页面格式、HMAC 认证、逐页解密、通过 SQLCipher 处理 WAL 及独立验收。
- 密钥获取的分支 A 提供完整实现思路；分支 B 的 passphrase 获取依赖具体客户端版本，不能保证仅凭文档即可完成。不要仅凭版本号判断分支。
- 输出目标是标准 SQLite 数据库，不是已解析好的聊天文本。正文编码、压缩、联系人与附件关联是后续工作。
- 默认 WAL 恢复和独立交叉验证仍使用 SQLCipher；“不依赖旧上游脚本”不等于不需要任何外部工具。
- MCP 服务需另行实现或适配，不是安装本技能后的现成功能。

## 前置条件

- macOS 与本机微信 4.x 数据库；目标记录必须实际存在于客户端的 `db_storage` 中，不保证包含手机端全部历史，也不处理任意手机备份归档格式。
- 仅限本人设备、本人账号和本人授权的数据；先确认要处理哪些账号。
- 可执行开发任务的 Agent，以及实现所需的工具链。Python 环境使用 `uv`，系统工具使用 Homebrew，安装前先检查是否已有。
- 按系统实际要求授予数据目录访问权限；读取进程内存另有权限与签名条件，详见指南第 3.2 节。

应用重签名可能影响钥匙串、权限授权、登录与更新，必须先说明风险并获得明确授权。不要因版本更新自动重签名或关闭 SIP。

## 快速开始

### 1. 安装完整技能目录

不能只复制 `SKILL.md`，它依赖同目录下的 `references/`。

在本仓库根目录执行，以下以 Claude Code 全局技能目录为例：

```bash
mkdir -p ~/.claude/skills/wechat-exporter
cp SKILL.md ~/.claude/skills/wechat-exporter/SKILL.md
cp -R references ~/.claude/skills/wechat-exporter/
```

项目级安装可将 `~/.claude/skills/` 换成 `<项目>/.claude/skills/`。其他 Agent 使用其支持的技能目录，保持 `SKILL.md` 与 `references/` 的相对结构；也可直接让 Agent 阅读本仓库的 `SKILL.md`。

### 2. 让 Agent 实现并验证

建议首次使用时输入：

```text
请阅读 wechat-exporter 的 SKILL.md 和完整的 references/agent-decrypt-guide.md。
先依据规范实现工具，用 SQLCipher 合成数据验证解密、WAL 和失败分支。
先不要读取我的微信数据、重签名或运行提权命令；完成后报告测试结果和后续所需授权。
```

已有通过验证的实现后，再要求导出指定账号。流程应当是：

```text
明确授权与范围 → 实现与合成测试 → 运行中取得并验证密钥
→ 停写并取得一致快照 → 工作副本上处理 WAL → 逐页认证与解密
→ 独立内容比对 → 逐库报告
```

### 3. 查看结果

工具工作目录和输出目录由使用者确认，建议位于源码仓库之外。多账号输出须隔离，例如：

```text
decrypted/
└── <账号标识>/
    ├── contact/
    │   └── contact.db
    ├── message/
    │   ├── message_0.db
    │   └── ...
    └── session/
        └── session.db
```

实际路径以发现结果为准。通过验收的产物可用普通 SQLite 工具只读查询；失败或受限恢复产物必须单独标记，不混为成功结果。不要将源密文 WAL/SHM 放到明文输出目录。

## 怎样判断成功

验收细则见指南第 9 节，必须分别报告：

1. 主库解密通过：全页 HMAC、页面布局与尺寸、SQLite 结构检查和独立内容比对。
2. 快照恢复通过：一致采集，正确处理有效已提交 WAL，并明确快照时点。
3. 授权范围全部处理并报告：每个目标库都有结论，包括缺密钥、失败、不支持、跳过和受限恢复；不表示全部成功。

只看 SQLite 文件头、表名、行数或 `integrity_check=ok` 不足以证明完整恢复。“最新”仅指所采集一致快照中的有效已提交状态，不代表客户端当前状态。

## 常见问题

| 问题 | 处理方向 |
| --- | --- |
| 原依赖仓库不可用 | 使用内置指南实现，不继续下载旧脚本，也不把未知镜像作为默认替代 |
| 扫描不到密钥 | 排查权限、目标进程、扫描覆盖和库是否已加载，再评估版本分支 |
| HMAC 失败 | 检查密钥与 salt、账号隔离、第一页偏移、派生参数和输入完整性；不能只归因于账号配置 |
| checkpoint 的 busy 为 0 | 仍须检查 log 与 checkpointed，按指南第 8.5 节完整判定 |
| 某个库失败 | 单独报告原因，不能预设“一个失败是正常的” |
| 解密后正文不可读 | 检查当前版本的表结构、正文编码或压缩；这不由页面解密自动解决 |
| 需要 MCP 查询 | 对已验收的明文库另行实现或适配只读服务，再验证客户端配置 |

## 隐私与安全

密钥、明文数据库、消息正文和内存转储不得进入版本控制、日志或对外输出。密钥如需保存，文件权限设为 `600`；不要把真实密钥放进 shell 命令历史或进程参数。原始库不得覆盖，checkpoint 只在可丢弃副本上进行。无法取得一致快照时，不宣称快照恢复成功。

## 历史来源与致谢

旧版工作流使用以下项目，保留链接用于来源记录，不再作为安装步骤或可用性承诺：

- [cocohahaha/wechat-decrypt-macos](https://github.com/cocohahaha/wechat-decrypt-macos) — 旧版 MCP 服务来源
- [ylytdeng/wechat-decrypt](https://github.com/ylytdeng/wechat-decrypt) — 旧版扫描与解密脚本来源

新实现应依据内置规范独立编写，不暗示本仓库包含或重新发布了上述项目的源码。公开发布与使用边界见指南第 13 节。
