1. dubbo2.8.4没有在版本库的问题
    先git clone出来： git clone https://github.com/dangdangdotcom/dubbox
    进入根目录，mvn编译打包：mvn install -Dmaven.test.skip=true

2. redis消息队列使用手札
    在有高并发插入前的写库前，加上一个redis队列，能有效减少数据库压力，防止死锁发生。
    具体做法就是，高并发数据先进入redis队列，再循环从队列中每次取出合适数量的数据（几千几万看数据库写入能力和IO性能）插入到数据库。
    这样做的好处显而易见，减少了数据库的链接次数，提高了每次插入的数量，保证了前端往后端的写入速度，保证了数据库的性能和稳定。
    以前应对日均2千万写入量的系统的时候，我就用这种方案轻松应对了。

    链接：http://www.zhihu.com/question/35400475/answer/64111765

    我说的架构，比较适用大量的不是马上需要用的日志数据的写入。重要的业务数据不太适用这个方法。你说的问题与题主的问题不太一样，需要另外更复杂的架构去实现。第一个问题，例如新浪微博，用户会员的关系、点赞评论数量等，都是存在redis中，更新与查询，前台都只与redis交互，redis和mysql服务器通过一定机制同步数据，这样就能保证不会存在延迟。第二个问题，高可用性redis服务器架构，可以保证一台服务器挂掉，客户端也能通过其他的redis负载服务器获取到相同的数据，及时一个机房断电，也有多不同地域不同机房的负载服务器，redis本身的持久化策略是第一层保障，rdb文件会写到服务器上。3.0以前的redis可以通过twemproxy实现redis的负载，3.0以后redis本事已经能够实现。所以准备妥当的情况下也能最大保证数据的完整性。




3. centos6设置防火墙端口
    vi /etc/sysconfig/iptables
    -A INPUT -m state –state NEW -m tcp -p tcp –dport * -j ACCEPT
    /etc/init.d/iptables restart

4. redis 集群启动
    cd 进redis集群config文件目录
    分别 redis-server redis.conf
    进入redis安装目录
    src/redis-trib create --replicas 1 127.0.0.1:7000 127.0.0.1:7001 127.0.0.1:7002 127.0.0.1:7003 127.0.0.1:7004 127.0.0.1:7005

    具体过程：
    vi /etc/sysconfig/iptables
-A INPUT -m state –state NEW -m tcp -p tcp –dport 21 -j ACCEPT
-A INPUT -m state –state NEW -m tcp -p tcp –dport 80 -j ACCEPT
-A INPUT -m state –state NEW -m tcp -p tcp –dport 8080 -j ACCEPT
-A INPUT -m state –state NEW -m tcp -p tcp –dport 8081 -j ACCEPT
-A INPUT -m state –state NEW -m tcp -p tcp –dport 3306 -j ACCEPT
-A INPUT -m state –state NEW -m tcp -p tcp –dport 7000 -j ACCEPT
-A INPUT -m state –state NEW -m tcp -p tcp –dport 7001 -j ACCEPT
-A INPUT -m state –state NEW -m tcp -p tcp –dport 7002 -j ACCEPT
-A INPUT -m state –state NEW -m tcp -p tcp –dport 7003 -j ACCEPT
-A INPUT -m state –state NEW -m tcp -p tcp –dport 7004 -j ACCEPT
-A INPUT -m state –state NEW -m tcp -p tcp –dport 7005 -j ACCEPT
-A INPUT -m state –state NEW -m tcp -p tcp –dport 2181 -j ACCEPT
-A INPUT -m state –state NEW -m tcp -p tcp –dport 2888 -j ACCEPT
-A INPUT -m state –state NEW -m tcp -p tcp –dport 3888 -j ACCEPT
-A INPUT -m state –state NEW -m tcp -p tcp –dport 9092 -j ACCEPT


    /etc/init.d/iptables restart

    wget http://download.redis.io/releases/redis-3.0.1.tar.gz
    tar xf redis-3.0.1.tar.gz                       
    cd redis-3.0.1
    make && make install

    修改配置
    修改配置文件redis.conf中下面选项
    port 7000
    daemonize yes
    cluster-enabled yes
    cluster-config-file nodes.conf
    cluster-node-timeout 5000
    appendonly yes

    cd /data/cluster/7000
    redis-server redis.conf
    cd /data/cluster/7001
    redis-server redis.conf
    cd /data/cluster/7002
    redis-server redis.conf
    cd /data/cluster/7003
    redis-server redis.conf
    cd /data/cluster/7004
    redis-server redis.conf
    cd /data/cluster/7005
    redis-server redis.conf

    yum install ruby rubygems -y
    gem install redis

    cp redis-3.0.1/src/redis-trib.rb /usr/local/bin/redis-trib

    redis-trib create --replicas 1 192.168.1.128:7000 192.168.1.128:7001 192.168.1.128:7002 192.168.1.128:7003 192.168.1.128:7004 192.168.1.128:7005
    
5. JedisCluster实现redis的keys命令的方法
    ```
    @Autowired
    private JedisCluster jedisCluster;

    @Override
    public TreeSet<String> keys(String pattern) {
        logger.debug("Start getting keys...");
        TreeSet<String> keys = new TreeSet<>();
        Map<String, JedisPool> clusterNodes = jedisCluster.getClusterNodes();
        for(String k : clusterNodes.keySet()){
            logger.debug("Getting keys from: {}", k);
            JedisPool jp = clusterNodes.get(k);
            Jedis connection = jp.getResource();
            try {
                keys.addAll(connection.keys(pattern));
            } catch(Exception e){
                logger.error("Getting keys error: {}", e);
            } finally{
                logger.debug("Connection closed.");
                connection.close();//用完一定要close这个链接！！！
            }
        }
        logger.debug("Keys gotten!");
        return keys;
    }
    ```
6. git怎么删除一个文件夹？
```
rm -rf dir
git add -A
git commit -m 'remove dir'
git push origin master
```
7. 安装启动es
下载tar包，解压即可安装
下载地址version1.5.2：https://download.elastic.co/elasticsearch/elasticsearch/elasticsearch-1.5.2.tar.gz
解压命令：tar xvf xxx.gz

创建elsearch用户组及elsearch用户

groupadd elsearch
useradd elsearch -g elsearch -p elasticsearch
更改elasticsearch文件夹及内部文件的所属用户及组为elsearch:elsearch

cd /opt
chown -R elsearch:elsearch  elasticsearch
切换到elsearch用户再启动

su elsearch cd elasticsearch/bin
./elasticsearch

8. 使用rest api操作Elasticsearch
访问Elasticsearch中数据的一个模式。这个模式可以被总结为：
curl -X<REST Verb> <Node>:<Port>/<Index>/<Type>/<ID>

例子：
```
curl -XPUT 'localhost:9200/customer'
curl -XPUT 'localhost:9200/customer/external/1' -d '
{
  "name": "John Doe"
}'
curl 'localhost:9200/customer/external/1'
curl -XDELETE 'localhost:9200/customer'
```
9. sql
```
select t.fr3_region_name,t.fr2_region_name,t.region_name, t.region_id, group_concat(t.real_name) from
(
select fsr.region_id, fsr.salesman_id, fsr.real_name, region.region_name, region.fr2_region_name,region.fr3_region_name from fc_salesman_region fsr 
    left join (
        select fr1.region_id, fr1.region_name , fr23.fr2_region_name, fr23.fr3_region_name from fc_region fr1 
            left join (
                select fr2.region_id, fr2.region_name fr2_region_name , fr3.region_name fr3_region_name from fc_region fr2 
                    left join fc_region fr3 on fr2.parent_id=fr3.region_id
            ) fr23 
            on fr1.parent_id = fr23.region_id
    ) region 
    on fsr.region_id=region.region_id
) t 
group by t.region_id
```
10. 可以通过www.json-generator.com/自动生成json
11. sql
```
select * from fc_region r where r.region_id=1
union
select * from fc_region r where r.parent_id=1
union
select * from fc_region r1 where r1.parent_id in (
    select r.region_id from fc_region r where r.parent_id=1
)
union
select * from fc_region r2 where r2.parent_id in (
    select r1.region_id from fc_region r1 where r1.parent_id in (
        select r.region_id from fc_region r where r.parent_id=1
    )
)
```
12. spring-boot-starter-data-elasticsearch启动报错：org.elasticsearch.client.transport.NoNodeAvailableException: None of the configured nodes are available

报错原因是找不到对应的es节点，如果程序配置正确，es启动也正确，那么要注意es启动的时候要设置正确的host。
服务器es启动使用默认的host是localhost，如果你不改动它的默认配置，那么它将会在host为localhost的ip启动，而远程连接是需要配置对应的服务器host的，自然找不到。
所以，需要先改一下默认host配置再启动es
进入es安装目录的config目录
vi application.yml
修改配置：
network.host: 192.168.1.xxx //设置为你es部署的服务器ip

./elasticsearch 启动即可

13. Error：Detected both log4j-over-slf4j.jar AND slf4j-log4j12.jar on the class path, preempting StackOverflowError
add ：exclusions
```
    <exclusions>
      <exclusion> 
        <groupId>org.slf4j</groupId>
        <artifactId>slf4j-log4j12</artifactId>
      </exclusion>
      <exclusion> 
        <groupId>log4j</groupId>
        <artifactId>log4j</artifactId>
      </exclusion>
    </exclusions>
```
14. centos6修改静态ip
vim /etc/sysconfig/network-scripts/ifcfg-eth0
sample：
DEVICE="eth0"
#BOOTPROTO="dhcp"
BOOTPROTO="static"
IPADDR=192.168.1.119
NETMASK=255.255.255.0
HWADDR="00:0C:29:8A:73:8F"
IPV6INIT="yes"
NM_CONTROLLED="yes"
ONBOOT="yes"
TYPE="Ethernet"
UUID="42549603-375b-4eac-8cc1-2383dd8ea130"
GATEWAY=192.168.1.1
DNS1=8.8.8.8
DNS2=114.114.114.114

service network restart

15. mysql
service mysqld start

安装：
yum list installed | grep mysql
yum -y remove mysql-libs.x86_64
yum list | grep mysql 或 yum -y list mysql*
yum -y install mysql-server mysql mysql-devel
rpm -qi mysql-server


登陆不上的问题：
在进入mysql工具时，总是有错误提示:


mysql -u root -p
Enter password:
ERROR 1045 (28000): Access denied for user 'root'@'localhost' (using password: NO)
解决：方法操作很简单，如下：

/etc/init.d/mysql stop
mysqld_safe --user=mysql --skip-grant-tables --skip-networking &
mysql -u root mysql
mysql> UPDATE user SET Password=PASSWORD('root') where USER='root' and host='root' or host='localhost';//把空的用户密码都修改成非空的密码就行了。
mysql> FLUSH PRIVILEGES;
mysql> quit
/etc/init.d/mysqld restart
mysql -uroot -p
Enter password: <输入新设的密码newpassword>

开启远程连接
grant all privileges  on *.* to root@'%' identified by "root";
FLUSH PRIVILEGES;
重启：
service mysqld restart

grant all privileges  on *.* to mysync@'%' identified by "root";

16. mysql主从复制
mysql主从复制
（超简单）
怎么安装mysql数据库，这里不说了，只说它的主从复制，步骤如下：
1、主从服务器分别作以下操作：
  1.1、版本一致
  1.2、初始化表，并在后台启动mysql
  1.3、修改root的密码

2、修改主服务器master:
   #vi /etc/my.cnf
       [mysqld]
       log-bin=mysql-bin   //[必须]启用二进制日志
       server-id=222      //[必须]服务器唯一ID，默认是1，一般取IP最后一段

3、修改从服务器slave:
   #vi /etc/my.cnf
       [mysqld]
       log-bin=mysql-bin   //[不是必须]启用二进制日志
       server-id=226      //[必须]服务器唯一ID，默认是1，一般取IP最后一段

4、重启两台服务器的mysql
   /etc/init.d/mysql restart

5、在主服务器上建立帐户并授权slave:
   #/usr/local/mysql/bin/mysql -uroot -pmttang   
   mysql>GRANT REPLICATION SLAVE ON *.* to 'mysync'@'%' identified by 'q123456'; //一般不用root帐号，&ldquo;%&rdquo;表示所有客户端都可能连，只要帐号，密码正确，此处可用具体客户端IP代替，如192.168.145.226，加强安全。

6、登录主服务器的mysql，查询master的状态
   mysql>show master status;
   +------------------+----------+--------------+------------------+
   | File             | Position | Binlog_Do_DB | Binlog_Ignore_DB |
   +------------------+----------+--------------+------------------+
   | mysql-bin.000004 |      308 |              |                  |
   +------------------+----------+--------------+------------------+
   1 row in set (0.00 sec)
   注：执行完此步骤后不要再操作主服务器MYSQL，防止主服务器状态值变化

7、配置从服务器Slave：
   mysql>change master to master_host='192.168.145.222',master_user='mysync',master_password='q123456',
         master_log_file='mysql-bin.000004',master_log_pos=308;   //注意不要断开，308数字前后无单引号。

   Mysql>start slave;    //启动从服务器复制功能

8、检查从服务器复制功能状态：

   mysql> show slave status\G

   *************************** 1. row ***************************

              Slave_IO_State: Waiting for master to send event
              Master_Host: 192.168.2.222  //主服务器地址
              Master_User: mysync   //授权帐户名，尽量避免使用root
              Master_Port: 3306    //数据库端口，部分版本没有此行
              Connect_Retry: 60
              Master_Log_File: mysql-bin.000004
              Read_Master_Log_Pos: 600     //#同步读取二进制日志的位置，大于等于Exec_Master_Log_Pos
              Relay_Log_File: ddte-relay-bin.000003
              Relay_Log_Pos: 251
              Relay_Master_Log_File: mysql-bin.000004
              Slave_IO_Running: Yes    //此状态必须YES
              Slave_SQL_Running: Yes     //此状态必须YES
                    ......

注：Slave_IO及Slave_SQL进程必须正常运行，即YES状态，否则都是错误的状态(如：其中一个NO均属错误)。

以上操作过程，主从服务器配置完成。

17. 商务部安装订单商品分类列表sql
```
#select * from 
#(select region_id from fc_region where region_id in( select region_id from fc_region where  parent_id=76)) fcr
#left join 
#(
select 
    #ro.parent_region_id, ro.parent_region_name, 
    #ro.grant_region_id as group_region_id, ro.grant_region_name as group_region_name, 
    #ro.parent_region_id as group_region_id, ro.parent_region_name as group_region_name, ro.grant_region_name,
    concat(ro.region_id,ro.cate_id) as id,
    ro.region_id as group_region_id,ro.region_name as group_region_name, ro.parent_region_name, ro.grant_region_name,
    ro.cate_id, ro.cate_name, count(*) as order_account, sum(ro.order_amount) as order_amount from 
        (select fo.order_id, fo.order_sn, fo.order_amount, fo.sstore_id, fo.add_time, fo.extension, cv.cate_name, cv.cate_id 
            , re.region_id, re.region_name, re.parent_region_id, re.parent_region_name, re.grant_region_id, re.grant_region_name
            from (
                select * from fc_order o where 1=1 and o.extension='fix'
                #and o.add_time > 1455661380 and o.add_time < 1461694617
                and o.add_time > 1425423715 and o.add_time < 1455661325
            ) fo 
            left join (
                select og.order_id, og.goods_id, g.cate_id, ca.cate_name from fc_order_goods og 
                left join fc_goods g on og.goods_id=g.goods_id
                left join fc_gcategory ca on g.cate_id=ca.cate_id
                where ca.cate_name is not null
                #and ca.cate_id=85
            ) cv on fo.related_order_id=cv.order_id
            left join (
                select fs.sstore_id, fs.sstore_name, fs.region_id, sfr.region_name, sfr.parent_region_id, sfr.parent_region_name, sfr.grant_region_id, sfr.grant_region_name from fc_sstore fs 
                left join (
                    select fr1.region_id, fr1.region_type, fr1.region_name,fr23.fr2_region_id as parent_region_id, fr23.fr2_region_name as parent_region_name
                        ,fr23.fr3_region_id as grant_region_id, fr23.fr3_region_name as grant_region_name 
                        from fc_region fr1 
                            left join (
                                    select fr2.region_id fr2_region_id, fr2.region_name fr2_region_name, fr3.region_id fr3_region_id, fr3.region_name fr3_region_name from fc_region fr2 
                                            left join fc_region fr3 on fr2.parent_id=fr3.region_id
                            ) fr23 
                            on fr1.parent_id = fr23.fr2_region_id
                ) sfr on fs.region_id=sfr.region_id
            ) re on fo.sstore_id=re.sstore_id
            where 1=1
            and cv.cate_id is not null
            order by fo.add_time desc
        ) ro
        where 1=1 
        #and ro.grant_region_id=76 or ro.parent_region_id=76
        #and ro.cate_id=85
        #and ro.region_id in (693)
        group by ro.region_id, ro.cate_id
        #group by ro.grant_region_id, ro.cate_id
#) fix
#on fcr.region_id=fix.group_region_id
```
18. sql:列出有订单的商品类别
```
select distinct te.cate_id, te.cate_name 
    from (select o.order_id, o.related_order_id from fc_order o where o.extension='fix') oo
    left join (select og.order_id, ca.cate_id,  ca.cate_name
            from fc_order_goods og 
            left join fc_goods g on og.goods_id=g.goods_id
            left join fc_gcategory ca on g.cate_id=ca.cate_id
            where ca.cate_name is not null and ca.cate_id is not null
    ) te
    on oo.related_order_id=te.order_id
    where te.cate_id is not null
```
19. svn安装配置启动

安装步骤如下：
1、yum install subversion
2、查看安装版本 svnserve --version
3、创建SVN版本库目录 mkdir -p /var/svn/svnrepos
4、创建版本库  svnadmin create /var/svn/svnrepos
   执行了这个命令之后会在/var/svn/svnrepos目录下生成如下这些文件
5、进入conf目录（该svn版本库配置文件）cd conf/
   authz文件是权限控制文件
   passwd是帐号密码文件
   svnserve.conf SVN服务配置文件
6、设置帐号密码 vi passwd
   在[users]块中添加用户和密码，格式：帐号=密码，如test=123456
7、设置权限 vi authz
   在末尾添加如下代码：
```
[/]  
test=rw
```
意思是版本库的根目录test对其有读写权限
8、修改svnserve.conf文件  vi svnserve.conf
   打开下面的几个注释：
```
   anon-access = read #匿名用户可读
   auth-access = write #授权用户可写
   password-db = passwd #使用哪个文件作为账号文件
   authz-db = authz #使用哪个文件作为权限文件
   realm = /var/svn/svnrepos # 认证空间名，版本库所在目录
```
后续实际使用时，分支合并到主干会出现："Unreadable path encountered; access denied;"的错误，解决办法是：
```
anon-access=none
```
9、启动svn版本库  svnserve -d -r /var/svn/svnrepos（停止SVN命令  killall svnserve）

20. 使用TortoiseSVN操作：svn分支主干相互操作

主干打分支：
右键主干目录，选择"branches/tag"
选择"to path"既是你要把主干复制到的分支目录

分支合并到主干：
右键主干目录，选择"merge"，
然后在弹出的选择框中选择第二种合并方式：
"merge two different trees",即合并两个不同的目录（第一种方式是填svn版本范围）
from：填的是主干目录；
to：填的是分支目录；
其他保持默认即可，最后merge
done!

Error：提示 : SVN 遇到不可读的路径；拒绝访问。 英文是: Unreadable path encountered; access denied;
Solution：Anyway it can be fixed by changing in "svnserve.conf" file the line:
```
[general]
anon-access = none
```
21. js中遍历map/hash/object的方式
```
var obj = {"id":1, "name":"name"};
for (var key in ds) {
    var value = obj[key];
}
```
22. java set 的add顺序问题
HashSet的add，顺序和插入顺序相反，后插入的在set的前面；
如果要按照顺序插入set，则可以使用LinkedHashSet

23. maven mybatis generate
mvn mybatis-generator:generate
24. maven centos6 install
```
1、官网找到最新版的安装包：
http://maven.apache.org/download.cgi

拷贝文件名为 *-bin.tar.gz 的链接地址；

2、下载
# wget http://mirrors.hust.edu.cn/apache/maven/maven-3/3.3.3/binaries/apache-maven-3.3.3-bin.tar.gz

3、解压
# tar xvf apache-maven-3.3.3-bin.tar.gz

如果需要：移动到其他目录
建立软连接：# ln -s apache-maven-3.3.3 maven

4、配置环境变量
# vi /etc/profile
export M2_HOME=/usr/local/apache-maven
export PATH=$PATH:$M2_HOME/bin

# source /etc/profile

5、验证是否安装成功
# mvn -version
Apache Maven 3.3.3 (7994120775791599e205a5524ec3e0dfe41d4a06; 2015-04-22T19:57:37+08:00)
Maven home: /opt/app/maven
Java version: 1.8.0_51, vendor: Oracle Corporation
Java home: /usr/java/jdk1.8.0_51/jre
Default locale: zh_CN, platform encoding: UTF-8
OS name: "linux", version: "3.10.0-229.el7.x86_64", arch: "amd64", family: "unix"
```

25. 使用Jenkins配置Git和Maven的自动化构建Tomcat项目

### 0- 前置条件
安装jdk和tomcat

### 1 - 部署Jenkins

官网下载http://jenkins-ci.org/
我下载的是最新的2.5版本
下载得到一个jenkins.war的war包
可以直接使用java -j jenkins.war运行，或者把war包放到tomcat运行
我这里是放到tomcat，启动tomcat即可.
启动后可以在http://localhost:8080/jenkins/，看到Jenkins已经在运行
第一次进入，需要你输入一个key,会提示您在/root/.jenkins/...某个路径下找到
进入后会提示你安装一些插件，根据需要安装即可，或者直接安装它提供的一键建议安装.
### 2 - 安装相关插件
我这里需要git plugin和maven plugin，可以在插件管理处搜索安装。
这两个基本是默认安装好了的.
### 3 - 安装git和maven

安装git
```
yum install git
```
安装maven
```
1、官网找到最新版的安装包：
http://maven.apache.org/download.cgi

拷贝文件名为 *-bin.tar.gz 的链接地址；

2、下载
# wget http://mirrors.hust.edu.cn/apache/maven/maven-3/3.3.3/binaries/apache-maven-3.3.3-bin.tar.gz

3、解压
# tar xvf apache-maven-3.3.3-bin.tar.gz

如果需要：移动到其他目录
建立软连接：# ln -s apache-maven-3.3.3 maven

4、配置环境变量
# vi /etc/profile
export M2_HOME=/usr/local/apache-maven
export PATH=$PATH:$M2_HOME/bin

# source /etc/profile

5、验证是否安装成功
# mvn -version
Apache Maven 3.3.3 (7994120775791599e205a5524ec3e0dfe41d4a06; 2015-04-22T19:57:37+08:00)
Maven home: /opt/app/maven
Java version: 1.8.0_51, vendor: Oracle Corporation
Java home: /usr/java/jdk1.8.0_51/jre
Default locale: zh_CN, platform encoding: UTF-8
OS name: "linux", version: "3.10.0-229.el7.x86_64", arch: "amd64", family: "unix"
```

### 4 - 配置jdk和maven路径等

选择"系统管理" -> "Global Tool Configuration"
在里面配置好jdk，maven的路径，只需要配置它们的根路径.

### 5 - 创建任务，配置项目信息

选择“新建”，填写项目任务名，并选择“构建一个maven项目”，保存.
进入项目配置.
填写项目名称
填写Repository URL
我这里使用的是一个git上的一个建议springmvc项目
https://github.com/bingyue/easy-springmvc-maven

如果需要，也可以配置一下构建后发送邮件到您的邮箱

### 6 - 配置构建成功后的动作，添加shell
配置项目中的“Post Steps”，设置构建完成后的动作.
这里我设置为将war包拷贝到Tomcat目录，删除项目原来的内容文件夹，并重启Tomcat。
选择Run only if build succeeds or is unstable ，点击添加Execute Shell：
shell脚本:
```
#!/bin/bash 
#copy file and restart tomcat
 
tomcat_path=/usr/local/tomcat/apache-tomcat-7.0.65
project=easy-springmvc-maven
war_name=easy-springmvc-maven.war
war_path=http://192.168.1.119:8080/jenkins/job/jenkins-test/ws/target
server_port=8081
file_path=/root/.jenkins/workspace/jenkins-test/target
 
now=$(date +"%Y%m%d%H%M%S")
echo "the shell execute time is ${now}"
 
echo `lsof -n -P -t -i :${server_port}`
tomcat_pid=`lsof -n -P -t -i :${server_port}`
echo "the tomcat_pid is ${tomcat_pid}"
 
if [ "${tomcat_pid}" != "" ]; then 
   kill -9 $tomcat_pid
   echo "kill the server"
fi 
 
echo "rm ${tomcat_path}/webapps/${war_name}"
rm ${tomcat_path}/webapps/${war_name}
 
echo "rm -rf ${tomcat_path}/webapps/${project}"
rm -rf ${tomcat_path}/webapps/${project}
 
cd $file_path
if [ -f ${war_name} ]; then 
   cp ${war_name} ${tomcat_path}/webapps
   echo "cp war to webapps finished"
else
   echo "${war_name} unexists"
fi

cd $tomcat_path/bin
echo "run startup"
sudo ./startup.sh
echo "server restarted"
```
我这里是在/usr/local/tomcat/apache-tomcat-7.0.65
这个路径下配置了另外一个tomcat来运行测试的web项目.
shell脚本启动tomcat的命令"./startup.sh"，注意要使用sudo


```
#ssh user@host.com
login: user
password: password
#cd $CATALINA_HOME/bin;
#sudo ./catalina.sh start;

You could also use this technique to restart Tomcat, by calling the startup and shutdown scripts in a single command
 #sudo ./catalina.sh stop;sudo ./catalina.sh start;
```
### 7 - 运行构建项目
打开项目的主面板，直接点击绿色的运行任务构建按钮
26. 134redis路径
/usr/local/lib/redis-3.0.1

27. maven build with resources
mvn clean install -P dev
28. shell remote 
ssh scp:
```
sshpass -p "your password" scp ./abc.txt hostname/abc.txt
```
install sshpass:
```
cd /etc/yum.repos.d/  
wget http://download.opensuse.org/repositories/home:Strahlex/CentOS_CentOS-6/home:Strahlex.repo  
yum install sshpass
```



29. Failure to find com.alibaba:dubbo:jar:2.8.4
https://github.com/dangdangdotcom/dubbox 
下载下来后，解压，在解压后的路径中，执行: mvn install -Dmaven.test.skip=true

30. shell to start project itimes
```
#!/bin/bash
# script to update and restart project itimes
remote_source_path=/root/.jenkins/workspace/dev-timeitem-itimes/target
target_jar=itimes-1.0.0-SNAPSHOT.jar
target_path=/usr/local/itimes
target_bak_path=/usr/local/itimes_bak

now=$(date '+%Y%m%d%H%M%S')
echo "update and restart project itimes start..."
echo "now time: ${now}"

mkdir -p ${target_bak_path}/${now}
/usr/bin/sshpass -p "root" scp root@192.168.1.119:${remote_source_path}/itimes-1.0.0-SNAPSHOT.jar ${target_bak_path
}/${now}

cd $target_bak_path/$now
if [ -f ${target_jar} ]; then
  pkill -f 'java.*itimes'
  sleep 3
  temppid=$(ps aux | grep java| grep "${target_jar}" | grep -v grep | awk '{ print $2}')
  if [[ "$temppid" != "" ]]; then
    echo "temppid is ${temppid}"
    ps -ef | grep ${target_jar} | grep -v grep | awk '{print $2}' | xargs kill
  fi
  echo "kill ${target_jar} done.."
  cd $target_path
  if [ -f ${target_jar} ]; then
    rm -f ${target_jar}
    echo "target_jar deleted.."
  else
    echo "${target_jar} not exist, go on to copy.."
  fi
  cp ${target_bak_path}/${now}/${target_jar} ${target_path}
  echo "cp done!"
  chmod 755 ${target_jar}
  nohup java -jar ${target_path}/${target_jar} < /dev/null > ${target_bak_path}/${now}/itime-${now}.log 2>&1 &
  echo "app startting.."
  for i in 2 4 6 10; do
    sleep $i
    echo "wait and check app starting up..."
    APP_PID=`ps aux | grep java| grep "${target_jar}" | grep -v grep | awk '{ print $2}'`
    if [ $APP_PID > 0 ]; then
        break;
    else
        echo "app is still starting up..."
    fi
  done
  pid=$(ps aux | grep java| grep "${target_jar}" | grep -v grep | awk '{ print $2}')
  echo "app id is: ${pid}"
  if [ $pid ]; then
    echo "${target_jar} is running now!"
  else
    echo "${target_jar} is not running! ${target_jar} startup failed!"
  fi
  success_flag="false"
  for i in 2 4 6 8 10 12; do
    echo "checking app startup status..."
    success_target_tag=`cat ${target_bak_path}/${now}/itime-${now}.log | grep "Started App in"`
    if [[ "" != "$success_target_tag" ]]; then
      echo "Found success target tag, app startup successfully.."
      success_flag="true"
      break;
    else
        echo "app is still starting up..."
        sleep $i
        echo "sleep $i ..."
    fi
  done
  if [[ "true" != "$success_flag"  ]]; then
    echo "Not found success target tag, app startup failed..."
  fi
else
  echo "Target jar not exist! Update not complete!"
fi

echo "update done!"
```

31. git分支常用命令
```
git checkout -b dev

git checkout master

git merge dev

git branch -d dev

git merge --no-ff -m "merge with no-ff" dev
```

```
git stash
git stash list
git stash pop


```

```
多人协作的工作模式通常是这样：

首先，可以试图用git push origin branch-name推送自己的修改；

如果推送失败，则因为远程分支比你的本地更新，需要先用git pull试图合并；

如果合并有冲突，则解决冲突，并在本地提交；

没有冲突或者解决掉冲突后，再用git push origin branch-name推送就能成功！

如果git pull提示“no tracking information”，则说明本地分支和远程分支的链接关系没有创建，用命令git branch --set-upstream branch-name origin/branch-name。

这就是多人协作的工作模式，一旦熟悉了，就非常简单。

小结

查看远程库信息，使用git remote -v；

本地新建的分支如果不推送到远程，对其他人就是不可见的；

从本地推送分支，使用git push origin branch-name，如果推送失败，先用git pull抓取远程的新提交；

在本地创建和远程分支对应的分支，使用git checkout -b branch-name origin/branch-name，本地和远程分支的名称最好一致；

建立本地分支和远程分支的关联，使用git branch --set-upstream branch-name origin/branch-name；

从远程抓取分支，使用git pull，如果有冲突，要先处理冲突。
```

32. 聚合函数使用例子
```
SELECT * FROM ssms_activity_hold h
LEFT JOIN  (
  SELECT
    t.hold_id,
    group_concat(t.detail) AS surplus_detail
  FROM
    (
      SELECT d.*, concat(l.service_name, num, '次') AS detail
      FROM ssms_activity_hold_detail d
      LEFT JOIN ssms_activity_licenses l ON d.license_id = l.license_id
    ) t
  GROUP BY
    t.hold_id
) hd
ON h.hold_id = hd.hold_id
```
33. 设置上传图片有效
http://192.168.1.206/upload/upload.php/index/file_accepted
参数：
timestamp 
token
file_ids  文件ID，格式：array(1,2)或  1|2
item_id 关联的实体ID，如商品ID、店铺ID等

设置上传图片有效
34. ssms_user表修改
```
alter table db_ssms.ssms_user add column province INT(11) not null default '0' COMMENT '门店区域省id';
alter table db_ssms.ssms_user add column city INT(11) not null default '0' COMMENT '门店区域市id';
alter table db_ssms.ssms_user add column area INT(11) not null default '0' COMMENT '门店区域区id';
```
35. ssms发布
walle：
http://192.168.1.222/task/

svn：
svn://192.168.1.206/fengche/web/deploy/ssms


36. deploy
svn://192.168.1.206/fengche/web/deploy/ssms/alpha


37. springmvc api version

https://github.com/mindhaq/restapi-versioning-spring.git
https://github.com/augusto/restVersioning.git

这两个git项目里提供了4种在springmvc中进行api版本控制的处理方法，你们看怎么样，能否不使用冗余部署的方式。

1. /api/v1/xxx
controller中的方法注解写法：
```
    @ResponseBody
    @RequestMapping(value = "/apiurl/{version}/hello", method = GET, produces = APPLICATION_JSON_VALUE)
    public Hello sayHelloWorldUrl(@PathVariable final ValidVersion version) {
        return new Hello();
    }
```
2. /api/xxx
在header中添加"X-API-Version":"v1"来进行版本请求的区别
controller中的方法注解写法：
```
    @ResponseBody
    @RequestMapping(value = "/apiheader/hello", method = GET, produces = APPLICATION_JSON_VALUE)
    public Hello sayHelloWorldHeader(@RequestHeader("X-API-Version") final ValidVersion version) {
        return new Hello();
    }
```

3. /api/xxx
在header中添加Accept: "application/vnd.company.app-v1+json"来进行版本区别
controller中的方法注解写法：
```
    @ResponseBody
    @RequestMapping(
        value = "/apiaccept/hello", method = GET,
        produces = {"application/vnd.company.app-v1+json", "application/vnd.company.app-v2+json"}
    )
```
4. /api/xxx
在header中添加Accept: "application/vnd.company.app-v1+json"来进行版本区别
controller写法：
```
@Controller
@VersionedResource(media = "application/vnd.app.resource")
public class TestController {

    @RequestMapping(value = {"/resource"}, method = RequestMethod.GET)
    @VersionedResource(from = "1.0", to = "1.0")
    @ResponseBody
    public Resource getResource_v1() {
        return new Resource("1.0");
    }

    @RequestMapping(value = {"/resource"}, method = RequestMethod.GET)
    @VersionedResource(from = "2.0")
    @ResponseBody
    public Resource getResource_v2_onwards() {
        return new Resource("2.0");
    }
}
```
38. 在单服务器上安装部署FastDFS+Nginx
那么多服务器和软件安装配置中，FastDFS算是比较复杂的一个了。
这个例子storage和tracker均部署在同一台服务器（ip：192.168.1.134）

## FastDFS安装配置

### Tracker的安装及配置
#### 1.安装编译器
```
yum install -y gcc gcc-c++
```

#### 2.下载安装libevent
```
wget https://sourceforge.net/projects/levent/files/libevent/libevent-2.0/libevent-2.0.22-stable.tar.gz
tar xvzf libevent-2.0.22-stable.tar.gz
cd libevent-2.0.22-stable
./configure
make && make install
```

#### 3.下载安装fastDFS
```
wget http://code.google.com/p/fastdfs/downloads/detail?name=FastDFS_v4.06.tar.gz
（code.google.com已无法访问，可以自己手动下载再上传到服务器）
tar -xvzf FastDFS_v4.06.tar.gz
cd FastDFS
./make.sh
./make.sh install
```
安装成功后/usr/local/bin下会出现一系列fastDFS命令

#### 4.配置tracker
```
vim /etc/fdfs/tracker.conf
```
修改base_path以存储tracker信息（这里不做修改，使用默认路径/home/yuqing/fastdfs，需要先行创建目录）

#### 5.启动tracker
```
/usr/local/bin/fdfs_trackerd /etc/fdfs/tracker.conf
```

### Storage配置 -> Storage1配置
正常情况下storage1部署一个服务器，storage2部署一个服务器，tracker部署一个服务器。
我这里只部署一个storage一个tracker，并部署在同一个服务器

#### 1~3：参考Tracker安装步骤

#### 4.配置storage
```
vim /etc/fdfs/storage.conf
```
修改tracker_server=192.168.1.134:22122
修改base_path以存储storage信息（这里不做修改，使用默认路径/home/yuqing/fastdfs，需要先行创建目录）
例子：
```
group_name=group1
base_path=/home/yuqing/fastdfs
store_path0=/home/yuqing/fastdfs
tracker_server=192.168.1.134:22122
http.server_port=8080      #web server的端口改成8080（与nginx 端口一致）
```

#### 5.启动storage
```
/usr/local/bin/fdfs_storaged /etc/fdfs/storage.conf
```
启动过程中，fastDFS会在base_path下的data目录中创建一系列文件夹，以存储数据

### 安装nginx以及fastdfs-nginx-module模块
nginx的rewrite模块和cache模块需要先安装pcre和openssl
#### 安装pcre
下载pcre
```
tar zxvf pcre-8.12.tar.gz
cd pcre-8.12
./configure
make
make install
```

#### 安装openssl
centos下解决办法:
```
yum -y install openssl openssl-devel
```

#### 安装nginx
```
cd /home/yihua
wget http://nginx.org/download/nginx-0.8.55.tar.gz
tar zxvf nginx-0.8.55.tar.gz
cd nginx-0.8.55
./configure --prefix=/opt/nginx --with-http_stub_status_module
make && make install
```

#### 安装fastdfs-nginx-module
下载并上传fastdfs-nginx-module_v1.15.tar.gz
```
tar xzf fastdfs-nginx-module_v1.15.tar.gz
cd /home/yihua/nginx-0.8.55 
./configure --add-module=/home/yihua/fastdfs-nginx-module/src
make; make install
```

#### 配置nginx和fastdfs-nginx-module
```
cp /home/yihua/fastdfs-nginx-module/src/mod_fastdfs.conf /etc/fdfs
vim /etc/fdfs/mod_fastdfs.conf
```
修改 fastdfs的nginx模块的配置文件 mod_fastdfs.conf
一般只需改动以下几个参数即可：
```
base_path=/home/yuqing/fastdfs      #保存日志目录
tracker_server=192.168.1.134:22122    #tracker 服务器的 IP 地址以及端口号
storage_server_port=23000            #storage 服务器的端口号
group_name=group1                    #当前服务器的 group 名
url_have_group_name = true           #文件 url 中是否有 group 名
store_path_count=1                   #存储路径个数，需要和 store_path 个数匹配
store_path0=/home/yuqing/fastdfs    #存储路径
http.need_find_content_type=true     # 从文件 扩展 名查 找 文件 类型 （ nginx 时 为true）
group_count = 1                      #设置组的个数
```
然后在末尾添加分组信息，目前只有一个分组，就只写一个
```
[group1]
group_name=group1
storage_server_port=23000
store_path_count=1
store_path0=/home/yuqing/fastdfs
```

建立 M00 至存储目录的符号连接
```
ln -s /home/yuqing/fastdfs/data /home/yuqing/fastdfs/data/M00
```
将 server 段中的 listen 端口号改为 8080：
```
vim /usr/local/nginx/conf/nginx.conf
listen       8080;
```
在 server 段中添加fastdfs的配置
```
        location /group1/M00 {
               root   /home/yuqing/fastdfs/data;
               ngx_fastdfs_module;
        }
```

#### 准备nginx启动脚本
编辑 /etc/init.d/nginx，如下内容:
```
#!/bin/bash
# nginx Startup script for the Nginx HTTP Server
# it is v.1.4.7 version.
# chkconfig: - 85 15
# description: Nginx is a high-performance web and proxy server.
# It has a lot of features, but it's not for everyone.
# processname: nginx
 
# pidfile: /usr/local/nginx/logs/nginx.pid
# config: /usr/local/nginx/conf/nginx.conf
 
nginxd=/usr/local/nginx/sbin/nginx
nginx_config=/usr/local/nginx/conf/nginx.conf
nginx_pid=/usr/local/nginx/logs/nginx.pid
nginx_lock=/var/lock/subsys/nginx
RETVAL=0
prog="nginx"
 
# Source function library.
. /etc/rc.d/init.d/functions
 
# Source networking configuration.
. /etc/sysconfig/network
 
# Check that networking is up.
[ ${NETWORKING} = "no" ] && exit 0
[ -x $nginxd ] || exit 0
 
# Start nginx daemons functions.
start() {
    nginx_is_run=`ps -ef | egrep 'nginx:\s*(worker|master)\s*process' | wc -l`
    if [ ${nginx_is_run} -gt 0 ];then
        echo "nginx already running...."
        exit 1
    fi
    echo -n $"Starting $prog: "
    daemon $nginxd -c ${nginx_config}
    RETVAL=$?
    echo
    [ $RETVAL = 0 ] && touch ${nginx_lock}
    return $RETVAL
}
 
# Stop nginx daemons functions.
stop() {
    echo -n $"Stopping $prog: "
    killproc $nginxd
    RETVAL=$?
    echo
    [ $RETVAL = 0 ] && rm -f ${nginx_lock} ${nginx_pid}
}
 
# Reload nginx config file
reload() {
    echo -n $"Reloading $prog: "
    #kill -HUP `cat ${nginx_pid}`
    killproc $nginxd -HUP
    RETVAL=$?
    echo
}
 
# See how we were called.
case "$1" in
    start)
        start
        ;;
    stop)
        stop
        ;;
    reload)
        reload
        ;;
    restart)
          stop
        start
        ;;
    status)
        status $prog
        RETVAL=$?
        ;;
    *)
        echo $"Usage: $prog {start|stop|restart|reload|status|help}"
        exit 1
esac
```

Nginx启动提示找不到libpcre.so.1解决方法
如果是32位系统
```
ln -s /usr/local/lib/libpcre.so.1 /lib
```
如果是64位系统
```
ln -s /usr/local/lib/libpcre.so.1 /lib64
```

### 启动nginx

```
# chmod u+x /etc/init.d/nginx
# chkconfig --add nginx
# chkconfig nginx on
# service nginx start
正在启动 nginx：                                           [确定]
# service nginx status
nginx (pid 26500) 正在运行...
```
查看nginx的日志 错误日志logs/error.log 看是否有问题

### 其他
#### 启动nginx，tracker和storage
重启tracker：
/usr/local/bin/restart.sh /usr/local/bin/fdfs_trackerd /etc/fdfs/tracker.conf
重启storage：
/usr/local/bin/restart.sh /usr/local/bin/fdfs_storaged /etc/fdfs/storage.conf
启动nginx：
/usr/local/nginx/sbin/nginx
检查nginx状态：
/usr/local/nginx/sbin/nginx -t
重启nginx：
/usr/local/nginx/sbin/nginx -s reload

#### tracker运行
直接使用 fdfs_trackerd 来启动tracker进程，然后使用netstat 查看端口是否起来。
```
[root@tracker fdfs]# fdfs_trackerd /etc/fdfs/tracker.conf restart
[root@tracker fdfs]# netstat -antp | grep trackerd
tcp        0      0 0.0.0.0:22122               0.0.0.0:*                   LISTEN      14520/fdfs_trackerd 
[root@tracker fdfs]# 
```

#### storage运行
```
# fdfs_storaged /etc/fdfs/storage.conf restart
查看端口是否起来
# netstat -antp | grep storage
tcp        0      0 0.0.0.0:23000               0.0.0.0:*                   LISTEN      10316/fdfs_storaged 
```
也可以以下命令来监控服务器的状态
```
# fdfs_monitor /etc/fdfs/client.conf
```
看到ACTIVE,就说明已经成功注册到了tracker。

#### 开机启动
设置tracker开机自动启动
```
[root@tracker tracker]# echo "/usr/bin/fdfs_trackerd /etc/fdfs/tracker.conf restart" >> /etc/rc.local
[root@tracker tracker]# cat /etc/rc.local
#!/bin/sh
#
# This script will be executed *after* all the other init scripts.
# You can put your own initialization stuff in here if you don't
# want to do the full Sys V style init stuff.
touch /var/lock/subsys/local
/usr/bin/fdfs_trackerd /etc/fdfs/tracker.conf restart
[root@tracker tracker]# 
```

设置storage开机启动
```
[root@server1 fdfs]# echo "/usr/bin/fdfs_storaged /etc/fdfs/storage.conf restart" >> /etc/rc.local
[root@server1 fdfs]# cat /etc/rc.local
#!/bin/sh
#
# This script will be executed *after* all the other init scripts.
# You can put your own initialization stuff in here if you don't
# want to do the full Sys V style init stuff.
touch /var/lock/subsys/local
/usr/bin/fdfs_storaged /etc/fdfs/storage.conf restart
```


### 使用client测试文件上传
先配置一下client
vi /etc/fdfs/client.conf
保证一下配置：
```
base_path=/home/yuqing/fastdfs
tracker_server=192.168.1.134:22122
http.tracker_server_port=8080
```
vi test.txt

/usr/local/bin/fdfs_test /etc/fdfs/client.conf  upload  test.txt
得到：
```
This is FastDFS client test program v4.06

Copyright (C) 2008, Happy Fish / YuQing

FastDFS may be copied only under the terms of the GNU General
Public License V3, which may be found in the FastDFS source kit.
Please visit the FastDFS Home Page http://www.csource.org/
for more detail.

[2016-07-06 07:31:57] DEBUG - base_path=/home/yuqing/fastdfs, connect_timeout=30, network_timeout=60, tracker_server_count=1, anti_steal_token=0, anti_steal_secret_key length=0, use_connection_pool=0, g_connection_pool_max_idle_time=3600s, use_storage_id=0, storage server id count: 0

tracker_query_storage_store_list_without_group:
        server 1. group_name=, ip_addr=192.168.1.134, port=23000

group_name=group1, ip_addr=192.168.1.134, port=23000
storage_upload_by_filename
group_name=group1, remote_filename=M00/00/00/wKgBhld9Fl2ATSOYAAAAB9mqc8s085.txt
source ip address: 192.168.1.134
file timestamp=2016-07-06 07:31:57
file size=7
file crc32=3651826635
file url: http://192.168.1.134:8080/group1/M00/00/00/wKgBhld9Fl2ATSOYAAAAB9mqc8s085.txt
storage_upload_slave_by_filename
group_name=group1, remote_filename=M00/00/00/wKgBhld9Fl2ATSOYAAAAB9mqc8s085_big.txt
source ip address: 192.168.1.134
file timestamp=2016-07-06 07:31:57
file size=7
file crc32=3651826635
file url: http://192.168.1.134:8080/group1/M00/00/00/wKgBhld9Fl2ATSOYAAAAB9mqc8s085_big.txt
[root@localhost yihua]# This is FastDFS client test program v4.06
-bash: This: command not found
Copyright (C) 2008, Happy Fish / YuQing

```
在浏览器打开http://192.168.1.134:8080/group1/M00/00/00/wKgBhld9Fl2ATSOYAAAAB9mqc8s085.txt
可以看见你的文件。那么就成功了。

done.


39. java项目和目录统一规范
以监控项目为例，项目名为：monitor

1) trunk下有一个{xx}-parent目录：trunk/monitor-parent

2) monitor-parent有一个pom.xml文件规划好monitor项目的所有模块

3) trunk/monitor-parent目录下，是项目的所有其它模块.模块命名格式：{xxx}-{模块}
例如：
trunk/monitor-parent/monitor-common
trunk/monitor-parent/monitor-web
...
模块名称原则上不限制，实际项目可以根据需要自定义需要的模块名称。
一般公共模块使用名称xxx-common，web模块使用名称xxx-web。


4) war包目录统一
war包目录统一建议如下：
以monitor项目为例，monitor项目通过mvn编译完成以后war所在目录为monitor-web/target下面
如果是jar项目而不是war项目，可以根据需要与运维沟通打包放进合适的目录下。




40. 让eclipse忽略js校验
```
1.Right-click your project.
2.Navigate to: Properties → JavaScript → Include Path
3.Select Source tab. (It looks identical to Java Build Path Source tab.)
4.Expand JavaScript source folder.
5.Highlight Excluded pattern.
6.Press the Edit button.
7.Press the Add button next to Exclusion patterns box.
8.You may either type Ant-style wildcard pattern, or click Browse button to mention the JavaScript source by name.
```
41. 多应用合并部署
多个应用系统（.war）同时部署在一个Tomcat实例中，共享同一个JVM进程，每个应用之间通过ClassLoader隔离，但是应用之间的rpc调用不走网络，而走Java的本地调用


42. 使用spring boot如何创建一个可部署的war文件？

产生一个可部署war包的第一步是提供一个SpringBootServletInitializer子类，并覆盖它的configure方法。这充分利用了Spring框架对Servlet 3.0的支持，并允许你在应用通过servlet容器启动时配置它。通常，你只需把应用的主类改为继承SpringBootServletInitializer即可：
```
@SpringBootApplication
public class Application extends SpringBootServletInitializer {

    @Override
    protected SpringApplicationBuilder configure(SpringApplicationBuilder application) {
        return application.sources(Application.class);
    }

    public static void main(String[] args) throws Exception {
        SpringApplication.run(Application.class, args);
    }

}
```
下一步是更新你的构建配置，这样你的项目将产生一个war包而不是jar包。如果你使用Maven，并使用spring-boot-starter-parent（为了配置Maven的war插件），所有你需要做的就是更改pom.xml的packaging为war：
```
<packaging>war</packaging>
```
如果你使用Gradle，你需要修改build.gradle来将war插件应用到项目上：
```
apply plugin: 'war'
```
该过程最后的一步是确保内嵌的servlet容器不能干扰war包将部署的servlet容器。为了达到这个目的，你需要将内嵌容器的依赖标记为provided。
如果使用Maven：
```
<dependencies>
    <!-- … -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-tomcat</artifactId>
        <scope>provided</scope>
    </dependency>
    <!-- … -->
</dependencies>
```
如果使用Gradle：
```
dependencies {
    // …
    providedRuntime 'org.springframework.boot:spring-boot-starter-tomcat'
    // …
}
```
如果你使用Spring Boot构建工具，将内嵌容器依赖标记为provided将产生一个可执行war包，在lib-provided目录有该war包的provided依赖。这意味着，除了部署到servlet容器，你还可以通过使用命令行java -jar命令来运行应用。


43. 配置中心内测环境
http://192.168.1.205:8081/
这里修改，用户名密码 admin,admin

44. dubbo架构优化
```
1 合并soa_search,soa_index,soa_es的三个项目为一个SOA；考虑WEB项目合并精简
2 将WEB项目重构为spring boot项目，以JAR的方式发布
3 上CAT监控，埋点整体程序全部划入监控里
4 替换cas为kisso
5 开启dubbo的failover集群，开启spring boot的监听钩子，升级时确认容器线程被钩子正常销毁，服务会被nginx&dubbo自己切换过去到正常机器，以此实现无影响升级
6 开启dubbo服务降级功能，出现异常对异常服务及时降级
7 如果能实现代码结构调整为最好 common,soa(soa_base,soa_业务),web(web_base,web_业务），定时器，消息
8 调整jenkson打包的方式，目前依赖严重，直接影响发版
```
45. 各个环境的key
内测：alpha
公测：beta
灰度：abtest
正式：release
46. 关于java项目中如何读取配置文件
- Using multiple property files (via PropertyPlaceholderConfigurer) in multiple projects/modules
If you ensure that every place holder, in each of the contexts involved, is ignoring unresolvable keys then both of these approaches work. For example:
```
<context:property-placeholder
location="classpath:dao.properties,
          classpath:services.properties,
          classpath:user.properties"
ignore-unresolvable="true"/>
```
or
```
    <bean id="propertyConfigurer" class="org.springframework.beans.factory.config.PropertyPlaceholderConfigurer">
        <property name="locations">
            <list>
                <value>classpath:dao.properties</value>
                <value>classpath:services.properties</value>
                <value>classpath:user.properties</value>
            </list>
        </property> 
        <property name="ignoreUnresolvablePlaceholders" value="true"/>
    </bean>
```

- Spring MVC : read file from src/main/resources
```
Resource resource = new ClassPathResource(fileLocationInClasspath);
InputStream resourceInputStream = resource.getInputStream();
```

- How do I load a resource and use its contents as a string in Spring
```
<bean id="contents" class="org.apache.commons.io.IOUtils" factory-method="toString">
    <constructor-arg value="classpath:path/to/resource.txt" type="java.io.InputStream" />
</bean>
```
This solution requires Apache Commons IO.

Another solution, suggested by @Parvez, without Apache Commons IO dependency is
```
<bean id="contents" class="java.lang.String">
    <constructor-arg>
        <bean class="org.springframework.util.FileCopyUtils" factory-method="copyToByteArray">
            <constructor-arg value="classpath:path/to/resource.txt" type="java.io.InputStream" />
        </bean>     
    </constructor-arg>
</bean>
```

48. maven全局排除工具
http://maven.apache.org/enforcer/maven-enforcer-plugin/
实现maven依赖的全局排除
http://yongpoliu.com/maven-global-exclude/





51. nginx启动
/usr/local/nginx/sbin/nginx -c /usr/local/nginx/conf/nginx.conf

sh脚本安装nginx遇到的问题解决：
1.nginx: [emerg] getpwnam("www") failed
```
/usr/sbin/groupadd -f www
/usr/sbin/useradd -g www www
```

52. logstash 安装
安装前，需要先安装java

1、下载jdk并上传到/usr/java目录
jdk7下载地址为：http://www.oracle.com/technetwork/java/javase/downloads/jdk7-downloads-1880260.html 选择对应的linux版本，下载rpm文件。这里选择的是jdk-7u79-linux-x64.rpm。并上传到Linux的/urs/java目录下（java目录不存在则进行创建）。

2、解压安装
进入/usr/java目录，运行如下命令进行解压(rpm -ivh rpm文件名称)
```
rpm -ivh jdk-7u79-linux-x64.rpm
```

3、配置profile文件
运行如下命令
```
vi /etc/profile
```
将如下内容添加到profile文件末尾并保持
```
export JAVA_HOME=/usr/java/jdk1.7.0_79
export CLASSPATH=.:$JAVA_HOME/jre/lib/rt.jar:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar export PATH=$PATH:$JAVA_HOME/bin 
```
/usr/java/jdk1.7.0_79 指的是jdk的路径

保存之后，运行如下命令使配置生效
```
source /etc/profile
```
检查jdk是否安装成功，运行如下命令
```
java -version
```



YUM方式
Download and install the public signing key:
```
rpm --import https://packages.elastic.co/GPG-KEY-elasticsearch
```
Add the following in your /etc/yum.repos.d/ directory in a file with a .repo suffix, for example logstash.repo
```
[logstash-2.3]
name=Logstash repository for 2.3.x packages
baseurl=https://packages.elastic.co/logstash/2.3/centos
gpgcheck=1
gpgkey=https://packages.elastic.co/GPG-KEY-elasticsearch
enabled=1
```
And your repository is ready for use. You can install it with:
```
yum install logstash
```


53. linux关闭防火墙
一、关闭防火墙
1、重启后永久性生效：
开启：chkconfig iptables on
关闭：chkconfig iptables off
2、即时生效，重启后失效：
开启：service iptables start
关闭：service iptables stop
在开启了防火墙时，做如下设置，开启相关端口，修改 /etc/sysconfig/iptables 文件，添加以下内容：
-A INPUT -m state --state NEW -m tcp -p tcp --dport 80 -j ACCEPT   #允许80端口通过防火墙
-A INPUT -m state --state NEW -m tcp -p tcp --dport 3306 -j ACCEPT   #允许3306端口通过防火墙
备注：很多网友把这两条规则添加到防火墙配置的最后一行，导致防火墙启动失败，
正确的应该是添加到默认的22端口这条规则的下面


54. java笔试面试高级内容
```
List,Set,Map用法以及区别-http://j2eemylove.iteye.com/blog/1195823
Hash算法-http://www.cnblogs.com/wangjy/archive/2011/09/08/2171638.html
单例模式的七种写法-http://cantellow.iteye.com/blog/838473
java中的匿名内部类总结 - http://www.cnblogs.com/nerxious/archive/2013/01/25/2876489.html
Java 内部类种类及使用解析 - http://www.cnblogs.com/mengdd/archive/2013/02/08/2909307.html
hibernate级联保存 - http://www.iteye.com/topic/2312?page=2
springsecurity3- http://www.mossle.com/docs/springsecurity3/html/technical-overview.html
HashMap实现原理分析 - http://blog.csdn.net/vking_wang/article/details/14166593
使用简单的 5 个步骤设置 Web 服务器集群 - http://www.ibm.com/developerworks/cn/linux/l-linux-ha/
java nio实现非阻塞Socket通信实例 - http://blog.csdn.net/x605940745/article/details/14123343
Java深入 - Java Socket和NIO - http://blog.csdn.net/initphp/article/details/39271755
```

55. spring-boot jar、war两种方式打包运行
1,jar

增加依赖
```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```
配置文件配置启用shutdown的HTTP访问
```
#启用 shutdown endpoint的HTTP访问
endpoints.shutdown.enabled=true
#不需要用户名密码验证 
endpoints.shutdown.sensitive=false
```
curl -X POST localhost:port/shutdown 发动post请求，即可优雅的停止应用：

spring boot maven项目打包运行和停止

1.获取到代码后，编译打包。使用命令：
```
mvn clean package
```
2.停止正在运行的spring boot应用。使用命令：
```
curl -X POST {host}:{port}/shutdown
```
{host}是应用部署的ip地址，{port}是应用部署的端口，替换成对应的参数即可

3.运行应用。
加入打包后的jar包名为app.jar，那么运行这个应用的命令是：
```
java -jar app.jar
```
以后台程序的方式启动，并指定输出日志的路径。最后使用命令：
```
nohup java -jar app.jar < /dev/null > /data/logs/app.log 2>&1 &
```


2,war

创建一个可部署的war文件

产生一个可部署war包的第一步是提供一个SpringBootServletInitializer子类，并覆盖它的configure方法。这充分利用了Spring框架对Servlet 3.0的支持，并允许你在应用通过servlet容器启动时配置它。通常，你只需把应用的主类改为继承SpringBootServletInitializer即可：
```
@SpringBootApplication
public class Application extends SpringBootServletInitializer {

    @Override
    protected SpringApplicationBuilder configure(SpringApplicationBuilder application) {
        return application.sources(Application.class);
    }

    public static void main(String[] args) throws Exception {
        SpringApplication.run(Application.class, args);
    }

}
```
下一步是更新你的构建配置，这样你的项目将产生一个war包而不是jar包。如果你使用Maven，并使用spring-boot-starter-parent（为了配置Maven的war插件），所有你需要做的就是更改pom.xml的packaging为war：
```
<packaging>war</packaging>
```
该过程最后的一步是确保内嵌的servlet容器不能干扰war包将部署的servlet容器。为了达到这个目的，你需要将内嵌容器的依赖标记为provided。

如果使用Maven：
```
<dependencies>
    <!-- … -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-tomcat</artifactId>
        <scope>provided</scope>
    </dependency>
    <!-- … -->
</dependencies>
```


56. SSH key的生成和使用

git config --global user.name "yihuawanglv"
git config --global user.email wanglvyihua@gmail.com

先看下是否已存在sshkey信息：
```
ls -al ~/.ssh
```
生成秘钥：
```
ssh-keygen -t rsa -C "wanglvyihua@gmail.com"
```

```
eval "$(ssh-agent -s)"
```
添加密钥到ssh：
```
ssh-add ~/.ssh/id_rsa
```
过程中会要求输入一些密码，这里直接回车留空即可.

查看并复制秘钥，保存到github：
```
cat /root/.ssh/id_rsa.pub
```
查看key，复制sshkey，登陆github，保存key到sshkey列表

测试一下：
```
ssh git@github.com
```
会收到一下信息：
Hi YihuaWanglv! You've successfully authenticated, but GitHub does not provide shell access

最后一步：
在你检出的某个本地repository中，修改.git/config内的配置，由https改为ssh或git.
Edit your .git/config file so that the url is using either ssh or git protocol instead of https:

url = git@github.com:yourgithubaccount/yourgithubrepository.git

然后就可以不用密码直接ssh提交git了。


