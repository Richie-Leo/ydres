#### 项目情况
MyCat基于阿里早期开源的分库分表组件Cobar改进而来，以下参考[Cobar的架构与实践](https://blog.csdn.net/jiao_fuyou/article/details/15809999)：
- [Amoeba](https://sourceforge.net/projects/amoeba)：阿里B2B开发的分库分表中间件，应该开始于2006年左右，2008年开源，2010年已经广泛运用于阿里B2B业务，目前项目停滞；<br />
  <img src="https://richie-leo.github.io/ydres/img/10/120/amoeba-architecture.jpg" style="width:99%;max-width:430px;" />
- [Cobar](https://github.com/alibaba)：Amoeba进化版，后端由JDBC Driver改为原生MySQL通信协议，2011年10月发布，2012年开源在code.alibabatech上，后来迁移到github，目前项目停滞；<br />
  <img src="https://richie-leo.github.io/ydres/img/10/120/cobar-architecture.jpg" style="width:99%;max-width:530px;" />
- Mycat：由leaderus等人基于Cobar改进发展而来，后端由BIO改为NIO，改掉一些重要bug，功能增强（增加对Order By、Group By、limit等聚合功能的支持）。当年社区比较活跃，目前活跃度一般；
- 2012年阿里云将TDDL和Cobar结合开发DRDS；

#### Mycat架构
<img src="https://richie-leo.github.io/ydres/img/10/120/mycat-architecture.jpg" style="width:99%;max-width:550px;" />

- Mycat实现了MySQL协议，MySQL命令行客户端、任何开发语言都能像直接连MySQL一样连接Mycat，对客户端透明，支持所有开发语言；
- Mycat解析SQL语句，根据SQL参数和分片规则进行路由，跨分片查询对结果集进行汇总、重排序、分页、聚合等，将分库分表、读写分离等数据存储伸缩方案与应用隔离；

#### 演示方案说明
##### 表结构及拆分方案
使用[my-demo](https://github.com/liuzhibin-cn/my-demo)项目作为演示，运行项目查看演示效果。

表结构：<br />
<img src="https://richie-leo.github.io/ydres/img/10/120/mycat-table-schema.png" style="width:99%;max-width:550px;" />

拆分情况：<br />
<img src="https://richie-leo.github.io/ydres/img/10/120/mycat-sharding-tables.png" style="width:99%;max-width:450px;" />

- 逻辑库`db_user`：
  - `usr_user`: 会员表
    - _分片字段_：`user_id`
    - _分片规则_：`(user_id % 32) => { 0-15: dn1, 16-31: dn2 }`
    - _数据节点_：`dn1`, `dn2`
  - `usr_user_account`: 会员登录账号表 
    - _分片字段_：`account_hash`（`account.hashCode()`的绝对值）
    - _分片规则_：`(account_hash % 2) => { 0: dn1, 1: dn2 }`
    - _数据节点_：`dn1`, `dn2` 
- 逻辑库`db_order`：
  - `ord_order`: 订单主表。`order_id`创建订单时由订单服务在应用代码中生成；
    - _分片字段_：`order_id`
    - _分片规则_：`(order_id % 32) => { 0-7: dn1, 8-15: dn2, 16-23: dn3, 24-31: dn4 }`
    - _数据节点_：`dn1`, `dn2`, `dn3`, `dn4`
  - `ord_order_item`：订单明细表，与`ord_order`以父子关系表进行分片（Mycat ER分片），分片情况同`ord_order`
  - `ord_user_order`: `user_id`与`order_id`的自定义索引表，用于查询会员订单列表
    - _分片字段_：`user_id` 
    - _分片规则_：`(user_id % 32) => { 0-7: dn1, 8-15: dn2, 16-23: dn3, 24-31: dn4 }` 
    - _数据节点_：`dn1`, `dn2`, `dn3`, `dn4`

##### 关键流程说明
- 会员注册
  1. 使用`account`, `account_hash`, ... 插入`usr_user_account`表，`account_hash`确定分片。利用Mycat全局序列生成`user_id`；
  2. 使用`user_id`, ...插入`usr_user`表，`user_id`确定分片；
- 会员登录
  1. 使用`account`, `account_hash`查`usr_user_account`表，`account_hash`确定分片。得到`user_id`；
  2. 使用`user_id`查`usr_user`表，得到会员数据，`user_id`确定分片；
  3. 将`user_id`保存到session，后续用户请求通过session获得`user_id`，用来读取会员数据；
- 下单
  1. 由订单服务在应用中生成`order_id`，`user_id`登录时已保存到session，整个会话期间可获得；
  2. 插入`ord_order`表，`order_id`确定分片；
  3. 插入`ord_order_item`表，`order_id`确定分片；
  4. 插入`ord_user_order`表，`user_id`确定分片；<br />
     > 自定义索引表的数据维护，可以采用MQ方式异步写，由订单服务 -> MQ -> `ord_user_order`；也可以用Canal订阅方式，由MySQL -> Canal -> MQ -> `ord_user_order`。
- 查询会员订单列表
  1. 查`ord_user_order`表，`user_id`确定分片，得到`order_id`列表；
  2. 用`order_id`列表查`ord_order`，获得订单数据，`order_id`确定分片；

##### 物理部署
`dn0~4`可以分别部署在不同机器的MySQL实例上，演示项目方便起见只使用了一个MySQL实例，多实例部署只需修改配置文件中`dataNode`与`dataHost`映射关系即可。

#### 部署Mycat Server
[Mycat](http://www.mycat.io/)版本[1.6.7.3](http://dl.mycat.io/1.6.7.3/)，Mycat配置文件参考[docs/mycat-conf](https://github.com/liuzhibin-cn/my-demo/tree/master/docs/mycat-conf)

##### 配置
使用了Mycat数据库方式的全局序列，SQL脚本参考[sql-schema.sql](https://github.com/liuzhibin-cn/my-demo/blob/master/docs/sql-schema.sql)。

- [server.xml](https://github.com/liuzhibin-cn/my-demo/blob/master/docs/mycat-conf/server.xml)：配置服务器参数，配置逻辑用户名密码：
  ```xml
  <!-- 为Mycat逻辑库定义用户、密码 -->
  <user name="mydemo" defaultAccount="true">
    <property name="password">mydemo</property>
    <property name="schemas">db_user,db_order</property>
  </user>
  ```
- [schema.xml](https://github.com/liuzhibin-cn/my-demo/blob/master/docs/mycat-conf/schema.xml)：配置dataHost、dataNode、逻辑schema：
  ```xml
  <mycat:schema xmlns:mycat="http://io.mycat/">
	  <schema name="db_order" checkSQLschema="false" sqlMaxLimit="100">
		<table name="ord_order" primaryKey="order_id" dataNode="dn$1-4" rule="order-rule">
			<childTable name="ord_order_item" primaryKey="order_item_id" joinKey="order_id" parentKey="order_id" />
		</table>
		<table name="ord_user_order" dataNode="dn$1-4" rule="user-order-rule" />
	  </schema>
	  <schema name="db_user" checkSQLschema="false" sqlMaxLimit="100">
		<table name="usr_user" primaryKey="user_id" dataNode="dn1, dn2" rule="user-rule" />
		<table name="usr_user_account" primaryKey="account" dataNode="dn1, dn2" rule="user-account-rule" />
	  </schema>
	  <dataNode name="dn0" dataHost="db1" database="mydemo-dn0" />
	  <dataNode name="dn1" dataHost="db1" database="mydemo-dn1" />
	  <dataNode name="dn2" dataHost="db1" database="mydemo-dn2" />
	  <dataNode name="dn3" dataHost="db2" database="mydemo-dn3" />
	  <dataNode name="dn4" dataHost="db2" database="mydemo-dn4" />
	  <dataHost name="db1" maxCon="5" minCon="5" balance="0" writeType="0" dbType="mysql" dbDriver="native">
		<heartbeat>select user()</heartbeat>
		<writeHost host="localhost" url="localhost:3306" user="root" password="1234" />
	  </dataHost>
	  <dataHost name="db2" maxCon="5" minCon="5" balance="0" writeType="0" dbType="mysql" dbDriver="native">
        <heartbeat>select user()</heartbeat>
        <writeHost host="127.0.0.1" url="127.0.0.1:3306" user="root" password="1234" />
	  </dataHost>
  </mycat:schema>
  ```
- [rule.xml](https://github.com/liuzhibin-cn/my-demo/blob/master/docs/mycat-conf/rule.xml)：配置分片规则：
  ```xml
  <mycat:rule xmlns:mycat="http://io.mycat/">
	  <tableRule name="order-rule">
		<rule>
			<columns>order_id</columns>
			<algorithm>order-func</algorithm>
		</rule>
	  </tableRule>
	  <tableRule name="user-order-rule">
		<rule>
			<columns>user_id</columns>
			<algorithm>user-order-func</algorithm>
		</rule>
	  </tableRule>
	  <tableRule name="user-rule">
		<rule>
			<columns>user_id</columns>
			<algorithm>user-func</algorithm>
		</rule>
	  </tableRule>
	  <tableRule name="user-account-rule">
		<rule>
			<columns>account_hash</columns>
			<algorithm>user-account-func</algorithm>
		</rule>
	  </tableRule>
	  <function name="order-func" class="io.mycat.route.function.PartitionByPattern">
		  <property name="patternValue">32</property> <!-- 对32求模 -->
		  <!-- 求模结果按照文件配置的规则映射到分片 -->
		  <property name="mapFile">order-partition.txt</property> 
	  </function>
	  <function name="user-order-func" class="io.mycat.route.function.PartitionByPattern">
		  <property name="patternValue">32</property>
		  <property name="mapFile">order-partition.txt</property> <!-- 与订单公用分片规则 -->
	  </function>
	  <function name="user-func" class="io.mycat.route.function.PartitionByPattern">
		  <property name="patternValue">32</property> <!-- 对32求模 -->
		  <property name="mapFile">user-partition.txt</property> <!-- 求模结果按照文件配置的规则映射到分片 -->
	  </function>
	  <function name="user-account-func" class="io.mycat.route.function.PartitionByMod">
		  <property name="count">2</property> <!-- 数据分为2片 -->
	  </function>
  </mycat:rule>
  ```
- [order-partition.txt](https://github.com/liuzhibin-cn/my-demo/blob/master/docs/mycat-conf/order-partition.txt)，配置分片映射关系：
  ```
  0-7=0
  8-15=1
  16-23=2
  24-31=3
  ```
- [sequence_db_conf.properties](https://github.com/liuzhibin-cn/my-demo/blob/master/docs/mycat-conf/sequence_db_conf.properties)，配置Mycat全局序列位于哪个dataNode：
  ```
  GLOBAL=dn0
  ORDERDETAIL=dn0
  USER=dn0
  ```

##### Windows环境部署
Windows环境下载[Mycat-server-1.6.7.3-release-20190927161129-win.tar.gz](http://dl.mycat.io/1.6.7.3/20190927161129/Mycat-server-1.6.7.3-release-20190927161129-win.tar.gz)

有2种运行方式：
- 通过`bin\startup_nowrap.bat`在命令行直接运行。
  > 注意：需要在命令行进入`bin`目录后再执行`startup_nowrap.bat`命令，否则会将命令行所处当前目录作为Mycat主目录，导致无法找到`lib`等目录，`classpath`无法加载jar文件。
- 通过`bin\mycat.bat install`注册为Windows服务（以管理员身份运行），开机自动启动。
  > 注意：
  > 1. 需要修改`conf\wrapper.conf`文件，将`wrapper.java.command=java`改为全路径，例如`wrapper.java.command=E:\dev-tools\java\bin\java`，否则服务启动时报无法找到`java`命令，启动失败；
  > 2. 在`conf\wrapper.conf`中配置Mycat JVM启动参数；

##### Mac/Linux环境部署
```sh
bin/mycat start      # 启动
bin/mycat stop       # 停止
bin/mycat console    # 前台运行
bin/mycat status     # 查看启动状态
```

#### Mycat管理
Mycat启动之后，`8066`为数据端口，`9066`为管理端口，不能操作数据，只能执行Mycat管理命令。连接Mycat使用`server.xml`文件中定义的用户名和密码。
> Mac环境连接Mycat必须指定TCP协议，否则会直接连接mysql的3306端口而不是Mycat，没有任何错误信息：
> ```sh
> mysql -h localhost -P 8066 -uroot -p --protocol=TCP
> mysql -h localhost -P 9066 -uroot -p --protocol=TCP
> ```

管理端口登录，通过`show @@help;`查看Mycat提供的管理命令，例如`show @@datasource;`：
```
+----------+-----------+-------+-----------+------+------+--------+------+------+---------+-----------+------------+
| DATANODE | NAME      | TYPE  | HOST      | PORT | W/R  | ACTIVE | IDLE | SIZE | EXECUTE | READ_LOAD | WRITE_LOAD |
+----------+-----------+-------+-----------+------+------+--------+------+------+---------+-----------+------------+
| dn1      | localhost | mysql | localhost | 3306 | W    |      0 |    5 |    5 |     623 |         0 |          0 |
| dn0      | localhost | mysql | localhost | 3306 | W    |      0 |    5 |    5 |     623 |         0 |          0 |
| dn3      | 127.0.0.1 | mysql | 127.0.0.1 | 3306 | W    |      0 |    5 |    5 |     623 |         0 |          0 |
| dn2      | localhost | mysql | localhost | 3306 | W    |      0 |    5 |    5 |     623 |         0 |          0 |
| dn4      | 127.0.0.1 | mysql | 127.0.0.1 | 3306 | W    |      0 |    5 |    5 |     623 |         0 |          0 |
+----------+-----------+-------+-----------+------+------+--------+------+------+---------+-----------+------------+
```

#### 备注
- Mycat的全局序列不太方便的地方：插入数据后未找到有效获取本次生成的sequence值的方法；
- Mycat 2.0在开发中，参考[Mycat2](https://github.com/MyCATApache/Mycat2) <br />
  从新特性来看，结果集缓存、自动集群管理、支持负载均衡等主要特性没有必要由Mycat管理，使用第三方即可。
- 简单性能对比测试 <br />
  Mac book pro，单机测试，50并发线程，对相同的业务逻辑功能（使用手机号+密码注册会员）进行测试，TPS指被测试业务逻辑的每秒执行次数（包含`select from usr_user_account` + `insert into usr_user` + `insert into usr_user_account`）：
  - Mycat + MyBatis，分片: TPS在2200上下波动；
  - MyBatis，不分片: TPS在2600上下波动；
  - 纯JDBC，不分片: TPS在3400上下波动；<br />
  单机测试，Mycat server的CPU占用对测试结果有一定影响。<br />
  从结果看中间加一层mycat后性能有一定下降，但幅度不大，不及MyBatis与原生JDBC之间的差异。
- 分片方案：算法选择充分考虑扩容时简化数据迁移、避免高并发插入时的热点、避免XA事物；