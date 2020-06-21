---
layout: post
title: GDB In Macos
date:  2020-06-21 15:12:18 +0800
categories: [tools]
---

  GDB是大家调试程序的常用工具，在Linux系统的终端中使用`gdb main`就可以愉快的进行程序调试了，但是在Macos系统中当你尝试使用gdb进行调试时，你将看到如下错误信息
```
Starting program: /x/y/foo
Unable to find Mach task port for process-id 28885: (os/kern) failure (0x5).
 (please check gdb is codesigned - see taskgated(8))
```
这个是因为Macos系统处于安全考虑，不允许GDB完全控制另一程序。网上解决此类问题的文章很多，这是简单记录一下。

### 安装
使用homebrew安装，在终端输入如下命令。安装完成后的路径为：`gdb /usr/local/bin/gdb`
```
    brew install gdb
```

### > 证书签名
方法步骤主要参考[这里](https://sourceware.org/gdb/wiki/PermissionsDarwin)，具体步骤如下
#### 在系统钥匙串中创建证书
1. Command + 空格，搜索"钥匙串访问.app"
2. 菜单栏>证书助理>创建证书
3. 名称随便填(如：gdb_cert), 身份类型为「自签名根证书」，证书类型为「代码签名」，并勾选「让我覆盖这些默认值」，继续，证书时间自己设置，我设置了十年
4. 继续，直到让我们指定用于该证书的位置，选择「系统」。然后输入密码即可创建。

查看：确保keychain是在系统钥匙串中
```
security find-certificate -c gdb-cert
```
检查是否过期
```
security find-certificate -p -c gdb-cert | openssl x509 -checkend 0
```
#### > 信任证书进行代码签名
1. 接着，我们可以在「系统」钥匙串的「我的证书」种类中找到这个证书。双击该证书打开，然后展开「信任」栏目，将「使用此证书时」选择为「始终信任」，关闭时输入密码即可保存。

查看：显示gdb-cert证书配置
```
security dump-trust-settings -d
```
#### > 签名并授权GDB
1. 新建`gdb-entitlement.xml`文件，填入一下内容

```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>com.apple.security.cs.debugger</key>
    <true/>
</dict>
</plist>
</pre>
```

2. 执行命令

```
codesign --entitlements gdb-entitlement.xml -fs gdb-cert $(which gdb)
```

### > 刷新系统的证书和代码签名数据
1. 杀死`taskgated`进程，确认是否重启
```
sudo killall taskgated
ps $(pgrep -f taskgated)
```

重新启动GDB进行程序调试，但是在执行`run`命令后出现如下问题：
```
Starting program: xx/zmain
[New Thread 0x1803 of process 1090]
[New Thread 0x1903 of process 1090]
```
然后整个gdb就卡住了，必须使用control+Z才能退出。

解决方法：在终端中执行如下命令
```shell
echo "set startup-with-shell off" >> ~/.gdbinit
```
这样就可以愉快的使用gdb玩耍了。

接下来就学习学习在emacs中如何使用GDB进行代码调试。