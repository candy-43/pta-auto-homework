# pta-auto-homework

拼题A（pintia.cn / PTA）上机作业自动化的 Agent Skill。

用 agent-browser（真实 Chromium + CDP 键盘事件）完成整个作业闭环：**微信扫码登录 → 进题集 → 逐题读题 → 键盘录入代码 → 提交判题 → 确认 Accepted → 下一题**。

## 实战验证

| 场次 | 成绩 |
|------|------|
| 第二次上机任务（2026-09-02） | 13/13 满分 |
| 第三次上机任务（2026-09-10） | 20/20 满分 |
| 第四次上机任务（2026-09-15） | 20/20 满分 |

## 结构

```
pta-auto-homework/
├── SKILL.md                 # 主工作流：登录 / 读题 / 录入 / 提交 / 确认 AC
└── references/
    ├── pitfalls.md          # 17 个实战踩坑点：现象 → 根因 → 处置
    └── code-patterns.md     # 高频题型 Python 3 代码模式库
```

## 使用

1. 把本目录放入你所用的 Agent 的 skills 目录（如 WorkBuddy 的 `~/.workbuddy/skills/`）
2. 前置依赖：`npm install -g agent-browser && agent-browser install`
3. 对 Agent 说"帮我做 PTA 上机任务刷题"即可触发

## 免责声明

本项目是浏览器自动化与 Agent Skill 的技术实践记录。请在遵守拼题A平台服务条款与所在课程学术规范的前提下使用。

## License

[MIT](./LICENSE)

## Buy Me a Coffee

如果这个项目帮到了你，欢迎微信扫码请我喝杯咖啡：

<p align="center">
  <img src="assets/wechat-reward-qr.png" alt="微信收款码" width="260">
</p>
