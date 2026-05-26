# 构建会话日志

> 每次 Claude Code 构建会话结束后追加。最新会话在最上方。

---

## 会话 3 — 2026-05-25

**本次完成**：
- 实现 PDF 报告导出功能（`@react-pdf/renderer`）
- 报告包含：封面页（文件名 + 审查日期）、风险概览（高/中/低条款数量汇总）、逐条详情（条款原文 + 风险等级 + 说明）、免责声明页
- 修复了 D2 发现的 bug：合同文件名含中文时 Supabase Storage 上传路径编码错误（改用 `encodeURIComponent` + UUID 作为存储 key）
- 加了每用户每小时 10 次审查的限速（Redis-free 方案：直接查 PostgreSQL 过去 1 小时的 reviews 计数）

**关键决策**：
- PDF 导出纯客户端生成（不走服务端），避免 Vercel serverless 函数超时风险。代价是首次渲染略慢（~2s），可接受
- 报告不嵌入合同原文（避免文件过大），只嵌入被标注的条款片段

**引入的假设**：
- 用户对"客户端生成 PDF"的等待时间（2s）有耐心——需要真实用户验证
- 律所场景下导出格式（A4竖版）符合归档习惯——访谈中未明确确认过

**遗留 bug / 已知问题**：
- Word (.docx) 文件中表格内的条款有时提取不完整（mammoth 对复杂表格支持弱）。MVP 阶段先在界面加提示："建议提交 PDF 格式以获得最佳效果"
- 超过 60 页的合同（~80k tokens）会触发分段处理，当前实现没有对分段结果去重，可能出现同一条款重复标注

**下次继续**：
1. 邮件邀请流程：生成邀请链接 → 被邀请人注册后自动关联到同一律所 workspace（但 v1 不做多用户，先做成"邀请好友得额外配额"的增长钩子）
2. 准备第一批种子用户邀请：从访谈的 8 位律所助理中选 3 位，发送内测邀请

---

## 会话 2 — 2026-05-22

**本次完成**：
- 实现核心 AI 审查流程（`/api/review` Route Handler）
  - 接收 `review_id`，从 Supabase Storage 下载文件
  - 用 `pdf-parse` / `mammoth` 解析为纯文本
  - 调用 OpenAI GPT-4o（structured outputs + Zod schema 验证）
  - 将结果写入 `reviews.result` jsonb 字段，更新 status 为 `done`
- 实现审查结果页：高亮卡片列表（高/中/低三色标注）、点击展开原文片段
- 轮询机制：结果页每 2 秒 fetch 一次 review status，最多等 90 秒
- 写了 system prompt v1（10 类条款定义 + 中国法律语境说明）

**关键决策**：
- Structured outputs schema 采用扁平数组（`clauses[]`），不做嵌套分组——前端排序/过滤更灵活，后续可按需调整展示逻辑
- System prompt 不锁版本，直接存代码里。等发现 prompt 漂移问题再考虑版本管理
- 页面不做 WebSocket，轮询够用（MVP 阶段并发用户个位数）

**引入的假设**：
- GPT-4o 对中文合同法律条款的识别准确率满足用户预期（> 80%）——未经用户实际验证
- 30 页以下的合同处理时间 < 30 秒——本地测试通过，但 Vercel 冷启动未测试
- 用户能理解"高亮卡片"式的呈现方式，不需要在原始 PDF 上直接标注——产品假设，需要观察用户行为

**遇到的问题**：
- OpenAI structured outputs 在 response 较长时偶尔截断（max_tokens 设置问题），已改为动态计算：输入 token 数 × 0.3 作为 max_tokens 下限，并设硬上限 4096
- `pdf-parse` 对扫描件（图片 PDF）返回空字符串，需要明确告知用户"暂不支持扫描件，请上传文字版 PDF"——当前在上传页加了说明文字，未做自动检测

**下次继续**：
1. PDF 报告导出
2. 修 Supabase Storage 中文文件名 bug（上传成功但 URL 访问 403）

---

## 会话 1 — 2026-05-20

**本次完成**：
- 初始化 Next.js 14 项目（App Router + TypeScript + Tailwind + shadcn/ui）
- 配置 Supabase：创建项目、设置 RLS 策略、建 `reviews` 表（见 CLAUDE.md schema）
- 实现 Supabase Auth 登录/注册页（邮箱 + 密码，暂不做 OAuth）
- 实现文件上传页：拖拽上传 + 点击上传，支持 PDF/Word，前端做文件大小限制（< 20MB）
- 上传后文件写入 Supabase Storage，创建 `reviews` 记录（status: pending），跳转到结果页（显示"处理中"）
- 部署到 Vercel，配置环境变量（`SUPABASE_URL`、`SUPABASE_ANON_KEY`、`OPENAI_API_KEY`）

**关键决策**：
- Auth 用 Supabase 内置方案，不自己实现 JWT。代价：与 Supabase 强绑定。接受，因为迁移成本低（标准 PostgreSQL）
- 文件大小上限定为 20MB（约 300 页合同）。理由：律所日常合同普遍 < 50 页；超大文件处理时间超 Vercel 30s 超时限制
- 先不做邮箱验证强制（Supabase 默认要求），关掉以降低注册摩擦——上线前再打开

**引入的假设**：
- 目标用户（律所助理）能独立完成注册流程，无需引导——待观察
- Vercel + Supabase 免费额度在前 50 个用户阶段不会触发限制——需持续监控用量

**下次继续**：
1. 实现 `/api/review` Route Handler（AI 审查核心逻辑）
2. 实现审查结果展示页（高亮卡片 + 轮询）
