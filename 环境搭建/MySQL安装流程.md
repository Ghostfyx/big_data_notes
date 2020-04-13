# Linux环境下MySQL的安装

## 一、YUM安装Hive

### 1.1 删除已安装的MySQL

**检查MariaDB**

```sh
shell> rpm -qa|grep mariadb
mariadb-server-5.5.60-1.el7_5.x86_64
mariadb-5.5.60-1.el7_5.x86_64
mariadb-libs-5.5.60-1.el7_5.x86_64
```

**删除mariadb**

如果不存在（上面检查结果返回空）则跳过步骤

```sh
shell> rpm -e --nodeps mariadb-server
shell> rpm -e --nodeps mariadb
shell> rpm -e --nodeps mariadb-libs
```

其实yum方式安装是可以不用删除mariadb的，安装MySQL会覆盖掉之前已存在的mariadb。

### 1.2 添加MySQL Yum Repository

从CentOS 7开始，MariaDB成为Yum源中默认的数据库安装包。也就是说在CentOS 7及以上的系统中使用yum安装MySQL默认安装的会是MariaDB（MySQL的一个分支）。如果想安装官方MySQL版本，需要使用MySQL提供的Yum源。

**下载MySQL源**

官网地址：[dev.mysql.com/downloads/r…](https://dev.mysql.com/downloads/repo/yum/)

查看系统版本：

```sh
shell> cat /etc/redhat-release
CentOS Linux release 7.7.1908 (Core)
```

选择对应的版本进行下载，例如CentOS 7当前在官网查看最新Yum源的下载地址为： [dev.mysql.com/get/mysql80…](https://dev.mysql.com/get/mysql80-community-release-el7-3.noarch.rpm)

```sh
shell> wget https://dev.mysql.com/get/mysql80-community-release-el7-3.noarch.rpm
```

**安装MySQL源**

```sh
shell> sudo rpm -Uvh platform-and-version-specific-package-name.rp
```

**检查是否安装成功**

执行成功后会在`/etc/yum.repos.d/`目录下生成两个repo文件`mysql-community.repo`及 `mysql-community-source.repo`

并且通过`yum repolist`可以看到mysql相关资源

```sh
shell> yum repolist enabled | grep "mysql.*-community.*"
!mysql-connectors-community/x86_64       MySQL Connectors Community          141
!mysql-tools-community/x86_64            MySQL Tools Community               105
!mysql57-community/x86_64                MySQL 5.7 Community Server          404
```

### 1.3 选择MySQL版本

使用MySQL Yum Repository安装MySQL，默认会选择当前最新的稳定版本，例如通过上面的MySQL源进行安装的话，默安装会选择MySQL 8.0版本，如果就是想要安装该版本，可以直接跳过此步骤，如果不是，比如我这里希望安装MySQL5.7版本，就需要“切换一下版本”：

##### 查看当前MySQL Yum Repository中所有MySQL版本（每个版本在不同的子仓库中）

```sh
shell> yum repolist all | grep mysql
mysql-cluster-7.5-community/x86_64 MySQL Cluster 7.5 Community   disabled
mysql-cluster-7.5-community-source MySQL Cluster 7.5 Community - disabled
mysql-cluster-7.6-community/x86_64 MySQL Cluster 7.6 Community   disabled
mysql-cluster-7.6-community-source MySQL Cluster 7.6 Community - disabled
!mysql-connectors-community/x86_64 MySQL Connectors Community    enabled:    141
mysql-connectors-community-source  MySQL Connectors Community -  disabled
!mysql-tools-community/x86_64      MySQL Tools Community         enabled:    105
mysql-tools-community-source       MySQL Tools Community - Sourc disabled
mysql-tools-preview/x86_64         MySQL Tools Preview           disabled
mysql-tools-preview-source         MySQL Tools Preview - Source  disabled
mysql55-community/x86_64           MySQL 5.5 Community Server    disabled
mysql55-community-source           MySQL 5.5 Community Server -  disabled
mysql56-community/x86_64           MySQL 5.6 Community Server    disabled
mysql56-community-source           MySQL 5.6 Community Server -  disabled
!mysql57-community/x86_64          MySQL 5.7 Community Server    enabled:    404
mysql57-community-source           MySQL 5.7 Community Server -  disabled
mysql80-community/x86_64           MySQL 8.0 Community Server    disabled
mysql80-community-source           MySQL 8.0 Community Server -  disabled

```

##### 切换版本

编辑`/etc/yum.repos.d/mysql-community.repo`文件，enabled=0禁用，enabled=1启用

### 1.4 安装MySQL

```sh
yum install mysql-community-server
```

**启动MySQL**

```
systemctl start mysqld.service
```

**查看MySQL状态**

```sh
systemctl status mysqld.service
```

**设置为开机启动**

```sh
systemctl enable mysqld.service
```

### 1.5 修改密码

MySQL第一次启动后会创建超级管理员账号`root@localhost`，初始密码存储在日志文件中：

```sh
grep 'temporary password' /var/log/mysqld.log
```

**修改默认密码**

```
mysql -uroot -p
```

输入日志文件中的初始密码。

```sh
mysql> ALTER USER 'root'@'localhost' IDENTIFIED BY '123456';
```

如果出现

```sh
ERROR 1819 (HY000): Your password does not satisfy the current policy requirements
```

解决方案：

输入语句 `SHOW VARIABLES LIKE 'validate_password%';`查看，需要设置密码的验证强度等级，设置 validate_password_policy 全局参数为 LOW ，输入设值语句 `set global validate_password_policy=LOW;` 进行设值；修改密码长度，输入设值语句 ` set global validate_password_length=${your_pwd_length}; ` 进行设值，

关于 mysql 密码策略相关参数；

1）validate_password_length  固定密码的总长度；

2）validate_password_dictionary_file 指定密码验证的文件路径；

3）validate_password_mixed_case_count  整个密码中至少要包含大/小写字母的总个数；

4）validate_password_number_count  整个密码中至少要包含阿拉伯数字的个数；

5）validate_password_policy 指定密码的强度验证等级，默认为 MEDIUM；
				关于 validate_password_policy 的取值：
					LOW：只验证长度；
					MEDIUM：验证长度、数字、大小写、特殊字符；
					2STRONG：验证长度、数字、大小写、特殊字符、字典文件；
		6）validate_password_special_char_count 整个密码中至少要包含特殊字符的个数；

### 1.6 允许root远程访问

```sh
mysql> GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY '123456' WITH GRANT OPTION;
mysql> FLUSH PRIVILEGES;
```

### 1.7 设置编码为utf8

##### 查看编码

```sh
mysql> SHOW VARIABLES LIKE 'character%';
```

##### 设置编码

编辑/etc/my.cnf，[mysqld]节点增加以下代码：

```sh
[mysqld]
character_set_server=utf8
init-connect='SET NAMES utf8'
```

### 1.8 远程测试连接MySQL

MySQL服务有可能状态本地、虚拟机或者远程机器，可使用本地navicat等工具测试连接。