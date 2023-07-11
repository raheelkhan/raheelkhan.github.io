---
title: Bypassing Cloudfare DDoS in Scrapy
date: 2018-04-19
categories:
---

![Cloudfare CDN](/img/cf-blog-logo-crop.png)

While doing web scraping I came across with a website who has implemented Cloudfare DDoS (Distributed Denial of Service) protection.

DDoS is an attempt where a target host is attacked by multiple sources commonly to bring it down. [Wikipedia](https://en.wikipedia.org/wiki/Denial-of-service_attack).

Cloudfare, apart from being a usual CDN also provides security features to the websites. One of which is the DDoS protection.

Cloudfare sits between a client and the actual server as a proxy and block any bot to hit the actual website. Basically it first check if the javascript is enabled on the client side, It then runs a challenge and set two cookies `cf_clearance` and `__cfduid`. Once you have the cookie set, every subsequent request on that domain should contain these two cookies in order to browse through the site.

Having said the above restrictions, It is impossible to bypass the security with the default [Scrapy](https://scrapy.org/) spiders.

After a little research I found a very nice Python [module](https://github.com/Anorov/cloudflare-scrape) that does all the heavy lifting for me.

If you read their documentation, It says that we need a node server running as I mentioned earlier that cloudfare checks the existence of Javascript.

In your server where scrapy is running, go ahead and install NodeJs. Chances are you already have it one in your server which you can verify by hitting `node -v`.

We also need two Python modules

1. `pip install cfscrape -U`
2. `pip install fake-useragent`
   ​
   Once done with the installations, Open up your scrapy spider.

```python
import cfscrape
from fake_useragent import UserAgent

class CloudfareSpider(scrapy.Spider):
    name = 'cf_spider'
    start_urls = ['https://somewebsite.com/']
    user_agent = UserAgent().random;

    def start_requests(self):
    """
    :param self:
    """
    try:
        token, agent = cfscrape.get_tokens(CloudfareSpider.start_urls[0],CloudfareSpider.user_agent)
        for url in CloudfareSpider.start_urls:
            yield scrapy.Request(
                url=url,
                cookies=token,
                headers={'User-Agent': agent}
            )

    def parse(self, response):
    """
    :param response:
    """
        pass
```

What we have done is simply import the two python modules. In the `start_request` method we’ve used the `get_tokens` method.

At this point you should have a token and agent in hand. Note that the agent is the one we have send as a header while getting the token. This is important as the subsequent request to that domain should have the same token otherwise cloudfare will reject the request with 503.

If you debug the token you should see a dictionary of two cookies

```
{'cf_clearance': '40feb3864d7df598afacbb75affc898bfc0d85c5-1524155358-1800', '__cfduid': 'd1724971b58a6df50e6b3c72072b55aae1524155350'
```

We are almost done. Next we will call our Scrapy request with the cookies and sending the same user agent in headers.

If you have a middleware dealing with user agents, that should be okay as the spider will take precedence over it.

{: .box-warning}
**Disclaimer**: Cloudfare may change this mechanism any time and the code won’t work as expected.
