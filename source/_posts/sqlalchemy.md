title: sqlalchemy的简单用法
date: 2014-10-23 21:43:47
tags: Python
---

数据库中表结构是现成的情况下ORM
--------------------------------

前几天想自己在网上抓些东西然后存放在数据库中以便于后续的数据分析，只是不想自己
用DB-API写SQL语句去操作数据，所以才想用ORM的方式，可是看了半天sqlalchemy的文档
，都是在讲如何进行建库、建关系等，可是问题是我们经常要操作的数据库，库表的建立
并不是通过ORM的方式建，更希望是通过原始的SQL语句去建，对一个现成的数据库进行ORM
方式访问有没有更方便的方法呢？这种情况下有没有一种更方便的方式去访问数据库？答
案当然是肯定的。

比如我用来存储用户信息的sqlite数据库 (test.db) 结构是这样的：

``` sql
create table if not exists users (
    id integer primary key,
    name varchar(20),
    password varchar(20)
);
```

sqlalchemy 0.9.1 之后增加了一种automap_base的机制，可以使现成的数据库中的数据结
构反射到类上。用来访问上述数据库中的表只需要简单几句就可以完成映射：

``` python
# -*- coding=utf-8 -*-
import sqlalchemy
from sqlalchemy.orm import Session
from sqlalchemy.ext.automap import automap_base

if __name__ == "__main__":
    engine_str = 'sqlite:///:test.db'
    engine = sqlalchemy.create_engine(engine_str)

    session = Session(engine)

    # 下面这两句话就完成了ORM映射 Base.classes.XXXX即为映射的类
    # Base.metadata.tables['XXX']即为相应的表
    Base = automap_base()
    Base.prepare(engine, reflect = True)

    # 查询操作
    result = session.query(Base.classes.users).all()

    # 插入操作
    item = Base.classes.users(name='lxq', password='1234')
    session.add(item)
    session.commit()
    
    session.close()
```

简单定义了两个类之后，不用关心数据表的内部实现，sqlalchemy会实现自动的映射，随
后手册上的很多操作都可以正常使用了。

数据的修改
----------

ORM系统对数据库中访问到的每一行数据会映射为一个唯一的Table对象，因此，当需要对
数据修改的时候，找到这行数据然后直接改其元素然后commit即可。

``` python
ed_user = session.query(User).filter_by(name='ed').first()
ed_user.name = 'love'
session.commit()
```

批量插入
--------

利用单纯的ORM方式进行大批量的插入数据，由于是调用多条INSERT语句，因此效率十分低
下，session.add_all函数实际上也是调用了多条INSERT语句，如果想批量插入数据，可以
进行如下处理：

``` python
engine.execute(User.__table__.insert(), 
    [{'name':'hello', 'password':'1234'},
    {'name':'hello2', 'password':'1234'}])
```

MISC
-----

Lazy connecting
~~~~~~~~~~~~~~~~

create_engine 按照手册上的说法是lazy_connecting，当你在create_engine的时候并不
会真正的建立数据库连接，而当第一次进行数据库相关操作的时候才会真正建立连接。


