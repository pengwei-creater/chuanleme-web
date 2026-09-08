# 串了么 · 电竞选手梗百科

> 查电竞梗，上串了么，新手也能随便串。

垂直电竞领域选手梗检索工具，服务网络赛事观众，实现一键查梗、快速学串、一键用串。首期覆盖英雄联盟（LOL）、CS2、无畏契约（VALORANT）三大电竞项目。

## 技术栈

- **Next.js**（App Router + `src/` 目录，SSG 静态生成为主）
- **TypeScript** + **Tailwind CSS**（电竞赛博极简深色主题）
- **MDX 管线**：`/data` 下 JSON + MDX 静态数据，Git 版本管理
- **Zod**：构建期校验 `players.json` 与 MDX frontmatter，格式错误直接构建失败
- **Fuse.js**：构建脚本生成按游戏分片的 `search-index-*.json`，前端按需懒加载
- **Supabase JS SDK v2**：仅用于匿名投稿（`submissions`）与互动聚合（`interactions`）
- **Vercel**：Git push 自动构建部署

## 目录结构

```
chuanleme-web/
├── data/                      # 核心静态数据（选手 + 梗，Git 托管）
│   ├── lol/{players.json, memes/*.mdx}
│   ├── cs2/{players.json, memes/*.mdx}
│   └── valorant/{players.json, memes/*.mdx}
├── public/                    # 搜索索引（构建脚本生成）
├── scripts/build-search-index.ts
├── src/
│   ├── app/                   # 页面 + API 路由
│   │   ├── page.tsx           # 首页（搜索 + 热门选手 + 今日热梗）
│   │   ├── [game]/page.tsx    # 游戏分类页
│   │   ├── [game]/player/[id]/page.tsx  # 选手详情页
│   │   ├── submit/page.tsx    # 投稿页
│   │   ├── api/submit/route.ts# 投稿接口
│   │   ├── not-found.tsx      # 404（随机热梗入口）
│   │   └── sitemap.ts
│   ├── components/            # 通用组件
│   ├── lib/                   # supabase / zod-schema / data / search-build / games / typo-map
│   └── types/                 # 全局类型
├── .env.local.example         # 环境变量模板（提交 Git）
├── .env.local                 # 本地环境变量（已 gitignore）
└── next.config.ts
```

## 本地启动

```bash
# 1. 安装依赖（Node 20 LTS 推荐）
npm install

# 2. 配置环境变量
cp .env.local.example .env.local
# 编辑 .env.local 填入 Supabase URL / anon key（不填则投稿接口返回提示、点赞降级为静态基线）

# 3. 生成搜索索引
npm run build-search-index

# 4. 启动开发服务器
npm run dev
# 访问 http://localhost:3000
```

生产构建验证：

```bash
npm run build-search-index
npm run build
npm run start
```

## 环境变量

| 变量 | 说明 |
| --- | --- |
| `NEXT_PUBLIC_SUPABASE_URL` | Supabase 项目 URL |
| `NEXT_PUBLIC_SUPABASE_ANON_KEY` | Supabase Publishable(anon) key |
| `NEXT_PUBLIC_SITE_URL` | 站点域名（用于 sitemap） |
| `NEXT_PUBLIC_CONTACT_EMAIL` | 内容异议/删除申请接收邮箱（可选） |

> 安全红线：仅使用 Publishable key；**Secret Key 严禁进入前端 / 仓库**；`.env.local` 已加入 `.gitignore`。

## 数据库说明（Supabase 只读模型）

- `submissions`：用户匿名投稿表，匿名仅允许 `INSERT`，审核在 Supabase Dashboard 人工完成。
- `interactions`：互动聚合表，匿名前端仅 `SELECT`，写入由二期 Edge Function 负责。

> 选手、梗主体数据**不在数据库**，全部托管在 `/data` 静态文件，改动经 Git 提交触发部署上线。

## 新增选手 / 梗

1. 编辑 `data/{game}/players.json` 追加选手对象（字段遵循 `src/lib/zod-schema.ts`）。
2. 在 `data/{game}/memes/` 新建 `{playerKey}.mdx`，frontmatter 使用 `memes` 数组，正文用 `## meme-{id}` 分段。
3. 运行 `npm run build-search-index` 并本地验证。
4. `git commit` + `git push`，Vercel 自动构建部署。

## 用户投稿审核发布流程（人工）

1. 用户前端投稿 → 写入 `submissions`（`audit_status=pending`）。
2. 运营登录 Supabase Dashboard → Table Editor → `submissions` 逐条审核。
3. 审核通过后，**人工**把内容整理写入对应 `/data/{game}/memes/{playerKey}.mdx`。
4. `npm run build-search-index` → `git push` → Vercel 部署上线。
5. 回到 Dashboard 将 `audit_status` 改为 `approved` 归档。

> 注意：仅改数据库 `audit_status` **不会**自动上线页面，必须写入 MDX 并部署。

## MVP 边界

- ✅ 实现：SSG 首页/分类/选手详情、Zod 校验、Fuse.js 分片搜索、匿名投稿、localStorage 点赞、三层合规、Vercel 部署。
- ❌ 二期：管理后台、点赞实时写库/聚合回写、AI 串文案、注册登录、OG 动态分享图、评论社区。
