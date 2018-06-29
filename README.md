## 减少HTTP请求
+ 10%~20%的最终用于响应时间花在接受所请求的HTML文档上，剩余80%~90%的时间是花在所引用的组件（图片、脚本、样式表）的HTTP请求上。

+ 改善响应时间最简单的方法就是通过减少组件的数量，来减少HTTP请求数。

### CSS Sprites
将多个图片合并到一个单独的图片中，利用CSS的background-position属性，将HTML元素放在背景图片中期望的位置上。

### Inline Images (内联图片)
使用data: [\<mediatype\>][;base64],<data>的模式在页面中包含图片无需任何额外的HTTP请求。
```
实例：<img src="data:image/gif;base64,R0lGODlhAwADAIABAL6+vv///yH5BAEAAAEALAAAAAADAAMAAAIDjA9WADs=" />

缺陷：
    1、IE7及其以下版本的浏览器不支持。
    2、数据大小上有限制。
    3、data:url 是内联在页面上的，跨页面使用时不会被缓存。（可以通过将data:url写在css样式表中，
       作为背景图片来实现缓存，因为css样式表会被缓存）
       
目前webpack已经可以通过url-loader来实现此项功能。
```

### Combined Scripts and Stylesheets （合并脚本和样式表）
1. 使用外部脚本、样式表对性能更有利，而行内脚本会阻塞HTML的渲染。
2. 保持javascript的模块化，在生成过程中从一组特定的模块生成一个目标文件。（目前webpack实现可以实现模块化开发、打包）

## 使用内容分发网络（CDN）
用户的带宽虽在增长，但是页面响应速度仍然受到用户与服务器物理位置的远近影响。

### CDN
内容分发网络是一组分布在多个不同地理位置的Web服务器，以便更加有效的向用户发布内容。

### 国内常用CDN厂商
- 阿里云
- 腾讯云
- 百度云
- 网宿科技
- 金山云

## 通过 HTTP Header 设置缓存静态资源

### Expires
```
设置 Expires: Sat, 27 May 2028 06:15:41 GMT 之后，浏览器在后续页面浏览中会使用缓存版本，HTTP请求会减少一个。

在nginx.conf中配置Expires：
    location ~* \.(jpg|jpeg|gif|bmp|png){
        expires 1d; #缓存1天
    }

缺陷：
    1、要求服务器、客户端时钟严格同步。
    2、需要经常检查过期时间。
    3、一旦过期之后，需要在服务器配置中提供一个新的日期。

```

### Cache-Control: max-age=*****
```
1、Cache-Control: max-age = 10000000，表示从组件被请求开始过去的秒数少于max-age，浏览器就
   继续使用缓存版本（HTTP 1.1版本以上）。
2、使用Cache-Control可以消除Expires的限制。
3、Cache-Control: max-age=***的优先级高于Expires。
    
基于Nginx的 ngx_http_headers_module 模块设置Cacht-Control：
location ~ .*\.(css|js|swf|php|htm|html )$ {
    add_header Cache-Control max-age=3600;
}
```
### 增加了缓存时间的文件，再次更新方案
1. 直接修改所有的链接，全新的请求就会从源服务器上下载最新版本的文件。-----此方案无法在大型、复杂的项目中使用，容易出错。
2. 为所有的组件的文件名使用变量，进行映射，在需要的文件中引用变量；修改时，只需要修改变量对应的内容（一般是在文件名后面增加版本号）即可。
```
实例:
    // map.js 导出一个映射
    export default map = {
        pic: 'https://www.xxx.com/assets/images/logo_0.0.1.png',
        jsFile: 'https//www.xxx.com/assets/js/util_0.0.1.js'
    }
    
    // index.js 应用映射导入文件
    import map from 'map'
    <script src=map.jsFile></script>
```

## 压缩组件
- 使用gzip编码来压缩HTTP响应包，可以减少网络响应时间。
- HTTP 1.1开始，服务端可以通过 Accept-Encoding 来识别浏览器对压缩类型的支持。Accept-Encoding: gzip, deflate
- 客户端可以通过 Content-Encoding 来识别服务器当前使用的压缩类型。Content-Encoding: gzip

```
在nginx.conf中开启gzip：

    # 开启gzip
    gzip on;
    
    # 启用gzip压缩的最小文件，小于设置值的文件将不会压缩
    gzip_min_length 1k;
    
    # gzip 压缩级别，1-10，数字越大压缩的越好，也越占用CPU时间，后面会有详细说明
    gzip_comp_level 2;
    
    # 进行压缩的文件类型。javascript有多种形式。其中的值可以在 mime.types 文件中找到。
    gzip_types text/plain application/javascript application/x-javascript text/css application/xml 
               text/javascript application/x-httpd-php image/jpeg image/gif 
               image/png font/ttf font/otf image/svg+xml;
    
    # 是否在http header中添加Vary: Accept-Encoding，建议开启-避免使用代理导致的问题（见下）
    gzip_vary on;
    
    # 禁用IE 6 gzip
    gzip_disable "MSIE [1-6]\.";
    
```

```
如果浏览器使用了代理来发送请求时，会出现一些复杂的情况：

    1、第一个请求是不支持gzip的浏览器发起的，此时代理缓存为空，代理会将请求转发到源服务器；
       此时代理获取到的响应是未经压缩的，并被代理缓存起来；
       第二个相同的请求是支持gzip的浏览器发起的，此时代理已有缓存，该缓存会作为响应发送到前端；
       这样第二个请求失去了gzip的压缩机会。
       
    2、第一个请求是支持gzip的浏览器发起的，此时代理缓存为空，代理会将请求转发到源服务器；
       此时代理获取到的响应是经gzip压缩的，并被代理缓存起来；
       第二个相同的请求是不支持gzip的浏览器发起的，此时代理已有缓存，该缓存会作为响应发送到前端；
       这样第二个请求拿到的是经过gzip压缩的响应。
       
解决方案：
    在源服务器的响应中加上Vary: Accept-Encoding，通知代理服务器基于Accept-Encoding来改变缓存的响应。
    代理会为有指定Accept-Encoding缓存一份，为没有指定该字段的缓存一份。
```

## CSS放在顶部，Javascript放在底部
同一时间针对同一域名下的请求
<table>
    <tr>
        <th>浏览器</th>
        <th>并行下载数</th>
    </tr>
    <tr>
        <td>Chrome</td>
        <td>6</td>
    </tr>
    <tr>
        <td>FirFox</td>
        <td>6</td>
    </tr>
    <tr>
        <td>IE8+</td>
        <td>6</td>
    </tr>
    <tr>
        <td>IE7</td>
        <td>4</td>
    </tr>
    <tr>
        <td>IE6</td>
        <td>2</td>
    </tr>
</table>

可以通过使用CNAME（DNS别名）将组件分别放在多个域名下。增加并行加载数

## 使用外部的CSS以及Javascript
- 外部的Javascript、CSS能够被浏览器缓存。
- 外部的Javascript、CSS可以被多个页面重用。

## 减少DNS查找

### DNS解析过程
1. 查找浏览器缓存。
```
浏览器会检查缓存中有没有这个域名对应的解析过的IP地址，如果缓存中有，
这个解析过程就将结束。浏览器缓存域名也是有限制的，不仅浏览器缓存大小有限制，
而且缓存的时间也有限制，通常情况下为几分钟到几小时不等。这个缓存时间太长和太短都不好，
如果缓存时间太长，一旦域名被解析到的IP有变化，会导致被客户端缓存的域名无法解析到变化后的IP地址，
以致该域名不能正常解析，这段时间内有可能会有一部分用户无法访问网站。
如果时间设置太短，会导致用户每次访问网站都要重新解析一次域名。
```
2. 查找系统缓存。
```
如果用户的浏览器缓存中没有，浏览器会查找操作系统缓存中是否有这个域名对应的DNS解析结果。其实操作系统也会有一个域名解析的过程，
在Windows中可以通过C:\Windows\System32\drivers\etc\hosts文件来设置，你可以将任何域名解析到任何能够访问的IP地址。
如果你在这里指定了一个域名对应的IP地址，那么浏览器会首先使用这个IP地址。例如，我们在测试时可以将一个域名解析到一台测试服务器上，
这样不用修改任何代码就能测试到单独服务器上的代码的业务逻辑是否正确。正是因为有这种本地DNS解析的规程，
所以黑客就有可能通过修改你的域名解析来把特定的域名解析到它指定的IP地址上，导致这些域名被劫持。
```
3. 查找路由器缓存。
```
如果系统缓存中也找不到，那么查询请求就会发向路由器，它一般会有自己的DNS缓存。
```
4. 查找ISP DNS 缓存。
```
运气实在不好，就只能查询ISP DNS 缓存服务器了。在我们的网络配置中都会有"DNS服务器地址"这一项，
操作系统会把这个域名发送给这里设置的DNS，也就是本地区的域名服务器，通常是提供给你接入互联网的应用提供商。
这个专门的域名解析服务器性能都会很好，它们一般都会缓存域名解析结果，当然缓存时间是受域名的失效时间控制的，
一般缓存空间不是影响域名失效的主要因素。大约80%的域名解析都到这里就已经完成了，所以ISP DNS主要承担了域名的解析工作。
```
5. 递归查询。
```
最无奈的情况发生了, 在前面都没有办法命中的DNS缓存的情况下,
(1)本地 DNS服务器即将该请求转发到互联网上的根域（即一个完整域名最后面的那个点，通常省略不写）。
(2)根域将所要查询域名中的顶级域（假设要查询ke.qq.com，该域名的顶级域就是com）的服务器IP地址返回到本地DNS。
(3) 本地DNS根据返回的IP地址，再向顶级域（就是com域）发送请求。(4) com域服务器再将域名中的二级域（即ke.qq.com中的qq）的IP地址返回给本地DNS。
(5) 本地DNS再向二级域发送请求进行查询。
(6) 之后不断重复这样的过程，直到本地DNS服务器得到最终的查询结果，并返回到主机。这时候主机才能通过域名访问该网站。
```

### 影响DNS缓存的因素
1. 服务器可以设置TTL值表示DNS记录的存活时间。本机DNS缓存将根据这个TTL值判断DNS记录什么时候被抛弃，这个TTL值一般都不会设置很大，主要是考虑到快速故障转移的问题。
2. 浏览器DNS缓存也有自己的过期时间，这个时间是独立于本机DNS缓存的，相对也比较短，例如chrome只有1分钟左右。
3. 浏览器DNS记录的数量也有限制，如果短时间内访问了大量不同域名的网站，则较早的DNS记录将被抛弃，必须重新查找。不过即使浏览器丢弃了DNS记录，操作系统的DNS缓存也有很大机率保留着该记录，这样可以避免通过网络查询而带来的延迟。

### DNS解析优化
1. 避免重定向、浏览器DNS缓存 、计算机DNS缓存、 服务器DNS缓存、使用Keep-Alive特性来减少DNS查找。
2. DNS的预解析。
+ 可以通过用meta信息来告知浏览器, 我这页面要做DNS预解析。
```
   <meta http-equiv="x-dns-prefetch-control" content="on" />
```
+ 可以使用link标签来强制对DNS做预解析。
```
   <link rel="dns-prefetch" href="http://ke.qq.com/" />
```
3. 当客户端的DNS缓存为空时，DNS查找的数量与Web页面中唯一主机名的数量相等。减少唯一主机名的数量就可以减少DNS查找的数量。较少的域名来减少DNS查找（2-4个主机）。