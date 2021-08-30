# Linux 设置开机启动项的几种方法
## 方法一：编辑 rc.loacl 脚本

Ubuntu 开机之后会执行 / etc/rc.local 文件中的脚本。

所以我们可以直接在 / etc/rc.local 中添加启动脚本。

```
$ vim /etc/rc.local

```

## 方法二：添加一个开机启动服务。

将你的启动脚本复制到 /etc/init.d 目录下，并设置脚本权限, 假设脚本为 test

```
$ mv test /etc/init.d/test
$ sudo chmod 755 /etc/init.d/test

```

将该脚本放倒启动列表中去

```
$ cd .etc/init.d
$ sudo update-rc.d test defaults 95

```

注：其中数字 95 是脚本启动的顺序号，按照自己的需要相应修改即可。在你有多个启动脚本，而它们之间又有先后启动的依赖关系时你就知道这个数字的具体作用了。

将该脚本从启动列表中剔除

```
$ cd /etc/init.d
$ sudo update-rc.d -f test remove

```

 [https://blog.csdn.net/autoliuweijie/article/details/73279136](https://blog.csdn.net/autoliuweijie/article/details/73279136)
