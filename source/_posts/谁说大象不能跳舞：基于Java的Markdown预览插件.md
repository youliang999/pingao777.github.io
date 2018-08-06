---
title: 谁说大象不能跳舞：基于Java的Markdown预览插件
date: 2018-08-04 11:49:59
categories: 技术人生
tags: [Markdown, Java, Vim]
---

Java一直以来都给人留下了笨重的印象，按说插件这种轻量的任务根本和Java没关系，但是这次我要霸王硬上弓，让大象跳次舞。

跳什么舞呢？这是个问题，突然想起，一直以来写博客都是用Vim编辑Markdown，有时候想看看预览，欣赏下文字跳动的样子，但是Vim不支持预览，用过[Markdown Viewer](https://chrome.google.com/webstore/detail/markdown-viewer/ckkdlimhmcjmikdlpkmbgfkaikojcbjk)可惜不支持同步滚动，能不能整个插件呢？

### 整体思路

借用Markdown Viewer的思路，通过程序将Vim的某种位置信息传给浏览器，浏览器调用js滚动到这个位置不就实现滚动了吗？既然想让浏览器显示网页而且网页的内容还得不停的变，需要一个Web服务器，这正是Java的强项，没有任何问题。从浏览器到Java的路走通了，但是Vim到Java的路怎么走呢？由于Vim不支持Java，二者怎么通信呢，这时看到著名的胶水语言，编程语言界的媒婆Python是被Vim支持的，方案有了：让Python与Java通信不就可以了。好，这样整个流程就走通了。

### Java服务器

Web简直就是Java的看家本领，自然一点问题没有。以前学过Netty，一直没有派上用场，这次终于可以小试牛刀了。

```java
public class HttpRequestHandler extends SimpleChannelInboundHandler<FullHttpRequest> {

    private MarkDownServer server;

    public HttpRequestHandler(MarkDownServer server) {
        this.server = server;
    }

    @Override
    public void channelRead0(ChannelHandlerContext ctx, FullHttpRequest request) {
        String uri = request.uri();
        if (uri.startsWith("/index")) {
            index(ctx, request);
        } else if (uri.startsWith("/ws")) {
            ctx.fireChannelRead(request.retain());
        } else if (uri.startsWith("/js") || uri.startsWith("/css")) {
            transferStaticFile(ctx, request);
        } else if (uri.startsWith("/image")) {
            image(ctx, request);
        } else if (uri.startsWith("/sync")) {
            sync(ctx, request);
        } else if (uri.startsWith("/close")) {
            close(ctx, request);
        } else if (uri.startsWith("/stop")) {
            stop(ctx, request);
        } else {
            commonResponse(ctx, request, FileUtils.getBytes("How do you do"), MiMeType.PLAIN);
        }
    }
}
```

上面即是Web服务的框架程序，细节就不贴了。但是这里有几个问题大家要注意：

- 服务器怎么和浏览器通信？
- 网页里的js和css怎么返回？
- Python怎么和Java通信？

这几个问题就是上面条件语句处理的内容了。先说服务器怎么和浏览器通信，最初定的是ajax，但是后来找到一种更好的方案WebSocket，WebSocket可以让浏览器和服务器保持长连接，有更好的流畅性，这就是上面`/ws`所处理的内容；第二个问题，网页中的静态文件js和css需要让服务器返回，这里没什么复杂的，就是一个简单的文件读写，只有一点需要注意，注意文件的编码，否则页面会乱码；第三个问题，Python怎么和Java通信，最初采用的是[py4j](https://www.py4j.org/)，优点是这个库可以直接在Python程序里调用Java程序，缺点是需要安装额外的库，对于有着代码洁癖的我不是一个完美的方案，既然已经有了Web服务器，直接使用Http不就可以了，所以最终将方案改为使用Http与Java通信。

### Java->浏览器

```javascript
var url = 'ws://127.0.0.1:' + window.location.port + '/ws';
if (!WebSocket) {
    console.warn('WebSocket is not support');
} else {
    console.log('Try to connect ' + url);
    var ws = new WebSocket(url);

    ws.onclose = function () {
        console.log('Disconnected');
        close();
    };

    ws.onmessage = function (d) {
        console.log('Response : ' + d.data.length);
        var data = JSON.parse(d.data);
        var path = $('#path');
        if (path.val() === '') {
            init(data);
        } else {
            if (path.val() === data.path) {
                if (data.command === 'close') {
                    close()
                } else if (data.command === 'sync') {
                    sync(data);
                }
            }
        }
    };
}

var init = function (data) {
    $('title').html(data.path);
    $('#path').val(data.path);
    $('.markdown-body').html('');
    markdown_refresh(data.units);
    highlight_code();
    scroll_if_possible();
};

var sync = function (data) {
    markdown_refresh(data.units);
    highlight_code();
    scroll_if_possible();
};
```

前端就是一个WebSoket客户端，接受两种指令：`close`和`sync`，前者用于关闭页面，后者用于同步信息。

大致的流程是浏览器第一次收到服务器消息调用`init`进行初始化，它会将网页的`title`和页面元素#`#path`置为文件的路径，将`.markdown-body`清空，因为浏览器刚启动时会启动一个`index`介绍页面，需要将其清空，然后调用`markdown_refresh`刷新显示新的内容，内容显示后`hightlight_code`负责高亮其中的代码，最后用`scroll_if_possible`滚动到指定位置。往后浏览器每收到服务器消息都会调用`sync`同步信息和内容，`sync`和`init`内部代码差不多，我就不再赘述了。

### Vim->Java

以上走通了Java到浏览器的路，下面看看如何走Vim到Java的路，在整体思路中提到Vim支持Python，所以这条路的主角就是Python，利用Python发送相应的Http请求代表相应的命令来达到控制浏览器显示的作用。主代码如下：

```python
try:
    from urllib.parse import urlencode
    from urllib.request import urlopen, Request
except ImportError:
    from urllib import urlencode
    from urllib2 import urlopen, Request


url_pre = 'http://127.0.0.1:port'


def set_port(port):
    global url_pre
    url_pre = url_pre.replace('port', port)


def sync(path, content, bottom):
    params = {'path': path, 'content': base64.b64encode(content), 'bottom': bottom}
    request = Request(url_pre + '/sync', data = urlencode(params))
    urlopen(request)


def close(path):
    params = {'path': path}
    request = Request(url_pre + '/close?' + urlencode(params))
    urlopen(request)


def stop():
    request = Request(url_pre + '/stop')
    urlopen(request)
```

主要命令如下：

- 同步内容和位置：`sync`
- 关闭浏览器页面：`close`
- 关闭Web服务器：`stop`

变量解释：path表示打开文件的路径，content为buffer内容，bottom为当前窗口的末行。

这里需要注意的有两点：一是python2和python3的`urllib`库路径不一样，这里采用了`try` `catch`语句做了下兼容；二是对content进行了base64编码，因为content里可能包含一些特殊字符，传输过程中对这些字符处理不一致导致出错。

下一步就是在Vimscript里调用Python代码，我就不贴代码了。

### 一步之遥

至此，插件就编写完成了，离成功仿佛只有一步之遥了。运行一下欣赏下自己的成果吧，整体效果还可以，但是存在下面几个问题：

#### MathJax抖动

这是因为前端每次加载内容，都要刷新所有的MathJax表达式，这样给人的感觉就是这些表达式一抖一抖的，用的代码如下：

```javascript
MathJax.Hub.Config({
extensions: ["tex2jax.js"],
    jax: ["input/TeX", "output/SVG"],
    tex2jax: {
        inlineMath: [ ['$','$'], ["\\(","\\)"] ],
        displayMath: [ ['$$','$$'], ["\\[","\\]"] ]
    },
    messageStyle: "none"
});
MathJax.Hub.Queue(["Typeset", MathJax.Hub]);
```

那怎么解决呢？既然是因为刷新全部的MathJax表达式，那么不让它刷新全部，让它刷新的内容越少不就不会抖动了。查阅MathJax的文档，找到一条命令`MathJax.Hub.Queue(['Typeset', MathJax.Hub, element])`，这条命令只会刷新页面元素element里的表达式，这意味着现在服务器往浏览器同步信息的时候不能再是全量而是增量的，如果里面包含数学表达式就调用上述命令刷新。

#### 本地图片不显示

由于浏览器安全策略，浏览器页面的img标签不能打开本地图片，即形如`<img src='D:\path-to-img.jpg'/>`这种写法浏览器不会正确的加载图片，而是会提示“Not allowed to load local resource”。
