[![使用 Instagram 爬虫 API 抓取 Instagram 数据：个人资料、帖子、Reels 短视频和搜索结果。可按 URL 或用户名采集或发现数据。免费开始使用。](.github/banner.png)](https://www.bright.cn/products/web-scraper/instagram?utm_source=github)

# instagram-scraper-node

[快速开始](#快速开始) · [命令行使用](#作为命令运行) · [API 接口](#其他-api-接口) · [数据](#数据) · [错误处理](#出错时) · [编程智能体](#编程智能体) · [文档](https://docs.brightdata.com/products/scrapers/instagram/introduction) · [支持](#支持)

使用 JavaScript 将 Instagram 个人资料、帖子、Reels 短视频和评论获取为 JSON。无需登录 Instagram，也无需浏览器。基于 [Bright Data Instagram 爬虫 API](https://www.bright.cn/products/web-scraper/instagram?utm_source=github) 构建。

本项目使用 [Bright Data JavaScript SDK](https://github.com/bright-cn/sdk-js)。完整 API 文档：[Instagram 爬虫 API](https://docs.brightdata.com/products/scrapers/instagram/introduction)。

本仓库还提供用于获取帖子的单命令 CLI，以及完全无需编写 JavaScript 的 [Bright Data CLI](#编程智能体)。

## 快速开始

需要 Node 20 或更新版本。这个包使用 ESM，因此请使用 `import`，不要使用 `require`。

```bash
npm install @brightdata/sdk
export BRIGHTDATA_API_TOKEN=YOUR_API_KEY
```

从 [Bright Data 控制面板](https://www.bright.cn/cp/setting/users)获取令牌。此 SDK 不会自行读取 `.env` 文件；可以让 Node 通过 `node --env-file=.env yourscript.mjs` 加载。

也可以不手动设置令牌。先运行一次 `npx -p @brightdata/cli bdata login`：它会打开浏览器。此后，SDK 会自行找到已保存的凭据，供你以及在该终端工作的任何编程智能体使用。智能体无法自行完成浏览器中的登录操作，因此请先亲自登录。同一个 CLI 也能直接抓取 Instagram 数据；详见[编程智能体](#编程智能体)。

还没有账户？[创建账户](https://www.bright.cn/cp/start)；新账户每月可获得 [5,000 免费积分](https://docs.brightdata.com/general/account/billing-and-pricing/free-tier)。

```javascript
import { bdclient } from "@brightdata/sdk";

const client = new bdclient({ autoCreateZones: false });
const job = await client.scrape.instagram.discoverPostsByProfileURL(
  [{ url: "https://www.instagram.com/nasa/", num_of_posts: 5 }],
  { includeErrors: true },
);
const result = await job.toResult({ pollTimeout: 600_000 });
if (!result.success) throw new Error(`${result.status}: ${result.error}`);
for (const post of result.data) {
  console.log(post.likes, post.num_comments, post.url);
}
await client.close();
```

```text
73112 340 https://www.instagram.com/p/DdWojaYFDf-/
2445311 4736 https://www.instagram.com/p/Db9IVmrDvQ4/
62758 227 https://www.instagram.com/p/DdWyDLcGiqo/
573325 4334 https://www.instagram.com/p/DcOX3hWFiey/
109891 1005 https://www.instagram.com/reel/DcMXl1IPNtB/
```

预计需要一至三分钟：API 会运行一个任务，`toResult` 则会等待任务完成。每条帖子消耗 [1 个积分](https://www.bright.cn/pricing/web-scraper)。

上面的代码片段中，以下设置都不可省略。

`autoCreateZones: false` 会阻止 SDK 在启动时创建区域。这些区域用于网络解锁器和搜索引擎 API，是此爬虫工具不会用到的另外两款 Bright Data 产品。如果账户未添加付款方式，创建区域会失败。

`includeErrors: true` 会让 API 把无效账户作为一行结果返回。不设置它，这一行会被丢弃，你将收不到对应结果。

`pollTimeout` 的单位是毫秒，不是秒。如果直接复制 Python 示例中的数字，可能还没进行第一次状态检查就已超时。

`discoverPostsByProfileURL` 返回任务，而不是直接返回记录。`toResult` 会轮询，直到快照就绪，然后获取结果。

## 作为命令运行

此仓库中的命令可以处理多个账户，并将结果写入一个 JSON 文件。

```bash
npm install -g github:bright-cn/instagram-scraper-node
instagram-scraper nasa natgeo
```

```text
Fetching up to 5 recent posts per account, for: nasa, natgeo
Usually one to three minutes each. One credit per post, 5,000 free per month.
asking  @nasa...
got     @nasa: 5 posts
asking  @natgeo...
got     @natgeo: 5 posts

Saved 10 posts as JSON to instagram.json (34 fields per post)
```

在终端中，`asking` 行会被下面的进度显示替换，并原地更新，让你知道任务仍在运行以及已经运行了多久：

```text
⠹ @nasa 0:01:47
```

```text
--limit N    posts per account, default 5, minimum 1
--out PATH   output file, default instagram.json
```

还可以导入它，而不是将它作为命令运行。这样，每个账户都会得到 `ok`、`note` 和 `error` 状态，而不只是原始数据行。单个账户出错不会使 `scrape` 拒绝整个请求；读取 `posts` 前请先检查 `ok`：

```javascript
import { scrape } from "@brightdata/instagram-scraper-node";

for (const outcome of await scrape(["nasa", "zz_not_a_real_account_zz"], { limit: 1 })) {
  if (outcome.ok) {
    console.log(`${outcome.handle}: ${outcome.posts.length} posts`);
  } else {
    console.log(`${outcome.handle} failed: ${outcome.error}`);
  }
}
```

```text
nasa: 1 posts
zz_not_a_real_account_zz failed: Crawler error: Cannot read properties of null (reading 'pk')
```

如果账户近期没有帖子，调用仍算成功，只是返回零条帖子。

## 其他 API 接口

上面的命令对应下表第一行。其余各行是 SDK 提供的其他 Instagram 接口，详见 [Instagram 爬虫 API 文档](https://docs.brightdata.com/products/scrapers/instagram/introduction)。下面的每段代码都是完整示例，只需要 `@brightdata/sdk`，可直接粘贴运行。所有示例每周一都会在 Actions 中运行，其他日子还会执行规模较小的检查。页面顶部的徽章显示最近一次结果。

| 已有信息 | 想获取 | 调用方式 |
| --- | --- | --- |
| 个人资料 URL | 近期帖子 | `discoverPostsByProfileURL([{ url, num_of_posts: 5 }], { includeErrors: true })` |
| 用户名 | 对应个人资料 | JavaScript SDK 没有对应方法；`client.search` 仅支持 Google、Bing 和 Yandex |
| 个人资料 URL | 对应个人资料 | `profiles([url])` |
| 个人资料 URL | 近期 Reels 短视频 | `discoverReelsByProfileURL([{ url, num_of_posts: 2 }], { includeErrors: true })` |
| 个人资料 URL | 该账户发布过的所有 Reels 短视频 | `discoverAllReelsByProfileURL([url], { includeErrors: true })`；每条 Reels 短视频消耗 1 个积分 |
| 帖子 URL | 对应帖子 | `posts([url, url])` |
| Reels 短视频 URL | 对应 Reels 短视频 | `reels([url])` |
| 帖子或 Reels 短视频 URL | 对应评论 | `comments([url])`；每条评论消耗 1 个积分 |

这些方法都位于 `client.scrape.instagram` 下。Python SDK 的 `search.instagram.profiles("nasa")` 可以直接接受用户名，但此处没有对应方法，因此请从个人资料 URL 开始。

这些调用都会创建异步任务：API 触发任务，SDK 轮询状态，并在任务就绪后返回。这就是一次调用需要一至三分钟，以及 JavaScript 中没有更快调用方式的原因。API 的[同步接口](https://docs.brightdata.com/api-reference/scrapers/synchronous-requests)仅供直接通过 HTTP 调用，最多接受 20 个 URL，且有一分钟的时间限制。

四个简短方法名会在一次调用中完成触发、轮询和获取，并返回 `ScrapeResult`。名称以 `collect` 或 `discover` 开头的方法会返回需要自行轮询的 `ScrapeJob`。只有后一组方法可以传入 `includeErrors`。如果需要区分无效账户与近期没有帖子的账户，请使用后一组方法并传入该参数。

这里特意没有列出 `post_type`。SDK 的过滤器只接受 `"post"` 和 `"reel"`，但目前使用这两个值都无法返回帖子（[sdk-js#34](https://github.com/bright-cn/sdk-js/issues/34)）。

### 两个账户，一个任务

一个过滤条件数组只会创建一个任务，不会为每个账户各建一个任务。

```javascript
import { bdclient } from "@brightdata/sdk";

const client = new bdclient({ autoCreateZones: false });
const job = await client.scrape.instagram.discoverPostsByProfileURL(
  [
    { url: "https://www.instagram.com/nasa/", num_of_posts: 1 },
    { url: "https://www.instagram.com/natgeo/", num_of_posts: 1 },
  ],
  { includeErrors: true },
);
const result = await job.toResult({ pollTimeout: 600_000 });
if (!result.success) throw new Error(`${result.status}: ${result.error}`);
for (const post of result.data) {
  console.log(post.user_posted, post.url);
}
await client.close();
```

```text
natgeo https://www.instagram.com/reel/DdG4RIxIPyf/
nasa https://www.instagram.com/p/DcOX3hWFiey/
```

### 现在触发，稍后获取

如果要处理的不止几个账户，不要让进程阻塞一小时。先触发任务并保存快照 ID，待任务就绪后再获取结果。快照可在 30 天内下载。

```javascript
import { bdclient } from "@brightdata/sdk";

const client = new bdclient({ autoCreateZones: false });
const job = await client.scrape.instagram.collectPosts(
  ["https://www.instagram.com/p/DcOX3hWFiey/"],
  { async: true, includeErrors: true },
);
console.log("snapshot:", job.snapshotId);
await job.wait({ pollInterval: 5_000, pollTimeout: 600_000 });
console.log("status:", await job.status());
const [record] = await job.fetch();
console.log("fetched:", record.url, "likes:", record.likes);
await client.close();
```

```text
snapshot: sd_mu4iuj8j22cmvrseth
status: ready
fetched: https://www.instagram.com/p/DcOX3hWFiey/ likes: 573306
```

`async: true` 会让 `collectPosts` 返回任务。`ScrapeJob` 还提供 `download()` 方法，可将快照写入磁盘；也提供 `cancel()` 方法，用于停止任务。

### 指定日期范围

```javascript
import { bdclient } from "@brightdata/sdk";

const client = new bdclient({ autoCreateZones: false });
const job = await client.scrape.instagram.discoverPostsByProfileURL(
  [
    {
      url: "https://www.instagram.com/nasa/",
      num_of_posts: 3,
      start_date: "08-01-2026", // MM-DD-YYYY
      end_date: "09-07-2026",
    },
  ],
  { includeErrors: true },
);
const result = await job.toResult({ pollTimeout: 600_000 });
if (!result.success) throw new Error(`${result.status}: ${result.error}`);
for (const post of result.data) {
  console.log(post.date_posted, post.content_type, post.url);
}
await client.close();
```

```text
2026-08-19T14:11:47.000Z Image https://www.instagram.com/p/DcOX3hWFiey/
2026-08-12T21:28:58.000Z Image https://www.instagram.com/p/Db9IVmrDvQ4/
2026-08-18T19:37:40.000Z Reel https://www.instagram.com/reel/DcMXl1IPNtB/
```

如果指定日期范围内没有内容，调用会返回零行和 `success: true`，而不是错误。`posts_to_not_include` 接受帖子 ID 数组。

### 通过 URL 获取个人资料

```javascript
import { bdclient } from "@brightdata/sdk";

const client = new bdclient({ autoCreateZones: false });
const result = await client.scrape.instagram.profiles(["https://www.instagram.com/nasa/"]);
if (!result.success) throw new Error(`${result.status}: ${result.error}`);
const [profile] = result.data;
console.log(profile.account, "followers:", profile.followers, "posts:", profile.posts_count);
await client.close();
```

```text
nasa followers: 104352721 posts: 4925
```

### 获取帖子的评论

每条评论消耗 1 个积分，而且无法限制返回条数。因此，请先查看帖子的 `num_comments`。本 README 其他示例中使用的 NASA 帖子显示有 4,334 条评论；请求评论后返回了 3,322 行，因此消耗 3,322 个积分。这一次调用就用掉了每月免费额度的大部分。下面示例中的帖子返回 5 条评论。

```javascript
import { bdclient } from "@brightdata/sdk";

const client = new bdclient({ autoCreateZones: false });
const result = await client.scrape.instagram.comments([
  "https://www.instagram.com/p/Dc1W1uFj-CW/",
]);
if (!result.success) throw new Error(`${result.status}: ${result.error}`);
console.log(result.data.length, "comments; first:", JSON.stringify(result.data[0].comment.slice(0, 60)));
await client.close();
```

```text
5 comments; first: "I only WISH that I could be there! What an incredible evenin"
```

### 获取近期 Reels 短视频

```javascript
import { bdclient } from "@brightdata/sdk";

const client = new bdclient({ autoCreateZones: false });
const job = await client.scrape.instagram.discoverReelsByProfileURL(
  [{ url: "https://www.instagram.com/nasa/", num_of_posts: 2 }],
  { includeErrors: true },
);
const result = await job.toResult({ pollTimeout: 900_000 });
if (!result.success) throw new Error(`${result.status}: ${result.error}`);
for (const reel of result.data) {
  console.log(reel.date_posted, reel.url);
}
await client.close();
```

```text
2026-09-14T18:58:07.000Z https://www.instagram.com/p/DdRyQxKteC1/
2026-08-18T19:37:40.000Z https://www.instagram.com/p/DcMXl1IPNtB/
```

Python SDK 的对应示例表明，发现 Reels 短视频比发现帖子慢；这里的测试并非如此。此调用单独运行时在 62 秒内完成。但当同一账户还有另外七个任务同时运行时，它曾在等待 900 秒后超时。示例中较长的超时时间不会增加快速任务的费用。

## 数据

大多数人关心的字段：

```text
url  date_posted  description  hashtags  likes  num_comments  user_posted
```

代码没有硬编码字段列表。API 返回的所有字段都会进入 `result.data`，也会写入命令生成的文件。

如果帖子由多个账户合作发布，`user_posted` 中可能是合作作者的用户名。因此，从 `nasa` 获取的帖子也可能显示 `nasajohnson`；`coauthor_producers` 会列出所有合作作者。

<!-- fields:start -->
<details>
<summary>全部 44 个字段及其类型和说明</summary>

以下内容每天都会通过 `client.datasets.instagramPosts.getMetadata()` 从数据集的数据结构重新生成，因此不会过时。每条帖子只包含适用于它的字段；示例文件包含这 44 个字段中的 34 个。

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `url` | URL | Instagram 帖子的直接 URL |
| `user_posted` | 文本 | 帖子发布者的用户名 |
| `description` | 文本 | 帖子的文字说明 |
| `hashtags` | 数组 | 帖子使用的话题标签 |
| `num_comments` | 数字 | 评论数量 |
| `date_posted` | 日期 | 帖子发布日期 |
| `likes` | 数字 | 帖子获赞数量 |
| `photos` | 数组 | 所附图片的 URL；根据 Instagram 政策，这些 URL 可能过期 |
| `videos` | 数组 | 所附视频的 URL；根据 Instagram 政策，这些 URL 可能过期 |
| `location` | 数组 | 与帖子关联的地理位置 |
| `location_details` | 对象 | Instagram 返回的详细地理位置元数据 |
| `latest_comments` | 数组 | 帖子的近期评论 |
| `post_id` | 文本 | 帖子的唯一标识符 |
| `discovery_input` | 对象 | 用于触发数据采集的发现类输入值 |
| `has_handshake` | 布尔值 | 帖子是否带有账户之间的合作发布标识 |
| `display_url` | 文本 | 已弃用：此前用于表示帖子媒体的展示 URL |
| `shortcode` | 文本 | Instagram 帖子的短代码，用于帖子 URL 路径 |
| `content_type` | 文本 | 内容类型：普通帖子或 Reels 短视频 |
| `pk` | 文本 | Instagram 为媒体内容分配的主键 |
| `content_id` | 文本 | 媒体项目的内容 ID |
| `engagement_score_view` | 数字 | 用作互动评分指标的视频观看次数 |
| `thumbnail` | 文本 | 帖子展示图片或视频缩略图的 URL |
| `video_view_count` | 文本 | 视频帖子的观看次数 |
| `product_type` | 文本 | 与帖子关联的产品类型，例如 Reels 短视频对应的 `clips` |
| `coauthor_producers` | 数组 | 参与合作发布帖子的作者或制作人列表 |
| `tagged_users` | 数组 | 帖子中标记的用户列表 |
| `video_play_count` | 数字 | 视频播放次数 |
| `followers` | 数字 | 数据采集时，帖子发布者的粉丝数量 |
| `posts_count` | 数字 | 数据采集时，该账户发布的帖子总数 |
| `profile_image_link` | 文本 | 直达帖子发布者 Instagram 头像的 URL |
| `is_verified` | 布尔值 | 帖子发布者的账户是否已获得认证 |
| `is_paid_partnership` | 布尔值 | 帖子是否属于赞助内容或付费合作 |
| `partnership_details` | 对象 | 与帖子关联的付费合作品牌详情 |
| `user_posted_id` | 文本 | 帖子发布账户的 Instagram 用户 ID |
| `post_content` | 数组 | 帖子所附媒体项目（图片或视频）的列表，包括轮播内容 |
| `audio` | 对象 | 与帖子或 Reels 短视频关联的音轨元数据 |
| `profile_url` | URL | 帖子发布者的 Instagram 个人资料 URL |
| `videos_duration` | 数组 | 帖子所附各个视频的时长列表 |
| `images` | 数组 | 帖子所附图片对象的列表 |
| `alt_text` | 文本 | 帖子主图的无障碍替代文本：用于向盲人或视障用户描述图片含义 |
| `photos_number` | 数字 | 帖子所附图片总数 |
| `audio_url` | URL | 帖子所用音轨的直接 URL |
| `thumbnail_array` | 数组 | 已弃用：帖子媒体缩略图 URL 的数组 |
| `country` | 文本 | 某些个人资料受到地区限制。请使用 Alpha-2 格式设置国家或地区代码 |

</details>
<!-- fields:end -->

<details>
<summary>真实输出文件的开头，来自 <code>instagram-scraper nasa --limit 1</code></summary>

```json
{
  "generated_at": "2026-09-16T20:31:01.098Z",
  "handles": [
    {
      "handle": "nasa",
      "posts": [
        {
          "url": "https://www.instagram.com/p/DcOX3hWFiey/",
          "user_posted": "nasa",
          "description": "With your powers combined…\n\nThis colorful picture of the cosmos is the product of teamwork between our @NASAHubble, @NASAWebb, and @NASAChandraXray telescopes. Scientists brought their data together to get a vibrant look at this star-forming region known as the Tarantula Nebula, located 160,000 light-years from Earth.\n\nChandra’s X-ray data fills in the deep blue parts of the image, showing the gas blown away by the stellar winds created by the nebula’s young stars. Webb’s infrared data shows up as red, displaying the thousands of stars and the dust that gives rise to them. Hubble’s visible data is represented in green, showing warmer gas and other stars.\n\nCredit: NASA\n\n#NASA #Universe #Nebula",
          "hashtags": [
            "#NASA",
            "#Universe",
            "#Nebula"
          ],
          "num_comments": 4341,
          "date_posted": "2026-08-19T14:11:47.000Z",
          "likes": 573346,
          "photos": [
  ...
```

包含一条帖子及其全部字段的完整文件见 [examples/sample_output.json](examples/sample_output.json)。

</details>

## 出错时

| 看到的信息 | 含义 |
| --- | --- |
| `API token required but not found.` | 在发送任何请求之前以状态码 2 退出。请设置令牌。 |
| `failed  @name: ...` | 以状态码 1 退出。账户不存在，通常是名称输入有误。提示可能有所不同：`Sorry, this page isn't available.` 和 `Crawler error: Cannot read properties of null` 都可能表示这种情况。 |
| `got     @name: 0 posts` | 以状态码 0 退出，属于正确结果。指定时间范围内没有公开帖子。 |
| `failed  @name: Polling timed out after 605s for sd_...` | 以状态码 1 退出。请求在等待 600 秒后放弃。API 运行缓慢时可能出现这种情况；请重新运行。 |

任何失败都会以状态码 1 退出，因此脚本可以安全地依据运行结果决定是否继续。

在 SDK 中，相同情况表现如下：

| 看到的信息 | 含义 |
| --- | --- |
| `AuthenticationError: No API token found.` | 在选项、环境变量和 CLI 登录信息中都找不到令牌。 |
| 状态码为 401 的 `APIError` | 已设置令牌，但令牌不正确。 |
| `result.success` 为 `false`，`result.status` 为 `"timeout"` | SDK 等待超时。增大以毫秒为单位的 `pollTimeout`，或重新运行。 |
| `result.data` 中某一行包含 `error` 键 | 这是启用 `includeErrors` 后 API 对某条输入返回的结果；其他行不受影响。 |
| 原本预计会返回错误，却得到零行 | 没有传入 `includeErrors: true`，因此 API 丢弃了原本会说明错误原因的那一行。 |

## 编程智能体

无需编写 JavaScript，也无需预先安装任何内容。粘贴以下两行即可；第一行会打开一次浏览器。如果通过 SSH 或在 CI 中使用，请改用 `bdata login --device`：

```bash
npx -p @brightdata/cli bdata login
npx -p @brightdata/cli bdata pipelines instagram_posts "https://www.instagram.com/p/DcOX3hWFiey/"
```

CLI 的四种 Instagram 数据管道分别接受帖子、Reels 短视频或个人资料 URL，并返回该 URL 对应的一条记录。若要获取某份个人资料的近期帖子，请使用「快速开始」中的 SDK 调用；CLI 没有对应命令。

运行 `npx skills add brightdata/skills`，可以让 Claude Code、Cursor 和 Codex 学会这些命令并了解相关文档，此后就能用自然语言提出需求。完整指南：[让编程智能体使用 Bright Data](https://docs.brightdata.com/quickstart-coding-agent)。

使用托管助手、没有终端？[Bright Data MCP 服务器](https://github.com/bright-cn/brightdata-mcp#which-tool-to-use)的 `social` 工具组中也提供这四种 Instagram 工具，每次处理一个 URL。该工具组默认关闭，需要明确启用：

```text
https://mcp.brightdata.com/mcp?token=YOUR_API_TOKEN&groups=social
```

智能体还可以自行创建账户，无需填写注册表单：[智能体注册](https://www.bright.cn/auth.md)。如需了解 Bright Data 与 LangChain、Zapier、n8n 等工具的其他集成方式，请参阅[集成文档](https://docs.brightdata.com/integrations/introduction)。

## 支持

发现此仓库中的问题？请[提交 issue](https://github.com/bright-cn/instagram-scraper-node/issues)，并参照 [CONTRIBUTING.md](CONTRIBUTING.md) 说明提供必要信息。

有关 API、账户或积分的问题，请联系 [Bright Data 支持团队](https://brightdata.zendesk.com/hc/en-us/requests/new)。

## 许可证

MIT。
