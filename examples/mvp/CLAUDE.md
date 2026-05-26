# DocFlow AI — 架构上下文

> 此文件放在代码仓库根目录。每个 Claude Code 会话开始前必读，以保持架构决策一致性。

## 项目概述

DocFlow AI：面向中小律所（3-50人）的 AI 合同审查工具。
核心价值：把人工 6-8 小时的合同风险条款识别压缩到 3 分钟以内。

**当前阶段**：MVP（2026-05）
**目标**：3 个非创始人朋友的真实用户连续 2 周自发回访

---

## 技术栈

| 层级 | 选择 | 原因 |
|------|------|------|
| Frontend | Next.js 14 (App Router) | 全栈一体，减少上下文切换；Vercel 部署零配置 |
| Styling | Tailwind CSS + shadcn/ui | 移动速度快，组件质量够用，不自定义设计系统 |
| Backend | Next.js Route Handlers | MVP 阶段不值得维护独立服务 |
| Database | PostgreSQL (Supabase) | Row Level Security 内置；免费额度够 MVP 用 |
| Auth | Supabase Auth | 与数据库同平台，减少集成复杂度 |
| File Storage | Supabase Storage | 同上，合同文件存这里 |
| AI | OpenAI API (GPT-4o) | 法律文本理解质量最稳；structured outputs 便于解析 |
| PDF Parsing | pdf-parse (Node.js) | 轻量；Word 文件用 mammoth |
| PDF Export | @react-pdf/renderer | 纯 JS，无需服务端二进制依赖 |
| Deployment | Vercel | 与 Next.js 原生集成，0 DevOps 成本 |

**刻意不用**：
- 不用 tRPC / GraphQL（REST 够用，减少学习成本）
- 不用独立 Python 后端（避免两套部署）
- 不用向量数据库（规则集固定，不需要语义检索）
- 不用 Redis（MVP 无并发压力）

---

## 目录结构

```
docflow-ai/
├── app/
│   ├── (auth)/           # 登录/注册页
│   ├── (dashboard)/      # 登录后主界面
│   │   ├── page.tsx      # 历史审查列表
│   │   └── review/
│   │       ├── new/      # 上传入口
│   │       └── [id]/     # 审查结果页
│   └── api/
│       ├── review/       # 触发 AI 审查
│       └── export/       # 生成 PDF 报告
├── components/
│   ├── ui/               # shadcn 基础组件（不修改）
│   ├── upload/           # 文件上传组件
│   └── review/           # 高亮标注、条款卡片
├── lib/
│   ├── ai/
│   │   ├── extract.ts    # OpenAI 调用 + prompt
│   │   └── schema.ts     # Zod schema for structured output
│   ├── parsers/
│   │   ├── pdf.ts        # pdf-parse 封装
│   │   └── docx.ts       # mammoth 封装
│   └── supabase/
│       ├── client.ts     # 浏览器端 client
│       └── server.ts     # 服务端 client
├── CLAUDE.md             # ← 本文件
└── supabase/
    └── migrations/       # 数据库 schema 版本
```

---

## 数据库 Schema（核心表）

```sql
-- 用户由 Supabase Auth 管理，无需手动建表

create table reviews (
  id          uuid primary key default gen_random_uuid(),
  user_id     uuid references auth.users not null,
  filename    text not null,
  file_url    text not null,           -- Supabase Storage URL
  status      text default 'pending',  -- pending | processing | done | error
  result      jsonb,                   -- AI 返回的结构化结果
  created_at  timestamptz default now()
);

-- RLS: 用户只能看自己的审查记录
alter table reviews enable row level security;
create policy "own reviews" on reviews
  for all using (auth.uid() = user_id);
```

`result` jsonb 结构：
```json
{
  "clauses": [
    {
      "type": "liability_cap",
      "risk_level": "high",
      "summary": "责任限制为合同金额的 10%，远低于行业惯例",
      "original_text": "...",
      "page": 3,
      "char_offset": 1240
    }
  ],
  "overall_risk": "high",
  "processing_time_ms": 8200
}
```

---

## AI 提取模式

**调用方式**：单次 OpenAI API 调用，使用 structured outputs（`response_format: { type: "json_schema" }`）

**提示词策略**：
- System prompt 固定（包含 10 类风险条款定义 + 中国法律语境）
- User prompt = 解析后的合同全文（按 token 预算截断，超长合同分段处理）
- 温度设为 0（确定性输出，避免幻觉）

**10 类标准风险条款**：
1. 责任限制条款
2. 自动续约条款
3. 管辖法院/仲裁条款
4. 保密义务范围
5. 知识产权归属
6. 违约金与罚则
7. 不可抗力定义
8. 单方变更权
9. 排他性条款
10. 终止权与后果

**Token 预算**：GPT-4o 128k context，单次处理上限 80k tokens（约 60k 汉字）。超出则分段，每段保留 500 字重叠。

---

## 规模约束（6个月内）

- **用户数**：< 500 注册用户，< 50 日活
- **并发**：Vercel serverless 自动扩缩，无需手动配置
- **存储**：Supabase 免费额度（500MB 数据库，1GB 文件存储）够用
- **OpenAI 费用**：按量计费，单次审查约 $0.10-0.30，月均成本可控
- **限速**：Route Handler 层加每用户每小时 10 次审查限制（防滥用，不做分布式限速）

超出以上约束时再考虑：独立服务器、自建队列、数据库分片。现在不提前优化。

---

## 有意接受的权衡

| 决策 | 代价 | 接受原因 |
|------|------|----------|
| 无消息队列，同步处理审查 | 超时风险（>30s 的大文件） | MVP 文件普遍 < 30页，超时率预计 < 5% |
| 不缓存 AI 结果 | 同一文件重复审查重复计费 | 极少发生，省去缓存复杂度 |
| 无自动化测试 | 回归风险 | 快速迭代优先，上线后补关键路径测试 |
| Supabase 托管数据库 | 供应商锁定 | 标准 PostgreSQL，迁移成本低 |
| 单一 OpenAI 提供商 | 故障单点 | 无业务关键需求要求高可用，出问题再加备用 |

---

## 必须遵循的模式

1. **服务端 AI 调用**：OpenAI API key 只在 Route Handler 里使用，绝不暴露到客户端
2. **RLS 优先**：所有数据访问依赖 Supabase RLS，Route Handler 不做手动 user_id 过滤作为主防线
3. **乐观 UI**：文件上传后立即跳转审查结果页（状态 pending），轮询更新（每 2 秒一次，最多 60 秒）
4. **错误边界**：AI 调用失败时保存 error 状态 + 原始错误信息，不向用户暴露 OpenAI 错误详情
5. **类型安全**：`result` jsonb 字段用 Zod schema 在读取时验证，不 cast any

## 禁止的模式

- 不在客户端组件里直接查询数据库
- 不把合同原文存入 PostgreSQL（存 Supabase Storage，DB 只存 URL）
- 不在没有 auth 检查的 Route Handler 里处理文件
- 不在 `result` 里存用户 PII 以外的数据
