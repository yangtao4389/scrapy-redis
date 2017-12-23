基于scrapy爬虫框架搭建的redis分布式组件
框架图参考:
http://note.youdao.com/noteshare?id=1ed21d3fc3da7c0a614a9063d72103dc&sub=FBF09061B91C4EDEBB047710781207FC


由图可知:
它基于scrapy框架,在scheduler调度器和Item pipeline数据管道中添加redis,
使redis来存储调度器的request队列,以及'判断该request队列是否已经实现过'而生成的指纹集合
还负责存储item管道处理后的数据

所以,如果想使用该组件:
    1.需要重写scheduler调度器,
    2.对请求过的request进行过滤去重
    3.需要pipelines的Item被redis来过滤,在setting中设置权重

而我们调用该scrapy_redis库后,只需要在设置中配置:
    1.重写scheduler调度器的组件:
        # Enables scheduling storing requests queue in redis.
        SCHEDULER = "scrapy_redis.scheduler.Scheduler"
    2.过滤request的组件:
        # Ensure all spiders share same duplicates filter through redis.
        DUPEFILTER_CLASS = "scrapy_redis.dupefilter.RFPDupeFilter"
    3.队列中的内容是否持久保持,为false则使在程序关闭的时候清除redis
        # Don't cleanup redis queues, allows to pause/resume crawls.
        SCHEDULER_PERSIST = True
    4.存储Item的组件
        # Store scraped item in redis for post-processing.
          ITEM_PIPELINES = {
              # 'example.pipelines.ExamplePipeline': 300,
              'scrapy_redis.pipelines.RedisPipeline': 400
          }
    5.redis配置
        # Specify the full Redis URL for connecting (optional).
        # If set, this takes precedence over the REDIS_HOST and REDIS_PORT settings.
        #REDIS_URL = 'redis://user:pass@hostname:9001'
        REDIS_URL = 'redis://127.0.0.1:6379'

如果需要分布式处理
    1.需要继承scrapy_redis中的spiders来实现爬虫,可以参考Feeding a Spider from Redis案例

具体配置可以参考官方配置
Usage
-----

Use the following settings in your project:

.. code-block:: python

  # Enables scheduling storing requests queue in redis.
  SCHEDULER = "scrapy_redis.scheduler.Scheduler"

  # Ensure all spiders share same duplicates filter through redis.
  DUPEFILTER_CLASS = "scrapy_redis.dupefilter.RFPDupeFilter"

  # Default requests serializer is pickle, but it can be changed to any module
  # with loads and dumps functions. Note that pickle is not compatible between
  # python versions.
  # Caveat: In python 3.x, the serializer must return strings keys and support
  # bytes as values. Because of this reason the json or msgpack module will not
  # work by default. In python 2.x there is no such issue and you can use
  # 'json' or 'msgpack' as serializers.
  #SCHEDULER_SERIALIZER = "scrapy_redis.picklecompat"

  # Don't cleanup redis queues, allows to pause/resume crawls.
  SCHEDULER_PERSIST = True

  # Schedule requests using a priority queue. (default)
  #SCHEDULER_QUEUE_CLASS = 'scrapy_redis.queue.PriorityQueue'

  # Alternative queues.
  #SCHEDULER_QUEUE_CLASS = 'scrapy_redis.queue.FifoQueue'
  #SCHEDULER_QUEUE_CLASS = 'scrapy_redis.queue.LifoQueue'

  # Max idle time to prevent the spider from being closed when distributed crawling.
  # This only works if queue class is SpiderQueue or SpiderStack,
  # and may also block the same time when your spider start at the first time (because the queue is empty).
  #SCHEDULER_IDLE_BEFORE_CLOSE = 10

  # Store scraped item in redis for post-processing.
  ITEM_PIPELINES = {
      'scrapy_redis.pipelines.RedisPipeline': 300
  }

  # The item pipeline serializes and stores the items in this redis key.
  #REDIS_ITEMS_KEY = '%(spider)s:items'

  # The items serializer is by default ScrapyJSONEncoder. You can use any
  # importable path to a callable object.
  #REDIS_ITEMS_SERIALIZER = 'json.dumps'

  # Specify the host and port to use when connecting to Redis (optional).
  REDIS_HOST = 'localhost'
  REDIS_PORT = 6379

  # Specify the full Redis URL for connecting (optional).
  # If set, this takes precedence over the REDIS_HOST and REDIS_PORT settings.
  #REDIS_URL = 'redis://user:pass@hostname:9001'

  # Custom redis client parameters (i.e.: socket timeout, etc.)
  #REDIS_PARAMS  = {}
  # Use custom redis client class.
  #REDIS_PARAMS['redis_cls'] = 'myproject.RedisClient'

  # If True, it uses redis' ``SPOP`` operation. You have to use the ``SADD``
  # command to add URLs to the redis queue. This could be useful if you
  # want to avoid duplicates in your start urls list and the order of
  # processing does not matter.
  #REDIS_START_URLS_AS_SET = False

  # Default start urls key for RedisSpider and RedisCrawlSpider.
  #REDIS_START_URLS_KEY = '%(name)s:start_urls'

  # Use other encoding than utf-8 for redis.
  #REDIS_ENCODING = 'latin1'

.. note::

  Version 0.3 changed the requests serialization from ``marshal`` to ``cPickle``,
  therefore persisted requests using version 0.2 will not able to work on 0.3.



Features
--------
去重:request去重/数据item去重
持久化:暂停过后再次重启,会从暂停的地方再次执行
分布式:可以联合多台电脑一起来实现爬虫,只需要将redis配置成一样,并且使用向redis添加start_url的方式就可以,
    此时的爬虫就需要继承scrapy_redis中的spiders来实现



* Distributed crawling/scraping

    You can start multiple spider instances that share a single redis queue.
    Best suitable for broad multi-domain crawls.

* Distributed post-processing

    Scraped items gets pushed into a redis queued meaning that you can start as
    many as needed post-processing processes sharing the items queue.

* Scrapy plug-and-play components

    Scheduler + Duplication Filter, Item Pipeline, Base Spiders.

.. note:: This features cover the basic case of distributing the workload across multiple workers. If you need more features like URL expiration, advanced URL prioritization, etc., we suggest you to take a look at the `Frontera`_ project.




Running the example project
---------------------------

两个案例,
第一个实现暂停过后重启,依旧从暂停位置处理
第二个实现分布式爬虫,需要先开启爬虫,然后向redis添加键为开始的url
具体的案例实现可以参考example-project,里面有相应解释
This example illustrates how to share a spider's requests queue
across multiple spider instances, highly suitable for broad crawls.

1. Setup scrapy_redis package in your PYTHONPATH

2. Run the crawler for first time then stop it::

    $ cd example-project
    $ scrapy crawl dmoz
    ... [dmoz] ...
    ^C

3. Run the crawler again to resume stopped crawling::

    $ scrapy crawl dmoz
    ... [dmoz] DEBUG: Resuming crawl (9019 requests scheduled)

4. Start one or more additional scrapy crawlers::

    $ scrapy crawl dmoz
    ... [dmoz] DEBUG: Resuming crawl (8712 requests scheduled)

5. Start one or more post-processing workers::

    $ python process_items.py dmoz:items -v
    ...
    Processing: Kilani Giftware (http://www.dmoz.org/Computers/Shopping/Gifts/)
    Processing: NinjaGizmos.com (http://www.dmoz.org/Computers/Shopping/Gifts/)
    ...


Feeding a Spider from Redis
---------------------------

The class `scrapy_redis.spiders.RedisSpider` enables a spider to read the
urls from redis. The urls in the redis queue will be processed one
after another, if the first request yields more requests, the spider
will process those requests before fetching another url from redis.

For example, create a file `myspider.py` with the code below:

.. code-block:: python

    from scrapy_redis.spiders import RedisSpider

    class MySpider(RedisSpider):
        name = 'myspider'

        def parse(self, response):
            # do stuff
            pass


Then:

1. run the spider::

    scrapy runspider myspider.py

2. push urls to redis::

    redis-cli lpush myspider:start_urls http://google.com


.. note::

    These spiders rely on the spider idle signal to fetch start urls, hence it
    may have a few seconds of delay between the time you push a new url and the
    spider starts crawling it.
