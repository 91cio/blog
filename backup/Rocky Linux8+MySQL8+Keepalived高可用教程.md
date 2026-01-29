##MySQL双机热备和主备复制高可用
首先准备两台Rocky Linux8服务器，IP分别为
192.168.5.71
192.168.5.72
分别都关闭Firewalld和selinux

一、安装和初始化MySQL
两台主机都要安装
命令
dnf  install  @mysql  -y
两台主机启动服务和开机启动mysql
命令
systemctl  start  mysqld
systemctl  enable  mysqld
两台主机初始化数据库
命令
mysql_secure_installation
初始化设置交互窗口：
NOTE: RUNNING ALL PARTS OF THIS SCRIPT IS RECOMMENDED FOR ALL MySQL
SERVERS IN PRODUCTION USE! PLEASE READ EACH STEP CAREFULLY!
In order to log into MySQL to secure it, we'll need the current
password for the root user. If you've just installed MySQL, and
you haven't set the root password yet, the password will be blank,
so you should just press enter here.
 
Enter current password for root (enter for none): <–-初次运行直接回车
OK, successfully used password, moving on…
Setting the root password ensures that nobody can log into the MySQL
root user without the proper authorisation.
Set root password? [Y/n]    //是否设置root用户密码，输入Y并回车或直接回车
 
New password:               //设置root用户的密码
Re-enter new password:      //再次输入你设置的密码
Password updated successfully!
Reloading privilege tables..
… Success!
 
By default, a MySQL installation has an anonymous user, allowing anyone
to log into MySQL without having to have a user account created for
them. This is intended only for testing, and to make the installation
go a bit smoother. You should remove them before moving into a
production environment.
Remove anonymous users? [Y/n]   //是否删除匿名用户,生产环境建议删除，所以直接回车
… Success!
 
Normally, root should only be allowed to connect from 'localhost'. This
ensures that someone cannot guess at the root password from the network.
Disallow root login remotely? [Y/n] //是否禁止root远程登录,根据自己的需求选择Y/n并回车,建议禁止
… Success!
 
By default, MySQL comes with a database named 'test' that anyone can
access. This is also intended only for testing, and should be removed
before moving into a production environment.
Remove test database and access to it? [Y/n] //是否删除test数据库,直接回车
- Dropping test database…
… Success!
- Removing privileges on test database…
… Success!
 
Reloading the privilege tables will ensure that all changes made so far
will take effect immediately.
Reload privilege tables now? [Y/n] //是否重新加载权限表，直接回车
… Success!
Cleaning up…
All done!
完成交互式初始化设置

二、MySQL服务高可用配置
1、创建同步用户
登录数据库
mysql -uroot -p
使用mysql库，
use mysql
 
分别在两台主机创建同步用户，该帐户必须授予REPLICATION SLAVE权限，因为mysql8在授权语句中不能出现IDENTIFIED BY ‘password’;，因此创建用户和授予权限需要分开执行：
1) 在192.168.5.71上执行创建用户并赋权：
CREATE USER 'dbsync'@'192.168.5.72' IDENTIFIED WITH 'mysql_native_password' BY 'abc@123';
GRANT REPLICATION SLAVE ON *.* TO 'dbsync'@'192.168.5.72';
2) 在192.168.5.72上执行创建用户并赋权：
CREATE USER 'dbsync'@'192.168.5.71' IDENTIFIED WITH 'mysql_native_password' BY 'abc@123';
GRANT REPLICATION SLAVE ON *.* TO 'dbsync'@'192.168.5.71';
两台主机分别执行flush privileges;以刷新权限


2、修改mysql配置文件
1）192.168.5.71主机配置/etc/my.cnf文件。在文件末尾加入以下配置：
[mysqld]
server-id=1
innodb_flush_log_at_trx_commit=2
log-bin=mysql-bin-1
binlog-ignore-db=mysql,information_schema,performance_schema,sys
sync_binlog=1
slave_skip_errors=1146
2）192.168.5.72主机配置/etc/my.cnf文件。在文件末尾加入以下配置：
[mysqld]
server-id=2
innodb_flush_log_at_trx_commit=2
log-bin=mysql-bin-2
binlog-ignore-db=mysql,information_schema,performance_schema,sys
sync_binlog=1
slave_skip_errors=1146
 
三、重启并配置同步
1.重启两台主机上的mysql
两台主机分别执行：systemctl restart mysqld
2.查看两个服务器作为主服务器的状态
mysql命令
show master statusG



3.用change mster 语句指定同步位置
1）两台主机进入mysql操作界面（mysql -uroot -p），分别执行如下指令：
unlock tables;
否则在执行stop slave;时会报如下异常：ERROR 1192 (HY000): Can’t execute the given command because you have active locked tables or an active transaction。
2）两台主机停步slave服务线程，这个是很重要的，如果不这样做会造成以下操作不成功。分别执行如下指令：
stop slave;
3）在192.168.5.71主机执行
change master to
master_host='192.168.5.72',master_user='dbsync',master_password='abc@123',
master_log_file='mysql-bin-2.000001',master_log_pos=749;  （master_log_pos使用上面图片中的Position的值）
4）在192.168.5.72主机执行
change master to
master_host='192.168.5.71',master_user='dbsync',master_password='abc@123',
master_log_file='mysql-bin-1.000001',master_log_pos=749;   （master_log_pos使用上面图片中的Position的值）
5）分别在服务器上重启从服务线程：
start slave;
4.查看从服务器状态
mysql命令
show slave statusG
如果如下两项状态为yes则说明配置成功：
Slave_IO_Running: Yes
Slave_SQL_Running: Yes


四、验证MySQL双机同步复制
需要使用到的mysql命令：
创建数据库CREATE DATABASE `test`;
删除数据库DROP DATABASE test;
查看数据库SHOW DATABASES;

五、配置keepalived实现mysql主备高可用

1.安装keepalived
命令
两台主机执行dnf  install  keepalived  -y
2、配置192.168.5.71主机 /etc/keepalived/keepalived.conf
vim /etc/keepalived/keepalived.conf
配置如下参数：
 
global_defs {
   router_id MySQL_DEVEL
}
 
vrrp_script check_mysql {
script "killall -0 mysqld"
interval 2
}
 
vrrp_instance HA_1 {
    state BACKUP
    interface ens32
    virtual_router_id 73
    priority 100
    advert_int 2
    nopreempt
    authentication {
        auth_type PASS
        auth_pass 1111
}
 
track_script {
check_mysql
}
 
    virtual_ipaddress {
        192.168.5.73/24 dev ens32
    }
}
 
3、配置192.168.5.72主机 /etc/keepalived/keepalived.conf
vim /etc/keepalived/keepalived.conf
配置如下参数：
 
global_defs {
   router_id MySQL_DEVEL
}
 
vrrp_script check_mysql {
script "killall -0 mysqld"
interval 2
}
 
vrrp_instance HA_1 {
    state BACKUP
    interface ens32
    virtual_router_id 73
    priority 80
    advert_int 2
    authentication {
        auth_type PASS
        auth_pass 1111
}
 
track_script {
check_mysql
}
 
    virtual_ipaddress {
        192.168.5.73/24 dev ens32
    }
}

4、两台主机依次执行
systemctl start keepalived
systemctl enable keepalived

六、Keepalived+MySQL8高可用主备验证

远程客户端通过虚IP 192.168.5.73登录数据库
使用命令show variables like '%hostname';查看虚IP所在的节点