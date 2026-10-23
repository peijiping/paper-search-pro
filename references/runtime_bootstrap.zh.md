# 运行时引导 —— `$PSP_HOME` 解析 + UI 语言路由

`SKILL.md` 的 STEP 0 / STEP 1 对这两件确定性的准备工作只保留了最简版本，这里是详细参考。只有当正文里的精简版本对你的情况不够用时才读本文件（例如环境变量注入失败，或者你需要跨 Agent 的路径清单，或者你需要推理一个非中文 / 非英文的检索式）。

---

## 1. `$PSP_HOME` —— Skill 的安装目录

所有辅助脚本都以 `PYTHONPATH=$PSP_HOME python3 -m scripts.<name> …` 的形式调用，因此 `$PSP_HOME` 必须指向包含这个 `SKILL.md` 的目录。它在 STEP 0 里**只解析一次**，之后处处复用。

### 为什么是三层链

本 Skill 以可移植的 SKILL.md 包形式分发，不同 Agent 会把它装在不同路径下。这条链优先尝试代价最低、也最权威的来源：

- **第 1 层 —— 显式注入。** 如果你的宿主环境已经暴露了这个 `SKILL.md` 的绝对路径，执行 `export PSP_HOME="<该目录>"`，然后跳过其余部分。这是最可靠的来源——它是基准真值。
- **第 2 层 —— Agent 注入的环境变量。** Claude Code / CodeBuddy 会填 `CLAUDE_SKILL_DIR` / `CODEBUDDY_SKILL_DIR`。精简版的 STEP 0 代码片段会自动读取它们。
- **第 3 层 —— 文件系统遍历。** 如果上面两者都没设置，就扫描已知的跨 Agent 安装位置（见下文），取第一个真正含有 `SKILL.md` 的目录。这是正文片段里跑的回退方案；这里完整复现一遍，方便你推理 / 扩展它。

```bash
PSP_HOME="${PSP_HOME:-${CLAUDE_SKILL_DIR:-${CODEBUDDY_SKILL_DIR:-}}}"
if [ -z "$PSP_HOME" ]; then
  for d in \
    "$HOME/.claude/skills/paper-search-pro" \
    "$HOME/.codex/skills/paper-search-pro" \
    "$HOME/.agents/skills/paper-search-pro" \
    "$HOME/.config/opencode/skills/paper-search-pro" \
    "$HOME/.codeium/windsurf/skills/paper-search-pro" \
    "$HOME/.config/goose/skills/paper-search-pro" \
    "$HOME/.cline/skills/paper-search-pro" \
    "$HOME/.roo/skills/paper-search-pro" \
    "$HOME/.copilot/skills/paper-search-pro" \
    "./.claude/skills/paper-search-pro" \
    "./.codex/skills/paper-search-pro" \
    "./.agents/skills/paper-search-pro" \
    "./.cursor/skills/paper-search-pro" \
    "./.opencode/skills/paper-search-pro" \
    "./.windsurf/skills/paper-search-pro"; do
    [ -f "$d/SKILL.md" ] && PSP_HOME="$d" && break
  done
fi
[ -z "$PSP_HOME" ] && { echo "ERROR: paper-search-pro install not found. Set PSP_HOME to the directory containing SKILL.md."; exit 1; }
export PSP_HOME
echo "Using Skill install: $PSP_HOME"
```

### 为什么这段留在正文里，而不是写成脚本

`$PSP_HOME` 正是你*用来找到所有脚本*的路径——所以解析它的过程本身不可能是脚本（先有鸡还是先有蛋）。它还必须 `export` 进 Agent 的工作 shell，而子 shell 脚本做不到这件事。因此 STEP 0 在正文里保留了一份紧凑、自包含的版本；本文件负责解释和完整的路径清单。

### 与"绝不 `cd` 进 Skill 目录"（规则 A）的关系

解析出 `$PSP_HOME` 正是让你*能够*避开 `cd` 的前提。你在用户的 PWD 下用 `PYTHONPATH=$PSP_HOME` 运行每一个辅助脚本，于是 `./paper-search-results/...` 依然解析到用户的工作目录，而不是 Skill 的资源目录。

---

## 2. UI 语言路由（HTML 报告用的 `UI_LANG`）

STEP 1 把 `UI_LANG` 设为 `en` 或 `zh`；这个单一布尔值只决定最终 HTML 报告用哪种 UI 语言渲染。论文标题 / 摘要 / 作者 / 期刊名**绝不**翻译——只翻译报告的 UI 外壳。

确定性检测器是 `scripts/detect_language.py`：

```bash
UI_LANG=$(PYTHONPATH=$PSP_HOME python3 -m scripts.detect_language "$USER_QUERY")
```

### 路由规则（先匹配者胜）

1. **出现日文假名**（平假名 U+3041–U+309F 或片假名 U+30A0–U+30FF）
   → `en`。日文汉字与中文汉字共用 CJK 统一表意文字区，所以像 `東京大学の最新研究` 这样的纯汉字检索式，在码位层面看起来和中文一模一样。先检测假名，能可靠地把真正的日文文本路由到 EN。
2. **出现 CJK 统一表意文字**（U+4E00–U+9FFF），且没有假名 → `zh`（中文）。
3. **其余一切**（拉丁字母、谚文/韩文、西里尔字母、数字）→ `en`。

### 为什么非中文的 CJK 路由到英文

资源包里只带英文和中文两套词典。日文 / 韩文 / 欧洲语言的检索式一律路由到**英文**——国际学术通用语言；让一位韩国研究者读英文界面，比让同一位研究者面对一个根本读不懂的中文界面友好得多。（把日文 / 韩文路由到中文是早先的启发式做法；之所以改掉，是因为"CJK 读者能读中文界面"这个前提并不成立。）

`UI_LANG` 在 STEP 12b 通过 `--language $UI_LANG` 传入。渲染器内部的解析顺序是：显式 `--language` > `metadata.language` > `en`。
