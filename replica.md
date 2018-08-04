
### 配置环境

虚拟机上搭建两台服务器，配置如下:

|配置项|主数据库|从数据库|
|:-----:|:------:|:------:|
|Linux系统| CentOS 6.5| CentOS 6.5|
|MySQL版本| 5.1.73| 5.1.73|
|IP地址| 192.168.0.66 |192.168.0.120|


### 配置复制

#### 配置步骤：

1. 每台服务上创建复制账号;
2. 配置主库和备库;
3. 通知备库连接到主库并从主库复制数据;


#### 配置过程：

##### 第1步：创建复制账号

在主库(192.168.0.66和从库(192.168.0.120)上，进行同步账号创建跟授权，如下:
*	创建同步用户账号跟密码， 用户账号: repl , 密码: password；
*	给repl用户授权复制限制，并且限制只能在192.169.0. 网络网段上。
```
mysql> CREATE USER 'repl'@'192.168.0.%' IDENTIFIED BY 'password';
mysql> GRANT REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'repl'@'192.168.0.%';
```
说明

* 复制账户其实只需要有主库上的**REPLICATION SLAVE**权限, 并不一定需要都有**REPLICATION CLIENT**权限
*  **REPLICATION CLIENT** 主要有2方面作用:
		(1). 监控和管理复制的账号
		(2). 方法交换主从库的角色
		
![Alt text](/images/replica/20180802224620.png)

##### 第2步：配置主库跟备库配置文件

**配置主库**

编辑  MySQL配置文件  $/etc/my.cnf$ 

```
$ vim /etc/my.cnf
```
添加或修改以下2项:
```
[mysqld]
log-bin = mysql-bin
server-id = 10
```

注意:
*	log-bin 跟 server-id 一定要在 **[mysqld]** 下配置，否则其他地方可能不会生效；
* server_id 一般使用服务器IP地址末尾8位，但要保证它不变且唯一的


**如果之前没在MySQL配置文件中指定log-bin选项，那就得重启MySQL**
```
service mysqld restart
```

![20180802230603](/images/replica/20180802230603.png)

![主库配置](/images/replica/20180802225919.png)

检查主库是否配置成功

```
mysql> show master status;
```

![Alt text](./images/replica/20180804102605.png)


**配置从库**

同样在$/etc/my.cnf$ 文件上的**[mysqld]**添加或修改以下2项:
```
[mysqld]
log-bin = mysql-bin
server-id = 2
relay-log = /var/lib/mysql/mysql-relay-bin
log_slave_updates = 1
read_only = 1
```
说明

* relay-log = /var/lib/mysql/mysql-relay-bin ： 指定中继日志的位置和命名
* log_slave_updates = 1 ：允许备库将其重放的事件也记录到自身的二进制日志中
* read_only :会阻止任何没有特权权限的线程修改数据

![Alt text](./images/replica/20180804152751.png)


重启数据库，同样检查是否配置成功


```
mysql> show master status;
```

![Alt text](./images/replica/20180804102605.png)

此时，到这里，主从库配置文件**/etc/my.cnf**就配置成功了，接下来第三步启动复制。

##### 第三步：启动复制