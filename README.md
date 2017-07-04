# HDFSImporter 预处理模块

## 1. 概述

HDFSImporter 是用于将存放于 HDFS 上的数据批量地导入到 Sensors Analytics 的工具。

默认情况下，HDFSImporter读取的数据内容应当符合 Sensors Analytics 数据格式定义，相关定义请参考：

https://www.sensorsdata.cn/manual/data_schema.html

若希望使用 HDFSImporter 导入其他格式的数据，就需要开发自己的 HDFSImporter数据预处理模块(Java)，预处理流程大致如下：

```
              自定义数据预处理模块
    原始数据 =====================> 符合 Sensors Analytics 的数据格式定义的数据
```

一个例子，比如原始数据是 Nginx 的 `access_log`，内容如下：

```
123.123.123.123 - - [03/Sep/2016:15:45:28 +0800] "GET /index.html HTTP/1.1" 200 396 "http://www.baidu.com/test" "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_11_5) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/52.0.2743.116 Safari/537.36"
```

经过 **自定义数据预处理模块** 处理，得到 **符合 Sensors Analytics 数据格式定义的数据**，内容如下：

```
[{"distinct_id":"123.123.123.123","time":1472888728000,"type":"event","event":"RawPageView","properties":{"referrer":"http://www.baidu.com/test","status_code":"200","body_bytes_sent":396,"$ip":"123.123.123.123","$user_agent":"Mozilla/5.0 (Macintosh; Intel Mac OS X 10_11_5) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/52.0.2743.116 Safari/537.36","request_line":"GET /index.html HTTP/1.1"}},{"distinct_id":"123.123.123.123","time":1472893665131,"type":"profile_set","properties":{"from_baidu":true}}]
```

`src` 中为一个样例，功能是解析 Nginx 日志并生成事件和用户属性。

## 2. 开发方法

自定义 Java 类，并实现 `com.sensorsdata.analytics.tools.hdfsimporter.Processor` 接口即可。

接口定义如下：

```java
package com.sensorsdata.analytics.tools.hdfsimporter;

public interface Processor {
  String process(String line) throws Exception;
}
```

* `参数 String line`: HDFSImporter 读到的 HDFS 上的原始数据；
* `返回值`: 处理后得到的 Json 数组，数组元素为 **符合 SensorsAnalytics 的数据格式定义的数据**；

## 3. 配置启动方法

1. 编译 Java 代码并打包。以样例为例子：

   ```bash
   javac -cp hdfs_importer-standalone.jar cn/kbyte/CustomProcessor.java
   jar cvf custom_processor.jar cn/kbyte/CustomProcessor.class
   ```

   hdfs_importer-standalone.jar 可在部署了神策服务的机器上找到，一般情况下目录地址为：`/home/sa_cluster/sa/tools/hdfs_importer/lib/` 。

2. 将编译出来的 `custom_processor.jar` 放到 HDFSImporter 的 `lib` 目录下，即 `/home/sa_cluster/sa/tools/hdfs_importer/lib/` 目录下。

3. 启动 HDFSImporter 导入任务，并增加如下参数指定自己定义的类名

   ```shell
   # 通过该参数测试预处理类名
   --custom_processor cn.kbyte.CustomProcessor
   ```

4. 可以通过 `debug` 模式测试预处理模块的结果是否符合预期，开启方式为添加 `--debug` 参数。在该模式下，导入工具会将结果以文本的方式输出到 `HDFS` 相应的目录下，并且**不会**真正的导入的神策系统当中

   ```shell
   # 通过该参数开启 debug 模式
   --debug
   ```

## 4. 其他

* 如果数据无效，可以直接返回 `null`，HDFSImporter 会跳过这条数据；
* `process` 函数抛异常则 HDFSImporter 任务会执行失败；

  ​

