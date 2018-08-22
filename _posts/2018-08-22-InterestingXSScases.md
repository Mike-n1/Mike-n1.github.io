---
layout: post
title: Unusual cases of reflected XSS
permalink: Unusual_XSS
---

<div style="text-align:center"><img src = "/public/XSS_title.png"/></div>
Hello, *friend*!

Today, I would like to talk about some cases, connected with XSS attack, which I faced with during web-application security analysis (in private bug bounty).


<!--more-->

## First case - XSS in burp, but not in the browsers
Analyzing requests in Burp, I paid attention to one of them; the parameter appeared on the web page without filtering.

<a href="/public/1_burp_reflect.png" data-fancybox data-toolbar="false" data-small-btn="true">
    ![alt text](/public/1_burp_reflect.png "Reflect")
</a>

I was surprised, because I thought that many researchers had already analyzed this web-application. I look through old Bug Bounty programs or well-known applications with aiming to find interesting vulnerabilities or to work with unknown for me software (servers, programming languages, frameworks, etc.)

Because the GET parameter was vulnerable and there was no protection against CSRF, I copied the URL and tried to follow to this address in browser

<a href="/public/1_browser_NOreflect.png" data-fancybox data-toolbar="false" data-small-btn="true">
    ![alt text](/public/1_browser_NOreflect.png "Browser without XSS")
</a>

Nothing happened and at first I didn't understand why. I checked response from server and didn't find any expected reflection there. I don't know why, but the first thing that came into my mind was that user's data somehow handled in javascript, which are connected earlier on the web page. Fortunately, I didn't really believe in a such case and noticed that the old one and the new request were different in the «Referer» header.

<a href="/public/1_burp_refferer.png" data-fancybox data-toolbar="false" data-small-btn="true">
    ![alt text](/public/1_burp_refferer.png "Referer in burp request")
</a>

I tried to remove this header in the old request and sent it again. Really, reflection disappeared. Hm-m, I tried to set Referrer to «https://subdomain.vulnerable.com» and nothing new happened. After that I recovered the old Referrer (https://subdomain.vulnerable.com/PublishTools/) and again saw the reflection that I was needed. After some attempts, one idea came to my mind that logic on the server may be something like that:
1. Check the domain: «subdomain.vulnerable» must be include in it
2. Check the URL path: it must exactly be «/PublishTools/»

If both points are true, the referrer will be valid and vulnerable parameter will be displayed on the page.

I checked this idea and I was absolutely right. Having added the A record ```subdomain.vulnerable.mysite.com A 1.2.3.4``` from my domain name registrar (1.2.3.4 – my server ip), I created the «PublishTools» folder and the «index.html» file on my web-server. The source code of the «index.html» file:

```html
<a href="https://subdomain.vulnerable.com/ToolForPagePreview?preview_url=<svg/onload=alert(1)>" id ="a_id">Click</a>
<script>
document.getElementById("a_id").click();
</script>

```

And now, after opening the [http://**subdomain.vulnerable.mysite.com**/PublishTools/](#) page, an automatic click on the [https://**subdomain.vulnerable.com**/ToolForPagePreview?preview_url=<svg/onload=alert(1)>](#) link will be performed and you will open this page with the following Referrer header: ```Referrer: http://subdomain.vulnerable.mysite.com/PublishTools/```. It was enough for bypassing the defense of this web-application. As a result, the expected alert-box was received and javascript code was executed in the context of the vulnerable domain ***subdomain.vulnerable.com***.

<a href="/public/1_browser_alert.png" data-fancybox data-toolbar="false" data-small-btn="true">
    ![alt text](/public/1_browser_alert.png "Browser with alert box")
</a>

---

## Second case - XSS, but not for everyone

>You can try to solve the same [XSS challenge](http://infosec.gearhostpreview.com/9p9ch/) before reading the continuation of the blog. Important: try your XSS vector in different browsers for a full vector that will work for any user.

Analyzing requests on Burp, I found explicit XSS again. Application hasn't custom headers, but it has one strange value of GET parameter similar with CSRF-token. After removing all Cookies, I sent request again through «Repeater» tab and saw the reflection.

<a href="/public/2_burp_reflect.png" data-fancybox data-toolbar="false" data-small-btn="true">
    ![alt text](/public/2_burp_reflect.png "Request in Burp")
</a>

OK, copied the URL and sent request in browser for checking reflection and making screenshot for the report. I saw the needed alert box and started to make a report about dull reflected XSS. I don't know why, but during creating the report one idea came into my mind and I checked the reflection in Chrome browser. When I opened URL with a payload I didn't see alert box, even reflection of «name» parameter disappeared.

So, I faced with something interesting and probably this suspicious behavior connected with ```cdp``` parameter. At the beginning, I started to search the value of ```cdp``` parameter in Burp history and found it on the one page. It was clear that its value depends on which User-Agent we are using when opening the page. I changed User-Agent to string «a» and tried to find the shown on the page hash in Google. This hash wasn't found, this means that we cannot calculate it by themselves, because of the salted data.

For solving this problem, we need to change the victim's User-Agent to the one whose hash we know, or somehow figure out the victim's hash. First case is impossible, on my opinion, so I decided to create the attack scheme like that: 

1. Victim opens the previously created php script
2. Script takes User-Agent header from victim's request and send the new request with this User-Agent to the web page where the hash value written
3. Grab the token from web page for next request
4. And finally, redirect victim to the page with malicious payload and correct token which was generated for his User-Agent

<a href="/public/schema.png" data-fancybox data-toolbar="false" data-small-btn="true">
    ![alt text](/public/schema.png "Attack scheme")
</a>

Now, after opening [http://mysite.com/script.php](#), we will be redirected to the malicious web page and saw execution of javascript code in the context of vulnerable domain.
