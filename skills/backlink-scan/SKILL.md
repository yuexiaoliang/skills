---
name: backlink-scan
description: '扫描域名发现新 URL，用于外链建设项目的 URL 发现阶段。仅当用户消息以 /backlink-scan 开头时激活。触发词：扫描域名、发现 URL、backlink scan、域名扫描、sitemap 扫描。'
metadata:
  author: yuexiaoliang
  version: '1.0.0'
---

# 扫描域名发现新 URL

**触发方式**：用户消息以 `/backlink-scan` 开头时激活。

读取 `targets.txt` 中的域名列表，逐个检查分析间隔，对到期域名执行完整扫描，发现新增 URL 并记录。

## 文件结构

```
urls/
  {域名}.json          — 按域名隔离，存储该域名下所有 URL 及各平台铺设状态
monitored-domains.json — 各域名监控状态（last_analyzed_at 等）
targets.txt            — 目标域名列表
```

---

## 执行流程（Claude 全自动）

### Step 1：读取配置并判断执行时机

1. 读取项目根目录 `targets.txt`，每行一个域名：

```
example.com
another-site.org
```

2. 读取 `monitored-domains.json`（如存在），获取各域名的 `last_analyzed_at` 和 `scan_interval_hours`（默认 24 小时）。
3. **执行时机判断**：逐个检查域名：
   - **当前时间 - last_analyzed_at >= scan_interval_hours**：执行扫描。
   - **当前时间 - last_analyzed_at < scan_interval_hours**：跳过该域名，记录"距离下次分析还有X小时"。
   - **新域名无记录**：视为从未分析过，立即执行扫描，`scan_interval_hours` 默认设为 24。

### Step 2：域名扫描与发现

对需要扫描的域名执行：

1. 爬取域名的 `sitemap.xml`、RSS 订阅源、根目录索引等，获取当前所有可索引页面 URL。
2. 读取 `urls/{域名}.json`（如存在），获取该域名已发现列表，与本次扫描结果比对，找出新增 URL。
3. **智能筛选**新增 URL —— Agent 根据域名核心关键词自主判断优先级：
   - 高价值页面-> **标记为待铺设**
   - 常规页面-> **视情况标记**
   - 低价值页面-> **跳过**
4. 将新增 URL 写入 `urls/{域名}.json`，`platforms` 初始化为空数组 `[]`。
5. 更新 `monitored-domains.json`，将 `last_analyzed_at` 设为当前时间。
6. **动态调整间隔**：若本次扫描发现新增 URL 数 > 5，在 history 文件中备注"建议缩短 scan_interval_hours"（如 24 -> 12），供用户下次手动调整。

### Step 3：记忆归档

将扫描结果归档到 `.agent-memory/scan/` 目录。

---

## 状态持久化文件

- `monitored-domains.json` — 已监控域名的状态（`last_analyzed_at` 上次扫描时间，`scan_interval_hours` 扫描间隔小时数，默认 24）
- `urls/{域名}.json` — **按域名隔离**，存储该域名下所有 URL，`platforms` 字段为字符串数组，仅记录已成功铺设的平台名
- `tmp/` — **临时文件目录**。执行过程中产生的临时文件（缓存、调试日志、中间 HTML 等）一律存入此处。该目录已被 `.gitignore` 排除，不会被 git 追踪。

---

## 记忆系统

### 节点格式（插入 `.agent-memory/scan/index.md` 顶部）

```markdown
- **{YYYY-MM-DD HH:mm}** | [scan] | {目标域名} | 发现{n}个新URL，{m}个待铺设 | [{status}] | [详情](history/{文件名}.md)
  - 备注：{异常、数据质量观察、后续建议}
```

### 详细记录文件（`.agent-memory/scan/history/{日期}-{时分}.md`）

**包含内容**：

- 扫描概览（日期、目标域名、扫描范围）
- 执行步骤（爬取源、比对过程）
- 发现的新URL清单（URL、页面类型、优先级、状态）
- 跳过的URL及原因
- 备注

### 记忆流程

1. **任务启动时：读取历史记忆**
   - 读取 `.agent-memory/scan/index.md`，了解该域名过往的扫描历史。
   - 吸收历史经验指导本次执行：上次扫描遇到什么异常？上次筛选标准是否漏掉了高价值页面？上次的数据质量观察？
   - 按当前日期+时分生成文件名（如 `.agent-memory/scan/history/2026-05-14-1430.md`）。
2. **执行过程中**：实时写入扫描事件到详情文件。
3. **任务结束时**：
   - 完善详情文件
   - 在 `.agent-memory/scan/index.md` 顶部插入 `[scan]` 类型节点
