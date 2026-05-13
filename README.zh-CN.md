# Skills

**Languages:** [English](README.md) | [简体中文](README.zh-CN.md)

个人编写的 AI Skills 集合。Skill 是一种打包好的指令、脚本与参考资料，按需加载，可以在不污染主上下文的前提下扩展 AI 助手的能力。

## Skills 一览

| Skill | 触发场景 |
| --- | --- |
| **hono** | 使用 Hono 编写跨运行时 Web API（Cloudflare Workers、Deno、Bun、Node.js），涵盖路由、中间件、JSX、RPC 客户端、Zod 校验。 |
| **primevue** | 使用 PrimeVue 100+ 组件、设计令牌主题、unstyled / pass-through 样式、表单与校验。 |
| **radix-motion** | 用 Motion（原 Framer Motion）为 Radix UI 原语添加进入/退出、布局、手势动画，仅使用免费 API。 |
| **tailwindcss-mobile-first** | Tailwind CSS v4 的移动优先响应式设计，包含断点、容器查询、安全区域、触屏 vs 悬停。 |
| **tmux-claude-babysitter** | 在 tmux 会话中看护 Claude 实例：启动、空闲/菜单检测、错误自治恢复、状态持久化。仅在消息以 `/tmux-claude-babysitter` 开头时激活。 |
| **xiexiu** | 邪修（heretical cultivator）方法论：质疑标准答案、第一性原理、用最小可行动作找出非常规捷径。 |

## 安装

推荐使用 `npx skills` 安装：

```bash
npx skills add https://github.com/yuexiaoliang/skills
```