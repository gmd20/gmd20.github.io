```text
#  忘了密码时可以按照下面步骤重置root密码
https://dev.mysql.com/doc/refman/5.7/en/resetting-permissions.html
B.5.3.2.3 Resetting the Root Password: Generic Instructions

# 增强密码安全性，管理你们用户和免密码登录
2.10.4 Securing the Initial MySQL Accounts
https://dev.mysql.com/doc/refman/5.7/en/default-privileges.html

# 用户管理
6.3 MySQL User Account Management
https://dev.mysql.com/doc/refman/5.7/en/user-account-management.html

centos安装完第一次启动，默认密码是空的 先试试这个，不行再下面的
mysql -u root  -p    提示密码时直接回车用空密码登录
mysql -u user124 随便输入一个用户名，用匿名用户登录，默认匿名用户可以用本地免密码登录，为了安全起见可以删掉匿名用户
mysqladmin -u root password newpass  可以直接改密码

如果知道旧的密码可以这样修改
$ mysqladmin -u root -p oldpassword newpass

旧版本mysql正常改密码的sql
mysql -u root -p
  use mysql;
  SET PASSWORD FOR 'root'@'localhost' = PASSWORD('MyNewPass');
  SET PASSWORD FOR 'root' = PASSWORD('new_password');
  update mysql.user set password=PASSWORD("MyNewPass") where User='root';
  exit


官方文档的重置root密码的步骤：
 --skip-grant-tables This enables anyone to connect without a password and
 with all privileges, and disables account-management statements such as ALTER
 USER and SET PASSWORD. Because this is insecure, you might want to use
 --skip-grant-tables in conjunction with --skip-networking to prevent remote clients from connecting.

sudo mysqld_safe --skip-grant-table --skip-networking &
sudo mysqld_safe --skip-grant &  这个好像也可以
mysql  用上面的安全模式启动，就不需要密码和用户了
  flush privileges; 这个只能成功执行一次，然后需要重启mysqld服务才能做第二次了  tell the server to reload the grant tables so that account-management statements work:
  select host,user,password from mysql.user;
  update mysql.user set password=PASSWORD("MyNewPass") where User='root';
  exit
sudo service mysqld stop

mysql> select host,user,password from mysql.user;
+-----------+------+------------------+
| host      | user | password         |
+-----------+------+------------------+
| localhost | root | dsdff77c5d195499 | 必须有这个 localhost这条才能用 mysql -u root -p 从本地登录
| 127.0.0.1 | root | dsdff77c5d195499 |
| localhost |      |                  | 这个空白用户名，密码也为空，就是允许匿名用户 不需要密码也可以本地登录
+-----------+------+------------------+


# 开放远程登陆
  select host,user,password from mysql.user;
  GRANT ALL PRIVILEGES ON *.* TO 'root'@'192.168.1.100' IDENTIFIED BY 'newpassword' WITH GRANT OPTION;  创建一个个用户允许192.168.1.100ip来登陆
  grant all privileges  on *.* to 'root'@'%' identified by "newpassword" WITH GRANT OPTION;  创建允许所有ip登录的root账号
  flush privileges;

#修复安装的表问题？
mysql_install_db --user=mysql
这个加强安全？会提示设置root密码，删掉匿名用户
mysql_secure_installation

登陆数据库
mysql -u root -p  然后输入密码
mysql -h 127.0.0.1 -P 3306 -u root -p

```
