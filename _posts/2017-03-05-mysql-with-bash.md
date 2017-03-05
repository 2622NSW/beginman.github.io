---
layout: post
title: "Bash下操作mysql"
description: "bash with mysql"
category: "mysql"
tags: [mysql, shell]
---

在看《Python web开发实战》中看到如下的命令，觉得这个技巧很好。

```bash
$ (echo "user r;" cat database/schema.sql) | mysql --user='web' --password='web'
```

直接在命令行中把它导入mysql数据库中。这里用到的主要是**管道**这一技术。

其他可行的操作如下：

```bash
$ echo "select 1;" | mysql -u web -p'fangpeng'
mysql: [Warning] Using a password on the command line interface can be insecure.
1
1
```

这里有Warnning提示，命令行密码安全问题。

```bash
$ mysql -u web -p'fangpeng' -e "select 1;"
mysql: [Warning] Using a password on the command line interface can be insecure.
+---+
| 1 |
+---+
| 1 |
+---+
```

还可以

```bash
$ mysql -u web -p'fangpeng' << END
heredoc> use r;
heredoc> select * from users;
heredoc> END
mysql: [Warning] Using a password on the command line interface can be insecure.
id	name	email
1	zhangsan	zhangsan@qq.com
2	lisan	lisan@qq.com
3	wangsan	wangsan@qq.com
```

推荐:[linux shell 管道命令(pipe)使用及与shell重定向区别](http://www.cnblogs.com/chengmo/archive/2010/10/21/1856577.html)

