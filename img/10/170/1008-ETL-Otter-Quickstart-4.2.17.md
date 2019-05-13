#### 环境准备
本示例使用otter的单机房复制方案：<br />
<img src="https://richie-leo.github.io/ydres/img/10/120/1011/example-architecture.jpeg" style="max-width:550px;width:99%;" />

使用2个源库(2个不同的MySQL实例)，目标库为1个MySQL实例中的2个schema，源与目标的表结构一一对应，没有任何转换处理：
```
graph TD
   A[:3307 mshop_order] --> |Canal| C(otter)
   B[:3308 mshop_user] --> |Canal| C(otter)
   C(otter) --> D[:3306 otter_order]
   C(otter) --> E[:3306 otter_user]
```

目标库`localhost:3306`：
```sql
CREATE SCHEMA `otter_order` DEFAULT CHARACTER SET utf8 ;
CREATE TABLE `ord_order` (
  `oid` int NOT NULL DEFAULT 0,
  `uid` int NOT NULL DEFAULT 0,
  `total` DECIMAL(10,2) NOT NULL DEFAULT 0,
  `payment` DECIMAL(10,2) NOT NULL DEFAULT 0,
  `contact` VARCHAR(30) NOT NULL DEFAULT '',
  `phone` VARCHAR(20) NOT NULL DEFAULT '',
  `address` VARCHAR(70) NOT NULL DEFAULT '',
  PRIMARY KEY (`oid`)
) ENGINE = InnoDB DEFAULT CHARACTER SET = utf8 COLLATE=utf8_general_ci;
CREATE TABLE `ord_item` (
  `oiid` int NOT NULL DEFAULT '0',
  `oid` int NOT NULL DEFAULT '0',
  `sku` varchar(20) NOT NULL DEFAULT '',
  `title` varchar(70) NOT NULL DEFAULT '',
  `price` decimal(8,2) NOT NULL DEFAULT '0.00',
  `quantity` int(11) NOT NULL DEFAULT '0',
  PRIMARY KEY (`oiid`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

CREATE SCHEMA `otter_user` DEFAULT CHARACTER SET utf8 ;
CREATE TABLE `user` (
  `uid` int(11) NOT NULL AUTO_INCREMENT,
  `email` varchar(30) NOT NULL DEFAULT '',
  `mobile` varchar(20) NOT NULL DEFAULT '',
  `created_at` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`uid`)
) ENGINE=InnoDB DEFAULT CHARACTER SET=utf8 COLLATE=utf8_general_ci;
```

源库1 `localhost:3307`：库名`mshop_order`，表结构同目标库`otter_order`；<br />
源库2 `localhost:3308`：库名`mshop_user`，表结构同目标库`otter_user`；

在两个源库上分别为`mshop_order`、`mshop_user`库开启MySQL binlog (`binlog-do-db`、`binlog-ignore-db`配置)，格式ROW。

两个源库上面都创建canal用户：
```sql
CREATE USER canal IDENTIFIED BY 'canal';  
GRANT SELECT, REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'canal'@'%';
FLUSH PRIVILEGES;
```

#### Otter Manager部署
1. 创建、初始化Manager管理库，这里使用目标库`localhost:3306`：
   1. 下载：
      ```bash
      wget https://raw.github.com/alibaba/otter/master/manager/deployer/src/main/resources/sql/otter-manager-schema.sql
      ```
   2. 在MySQL中执行：`source otter-manager-schema.sql` <br />
      执行过程可能会报错：
      ```
      ERROR 1067 (42000): Invalid default value for 'GMT_CREATE'
      ```
      原因：5.6版本以后不允许将TIMESTAMP默认值设为`0000 00-00 00:00:00`<br />
      解决方案：
      ```sql
      /* 查看SQL_MODE */
      show session variables like '%sql_mode%';
      /* 去掉NO_ZERO_IN_DATE, NO_ZERO_DATE */
      set sql_mode="ONLY_FULL_GROUP_BY,STRICT_TRANS_TABLES,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION";
      /* 重新执行manager元数据表创建SQL */
      source otter-manager-schema.sql
      ```
2. 准备Zookeeper：`localhost:2181`
3. 下载[otter manager](https://github.com/alibaba/otter/releases/download/otter-4.2.17/manager.deployer-4.2.17.tar.gz)，解压；
4. 配置otter manager `conf/otter.properties`：
   ```bash
   otter.domainName = 127.0.0.1
   otter.port = 8080
   otter.jetty = jetty.xml
   # otter manager库
   otter.database.driver.class.name = com.mysql.jdbc.Driver
   otter.database.driver.url = jdbc:mysql://127.0.0.1:3306/otter
   otter.database.driver.username = root
   otter.database.driver.password = dev
   otter.communication.manager.port = 1099
   otter.zookeeper.cluster.default = 127.0.0.1:2181
   # 其它配置项使用默认值
   ```
5. 管理：
   ```bash
   bin/startup.sh   # 启动manager
   bin/stop.sh      # 停止manager
   tail -f logs/manager.log # 监控manager日志
   ```
   访问[http://127.0.0.1:8080](http://127.0.0.1:8080/login.htm)，用户密码均为`admin`

#### Otter Node部署
1. Otter Manager设置
   1. 登录Otter Manager，在`机器管理->Zookeeper管理`中，添加Zookeeper集群设置；
   2. 在`机器管理->Node管理`中添加一个node节点设置：<br />
      <img src="https://richie-leo.github.io/ydres/img/10/120/1011/otter-node-config.png" style="max-width:360px;width:99%;" /> 
      - 机器名称：随意定义；
      - 机器IP：多机器集群部署时，别使用`127.0.0.1`，否则无法识别node节点；<br />
   3. 在`机器管理->Node管理`列表页面得到`node id`，简称`nid`；<br />
      <img src="https://richie-leo.github.io/ydres/img/10/120/1011/otter-node-nid.png" style="max-width:700px;width:99%;" />
2. Node节点进行跨机房传输时，会使用到HTTP多线程传输技术，目前主要依赖aria2c做为下载客户端，后续会推出java版本。[aria2主页](https://aria2.github.io/)，Mac安装`brew install aria2`。
   > Mac编译安装`aria2`参考[aria2/makerelease-osx.mk](https://github.com/aria2/aria2/blob/master/makerelease-osx.mk)：
   > 1. [下载aria2-1.34.0.tar.gz](https://github.com/aria2/aria2/releases/download/release-1.34.0/aria2-1.34.0.tar.gz)，解压；
   > 2. 在`aria2`源码目录执行`autoconf`，由`configure.ac`生成`configure`，若未安装[GNU autoconf](https://www.gnu.org/software/autoconf/autoconf.html)，Mac安装：`brew install autoconf` (`autoconf`依赖[GNU M4](https://www.gnu.org/software/m4/m4.html)，Mac系统自带了M4)；
   > 3. Mac安装`sphinx-doc`：`brew install sphinx-doc`，其它系统安装参考[Installing Sphinx](http://www.sphinx-doc.org/en/master/usage/installation.html)；
   > 4. 编译安装`aria2`：
   >    ```bash
   >    export NON_RELEASE=1
   >    make -f makerelease-osx.mk
   >    ```
   > 5. 将`aria2c`添加到PATH中；
3. 下载[otter node](https://github.com/alibaba/otter/releases/download/otter-4.2.17/node.deployer-4.2.17.tar.gz)，解压；
4. `nid`配置：`echo 1 > conf/nid`；
5. `otter.properties`配置：
   ```bash
   otter.nodeHome = /Users/richie-home/Documents/workspace/otter/node/data
   otter.manager.address = 127.0.0.1:1099
   # 其它属性使用默认值
   ```
6. 管理：
   ```bash
   bin/startup.sh   # 启动node
   bin/stop.sh      # 停止node
   tail -f logs/node/node.log # 监控node日志
   ```

#### 管理数据同步任务
1. `配置管理->数据源配置`：对应3个MySQL实例3306、3307、3308分别建立1个数据源：<br />
   <img src="https://richie-leo.github.io/ydres/img/10/120/1011/otter-cfg-datasources.png" style="max-width:700px;width:99%;" />
2. `配置管理->数据表配置`：为每个源库定义一个数据表配置，为目标库的每个schema定义一个数据表配置<br />
   <img src="https://richie-leo.github.io/ydres/img/10/120/1011/otter-cfg-tables.png" style="max-width:700px;width:99%;" />
3. `配置管理->canal配置`：对应每个源库定义1个canal配置<br />
   <img src="https://richie-leo.github.io/ydres/img/10/120/1011/otter-cfg-canal.png" style="max-width:700px;width:99%;" /> <br />
   ==一个canal只能在一个Pipeline中使用==。Canal是otter的子项目，目前otter4开源版本只能使用otter自带的canal，不能自定义部署Canal给otter使用。
   - 位点自定义设置：`{"journalName":"mysql-bin.000006","position":154,"timestamp":0};`，查询对应源库的位点信息`show master status`；
   - 过滤表达式：`.*\\..*`
4. `同步管理->Channel管理`：对应每个源库建立1个Channel：<br />
   <img src="https://richie-leo.github.io/ydres/img/10/120/1011/otter-cfg-channel.png" style="max-width:700px;width:99%;" /> <br />
   ==一个Channel对应一个数据同步任务== <br />
   - 同步一致性：
     - 基于数据库反查：用binlog中的ID反查源库得到最新数据，以此同步到目标库；
     - 基于当前日志变更：仅根据当前批次获取的binlog日志得到最新数据；
   - 同步模式：
     - 行模式：如果目标库不存在记录时，执行插入；
     - 列模式：主要是变更哪个字段，只会单独修改该字段，在双Ａ同步时，为减少数据冲突，建议选择列模式；
5. `同步管理->Channel管理->Pipleline管理`：在每个Channel下面建立1个Pipleline：<br />
   <img src="https://richie-leo.github.io/ydres/img/10/120/1011/otter-cfg-pipeline-1.png" style="max-width:700px;width:99%;" /> <br />
   <img src="https://richie-leo.github.io/ydres/img/10/120/1011/otter-cfg-pipeline-2.png" style="max-width:700px;width:99%;" /> <br />
   ==单向同步在Channel下面建立1个Pipeline，双向同步在Channel下建立2个Pipeline==。
6. `同步管理->Channel管理->Pipleline管理->映射关系列表`：在每个Pipeline下面建立1个映射关系 <br />
   <img src="https://richie-leo.github.io/ydres/img/10/120/1011/otter-cfg-table-mappings-1.png" style="max-width:700px;width:99%;" /> <br />
   <img src="https://richie-leo.github.io/ydres/img/10/120/1011/otter-cfg-table-mappings-2.png" style="max-width:700px;width:99%;" /> 

经过上面配置后，即可启动每个channel。

#### 测试验证
##### 数据验证
在源库`:3307 mshop_order`：
```sql
insert into ord_order (oid, uid, total, payment, contact, phone, address) 
   values(1, 1, 100, 100, '刘先生', '13688888888', '武汉市东湖高新技术开发区');
insert into ord_order (oid, uid, total, payment, contact, phone, address) 
   values(2, 1, 989.00, 989.00, '刘先生', '13633333333', '北京市海淀区丹棱18号街创富大厦');
insert into ord_item(oiid, oid, sku, title, price, quantity) 
   values(1, 1, 'MU9810', 'Mac Book Pro 13#', 13888, 1);
```

在源库`:3308 mshop_user`：
```sql
insert into user(email, mobile) values('test1@qq.com', '13611111111');
insert into user(email, mobile) values('test2@qq.com', '13622222222');
insert into user(email, mobile) values('test3@qq.com', '13633333333');
```

查看目标库`:3306 otter_order`、`:3306 otter_user`，数据是否已经同步过来。若未同步，在Otter Manager的`Channel管理 > Pipeline管理 > 日志记录`中查看错误信息。

在源库执行`truncate`，目标库正确同步。

##### Schema变更验证
1. 修改字段名、字段类型，添加删除字段，添加删除索引，都能正确同步到目标库：
   ```sql
   /* 修改字段名 */
   ALTER TABLE `user`  CHANGE COLUMN `mobile` `mobile1` VARCHAR(20) NOT NULL DEFAULT '';
   ALTER TABLE `user` CHANGE COLUMN `email` `email1` VARCHAR(30) NOT NULL DEFAULT '' ;
   /* 添加新字段 */
   ALTER TABLE `user` ADD COLUMN `birthday` DATE NOT NULL DEFAULT '1900-01-01' AFTER `mobile1`;
   /* 修改字段类型 */
   ALTER TABLE `user` CHANGE COLUMN `birthday` `birthday` VARCHAR(10) NOT NULL DEFAULT '1900-01-01' ;
   /* 表结构修改之后的数据验证 */
   insert into user(email1, mobile1, birthday) values('test@sina.com', '18655555555', '2001-01-01');
   /* 尝试添加索引 */
   ALTER TABLE `user` ADD INDEX `ix_created` (`created_at` ASC);
   /* 删除字段和索引 */
   ALTER TABLE `user` DROP COLUMN `birthday`;
   ALTER TABLE `user` DROP INDEX `ix_created` ;
   ```
2. 添加、删除表，正确同步到目标库：
   ```sql
   CREATE TABLE `test` (
    `id` int(11) NOT NULL,
    `name` varchar(30) NOT NULL DEFAULT ''
   ) ENGINE=InnoDB DEFAULT CHARACTER SET=utf8 COLLATE=utf8_general_ci;
   drop table test;
   ```
3. 源、目标表结构差异验证 <br />
   在目标库`:3306 otter_user`添加时间戳字段：
   ```sql
   ALTER TABLE `user` ADD COLUMN `updated_at` TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP AFTER `created_at`;
   ```
   在源库`:3308 mshop_user`执行数据插入更新操作：
   ```sql
   insert into user(email1, mobile1, birthday) values('test@sina.com', '18655555555', '2001-01-01');
   ```
   目标库数据同步正常，timestamp值正常。
4. 无主键的表 <br />
   ==无主键的表，DDL语句可以正确同步到目标库，数据同步时会导致otter任务挂起==。
   - 创建一个无主键表test，插入数据，从Otter Manager的Channel管理可以看到同步任务挂起了，可以查看异常信息；
   - 停止同步任务，在`配置管理->数据表配置`中将mshop_user的配置值由`.*`改为`(?!.*?test).*`，排除掉test表，重新启动同步任务，恢复正常；<br />
     若不排除test表，同步任务将无法启动；

#### Otter注意事项
1. Otter只支持ROW模式的数据同步，其他两种模式不支持；
2. 源库只支持MySQL,目标库支持MySQL和Oracle；
3. ==同步的表必须有主键，无主键表如果出现重复记录的话,同步会导致数据错乱==；
4. ==不支持带外键的记录同步 (Load算法会打散事务，进行并行处理，会导致外键约束无法满足)==；
5. ==数据库上慎重Trigger，如果源库、目标库都有trigger，Otter同步后可能会造成多次更新，数据一致性没法保证==；
6. ==支持部分ddl同步== (支持create table / drop table / alter table / truncate table / rename table / create index / drop index，其他类型的暂不支持，比如grant,create user,trigger等等)，==同时ddl语句不支持幂等性操作==，所以出现重复同步时，会导致同步挂起，可通过配置高级参数:跳过ddl异常，来解决这个问题；

参考[Canal & Otter 的一些注意事项和最佳实践](https://my.oschina.net/dxqr/blog/524795)

#### Otter的恢复
Otter同步遇到异常时，Channel会被挂起，例如前面对无主键表的操作。造成异常的原因没有排除掉，任务无法启动。因为一旦任务启动，仍然从有问题的binlog位点位置开始，发生异常后继续挂起。

如果无法优雅的解决异常原因，可以考虑暴力恢复：
1. 像前面提到的，在数据表配置中过滤掉导致异常的表；
2. 将Otter当前正在处理的binlog位点后移，跳过异常位点。<br />
   Otter将位点信息保存在Zookeeper中，例如上面示例中，`canal-3308`的位点信息保存路径为：`/otter/canal/destinations/canal-3308/2/cursor`，详细信息为：
   ```zk
   [zk: localhost:2181(CONNECTED) 14] get /otter/canal/destinations/canal-3308/2/cursor
   {"@type":"com.alibaba.otter.canal.protocol.position.LogPosition","identity":{"slaveId":-1,"sourceAddress":{"address":"localhost","port":3308}},"postion":{"gtid":"","included":false,"journalName":"mysql-bin.000008","position":5049,"serverId":2,"timestamp":1557417382000}}
   ```
   可以更新`position`值，将位点后移，跳过导致异常的binlog条目：
   ```zk
   set /otter/canal/destinations/canal-3308/2/cursor {"@type":"com.alibaba.otter.canal.protocol.position.LogPosition","identity":{"slaveId":-1,"sourceAddress":{"address":"localhost","port":3308}},"postion":{"gtid":"","included":false,"journalName":"mysql-bin.000008","position":5433,"serverId":2,"timestamp":1557417382000}}
   ```
   更新位点后重新启动Channel。<br />
   注意：暴力恢复要考虑数据修复问题，可以对相关表做一个全量导入，再恢复同步。==手工调整位点时要注意Otter的同步原理，DML的binlog重复执行基本没问题(前提是中间没有加入过多的自定义处理)，但DDL的重复执行很可能不具备幂等性，报错等==。

#### Otter用于ETL的问题和解决方案
将Otter用于ETL，需要解决以下场景：
- 仅同步部分表 <br />
  需要同步的表数量比较多，排除掉不同步的表数量也比较多，通过Otter正则过滤不太方便实现。业务系统有些表，或者运维目的创建的一些临时表，不符合Otter同步要求的，会导致Otter挂起。<br />
  - 需求上，只需要同步部分表；
  - 有些临时表、备份表无需同步；
  - 不符合Otter同步要求的，例如没有主键的表不能同步；
- 物理删除 <br />
  业务系统存在物理删除的情况，希望在Otter目标库上改为逻辑删除 (如果业务系统delete后重新insert会有问题)。

使用上面示例中的`channel-3308`进行测试：
1. 先清空源库中的`user`表；
2. 为目标库`user`表添加`del_flag`逻辑删除字段：
   ```sql
   ALTER TABLE `user` ADD COLUMN `del_flag` INTEGER NOT NULL DEFAULT 0 AFTER `mobile`;
   ```
3. 在node节点的conf目录添加文件`sync-tables.txt`：
   ```bash
   user
   test_1
   ```
4. 停用`channel-3308`，编辑映射关系，Event Processor选择SOURCE，内容如下：
   ```java
	package org.liuzhibin.research.otter;
	
	import java.io.BufferedReader;
	import java.io.File;
	import java.io.FileReader;
	import java.sql.Types;
	import java.util.Arrays;
	import java.util.HashSet;
	import java.util.Set;
	
	import com.alibaba.otter.node.extend.processor.AbstractEventProcessor;
	import com.alibaba.otter.shared.etl.model.EventColumn;
	import com.alibaba.otter.shared.etl.model.EventData;
	import com.alibaba.otter.shared.etl.model.EventType;
	
	public class TestEventProcessor extends AbstractEventProcessor {
		// 配置需要同步的表清单
		private Set<String> syncTables = new HashSet<>();
		public TestEventProcessor() {
			System.out.println("[event-processor] Sync tables:");
			try {
				// 读取配置，加载需要同步的表清单
				// sync-tables.txt: 放在node节点的conf目录中
				FileReader fr = new FileReader(new File(this.getClass().getClassLoader().getResource("sync-tables.txt").toURI()));
				BufferedReader bf = new BufferedReader(fr);
				String table;
				while ((table = bf.readLine()) != null) {
					table = table.trim().toLowerCase();
					if(table.isEmpty()) continue;
					syncTables.add(table);
					System.out.println("[event-processor]     " + table);
				}
				bf.close();
				fr.close();
			} catch(Exception ex) {
				System.out.println("[event-processor] Initialization error: " + ex.getMessage());
				return;
			}
			System.out.println("[event-processor] Initialized");
		}
		/**
		 * @return true:该事件同步到目标库；false:该事件不同步到目标库。
		 */
		@Override
		public boolean process(EventData event) {
			System.out.println("[event-processor] Event: " + event.getEventType() + " on " + event.getSchemaName() + "." + event.getTableName());
			//未配置的表，不同步
			if(event.getEventType().isDml() && !syncTables.contains(event.getTableName().toLowerCase())) {
				System.out.println("[event-processor] Non sync table: " + event.getSchemaName() + "." + event.getTableName() + ", ignored");
				return false;
			}
			switch(event.getEventType()) {
				case DELETE: //删除记录
					//物理删除改为逻辑删除：在目标库的表中添加逻辑删除字段del_flag，源库执行delete语句时改为在目标库更新del_flag值
					//某些情况下，业务系统先执行delete再重新insert，则逻辑删除方案会导致错误，通用解决方案是将物理删除数据额外记录到另一个表中，
					//  实现方式可以直接在EventProcessor中执行数据库插入操作，或者向MQ发送消息异步操作。
					event.setEventType(EventType.UPDATE); //事件类型由delete修改为update
					//将del_flag更新为1
					EventColumn delFlag = new EventColumn();
					delFlag.setColumnValue("1");
					delFlag.setColumnType(Types.INTEGER);
					delFlag.setColumnName("del_flag");
					delFlag.setUpdate(true);
					event.setColumns(Arrays.asList(delFlag));
					//打印日志
					StringBuilder sb = new StringBuilder();
					for(EventColumn col : event.getKeys()) {
						if(sb.length()>0) sb.append(", ");
						sb.append(col.getColumnName()).append("=").append(col.getColumnValue());
					}
					System.out.println("[event-processor] Rewrite: " + event.getSchemaName() + "." + event.getTableName() + ": " + sb.toString() + ", DELETE->UPDATE");
					return true;
				case TRUNCATE: //貌似在源表上执行truncate操作时，Otter触发2个事件，先ERASE，然后TRUNCATE?
				case ERASE:
				default: return true;
			}
		}
	}
   ```
5. 重启node节点；

可以在源库上操作`user`表，创建`test_1`并操作数据等，`node/logs/node/node.log`的输出如下：
```bash
[event-processor] Sync tables:
[event-processor]     user
[event-processor]     test_1
[event-processor] Initialized
[event-processor] Event: TRUNCATE on mshop_user.user
[event-processor] Event: INSERT on mshop_user.user
[event-processor] Event: INSERT on mshop_user.user
[event-processor] Event: DELETE on mshop_user.user
[event-processor] Rewrite: mshop_user.user: uid=1, DELETE->UPDATE
[event-processor] Event: CREATE on mshop_user.test_1
[event-processor] Event: INSERT on mshop_user.test_1
[event-processor] Event: CREATE on mshop_user.test_2
```

测试结果 & 注意事项：
1. Otter是基于Spring的，但EventProcessor本身不受Spring管理；
2. ==更改了EventProcessor，需要重启node节点== (不是在manager管理界面中停用/启用)，否则新的EventProcessor代码不会生效；
3. ==没有主键的表，无法在EventProcessor中来判断、过滤，因为执行流程在EventProcessor介入前已经抛异常了==；<br />
   解决方案：
   1. 明确需要同步的表清单，在Canal的过滤表达式中用正则过滤。缺点：正则表达式会比较复杂或者内容比较大；
   2. 修改Otter源码，可以考虑将无主键的情况延迟到EventProcessor中来处理。<br />
      抛异常的位置在[MessageParser.java](https://github.com/alibaba/otter/blob/master/node/etl/src/main/java/com/alibaba/otter/node/etl/select/selector/MessageParser.java)的591行 (搜`this rowdata has no pks`)；
4. 新增了业务表，如下流程添加到同步任务中来：
   1. 在Manager中停用相应的Channel；
   2. 在源库上全量导出新增表数据；
   3. 在目标库上全量导入新增表数据；
   4. 将表名添加到配置中；
   5. 重启node节点，在Manager中启用相应Channel；

#### 深入理解Otter
1. [项目主页alibaba/otter](https://github.com/alibaba/otter)、
   [Introduction](https://github.com/alibaba/otter/wiki/Introduction)
2. [Manager配置介绍](https://github.com/alibaba/otter/wiki/Manager%E9%85%8D%E7%BD%AE%E4%BB%8B%E7%BB%8D)、
   [Manager使用介绍](https://github.com/alibaba/otter/wiki/Manager%E4%BD%BF%E7%94%A8%E4%BB%8B%E7%BB%8D)、
   [映射规则配置](https://github.com/alibaba/otter/wiki/%E6%98%A0%E5%B0%84%E8%A7%84%E5%88%99%E9%85%8D%E7%BD%AE)
3. [Otter调度模型](https://github.com/alibaba/otter/wiki/Otter%E8%B0%83%E5%BA%A6%E6%A8%A1%E5%9E%8B)、
   [Otter数据入库算法](https://github.com/alibaba/otter/wiki/Otter%E6%95%B0%E6%8D%AE%E5%85%A5%E5%BA%93%E7%AE%97%E6%B3%95)、
   [Otter双向回环控制](https://github.com/alibaba/otter/wiki/Otter%E5%8F%8C%E5%90%91%E5%9B%9E%E7%8E%AF%E6%8E%A7%E5%88%B6)、
   [Otter数据一致性](https://github.com/alibaba/otter/wiki/Otter%E6%95%B0%E6%8D%AE%E4%B8%80%E8%87%B4%E6%80%A7)、
   [Otter高可用性](https://github.com/alibaba/otter/wiki/Otter%E9%AB%98%E5%8F%AF%E7%94%A8%E6%80%A7)、
   [Otter扩展性](https://github.com/alibaba/otter/wiki/Otter%E6%89%A9%E5%B1%95%E6%80%A7)
4. [Otter使用介绍 (偏向使用层面)](https://docs.google.com/presentation/d/1Yrc6CfHT1raWPqWQOENDT3pdTgVC0ADoNipbrmDm418/edit?usp=sharing)、
   [深入理解otter (偏向技术层面)](https://docs.google.com/presentation/d/1FDrfVEoNV7AlPOrgXYKo2L_70NDizhwrMpfCG0vztf8/edit?usp=sharing)
