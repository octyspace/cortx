# 联网搜索 MCP 自建调研报告 —— 复刻智谱 Web Search Pro

> 目标：在香港服务器上用开源组件搭建一套"搜索聚合 + 内容增强"服务，
> 通过 HTTP JSON API 和 MCP（streamable HTTP + Bearer）对外提供，
> 返回结构类似智谱 `web_search_pro` 的高质量搜索结果（含页面正文）。
> 调研日期：2026-08-20。响应结构对照依据：cortex `src/tools/net.ts` 中
> 智谱/Tavily/Exa/Firecrawl/Brave 五家引擎的真实集成代码。

---

## 0. 结论摘要

- **可复刻度约 80%**：智谱的护城河是搜狗/夸克等**国内搜索引擎合作索引** + 自建爬虫集群，
  这两项买不到；但"多引擎聚合 → 正文提取 → 结构化 JSON"的管道完全可以用
  **SearXNG + Trafilatura + FastMCP** 在一台 2C2G 香港服务器上复刻，年成本 ≈ 智谱 3 万次调用费。
- **架构四层**：搜索聚合（SearXNG）→ 排序去重（RRF + URL 归一化 + 加权）→
  内容增强（Trafilatura 三级提取链）→ 服务封装（FastAPI + FastMCP + Redis 缓存）。
- **两种模式**：fast（仅 snippet，1–2s，对标智谱 search_std）/ rich（正文增强，3–5s，对标 search_pro 与 Tavily）。
- **最大风险**：Google 对数据中心 IP 的 CAPTCHA 限流 —— 用多引擎矩阵 + 缓存 + QPS 控制缓解，单引擎挂掉自动降级。
- **与 cortex 集成双通道**：web_search 引擎链加 `searxng` 引擎（主 agent 低延迟直连）+ MCP_REGISTRY 注册（子 agent 工具化）。

---

## 1. 智谱 web_search_pro 拆解 —— "高质量"到底从哪来

智谱联网 MCP（`open.bigmodel.cn/api/mcp/web_search_pro/mcp`，SSE/streamable HTTP + Bearer）
背后是 HTTP API `POST /api/paas/v4/web_search`，参数 `search_engine` 可选
`search_std / search_pro / sogou / quark`。它的质量来自四件事：

1. **合作索引**：sogou/quark 档位直接接腾讯搜狗、夸克的国内索引 —— 中文覆盖率天然高于
   海外引擎。这是买不到的部分；替代方案是聚合 Google 中文（质量本就不错）+ Bing 国内版。
2. **多引擎聚合**：标准档内部也是多源聚合后统一排序。
3. **内容增强**：返回的 `search_result[].content` 不只是引擎 snippet，而是**抓取清洗后的正文摘录**
   —— 说明它有完整的"搜完即抓"管道。
4. **结构化输出**：`{title, link, content, media, type, icon}`，为 LLM 消费设计。

我们要复刻的就是第 2/3/4 点。对照 cortex 现有引擎链（zhipu→exa→firecrawl→brave→tavily→bing→ddg），
自建服务相当于把这条链搬到自己服务器上跑，并补上 Trafilatura 正文提取。

---

## 2. 总体架构：四层流水线

```
Agent（cortex / 任意 MCP 客户端 / 第三方用户）
   │   MCP streamable HTTP + Bearer（/mcp）      或   HTTP JSON（/v1/search）
   ▼
┌────────────────── Search Hub（HK 2C2G，Docker Compose）──────────────────┐
│                                                                          │
│  L4 服务层    FastAPI + FastMCP    Bearer 认证 · 限流 · Redis 缓存(24h)   │
│  L3 增强层    Trafilatura → readability-lxml → Firecrawl/Playwright      │
│               （并行抓 top-K，每页预算 8s，总预算 12s，失败降级到 snippet） │
│  L2 排序层    RRF 聚合 · URL 归一化去重 · 语言/时效/域名加权 · snippet 合并 │
│  L1 搜索层    SearXNG：google / bing / brave / duckduckgo / qwant 并行    │
│               （JSON API，单引擎失败自动降级，score 内置多源累加）          │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## 3. 逐层技术选型与理由

### 3.1 L1 搜索聚合层 —— SearXNG

- **为什么**：唯一真正可自部署的搜索层（元搜索，聚合多家上游引擎），MIT 之外的 AGPL 对自用无影响。
  Tavily/Brave/Exa/Jina 的"自部署 MCP"只是云 API 连接器，配额照算、国内不可用照旧。
- **部署**：Docker `searxng/searxng`，`settings.yml` 关键配置：
  ```yaml
  search:
    formats: [html, json]        # JSON 默认关闭，必须显式开启
    safe_search: 0
  server:
    limiter: false               # 自用关掉内置限流（外层自己做）
    public_instance: false
  ```
  引擎矩阵：`google, bing, brave, duckduckgo, qwant, mojeek, startpage`（中文可试加 sogou）。
- **JSON API**：`GET /search?q=...&format=json&language=zh-CN&time_range=week`，
  返回 `results[]{url, title, content, engine, engines, score, publishedDate}`、
  `answers[]`（计算/字典类直答）、`infoboxes[]`（知识卡片）。
  `score` 已是**多引擎归一化累加**，`engines[]` 标注结果来源 —— 与 cortex 的
  `triedEngines` 尝试链思路一致，可直接透出做可观测性。
- **与 Firecrawl 的关系**：Firecrawl 自部署后 `/v1/search` 的默认后端就是 SearXNG
  （`SEARCH_BACKEND=searxng` + `SEARXNG_URL`）。也就是说无论走哪条路，搜索底座都是 SearXNG；
  Firecrawl 的增量价值只在 JS 渲染抓取侧（Phase 4 再上）。

### 3.2 L2 排序去重层 —— 自研后处理（~200 行 Python）

SearXNG 的内置聚合偏"求和"，自建层做四件事把质量拉到商业水平：

1. **RRF（Reciprocal Rank Fusion）**：`score = Σ 1/(60 + rank_i)`。
   多引擎共识的结果上浮，单引擎孤例下沉 —— 比简单分数求和更抗单源噪声（Tavily 同思路）。
2. **URL 归一化去重**：剥 `utm_*` 参数、fragment、尾斜杠，统一 https，
   `m.example.com → www.example.com`；同一 URL 的多引擎 snippet 拼接合并（互为补充）。
3. **加权信号**（全部可配置）：
   - 语言匹配：中文 query 命中中文页面 boost（Google 中文结果本就强）；
   - 时效：`publishedDate` 7 天内 boost（`time_range` 直通 SearXNG）；
   - 域名质量表：官方文档/权威站加分，内容农场减分（自维护 JSON，起步可为空）；
   - 标题词覆盖率：query 关键词在 title 中的比例。
4. **配额**：取 top-N（默认 10），标注 `rank`。

### 3.3 L3 内容增强层 —— "json 页面内容"的核心

对 top-K（默认 3，可配 5/8）结果并行抓取正文，**三级提取链**：

| 级别 | 工具 | 覆盖场景 | 说明 |
| --- | --- | --- | --- |
| ① 主提取 | **Trafilatura** | 静态页 80–90% | 正文+标题+日期+作者，官方基准 F1 90+，优于 readability-lxml；输出 markdown |
| ② 兜底 | readability-lxml | ① 失败/为空 | 更快但更噪，作为 fallback |
| ③ 渲染 | Firecrawl self-host（Playwright） | JS 空壳页（SPA） | 触发条件：原始 HTML < 2KB 或 body 内无有效文本 |

- **预算控制**：每页连接 5s / 读取 8s，整批总预算 12s，`asyncio.gather` 并行；
  任一页失败**不阻塞**——snippet 永远在，正文是增量。
- **输出规整**：markdown，单页截断 2000–4000 字（对齐 cortex `max_result_chars` 哲学），
  保留代码块与表格；多页内容做跨页去重（同源转载文只保留一份）。
- **现成轮子**：cortex `web_fetch` 已有 Firecurl→Jina→raw HTML 回退链和
  `extractPageMetadata`，服务端照搬这套模式即可，逻辑甚至可以直接从 `python/cortex_agent/tools.py` 移植。

### 3.4 L4 服务封装层 —— FastAPI + FastMCP + Redis

- **HTTP API**（FastAPI）：`GET /v1/search`，参数 `q, count, enrich, lang, time_range`。
- **MCP 端点**（FastMCP，streamable HTTP）：挂在 `/mcp`，**对齐智谱 MCP 的接入形态**
  （URL + Bearer），cortex 的 `MCP_REGISTRY` 直接按 URL 条目注册，`mcp_session_start` 即用。
- **认证/防护**：Bearer token（多 token 支持做多用户配额）；非标端口 + 防火墙白名单兜底；
  每 token QPS 限流。SearXNG 的 JSON API 绝不直接裸奔公网。
- **缓存**：Redis，query 归一化（去空白/小写/去停用词 hash）为 key，24h TTL ——
  agent 高频重复查询零成本，同时显著降低上游引擎 QPS（缓解 4.1 风险）。

---

## 4. API 响应 Schema 设计

对齐智谱 `search_result[]` 结构（我们已集成，零适配成本），并借鉴 Tavily 的
`score / raw_content / answer` 字段：

```json
{
  "search_result": [
    {
      "title": "React 20 发布公告",
      "link": "https://react.dev/blog/2026/08/react-20",
      "content": "## 正文 markdown（enrich>0 时为 Trafilatura 提取全文，否则=snippet）",
      "snippet": "引擎聚合摘要，多引擎 snippet 合并去重",
      "rank": 1,
      "engines": ["google", "bing"],
      "score": 0.83,
      "published": "2026-08-15",
      "media": "react.dev"
    }
  ],
  "answer": null,
  "usage": { "latency_ms": 2400, "engines_used": ["google","bing","brave","ddg"], "enriched": 3, "cache_hit": false }
}
```

- `content` 字段语义与智谱一致：开 enrich 时是正文，不开时是 snippet —— LLM 端解析逻辑完全复用。
- `answer` 预留（Phase 4：调 LLM 对 top 结果生成摘要直答，Tavily 式；会增加延迟与 token 成本）。

**MCP 工具集**（一个服务替代 mcp-searxng + fetch 两个）：

| 工具 | 参数 | 返回 |
| --- | --- | --- |
| `web_search` | query, count?, enrich?, lang?, time_range? | 上面的 JSON |
| `fetch` | url, max_chars? | Trafilatura 提取的 markdown（单页正文） |

---

## 5. 延迟与资源预算

| 阶段 | 耗时 | 说明 |
| --- | --- | --- |
| SearXNG 多引擎聚合 | 0.8 – 2.0s | 上游引擎并行，取最慢者 |
| L2 排序 | < 10ms | 纯内存计算 |
| L3 并行提取 top-3 | +1 – 4s | 命中缓存为 0ms |
| **fast 模式合计** | **1 – 2s** | ≈ 智谱 search_std（<1s，纯 snippet） |
| **rich 模式合计** | **3 – 5s** | ≈ Tavily / 智谱 search_pro 实测水平 |

资源：SearXNG + API + Redis 在 **2C2G** 轻量服务器（HK，活动价约 300 元/年级别）富余；
上 Firecrawl（Playwright 无头浏览器池）建议 **2C4G**。带宽：搜索 JSON 极小，
enrich 每页平均约 500KB，30Mbps 轻量套餐够用。缓存命中率 30%+ 时，实际平均延迟显著低于上表。

---

## 6. 风险与缓解

| 风险 | 概率 | 影响 | 缓解 |
| --- | --- | --- | --- |
| **Google 封数据中心 IP**（CAPTCHA/限流，自建 SearXNG 最大风险） | 高（高频时） | google 引擎质量下降 | ① 7 引擎矩阵，SearXNG 单引擎失败自动降级不阻塞；② Redis 缓存 + 每 token QPS 限制；③ 实测劣化则调低 google 权重；④ 极端情况出站走住宅代理 |
| 目标站反爬（Cloudflare 等） | 中 | 部分页正文提取失败 | 三级提取链兜底；失败不阻塞（snippet 仍在）；Firecrawl 第三级 |
| 提取失败率 | 10–20% | 个别结果无正文 | readability 兜底 + snippet 兜底 + `enriched` 计数透出 |
| 公网滥用 JSON API | 中 | 带宽/QPS 被打爆 | Bearer + 限流 + 非标端口 + 防火墙白名单 |
| AGPL 协议 | — | — | 自用/内部服务无义务；若对外提供服务需开源该层（L2–L4 是自研 MIT 无碍，仅 SearXNG 容器本身） |
| 智谱式国内索引缺失 | — | 极中文长尾 query 覆盖不如 sogou/quark 档 | Google 中文 + Bing 国内版已覆盖 95% 场景；保留 cortex 引擎链的 zhipu 作为付费备选 |

---

## 7. 分阶段实施计划

| 阶段 | 工作量 | 内容 |
| --- | --- | --- |
| **Phase 1** | 半天 | HK 服务器 Docker Compose 起 SearXNG（json 开启、7 引擎矩阵、limiter 配置）；curl 验证 `/search?format=json` 质量 |
| **Phase 2** | 1 天 | FastAPI 服务：L2 RRF 排序去重 + L3 Trafilatura 三级提取链 + Redis 缓存 + Bearer/限流 → `/v1/search` 上线 |
| **Phase 3** | 半天 | FastMCP 挂 `/mcp`（web_search + fetch 两工具）；**cortex 双端集成**：`web_search` 引擎链加 `searxng` 引擎（settings 增 `searxng_url`，排 zhipu 之后）+ `MCP_REGISTRY` 注册 URL 条目 + 配置向导加自建选项（TS+Python 同步） |
| **Phase 4**（可选） | 按需 | Firecrawl self-host（JS 渲染档）；`answer` LLM 摘要；查询改写；多 token 配额面板 |

---

## 8. 与商用方案对比

| 维度 | 智谱联网搜索 | Tavily | **自建 Search Hub** |
| --- | --- | --- | --- |
| 成本 | 0.01–0.03 元/次 | 1000 次/月免费后 $30–70/月 | ~300 元/年 服务器，**无限次** |
| 中文搜索质量 | 高（国内索引合作） | 中 | 高（Google 中文 + Bing） |
| 页面正文 | ✅ | ✅ | ✅ Trafilatura |
| 延迟 | std <1s / pro 2–4s | 2–4s | fast 1–2s / rich 3–5s |
| 国内直连 | ✅ | ❌ | ✅（HK 直连） |
| 可控性/可观测 | ❌ | ❌ | ✅（引擎来源/延迟/缓存全透出） |
| 数据隐私 | 出境到智谱 | 出境 | 自持 |
| 运维成本 | 零 | 零 | 低（Docker Compose 单机） |

**适用结论**：cortex 主力搜索走自建（成本趋零 + 可控），智谱保留为付费高质量备选档（引擎链回退语义现成），Tavily/Exa 降级为海外质量对照组。

---

## 参考来源

- Firecrawl 自部署文档（SEARCH_BACKEND=searxng / SEARXNG_URL）：https://docs.firecrawl.dev/contributing/self-host
- SearXNG 官方文档（JSON format、引擎配置）：https://docs.searxng.org
- Trafilatura（基准与用法，F1 对比 readability-lxml）：https://github.com/adbar/trafilatura 、https://trafilatura.readthedocs.io
- mcp-searxng（社区 MCP 包装参考）：https://github.com/ihor-sokoliuk/mcp-searxng
- 响应结构对照：cortex `src/tools/net.ts`（智谱/Tavily/Exa/Firecrawl/Brave 引擎解析代码）
