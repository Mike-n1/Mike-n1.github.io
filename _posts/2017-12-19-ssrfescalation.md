---
layout: post
title: P4 to P2 - The story of one blind SSRF
permalink: SSRF_P4toP2
---

<div style="text-align:center"><img src = "/public/CardTrick.jpg"/></div>
Hello everyone. This is my second blog post where I want to tell how I managed to get Blind Local SSRF (P2) instead of External SSRF (P4). Unfortunately, I can't disclose the vulnerable application, so instead of some screenshots I will be using cute kittens or funny gifs.



<!--more-->

Initially, I found functional of uploading receipts that were later converted in PDF format. I noticed that the file could be uploaded in the "html" format. 

The first thing I tried was uploading "html" file, which included <img> tag with link to my ordinary sniffer. This was done in order to understand whether a vulnerable script parsed our file.

<div style="text-align:center">
    <a href="/public/SnifferScreen.png" data-fancybox data-toolbar="false" data-small-btn="true">
        <img src = "/public/SnifferScreen_s.png"/>
    </a>
</div>

After that "html" files were uploaded with different javascript payloads (with "script" tag, with different events, etc.). I tried to read local files using js, or execute an alert box, or dynamically changed the source code of web page. Sadly, none of these options didn't worked.

If you find yourself in similar situation, but you'll be more lucky and you'll have an opportunity to execute a javascript code, here are some examples that can help you:
* [http://www.noob.ninja/2017/11/local-file-read-via-xss-in-dynamically.html](http://www.noob.ninja/2017/11/local-file-read-via-xss-in-dynamically.html)
* [https://buer.haus/2017/06/29/escalating-xss-in-phantomjs-image-rendering-to-ssrflocal-file-read/](https://buer.haus/2017/06/29/escalating-xss-in-phantomjs-image-rendering-to-ssrflocal-file-read/)

It is very important to note that at the stage of trying to read a local file it was noted that if the substring ```file://``` appears in the uploaded html file, it won\`t be processed. However, if we send ```file	:///asd``` string (here is a tab character, not a whitespace), our file will be processed in ordinary way.

Next idea that came to my mind was reading local files using iframe, object and so on, but tag <iframe> was completely blocked and html didn't processed in any ways, and tag <object> simply didn't read the local file.

<div style="text-align:center"><img src = "/public/Sadly.png"/></div>

In contemporary browsers, you can read local files using object, iframe or embed tags. For example:
* ```<object width="400" height="400" data="file://c:/windows/win.ini"></object>```
* ```<iframe src="file:///C:/Windows/win.ini" width="400" height="400">```
* ```<embed src="file://c:/windows/win.ini" width="400" height="400">```

<a href="/public/ObjectRead.png" data-fancybox data-toolbar="false" data-small-btn="true">
    ![alt text](/public/ObjectRead_s.png "Read win.ini")
</a>

None of the methods I tried didn't worked, but it seemed to me that I could achieve more than just a simple External SSRF. Then I remembered about [@BugBountyHQ](https://twitter.com/BugBountyHQ) [tweet](https://twitter.com/BugBountyHQ/status/868242771617792000) and decided to try to include existed local image via img tag. But, what kind of picture on a 100 percent exists on the web server? Remembering about type of the user-agent ```Mozilla/4.0 (compatible; MSIE 8.0; Windows NT 6.1; Trident/4.0)``` which probably means that it was header of the Internet Explorer. I found the image that exist on the web server by default with this browser - ```C:\Program Files\Internet Explorer\images\bing.ico``` and tried to include it (don't forget about using 'file://').

Finally, I've got a payload: ```<img src="file	:///C:\Program Files\Internet Explorer\images\bing.ico">```

<a href="/public/BingIMage.png" data-fancybox data-toolbar="false" data-small-btn="true">
![alt text](/public/BingIMage_s.png "Bing Image")
</a>

In the result, we successfully bypassed the filter and changed the priority from P4 to P2.

<div style="text-align:center"><img src = "/public/Woow.gif"/></div>