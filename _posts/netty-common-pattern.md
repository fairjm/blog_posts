---
title: "netty常用使用方式"
date: 2017/10/08 00:55:00
---
最近在重新看netty，在这里总结一些netty的一些常用的使用方式，类似于模板，方便速查。  
以netty 4.1.x的API作记录，不同版本可能API有略微差异，需要注意netty5被废弃掉了（辨别方式是看`SimpleChannelInboundHandler`是否有一个`messageReceived`方法 有的话就是5)，netty3是以`org.jboss`开头为包名。

# 统一模板  
大多数情况下使用netty的步骤是定义好`EventLoopGroup`,定义好`Bootstrap`(`ServerBootstrap`)以及使用的channel类型（一般就是`NioSocketChannel`，服务端是`NioServerSocketChannel`）。  
剩下是业务相关，使用的`ChannelInitializer`和具体的handler。  
主要就是2+N（N为自定义的handler数量）个类。  
服务端启动模板（也可以不区分boss和worker 用一个）:  

	public static void main(String[] args) throws InterruptedException {
			EventLoopGroup bossGroup = new NioEventLoopGroup();
			EventLoopGroup workerGroup = new NioEventLoopGroup();
			try {
				ServerBootstrap serverBootstrap = new ServerBootstrap();
				serverBootstrap.group(bossGroup, workerGroup).channel(NioServerSocketChannel.class)
						.childHandler(new MyChannelInitializer());
				ChannelFuture future = serverBootstrap.bind(8999).sync();
				future.channel().closeFuture().sync();
			} finally {
				bossGroup.shutdownGracefully();
				workerGroup.shutdownGracefully();
			}
	}
  
客户端启动模板：

	EventLoopGroup group = new NioEventLoopGroup();
	try {
		Bootstrap bootstrap = new Bootstrap()
				.group(group)
				.channel(NioSocketChannel.class)
				.handler(new MyChannelInitializer());
		ChannelFuture future = bootstrap.connect("localhost", 8888).sync();
		future.channel().closeFuture().sync();
	} finally {
		group.shutdownGracefully();
	}

ChannelInitializer模板（继承ChannelInitializer即可）：  

	public class MyChannelInitializer extends ChannelInitializer<SocketChannel> {

		@Override
		protected void initChannel(SocketChannel ch) throws Exception {
			ChannelPipeline pipeline = ch.pipeline();
			pipeline.addLast(...);
		}

	}
  
接下来的例子就以这个模板为骨架，主要涉及到初始化器的代码，启动代码一致。  

# 处理Http请求  
netty自带了对用的codec类比较方便。  
`pipeline.addLast("httpServerCodec", new HttpServerCodec());`  
自己实现的handler最简单的方式用`SimpleChannelInboundHandler`接收`HttpRequest`方法即可  
`class MyHttpHandler extends SimpleChannelInboundHandler<HttpRequest>`  
这里简单说下`SimpleChannelInboundHandler`这个类，他是简化处理接受信息并处理的一个类，主要做两件事。 

第一件事是根据泛型决定是否处理这个消息，能够处理就自己处理，不行就交给下一个（可以参考`acceptInboundMessage`和`channelRead`方法）。  
第二件事是消息的自动回收（有构造函数支持 默认是true），消息的引用计数会减一（所以在其他地方还要使用记得再retain一下）。  
使用它可以节省很多冗余代码的编写。  
一个简单例子：  

	public class MyHttpHandler extends SimpleChannelInboundHandler<HttpRequest> {
		@Override
		protected void channelRead0(ChannelHandlerContext ctx, HttpRequest msg) throws Exception {
			System.out.println(msg.getClass());
			System.out.println(msg.uri());
			System.out.println(msg.method().name());
			System.out.println(ctx.channel().remoteAddress());
			System.out.println("headers:");
			msg.headers().forEach(System.out::println);
			ByteBuf buf = Unpooled.copiedBuffer("Hello World", CharsetUtil.UTF_8);
			FullHttpResponse response = new DefaultFullHttpResponse(HttpVersion.HTTP_1_1, HttpResponseStatus.OK, buf);
			response.headers().set(HttpHeaderNames.CONTENT_TYPE, "text/plain");
			response.headers().set(HttpHeaderNames.CONTENT_LENGTH, buf.readableBytes());
			ctx.writeAndFlush(response);
	//         ctx.channel().close();
		}
	}

不过只用这个handler并不能拿到content,还需要配合`ChunkedWriteHandler`和`HttpObjectAggregator`得到`FullHttpRequest`对象。

## 处理WebSocket请求
只需要在上面的基础上增加一个`WebSocketServerProtocolHandler`即可，完整如下：

        pipeline.addLast("httpServerCodec", new HttpServerCodec());
        pipeline.addLast(new ChunkedWriteHandler());
        pipeline.addLast(new HttpObjectAggregator(8096));
        pipeline.addLast(new WebSocketServerProtocolHandler("/ws"));

自己的处理器可以接收并处理`WebSocketFrame`的子类。  


# 自定义文本协议  
netty提供了几个比较方便的用于自定义文本协议的编解码器。  
## 基于长度

	pipeline.addLast(new LengthFieldBasedFrameDecoder(Integer.MAX_VALUE, 0, 4, 0, 4));
	pipeline.addLast(new LengthFieldPrepender(4));
	pipeline.addLast(new StringDecoder(CharsetUtil.UTF_8));
	pipeline.addLast(new StringEncoder(CharsetUtil.UTF_8));
	pipeline.addLast(new MyHandler());  

上述参数的意义直接搬了文档：  

	 lengthFieldOffset   = 0
	 lengthFieldLength   = 2
	 lengthAdjustment    = 0
	 initialBytesToStrip = 2 (= the length of the Length field)

	 BEFORE DECODE (14 bytes)         AFTER DECODE (12 bytes)
	 +--------+----------------+      +----------------+
	 | Length | Actual Content |----->| Actual Content |
	 | 0x000C | "HELLO, WORLD" |      | "HELLO, WORLD" |
	 +--------+----------------+      +----------------+

## 基于分隔符  
        ChannelPipeline pipeline = ch.pipeline();
        pipeline.addLast(new DelimiterBasedFrameDecoder(4096, Delimiters.lineDelimiter()));
        pipeline.addLast(new StringDecoder(CharsetUtil.UTF_8));
        pipeline.addLast(new StringEncoder(CharsetUtil.UTF_8));  
第二个参数需要一个`ByteBuf[]`，自己指定下分隔符即可。  
注意使用这种发消息的时候要带上那个分隔符，不然会处理失败（被当做未结束）。  

# 广播实现  
利用了netty提供的一个很便利的类，`ChannelGroup`.  
首先要了解一下Channel生命周期函数的调用：  
`channel added -> channel registered -> channel active -> channel inactive -> channel unregistered -> channel removed`  
我们可以定义一个静态（保证共享和唯一）的`ChannelGroup`在channel added的时候把对应channel增加到`ChannelGroup`中即可（但不需要自己移除，这一点他自己实现了）  
然后利用它的写方法就可以实现广播，或者forEach做下过滤做多播也可以。  
这个实现和用什么协议无关，主要涉及到`ChannelGroup`使用。  

# 心跳  
netty自带的`IdleStateHandler`，超时后会向下一个handler发出`IdleStateEvent`消息，接收并处理即可。 

    @Override
    public void userEventTriggered(ChannelHandlerContext ctx, Object evt) throws Exception {
        super.userEventTriggered(ctx, evt);
        if (evt instanceof IdleStateEvent) {
            ctx.channel().writeAndFlush("超时").addListener((ChannelFuture ch) -> ch.channel().close());
        }
    }

它细分为3中类型的超时，读、写、读写，通过`IdleStateEvent`的state属性可以获取，可以单独判断。  

其他的一些模式会在之后复习和使用过程中不断补完~  
2017年10月08日00点53分 init
2017年10月10日10点02分 更新websocket
