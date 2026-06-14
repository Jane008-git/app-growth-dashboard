# APP用户增长与A/B实验分析平台

本项目是一个面向数据分析师作品集的 **用户增长与A/B实验分析Demo**，覆盖完整的分析链路：埋点设计 → 数据建表 → SQL分析 → 可视化看板。支持上传自定义CSV数据，并基于SQLite引擎执行留存分析、漏斗分析、渠道质量评估、A/B实验显著性检验等。

---

## 📁 项目结构

```
app-growth-dashboard/
├── index.html          # 交互式分析看板（支持上传CSV + SQL查询）
├── data/               # 示例数据文件夹（可选的演示数据）
└── README.md           # 项目说明与数据建表规范
```

> 实际分析时，只需要通过页面上的上传按钮加载符合格式的CSV文件即可，无需手动配置数据库。

---

## 🗃️ 数据表结构规范

平台支持 **5张核心表**（表名必须严格一致，字段名建议保持一致）。若字段名不同，可在SQL编辑器中手动调整查询语句。

### 1. users 用户表
> 每行一个用户，描述用户基础信息与安装渠道。

| 字段名 | 类型 | 说明 | 示例 |
|--------|------|------|------|
| user_id | STRING | 用户唯一标识 | U000001 |
| install_date | DATE | 安装日期（YYYY-MM-DD） | 2026-05-01 |
| channel | STRING | 安装渠道 | tiktok_ads |
| country | STRING | 国家代码 | US |
| device_os | STRING | 操作系统 | iOS / Android |
| app_version | STRING | 安装时的APP版本 | 1.2.0 |
| age_group | STRING | 年龄段 | 25-34 |
| gender | STRING | 性别 | F / M |

**数据分布建议**（造数参考）：
- 用户总数：20,000 - 50,000
- 时间范围：最近30天
- 渠道分布：organic(25%), tiktok_ads(30%), google_ads(20%), facebook_ads(15%), referral(10%)
- 国家分布：US(30%), CN(35%), BR(20%), JP(10%), others(5%)

---

### 2. events 用户行为事件表
> 记录用户所有行为事件，是最大的明细表。

| 字段名 | 类型 | 说明 | 示例 |
|--------|------|------|------|
| event_id | STRING | 事件唯一ID（可选） | evt_001 |
| user_id | STRING | 用户ID | U000001 |
| event_time | DATETIME | 事件发生时间 | 2026-05-01 10:30:00 |
| event_date | DATE | 事件日期（分区） | 2026-05-01 |
| event_name | STRING | 事件名称 | finish_onboarding |
| session_id | STRING | 会话ID（可选） | sess_123 |
| channel | STRING | 当时渠道 | tiktok_ads |
| country | STRING | 国家 | US |
| device_os | STRING | 系统 | iOS |
| app_version | STRING | APP版本 | 1.2.0 |
| event_value | FLOAT | 事件数值（如时长/金额） | 120.5 |

**核心事件名称（用于漏斗/留存分析）**：

| 事件名 | 含义 | 分析用途 |
|--------|------|----------|
| app_open | 打开APP | 留存计算 |
| register | 完成注册 | 漏斗第一转化 |
| start_onboarding | 开始新手引导 | 漏斗步骤 |
| finish_onboarding | 完成新手引导 | **关键节点**（影响留存） |
| view_home | 查看首页 | 参与度 |
| click_core_feature | 点击核心功能 | 激活指标 |
| purchase | 购买行为 | 付费转化 |
| logout | 退出登录 | 可选 |

**数据分布建议**：
- 事件总数：500,000 - 2,000,000
- 每个用户平均事件数：20 - 40条
- 新手引导完成率：约55%～65%（对照组与实验组存在差异）
- 业务规律：完成 `finish_onboarding` 的用户 D1 留存显著更高（+15pp～+25pp）

---

### 3. channels 渠道投放表
> 记录每日各渠道的广告投放成本与曝光数据。

| 字段名 | 类型 | 说明 | 示例 |
|--------|------|------|------|
| date | DATE | 日期 | 2026-05-01 |
| channel | STRING | 渠道名 | tiktok_ads |
| country | STRING | 国家 | US |
| impressions | INT | 曝光次数 | 120000 |
| clicks | INT | 点击次数 | 5400 |
| installs | INT | 安装数 | 1200 |
| cost | FLOAT | 广告消耗（美元） | 3450.0 |

**建议渠道列表**：
```
organic, app_store, google_ads, tiktok_ads, facebook_ads, referral
```
> `organic` 和 `referral` 可以没有广告成本或成本为0。

**数据分布**：
- 数据量：约180行（6渠道 × 30天）
- CTR（点击率）：1%～6%
- CPI（单次安装成本）：$0.8～$5，因渠道而异

---

### 4. ab_tests 实验分组表
> 记录用户进入哪个A/B实验及分组。

| 字段名 | 类型 | 说明 | 示例 |
|--------|------|------|------|
| experiment_id | STRING | 实验ID | exp_onboarding_v2 |
| user_id | STRING | 用户ID | U010203 |
| group_name | STRING | 分组：control / treatment | treatment |
| assign_time | DATETIME | 分配时间 | 2026-05-01 09:00:00 |
| experiment_name | STRING | 实验名称 | 新手引导优化实验 |
| is_exposed | INT | 是否实际曝光（1/0） | 1 |

**实验设计建议**：
- **对照组（control）**：旧版新手引导（步骤多、无奖励）
- **实验组（treatment）**：新版引导（步骤简化 + 核心功能前置 + 首日奖励）

**数据分布**：
- 实验用户总数：8,000 - 20,000 人（每组约对半）
- 期望效果：treatment组 `finish_onboarding` 完成率 **提高 4%～8%**，D1留存 **提高 2%～4%**

---

### 5. payments 付费表（可选）
> 记录用户付费订单，用于LTV与ROI分析。

| 字段名 | 类型 | 说明 | 示例 |
|--------|------|------|------|
| payment_id | STRING | 订单ID | pay_20260501_001 |
| user_id | STRING | 用户ID | U000888 |
| payment_time | DATETIME | 支付时间 | 2026-05-15 14:20:00 |
| payment_date | DATE | 支付日期 | 2026-05-15 |
| amount | FLOAT | 金额（美元） | 4.99 |
| currency | STRING | 币种 | USD |
| product_type | STRING | 商品类型 | starter_pack |
| channel | STRING | 用户渠道 | google_ads |
| country | STRING | 国家 | US |

**数据分布**：
- 订单数量：1,000 - 8,000（约2%～5%付费率）
- ARPU（每用户平均收入）：$1.0～$3.0
- LTV随渠道差异明显：Referral / Organic > 买量渠道

---

## 🧪 数据中的业务故事（造数逻辑建议）

为了让分析结论真实可信，造数时应体现以下业务规律：

1. **完成新手引导 vs 未完成**  
   - `finish_onboarding = YES` 的用户 D1 留存 ≈ 40%+  
   - `finish_onboarding = NO` 的用户 D1 留存 ≈ 18% 以下  

2. **A/B实验效果**  
   - treatment组 `finish_onboarding` 完成率明显高于 control 组  
   - treatment组 D1/D7 留存均高于 control 组（提升幅度真实但不夸张）

3. **渠道质量差异**  
   - tiktok_ads：新增量大，但留存低、ROI＜1  
   - referral / organic：新增较少，但留存高、LTV高  

4. **版本迭代影响**  
   - 新版APP（v1.3.0）引导完成率与留存均优于旧版（v1.2.0）

5. **国家/地区差异**  
   - US / JP 付费高、留存好  
   - BR 新增多但付费低  
   - CN 留存中等、付费适中  

---

## 📊 推荐数据规模（作品集友好）

| 表名 | 建议行数 | 说明 |
|------|----------|------|
| users | 30,000 | 30天新增用户 |
| events | 800,000 | 每用户约25～30条事件 |
| channels | 180 | 6渠道 × 30天 |
| ab_tests | 12,000 | 参与实验用户 |
| payments | 3,000 | 付费订单 |

> ⚠️ 前端HTML上传限制单个文件不超过 **25MB**，若事件表超过限制，请压缩至15万行左右（约15～20MB）。

---

## 🚀 使用方式

1. 打开部署后的网页：`https://jane008-git.github.io/app-growth-dashboard/`
2. 依次上传符合上述格式的5张CSV表
3. 点击 **加载所有CSV到内存数据库**
4. 使用内置SQL模板或编写自定义SQL进行分析
5. 查看留存、漏斗、渠道、A/B实验等自动生成的可视化结论

---

## 📝 自定义SQL示例

```sql
-- Cohort 留存分析
SELECT install_date,
       COUNT(DISTINCT user_id) as new_users,
       COUNT(DISTINCT CASE WHEN day_diff = 1 THEN user_id END) * 1.0 / COUNT(DISTINCT user_id) as D1_retention
FROM (
    SELECT u.user_id, u.install_date, julianday(e.event_time) - julianday(u.install_date) as day_diff
    FROM users u JOIN events e ON u.user_id = e.user_id
    WHERE e.event_name = 'app_open'
) GROUP BY install_date;
```

---

## 📄 License

本项目仅供数据分析学习与作品集展示使用。

**作者**：Jane008-git  
**项目地址**：https://github.com/jane008-git/app-growth-dashboard
```# app-growth-dashboard
