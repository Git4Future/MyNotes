# WorkNote

## Linux

### 如何在Linux服务器部署SVN服务?

***环境:centos7.5,svnserve***

**1.检查是否已经安装**

```
rpm -qa | grep subversion
```

如果已经安装过会有如下显示:

![1582680744880](./image/1582680744880.png)

**2.通过yum命令安装svnserve**

```
yum -y install subversion
```

**3.检查是否安装成功**

```
svnserve --version
```

![1582680851571](./image/1582680851571.png)

**4.创建仓库根目录**

```
mkdir -p /home/svn/svnrepos
```

mkdir ： 如果存在上级目录不存在的情况，会无法创建下级目录, 会提出ERROR

![1582681063150](./image/1582681063150.png)

mkdir -p ：递归创建目录，即使上级目录不存在，会按目录层级自动创建目录

**5.创建一个svn仓库**

```
svnadmin create /home/svn/svnrepos/documents
```

![1582681241142](./image/1582681241142.png)

**6.查看新建好的仓库目录**

![1582681326919](./image/1582681326919.png)

仓库目录说明:

```
hooks目录：放置hook脚步文件的目录
locks目录：用来放置subversion的db锁文件和db_logs锁文件的目录，用来追踪存取文件库的客户端
format目录：是一个文本文件，里边只放了一个整数，表示当前文件库配置的版本号
conf目录：是这个仓库配置文件（仓库用户访问账户，权限）db目录:仓库数据目录
```

**7.修改SVN服务配置文件**

```
vim svnserve.conf
```

**去掉以下配置项的##注释,并且注意顶行不要有空格**

```
anon-access = read   # 匿名用户可读，改成none，禁止匿名访问
auth-access = write   # 授权用户可写
password-db = passwd   # 使用哪个文件作为账号文件
authz-db = authz   # 使用哪个文件作为权限文件
```

```c++
### This file controls the configuration of the svnserve daemon, if you
### use it to allow access to this repository.  (If you only allow
### access through http: and/or file: URLs, then this file is
### irrelevant.)

### Visit http://subversion.apache.org/ for more information.

[general]
### The anon-access and auth-access options control access to the
### repository for unauthenticated (a.k.a. anonymous) users and
### authenticated users, respectively.
### Valid values are "write", "read", and "none".
### Setting the value to "none" prohibits both reading and writing;
### "read" allows read-only access, and "write" allows complete 
### read/write access to the repository.
### The sample settings below are the defaults and specify that anonymous
### users have read-only access to the repository, while authenticated
### users have read and write access to the repository.
anon-access = read  ###注意前面不要有空格
auth-access = write ###注意前面不要有空格
### The password-db option controls the location of the password
### database file.  Unless you specify a path starting with a /,
### the file's location is relative to the directory containing
### this configuration file.
### If SASL is enabled (see below), this file will NOT be used.
### Uncomment the line below to use the default password file.
password-db = passwd  ###注意前面不要有空格
### The authz-db option controls the location of the authorization
### rules for path-based access control.  Unless you specify a path
### starting with a /, the file's location is relative to the the
### directory containing this file.  If you don't specify an
### authz-db, no path-based access control is done.
### Uncomment the line below to use the default authorization file.
authz-db = authz  ###注意前面不要有空格
### This option specifies the authentication realm of the repository.
### If two repositories have the same authentication realm, they should
### have the same password database, and vice versa.  The default realm
### is repository's uuid.
realm = My First Repository
### The force-username-case option causes svnserve to case-normalize
### usernames before comparing them against the authorization rules in the
### authz-db file configured above.  Valid values are "upper" (to upper-
### case the usernames), "lower" (to lowercase the usernames), and
### "none" (to compare usernames as-is without case conversion, which
### is the default behavior).
# force-username-case = none

[sasl]
### This option specifies whether you want to use the Cyrus SASL
### library for authentication. Default is false.
### This section will be ignored if svnserve is not built with Cyrus
### SASL support; to check, run 'svnserve --version' and look for a line
### reading 'Cyrus SASL authentication is available.'
# use-sasl = true
### These options specify the desired strength of the security layer
### that you want SASL to provide. 0 means no encryption, 1 means
### integrity-checking only, values larger than 1 are correlated
### to the effective key length for encryption (e.g. 128 means 128-bit
### encryption). The values below are the defaults.
# min-encryption = 0
# max-encryption = 256
```

**8.配置账户密码文件**

```
vim passwd 
```

新建一个用户账户:langzhe,密码为langzhe

![1582682937209](./image/1582682937209.png)

**9.配置访问权限文件**

```
vim authz
```

新建研发分组DEV_GROUP并且为用户langzhe和DEV_GROUP组配置访问权限

![1582683487984](./image/1582683487984.png)

**10.修改svnserve的配置文件，指定版本库的路径**

```
vim /etc/sysconfig/svnserve
```

```
用来明确-r参数的确切仓库位置
```

![1582683720475](./image/1582683720475.png)

**11.启动svn服务**

**12.查看SVN服务状态是否正常**

```
ps -ef | grep svn
```

![1582684431746](./image/1582684431746.png)

```java
ps
-e : 显示所有进程
-f : 全格式输出
-a : 显示所有进程
-u : by effective user ID (supports names)
-x : processes w/o controlling ttys 
```

```
netstat -antpl | grep svnserve
```

![1582684920295](./image/1582684920295.png)

```c
netstat常见参数
-a或--all：显示所有选项，默认不显示LISTEN相关
-t或--tcp：仅显示tcp相关选项
-u或--udp：仅显示udp相关选项
-n或--numeric：直接使用ip地址，而不通过域名服务器（拒绝显示别名，能显示数字的全部转化成数字）
-l或--listening：仅列出有在 Listen (监听) 的服務状态
-p或--programs：显示建立相关链接的程序名
-r或--route：显示路由信息，路由表
-e或--extend：显示扩展信息，例如uid等
-s或--statistice：显示网络工作信息统计表（按各个协议进行统计）
-c或--continuous：每隔一个固定时间，执行该netstat命令。
提示：LISTEN和LISTENING的状态只有用-a或者-l才能看到
```

**13.确认服务无误后,访问通过客户端访问SVN服务**

```
svn://ip地址/documents
```



























## Java