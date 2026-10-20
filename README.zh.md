# stackmap-check

给编码 Agent 用的只读文档导航工具。

Agent 想回答"这个写在哪儿"时，常见做法是把一堆文件读一遍。这个工具把它变成两步：**先定位**少量候选文档，**再读原文的对应章节**。它不修改文件、不启动服务、不访问网络。

```
stackmap query --query <词>              # 每份文档一行，最多 20 份
stackmap query --query <词> --count      # 有多少份命中，以及穷举要多少 token
stackmap query --query <词> --all        # 穷举：每份文档一行
stackmap query --query <词> --select entries   # 逐标题 + 240 字引文
stackmap boot --task <标签>              # 有精选目录时，列该任务的入口
stackmap check                           # 校验精选记录与当前提交是否一致
```

## 为什么需要

想找一段话，却要读整棵文档树，是最贵的找法。在一个 267 份文档的仓库上实测：整棵树约 2.9 MB（约 90 万 token，任何上下文都放不下）；一次典型的"这写在哪儿"查询返回 20 份文档，约 1.3k token、1 秒。Agent 随后只读一两段原文，而不是整棵树。

它是**定位器**，不是权威：告诉你**去哪儿看**，不告诉你那段话是否正确、是否现行、是否完整。

## 安装

零依赖。需要 Node.js ≥ 22 和 `git`。

```sh
git clone <本仓库> stackmap
node stackmap/cli.mjs query --query <词> --root /path/to/project
```

或者直接把 `cli.mjs` 拷进任意仓库，`node cli.mjs ...` 即可。不需要其它文件。

## 它怎么读一个仓库

- 只扫 **git 跟踪的 Markdown**；未跟踪笔记和其它文件不参与。
- 排除名为 `archive` 的目录和 `handoff/debt-marathon/waves/`（历史材料）。
- 字面匹配（大小写不敏感），命中路径、标题和正文——不是语义检索。
- 从 git 索引或指定提交读原文，不读工作树：行尾差异和未暂存修改不会污染结果。身份以 git blob 为准。
- `staged_change` / `unstaged_change` 分开报告；工作树可编辑，但不会悄悄改变查询结果。

## 模式与成本

| 模式 | 计数单位 | 实测成本 |
|---|---|---|
| 默认（`--limit 20`） | 文档 | 20 份 ≈ 1.3k token |
| `--count` | — | 几个 token |
| `--all` | 文档 | 112 份 ≈ 4.5k token |
| `--select entries` | 标题 + 240 字引文 | 每条约 400 字符 |

默认 `--limit` 数文档；`--select entries` 改成数标题。截断时打印 `Showing 20/N`，并给出 `--all` 的估算成本。

## 退出码

| 码 | 含义 |
|---|---|
| `0` | 本次所列导航未发现问题 |
| `1` | 所列记录需要核对（版本失配、工作树修改、文档不可读） |
| `2` | 参数非法、目录不可读、git 失败 |

退出码描述这次运行，不描述文档正确性。

## 可选的精选目录

仓库可以用 `catalog/records.json`（或 `handoff/catalog/records.json`，或 `--catalog PATH`）登记少量高频文档：

```json
{
  "schema_version": 1,
  "roots": ["docs", "README.md"],
  "records": [
    {
      "id": "guide",
      "path": "docs/guide.md",
      "purpose": "如何安装",
      "tags": ["docs"],
      "source_ref": { "scheme": "git-blob", "object_format": "sha1", "oid": "<blob id>" },
      "entries": [
        { "key": "install", "label": "安装", "locator": { "kind": "heading-path", "value": ["Install"] } }
      ]
    }
  ]
}
```

没有目录也能跑：只做即时导航，没有精选入口。

## 评测

`eval.mjs` 衡量"它到底能不能找到任务需要的文档"。用例由各仓库自己提供：

```json
{
  "locate": [{ "q": "build_prompt", "expect": ["docs/prompt.md"] }],
  "coupling": ["module-compat"]
}
```

```sh
node eval.mjs --cases eval.cases.json          # 只报告
node eval.mjs --cases eval.cases.json --gate   # 不达线时退出 1
```

`locate` 报告 8/12/20 份文档下的召回；`coupling` 用 `--all` 与全文基线对比完整性。示例见 `eval.cases.example.json`。

## 边界（信任它之前请读）

- **字面匹配**：同义词、改写会漏。请用标识符、命令名、ID。
- **窗口**：命中超过 20 份时先用 `--count`，再决定要不要 `--all`。
- **覆盖 ≠ 语义**：它发现不了文档之间的矛盾、隐式约定和间接影响；那些要靠审查规则。
- **没有相关性排序**：顺序是精选优先、再按路径与行号。把结果当候选集。

## 开发

```sh
node --test cli.test.mjs
```

测试会建临时 git 仓库，覆盖发现、引文、目录校验、提交隔离、行尾和退出码。

## 许可

Apache-2.0，见 `LICENSE`。
