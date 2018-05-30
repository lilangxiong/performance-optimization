## [1] 减少HTTP请求
<p>    10%~20%的最终用于响应时间花在接受所请求的HTML文档上，剩余80%~90%的时间是花在所引用的组件（图片、脚本、样式表）的HTTP请求上。</p>

<p>    改善响应时间最简单的方法就是通过减少组件的数量，来减少HTTP请求数。</p>

#### [1-1] CSS Sprites
```
将多个图片合并到一个单独的图片中，利用CSS的background-position属性，将HTML元素放在背景图片中期望的位置上。
```
#### [1-2] Inline Images (内联图片)
----> 使用data: [\<mediatype\>][;base64],<data>的模式在页面中包含图片无需任何额外的HTTP请求。
```
实例：<img src="data:image/gif;base64,R0lGODlhAwADAIABAL6+vv///yH5BAEAAAEALAAAAAADAAMAAAIDjA9WADs=" />

缺陷：
    1、IE7及其以下版本的浏览器不支持。
    2、数据大小上有限制。
    3、data:url 是内联在页面上的，跨页面使用时不会被缓存。（可以通过将data:url卸载css样式表中，
       作为背景图片来实现缓存，因为css样式表会被缓存）
       
目前webpack已经可以通过url-loader来实现此项功能。
```

#### [1-3] Combined Scripts and Stylesheets （合并脚本和样式表）
```
1、使用外部脚本、样式表对性能更有利，而行内脚本会阻塞HTML的渲染。
2、保持javascript的模块化，在生成过程中从一组特定的模块生成一个目标文件。（目前webpack实现可以实现模块化开发、打包）
```

## [2] 使用内容分发网络（CDN）
<p>用户的带宽虽在增长，但是页面响应速度仍然受到用户与服务器物理位置的远近影响。</p>

#### [2-1] CDN
----> 内容分发网络是一组分布在多个不同地理位置的Web服务器，以便更加有效的向用户发布内容。

#### [2-2] 国内常用CDN厂商
- 阿里云
- 腾讯云
- 百度云
- 网宿科技
- 金山云

## [3] 通过 HTTP Header 设置缓存静态资源

#### [3-1] Expires
```
设置 Expires: Sat, 27 May 2028 06:15:41 GMT 之后，浏览器在后续页面浏览中会使用缓存版本，HTTP请求会减少一个。

在nginx.conf中配置Expires：
    location ~ image {
        root /var/www/;
        expires 1d;
        index index.html;
    }

缺陷：
    1、要求服务器、客户端时钟严格同步。
    2、需要经常检查过期时间。
    3、一旦过期之后，需要在服务器配置中提供一个新的日期。

```

#### [3-2] Cache-Control: max-age=*****
```
1、Cache-Control: max-age = 10000000，表示从组件被请求开始过去的秒数少于max-age，浏览器就
   继续使用缓存版本（HTTP 1.1版本以上）。
2、使用Cache-Control可以消除Expires的限制。
3、Cache-Control: max-age=***的优先级高于Expires。
    
基于Nginx的 ngx_http_headers_module 模块设置Cacht-Control：
if ($request_uri ~* "^/$|^/search/.+/|^/company/.+/") {
     add_header    Cache-Control  max-age=3600;
}
```
#### [3-3] 增加了缓存时间的文件，再次更新方案
- 直接修改所有的链接，全新的请求就会从源服务器上下载最新版本的文件。-----此方案无法再大型、复杂的项目中使用，容易出错。
- 为所有的组件的文件名使用变量，进行映射，在需要的文件中引用变量；修改时，只需要修改变量对应的内容（一般是在文件名后面增加版本号）即可。
```
实例:
    // map.js 导出一个映射
    export default map = {
        pic: 'https://www.xxx.com/assets/images/logo_0.0.1.png',
        js: 'https//www.xxx.com/assets/js/util_0.0.1.js'
    }
    
    // index.js 应用映射导入文件
    import map from 'map'
    <script src=map.js></script>
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

## CSS放在顶部

## Javascript放在底部
**同一时间针对同一域名下的请求**
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

**可以通过使用CNAME（DNS别名）将组件分别放在多个域名下。增加并行加载数**

## 使用外部的CSS以及Javascript
- 外部的Javascript、CSS能够被浏览器缓存。
- 外部的Javascript、CSS可以被多个页面重用。

## 减少DNS查找

