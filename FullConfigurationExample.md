## Example ##

### Proxy controller ###
Proxy controller manages proxy list and balances proxies during web pages crawling. You can implement your own proxy controller instead of default version.

```
        List<Proxy> proxies = new ArrayList<Proxy>();
        proxies.add(Proxy.NO_PROXY);
        // ... add your proxies ...
        // Creating an instance of proxy controller
        DefaultProxyController proxyController = new DefaultProxyController(proxies);
```

### Downloader ###
It downloads content from the specified URL and returns it as a Page object. This object contains a response code, content and some other essential data. You can implement your own Downloader or use default implementation. Default downloaders supports usage of the proxy controller. Also you can control it changing three properties:
  * allowedContentTypes - array of supported content types. If <a href='http://www.w3.org/Protocols/rfc2616/rfc2616-sec9.html'>HEAD-request</a> to the web page returns other type - "null" is returned instead of "Page" object. Default value is "text/html" only.
  * maxContentLength - maximum content length allowed for this downloader. Downloader checks "Content-Length" header of the "HEAD" response. If this constraint is violated - it returns "null". Default value is "0" - unlimited content length.
  * triesCount - downloader tries to download web page this number of times if response code is not HTTP\_OK or some of redirect codes

```
        // Creaing an instance of generic downloader
        DefaultDownloader genericDownloader = new DefaultDownloader();
        // Setting allowed content types
        genericDownloader.setAllowedContentTypes(new String[]{"text/html", "text/plain"});
        // Setting maximum content length limit
        genericDownloader.setMaxContentLength(500000);
        // Setting tries count limit
        genericDownloader.setTriesCount(3);
        // Setting proxy controller for this downloader
        genericDownloader.setProxyController(proxyController);

        // Creating an instance of custom downloader
        DefaultDownloader customDownloader = new DefaultDownloader();
        // Setting allowed content types to "null" - all content types are allowed for this downloader
        customDownloader.setAllowedContentTypes(null);
        // Setting maximum content length to 0 - unlimited
        customDownloader.setMaxContentLength(0);
        // Setting tries count to 0 - unlimited
        customDownloader.setTriesCount(0);
```

### DownloaderController ###
DownloaderController manages downloaders. Interface consists of a single method - "Downloader getDownloader(URL url)". Default implementation supports setting a "generic" downloader that will be used for any URL and any number of "custom" downloaders that will be used only for URLs with specific domains.

```
        // Creating downloader controller
        DefaultDownloaderController downloaderController = new DefaultDownloaderController();
        // You can avoid setting generic downloader - default downloader will be used
        // Setting generic downloader
        downloaderController.setGenericDownloader(genericDownloader);
        // Setting custom downloaders for domains "google.com" and "wikipedia.com"
        downloaderController.addCustomDownloader("google.com", customDownloader);
        downloaderController.addCustomDownloader("wikipedia.org", customDownloader);
```

### Parser and ParserController ###
By default parser extracts links and page title. But you can implement your own parser implementation or use ParserCallbacks to extend DefaultParser behaviour. Also you can set "custom" parsers to be used with chosen domains. **Be aware that extracting links and setting "links" property of the "Page" object is essential for crawler to continue working.**

```
        // Creating parser controller
        DefaultParserController defaultParserController = new DefaultParserController();
        // Setting generic parser class
        defaultParserController.setGenericParser(DefaultParser.class);
        // Setting custom parser classes for domains "google.com" and "wikipedia.com"
        defaultParserController.addCustomParser("google.com", CustomParser.class);
        defaultParserController.addCustomParser("wikipedia.org", CustomParser.class);
```
Example of the custom parser class:
```
    /**
     * Custom parser class
     */
    private static class CustomParser extends DefaultParser {

        @Override
        public void parse(Page page) {
            // Calls for super method that extracts links and title and populates
            // page.links and page.title properties
            super.parse(page);
            System.out.println("Inside custom parser");
        }
    }
```

### CrawlerConfiguration ###
Crawler configuration contains a set of properties controlling crawler's work:
  * maxHttpErrors - you can limit http errors count by domain. If this limit is violated for some site - crawler stop processing URLs of this site. By default no limits are set.
  * maxLevel - maximum crawling depth level. By default set to "0" - unlimited.
  * maxParallelRequests - maximum number of the parallel requests to a single site. By default set to "0" - unlimited
  * politenessPeriod - minimum period between two requests to a single site. By default "0" - no period.

```
        // Creating crawler configuration
        CrawlerConfiguration configuration = new CrawlerConfiguration();
        // Setting errors limit
        configuration.setMaxHttpErrors(HttpURLConnection.HTTP_BAD_GATEWAY, 10);
        // Setting maximum crawling depth level
        configuration.setMaxLevel(10);
        // Setting maximum parallel requests to a single domain limit
        configuration.setMaxParallelRequests(5);
        // Setting politeness period
        configuration.setPolitenessPeriod(0);
```

### DomainConstraints ###
You can override crawler constraints for chosen domains.

```
        // Creating specific constraints for google.com and wikipedia.org
        DomainConstraints domainConstraints = new DomainConstraints();
        // Maximum crawling depth level
        domainConstraints.setMaxLevel(100);
        // Maximum parallel requests count
        domainConstraints.setMaxParallelRequests(1);
        // Politeness period
        domainConstraints.setPolitenessPeriod(500);
        // Adding specific domain constraints to the crawler configuration
        configuration.setDomainConstraints("google.com", domainConstraints);
        configuration.setDomainConstraints("wikipedia.org", domainConstraints);
```

### Setting crawlers ###
Before crawling you should add desirable number of crawlers to your configuration. Default crawler implementation must be populated with DownloaderController and ParserController before start.

```
        // Adding crawlers
        for (int i = 0; i < 5; i++) {
            // Creating crawler object
            ExampleCrawler crawler = new ExampleCrawler();
            crawler.setDownloaderController(downloaderController);
            crawler.setParserController(defaultParserController);
            configuration.addCrawler(crawler);
        }
```

### CrawlerController ###
Default crawler controller manages crawling process, starts and stops, etc..

```
        // Creating crawler controller
        CrawlerController crawlerController = new CrawlerController(configuration);
        crawlerController.addSeed(new URL("http://google.com/"));
        crawlerController.addSeed(new URL("http://wikipedia.org/"));
        // Starting and joining crawler
        crawlerController.start();
        crawlerController.join();
```


### Full source code ###
```
package com.googlecode.flaxcrawler.examples;

import java.net.HttpURLConnection;
import java.net.MalformedURLException;
import java.net.Proxy;
import java.net.URL;
import java.util.ArrayList;
import java.util.List;
import com.googlecode.flaxcrawler.CrawlerConfiguration;
import com.googlecode.flaxcrawler.CrawlerController;
import com.googlecode.flaxcrawler.CrawlerException;
import com.googlecode.flaxcrawler.DefaultCrawler;
import com.googlecode.flaxcrawler.DomainConstraints;
import com.googlecode.flaxcrawler.download.DefaultDownloader;
import com.googlecode.flaxcrawler.download.DefaultDownloaderController;
import com.googlecode.flaxcrawler.download.DefaultProxyController;
import com.googlecode.flaxcrawler.model.CrawlerTask;
import com.googlecode.flaxcrawler.model.Page;
import com.googlecode.flaxcrawler.parse.DefaultParser;
import com.googlecode.flaxcrawler.parse.DefaultParserController;

/**
 *
 * @author ameshkov
 */
public class FullConfigurationExample {

    /**
     * @param args the command line arguments
     */
    public static void main(String[] args) throws MalformedURLException, CrawlerException {
        List<Proxy> proxies = new ArrayList<Proxy>();
        proxies.add(Proxy.NO_PROXY);
        // Creating an instance of proxy controller
        DefaultProxyController proxyController = new DefaultProxyController(proxies);

        // Creaing an instance of generic downloader
        DefaultDownloader genericDownloader = new DefaultDownloader();
        // Setting allowed content types
        genericDownloader.setAllowedContentTypes(new String[]{"text/html", "text/plain"});
        // Setting maximum content length
        genericDownloader.setMaxContentLength(500000);
        // Setting tries count
        genericDownloader.setTriesCount(3);
        // Setting proxy controller for this downloader
        genericDownloader.setProxyController(proxyController);

        // Creating an instance of custom downloader
        DefaultDownloader customDownloader = new DefaultDownloader();
        // Setting allowed content types to "null" - all content types are allowed for this downloader
        customDownloader.setAllowedContentTypes(null);
        // Setting maximum content length to 0 - unlimited
        customDownloader.setMaxContentLength(0);
        // Setting tries count to 0 - unlimited
        customDownloader.setTriesCount(0);

        // Creating downloader controller
        DefaultDownloaderController downloaderController = new DefaultDownloaderController();
        // Setting generic downloader
        downloaderController.setGenericDownloader(genericDownloader);
        // Setting custom downloaders for domains "google.com" and "wikipedia.com"
        downloaderController.addCustomDownloader("google.com", customDownloader);
        downloaderController.addCustomDownloader("wikipedia.org", customDownloader);

        // Creating parser controller
        DefaultParserController defaultParserController = new DefaultParserController();
        // Setting generic parser class
        defaultParserController.setGenericParser(DefaultParser.class);
        // Setting custom parser classes for domains "google.com" and "wikipedia.com"
        defaultParserController.addCustomParser("google.com", CustomParser.class);
        defaultParserController.addCustomParser("wikipedia.org", CustomParser.class);

        // Creating crawler configuration
        CrawlerConfiguration configuration = new CrawlerConfiguration();
        // Setting errors limit
        configuration.setMaxHttpErrors(HttpURLConnection.HTTP_BAD_GATEWAY, 10);
        // Setting maximum crawling depth level
        configuration.setMaxLevel(10);
        // Setting maximum parallel requests to a single domain limit
        configuration.setMaxParallelRequests(5);
        // Setting politeness period
        configuration.setPolitenessPeriod(0);

        // Creating specific constraints for google.com and wikipedia.org
        DomainConstraints domainConstraints = new DomainConstraints();
        // Maximum crawling depth level
        domainConstraints.setMaxLevel(100);
        // Maximum parallel requests count
        domainConstraints.setMaxParallelRequests(1);
        // Politeness period
        domainConstraints.setPolitenessPeriod(500);
        // Adding specific domain constraints to the crawler configuration
        configuration.setDomainConstraints("google.com", domainConstraints);
        configuration.setDomainConstraints("wikipedia.org", domainConstraints);


        // Adding crawler
        for (int i = 0; i < 5; i++) {
            // Creating crawler object
            ExampleCrawler crawler = new ExampleCrawler();
            crawler.setDownloaderController(downloaderController);
            crawler.setParserController(defaultParserController);
            configuration.addCrawler(crawler);
        }

        // Creating crawler controller
        CrawlerController crawlerController = new CrawlerController(configuration);
        crawlerController.addSeed(new URL("http://google.com/"));
        crawlerController.addSeed(new URL("http://wikipedia.org/"));
        // Starting and joining crawler
        crawlerController.start();
        crawlerController.join();
    }

    /**
     * Custom parser class
     */
    private static class CustomParser extends DefaultParser {

        @Override
        public void parse(Page page) {
            // Calls for super method that extracts links and title and populates
            // page.links and page.title properties
            super.parse(page);
            System.out.println("Inside custom parser");
        }
    }

    /**
     * Test crawler
     */
    private static class ExampleCrawler extends DefaultCrawler {

        /**
         * This method is called before each crawl attempt.
         * @param crawlerTask
         */
        @Override
        protected void beforeCrawl(CrawlerTask crawlerTask) {
            super.beforeCrawl(crawlerTask);
            System.out.println("Before crawling");
        }

        /**
         * This method is called after each crawl attempt.
         * Warning - it does not matter if it was unsuccessfull attempt or response was redirected.
         * So you should check response code before handling it.
         * @param crawlerTask
         * @param page
         */
        @Override
        protected void afterCrawl(CrawlerTask crawlerTask, Page page) {
            super.afterCrawl(crawlerTask, page);

            if (page == null) {
                System.out.println(crawlerTask.getUrl() + " violates crawler constraints (content-type or content-length or other)");
            } else if (page.getResponseCode() >= 300 && page.getResponseCode() < 400) {
                // If response is redirected - crawler schedulles new task with new url
                System.out.println("Response was redirected from " + crawlerTask.getUrl());
            } else if (page.getResponseCode() == HttpURLConnection.HTTP_OK) {
                // Printing url crawled
                System.out.println(crawlerTask.getUrl() + ". Found " + (page.getLinks() != null ? page.getLinks().size() : 0) + " links.");
            }
        }

        /**
         * You may check if you want to crawl next task
         * @param crawlerTask Task that is going to be crawled if you return {@code true}
         * @param parent parent.getUrl() page contain link to a crawlerTask.getUrl() or redirects to it
         * @return
         */
        @Override
        public boolean shouldCrawl(CrawlerTask crawlerTask, CrawlerTask parent) {
            // Default implementation returns true if crawlerTask.getDomainName() == parent.getDomainName()
            return super.shouldCrawl(crawlerTask, parent);
        }
    }
}
```