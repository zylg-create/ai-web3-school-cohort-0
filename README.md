# AI × Web3 School - Personal Learning Journal

## 简介

这是我在 [AI × Web3 School](https://aiweb3.school) 的个人学习仓库，记录学习过程、任务证明和实验记录。

## 资源链接

- **Handbook**: https://aiweb3.school/zh/handbook/
- **WCB 课程页面**: https://web3career.build/zh/programs/AI-Web3-School
- **WCB Learning**: https://web3career.build/zh/programs/AI-Web3-School#tab=learning
- **WCB Agent API 文档**: https://web3career.build/llms.txt

## 隐私提醒

⚠️ 本仓库为 **public（公开）** 仓库。

**请勿** 在本仓库中存放以下内容：
- API Key / Secret Key
- 钱包助记词、私钥
- 密码、内部链接
- 他人个人信息

## 目录说明

```
├── README.md           # 本文件
├── profile.md          # 个人学习画像
├── learning-plan.md    # 学习计划
├── daily/             # 每日学习打卡笔记
├── tasks/             # 任务记录
├── experiments/       # 实验与代码
├── handbook-feedback/ # Handbook 反馈与建议
├── hackathon/         # Hackathon 项目
├── submissions/       # 提交记录
└── templates/         # 模板文件
    ├── daily-note.md  # 每日笔记模板
    └── task-note.md   # 任务笔记模板
```

## 关于 AI × Web3 School

AI × Web3 School 是面向 builders 的开源学习计划，由 LXDAO 与 ETHPanda 共同发起。目标是建立 AI 与 Web3 交叉领域的共同语言，涵盖模型能力、Agent 工作流、钱包签名、支付结算、身份权限、安全执行和治理协作等核心议题。

更多信息请阅读 [Handbook](https://aiweb3.school/zh/handbook/)。

## Week 1 学习目标

**目标方向**：开发（基于 Web3 基础好、能借助 AI 开发的背景）

**本周重点**：
1. 阅读 Handbook AI 基础章节（LLM、Prompt、Context）
2. 了解 WCB 课程安排和 Hackathon 方向
3. 熟悉 AI × Web3 Bridge 核心概念
4. 建立每日学习打卡习惯

**记录结构**：
- `daily/` - 每日学习打卡笔记（YYYY-MM-DD.md）
- `tasks/` - 任务完成记录
- `experiments/` - 代码实验和 Demo
- `handbook-feedback/` - Handbook 阅读反馈与建议
- `hackathon/` - Hackathon 项目准备
- `submissions/` - 任务提交记录

## Agent 初始化说明

本仓库通过 **Hermes Agent** 初始化，初始化过程如下：

**Agent 做了什么**：
- 安装 GitHub CLI (`gh`) 到用户目录 `~/bin/`
- 通过 `gh auth login` 完成 GitHub 登录
- 执行 `gh repo create` 创建仓库
- 初始化目录结构（daily/、tasks/、experiments/ 等）
- 创建 README.md、profile.md、learning-plan.md、templates/
- 生成今日打卡草稿 `daily/2025-05-25.md`
- 执行 `git add .`、`git commit`、`git push`

**人工确认内容**：
- 学员画像（AI 基础、Web3 基础、编程能力、目标方向）
- GitHub 用户信息和提交配置
- 仓库名、可见性（public）、本地路径
- commit 前展示变更内容，确认后执行提交
