---
published: true
---

**Why your next go-to browser instrumentation tool should be CDP**

Whether it's to "copp" the latest Air Jordans, get your partner the latest Prada bag, or even book an appointment on public platforms, the use of bots has significantly increased and will continue to do so. Some of the most famous tools used to create them are web app testing frameworks like Puppeteer, Playwright, or Selenium. Those tools can offer you a variety of important features that go all the way to Iframe manipulation and mouse movement.  
In practice, though, these tools have certain drawbacks that can make the task execution harder and sometimes even impossible, especially with the need of the Webdriver component that adds more complexity to the pipeline and provides signals that make the bot easily detectable.  
The approach explained in this article is for developers that are looking for a more lightweight browser instrumentation tool that will enable them to achieve their goal with less computational effort. Similar work has been done in [Nikolai Tschacher's blog](https://incolumitas.com/2021/05/20/avoid-puppeteer-and-playwright-for-scraping/).


## Chrome DevoTool Protocol

The approach consists of directly manipulating chrome directly through [CDP](https://developer.chrome.com/docs/devtools/) who is actually used under the hood in many higher-level APIs like Puppeteer. In our example, we will use the JavaScript library [chrome-remote-interface]( https://github.com/cyrus-and/chrome-remote-interface ) which is an interface that helps to instrument Chrome (or any other suitable implementation) by providing a simple abstraction of commands and notifications using a straightforward JavaScript API. As an example, we will build a simple bot that scraps the first page of [Hacker News](https://news.ycombinator.com/) and crawls through all the articles and scrap all the links contained in them, we will use it in a headful mode so the process can be visualized in realtime. We will have to take the following steps:

1. launch google chrome and connect to it through port 9222
2. load hacker news main page, and request all the article links
3. iterate through all the articles and scrap all the links in them

## Example


### Setup

We will first install chrome-remote-interface library and connect to a chrome browser via port 9222

```bash
npm install chrome-remote-interface
```

It can run in headless mode with the use the `--headless` option.

```bash
google-chrome --remote-debugging-port=9222
```


## The code will look like this 


```javascript

const CDP = require('chrome-remote-interface');
const fs = require('fs');

CDP(async (client) => {
    const startDate = new Date().getTime();
    try 
    {
        const {Input, Page, Runtime} = client;

        await Page.navigate({url: 'https://news.ycombinator.com/front'});

        const tree = await Page.getResourceTree();
        
        // fetching the article links 
        let result = await Runtime.evaluate({returnByValue: true,
            expression: "Array.from(document.getElementsByClassName('titlelink')).map(x => x.href)"});
        // the method returns a JSON file
        let hackerNewsLinks = result.result['value']

       
        var i;
        // create stream where to append files 
      	var logger = fs.createWriteStream('log.txt', {
            flags: 'a' 
          })
        // iterating over all links
        for (i = 0; i < hackerNewsLinks.length; ++i) {
            console.log("accessing : " + hackerNewsLinks[i])
          	// navigate to article
            await Page.navigate({url:hackerNewsLinks[i]});
          	// requesting all links from the article
          	let linkresult = await Runtime.evaluate({returnByValue: true,
                expression: "Array.from(document.getElementsByTagName('a')).map(x => x.href)"});
            let links = linkresult.result['value'].toString();
            logger.write(hackerNewsLinks[i] + links + '\n');
        }

    
    } catch (err) {
        console.error(err);
    } finally {
        await client.close();
    }
  	console.log(`Time elapsed ${Math.round((new Date().getTime() - startDate) / 1000)} s`);
}).on('error', (err) => {
    console.error(err);
});

```
Advantages: 
* lightweight: complexity-wise, it is faster than libraries like Puppeteer.

drawbacks: 
* slowness: like every other browser-based bot, it still is considered a slow bot. The task on headless mode takes on average 11s.

# Conclusion

In this post, we showcased a lightweight browser-based crawler that is often overlooked when discussing this bots category. We used it to scrap [Hacker News](https://news.ycombinator.com/). Although the syntax is pretty straightforward, the task was time-consuming in comparison to browserless bots. The next post will show how you can easily parallelize your crawling process by spawning multiple browsers and using multiple pages.
