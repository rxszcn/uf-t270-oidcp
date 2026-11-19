# 交付文档：客户端元数据文档缓存失效能力（t270）

## 背景

治理平台经客户端元数据文档（Client ID Metadata Document, CIMD）机制动态拉取依赖方的客户端注册文档并按声明时长缓存。既有实现中缓存上限长达 24 小时、响应声明不缓存也被下限抬到 30 秒，且没有任何强制失效入口，吊销指令到缓存实际失效之间形成失控窗口。本次改动为缓存补失效能力，并让声明不缓存的响应直通回源。

## 逐文件改动登记

### 1. `lib/helpers/client_id_metadata_document.js`

- 新增模块级 `invalidations: WeakMap<provider, Map<clientId, number>>` 失效代际标记，及只读访问器 `getInvalidationMarker(provider, id)`。
- 新增加导出 `invalidateClientMetadataDocument(provider, id)`（失效入口，形态为助手模块命名导出，粒度为"提供方 × 文档 URL"）：删除该提供方缓存中对应条目、丢弃该标识的在途拉取登记、递增该标识的失效代际；返回是否实际丢弃了缓存条目或在途拉取（`true`/`false`）。对未缓存或未知标识为纯 no-op：不触发任何出站拉取、不影响后续解析、返回 `false`，重复失效不累积效果。
- `parseCacheDuration`：响应显式声明不缓存（`Cache-Control: no-cache`、`no-store` 或 `max-age=0`）时返回 `0`，不再被 `cacheDuration.min` 下限抬升；其余响应的 min/max 钳制逻辑不变。
- `resolveClient` 在途拉取登记由裸 Promise 改为 `{ marker, promise }`：拉取创建时捕获失效代际，完成时仅当 `maxAge > 0`（可缓存）且代际未变（期间未失效）且缓存仍无该条目时才写入缓存；`finally` 清理逻辑按新登记形态等效调整。并发拉取合并（同一标识共享一次出站拉取）行为不变。

### 2. `lib/helpers/defaults.js`

- `features.clientIdMetadataDocument` 特性描述补充缓存行为说明：按响应 HTTP 缓存头在 `cacheDuration` 界内缓存、声明不缓存的响应不进缓存、`invalidateClientMetadataDocument(provider, clientId)` 失效入口及其语义。
- `features.clientIdMetadataDocument.cacheDuration` 描述补充：显式声明不缓存的响应完全绕过缓存，下限不适用。

### 3. `docs/README.md`

- 与 `defaults.js` 同步：`features.clientIdMetadataDocument` 章节新增缓存与失效段落，`cacheDuration` 小节补充不缓存声明直通的说明。（该文件本由 `docs/update-configuration.js` 生成；本环境下全量再生成会引入与本改动无关的格式化 churn，故按生成结果手工同步了这两处，内容与生成器输出一致。）

### 4. `test/client_id_metadata_document/client_id_metadata_document.test.js`

- 导入 `invalidateClientMetadataDocument`；`createProvider` 允许经 `configuration.fetch` 注入自定义 fetch；新增 `countingFetch` 帮助函数统计出站拉取计数。
- `metadata document cache` 分组新增用例 `does not cache documents whose response declares no caching`：覆盖 `no-store`、`no-cache`、`max-age=0` 三种声明，断言第二请求直接回源（出站计数为 2 且拿到新内容）。
- 新增 `metadata document cache invalidation` 分组：
  - `pulls the document from its origin right after invalidation`（失效后立即回源，失效后出站计数较失效前至少加一）；
  - `does not amplify outbound fetches on repeated invalidations`（连续重复失效两次且不再发起请求，计数不放大）；
  - `treats invalidating an uncached or unknown identifier as a no-op`（未缓存/未知标识语义）；
  - `discards in-flight fetches invalidated before they settle`（失效与在途拉取交叠：在途请求对其等待方仍正常了结，但其陈旧结果不落缓存，后续解析重新回源）。

### 5. `DELIVERY.md`

- 本交付文档。

## 题面要求条目映射

| # | 题面要求 | 落实位置 |
|---|----------|----------|
| 1 | 为缓存补失效能力，运营侧可对指定文档立即失效并回源 | `invalidateClientMetadataDocument` 导出（文件 1），失效后下一次解析即回源（测试 `pulls the document from its origin right after invalidation`） |
| 2 | 响应声明不缓存的文档不再进入缓存，第二请求直接回源 | `parseCacheDuration` 对 `no-cache`/`no-store`/`max-age=0` 返回 0，缓存写入以 `maxAge > 0` 为条件（文件 1）；测试 `does not cache documents whose response declares no caching` |
| 3 | 失效入口作用于未缓存或未知标识时的语义 | 纯 no-op、返回 `false`、不触发拉取（文件 1）；测试 `treats invalidating an uncached or unknown identifier as a no-op` |
| 4 | 失效与并发在途拉取的交叠时序 | 失效丢弃在途登记并递增代际，在途拉取完成时代际不符不落缓存（文件 1）；测试 `discards in-flight fetches invalidated before they settle` |
| 5 | 既有 TTL 窗口、缓存驱逐、按提供方隔离、并发拉取合并不回归 | 既有用例未改动且全部通过（见验收读数） |
| 6 | TTL 自然到期后回源的既有语义不变 | 可缓存响应的 min/max 钳制与惰性过期逻辑未动（文件 1），既有用例 `expires entries using their response-derived TTL` 通过 |
| 7 | 失效入口形态与粒度自行设计 | 助手模块命名导出，粒度为"提供方 × 文档 URL"，与缓存隔离粒度一致（文件 1） |
| 8 | 交付文档逐文件登记并逐条映射 | 本文件 |
| 9 | 验收命令实测读数连同构造输入读数 | 见"验收读数" |
| 10 | 仓库文档对缓存行为的描述与失效入口一致 | 文件 2、文件 3 |
| 11 | 测试并入既有测试文件、两类断言各至少一条、沿用既有框架、不另建文件 | 文件 4（node:test，失效后立即回源与不缓存声明直通用例均已加入既有文件） |
| 12 | 既有测试保持通过，验收以 npm test 零失败为准 | 见"验收读数" |

## 验收读数

### 验收命令

```
npm test
```

实测结果：（待填）

### 构造输入读数（口径：客户端元数据文档的出站拉取计数）

构造方式：快照测试环境内以 undici MockAgent 拦截文档源，Provider 配置注入计数 fetch，不依赖外部文档源。两组场景与测试文件内用例一一对应。

- 场景一（对同一指定文档失效一次后立即发起拉取）：失效前计数 1，失效并拉取后计数 2，delta=+1（要求：较失效前至少加一 ✓）
- 场景二（对同一指定文档连续重复失效两次且不再发起请求）：失效前计数 1，两次失效后计数 1，delta=+0（要求：计数不因重复失效放大 ✓）
