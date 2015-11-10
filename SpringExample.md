## Spring configuration example ##

Look at FullConfigurationExample before reading this.

```
<!--
Proxy controller definition. You can set a map, containing proxy addresses. Otherwise proxy will be omitted.
-->
    <bean id="proxyController" class="com.googlecode.flaxcrawler.download.DefaultProxyController">
<!--
        <constructor-arg index="0">
            <map>
                <entry key="ip address" value="port" />
            </map>
        </constructor-arg>
-->
    </bean>

    <bean id="genericDownloader" class="com.googlecode.flaxcrawler.download.DefaultDownloader">
        <property name="maxContentLength" value="500000" />
        <property name="triesCount" value="3" />
        <property name="allowedContentTypes">
            <list>
                <value>text/html</value>
                <value>text/plain</value>
            </list>
        </property>
        <property name="proxyController" ref="proxyController" />
    </bean>

    <bean id="customDownloader" class="com.googlecode.flaxcrawler.download.DefaultDownloader">
        <property name="maxContentLength" value="0" />
        <property name="triesCount" value="1" />
<!-- Any content type is allowed -->
        <property name="allowedContentTypes"><null /></property>
        <property name="proxyController" ref="proxyController" />
    </bean>

    <bean id="downloaderController" class="com.googlecode.flaxcrawler.download.DefaultDownloaderController">
        <constructor-arg index="0" ref="genericDownloader" />
        <constructor-arg index="1">
            <map>
                <entry key="google.com" value-ref="customDownloader" />
                <entry key="wikipedia.org value-ref="customDownloader" />
            </map>
        </constructor-arg>
    </bean>

    <bean id="parserController" class="com.googlecode.flaxcrawler.parse.DefaultParserController">
    </bean>

    <bean id="customConstraints" class="com.googlecode.flaxcrawler.DomainConstraints">
        <property name="maxLevel" value="5" />
        <property name="maxParallelRequests" value="1" />
        <property name="politenessPeriod" value="200" />
    </bean>

    <bean id="crawlerConfiguration" class="com.googlecode.flaxcrawler.CrawlerConfiguration">
        <property name="maxLevel" value="100" />
        <property name="maxParallelRequests" value="5" />
        <property name="politenessPeriod" value="0" />
        <property name="domainConstraints">
            <map>
                <entry key="google.com" value-ref="customConstraints" />
                <entry key="wikipedia.org" value-ref="customConstraints" />
            </map>
        </property>
    </bean>

<!--
Attention - scope is prototype, not singleton. You can use singleton implementation, but be sure that your implementation is thread-safe.
-->
    <bean id="crawler" class="YOUR CUSTOM CRAWLER IMPLEMENTATION" scope="prototype">
        <property name="downloaderController" ref="downloaderController" />
        <property name="parserController" ref="parserController" />
    </bean>
```

## Using spring ##

```
        ApplicationContext applicationContext = new ClassPathXmlApplicationContext(new String[]{"spring/applicationContext-*.xml"});
        CrawlerConfiguration configuration = (CrawlerConfiguration) applicationContext.getBean("crawlerConfiguration");

        // Adding 10 crawlers
        for (int i = 0; i < 10; i++) {
            FileHostCrawler fileHostCrawler = (FileHostCrawler) applicationContext.getBean("crawler");
            fileHostCrawler.setCrawlerId(i);
            configuration.addCrawler(fileHostCrawler);
        }

        CrawlerController crawlerController = new CrawlerController(configuration);
        crawlerController.start();
        crawlerController.join();
```