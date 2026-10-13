---
name: pta-auto-homework
description: 自动完成拼题A（pintia.cn / PTA）平台的上机作业题集。当用户提到"拼题A"、"pintia"、"PTA"、"上机任务/上机作业"、"帮我做作业/刷题"时触发。覆盖微信扫码登录、读题、键盘输入代码、提交判题、确认 Accepted 的完整闭环，并内置 17 个实战踩坑点的处置方案。
---

# 拼题A（PTA）上机作业自动化

## Overview

用 agent-browser（真实 Chromium + CDP 键盘事件）在拼题A平台完成整个作业闭环：登录 → 进题集 → 逐题"读题 → 写码 → 键盘录入 → 提交 → 确认 AC → 下一题"。本 skill 来自一次 13/13 满分的实战（2026-09，"第二次上机任务"，Python 3），所有坑点均现场踩过并验证了解法。

**铁律（先记住这三条，细节见 pitfalls.md）：**

1. 域名是 `https://pintia.cn`（拼题A）。`pintian.cn` 是已挂售的闲置域名，用户口述时极易混淆，先确认完整 URL。
2. PTA 代码编辑器**禁止复制粘贴**——必须用 `agent-browser keyboard type`（真实键盘事件逐键录入）。每次录入后必须 snapshot 核对编辑器内容。
3. 页面元素 ref 每次操作后都会重排。**任何 click 前都重新 snapshot 取 ref，click 后验证目标状态是否真的变了。**

## 前置条件

- `agent-browser` CLI 已安装（`npm install -g agent-browser && agent-browser install`）
- 用户能收到图片（用于发微信登录二维码）
- 编程语言：进题后看编辑器上方 combobox 的语言，本课程题集默认 `Python (python3)`；如果题目语言不对，先切语言再写码

## 主工作流

### Step 0 登录（微信扫码）

```bash
agent-browser open https://pintia.cn/auth/login
agent-browser wait --load networkidle    # 卡住就降级为 wait --load load
agent-browser snapshot -i                # 找 link "Wechat Login"
agent-browser click <Wechat Login 的 ref>
```

出现 "Please use WeChat to scan this QR code" 后**立即**截图（二维码有时效）：

```bash
agent-browser screenshot /root/.codebuddy/artifact/wechat_qr.png
# 用 mobile_share_image 把二维码发到用户手机，提醒：保存图片 → 微信扫一扫 → 相册选图
```

然后等用户回复"扫完了"，用 snapshot 确认已进入登录后的首页（出现题集卡片才算成功）。**浏览器 daemon 全程不要 close**，登录态依赖它。

### Step 1 进入题集并开始答题

1. 首页 snapshot 找目标题集链接（如 "第二次上机任务 Will close ..."），click 进入
2. 点击 `button "Start Answering"`（"Ready" 状态才有此按钮）
3. 出现题目表格（Label / Title / Score / Acceptance Rate）即成功，记录题目总数

### Step 2 单题标准循环（对每道题重复）

```
读题 → 定位编辑器 → 键盘录入 → 核对 → 提交 → 确认 AC → 下一题
```

**2.1 切题 / 读题**

```bash
# 切题优先用 Next/Previous 按钮；点击后必须验证标题变了没有
agent-browser click <Next 的 ref>
agent-browser snapshot | grep -E '2-[0-9]+ ' | head -2   # 验证当前题号
# 若标题没变：重新取 ref 再点一次（Next 首次点击常被吞，见坑 P4）
```

读题用这个过滤，能拿到题目描述 + 输入输出样例，且滤掉噪音按钮：

```bash
agent-browser snapshot | sed -n '/Problem Description/,/Code Size Limit/p' \
  | grep -vE '复制内容|格式|全屏|clickable'
```

注意：题目段落可能被折叠（有"收起"按钮），长题务必展开读全；**输出样例要逐字符看**（含行尾空格、逗号后的空格）。

**2.2 定位编辑器并录入代码**

```bash
# 编辑器 ref 不固定（e50/e51 都出现过），每次动态定位：
editor=$(agent-browser snapshot | sed -n '/Stack size limit/,/Test Case/p' \
  | grep -oE 'textbox \[ref=e[0-9]+\]' | grep -oE 'e[0-9]+' | head -1)
agent-browser click e$editor
agent-browser keyboard type "<完整代码，含换行>"
```

录入后**必须核对**（普通字符 + 中文都要看）：

```bash
agent-browser snapshot | grep -A 4 "textbox \[ref=$editor\]"
```

**2.3 提交与判题确认**

```bash
sub=$(agent-browser snapshot | grep -oE 'button "Submit For This Problem" \[ref=e[0-9]+\]' \
  | grep -oE 'e[0-9]+' | head -1)
agent-browser click e$sub
sleep 10
agent-browser snapshot | sed -n '/Submit Result/,/Submission Detail/p' \
  | grep -E '"(Accepted|Wrong Answer|Waiting|[0-9]+ / [0-9]+|- / [0-9]+)"|In queue'
```

- `Accepted N/M` → 本题完成，进入下一题
- `In queue, XX submissions ahead` / `Waiting` → 判题排队中。等 30~60 秒后点结果面板里的 `button "Reload"` 刷新，**不要重复提交**（见坑 P7）
- `Wrong Answer` → 回读题区逐字符比对样例（重点查：行尾空格、分隔符、小数位数、前导 0），改代码重交

**2.4 进度确认**

侧边栏顶部 `编程题 N/13` 是全局进度；全部做完后到 Submission list 核对每题记录（见坑 P8），最后全页截图留档发给用户。

## 读题 → 写码的转换心法

本平台作业以 Python 格式化输出为主，按样例反推格式串即可，不要凭题面猜。高频映射见 `references/code-patterns.md`：

- "控制长度，左边补空格" → `'{:3}'.format()` / `rjust()`
- "右边补空格" → `'{:<3}'.format()` / `ljust()`
- "前导 0" → `str(n).zfill(2)`
- "保留 N 位小数" → `'%.Nf'`（注意 `%.3f` 是四舍五入）
- "用 X 分隔" → `print(a, b, sep='X')`；"最后一项后不能有分隔符" 正是 sep 的意义
- 输入样例 `3, 5`（逗号后带空格）→ `split(',')` 后 `int()` 自带去空格，无需 strip

## 资源

- `references/pitfalls.md` — 17 个实战踩坑点：现象 → 根因 → 处置。**开工前先通读一遍**。
- `references/code-patterns.md` — 高频题型 Python 代码模式库（含易错点注释）。
