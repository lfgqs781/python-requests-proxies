# Python Requests Proxies 完整使用教程：如何在 Python 中配置代理？如何处理认证、会话与轮换？常见报错怎么修？（含 Webshare 实战代码与套餐对比）

凌晨三点，爬虫脚本第 47 次被目标网站封 IP。控制台滚动着 `403 Forbidden`，而你只是想抓两万条公开商品数据。这个场景对写过爬虫的人来说太熟悉了——而解决它的钥匙，往就藏在 `requests` 库一个不起眼的参数里：`proxies`。

这篇文章就讲清楚一件事：**python requests proxies 到底怎么用才不踩坑**。从最基础的 HTTP/HTTPS 代理配置，到 SOCKS5 协议、用户名密码认证、Session 复用、IP 轮换、错误重试，再到怎么挑一个不让你半夜起来救火的代理服务，全部一次说透。

> **Python requests proxies 是什么？** 它是 Python 的 `requests` 库内置的一个参数，接收一个字典，键为协议名（`http` / `https`），值为代理服务器地址。所有发出的 HTTP 请求都会先经过这个代理，再到达目标服务器。

如果你只想直接拿到能跑的方案，我把后面用到的代理服务先放这：[👉 查看 Webshare 全部代理套餐与最新优惠](https://bit.ly/web_share)。下面是完整的技术拆解。

## 为什么你的 Python 爬虫离不开 requests proxies

直接拿本机 IP 去抓数据，结果通常有三种：被限速、被封 IP、被返回假数据。目标网站的反爬系统会盯着每个 IP 的请求频率、请求路径、Header 指纹。一旦你的脚本表现得"不像人"，封禁就是分钟的事。

代理的作用是把你的真实 IP 换掉。每次请求看上去都来自不同的地方——美国住宅、德国机房、巴西移动网络——目标网站没法把这些请求关联到同一个抓取者身上。

但代理本身分好几种。HTTP 代理只能转发 HTTP/TTPS 流量，速度快但匿名度有限。SOCKS5 代理协议层级更低，几乎什么流量都能转，应对复杂场景更稳。住宅代理用的是真实家庭宽带 IP，反爬系统几乎识别不出来；数据中心代理便宜量大，但容易被 IP 段整体拉黑。

挑错代理类型，后面的所有代码都白写。

## Python requests proxies 最基础的用法

`requests` 库本身已经支持代理，不需要额外装什么东西。最简单的写法：

python
import requests

proxies = {
    "http": "http://proxy.example.com:8080",
    "https": "http://proxy.example.com:8080",
}

response = requests.get("https://httpbin.org/ip", proxies=proxies)
print(response.json())


返回的 JSON 里 `origin` 字段就是当前出口 IP。如果显示的是代理服务器的 IP，而不是你本机的，配置就成功了。

注意一个细节：**`https 这个键的值，可以也是 `http://` 开头的**。这里指的不是目标网站的协议，而是连接代理服务器使用的协议。绝大多数代理服务用的就是 HTTP 协议来建立隧道，即便目标是 HTTPS 也一样。

## 带认证的代理怎么配

商业代理服务基本都需要用户名密码。`requests` 支持两种写法。

第一种，把凭据直接拼在 URL 里：

python
proxies = {
    "http": "http://username:password@proxy.example.com:8080",
    "https": "http://username:password@proxy.example.com:8080",
}


第二种，用 `HTTPProxyAuth`（不太常见，但场景特殊时有用）：

python
from requests.auth import HTPProxyAuth

auth = HTTPProxyAuth("username", "password")
response = requests.get(url, proxies=proxies, auth=auth)


实际工作中第一种用得最多。如果用户名或密码里包含 `@`、`:`、`/ 这种特殊字符，记得 URL 编码：

python
from urllib.parse import quote

user = quote("my@email.com")
pwd = quote("p@ss:word")
proxy_url = f"http://{user}:{pwd}@proxy.example.com:8080"


这个坑踩过的人不少。密码里一个普通的 `@` 就能让你 debug 一下午。

## SOCKS5 代理：requests 默认不支持，得装东西

`requests` 原生只认 HTTP/HTTPS 代理。想用 SOCKS5？先装依赖：

bash
pip install requests[socks]


或者：

bash
pip install pysocks


然后：

python
proxies = {
    "http": "socks5://username:password@proxy.example.com:1080",
    "https": "socks5://username:password@proxy.example.com:1080",
}

response = requests.get("https://httpbin.org/ip", proxies=proxies)


`socks5h://` 和 `socks5://` 的区别也得知道。前者把 DNS 解析也丢给代理服务器做，后者在本地解析域名再走代理。爬境外网站推荐用 `socks5h://`，避免本地 DNS 污染暴露真实意图。

## Session：高频抓取必须用的复用方式

如果你要抓 1000 个页面，每次都 `requests.get()` 重新建立连接，TCP 握手和 TLS 协商的开销会拖垮速度。正确做法是用 `Session`：

python
import requests

session = requests.Session()
session.proxies = {
    "http": "http://username:password@proxy.example.com:8080",
    "https": "http://username:password@proxy.example.com:8080",
}

for url in url_list:
    response = session.get(url, timeout=10)
    # 处理 response


Session 会自动复用底层连接，速度通常能快 30% 到 50%。Cookie 也会自动维护，登录态可以一直保持。

## IP 轮换：让每次请求换一个出口

单个代理 IP 抓久了照样被封。轮换策略有两种实现路径。

### 路径一：自己维护代理池

python
import requests
import random

proxy_list = [
    "http://user:pass@proxy1.example.com:8080",
    "http://user:pass@proxy2.example.com:8080",
    "http://user:pass@proxy3.example.com:8080",
    # ...更多代理
]

def get_with_random_proxy(url):
    proxy = random.choice(proxy_list)
    proxies = {"http": proxy, "https": proxy}
    return requests.get(url, proxies=proxies, timeout=10)


这种方式适合代理数量不多、有完整列表的场景。

### 路径二：用 Backconect 端点

商业代理服务通常提供一个固定入口端点，每次请求自动从池子里分配新 IP。代码不用改，配置一次搞定：

python
proxies = {
    "http": "http://user:pass@p.webshare.io:80",
    "https": "http://user:pass@p.webshare.io:80",
}

# 每次请求都会从代理池中分配不同的出口 IP
for in range(100):
    response = requests.get("https://httpbin.org/ip", proxies=proxies)
    print(response.json()["origin"])


后者省心得多，尤其当池子规模到几万 IP 时。Webshare 的旋转代理（Rotating Residential / Rotating Datacenter）就是这种模式，[👉 立即开通 Webshare 旋转代理](https://bit.ly/web_share) 几分钟就能跑起来。

## 超时、重试、错误处理：爬虫的三件套

代理总会偶尔抽风。一个稳定的爬虫脚本必须有完整的容错：

python
import requests
from requests.adapters import HTTPAdapter
from urllib3.util.retry import Retry

session = requests.Session()

retry_strategy = Retry(
    total=3,
    backoff_factor=1,
    status_forcelist=[429, 500, 502, 503, 504],
    allowed_methods=["GET", "POST"]
)

adapter = HTTPAdapter(max_retries=retry_strategy)
session.mount("http://", adapter)
session.mount("https://", adapter)

session.proxies = {
    "http": "http://user:pass@p.webshare.io:80",
    "https": "http://user:pass@p.webshare.io:80",
}

try:
    response = session.get("https://target.com/page", timeout=(5, 15))
    response.raise_for_status()
except requests.exceptions.ProxyError as e:
    print(f"代理连接失败: {e}")
except requests.exceptions.Timeout as e:
    print(f"请求超时: {e}")
except requests.exceptions.RequestException as e:
    print(f"其他错误: {e}")


`timeout=(5, 15)` 是分别设置连接超时和读取超时——前者 5 秒内没建立连接就放弃，后者数据传输超过 15 秒中断。这两个值分开设很重要，代理慢和目标站慢是两码事。

## 常见报错与排查

写爬虫总会撞上几个经典错误。挨个说。

**`ProxyError: Cannot connect to proxy`**
代理地址或端口写错。或者代理服务挂了。先用 `curl` 或 `telnet` 测一下端口通不通。

**`407 Proxy Authentication Required`**
代理要认证但你没给密码。检查用户名密码是否正确，特殊字符是否做了 URL 编码。

**`SLError: HTTPSConnectionPool`**
代理服务证书问题。临时方案：`verify=False`。但生产环境别这么干，不安全。正确做法是检查代理服务商的 CA 证书配置。

**返回内容是代理商的错误页**
说明请求确实经过了代理，但代理拒绝了你的请求。常见原因：账户流量耗尽、目标站点在代理白名单之外、并发数超限。

**`requests.exceptions.ConnectionError: ('Connection aborted.', ...)`**
连接被对方主动关闭。可能触发了反爬。降低请求频率，或者换 IP 池。

## 选代理服务的几个判断维度

代码写得再漂亮，代理本身不行也是白搭。挑服务时盯紧这几个点：

1. **IP 池规模**：池子越大，被关联的概率越低。少于 1万 IP 的服务慎选。
2. **协议支持**：HTTP/HTTPS/SOCKS5 都要有。只支持 HTTP 的现在已经不够用。
3. **认证方式**：用户名密码 + IP 白名单两种都支持的更灵活。
4. **地理位置覆盖**：抓不同国家的站需要对应国家的 IP 出口。
5. **计费模式**：流量计费（按 GB）适合大体量任务，请求计费（按次数）适合精准抓取。
6. **稳定性 SLA**：正经服务商会给可用率承诺，比如 99.9% uptime。

## Webshare 全部套餐对比

Webshare 是目前性价比讨论度比较高的一家代理服务商，2018 年成立于美国旧金山，IP 池规模超过 8000 万，覆盖 195 个国家。免费套餐给 10 个代理 + 1GB 流量这个力度在行业里算诚意——大部分同类服务都是 7 天试用就掐断。

下面是当前可选的全部套餐，按使用场景分类：

### 共享代理与免费方案

| 套餐 | 配置 | 价格 | 适合场景 | 购买链接 |
| --- | --- | --- | --- | --- |
| Free Proxy | 10 个数据中心代理 + 1GB/月流量 | $0/月 | 学习、测试、个人小规模脚本 | [ 免费注册领取 10 个代理](https://bit.ly/web_share) |
| Proxy Server (按代理数) | 100 个共享数据中心代理 + 250GB 带宽 | 起步 $2.99/月 | 入门级爬虫、API 抓取 | [ 选择共享代理套餐](https://bit.ly/web_share) |

### 静态私有代理

| 套餐 | 配置 | 价格 | 适合场景 | 购买链接 |
| --- | --- | --- | --- | --- |
| Private Proxy | 独享数据中心 IP，不共享给他人 | 起步约 $0.50/proxy/月 | 账号管理、需要稳定固定 IP 的业务 | [ 立即开通独享私有代理](https://bit.ly/web_share) |
| Static Residential | 静态住宅 IP，长期不变 | 起步约 $1/proxy/月 | 社交媒体管理、电商运营 | [ 获取静态住宅代理](https://bit.ly/web_share) |

### 动态轮换代理

| 套餐 | 配置 | 价格 | 适合场景 | 购买链接 |
| --- | --- | --- | --- | --- |
| Rotating Datacenter | 数据中心 IP 池自动轮换 | 起步约 $6/月 起 | 大规模爬虫、SEO 监控 | [ 开启数据中心轮换代理](https://bit.ly/web_share) |
| Rotating Residential | 住宅 IP 池，按流量计费 | 起步 $7/GB | 反爬严格的目标站、价格抓取 | [ 立即体验住宅轮换代理](https://bit.ly/web_share) |
| ISP Proxies | 来自运营商的住宅级 IP | 按需定价 | 高匿名要求的商业场景 | [ 询价 ISP 代理方案](https://bit.ly/web_share) |

定价细节会随促销调整，下单前建议在套餐页面核对一下当时的实际价格。Webshare 提供 30 天以内未使用流量的退款承诺，相当于半个月的试错窗口——风险其实挺低。

> 站在用户视角看，每月 $2.99 起的入门价，分摊到 30 天每天不到 $0.10。一杯咖啡的钱换爬虫不被封，账其实很好算。

## Webshare 在 Python 中的完整实战代码

把前面所有概念组合起来，一个生产级的 Webshare 爬虫骨架长这样：

python
import requests
from requests.adapters import HTTPAdapter
from urllib3.util.retry import Retry
import time
import random

class WebshareScraper:
    def __init__(self, username, password):
        self.session = requests.Session()
        retry_strategy = Retry(
            total=3,
            backoff_factor=2,
            status_forcelist=[429, 500, 502, 503, 504],
        )
        adapter = HTTPAdapter(max_retries=retry_strategy)
        self.session.mount("http://", adapter)
        self.session.mount("https://", adapter)
        
        # 使用 Webshare 的 backconnect 端点，自动 IP 轮换
        proxy_url = f"http://{username}:{password}@p.webshare.io:80"
        self.session.proxies = {
            "http": proxy_url,
            "https": proxy_url,
        }
        self.session.headers.update({
            "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) "
                          "AppleWebKit/537.36 (KHTML, like Gecko) "
                          "Chrome/120.0.0 Safari/537.36"
        })
    
    def fetch(self, url):
        try:
            response = self.session.get(url, timeout=(5, 20))
            response.raise_for_status()
            return response.text
        except requests.exceptions.RequestException as e:
            print(f"[ERROR] {url}: {e}")
            return None
    
    def batch_fetch(self, urls, delay_range=(1, 3)):
        results = {}
        for url in urls:
            html = self.fetch(url)
            if html:
                results[url] = html
            time.sleep(random.uniform(*delay_range))
        return results

if __name__ == "__main__":
    scraper = WebshareScraper("your_username", "your_password")
    
    # 验证当前出口 IP
    print(scraper.fetch("https://httpbin.org/ip"))
    
    # 批量抓取
    targets = [
        "https://example.com/page1",
        "https://example.com/page2",
        "https://example.com/page3",
    ]
    data = scraper.batch_fetch(targets)
    print(f"成功抓取 {len(data)} 个页面")


这段代码可以直接拿去改，把 `your_username` 和 `your_password` 换成 Webshare 控制台拿到的凭据就行。控制台进去后，左侧菜单 "Proxy → List" 里能看到所有授权信息。

## FAQ：Python requests proxies 高频问题

**Q1: 为什么我设置了 proxies 但 requests 没走代理？**

通常是环境变量在捣乱。检查 `HTTP_PROXY` 和 `HTTPS_PROXY` 环境变量。`requests` 的 `proxies` 参数优先级低于环境变量。在代码里强制覆盖：

python
session.trust_env = False


**Q2: requests 能不能同时用多个代理做并发？**

不能在同一个 Session 里。但你可以用 `concurrent.futures.ThreadPoolExecutor`，每个线程独立 Session 配不同代理。或者直接换异步方案，比如 `aiohttp`，原生支持高并发场景。

**Q3: 免费代理能用吗？**

技术上能用，实际上别用。免费代理的问题：99% 都不稳定，可能记录你的请求内容，HTTPS 流量可能被中间人攻击。商业服务每月几美元的成本，比花一晚上 debug 免费代理划算。

**Q4: 数据中心代理和住宅代理怎么选？**

抓 API、抓非反爬目标——选数据中心代理，便宜量大。抓 Google、抓社交媒体、抓电商——必须用住宅代理，否则秒封。预算紧张就先用数据中心试，不行再升级。

**Q5: 一个 IP 一天能抓多少次不被封？**

没标准答案。取决于目标站的反爬强度、你的请求间隔、Header 配置、Cookie 策略。经验值：温和站 1000-5000 次/IP/日，严格站 50-200 次/IP/日。建议从低频开始测，再逐步上调。

**Q6: 用了代理还会被识别为爬虫吗？**

会。代理只解决 IP 维度的暴露。还有 User-Agent 指纹、TLS 指纹、JavaScript 行为、鼠标轨迹、Cookie 一致性等等。完整的反爬方案是代理 + 浏览器指纹模拟 + 行为模拟 + 速率控制的组合拳。

## 一句话总结

`python requests proxies` 这个参数本身不复杂，但要让爬虫真的稳跑，得把代理类型、认证方式、Session 复用、IP 轮换、错误重试这套组合拳全部打齐。代码层面的事一天能学会，剩下的是对工具的选择。

如果你正打算认真做点什么——不管是数据分析项目、商品监控、SEO 排名追踪还是自己的副业——一个靠谱的代理服务能省下大量时间。Webshare 的免费 10 个代理足够测试，付费起步价低，30 天退款也兜底，是低风险的入门选择。

[👉 现在领取 Webshare 免费代理并查看全部套餐](https://bit.ly/web_share)

写好爬虫的关键，从来不是代码有多花哨，而是基础设施有多稳。
