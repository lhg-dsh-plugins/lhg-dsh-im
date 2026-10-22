# BUGFIX — 飞书过程卡切分按错误的渲染形态量字节，成品卡可超字节预算

## 标题

`splitStepStreamCardBlocks` 用 running 渲染量字节预算，但实际发送的 finished 渲染（sealed / completed / stopped）会给工具/思考面板多包一层过程详情面板，体积反而更大；临界大小的块会被切成单块、却以超预算的形态发给飞书。

## 证据链（修复前）

基线：上游 `43f39a4a746c89048caee93447e7d5b82b597685`（v5.4.0）。与本次改动直接相关的 `feishu-cards.test.mjs` 基线全部通过；全量 `npm test` 在本沙箱环境下有一条与改动无关的既存失败：`feishu-proxy.test.mjs` 的「Feishu verification uses the SDK HTTP client through HTTPS_PROXY」在基线轮次即以同一断言失败（:63，`0 !== 1`），属沙箱代理环境不兼容。

离线复现（直接调用 `src/channels/feishu/feishu-cards.mjs` 的公开函数）：取 tools + notes 两个面板块加一条填充正文，使 running 渲染恰好落在预算内：

| 渲染形态 | 字节数（UTF-8） | 对照预算 `STEP_STREAM_CARD_MAX_BYTES = 24000` |
|---|---|---|
| running（切分器量测口径） | 23,998 | ✅ 切分器据此判定单块不切 |
| sealed（非末块实际发送形态） | 24,393 | ❌ 超预算 393 |
| completed / stopped（末块实际发送形态） | 24,436 | ❌ 超预算 436 |

对照：纯文本块三态等大（22,961 / 22,918 / 22,961），只有带面板的块才中招——而过程卡几乎总带 tools 面板。

## 根因

- `stepStreamCard()` 中，非 running 状态会把 tools/notes 面板收进一个 `processDetailsPanel` 折叠 wrapper（多出约 438 字节的序列化脚手架），且 completed/stopped 还保留状态页脚；running 渲染则把面板平铺、无 wrapper。
- 因此 finished 渲染 ≥ running 渲染，与切分器文档注释「budgeted by the encoded running-status card (the largest render)」的前提相反。
- `splitStepStreamCardBlocks` 用默认 status（running）量测候选块，于是临界块通过预算检查、被整块发出，最终以 sealed/completed 形态patch/发送时超过预算——正是该字节预算要防的飞书整卡拒写风险（同文件表格数预算的注释已记录过同类拒写导致卡片停留在旧内容、回复不出现）。

## 改动文件

- `src/channels/feishu/feishu-cards.mjs`：切分器的量测口径改为 `status: 'completed'` 渲染（成品形态中最大者：wrapper + 状态页脚），并修正文档注释说明理由。仅此一处逻辑改动，未动渲染本身。
- `lib/index.js`：`npm run build` 按上游流程同步再生的宿主 bundle（内含同一行改动）。
- `test/channels/feishu/feishu-cards.test.mjs`：
  - 原「measures the selected panel expansion at the byte boundary」测试把「以 running 渲染量边界」这一错误前提钉死，改为「measures the finished render at the byte boundary」：以 completed 渲染定边界，并断言所得块在 running / sealed / completed / stopped 四态下均不超限。
  - 新增回归测试「keeps finished renders within the byte budget」：复现上表场景，断言切分器必须切块、内容不丢、且每块四态渲染均 ≤ 24,000。

## 复验结果（修复后）

- 复现场景：切分器对同一组块由 1 块改为 2 块；回归测试断言每块四态渲染均在预算内。
- `test/channels/feishu/feishu-cards.test.mjs`：36/36 通过。
- 全量 `npm test`：失败项均为真实 SDK / 代理环境类测试——`feishu-operation.test.mjs` 的 silent-endpoint 时序测试（单独运行时修复前后均 3/3 通过，全量并行下偶发）与 `feishu-proxy.test.mjs` 的 HTTPS_PROXY 测试（基线轮次即失败，见上）。两者与本次改动（仅飞书卡片切分量测口径）无关。
- `npm run build` + `node scripts/verify-package.mjs`：通过（Verified dsh-im package artifacts）。
