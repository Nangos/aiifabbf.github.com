# X11协议的研究
*12/22*

## 先来一波牢骚……

其实很早就想深入研究X Window System了，关于这个历史悠久的图形用户界面系统，我知道的东西应该不算多，但肯定也不算少，应该比常人知道的要多一些。这边不禁要吐槽一下，国内对这个东西感兴趣的人真的很少，以至于我在现实世界里，从来没有遇到任何一个除了Harvey以外的人跟我提过这个系统，对了，请不要认为X Window System是个操作系统=_=；我觉得究其原因，其一就是这个东西其实“没卵用”。

我在某岩写了一个学期的网页，现在终于退出了，身边很多同学都不理解，我这么轻轻松松进一个学校里顶尖社团，为什么要退出。好吧，其实是我自己实在写不了任何网页了。现在做web前端的有些工具的有些特性让我十分恶心，我不否认现在的HTML+CSS+JavaScript不是一个好架构，人们确实用这个架构写出了许许多多以前不敢想象的新鲜玩意，然而，其中有些事情实在让我没法理解，看了*CSS: The Definite Guide*也没法理解，比如为什么要无脑地少用使用内嵌样式表和内联样式表——是的这点我在做某岩的第二个新手任务重构[Twitter首页](twitter.com)时，组长提出来的问题，我说，既然这个网页长的那么像某个app的home screen，而且这个网页的样式也很独特，那么为什么不把这个HTML文件当作是一个界面描述呢，没有理由的外部链接是我没法接受的。组长说，以后你就知道了，然后，没有以后了。

当然其实我还是很欣赏组长的态度的，如果组长有个人博客的话我会很高兴加他友链的。组长说，我们还是各自做自己的事吧，既然你想要变革这一切，那就去做吧，团队也给不了你什么。

所以我研究这个协议的目的，是想从根本上看清图形界面到底是什么，需要什么，怎样才是理想的架构，我想X Window System经历了这么三十年的考验，其中一定是有一些道理的。另外，我也在搞[自己的试验](https://github.com/aiifabbf/casper)，希望能从这个架构里得到一些精华。Web中已经有很多精华了，我肯定会在我的试验中加入这些东西。当然这是自上而下的了，现在就让我来讨论下实现相关的东西，X11应该会变成我试验中的一种后端。

## 开始吧

一些基本的东西我就玩一波翻译吧。-w-

> Request Format

> Every request contains an 8-bit major opcode  and a 16-bit length field  expressed in units of four bytes. Every request consists of four bytes of a header (containing the major opcode, the length field, and a data byte) followed by zero or more additional bytes of data. The length field defines the total length of the request, including the header. The length field in a request must equal the minimum length required to contain the request. If the specified length is smaller or larger than the required length, an error is generated. Unused bytes in a request are not required to be zero. Major opcodes 128 through 255 are reserved for extensions.  Extensions are intended to contain multiple requests, so extension requests typically have an additional minor opcode encoded in the second data byte in the request header. However, the placement and interpretation of this minor opcode and of all other fields in extension requests are not defined by the core protocol. Every request on a given connection is implicitly assigned a sequence number,  starting with one, that is used in replies, errors, and events. 

> 请求的格式

> 请求包含一个4字节长度的头（里面有一个8位的主操作码、一个16位的长度域和数据字节），头后面跟着其他数据或零。长度域表示的是包含头的**整个**请求的长度，长度域必须等于这种请求类型指定的最小长度，如果小了或者大了会出错。请求中没用到的字节可以不为零。128到255的主操作码是为扩展预留的。因为扩展是要包含多个请求的，所以扩展请求往往在请求头的第二个数据字节里带有附加的副操作码。然而，核心协议没有定义副操作码的放置位置和解释方法，也没有定义扩展请求里任何其他东西。在现有的连接上传输的每个请求会被显式地分配一个从1开始排列的序列号，这个序号用于后续传输回应、错误和事件。

> Reply Format

> Every reply contains a 32-bit length field expressed in units of four bytes. Every reply consists of 32 bytes followed by zero or more additional bytes of data, as specified in the length field. Unused bytes within a reply are not guaranteed to be zero. Every reply also contains the least significant 16 bits of the sequence number of the corresponding request. 

> 回应的格式

> 回应包含一个4字节长度的头，头里面是32位的长度域。回应由32字节的数据和长度域中指定的其他数据或零组成。在回应中没用到的字节不保证一定是零。回应也包含最低位16位长度的序列号，该序列号对应着刚刚的请求。

> Error Format
> Error reports are 32 bytes long. Every error includes an 8-bit error code.  Error codes 128 through 255 are reserved for extensions.    Every error also includes the major and minor opcodes of the failed request and the least significant 16 bits of the sequence number of the request. For the following errors (see section 4), the failing resource ID is also returned: Colormap, Cursor, Drawable, Font, GContext, IDChoice, Pixmap and Window. For Atom errors, the failing atom is returned. For Value errors, the failing value is returned. Other core errors return no additional data. Unused bytes within an error are not guaranteed to be zero. 

> 错误的格式

> 错误都是32字节长度，包含8位的错误码。128到255的错误码是为扩展保留的。错误还包含对应错误请求的主副操作号、以及请求序列号中最低16位。对于*颜色映射?*、光标、可画对象、字体、图形上下文、*标示选择?*、位图、窗口来说，错误对应的资源标识也会返回；对于*原子?*、值错误，错误的原子、值也会分别返回。其他核心协议定义的错误不返回其他附加数据。在错误中没用到的字节不一定保证是零。

> Event Format

> Events are 32 bytes long. Unused bytes within an event are not guaranteed to be zero. Every event contains an 8-bit type code. The most significant bit in this code is set if the event was generated from a SendEvent request.  Event codes 64 through 127 are reserved for extensions, although the core protocol does not define a mechanism for selecting interest in such events.    Every core event (with the exception of KeymapNotify) also contains the least significant 16 bits of the sequence number of the last request issued by the client that was (or is currently being) processed by the server.

> 事件的格式

> 事件都是32字节长度。没用到的字节不一定保证是零。事件中包含一个8位的类型码，如果事件是从SendEvent请求产生的，那么这个类型码中的最高位会被设置。64到127的事件码是为扩展预留的，虽然核心协议中没有定义一种机制，使得扩展从这些事件里挑选自己感兴趣的内容。核心事件（除了KeymapNotify）也包含X服务器处理过或正在处理的此客户端发送的前一个请求对应的序列号的最低16位。

这个文档写的还是蛮无聊的……有一些句子感觉有歧义，比如定语后置修饰的对象。我这边用下图示法给大家演示下吧。


<style>
.bar {
    width: 100%;
    height: 40px;
    display: inline-block;
    background-color: #ffffff;
    color: #ffffff;
    font-family: sans-serif;
    font-size: 15px;
    line-height: 40px;
    text-align: center;
    overflow: hidden;
    margin-right: -5px;
}
</style>

请求
<div class="bar" style="text-align: left;">
    <div class="bar" style="background-color: #000000; width: 15%;">opcode 8 bits</div>
    <div class="bar" style="background-color: #4d4d4d; width: 30%;">length field 16 bits</div>
    <div class="bar" style="background-color: #808080; width: 15%;">data byte 8 bits</div>
    <div class="bar" style="background-color: #aaaaaa; width: 40%;"></div>
</div>
<div class="bar" style="color: black;">
    All (length depends)
</div>

回应
<div class="bar" style="text-align: left;">
    <div class="bar" style="background-color: #000000; width: 60%;">length field 32 bits</div>
    <div class="bar" style="background-color: #808080; width: 40%;">LSB sequence 16 bits</div>
</div>
<div class="bar" style="color: black;">
    All (often 32 bytes = 128 bits)
</div>

错误
<div class="bar" style="text-align: left;">
    <div class="bar" style="background-color: #000000; width: 15%;">error code 8 bits</div>
    <div class="bar" style="background-color: #808080; width: 85%;">maj, min opcodes & LSB sequence 16 bits</div>
</div>
<div class="bar" style="color: black;">
    All 32 bytes
</div>

事件
<div class="bar" style="text-align: left;">
    <div class="bar" style="background-color: #000000; width: 15%;">type code 8 bits</div>
    <div class="bar" style="background-color: #808080; width: 85%;">LSB sequence 16 bits & rest</div>
</div>
<div class="bar" style="color: black;">
    All 32 bytes
</div>

**上面译文的准确性有待核实。*

## 数据类型

*未完*