---
layout: post
title: "recover table structure from .frm files with MySQL Utilities"
keywords: ["mysql"]
description: "MySQL"
category: "MySQL"
tags: ["mysql","MySQL Utilities"]
---

  数据崩溃了，仅剩下.frm文件

###  安装


```
python ./setup.py build
python ./setup.py install
```

恢复表结构

```
 mysqlfrm --diagnostic /var/lib/mysql/afsbak/y004_f_import_info.frm 
```

[MySQL Utilities-Download](https://downloads.mysql.com/archives/utilities/)   
[How to Install MySQL Utilities](https://dev.mysql.com/doc/mysql-utilities/1.6/en/mysql-utils-install-source.html)
