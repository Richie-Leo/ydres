#### 环境说明
- Hadoop: 2.9.2
- Hive: 2.3.4
- Sqoop: sqoop-1.4.7.bin__hadoop-2.6.0 <br />
  手工将`mysql-connector-java-8.0.12.jar`拷贝到Sqoop的lib目录中

MySQL表结构：
```sql
CREATE TABLE `otter_user.user` (
  `uid` int(11) NOT NULL AUTO_INCREMENT,
  `email1` varchar(30) NOT NULL DEFAULT '',
  `mobile1` varchar(20) NOT NULL DEFAULT '',
  `created_at` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP,
  `updated_at` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (`uid`)
) ENGINE=InnoDB AUTO_INCREMENT=11 DEFAULT CHARSET=utf8
```

```bash
# 在Hive创建测试库：
CREATE DATABASE ods_test COMMENT 'ODS for test cases' LOCATION '/test/ods';
# 导入过程中遇到某些错误时，查看hdfs目录文件情况：
bin/hdfs dfs -ls /your/path
bin/hdfs dfs -rm -r /your/path
```

#### 使用Sqoop增量导入
1. 清空`otter_user.user`表，在其中insert一些数据；
2. 在Sqoop主目录建立文件`connection-param.properties`，使用Sqoop导入时从该文件读取数据库连接参数：
   ```properties
   useTimezone = true
   serverTimezone = Asia/Shanghai
   characterEncoding=utf8
   zeroDateTimeBehavior=CONVERT_TO_NULL
   useSSL = false
   ```
   - 示例使用MySQL Java Connector 8.0.12，若未指定`zeroDateTimeBehavior`参数值，sqoop默认使用`convertToNull`，低版本的MySQL Java Connector支持这个配置值，高版本改为了`CONVERT_TO_NULL`；
   - ==时区问题==。本地MySQL使用了Mac OSX的系统时区`Asia/Shanghai`，如果不在JDBC连接上指定`serverTimezone`参数，Java将按标准时区处理datetime类型，导致抽取到Hive的日期时间与MySQL相差几个小时；
3. 将MySQL全量数据初始导入Hive：
   ```bash
   bin/sqoop import --connect jdbc:mysql://127.0.0.1:3306/otter_user --connection-param-file connection-param.properties --username root -P --table user --hive-import --hive-overwrite --hive-table ods_test.user --map-column-hive created_at=TIMESTAMP,updated_at=TIMESTAMP --fields-terminated-by ',' --hive-drop-import-delims --null-string '\\N' --null-non-string '\\N' -m 1
   ```
   查看Hive表数据如下：
   ```bash
   hive> select * from `user`;
   1	test1@qq.com	18600000001	2019-05-04 16:15:28	2019-05-04 16:15:28
   2	test2@qq.com	11111111111	2019-05-04 16:15:35	2019-05-05 14:32:32
   4	test3@qq.com	33333333333	2019-05-04 16:15:55	2019-05-05 14:32:44
   5	test3@gmail.com	18666666665	2019-05-04 16:16:02	2019-05-04 16:16:02
   6	test4@gmail.com	18666666667	2019-05-04 16:16:13	2019-05-04 16:16:13
   7	test5@gmail.com	18666666668	2019-05-04 16:16:19	2019-05-04 16:16:19
   ```
4. 在`otter_user.user`中insert、update一些数据；
5. 将MySQL增量数据导入Hive：
   ```bash
   bin/sqoop import --connect jdbc:mysql://127.0.0.1:3306/otter_user --connection-param-file connection-param.properties --username root -P --table user --target-dir '/test/ods/user' --incremental lastmodified --last-value '2019-05-05' --check-column updated_at --merge-key uid --map-column-hive created_at=TIMESTAMP,updated_at=TIMESTAMP --fields-terminated-by ',' --null-string '\\N' --null-non-string '\\N' -m 1
   ```
   查看Hive表数据如下：
   ```bash
   hive> select * from `user` order by uid;
   1	test1@qq.com	18600000001	2019-05-04 16:15:28	2019-05-04 16:15:28
   2	test2@qq.com	00000000000	2019-05-04 16:15:35	2019-05-05 16:20:48
   4	test3@qq.com	1777671812	2019-05-04 16:15:55	2019-05-05 16:21:21
   5	test3@gmail.com	18666666665	2019-05-04 16:16:02	2019-05-04 16:16:02
   6	test4@gmail.com	18666666667	2019-05-04 16:16:13	2019-05-04 16:16:13
   7	test5@gmail.com	18666666668	2019-05-04 16:16:19	2019-05-04 16:16:19
   13	aa1@sina.com	13811111111	2019-05-05 16:20:06	2019-05-05 16:20:06
   14	aa2@sina.com	13822222222	2019-05-05 16:20:14	2019-05-05 16:20:14
   ```

> ==若不使用`--map-column-hive`为`created_at`、`updated_at`指定Hive数据类型，默认情况下Sqoop将MySQL的日期和时间戳导入Hive后使用String类型==。
>    ```bash
>    hive> desc `user`;
>    uid                 	int     
>    email1              	string     
>    mobile1             	string     
>    created_at          	timestamp     
>    updated_at          	timestamp
>    ```

##### 增量导入参数说明
- `--incremental (mode)`：增量模式，`append`或`lastmodified`；
- `--check-column `：增量判断字段，字段类型不能为任何char、varchar，可以是自增字段、时间戳等；
- `--last-value `：上次导入时的最大值；
- `--merge-key`：增量数据合并时判断唯一性的主键，一般在增量模式为`lastmodified`时使用，Sqoop启动一个MR job，将增量数据合并到老的数据中，写入HDFS `--target-dir`路径；

> ==Sqoop 1.4.6开始不再支持`--hive-import`和`--incremental lastmodified`组合，上面示例使用`--target-dir`和`--incremental lastmodified`实现增量导入==，参考[sqoop incremental import to hive table
](https://stackoverflow.com/questions/47264844/sqoop-incremental-import-to-hive-table)。

##### Saved Jobs
通过`sqoop job`管理Sqoop任务。<br />
Job定义默认保存在本地`$HOME/.sqoop/`，可以配置为远程仓库方式。

对于增量Job，如果从命令行手工执行，则`--last-value`值每次也得手工提供；如果在saved job中执行，每次执行的`--last-value`值会保存在仓库中，下次执行时Sqoop自动处理以实现增量逻辑。

远程仓库通过`sqoop metastore`命令启动一个Sqoop服务，使用HSQLDB数据库作为存储。Sqoop Job节点通过`sqoop-site.xml`或者`--meta-connect`指定远程仓库连接信息。

#### 手工增量导入
1. 将MySQL全量数据初始导入Hive表`ods_test.user_all`：
   ```bash
   bin/sqoop import --connect jdbc:mysql://127.0.0.1:3306/otter_user --connection-param-file connection-param.properties --username root -P --table user --hive-import --hive-overwrite --hive-table ods_test.user_all --map-column-hive created_at=TIMESTAMP,updated_at=TIMESTAMP --fields-terminated-by ',' --hive-drop-import-delims --null-string '\\N' --null-non-string '\\N' -m 1
   ```
2. 在`otter_user.user`中insert、update一些数据；
3. 将MySQL增量数据导入Hive表`ods_test.user_updated`：
   ```bash
   bin/sqoop import --connect jdbc:mysql://127.0.0.1:3306/otter_user --connection-param-file connection-param.properties --username root -P --table user --where "updated_at>='2019-05-08'" --hive-import --hive-overwrite --hive-table ods_test.user_updated --map-column-hive created_at=TIMESTAMP,updated_at=TIMESTAMP --fields-terminated-by ',' --hive-drop-import-delims --null-string '\\N' --null-non-string '\\N' -m 1
   ```
4. 将增量数据合并到全量数据中：
   ```sql
   INSERT OVERWRITE TABLE ods_test.user_all SELECT * FROM (
      SELECT i.uid, i.email1, i.mobile1, i.created_at, i.updated_at
      FROM ods_test.user_updated AS i
      UNION
      SELECT f.uid, f.email1, f.mobile1, f.created_at, f.updated_at
      FROM ods_test.user_all AS f
      LEFT JOIN ods_test.user_updated AS i on f.uid=i.uid
      WHERE i.uid is null
   ) AS t
   ```
