## 安装



压缩目录，在opt目录解压

/opt  



默认安装目录

/usr/bin



复制修改配置文件



绑定服务器要使用的配置文件，且==启动服务器==

redis-server /myredis/redis.conf



连接服务

redis-cli -a 111111 -p 6379(-a 后面是redis设置的密码，6379是默认端口号)



启动之后验证是否正确启动

ping  返回pong代表启动成功



退出连接

quit(并没有关闭服务器，只是退出连接的客户端)

SHUTDOWN（关闭服务器） quit(关闭服务器后退出连接)

关闭方法2

redis-cli -a 111111 shutdown

多实例关闭，指定端口关闭：redis-cli -p 6379 shutdown

==启动服务器==

redis-server /myredis/redis7.conf





hello word

```redis
set k1 helloword 
set k1 
```



## 卸载Redis步骤

1.停止redis-server服务

![](https://zhangwenkang666.oss-cn-beijing.aliyuncs.com/5.%E5%81%9C%E6%AD%A2redis-server%E6%9C%8D%E5%8A%A1.png)

2.删除/usr/local/bin目录下与redis相关的文件

ls -l /usr/local/bin/redis-*

rm -rf /usr/local/bin/redis-*

![](https://zhangwenkang666.oss-cn-beijing.aliyuncs.com/6.%E5%88%A0%E9%99%A4redis%E6%96%87%E4%BB%B6.png)



## 十大数据类型







## redis持久化快照

rdb:缓存快照

AOF：记录写操作





## redis 事物





## redis管道





## redis发布订阅



## redis复制



## redis哨兵













