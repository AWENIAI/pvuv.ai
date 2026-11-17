# pvuv.ai 自部署记录（aweniai.com）

部署日期：2026-11-12　·　部署方式：Cloudflare Workers + D1 + Queues + KV
仓库路径：`/Volumes/DaHua_SSD/部署项目/codex/其他/pvuv.ai`

## 一、域名与入口

| 用途 | 域名 | Worker |
|---|---|---|
| 控制台（后台） | `https://pvuv.aweniai.com` | `pvuv-console` |
| 事件上报（埋点） | `https://in.aweniai.com` | `pvuv-ingest` |
| 查询 API | `https://api.aweniai.com` | `pvuv-api` |

三个域名均由 wrangler 的 `custom_domain = true` 自动创建 DNS 记录与 TLS 证书，
**不占用主域名 `aweniai.com`**（主域名仍按原规则跳转 `jevailab.com`）。

> 注意：`aweniai.com` 上存在一条既有的全站 301 → `jevailab.com` 重定向规则，
> 其优先级高于 Worker 自定义域名。已改为**排除**上述三个子域名，其余流量不变。
> 以后再新增子域名时，需同步在该重定向规则的表达式里排除新子域名。

## 二、Cloudflare 资源清单

账号：`Q371317428@gmail.com's Account`　·　account_id `0a9b110b9b2a824a62004210f7d21a2b`
Zone：`aweniai.com`　·　zone_id `176cd38d30d99c45fe349b9573b6dd75`

| 类型 | 名称 | ID |
|---|---|---|
| D1 | `pvuv-db` | `062e5d96-f650-44bb-ac94-fe42f71a940c` |
| KV | `BLOCKLIST` | `9a3c7fb92e3d45028b18b1ffc58725e9` |
| KV | `SITE_CONFIG` | `aee7286c18d14d3bb751f1a801191429` |
| Queue | `pvuv-ingest` | `9c9d4e706c7b46b486a038f226a1246a` |
| Queue | `pvuv-ingest-dlq` | `c2fc042411ef46b490d1d131c80371b5` |

命名一律带 `pvuv-` 前缀，与账号内既有项目
（`ai-image`、`ssabenefitcalculator`、`jevailab-*` 等）无冲突，未改动任何既有资源。

## 三、Worker 清单

| Worker | 路由 | 备注 |
|---|---|---|
| `pvuv-ingest` | `in.aweniai.com` | 事件上报 + 广告判定 `/v` |
| `pvuv-consumer` | （无域名） | 消费队列，批量写入 D1 |
| `pvuv-api` | `api.aweniai.com` | 只读查询 API |
| `pvuv-console` | `pvuv.aweniai.com` | 控制台 + OAuth 登录 |
| `pvuv-cron` | （无域名） | 定时任务：`5 * * * *` / `30 3 * * *` |

`workers/proxy/` 是可选的第一方反代模板，**未部署**，仍为占位符配置。

## 四、密钥（均在 Workers Secrets，不落盘）

| Worker | Secret | 说明 |
|---|---|---|
| ingest / api / console | `HMAC_KEY` | 会话签名与判定 Cookie 签名，三者共用同一把 |
| console | `GOOGLE_CLIENT_SECRET` | Google 登录 |

`GOOGLE_CLIENT_ID` 属于公开值，写在 `workers/console/wrangler.toml` 的 `[vars]`。

轮换密钥（三个 worker 要用**同一把**新密钥）：

```bash
KEY=$(python3 -c "import secrets,base64;print(base64.b64encode(secrets.token_bytes(32)).decode())")
for w in ingest api console; do
  printf '%s\n' "$KEY" | npx wrangler secret put HMAC_KEY -c workers/$w/wrangler.toml
done
```

## 五、登录与权限（个人独占）

- 仅 `q371317428@gmail.com` 可登录；控制台为单租户，无开放注册入口。
- 双层防护：
  1. `ADMIN_EMAILS = "q371317428@gmail.com"`（`workers/console/wrangler.toml` 的 `[vars]`）
  2. `workers/console/src/index.ts` 的 `isAdminEmail()` 内置硬锁，
     即使 `ADMIN_EMAILS` 被改成别的邮箱，非 owner 账号一律被拒。
- OAuth 回调地址（需在 Google Cloud 侧登记）：
  `https://pvuv.aweniai.com/api/auth/google/callback`

## 六、本地改动（相对上游）

1. `workers/console/src/index.ts`
   - `isAdminEmail()`：增加 owner 硬锁。
   - 新增 `DELETE /api/sites/:id`：删除站点并清理其全部数据
     （sites / sessions / identities / visitor_profiles / 各 rollup /
     cluster_flags / anomaly_reports / ai_reports / funnels / 各月 events_YYYYMM 分区），
     `api_tokens` 中限定该站的令牌改为不限站点而非删除，KV 缓存 `site:<id>` 一并清除。
2. `workers/console/public/site.html`
   - 工具栏新增「删除站点」按钮（`.btn.danger`）+ 输入站点名二次确认的弹窗。
3. `workers/console/public/i18n.js`
   - 补充 zh 词条（其余语言回落英文）。

## 七、日常运维命令

```bash
cd "/Volumes/DaHua_SSD/部署项目/codex/其他/pvuv.ai"

# 重新部署某个 worker
npm run deploy:console
npm run deploy:ingest

# 查看日志
npx wrangler tail -c workers/console/wrangler.toml

# 数据库：查看表 / 执行 SQL
npx wrangler d1 execute pvuv-db --remote --command "SELECT ...;"

# 应用迁移（新增迁移文件后）
npx wrangler d1 migrations apply pvuv-db --remote

# 重新构建埋点脚本（改了 sdk/ 后）
npm run build:sdk && cp sdk/dist/f.js workers/console/public/f.js && npm run deploy:console
```

> `npm run setup` 在本机不可用：它的 `findD1Id()` 解析 `wrangler d1 list --json` 失败，
> 会在第 2 步误报「找不到 D1 id」。资源已手工创建并回填配置，重跑 setup 反而会中断。
> 若要用 setup，请先修 `scripts/setup.mjs` 的 JSON 解析，或直接按上面的运维命令分步执行。

## 八、回滚与卸载

**回滚代码**（回到上一个版本）：

```bash
npx wrangler rollback -c workers/console/wrangler.toml
```

**完全卸载**（删 Worker 会一并删除自定义域名；D1/KV/Queue 需单独删）：

```bash
for w in pvuv-ingest pvuv-consumer pvuv-api pvuv-console pvuv-cron; do
  npx wrangler delete "$w"
done
npx wrangler d1 delete pvuv-db            # 需输入确认
npx wrangler kv namespace delete BLOCKLIST
npx wrangler kv namespace delete SITE_CONFIG
npx wrangler queues delete pvuv-ingest
npx wrangler queues delete pvuv-ingest-dlq
```

卸载后 `pvuv/in/api.aweniai.com` 三条 DNS 记录会随自定义域名一并移除；
主域名与其它项目的跳转规则不受影响。
