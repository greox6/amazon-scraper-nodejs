# Node.js 亚马逊抓取教程：用 ScraperAPI 绕过反爬机制实现商品数据自动采集

你试过用 Node.js 直接请求亚马逊商品页面吗？大概率会收到一个 503 或者验证码页面。亚马逊的反爬系统是电商平台里最激进的之一——IP 封禁、浏览器指纹检测、动态渲染、验证码轮番上阵。我去年做一个选品分析工具时，光是维护代理池和 header轮换逻辑就花了两周，最后稳定性还是不到 70%。

后来切到 ScraperAPI 之后，整个抓取层的代码缩减到不到 30 行，成功率直接拉到 99% 以上。这篇文章就把我实际跑通的方案拆开讲：从环境搭建、请求构造、结构化解析到批量抓取调度，一步步带你搞定亚马逊商品数据采集。

## 为什么亚马逊抓取这么难

亚马逊的反爬策略不是单一维度的，而是多层叠加：

1. **IP 频率限制**——同一 IP 短时间内请求超过阈值直接封禁，住宅代理也扛不住高频

2. **浏览器指纹检测**——检查 TLS 指纹、Canvas 渲染、WebGL 参数，Headless Chrome 默认配置秒被识别

3. **动态内容加载**——价格、库存、评论区大量依赖 JavaScript 渲染，纯 HTTP 请求拿到的是空壳

4. **地理位置敏感**——不同地区看到的价格、库存、配送选项完全不同

5. **验证码挑战**——CAPTCHA 触发频率高，尤其在搜索结果页和高流量商品页

自己搭方案意味着你要同时解决代理轮换、浏览器模拟、JS 渲染、验证码识别这四个问题。能跑通，但维护成本极高。

## ScraperAPI 怎么解决这些问题

ScraperAPI 本质上是一个智能代理网关。你把目标 URL 丢给它的 API 端点，它在后端自动处理：

- **代理轮换**：超过 4000 万个 IP 池，覆盖住宅、数据中心、移动端三种类型

- **自动重试**：请求失败时自动切换 IP 和 header 组合重试

- **JS 渲染**：通过 `render=true` 参数启用无头浏览器渲染

- **地理定位**：支持指定国家/地区的出口 IP

- **结构化数据端点**：针对亚马逊有专门的 Structured Data Endpoint，直接返回 JSON 格式的商品信息

对开发者来说，最大的好处是把反爬对抗的复杂度从你的代码里彻底剥离。你只需要关心"拿到数据后怎么用"。

## 环境搭建与基础请求

### 前置准备

```bash

mkdir amazon-scraper && cd amazon-scraper

npm init -y

npm install axios cheerio

```

注册 ScraperAPI 账号后会拿到一个 API Key。免费套餐给 5000 次请求额度，足够跑通整个教程。

### 最简单的请求方式

```javascript

const axios = require('axios');

const API_KEY = 'YOUR_API_KEY';

const targetUrl = 'https://www.amazon.com/dp/B09V3KXJPB';

async function scrapeProduct(url) {

const response = await axios.get('https://api.scraperapi.com', {

params: {

api_key: API_KEY,

url: url,

render: true,

country_code: 'us'

}

});

return response.data;

}

scrapeProduct(targetUrl).then(html => {

console.log(`页面长度: ${html.length} 字符`);

});

```

三个关键参数：

- `render=true`：启用 JS 渲染，确保动态内容加载完成

- `country_code`：指定地理位置，亚马逊价格和库存因地区而异

- `url`：目标页面的完整 URL

一次 API 调用，代理轮换、header 伪装、验证码处理全在后端完成。

## 用 Cheerio 解析商品数据

拿到 HTML 后用 Cheerio 提取结构化信息：

```javascript

const cheerio = require('cheerio');

function parseProduct(html) {

const $ = cheerio.load(html);

return {

title: $('#productTitle').text().trim(),

price: $('.a-price-whole').first().text().trim()

+ $('.a-price-fraction').first().text().trim(),

rating: $('#acrPopover .a-icon-alt').first().text().trim(),

reviewCount: $('#acrCustomerReviewText').first().text().trim(),

availability: $('#availability span').first().text().trim(),

features: $('#feature-bullets .a-list-item')

.map((i, el) => $(el).text().trim())

.get()

.filter(t => t.length > 0),

imageUrl: $('#landingImage').attr('src')

};

}

```

> 亚马逊的 DOM 结构会不定期调整，建议每次跑之前先手动检查一下选择器是否还有效。我一般每两周验证一次。

## 亚马逊结构化数据端点（推荐）

如果你不想自己写解析逻辑，ScraperAPI 提供了针对亚马逊的 Structured Data Endpoint，直接返回 JSON：

```javascript

async function getProductStructured(asin) {

const response = await axios.get('https://api.scraperapi.com/structured/amazon/product', {

params: {

api_key: API_KEY,

asin: asin,

country: 'us',

tld: '.com'

}

});

return response.data;

}

// 返回的 JSON 包含 name, price, rating, reviews, images, specifications 等字段

```

这个端点省掉了 Cheerio 解析那一层，数据结构稳定，不受亚马逊前端改版影响。缺点是每次调用消耗的 API credits 比普通请求多（算 25 个 credits）。

## 批量抓取与并发控制

实际项目里你不会只抓一个商品。批量抓取需要控制并发，避免触发 ScraperAPI 的并发限制：

```javascript

async function batchScrape(asins, concurrency = 5) {

const results = [];

for (let i = 0; i < asins.length; i += concurrency) {

const batch = asins.slice(i, i + concurrency);

const promises = batch.map(asin =>

getProductStructured(asin).catch(err => ({

asin,

error: err.message

}))

);

const batchResults = await Promise.all(promises);

results.push(...batchResults);

// 批次间隔，降低压力

if (i + concurrency < asins.length) {

await new Promise(r => setTimeout(r, 1000));

}

}

return results;

}

```

并发数取决于你的套餐。免费版支持 5 个并发线程，Business 套餐到 100 个。跑大规模采集任务时，并发上限直接决定了你的吞吐量。

## 搜索结果页抓取

除了单品页，搜索结果页也是常见需求——比如监控某个关键词下的竞品排名：

```javascript

async function scrapeSearchResults(keyword, page = 1) {

const searchUrl = `https://www.amazon.com/s?k=${encodeURIComponent(keyword)}&page=${page}`;

const response = await axios.get('https://api.scraperapi.com', {

params: {

api_key: API_KEY,

url: searchUrl,

render: true,

country_code: 'us'

}

});

const $ = cheerio.load(response.data);

const products = [];

$('[data-component-type="s-search-result"]').each((i, el) => {

products.push({

position: i + 1,

asin: $(el).attr('data-asin'),

title: $(el).find('h2 span').text().trim(),

price: $(el).find('.a-price-whole').first().text().trim(),

rating: $(el).find('.a-icon-alt').first().text().trim(),

sponsored: $(el).find('.s-label-popover-default').length > 0

});

});

return products;

}

```

搜索结果页的反爬比单品页更严格。我测试过不带 `render=true` 的情况，大约 40% 的请求会返回验证码页面。加上渲染参数后基本稳定。

## ScraperAPI 套餐怎么选

不同规模的抓取需求对应不同套餐，这是目前在售的全部方案：

| 套餐名 | API Credits/月 | 并发线程数 | 地理定位 | 价格（月付） | 适合人群 | 操作 |
| ------ | --------------- | ---------- | ------------- | ------ | --- | --- |
| Free | 5,000 | 5 | ✓ | $0 | 学习测试、跑通 Demo | - |
| Hobby | 100,000 | 10 | ✓ | $49 | 个人项目、小规模监控 | - |
| Startup | 500,000 | 50 | ✓ | $149 | 中型电商团队、日常选品分析 | - |
| Business | 3,000,000 | 100 | ✓ | $299 | 大规模数据采集、多站点监控 | - |
| Enterprise | 自定义 | 自定义 | ✓ | 联系销售 | 企业级需求、定制 SLA | - |

> 结构化数据端点（Structured Data）每次调用消耗 25 credits，普通请求消耗 1-10 credits（取决于是否启用渲染和高级功能）。选套餐时按实际消耗量估算，别只看 credits 总数。

我自己跑一个日均 2000 个 ASIN 的监控任务，用的是 Startup 套餐，每月消耗大约 35 万 credits（主要用结构化端点）。刚好够用。

## 实战技巧与踩坑记录

**关于重试策略**：ScraperAPI 后端已经有自动重试，但偶尔还是会返回 500。我在应用层加了一个简单的指数退避：

```javascript

async function scrapeWithRetry(url, maxRetries = 3) {

for (let i = 0; i < maxRetries; i++) {

try {

return await scrapeProduct(url);

} catch (err) {

if (i === maxRetries - 1) throw err;

await new Promise(r => setTimeout(r, Math.pow(2, i) * 1000));

}

}

}

```

**关于 credits 消耗优化**：

- 不需要动态内容时关掉 `render=true`，能省 5-10 倍 credits

- 商品详情页的价格在 HTML 源码里就有，不一定需要渲染

- 搜索结果页建议开渲染，否则容易拿到不完整的列表

**关于数据存储**：抓下来的数据我一般先写 JSON Lines 文件，再批量导入数据库。别在抓取循环里做数据库写入，IO 阻塞会拖慢整体速度。

## 常见问题

### 免费套餐的 5000 credits 能抓多少商品？

如果用普通请求 + 渲染，大约每次消耗 10 credits，能抓 500 个页面。用结构化端点的话每次 25 credits，能抓 200 个商品。够跑通整个流程和做小规模测试。

### ScraperAPI 支持抓取亚马逊哪些国家站点？

所有主流站点都支持——美国、英国、德国、日本、加拿大、印度、法国、意大利、西班牙等。通过 `country_code` 参数或结构化端点的 `tld` 参数指定。

### 抓取亚马逊数据合法吗？

公开展示的商品信息（标题、价格、评分）属于公开数据。但要注意遵守亚马逊的 robots.txt 和服务条款，不要抓取用户隐私数据，不要对服务器造成过大压力。ScraperAPI 的请求频率控制本身就帮你做了一层保护。

### 结构化端点和自己用 Cheerio 解析哪个好？

结构化端点省心，数据格式稳定，不怕亚马逊改版。自己解析灵活度高，credits 消耗少，适合只需要特定字段的场景。我的建议是：原型阶段用结构化端点快速验证，上线后根据实际需要的字段切回自定义解析。

### 并发线程数不够用怎么办？

两个思路：升级套餐直接提升并发上限；或者在应用层做队列调度，把任务分散到更长的时间窗口里。如果你的场景不要求实时性，后者更经济。

### 被亚马逊封 IP 了怎么办？

用 ScraperAPI 的话这个问题基本不存在——被封的是它的代理 IP，它会自动切换。如果你发现成功率下降，检查一下是不是请求频率太高触发了 ScraperAPI 自身的限流。

## 完整项目结构参考

```

amazon-scraper/

├── src/

│ ├── scraper.js # 核心抓取逻辑

│ ├── parser.js # Cheerio 解析器

│ ├── scheduler.js # 批量调度与并发控制

│ └── storage.js # 数据持久化

├── data/

│ └── results.jsonl # 抓取结果

├── config.js # API Key、并发数等配置

└── package.json

```

把抓取、解析、调度、存储四层拆开，后续换数据源或者换存储方案时改动最小。

---

亚马逊抓取的核心难点从来不是"怎么发请求"，而是"怎么持续稳定地拿到数据"。ScraperAPI 把反爬对抗这一层完全托管掉，让你专注在数据处理和业务逻辑上。如果你正在做选品分析、价格监控、竞品追踪这类项目，省下来的时间远比套餐费用值得。

👉 [现在下单解锁限时折扣](https://www.scraperapi.com/?fp_ref=coupons#startup)
