SSRF补

# php——curl  #

工具cURL可以使用URL的语法模拟浏览器来传输数据，因为它是模拟浏览器，因此它同样支持多种协议，FTP, FTPS, HTTP, HTTPS, GOPHER, TELNET, DICT, FILE 以及 LDAP等协议都可以很好的支持，包括一些：HTTPS认证，HTTP POST方法，HTTP PUT方法，FTP上传，keyberos认证，HTTP上传，代理服务器，cookies，用户名/密码认证,下载文件断点续传，上传文件断点续传，http代理服务器管道，甚至它还支持IPv6，scoket5代理服务器，通过http代理服务器上传文件
 
## Curl支持的协议 ##   

dict file ftp ftps gopher http https imap imaps pop3 pop3s rtsp smb smbs smtp smtps telnet tftp  

	- ftp、ftps （FTP爆破）
	- tftp（UDP协议扩展）
	- imap/imaps/pop3/pop3s/smtp/smtps（爆破邮件用户名密码）
	- rtsp
	- smb/smbs （连接SMB）
	- telnet - 连接SSH/Telnet
	- http、https - 内网服务探测
	- 网络服务探测
	- ShellShock命令执行
	- JBOSS远程Invoker war命令执行
	- Java调试接口命令执行
	- axis2-admin部署Server命令执行
	- Jenkins Scripts接口命令执行
	- Confluence SSRF
	- Struts2一堆命令执行
	- counchdb WEB API远程命令执行
	- mongodb SSRF
	- docker API远程命令执行
	- php_fpm/fastcgi 命令执行
	- tomcat命令执行
	- Elasticsearch引擎Groovy脚本命令执行
	- WebDav PUT上传任意文件
	- WebSphere Admin可部署war间接命令执行
	- Apache Hadoop远程命令执行
	- zentoPMS远程命令执行
	- HFS远程命令执行
	- glassfish任意文件读取和war文件部署间接命令执行

php Curl函数 无防护简单模拟ssrf功能代码

	<?php
		$url = $_GET['url'];
    	$ch = curl_init();
    	curl_setopt($ch, CURLOPT_URL, $url);
    	curl_setopt($ch, CURLOPT_HEADER, false);
    	curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
    	curl_setopt($ch, CURLOPT_SSL_VERIFYPEER, false);
    	curl_setopt($ch, CURLOPT_USERAGENT, 'Mozilla/5.0 (Windows NT 6.1) AppleWebKit/537.11 (KHTML, like Gecko) Chrome/23.0.1271.1 Safari/537.11');
        // 允许302跳转
        curl_setopt($ch, CURLOPT_FOLLOWLOCATION, true);
        $res = curl_exec($ch);
        // 设置content-type
        header('Content-Type: image/png');
        curl_close($ch) ;
        //返回响应
        echo $res;
	?>

当cURL允许follow redirect时  当URL存在临时(302)或永久(301)跳转时，则继续请求跳转后的URL 那么我们可以通过HTTP(S)的链接302跳转到gopher协议上。
CURL具体可用函数 

	curl_copy_handle() 	复制一个cURL句柄和它的所有选项。
	curl_escape() 	返回转义字符串，对给定的字符串进行URL编码。
	curl_pause() 	暂停及恢复连接。
	curl_reset() 	重置libcurl的会话句柄的所有选项。
	curl_share_close() 	关闭cURL共享句柄。
	curl_share_init() 	初始化cURL共享句柄。
	curl_share_setopt() 	设置一个共享句柄的	cURL传输选项。
	curl_setopt_array() 	为cURL传输会话批量设置选项。
	curl_setopt() 	设置一个cURL传输选项。
关键在于 curl_setopt()的设置 可以完成众多功能 下面例举部分我们可能用到或者需要注意的设置

	CURLOPT_AUTOREFERER	当根据Location:重定向时，自动设置header中的Referer:信息。	 
	CURLOPT_BINARYTRANSFER	在启用	CURLOPT_RETURNTRANSFER的时候，返回原生的（Raw）输出。	 
	CURLOPT_COOKIESESSION	启用时curl会仅仅传递一个session cookie，忽略其他的cookie，默认状况下cURL会将所有的cookie返回给服务端。session cookie是指那些用来判断服务器端的session是否有效而存在的cookie。	
	CURLOPT_DNS_USE_GLOBAL_CACHE	启用时会启用一个全局的DNS缓存，此项为线程安全的，并且默认启用。
	CURLOPT_FTPAPPEND	启用时追加写入文件而不是覆盖它。
	CURLOPT_RETURNTRANSFER	将curl_exec()获取的信息以文件流的形式返回，而不是直接输出。	 
	CURLOPT_UPLOAD	启用后允许文件上传。
	CURLOPT_INFILESIZE	设定上传文件的大小限制，字节(byte)为单位。
	CURLOPT_MAXCONNECTS	允许的最大连接数量，超过是会通过CURLOPT_CLOSEPOLICY决定应该停止哪些连接。	 
	CURLOPT_MAXREDIRS	指定最多的HTTP重定向的数量，这个选项是和CURLOPT_FOLLOWLOCATION一起使用的。	 
	CURLOPT_PORT	用来指定连接端口。	
	CURLOPT_REDIR_PROTOCOLS	CURLPROTO_*中的位域值。如果被启用，位域值将会限制传输线程在	CURLOPT_FOLLOWLOCATION开启时跟随某个重定向时可使用的协议。这将使你对重定向时限制传输线程使用被允许的协议子集默认libcurl将会允许除FILE和SCP之外的全部协议。这个和7.19.4预发布版本种无条件地跟随所有支持的协议有一些不同。关于协议常量，请参照CURLOPT_PROTOCOLS。	在	cURL 7.19.4中被加入。
	CURLOPT_POSTFIELDS	全部数据使用HTTP协议中的"POST"操作来发送。要发送文件，在文件名前面加上@前缀并使用完整路径。这个参数可以通过urlencoded后的字符串类似'para1=val1&para2=val2&...'或使用一个以字段名为键值，字段数据为值的数组。如果value是一个数组，Content-Type头将会被设置成multipart/form-data。	 
	CURLOPT_PROXY	HTTP代理通道。	
功能例举 文件上传：

	<?php
 	$url = "www.xxx.com/upload.php";

        $post_data = array (
          "attachment" => "@D:/1.jpg"
        );

        //初始化cURL会话
        $ch = curl_init();

        //设置请求的url
        curl_setopt($ch, CURLOPT_URL, $url);
        curl_setopt($ch, CURLOPT_RETURNTRANSFER, 1);

        //设置为post请求类型
        curl_setopt($ch, CURLOPT_POST, 1);

        //设置具体的post数据
        curl_setopt($ch, CURLOPT_POSTFIELDS, $post_data);

        $response = curl_exec($ch);
        curl_close($ch);

        print_r($response);
?>

# Gopher 协议 #

	-用于分布式文档搜索与检索（攻击多用）
	-基于 TCP/IP 协议 
	-默认端口为 70 
	-无状态协议
	gopher://<host>:<port>/<gopher-path> 

gopher-path的格式如下所示 

	①<gophertype><selector>
	②<gophertype><selector>%09<search>
	③<gophertype><selector>%09<search>%09<gopher+_string>

利用此协议可以攻击内网的 FTP、Telnet、Redis、Memcache，也可以进行 GET、POST 请求。借此扩大攻击面


局限：

	-大部分 PHP 并不会开启 fopen 的 gopher wrapper file_get_contents 的 gopher 协议
	-URLencode file_get_contents 关于 Gopher 的 302 跳转有 bug，导致利用失败
	-PHP 的 curl 默认follow 302 不跳转 
	-curl/libcurl 7.43 上 gopher 协议存在 bug（%00 截断），经测试 7.49 可用 

Gopher协议 过去比较流行 通用 现在被逐渐替代

详细 英文站 太多太杂 
[https://en.wikipedia.org/wiki/Gopher_(protocol)](https://en.wikipedia.org/wiki/Gopher_(protocol))




## 文件操作函数（php）： ##

	返回规范化的绝对路径 realpathf()
	复制文件 copy()
	返回目录中的可用空间 disk_free_space()
	返回目录的磁盘总大小 disk_total_space()
	将一个字符串写入文件 file_put_contents()
	取得上次访问时间 fileatime()
	给出文件的信息 stat()
	通过已打开的文件指针取得文件信息 fstat
	取得文件类型 filetype()
	判断是否是一个目录 is_dir()
	进行文件锁定 flock()
	判断文件是否能过HTPP POST 上传的 is_uploaded_file()
	将上传的文件移动到新位置 move_uploaded_file()
	
 
# 目录遍历/文件包含测试  #
 
黑盒：
输入测试  关键在于 查找可控的文件操作变量，广撒网。其次测试查看特殊的变量内容例;函数,文件操作命令,字符,路径,外网地址(没试过传参外网地址 包含或者ssrf上地址可能会出现变量后跟外网地址)等 是否可能触发特殊页面，最后扩展名测试   
ps
上次系统分隔符 补：
url编码:

	%2e%2e%2f   ../
	%2e%2e%5c 或 %252e%252e%252c  ..\
unicode/utf-8（仅适用于超长utf-8系统）:

	..%c0%af ../
	..%c1%9c ..\
灰盒：
相比黑盒可以搜索输入变量 函数、方法。文件系统操作

# 文件删除 #
运用 删除站中的安全文件 安全锁之类的 以此为后续攻击打开方便之门
例：
phpyun任意文件删除漏洞
phpyun4.1beta版本
　　`$re=file_put_contents(DATA_PATH."upload/user/".date('Ymd')."/".$new_file, _decode($uimage));` 
两个参数均是来自于$_POST['uimage']，而且还经过了_decode的反编码，就可以无视WAF了
可控变量查找


# php反序列（对象注入）漏洞： #

php里面的值，我们都可以使用函数serialize()来返回一个包含字节流的字符串来表示。序列化一个对象将会保存对象的所有变量，但是不会保存对象的方法，只会保存类的名字。unserialize()函数能够重新把字符串变回php原来的值。

漏洞的根源
在于unserialize()函数的参数可控。如果反序列化对象中存在魔术方法，而且魔术方法中的代码有能够被我们控制，漏洞就这样产生了，根据不同的代码可以导致各种攻击，如代码注入、SQL注入、目录遍历等等。
例：

    <?php
    class A{
    	var $a = "test";
    	function __destruct(){
    		$fp = fopen("D:\\phpStudy\\WWW\\hello.php","w");
    		fputs($fp,$this->a);
    		fclose($fp);
    	}
    }
    $test = $_POST['test'];
    $test_unser = unserialize($test);
    ?>
    
php的魔法方法destruct（当一个对象被销毁时被自动调用的析构方法）自动执行时 unserialize参数可控
## 常见形式 ##
- 一是将传来的序列化数据直接unserilize，造成魔幻函数的执行。这种情况在一般的应用中依然屡见不鲜。
- 二是PHP Session 序列化及反序列化处理器设置使用不当会带来的安全隐患。

如果脚本中设置的序列化处理器与php.ini设置的不同，或者两个脚本注册session使用的序列化处理器不同，那么就会出现安全问题。

	处理器  对应的存储格式
	php键名 ＋ 竖线 ＋ 经过 serialize() 函数反序列处理的值
	php_binary 键名的长度对应的 ASCII 字符 ＋ 键名 ＋ 经过 serialize() 函数反序列处理的值
	php_serialize (php>=5.5.4) 经过 serialize() 函数反序列处理的数组
注入\\’|\\’来使得后面任意伪造的序列化字符串，以此来利用反序列化漏洞。 利用php的魔法方法

在实际代码审计中，这类漏洞是较为常见的。但是利用起来往往比较复杂。一是要跟踪到达漏洞点去，提供满足前期必要条件；二是将包含的类如何进行 有效利用。而一些大型的利用，pop链往往能达到5个以上。
ps序列化对象包含攻击者控制的对象值

CVE-2016-7124

当序列化字符串中表示对象属性个数的值大于真实的属性个数时会跳过__wakeup的执行

    <html>
    <head>
    <title>PHP反序列化</title>
    </head>
    <body>
    <?php
    class A{
    	var $a = "test";
    	function __destruct(){
    		$fp = fopen("D:\\phpStudy\\WWW\\hello.php","w");
    		fputs($fp,$this->a);
    		fclose($fp);
    	}
    	function __wakeup()
    	{
    	foreach(get_object_vars($this) as $k => $v) {
    		$this->$k = null;
    	}
    	echo "Waking up...\n";
    	}
    }
    $test = $_POST['test'];
    $test_unser = unserialize($test);
    ?>
    </body>
    </html>
wecenter奇葩的利用点 微信认证的服务号：
[https://www.leavesongs.com/PENETRATION/wecenter-unserialize-arbitrary-sql-execute.html](https://www.leavesongs.com/PENETRATION/wecenter-unserialize-arbitrary-sql-execute.html)

php反序列基础：[http://www.91ri.org/15925.html](http://www.91ri.org/15925.html)

杂七杂八
sql语句有这么一个特性，和C语言里的atoi函数类似，当需要转换参数是数字但传入的字符串不是纯数字时就会截取最前面是数字的位数进行转换，而不会出错。
