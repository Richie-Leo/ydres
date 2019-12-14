-------------------------
#### PinPoint架构
<img src="https://richie-leo.github.io/ydres/img/10/120/1012/pinpoint-architecture.png" style="width:99%;max-width:600px;" />

-------------------------
#### 部署PinPoint
[PinPoint](https://github.com/naver/pinpoint)版本[1.8.5](https://github.com/naver/pinpoint/releases/tag/1.8.5)，Mac环境单机部署，存储只支持HBase，使用[my-demo](https://github.com/liuzhibin-cn/my-demo)作为演示项目。本文将PinPoint Collector和Web UI部署在同一个Tomcat下。
> [my-demo](https://github.com/liuzhibin-cn/my-demo)项目原本使用Apache Dubbo 2.7.4.1，但PinPoint不支持，无法采集到任何跟踪数据，因此改为Alibaba Dubbo 2.6.7。

1. 下载[pinpoint-agent-1.8.5.tar.gz](https://github.com/naver/pinpoint/releases/download/1.8.5/pinpoint-agent-1.8.5.tar.gz)、[pinpoint-collector-1.8.5.war](https://github.com/naver/pinpoint/releases/download/1.8.5/pinpoint-collector-1.8.5.war)、[pinpoint-web-1.8.5.war](https://github.com/naver/pinpoint/releases/download/1.8.5/pinpoint-web-1.8.5.war)；
   > 无法翻墙可能下载不了，可以从[gitee的PinPoint镜像](https://gitee.com/mirrors/pinpoint)拉取源码，参考文档本地编译。
2. 部署HBase，执行[hbase-create.hbase](https://github.com/naver/pinpoint/blob/master/hbase/scripts/hbase-create.hbase)在HBase中创建表结构：
   ```sh
   $HBASE_HOME/bin/hbase shell hbase-create.hbase
   ```
3. 下载部署[Tomcat 8](https://tomcat.apache.org/download-80.cgi)；
4. 部署Collector：
   1. 解压`pinpoint-collector-1.8.5.war`到Tomcat `webapps/pinpoint-collector`目录（`tar -xvzf`）；
   2. 配置`pinpoint-collector/WEB-INF/classes/hbase.properties`，指定HBase所用的Zookeeper信息：
      ```
      hbase.client.host=localhost
      hbase.client.port=2181
      hbase.zookeeper.znode.parent=/hbase
      hbase.namespace=default
      ```
      配置`pinpoint-collector/WEB-INF/classes/pinpoint-collector.properties`，本文全部采用默认配置。
5. 部署Web UI：
   1. 解压`pinpoint-web-1.8.5.war`到Tomcat `webapps/ROOT`目录；
   2. 配置`ROOT/WEB-INF/classes/hbase.properties`，同Collector；
   3. 配置`ROOT/WEB-INF/classes/pinpoint-web.properties`，本文采用默认配置，可以将`config.sendUsage`关闭；
   4. 配置`ROOT/WEB-INF/classes/jdbc.properties`，并在MySQL中建立database和表结构，参考[pinpoint/web/src/main/resources/sql/](https://github.com/naver/pinpoint/tree/master/web/src/main/resources/sql)；
      > 该步骤可选，用于配置用户、用户组、报警规则等。
6. 启动Tomcat；

-------------------------
#### 客户端使用agent
解压`pinpoint-agent-1.8.5.tar.gz`，配置`pinpoint.config`，其中的修改项如下：
```
# Collector和演示项目部署在不同机器，这里指定Collector IP
profiler.collector.ip=192.168.31.108
# 演示项目类型
profiler.applicationservertype=SPRING_BOOT
# 为演示项目中的test-app启用跟踪
profiler.entrypoint=my.demo.test.Application.runTestCaseWithTrace
profiler.tomcat.conditional.transform=false
```

启动应用：
- `-javaagent:`：指定PinPoint代理；
- `-Dpinpoint.agentId=`：必须全局唯一，代表一个服务、应用实例；
- `-Dpinpoint.applicationName=`：指定服务、应用名称。相同服务部署不同实例，`applicationName`相同，`agentId`不同；

演示项目启动脚本：
```sh
java -javaagent:F:\workspace\pinpoint\agent\pinpoint-bootstrap-1.8.5.jar -Dpinpoint.agentId=item-srv-1 -Dpinpoint.applicationName=item-srv -jar item-service\target\item-service-0.0.1-SNAPSHOT.jar
java -javaagent:F:\workspace\pinpoint\agent\pinpoint-bootstrap-1.8.5.jar -Dpinpoint.agentId=stock-srv-1 -Dpinpoint.applicationName=stock-srv -jar stock-service\target\stock-service-0.0.1-SNAPSHOT.jar
java -javaagent:F:\workspace\pinpoint\agent\pinpoint-bootstrap-1.8.5.jar -Dpinpoint.agentId=user-srv-1 -Dpinpoint.applicationName=user-srv -jar user-service\target\user-service-0.0.1-SNAPSHOT.jar
java -javaagent:F:\workspace\pinpoint\agent\pinpoint-bootstrap-1.8.5.jar -Dpinpoint.agentId=order-srv-1 -Dpinpoint.applicationName=order-srv -jar order-service\target\order-service-0.0.1-SNAPSHOT.jar
java -javaagent:F:\workspace\pinpoint\agent\pinpoint-bootstrap-1.8.5.jar -Dpinpoint.agentId=test-app -Dpinpoint.applicationName=test-app -jar test-app\target\test-app-0.0.1-SNAPSHOT.jar
```

-------------------------
#### 全链路跟踪
支持情况参考[Supported Modules](https://naver.github.io/pinpoint/main.html#supported-modules)，对支持框架的关键方法进行跟踪，例如Dubbo服务的客户端调用和服务端入口，JDBC的`PrepareStatement`数据库操作等。

`TransactionId`标记一个调用链，全局唯一，对应于[Google Dapper](http://research.google.com/pubs/pub36356.html)的`TraceId`；调用链上每次服务实例处理RPC请求将生成一个Span，Span通过父子关系记录RPC调用层次关系：<br />
<img src="https://richie-leo.github.io/ydres/img/10/120/1012/trace-behavior.png" style="width:99%;max-width:540px;" />

Span中每次方法调用生成一个SpanEvent：<br />
<img src="https://richie-leo.github.io/ydres/img/10/120/1012/trace-data-structure.jpg" style="width:99%;max-width:700px;" />

业务代码中未被跟踪的方法，如果需要跟踪，通过`agent/pinpoint.config`配置：
1. `profiler.entrypoint`：配置一个全链路跟踪的入口点，新建一个全链路跟踪，必须配置类+方法的全限定名，多个方法逗号分隔，不支持通配符；
2. `profiler.include`：类的全限定名，或者包的全限定名（支持通配符），则指定类或者包下的所有方法都将被跟踪；

-------------------------
#### 应用日志中输出全局transaction-id
1. 修改`agent/pinpoint.config`：
   ```
   profiler.logback.logging.transactioninfo=false
   ````
2. logback通过`%X{PtxId}`输出`TransactionId`，`%X{PspanId}`输出`SpanId`：
   ```
   <appender name="APP" class="ch.qos.logback.core.ConsoleAppender">
       <encoder class="ch.qos.logback.core.encoder.LayoutWrappingEncoder">
           <charset>UTF-8</charset>
           <layout class="ch.qos.logback.classic.PatternLayout">
               <!-- %X{PtxId}: PinPoint TransactionId -->
               <!-- %X{PspanId}: PinPoint SpanId -->
               <pattern>${CONSOLE_LOG_PATTERN:-%clr([%d{${LOG_DATEFORMAT_PATTERN:-yyMMdd HH:mm:ss.SSS}}]){faint} %clr(${LOG_LEVEL_PATTERN:-%5p}) %clr(${PID:- }){magenta} %clr(%X{PtxId}-%X{PspanId}){yellow}%clr(:){faint} %m%n${LOG_EXCEPTION_CONVERSION_WORD:-%wEx}}</pattern>
           </layout>
       </encoder>
   </appender>
   ```

<img src="https://richie-leo.github.io/ydres/img/10/120/1012/app-log.png" style="width:99%;max-width:1000px;" />

-------------------------
#### 界面功能

主界面：
- 服务关系拓补图中：绿色圆圈表示正常，有红色部分表示发生了异常，右边散点图上也是用红色点表示，鼠标框住可以在全链路跟踪中查看发生异常的请求；
- 散点图：按时间顺序的响应时间分布图，用鼠标左键框一个时间窗口，将展示该时间范围内的全链路跟踪情况；

<img src="https://richie-leo.github.io/ydres/img/10/120/1012/pinpoint-screen-main.jpg" style="width:99%;max-width:1000px;" />

Agent Inspector：在主界面选中某个服务或应用，点击`Inspector`按钮打开，其中展示：**JVM堆/非堆内存和GC情况**、**JVM/System CPU**、**每秒事物数(吞吐率)**、**活动线程数**、**响应时间**等。<br />
<img src="https://richie-leo.github.io/ydres/img/10/120/1012/pinpoint-screen-agent-inspector.jpg" style="width:99%;max-width:1000px;" />

Application Inspector：Agent Inspector针对单个应用、服务实例，Application Inspector则针对某个服务的全部实例（applicationName相同，agentId不同），参考[How to use Application Inspector](https://naver.github.io/pinpoint/1.7.3/applicationinspector.html)。<br />
<img src="https://richie-leo.github.io/ydres/img/10/120/1012/pinpoint-screen-application-inspector1.jpg" style="width:99%;max-width:1000px;" /><br />
<img src="https://richie-leo.github.io/ydres/img/10/120/1012/pinpoint-screen-application-inspector2.jpg" style="width:99%;max-width:700px;" />

全链路跟踪、性能分析：

Call Tree形式展示：<br />
<img src="https://richie-leo.github.io/ydres/img/10/120/1012/pinpoint-screen-trace-call-tree.png" style="width:99%;max-width:1000px;" />

Timeline形式展示：<br />
<img src="https://richie-leo.github.io/ydres/img/10/120/1012/pinpoint-screen-trace-timeline.png" style="width:99%;max-width:1000px;" />

Mixed View形式展示：<br />
<img src="https://richie-leo.github.io/ydres/img/10/120/1012/pinpoint-screen-trace-mixed-view.png" style="width:99%;max-width:1000px;" />
