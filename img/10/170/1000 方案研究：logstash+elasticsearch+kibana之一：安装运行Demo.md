#### 安装运行elasticsearch

参考：[Installing and Running Elasticsearch](https://www.elastic.co/guide/en/elasticsearch/guide/current/running-elasticsearch.html)
1. 下载：[Download Elasticsearch](https://www.elastic.co/downloads/elasticsearch)
2. 解压：`tar -xvzf elasticsearch-2.3.5.tar.gz`
3. 配置：通过环境变量`ES_HEAP_SIZE`设置elasticsearch堆内存，配置`config/elasticsearch.yml`
   ```
    cluster.name: my-es
    node.name: node-1
    path.data: /Users/richie/Documents/workspace/bigdata/elasticsearch/elasticsearch-2.3.5/data
    path.logs: /Users/richie/Documents/workspace/bigdata/elasticsearch/elasticsearch-2.3.5/log
    bootstrap.mlockall: false
    network.host: 0.0.0.0
    http.port: 9200
    node.max_local_storage_nodes: 1
    action.destructive_requires_name: true
    ```
4. 启动：`bin/elasticsearch`
5. 验证：http://127.0.0.1:9200/

#### 安装运行logstash

1. 下载：[Download Logstash](https://www.elastic.co/downloads/logstash)
2. 解压：`tar -xvzf logstash-2.3.4.tar.gz`
3. 配置：运行demo的配置如下：
   1. 建立`jdbc-demo.conf`，内容如下：
    ```
    input {
        jdbc {
            jdbc_connection_string => "jdbc:mysql://localhost:3306/my_blog"
            jdbc_user => "root"
            jdbc_password => "dev"
            jdbc_driver_library => "/Users/richie/Documents/workspace/bigdata/mysql-jdbc/mysql-connector-java-5.1.39-bin.jar"
            jdbc_driver_class => "com.mysql.jdbc.Driver"
            statement_filepath => "/Users/richie/Documents/workspace/bigdata/logstash/logstash-2.3.4/jdbc-demo.sql"
        }
    }
    output {
        elasticsearch {
            hosts => "127.0.0.1:9200"
            index => "myblog"
            document_type => "article"
        }
        file {
            path => "/Users/richie/Documents/workspace/bigdata/logstash/logstash-2.3.4/jdbc-output.json"
        }
    }
    ```
   2. 建立`jdbc-demo.sql`，内容如下：
    ```sql
    select a.article_id, a.type, c.cat_id, c.cat_name, c.cat_desc, a.title, a.raw_content, a.create_time
            , group_concat(ua.tag_name separator ';') as tags
    from article a
    inner join category c on a.cat_id=c.cat_id
    inner join tag_to_article ta on ta.article_id=a.article_id
    inner join user_tag ua on ua.tag_id=ta.tag_id
    where a.cat_id<>10912482
    group by a.article_id, a.type, c.cat_id, c.cat_name, c.cat_desc, a.title, a.raw_content, a.create_time
    ```
    关于logstash jdbc插件使用，可参考[Logstash JDBC input](https://www.elastic.co/blog/logstash-jdbc-input-plugin)，详细配置可参考[Jdbc input plugin](https://www.elastic.co/guide/en/logstash/current/plugins-inputs-jdbc.html)
4. 为elasticsearch的index设置mapping，执行下面命令：
    ```
    curl -XPUT http://localhost:9200/myblog -d '{
    mappings: {
      article: {
        properties: {
          article_id: { type: "integer" },
          type: { type: "string", index: "not_analyzed" },
          cat_id: { type: "integer" },
          cat_name: { type: "string", index: "not_analyzed" },
          cat_desc: { type: "string", index: "analyzed" },
          title: { type: "string", index: "analyzed" },
          raw_content: { type: "string", index: "analyzed" },
          create_time: { type: "date" },
          tags: { type: "string", index: "analyzed" }
        }
      }
    }
    }';
    ```
5. 启动：`bin/logstash -f jdbc-demo.conf`
6. 验证：<br />
    基于上面logstash配置，将使用jdbc插件连接mysql数据库，执行sql命令读取数据，根据mapping中属性映射关系的配置，在elasticsearch中创建索引myblog，在myblog索引下面创建arcicle类型，并将sql语句执行的结果数据添加到elasticsearch中。<br />
    可以从输出文件`jdbc-output.json`查看logstash是否执行成功。<br />
    可以直接访问elasticsearch查看执行结果：http://127.0.0.1:9200/myblog/_search?q=Nhibernate

#### 安装运行kibana

1. 下载：[Download Kibana](https://www.elastic.co/downloads/kibana)
2. 解压：`tar -xvzf kibana-4.5.4-darwin-x64.tar.gz`
3. 配置：简单配置`config/kibana.yml`
   ```
   server.port: 5601
   server.host: "0.0.0.0"
   elasticsearch.url: "http://127.0.0.1:9200"
   ```
4. 启动：`bin/kibana`
5. 验证：http://localhost:5601/
   1. 为kibana指定elasticsearch索引。索引名称填写elasticsearch中创建的索引名称。注意去掉Index contains time-based events选项，这个选项的意思是elasticsearch中的索引名称是否按日期分片存储。
   ![Kibana1](https://github.com/Richie-Leo/ydres/blob/master/img/10/170/10-170-1000-01-kibana.png?raw=true)
   2. 可以在kibana的Discovery中选择索引进行搜索
   3. 使用Visualize创建图表
   ![Kibana2](https://github.com/Richie-Leo/ydres/blob/master/img/10/170/10-170-1000-02-kibana.png?raw=true)