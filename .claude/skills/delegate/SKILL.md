---
name: delegate
description: 把一部分工作派给本机的 codex(OpenAI)或 agy(Google)命令行代理去干,收集并汇总它们的结果。当用户说"用 codex/agy 做…""派给 codex/agy""让 codex/agy 跑一下""我的用量不够了,外包给…"时使用。
user-invocable: true
allowed-tools:
  - Bash
  - Read
  - Write
---

# /delegate — 把活派给 codex 或 agy

当本机额度/用量吃紧,或用户想让外部代理分担时,用本 skill 把任务转交给本机已安装的两个命令行代理执行,然后把结果拿回来汇总给用户。**你是调度者,不是执行者**:负责选执行器、构造命令、驱动运行、读取输出、汇总,不要自己动手做那部分被派出去的活。

用户传入的任务:`$ARGUMENTS`

> 前置要求:本机已安装并登录 `codex`(OpenAI Codex CLI)与 `agy`(Google 的命令行代理)。命令名假定在 `PATH` 中可直接调用;若你的安装位置不同,把下文 `codex` / `agy` 换成对应绝对路径即可。

---

## 两个执行器

| 执行器 | 命令 | 厂商/默认模型 | 适合 |
|--------|------|------|------|
| **codex** | `codex` | OpenAI · `gpt-5.5` | 代码改写、调试、代码审查(有专门 `codex exec review`) |
| **agy** | `agy` | Google · Gemini 3.x(也可选 Claude/GPT-OSS) | 轻量问答、分析、快速任务,启动开销小 |

`agy` 可用模型(`agy models` 查看)随版本变化,例如:Gemini 3.5 Flash (Low/Medium/High)、Gemini 3.1 Pro (Low/High)、Claude Sonnet 4.6 (Thinking)、Claude Opus 4.6 (Thinking)、GPT-OSS 120B (Medium)。用 `--model "<名称>"` 指定。

### ⚠️ 一律用最强思考档(强制)
派出去就要它认真想,**默认必须选每个执行器的最强推理档**,除非用户明确说要省钱/要快:
- **codex**:必加 `-c model_reasoning_effort="xhigh"`(gpt-5.5 的最高档,比 high 更强;若你的全局默认只是 medium,不覆盖就不是最强)。
- **agy**:默认 `--model "Gemini 3.1 Pro (High)"`(Pro 比 Flash 强,High 是 Pro 的最高思考档)。若用户点名要 Claude,则用 `--model "Claude Opus 4.6 (Thinking)"`(带 Thinking 即开启推理)。**不要用不带 High/Thinking 的档**。

### 选谁
- 用户**明确点名**就用谁。
- 没点名:**改代码/调试/审查 → codex**;**纯问答/分析/快速活 → agy**;用户要**交叉验证**就两个都派、最后对比结论。
- 拿不准时,用 AskUserQuestion 问一句,别擅自两个都跑(省额度)。

### 任务 → 权限映射(先定权限再发命令)
派之前先判这是**只读**还是**要改文件**,据此给最小权限——别把"非交互执行"误当成"该给写权限":
- **审查 / 分析 / 问答 / 解释 / 调研 → 只读**:codex 用 `-s read-only`;agy **不加** `--dangerously-skip-permissions`(它无 read-only 档,加了就是给写权限)。
- **改代码 / 调试修复 / 重构 → 写**:codex 用 `-s workspace-write`;agy 才加 `--dangerously-skip-permissions`。
- 任务措辞含"审查/看看/评估/有没有问题"却**没让改** = 只读;别让执行器"顺手把发现的问题改了"。

---

## 调用模板

### codex(非交互)
```bash
# 纯分析/问答:read-only 更快更安全,不改文件
codex exec --skip-git-repo-check -s read-only -c model_reasoning_effort="xhigh" \
  -C <工作目录> -o /tmp/codex_out.md "<任务描述>"

# 需要它改代码:workspace-write(默认),只写工作区+/tmp
codex exec --skip-git-repo-check -s workspace-write -c model_reasoning_effort="xhigh" \
  -C <项目目录> -o /tmp/codex_out.md "<任务描述>"
```
- **必加** `--skip-git-repo-check`,否则在非 git 目录会报 "Not inside a trusted directory"。
- **必加** `-c model_reasoning_effort="xhigh"`,用最强推理档(见上「一律用最强思考档」)。
- `-C <dir>` 指定工作根目录;不给则用当前目录。
- `-o <file>` 把**最后一条回复**写到文件——比解析 stdout 干净(stdout 含一大段头信息+token 统计)。运行后 Read 这个文件拿结果。
- 选模型:`-m <model>`。需要结构化输出:`--output-schema <json-schema文件>`。

### agy(非交互)
```bash
# 审查/分析/问答(只读):务必经 stdin 管道喂 prompt,不传 --dangerously-skip-permissions
cat /tmp/delegate_prompt.txt | agy -p "$(cat)" --add-dir <工作目录> \
  --model "Gemini 3.1 Pro (High)" --print-timeout 15m > /tmp/agy_out.md 2>&1

# 仅当任务明确要改代码:才加 skip 权限(它会既读又写,无 read-only 沙箱档)
cat /tmp/delegate_prompt.txt | agy -p "$(cat)" --add-dir <项目目录> \
  --model "Gemini 3.1 Pro (High)" --dangerously-skip-permissions > /tmp/agy_out.md 2>&1

# 用户点名要 Claude 时,用带 Thinking 的最强档
agy -p "<短任务描述>" --add-dir <工作目录> --model "Claude Opus 4.6 (Thinking)"
```
- **必加** `--model`,默认 `"Gemini 3.1 Pro (High)"`,用最强推理档(见上「一律用最强思考档」)。
- ⚠️ **权限即写权限**:agy **没有 codex 那种 `read-only` 沙箱档**;`--dangerously-skip-permissions` 一加,它**既能读也能写**。**审查/分析/问答类只读任务绝不要加它**——宁可它卡在权限询问,也别给它改文件的口子(否则它可能擅自"顺手改代码",且无 git 时不可回退)。只有用户明确要它改代码才加。
- ⚠️ **长/含代码的 prompt 必须经 stdin 管道**(`cat file | agy -p "$(cat)"`):直接当 shell 参数 `"$PROMPT"` 传,遇反引号/`$`/引号会**静默失败(空输出 + exit 0)**。prompt 先 Write 到 `/tmp/delegate_prompt.txt` 再管道喂。
- ⚠️ **stdout 不一定可靠**:涉及读多文件/长输出的任务,agy 常只打印过程叙述("I'll read files… now I'll produce the report")而**最终结论不落 stdout**。**取结果后必须校验输出非空且含真正结论**,空了就改用 stdin 管道重试,仍空则换 codex(codex 的 `-o` 落文件稳定得多)。
- agy **无结构化输出能力**(不像 codex 的 `--output-schema`);要可靠的结构化结果优先选 codex。
- `-p`(= `--print`/`--prompt`)单次非交互执行。默认 print 超时 5 分钟,长任务用 `--print-timeout <时长>` 调大(如 `15m`)。

---

## 执行流程

1. **理清任务**:从 `$ARGUMENTS` 和上下文确定——做什么、在哪个目录、**是只读分析还是要改文件**(决定权限,见「任务→权限映射」)。任务太含糊就先问清楚。
2. **选执行器**(见上)。
3. **(仅写任务)跑前先建基线**:要给执行器写权限前,先 `git -C <目录> rev-parse --is-inside-work-tree` 查是否在 git 仓库。**不是 git 仓库**则改动不可回退——提示用户先 `git init && git add -A && git commit -m baseline`(或别的快照),用户同意建好基线再跑。只读任务跳过此步。
4. **构造命令并运行**:
   - prompt 一律先 Write 到 `/tmp/delegate_prompt.txt`,再 `"$(cat ...)"`(codex)或 **stdin 管道**(agy:`cat 文件 | agy -p "$(cat)"`)喂入,避免转义/特殊字符静默失败。
   - **长任务用后台执行**(`run_in_background: true`),并把 Bash `timeout` 设大(如 300000~600000ms);完成后再读输出。短任务前台跑即可。
5. **取结果**:codex 读 `-o` 指定的文件;agy 读重定向的 `/tmp/agy_out.md`。**agy 必校验**:输出为空或只有过程叙述(无真正结论)= 失败,按 agy 注意事项重试或换 codex,**别把空结果当成功**。
6. **汇总**:用中文向用户复述执行器的关键产出与改动(改了哪些文件、结论是什么)。**别原样粘贴**整段噪声输出,提炼要点。两个都派了就并排对比。
7. **核对改动**:本以为是只读任务,也要查执行器**有没有擅自改文件**(`git status`,或非 git 时 `find <目录> -newermt <开跑时刻> -name '*.ts'` 等比对修改时间)。改了就如实告诉用户、连同 `git diff`(有 git 时)让他决定保留还是回退。**无 git 又无基线时改动不可逆,务必在第 3 步就拦住。**

---

## 注意事项

- **额度归属**:codex 走用户的 OpenAI 账号,agy 走 Google 账号,各自计费/耗额度——本 skill 的意义就是把活从当前会话挪到它们那。
- **认证**:首次使用需各自登录(`codex login`,以及按 agy 自身方式登录)。若报未登录,提示用户去登录,不要替他登录。
- **沙箱边界**:codex 默认 `workspace-write` 只写工作区和临时目录;要它在工作区外写文件需 `--add-dir` 或更高权限,谨慎使用 `danger-full-access` / `--dangerously-bypass-approvals-and-sandbox`,改前向用户确认。
- **不要套娃**:被派出去的代理就让它独立完成那块活,你只在外层调度和汇总,避免你和它来回交叉做同一件事。
- 失败时:读 stderr,把真实错误如实转述给用户,必要时换执行器或调参重试,不要假装成功。
