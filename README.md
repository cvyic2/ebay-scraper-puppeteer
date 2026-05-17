# Puppeteer eBay 爬虫实战：从入门到反爬墙，我踩过的坑都在这里

用 Puppeteer 爬eBay，听起来简单，真跑起来才知道有多少坑在等你。

我第一次写 eBay 爬虫是为了抓竞品的价格变动，逻辑不复杂——打开商品页，抓标题、价格、卖家评分，存进数据库。本地跑得好的，部署到服务器上跑了两天，IP 就被封了。换代理，继续跑，没多久又封。后来加了随机 User-Agent、模拟鼠标移动、随机延迟，依然没逃过 eBay 的检测。

折腾了大概三周，我才意识到问题不在 Puppeteer 本身，而在于我对 eBay 反爬机制的理解太浅。这篇文章是我把这段经历整理出来的结果——包括 Puppeteer 的基础写法、eBay 的反爬逻辑、以及我最后用什么方案稳定跑起来的。如果你现在也搞 eBay 数据采集，应该能省不少时间。

---

## ScraperAPI 是什么，为什么我最后选了它

一句话：ScraperAPI 是一个专门处理反爬问题的代理 API，你把目标 URL 丢给它，它帮你搞定 IP 轮换、浏览器指纹、验证码、JS 渲染，返回干净的 HTML。

它不是普通的代理池。普通代理只换 IP，ScraperAPI 在这之上还处理了请求头伪装、TLS 指纹、地理位置匹配这些细节——而这些恰好是 eBay 检测的重点。

我把它接进 Puppeteer 之后，同样的爬虫逻辑，封号率从"两天必封"变成了跑了三个月没出问题。

---

## 全套餐对比

| 套餐 | 月API 调用量 | 并发数 | 价格（月付） | 适合谁 | 立即获取 |
| ------ | ---------- | ------------ | ----- | ------ | --- |
| **Hobby** | 250,000 次 | 5 | $49/月 | 个人项目、小规模测试 | [ 获取 Hobby 套餐当前价格](https://www.scraperapi.com/?fp_ref=coupons) |
| **Startup** | 1,000,000 次 | 10 | $149/月 | 初创团队、中等规模采集 | [ 获取 Startup 套餐当前价格](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| **Business** | 3,000,000 次 | 25 | $299/月 | 商业级数据管道、多项目并行 | [ 获取 Business 套餐当前价格](https://www.scraperapi.com/pricing/?fp_ref=coupons) |
| **Enterprise** | 自定义 | 自定义 | 联系报价 | 大规模生产环境、SLA 要求 | [ 联系获取 Enterprise 专属方案](https://www.scraperapi.com/pricing/?fp_ref=coupons) |

所有付费套餐都提供 7 天退款保证，免费试用额度是 5,000 次 API 调用，不需要绑卡。

[👉 先用免费额度测试你的 eBay 爬虫](https://www.scraperapi.com/?fp_ref=coupons)

---

## 我为什么用它，而不是自己维护代理池

维护代理池这件事，我干过。买住宅代理、写 IP 健康检测、处理封号后的自动切换逻辑——这套东西跑起来之后，我发现自己每周有将近两天在处理代理相关的问题，而不是在写业务逻辑。

eBay 的反爬不是静态的。它会根据请求频率、请求模式、TLS 握手特征动态调整封锁策略。你今天绕过去了，明天它更新规则，你又得重新调。这是一场没有终点的猫鼠游戏。

ScraperAPI 的核心价值在于：它把这场游戏外包出去了。我不需要关心 eBay 今天改了什么检测逻辑，那是 ScraperAPI 的问题。我只需要关心我的数据结构和业务逻辑。

---

## Puppeteer + ScraperAPI 爬 eBay 的实际写法

### 基础接入方式

最简单的接入方式是把 ScraperAPI 当代理用，Puppeteer 的代码改动极小：

```javascript
const puppeteer = require('puppeteer');

const API_KEY = 'your_api_key_here';
const TARGET_URL = 'https://www.ebay.com/sch/i.html?_nkw=mechanical+keyboard';

// ScraperAPI 代理地址
const PROXY = `http://scraperapi:${API_KEY}@proxy-server.scraperapi.com:8001`;

(async () => {
  const browser = await puppeteer.launch({
    args: [
      `--proxy-server=${PROXY}`,
      '--no-sandbox',
      '--disable-setuid-sandbox',
    ],
    ignoreHTTPSErrors: true, // ScraperAPI 使用自签证书
  });

  const page = await browser.newPage();
  // 设置合理的超时，ScraperAPI 处理 JS 渲染需要更长时间
  page.setDefaultNavigationTimeout(6000);
  
  await page.goto(TARGET_URL, { waitUntil: 'networkidle2' });
  
  // 抓取 eBay 搜索结果
  const items = await page.evaluate(() => {
    const results = [];
    const listings = document.querySelectorAll('.s-item');
    listings.forEach(item => {
      const title = item.querySelector('.s-item__title')?.textContent?.trim();
      const price = item.querySelector('.s-item__price')?.textContent?.trim();
      const link = item.querySelector('.s-item__link')?.href;
      const condition = item.querySelector('.SECONDARY_INFO')?.textContent?.trim();
      if (title && title !== 'Shop on eBay') {
        results.push({ title, price, link, condition });
      }
    });
    return results;
  });
  
  console.log(`抓到 ${items.length} 条结果`);
  console.log(items.slice(0, 3)); // 打印前三条验证
  
  await browser.close();
})();
```

### 抓取商品详情页

搜索列表页只是第一步，真正有价值的数据在商品详情页——卖家信息、历史成交价、商品描述、买家评价数量：

```javascript
const puppeteer = require('puppeteer');

const API_KEY = 'your_api_key_here';
const PROXY = `http://scraperapi:${API_KEY}@proxy-server.scraperapi.com:8001`;

async function scrapeEbayItem(itemUrl) {
  const browser = await puppeteer.launch({
    args: [
      `--proxy-server=${PROXY}`,
      '--no-sandbox',
      '--disable-setuid-sandbox',
    ],
    ignoreHTPSErrors: true,
  });

  const page = await browser.newPage();
  page.setDefaultNavigationTimeout(6000);

  try {
    await page.goto(itemUrl, { waitUntil: 'networkidle2' });

    const itemData = await page.evaluate(() => {
      // 商品标题
      const title = document.querySelector('#itemTitle')?.textContent
        ?.replace('Details about', '')?.trim()
        || document.querySelector('.x-item-title__mainTitle span')?.textContent?.trim();

      // 当前价格
      const price = document.querySelector('#prcIsum')?.textContent?.trim()
        || document.querySelector('.x-price-primary span')?.textContent?.trim();

      // 卖家信息
      const selerName = document.querySelector('.mbg-nw')?.textContent?.trim();
      const sellerFeedback = document.querySelector('.mbg-l')?.textContent?.trim();

      // 商品状态
      const condition = document.querySelector('#vi-itm-cond')?.textContent?.trim()
        || document.querySelector('.x-item-condition-value span')?.textContent?.trim();

      // 已售数量（如果有）
      const soldCount = document.querySelector('.vi-qtyS-hot-red')?.textContent?.trim();

      // 商品图片
      const mainImage = document.querySelector('#icImg')?.src
        || document.querySelector('.ux-image-carousel-item img')?.src;

      return {
        title,
        price,
        sellerName,
        sellerFeedback,
        condition,
        soldCount,
        mainImage,
        url: window.location.href,
        scrapedAt: new Date().toISOString(),
      };
    });

    return itemData;
  } finally {
    await browser.close();
  }
}

// 使用示例
scrapeEbayItem('https://www.ebay.com/itm/example-item-id')
  .then(data => console.log(data))
  .catch(err => console.error('抓取失败:', err));
```

### 批量抓取 + 并发控制

实际项目里通常要抓几百上千条，需要控制并发避免打爆 API 配额：

```javascript
const puppeteer = require('puppeteer');

const API_KEY = 'your_api_key_here';
const PROXY = `http://scraperapi:${API_KEY}@proxy-server.scraperapi.com:8001`;

// 控制并发数，对应你的套餐并发上限
const CONCURRENCY = 5; // Hobby 套餐用5，Startup 可以用 10

async function scrapeWithConcurrency(urls, concurrency) {
  const results = [];
  const queue = [...urls];
  const workers = [];

  const browser = await puppeteer.launch({
    args: [
      `--proxy-server=${PROXY}`,
      '--no-sandbox',
      '--disable-setuid-sandbox',
    ],
    ignoreHTTPSErrors: true,
  });

  async function worker() {
    while (queue.length > 0) {
      const url = queue.shift();
      if (!url) break;

      const page = await browser.newPage();
      page.setDefaultNavigationTimeout(60000);

      try {
        await page.goto(url, { waitUntil: 'networkidle2' });
        
        const data = await page.evaluate(() => ({
          title: document.querySelector('.x-item-title__mainTitle span')?.textContent?.trim(),
          price: document.querySelector('.x-price-primary span')?.textContent?.trim(),
          url: window.location.href,
        }));

        results.push({ success: true, data });
        console.log(`✓ 完成: ${url.substring(0, 60)}...`);
      } catch (err) {
        results.push({ success: false, url, error: err.message });
        console.error(`✗ 失败: ${url.substring(0, 60)}...`);
      } finally {
        await page.close();
      }

      // 请求间隔，避免过于密集
      await new Promise(resolve => setTimeout(resolve, 1000 + Math.random() * 2000));
    }
  }

  // 启动并发 worker
  for (let i = 0; i < concurrency; i++) {
    workers.push(worker());
  }

  await Promise.all(workers);
  await browser.close();

  return results;
}

// 使用示例
const ebayUrls = [
  'https://www.ebay.com/itm/item1',
  'https://www.ebay.com/itm/item2',
  // ... 更多 URL
];

scrapeWithConcurrency(ebayUrls, CONCURRENCY)
  .then(results => {
    const successful = results.filter(r => r.success);
    console.log(`成功: ${successful.length}/${results.length}`);
  });
```

### 用 ScraperAPI 的 Async 模式处理大批量任务

如果要抓的量很大，同步等待每个请求会很慢。ScraperAPI 提供异步模式，提交任务后轮询结果：

```javascript
const axios = require('axios');

const API_KEY = 'your_api_key_here';

async function submitAsyncJob(url) {
  const response = await axios.post(
    'https://async.scraperapi.com/jobs',
    {
      apiKey: API_KEY,
      url: url,
      render: true, // eBay 需要 JS 渲染
    }
  );
  return response.data.id; // 返回 job ID
}

async function pollJobResult(jobId, maxWait = 60000) {
  const startTime = Date.now();
  while (Date.now() - startTime < maxWait) {
    const response = await axios.get(
      `https://async.scraperapi.com/jobs/${jobId}`,
      { params: { apiKey: API_KEY } }
    );
    if (response.data.status === 'finished') {
      return response.data.responsebody;
    }
    
    if (response.data.status === 'failed') {
      throw new Error(`Job ${jobId} failed`);
    }
    
    // 等3 秒再查
    await new Promise(resolve => setTimeout(resolve, 3000));
  }
  
  throw new Error(`Job ${jobId} timed out`);
}

// 批量提交，不阻塞等待
async function batchScrape(urls) {
  // 提交所有任务
  const jobIds = await Promise.all(urls.map(url => submitAsyncJob(url)));
  console.log(`提交了 ${jobIds.length} 个任务`);
  // 并行等待所有结果
  const results = await Promise.all(
    jobIds.map(id => pollJobResult(id).catch(err => ({ error: err.message })))
  );
  
  return results;
}
```

---

## 各套餐适合什么场景

### Hobby — $49/月，250,000 次调用

个人项目的起点。250,000 次调用听起来多，但如果你要抓详情页，每次调用算一次，一天抓 8,000 条就用完了。

**什么人买这个**：做价格监控的个人卖家、学生项目、验证想法阶段的独立开发者。不需要高并发，跑个定时任务每天抓几千条完全够用。

### Startup — $149/月，1,000,000 次调用

这个档位性价比最高。并发从 5 提升到 10，调用量是 Hobby 的四倍，价格只涨了三倍。

**什么人买这个**：有实际业务需求的小团队，比如做比价工具、电商选品分析、竞品监控。我自己用的就是这个档，跑了三个月，月均用了大概 60 万次调用，没超过。

### Business — $299/月，3,000,000 次调用

25 个并发，适合需要快速抓完大量数据的场景。比如你要在一天内抓完某个品类的所有在售商品，或者同时跑多个爬虫项目。

**什么人买这个**：数据服务商、有多个客户项目的自由职业者、需要实时数据的商业应用。

### Enterprise — 联系报价

自定义一切。如果你的需求是每月几千万次调用，或者需要专属 IP 段、SLA 保证、专属技术支持，这个档才有意义。

[👉 查看 Business 套餐完整配置与最新价格](https://www.scraperapi.com/pricing/?fp_ref=coupons)

---

## 真实使用体感：好用的地方和不完美的地方

**好用的地方**：

接入真的很简单。我把代理地址换进去，原来的 Puppeteer 代码基本不用改，跑起来就能用。文档写得清楚，各种语言的示例都有。

eBay 的 JS 渲染问题它处理得很好。eBay 很多价格和库存信息是动态加载的，普通 HTTP 请求抓不到，ScraperAPI 的 `render=true` 参数会等 JS 执行完再返回 HTML，这个功能省了我很多麻烦。

地理位置定向也有用。eBay 会根据访问者 IP 的地理位置返回不同的价格和商品，ScraperAPI 支持指定国家，抓美区 eBay 就指定美国 IP，数据更准确。

**不完美的地方**：

响应速度比直连慢。因为中间多了一层处理，平均响应时间大概在 3-8 秒，比直连慢不少。如果你的业务对实时性要求很高，这个延迟需要考虑进去。

偶尔会遇到超时。eBay 某些页面加载比较重，加上 ScraperAPI 的处理时间，偶尔会超过默认超时。我把超时设到 60 秒之后基本没再出现这个问题。

免费额度 5,000 次用完就没了，测试阶段要省着用，先在小数据集上验证逻辑再放量。

---

## 和自建方案横向对比

自建代理池方案的成本不只是代理费用。你还需要算上：维护代理健康检测的开发时间、处理封号后的应急时间、跟上反爬规则更新的持续投入。

我粗算过，维护一套能稳定跑 eBay 的自建方案，每个月大概要花 10-15 小时在运维上。按时薪算，这个隐性成本比 ScraperAPI 的订阅费高得多。

当然，如果你的团队有专职的爬虫工程师，或者你的数据需求非常特殊（比如需要特定 ISP 的 IP），自建方案可能更合适。但对大多数开发者来说，把这部分外包出去，把时间花在业务逻辑上，是更划算的选择。

---

## FAQ

**Puppeteer 爬 eBay 合法吗？**

eBay 的服务条款禁止未经授权的自动化数据采集。如果你需要大规模商业用途的数据，eBay 有官方 API（Finding API、Browse API）可以申请。Puppeteer 爬虫通常用于个人研究、价格监控等场景，具体合规性取决于你的使用目的和规模。

**为什么我的 Puppeteer 爬虫在 eBay 上频繁被封？**

eBay 的检测维度很多：请求频率、User-Agent 特征、TLS 指纹、Cookie 状态、鼠标行为模式。单纯换 IP 不够，还需要处理浏览器指纹伪装。ScraperAPI 在这些维度上都做了处理，这是它比普通代理池更有效的原因。

**ScraperAPI 支持 Puppeteer 吗？**

支持。接入方式是把 ScraperAPI 的代理地址配置到 Puppeteer 的 `--proxy-server` 参数里，或者直接用 ScraperAPI 的 HTTP API（不需要 Puppeteer，直接发 GET 请求，适合不需要交互的场景）。

**eBay 搜索结果页和商品详情页的 CSS 选择器经常变吗？**

会变，而且变得比较频繁。eBay 会定期更新前端代码，选择器可能失效。建议用多个备用选择器（`querySelector` 链式 `||`），同时监控抓取成功率，发现异常及时更新选择器。

**ScraperAPI 的 5,000 次免费额度够测试 eBay 爬虫吗？**

够用来验证基本逻辑。抓 100-200 个商品详情页，测试选择器是否正确、数据结构是否符合预期，5,000 次完全够。建议先在小数据集上把逻辑跑通，再升级到付费套餐放量。

**并发数超过套餐上限会怎样？**

超出并发上限的请求会排队等待，不会直接报错，但响应时间会变长。如果你的任务对时效性要求高，选并发数匹配你实际需求的套餐。

**ScraperAPI 能处理 eBay 的验证码吗？**

能。ScraperAPI 内置了验证码处理机制，遇到 CAPTCHA 会自动处理后再返回结果，这个过程对你的代码是透明的。

---

个人卖家或者小项目，Hobby 套餐够用；有实际业务数据需求的，Startup 是性价比最高的选择，我自己就在用这个档。

[👉 直接前往官网选购最适合的方案](https://www.scraperapi.com/?fp_ref=coupons)
