#  一、scrapy命令

**1.scrapy fetch **（额外参数 -- headers，--nolog）**url**

这个命令主要来显示爬虫爬取的过程（可获取header信息）

**2.scrapy runspider spider文件名**

通过该命令可以实现不依托Scrapy爬虫项目，直接运行爬虫文件

**3.scrapy shell url**

启动scrapy的交互终端

**4.scrapy version -v**

显示scrapy相关的详细版本信息

**5.scrapy view url**

下载该网页到本地

**6.scrapy crawl 爬虫名**

启动一个爬虫

```
Basicspider 与Crawlspider的区别
Crawlspider提供了一种自带的自动爬去网页的爬虫（默认模版中包含rules、LinkExtractor即链接提取器）
```



# 二、scrapy分布式

第一步、导入RedisSpider或者RedisCrawlSpider(from scrapy_redis.spiders import RedisSpider)

第二步、更换基类（换成RedisSpider或者RedisCrawlSpider）

第三步、设置redis_key

第四步、设置_init_ (动态域范围获取)与allowed_domains(规定静态域)相对

第五步、修改seeting.py

第五步详细（必选）：

```python
# 1.指定使用scrapy-redis的调度器
SCHEDULER = "scrapy_redis.scheduler.Scheduler"

# 2.指定使用scrapy-redis的去重
DUPEFILTER_CLASS = 'scrapy_redis.dupefilter.RFPDupeFilter'

# 3.指定排序爬取地址时使用的队列，
# 默认的 按优先级排序(Scrapy默认)，由sorted set实现的一种非FIFO、LIFO方式。
SCHEDULER_QUEUE_CLASS = 'scrapy_redis.queue.SpiderPriorityQueue'
# 可选的 按先进先出排序（FIFO）
# SCHEDULER_QUEUE_CLASS = 'scrapy_redis.queue.SpiderQueue'
# 可选的 按后进先出排序（LIFO）
# SCHEDULER_QUEUE_CLASS = 'scrapy_redis.queue.SpiderStack'

# 4.在redis中保持scrapy-redis用到的各个队列，从而允许暂停和暂停后恢复，也就是不清理redis queues
SCHEDULER_PERSIST = True

# 5.配置REDIS的相关参数
REDIS_HOST = '42.194.135.205'
REDIS_PORT = 6379
# REDIS_PASS = 'redisP@ssw0rd'
REDIS_PARAMS = {
    'password':'love530.',
    'db':3
}
REDIS_ENCODING = 'utf-8'

# （可选）6.默认情况下,RFPDupeFilter只记录第一个重复请求。将DUPEFILTER_DEBUG设置为True会记录所有重复的请求。
DUPEFILTER_DEBUG = True
```

第六步、启动分布式

1. 打开redis-server.exe
2. 打开redis-cli.exe
3. 找到爬虫文件下的spider
4. scrapy runspider spider文件名
5. 在redis-cli中输入：lpush redis-keyName(spider中定义的redis-key名字) URL（网页的链接）

注意：在其他机器上跑程序的时候要注意修改redis的host、修改该机器的防火墙、以及要配置redis的监听范围，使redis可远程连接。



# 三、知乎分布式核心源码

**zhihu.pro**

```python
import json
import re
import time
import scrapy
from scrapy_redis.spiders import RedisSpider
from zhihu.items import ZhihuItem


class ZhihuSpider(RedisSpider):  
    name = 'zhihupro'
    redis_key = 'zhihu:start_urls'
    
    def __init__(self, *args, **kwargs):
        domain = kwargs.pop('domain', '')
        self.allowed_domains = list(filter(None, domain.split(',')))
        super(ZhihuSpider, self).__init__(*args, **kwargs)

    def parse(self, response):
        first_layer = json.loads(response.text)
        is_end = first_layer['paging']['is_end']
        next_url = first_layer['paging']['next']
        for data in first_layer['data']:
            author_id = data['target']['id']
            type_comment = data['target']['type']
            if data['target']['comment_count'] != 0:
                url = f"https://www.zhihu.com/api/v4/{type_comment}s/{author_id}/root_comments?order=normal&limit=20&offset=0&status=open"
                yield scrapy.Request(url, callback=self.parse_comment)
        if not is_end:
            yield scrapy.Request(next_url, callback=self.parse)

    def parse_comment(self, response):
        # response.encoding = ('utf-8')
        second_layer = json.loads(response.text)
        gender_dic = {'1': '男', '-1': '匿名', '0': '女'}
        is_end = second_layer['paging']['is_end']
        next_url = second_layer['paging']['next']
        for data in second_layer['data']:
            if '<p>' in data['content']:
                data['content'] = re.findall('<p>(.+?)</p>', data['content'])[0]
            comment_item = ZhihuItem()
            comment_item['author'] = data['id']
            comment_item['comment'] = data['content']
            comment_item['comment_time'] = time.strftime('%Y%m%d%H%S', time.gmtime(data['created_time']))
            comment_item['gender'] = gender_dic[str(data['author']['member']['gender'])]

            yield comment_item
        if not is_end:
            yield scrapy.Request(next_url, callback=self.rest_comment)

    def rest_comment(self, response):
        gender_dic = {'1': '男', '-1': '匿名', '0': '女'}
        second_layer = json.loads(response.text)
        is_end = second_layer['paging']['is_end']
        next_url = second_layer['paging']['next']
        for data in second_layer['data']:
            if '<p>' in data['content']:
                data['content'] = re.findall('<p>(.+?)</p>', data['content'])[0]
            comment_item = ZhihuItem()
            comment_item['author'] = data['id']
            comment_item['comment'] = data['content']
            comment_item['comment_time'] = time.strftime('%Y%m%d%H%S', time.gmtime(data['created_time']))
            comment_item['gender'] = gender_dic[str(data['author']['member']['gender'])]
            yield comment_item
        if not is_end:
            yield scrapy.Request(next_url, callback=self.rest_comment)

```

**Setting.py**

```python
# Scrapy settings for zhihu project
#
# For simplicity, this file contains only settings considered important or
# commonly used. You can find more settings consulting the documentation:
#
#     https://docs.scrapy.org/en/latest/topics/settings.html
#     https://docs.scrapy.org/en/latest/topics/downloader-middleware.html
#     https://docs.scrapy.org/en/latest/topics/spider-middleware.html

BOT_NAME = 'zhihu'

SPIDER_MODULES = ['zhihu.spiders']
NEWSPIDER_MODULE = 'zhihu.spiders'


# Crawl responsibly by identifying yourself (and your website) on the user-agent
#USER_AGENT = 'zhihu (+http://www.yourdomain.com)'

# Obey robots.txt rules
ROBOTSTXT_OBEY = False
ITEM_PIPELINES = {
    'scrapy_redis.pipelines.RedisPipeline': 400,
    # 'zhihu_comment.pipelines.ZhihuCommentPipeline': 500,
}

# 指定使用scrapy-redis的调度器
SCHEDULER = "scrapy_redis.scheduler.Scheduler"

# 指定使用scrapy-redis的去重
DUPEFILTER_CLASS = 'scrapy_redis.dupefilter.RFPDupeFilter'

# 指定排序爬取地址时使用的队列，
# 默认的 按优先级排序(Scrapy默认)，由sorted set实现的一种非FIFO、LIFO方式。
SCHEDULER_QUEUE_CLASS = 'scrapy_redis.queue.SpiderPriorityQueue'
# 可选的 按先进先出排序（FIFO）
# SCHEDULER_QUEUE_CLASS = 'scrapy_redis.queue.SpiderQueue'
# 可选的 按后进先出排序（LIFO）
# SCHEDULER_QUEUE_CLASS = 'scrapy_redis.queue.SpiderStack'

# 在redis中保持scrapy-redis用到的各个队列，从而允许暂停和暂停后恢复，也就是不清理redis queues
SCHEDULER_PERSIST = True
REDIS_HOST = '42.194.135.205'
REDIS_PORT = 6379
# REDIS_PASS = 'redisP@ssw0rd'
REDIS_PARAMS = {
    'password':'love530.',
    'db':3
}
REDIS_ENCODING = 'utf-8'

# LOG等级
LOG_LEVEL = 'DEBUG'

# 默认情况下,RFPDupeFilter只记录第一个重复请求。将DUPEFILTER_DEBUG设置为True会记录所有重复的请求。
DUPEFILTER_DEBUG = True

# 覆盖默认请求头，可以自己编写Downloader Middlewares设置代理和UserAgent
# DEFAULT_REQUEST_HEADERS = {
#     'Accept': 'text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8',
#     'Accept-Language': 'zh-CN,zh;q=0.8',
#     'Connection': 'keep-alive',
#     'Accept-Encoding': 'gzip, deflate, sdch'
# }
USER_AGENT = 'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_13_6) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/70.0.3538.102 Safari/537.36'



# Configure maximum concurrent requests performed by Scrapy (default: 16)
#CONCURRENT_REQUESTS = 32

# Configure a delay for requests for the same website (default: 0)
# See https://docs.scrapy.org/en/latest/topics/settings.html#download-delay
# See also autothrottle settings and docs
#DOWNLOAD_DELAY = 3
# The download delay setting will honor only one of:
#CONCURRENT_REQUESTS_PER_DOMAIN = 16
#CONCURRENT_REQUESTS_PER_IP = 16

# Disable cookies (enabled by default)
#COOKIES_ENABLED = False

# Disable Telnet Console (enabled by default)
#TELNETCONSOLE_ENABLED = False

# Override the default request headers:
#DEFAULT_REQUEST_HEADERS = {
#   'Accept': 'text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8',
#   'Accept-Language': 'en',
#}

# Enable or disable spider middlewares
# See https://docs.scrapy.org/en/latest/topics/spider-middleware.html
#SPIDER_MIDDLEWARES = {
#    'zhihu.middlewares.ZhihuSpiderMiddleware': 543,
#}

# Enable or disable downloader middlewares
# See https://docs.scrapy.org/en/latest/topics/downloader-middleware.html
#DOWNLOADER_MIDDLEWARES = {
#    'zhihu.middlewares.ZhihuDownloaderMiddleware': 543,
#}



# 自动停止
 MYEXT_ENABLED = True
 MYEXT_ITEMCOUNT = 3   # 12s
 EXTENSIONS = {
    'zhihu.extensions.RedisSpiderSmartIdleClosedExensions': 540,
}

DOWNLOADER_MIDDLEWARES = {
    # 'zhihu_comment.middlewares.ZhihuCommentDownloaderMiddleware': 543,
    # 'zhihu_comment.middlewares.UserAgentDownloadMiddleware': 150,
    'zhihu.middlewares.CookiesMiddleware': 100,
    # 'zhihu_comment.middlewares.ProxyMiddleware':400,
}
```

