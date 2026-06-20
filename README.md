# delegate — 把活派给 codex / agy 的 Claude Code Skill

> A [Claude Code](https://claude.com/claude-code) skill that delegates work to your local **codex** (OpenAI) and **agy** (Google Gemini) CLI agents — Claude orchestrates, the agents execute, results come back summarized. Save your Claude quota by outsourcing tasks to other CLI agents.

<p>
  <img alt="License: MIT" src="https://img.shields.io/badge/License-MIT-blue.svg">
  <img alt="Claude Code Skill" src="https://img.shields.io/badge/Claude%20Code-Skill-d97757">
  <img alt="stars" src="https://img.shields.io/github/stars/yangkaiHandsome/claude-skill-delegate?style=social">
</p>

一个 [Claude Code](https://claude.com/claude-code) 的 **Agent Skill**。当 Claude Code 本身的额度/用量吃紧,或你想让别的命令行代理分担时,用 `/delegate` 把任务转交给本机已安装的两个 CLI 代理执行,再把结果拿回来汇总。

Claude 在这里是**调度者**:负责选执行器、构造命令、驱动运行、读取并校验输出、汇总给你——不亲自做被派出去的那部分活。

## 支持的两个执行器

| 执行器 | 命令 | 厂商 / 默认模型 | 适合 |
|--------|------|------|------|
| **codex** | `codex` | OpenAI · `gpt-5.5` | 代码改写、调试、代码审查 |
| **agy** | `agy` | Google · Gemini 3.x(也可选 Claude/GPT-OSS) | 轻量问答、分析、快速任务 |

> 本 skill 不安装这两个工具,只调用它们。你需要自行安装并登录 `codex`(OpenAI Codex CLI)和 `agy`,并确保命令在 `PATH` 中。若安装路径不同,把 `SKILL.md` 里的 `codex` / `agy` 换成对应绝对路径。

## 安装

把 `delegate` 目录放到 Claude Code 的 skills 目录下即可。

**用户级(对所有项目生效):**
```bash
git clone https://github.com/yangkaiHandsome/claude-skill-delegate.git
cp -r claude-skill-delegate/.claude/skills/delegate ~/.claude/skills/
```

**项目级(只对当前项目生效):**
```bash
cp -r claude-skill-delegate/.claude/skills/delegate <你的项目>/.claude/skills/
```

装好后在 Claude Code 里输入 `/delegate` 应该能看到它。

## 用法

直接用斜杠命令,把任务写在后面:

```
/delegate 审查 src/auth.ts 有没有越权漏洞
/delegate 用 codex 把这个项目的 ESLint 错误全修了
/delegate 让 agy 快速解释一下这段正则在干嘛
/delegate 我额度不够了,把这个重构外包出去
/delegate 用 codex 和 agy 交叉验证这个并发 bug 的根因
```

你也可以用自然语言触发,无需斜杠,例如:「**用 codex 跑一下这个测试看哪里挂了**」「**派给 agy 分析这份日志**」。

### 它会怎么做

1. **理清任务**:做什么、在哪个目录、是**只读分析**还是**要改文件**。
2. **选执行器**:你点名就用谁;没点名时——改代码/调试/审查走 codex,纯问答/分析走 agy;要交叉验证就两个都派。
3. **最小权限**:只读任务绝不给写权限(防止代理"顺手改代码")。要改文件且目录不是 git 仓库时,会先提示你建基线快照,改动才可回退。
4. **稳健调用**:长 prompt 走临时文件 + stdin 管道(规避特殊字符静默失败);长任务后台跑。
5. **校验 + 汇总**:校验输出确有结论(不把空结果当成功),提炼要点用中文汇报;若两个都派了就并排对比。

## 设计要点(为什么这么写)

这个 skill 把几个踩过的坑固化成了硬规则:

- **一律用最强思考档**:codex 强制 `-c model_reasoning_effort="xhigh"`,agy 默认 `Gemini 3.1 Pro (High)`。派出去就要它认真想。
- **权限即写权限(agy)**:agy 没有 codex 的 `read-only` 沙箱档,`--dangerously-skip-permissions` 一加就能读能写——所以只读任务绝不加它。
- **agy 的 prompt 必须走 stdin 管道**:直接当 shell 参数传,遇反引号/`$`/引号会静默失败(空输出 + exit 0)。
- **agy 输出要校验**:它常只打印过程叙述而最终结论不落 stdout;取结果后必须确认有真正结论,否则重试或换 codex。
- **codex 用 `-o` 落文件**取结果,比解析 stdout 干净。

完整规则见 [`SKILL.md`](.claude/skills/delegate/SKILL.md)。

## 额度归属

codex 走你的 OpenAI 账号,agy 走你的 Google 账号,各自计费——本 skill 的意义就是把活从当前 Claude 会话挪到它们那里,分担用量。

## License

[MIT](LICENSE)
