*****************************
设置HIVE的元数据库
*****************************
::

  <property>
      <name>javax.jdo.option.ConnectionURL</name>
      <value>jdbc:mysql://hadooptest:3306/hive?useUnicode=true&amp;createDatabaseIfNotExist=true&amp;characterEncoding=UTF-8</value>
      <description>JDBC connect string for a JDBC metastore</description>
  </property>


通过配置mysql的访问控制，可以有效的指定能够访问到你的数据库的ip范围。

root到你的mysql ::

  mysql -u root -h（mysql的地址） -P（监听端口） -p

输入密码，进入mysql控制台，输入如下指令 ::

  grant all privileges on *.* to '用户名'@'白名单' identified by '密码'

  flush privileges;

(其中用户名是允许访问mysql的用户名，密码是对应用户的密码)

白名单内容可以如下：

% 表示所有ip可以访问；

也可以指定单个ip，如：10.81.7.80

或指定ip段，如：10.81.7.%

也可以是域名、机器名等；

上述语句的效果是在mysql的默认数据库 mysql中的user表插入或修改一条记录~
