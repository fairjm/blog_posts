最近在进行旧版本的`prestosql`和`prestodb`升级相关的操作，尝试自己编译了一下，这里记录一下过程和遇到问题的处理。    
因为`Trino`不支持windows下的编译，如果使用windows最方便的方式就是使用`wsl`了。

# WSL中编译和调试
`wsl`的准备工作不累述了，升级到`wsl2`，使用的是`ubuntu`.  
详见：
[Windows Subsystem for Linux Installation Guide for Windows 10](https://docs.microsoft.com/en-us/windows/wsl/install-win10)  

## 工具安装
其他的过程包括安装`java`并设置一下`JAVA_HOME`(maven需要使用) ，`maven`和`git`之类。  
```bash
sudo apt install openjdk-11-jdk

#如果之前有其他发行版
update-alternatives --list java

wget https://mirror.bit.edu.cn/apache/maven/maven-3/3.6.3/binaries/apache-maven-3.6.3-bin.tar.gz

sudo tar -xzvf apache-maven-3.6.3-bin.tar.gz  -C /opt/maven
```  

`idea`同理，去官网下载然后移动到对应目录即可。  
（本来想通过`jetbrain-toolbox`安装，但不知道为什么不能显示gui界面放弃了） 

## 配置修改
修改一下`/etc/profile`
```bash
export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64
export M2_HOME=/opt/maven/apache-maven-3.6.3
export IDEA_HOME=/opt/idea/idea-IU-203.7148.57
export PATH=$M2_HOME/bin:$IDEA_HOME/bin:$PATH
```  

## xserver
为了方便xserver的配置，直接使用了`mobaxterm`，注意这里不用再去看网上的wsl的xserver设置，`moba`自己已经设置好了而且用的是和网上其他文档不同的port，自己设置了反而弹不出来。    

在wsl中输入`idea.sh`即可弹出界面。  
（这样简单设置无法使用外部的输入法，已经不支持外部剪切板（默认支持内部的复制到外部）  

## 编译
默认的分支是最新版本的snapshot，需要切换到对应tag：
```bash
git fetch --all
git checkout tags/352
```
可以将`wsl`的`maven`的本地仓库路径设置到外部的仓库，这样就可以复用已有的不需要完全下载新的。  

编译的方式和运行就按照`trino`仓库即可（见[trino](https://github.com/trinodb/trino))，其中有一个文档是不需要编译的，且会比较耗时间，可以这么跳过：
```bash
mvn -pl '!docs' clean install -DskipTests
```  
## 调试插件
启动命令在官网仓库中有，直接使用即可。  

对于要调试的插件，将项目放入`plugin`目录中，默认是不会加载的，修改一下`core/trino-server-main/etc/config.properties`在`plugin.bundles`中加入自己项目的路径即可。（这里的加载插件很多，启动会比较慢可以适当减少一些）

# 升级遇到的问题
迁移的方法官网给了说明：[Migrating from PrestoSQL to Trino](https://trino.io/blog/2021/01/04/migrating-from-prestosql-to-trino.html)
最主要的一点是在配置文件（`$TRINO_HOME/etc/config.properties`）中增加
`protocol.v1.alternate-header-name=Presto`  。  

`UDF`升级过程还可以，不得不说API的兼容性还是很好的，升级包之后API都是兼容的只是修改了一下路径。      
需要注意的是之前一直需要的那个`io.trino.spi.Plugin`文件不需要了，当前打包会自动生成，有了他反而会编译失败。

主要遇到了两个问题：`@OutputFunction`注解的内容解析方式改变了，新版的`Trino`使用了`SqlBase.g4`中`type`的语法。  
我们之前的形式`array(row(start timestamp,end timestamp))`会解析失败，在插件load的时候会挂掉（服务启动失败），因为end是一个保留字。  
```java
io.trino.sql.parser.ParsingException: line 1:35: mismatched input 'end'. Expecting: <identifier>, <type>
	at io.trino.sql.parser.ErrorHandler.syntaxError(ErrorHandler.java:108)
	at org.antlr.v4.runtime.ProxyErrorListener.syntaxError(ProxyErrorListener.java:41)
	at org.antlr.v4.runtime.Parser.notifyErrorListeners(Parser.java:544)
	at org.antlr.v4.runtime.DefaultErrorStrategy.reportUnwantedToken(DefaultErrorStrategy.java:377)
	at org.antlr.v4.runtime.DefaultErrorStrategy.singleTokenDeletion(DefaultErrorStrategy.java:548)
	at org.antlr.v4.runtime.DefaultErrorStrategy.sync(DefaultErrorStrategy.java:266)
	at io.trino.sql.parser.SqlBaseParser.rowField(SqlBaseParser.java:11435)
	at io.trino.sql.parser.SqlBaseParser.type(SqlBaseParser.java:11103)
	at io.trino.sql.parser.SqlBaseParser.typeParameter(SqlBaseParser.java:11645)
	at io.trino.sql.parser.SqlBaseParser.type(SqlBaseParser.java:11329)
	at io.trino.sql.parser.SqlBaseParser.standaloneType(SqlBaseParser.java:404)
	at io.trino.sql.parser.SqlParser.invokeParser(SqlParser.java:139)
	at io.trino.sql.parser.SqlParser.createType(SqlParser.java:94)
	at io.trino.sql.analyzer.TypeSignatureTranslator.parseTypeSignature(TypeSignatureTranslator.java:98)
	at io.trino.operator.aggregation.AggregationImplementation$Parser.<init>(AggregationImplementation.java:315)
	at io.trino.operator.aggregation.AggregationImplementation$Parser.parseImplementation(AggregationImplementation.java:357)
	at io.trino.operator.aggregation.AggregationFromAnnotationsParser.parseFunctionDefinitions(AggregationFromAnnotationsParser.java:83)
	at io.trino.metadata.SqlAggregationFunction.createFunctionsByAnnotations(SqlAggregationFunction.java:45)
	at io.trino.metadata.FunctionExtractor.extractFunctions(FunctionExtractor.java:49)
	at io.trino.server.PluginManager.installPluginInternal(PluginManager.java:203)
	at io.trino.server.PluginManager.installPlugin(PluginManager.java:175)
	at io.trino.server.PluginManager.loadPlugin(PluginManager.java:169)
	at io.trino.server.PluginManager.loadPlugin(PluginManager.java:157)
	at io.trino.server.PluginManager.loadPlugins(PluginManager.java:143)
```
这部分的规则是这样组成的：
```
type
    : ROW '(' rowField (',' rowField)* ')'     <- 命中这条                                    #rowType
...  

rowField
    : type
    | identifier type; <- 命中这条  

identifier
    : IDENTIFIER             #unquotedIdentifier
    | QUOTED_IDENTIFIER      #quotedIdentifier
    | nonReserved            #unquotedIdentifier <- END不在这里
    | BACKQUOTED_IDENTIFIER  #backQuotedIdentifier
    | DIGIT_IDENTIFIER       #digitIdentifier
    ;

nonReserved
    // IMPORTANT: this rule must only contain tokens. Nested rules are not supported. See SqlParser.exitNonReserved
    : ADD | ADMIN | ALL | ANALYZE | ANY | ARRAY | ASC | AT | AUTHORIZATION
    | BERNOULLI
    | CALL | CASCADE | CATALOGS | COLUMN | COLUMNS | COMMENT | COMMIT | COMMITTED | CURRENT
    | DATA | DATE | DAY | DEFINER | DESC | DISTRIBUTED | DOUBLE
    | EXCLUDING | EXPLAIN
...  

END: 'END';
```  

修改很简单，只需要end加上转义即可，变为了`array(row(start timestamp,\"end\" timestamp))`。
至此可以编译成功（类型挂掉的都是因为和解析不符）。  

但运行使用这个`UDF`的`sql`会报错，这就是另一个问题了，`timestamp`的类型增加了。 `timestamp(3)`和`timestamp`不匹配，这个比较有意思，文档里说了`timestamp`是`timestamp(3)`的别名：  
> [#TIMESTAMP](https://trino.io/docs/current/language/types.html?highlight=timestamp%203#timestamp)  
> `TIMESTAMP` is an alias for `TIMESTAMP(3)` (millisecond precision).  

但`UDF`里就是不能这么写，最后改为`array(row(start timestamp(3),\"end\" timestamp(3)))`。  
至此问题解决。  

现在还在测试中，`UDF`这些的文档有些欠缺了，自己摸索了一下发现还不如看源码来得直接，这部分的文档缺失的也厉害，写/改`UDF`基本也是靠已有的例子摸索。  
其他的部分都还不错，兼容性也很好，可见`Trino`的社区支持还是很到位的，要感谢各位大佬的努力 :)。

希望之后替换可以顺利吧~

> Written with [StackEdit](https://stackedit.io/).
