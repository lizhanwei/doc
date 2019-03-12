
# 1类SnowFlake方案
## 1.1 UUID
> 550e8400-e29b-41d4-a716-446655440000  

### 五种生成UUID的方法
### 1.1.1 基于时间的UUID
通过当前时间戳、机器MAC地址生成。  
UUID('444b5cc0-ae5d-11e6-8d22-28924a431726')

### 1.1.2 DCE安全的UUID
DCE安全的UUID和基于时间的UUID算法相同，但会把时间戳的前4位置换为POSIX的UID或GID。不过，在UUID的规范里面没有明确地指定，所以基本上所有的UUID实现都不会实现这个版本。

### 1.1.3 基于名字空间的UUID（MD5）
由用户指定1个namespace和1个具体的字符串，通过MD5散列，来生成1个UUID  
```java 
System.out.println(UUID.nameUUIDFromBytes("myString".getBytes("UTF-8")).toString());
```

### 1.1.4 基于随机数的UUID
根据随机数，或者伪随机数生成UUID
```java 
System.out.println(UUID.randomUUID().toString());
```

### 1.1.5 基于名字空间的UUID（SHA1）
和版本3一样，不过散列函数换成了SHA1


[java-uuid-generator](https://github.com/cowtowncoder/java-uuid-generator)

| 优点        | 缺点    | 
| --------   | :----- | 
|性能非常高：本地生成，没有网络消耗|不易于存储，需要36长度的字符串<br>基于Mac的UUID算法会导致Mac地址泄露<br>无序导致数据位置频繁变动影响性能|

## 1.2 Mongodb的ObjectID
例如：4df2dcec2cdcd20936a8b817

|   4df2dcec      | 2cdcd2    | 0936 | a8b817 |
| :-------   | :----- | :----- | :----- | 
|time|machine|pid|inc|
|时间戳：1307761900的十六进制表示|主机名哈希值|进程ID|自增计数器|

当然你也可以自己设计ID策略，比如：
> 时间戳 + IP的C,D段 + 进程ID + 自增序列 + 随机数


## 1.3 SnowFlake算法
图  
[1位正负标识位][41位时间戳][10位机器ID][12位自增]
1位：不用。二进制中最高位为1的都是负数，但是我们生成的id一般都使用整数，所以这个最高位固定是0  
41位：用来记录时间戳（毫秒），可以保存69年的数据  
10位：用来记录工作机器id，可以存储1024个  
12位：序列号，每毫秒产生4095个ID

[SnowFlake算法](https://segmentfault.com/a/1190000011282426)  

# 2 类数据库自增  
## 2.1 数据库
主要思路是采用数据库自增ID + replace_into实现唯一ID的获取。
```sql
create table t_global_id(
    id bigint(20) unsigned not null auto_increment,
    stub char(1) not null default '',
    primary key (id),
    unique key stub (stub)
) engine=MyISAM;

# 每次业务可以使用以下SQL读写MySQL得到ID号
replace into t_golbal_id(stub) values('a');
select last_insert_id();

# 步长配置 
Server1：
auto-increment-increment = 2
auto-increment-offset = 1

Server2：
auto-increment-increment = 2
auto-increment-offset = 2

```

## 2.2 Redis
当使用数据库来生成ID性能不够要求的时候，我们可以尝试使用Redis来生成ID。这主要依赖于Redis是单线程的，所以也可以用生成全局唯一的ID。可以用Redis的原子操作 INCR和INCRBY来实现。

## 2.3 Zk
zookeeper主要通过其znode数据版本来生成序列号，可以生成32位和64位的数据版本号，客户端可以使用这个版本号来作为唯一的序列号。很少会使用zookeeper来生成唯一ID。主要是由于需要依赖zookeeper，并且是多步调用API，如果在竞争较大的情况下，需要考虑使用分布式锁。因此，性能在高并发的分布式环境下，也不甚理想。

# 3 ID解决方案
类SnowFlake方案由于依赖系统时钟存在时钟回拨的问题，因此需要在中心节点来3秒钟维持当前时间，发现回拨及时失败；类数据库自增ID方案由于每次请求需要访问存储因此采用每次批量获取一段ID来应对。