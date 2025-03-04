---
title: "Hacking FirstBloodHackers v1.0"
draft: false
tags: ["web_pentesting"]
categories: ["Web Security"]
weight: 1
---
![Description of Image](/images/first-blood-v1/First-Blood-v1.0-Poster.png)

In this article I will reveal techniques I used when pentesting a web application i will try it on `FirstBloodHackers v1.0` it's a hospital website which is a hackevent of bugbountyhunter.com website

I opened burpsuite and opened the given url 
```php
https://f22246f644be-l33ch.a.firstbloodhackers.com/
```
I added the URL to scope to see only the files related to this url
![Image](/images/first-blood-v1/burp-scope.png)
Press here 
![Image](/images/first-blood-v1/press-here-with-mouse.png)
check this box
![Image](/images/first-blood-v1/checkbox1.png)
Choose `Discover content` from `Engagement tools` to visit all the endpoints on the website 
![Image](/images/first-blood-v1/content-discover1.png)

As the content discovery is working I need to know the technology the website is working with. One way is by using quickhits.txt file from seclists 
```php
ffuf -u https://f22246f644be-l33ch.a.firstbloodhackers.com/FUZZ -w /usr/share/seclists/Discovery/Web-Content/quickhits.txt 
```
![Image](/images/first-blood-v1/fuff1.png)
From the above about the backend server supports php files

Another way to find the technology at the backend is from the `Error Default Page` type any unknown words to get to the error page. This is the error of Nginx Server
![Image](/images/first-blood-v1/nginx-default-page.png)

After some time i will close the running Content Discovery
![Image](/images/first-blood-v1/close-content-discovery.png) 

From the sitemap you find that the available extensions are `html & php`
![Image](/images/first-blood-v1/sitemap-1.png) 

So I will fuzz with these extensions `html & php`
```php
ffuf -u https://f22246f644be-l33ch.a.firstbloodhackers.com/FUZZ -w /usr/share/seclists/Discovery/Web-Content/raft-small-words-lowercase.txt -e .html,.php
```
![Image](/images/first-blood-v1/fuzzing1.png) 
I want to see all the URLs available at the site if the fuzzing missed ones `Right Click - View page source`
![Image](/images/first-blood-v1/pagesource-1.png) 
search for `href` 
![Image](/images/first-blood-v1/href1.png) 
These are the files I got from Fuzzing & href & content discovery
- login.php
- yourappointments.php
- doctors.html
- about.html
- hackerback.html
- book-appointment.html

Let's begin with login.php file 
<br>
I usually view the `page source` to find any comments that can reveal sensitive data
<br>
But Nothing found here
![Image](/images/first-blood-v1/login1.png) 
### Looking for SQLi vulnerability
I put `single quote` in the `Username & Password` fields to see if i got any SQL Errors, but i didn't get any SQL Errors
![Image](/images/first-blood-v1/sqli-test1.png) 
so maybe it is vulnerable to Time-Based SQLi or Boolean-Based SQLi
I intercepted the request with burpsuite and saved it to file name login-sql-test.txt. I put asterisk at the parameters i want Sqlmap to test the payloads in which is `username & password` parameters 
```jfx
POST /login.php?action=login HTTP/1.1
Host: f22246f644be-l33ch.a.firstbloodhackers.com
Content-Length: 27
Cache-Control: max-age=0
Sec-Ch-Ua: "Not A(Brand";v="8", "Chromium";v="132"
Sec-Ch-Ua-Mobile: ?0
Sec-Ch-Ua-Platform: "Linux"
Accept-Language: en-US,en;q=0.9
Origin: https://f22246f644be-l33ch.a.firstbloodhackers.com
Content-Type: application/x-www-form-urlencoded
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/132.0.0.0 Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: navigate
Sec-Fetch-User: ?1
Sec-Fetch-Dest: document
Referer: https://f22246f644be-l33ch.a.firstbloodhackers.com/login.php?action=login
Accept-Encoding: gzip, deflate, br
Priority: u=0, i
Connection: keep-alive

username=test*&password=te*
```
and will test it with sqlmap
```php
sqlmap -r login-sql-test.txt --random-agent --batch
```
Sqlmap didn't get any output

#### Next I want to discover all the parameters that `login.php` file accepts
**First** i will use `Param Miner` Extension
![Image](/images/first-blood-v1/param-miner-guess1.png) 
![Image](/images/first-blood-v1/param-miner-guess2.png) 
Then go to `Extensions - Param Miner - Output` as you can see Param Miner has discovered two parameters `ref, goto`
![Image](/images/first-blood-v1/paramminer-login-op1.png) 
**Second** i will use [Arjun](https://github.com/s0md3v/Arjun) tool
```php
arjun -u https://f22246f644be-l33ch.a.firstbloodhackers.com/login.php --stable
```
It also found these two parameters `ref, goto`
![Image](/images/first-blood-v1/arjun-login-op1.png) 

Let's test the first parameter `ref` 
<br>
open the url
```php
https://52e0fa57b5ac-l33ch.a.firstbloodhackers.com/login.php?ref=test123
```
Intercept the request with BurpSuite and send it to Repeater tab, check the `Auto-scroll to match when text changes` box to search for the what you type in the search bar `test123` on every request
![Image](/images/first-blood-v1/Repeater-checkbox1.png) 
At the response the input of `ref` parameter is inside <span style="color: red;">&lt;a href=&quot;</span><span style="color: blue;">{User-Input}</span><span style="color: red;">&quot;&gt;</span>
<br>
I changed the searchbox to `Return to previous page` to scroll to the line that href exist on every request 
![Image](/images/first-blood-v1/Repeater-checkbox2.png) 
Since the user input is reflected at the response let's test for **XSS** first. I will try to fix the tag by closing the tags and write our payload **`<script>alert(1)</script>`**
<br>
<span style="color: red;">&lt;a href=&quot;</span>
<span style="color: blue;">&quot;&lt;script&gt;alert(1)&lt;/script&gt;&lt;&quot;</span>
<span style="color: red;">&quot;&gt;</span>
As you see our payload is encoded
![Image](/images/first-blood-v1/xss-encoded-input.png)
Let's try different payload <span style="color: blue;">javascript:alert(1)</span>. From the about we got this <span style="color: red;">:b|1)</span> so `javascript` & `alert` keywords are filtered
![Image](/images/first-blood-v1/xss-login-php-ref-javasript1.png)

Let's try <span style="color: blue;">javascript:confirm(1)</span> as you see from the output the word `confirm` is passed but the parenthesis`()` is also filtered
![Image](/images/first-blood-v1/xss-login-php-ref-confirm1.png)

Let's try **confirm&#96;1&#96;** . It passed
![Image](/images/first-blood-v1/xss-login-php-ref-confirm2.png)

lets try now to pass `javascript` keyword. If you tried a different combination of keywords `javascript` you will find the problem is with the `a` character after `jav`
![Image](/images/first-blood-v1/xss-login-php-ref-javasript2.png)
I will try to bypass that by using the the newline url-encoding <span style="color: blue;">%0a</span> so the full payload will be <span style="color: blue;">jav%0aascript:confirm&#96;1&#96;</span>
![Image](/images/first-blood-v1/xss-login-php-ref-javasript3.png)
Let's try the payload at the browser now 
![Image](/images/first-blood-v1/xss-login-php-ref-javasript4.png)
If you click now on `Return to previous page` the XSS is triggered
![Image](/images/first-blood-v1/xss-login-php-ref-javasript5.png)

Let's test now the second parameter `goto` open the url
```php
https://aca66d1c48fe-l33ch.a.firstbloodhackers.com/login.php?goto=test123
```
Intercept the request with BurpSuite and send it to Repeater tab, check the `Auto-scroll to match when text changes` box to search for the what you type in the search bar `test123` on every request
![Image](/images/first-blood-v1/Repeater-goto-checkbox1.png) 
At the response the input of `goto` parameter is inside 
<br>
<span style="color: red;">&lt;input name="goto" value="</span></span><span style="color: blue;">{User-Input}</span><span style="color: red;">" type="hidden"></span>
<br>
I changed the searchbox to `type="hidden"` to scroll to the line that `value=` exist on every request 
![Image](/images/first-blood-v1/Repeater-checkbox3.png) 
Since the user input is reflected at the response lets test for **XSS** first. I will try to fix the tag by closing the tags and write our payload **`<script>alert(1)</script>`**
<br>
<span style="color: red;">&lt;input name="goto" value="</span>
<span style="color: blue;">&quot;&gt;&lt;script&gt;alert(1)&lt;/script&gt;&lt;"</span>
<span style="color: red;"> " type="hidden"></span>
<br>
As you see our payload is filtered so maybe the words **`<script>`** & **`alert`** is backlisted
![Image](/images/first-blood-v1/xss-filtered-input1.png) 
lets begin with word `alert` and change it to `confirm`. It's passed but the parenthesis`()` is also filtered
![Image](/images/first-blood-v1/xss-filtered-input2.png) 
lets try **confirm&#96;1&#96;** . it passed
![Image](/images/first-blood-v1/xss-filtered-input3.png) 

Let's try now to pass word `<script>`. if the filter remove the whole keyword `<script>` i tried this `<scr<script>ipt>` so it the filter remove the word `<script>` the remaining `<scr` & `ipt>` will be combined and will do the same with `</script>` will replace it with `</scr</script>ipt>`
<br>
It works 
![Image](/images/first-blood-v1/xss-login-php-goto-1.png) 
Lets test it at the browser now
```php
https://aca66d1c48fe-l33ch.a.firstbloodhackers.com/login.php?goto="><scr<script>ipt>confirm`1`</scr</script>ipt><"
```
![Image](/images/first-blood-v1/xss-login-php-goto-2.png) 

For all the vulnerabilities you can see the disclosed reports here
https://www.bugbountyhunter.com/hackevents/firstblood

### References
<https://0xdf.gitlab.io/cheatsheets/404>