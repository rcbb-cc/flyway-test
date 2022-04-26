## 概念

### 什么是Flyway

Flyway 是一个能对数据库变更做版本控制的工具。

### 为什么要用Flyway

在多人开发的项目中，我们都习惯了使用 SVN 或者 Git 来对代码做版本控制，主要的目的就是为了解决多人开发代码冲突和版本回退的问题。

其实，数据库的变更也需要版本控制，在日常开发中，我们经常会遇到下面的问题：

1. 自己写的 SQL 忘了在所有环境执行；
2. 别人写的 SQL 我们不能确定是否都在所有环境执行过了；
3. 有人修改了已经执行过的 SQL，期望再次执行；
4. 需要新增环境做数据迁移；
5. 每次发版需要手动控制先发 DB 版本，再发布应用版本；
6. 其它场景...

有了Flyway，这些问题都能得到很好的解决。

## 最佳实践

通过 [Aliyun Java Initializr](https://start.aliyun.com/bootstrap.html) 创建项目

![依赖](https://rcbb-blog.oss-cn-guangzhou.aliyuncs.com/2022/04/20220416100653-89c795.png?x-oss-process=style/yuantu_shuiyin)

我使用的是 PostgreSQL，创建一个库。

> create database test_flyway;

在`src/main/resources`目录下面新建`db.migration`文件夹，默认情况下，该目录下的.sql 文件就算是需要被 flyway 做版本控制的数据库 SQL 语句。

但是此处的 SQL 语句命名需要遵从一定的规范，否则运行的时候 flyway 会报错。命名规则主要有两种：

* **仅需被执行一次的 SQL** 。文件命名以大写`V`字开头，后面可跟上`0-9`数字，数字之间可用`.`或`_`分隔开，然后再以**两个下划线**分隔，后面为文件名，最后以`.sql`结尾。例：`V1.0.0__init.sql`

* **可重复运行的SQL**。只要脚本内容发生了变化，项目启动时脚本就会被重新执行。文件名以大写`R`字开头，后面再以两个下划线分隔后面为文件名，最后以`.sql`结尾。


项目配置。

```
spring:
  application:
    name: flyway-test
  datasource:
    driver-class-name: org.postgresql.Driver
    url: jdbc:postgresql://192.168.1.56:5432/test_flyway?&ssl=false&stringtype=unspecified
    username: postgres
    password: MC+2021i
server:
  port: 8080
```

初次启动，报错。因为无任何内容。

![初次启动，报错](https://rcbb-blog.oss-cn-guangzhou.aliyuncs.com/2022/04/20220416102645-c2a0f3.png?x-oss-process=style/yuantu_shuiyin)

### 仅需被执行一次的

文件命名以大写`V`字开头，后面可跟上`0-9`数字，数字之间可用`.`或`_`分隔开，然后再以**两个下划线**分隔，后面为文件名，最后以`.sql`结尾。例：`V1.0.0__init.sql`

创建第一个脚本：`V1.0.0_init.sql`

```
create table tb_user
(
    id          bigserial primary key,
    username    varchar(30) not null,
    nickname    varchar(30),
    age         smallint,
    create_time bigint,
    update_time bigint
);

comment on table tb_user is '用户表';
comment on column tb_user.id is '用户ID';
comment on column tb_user.username is '用户名';
comment on column tb_user.nickname is '用户昵称';
comment on column tb_user.age is '年龄';
comment on column tb_user.create_time is '创建时间';
comment on column tb_user.update_time is '更新时间';
```
![目录结构](https://rcbb-blog.oss-cn-guangzhou.aliyuncs.com/2022/04/20220426145200-3f8d93.png?x-oss-process=style/yuantu_shuiyin)

我在 `db.migration` 下还创建了一个 1.0.0 的文件夹，这个不会影响 flyway 对 sql 的识别，可以自行取名和分类。

运行成功。

![运行成功](https://rcbb-blog.oss-cn-guangzhou.aliyuncs.com/2022/04/20220426145647-1671cc.png?x-oss-process=style/yuantu_shuiyin)

创建了`flyway_schema_history`表，并成功执行一个脚本。

![数据库内容](https://rcbb-blog.oss-cn-guangzhou.aliyuncs.com/2022/04/20220426145954-9e8553.png?x-oss-process=style/yuantu_shuiyin)

`flyway_schema_history`表中记录的是脚本信息和脚本执行状态。

![flyway_schema_history](https://rcbb-blog.oss-cn-guangzhou.aliyuncs.com/2022/04/20220426145919-9db8f9.png?x-oss-process=style/yuantu_shuiyin)

有了这条记录，下次再启动项目，`1.0.0/V1.0.0__init.sql` 这个脚本文件就不会执行了，因为系统知道这个脚本已经执行过了。  
如果你还想让 `1.0.0/V1.0.0__init.sql`脚本再执行一遍，需要手动删除`flyway_schema_history`  表中的对应记录，那么项目启动时，这个脚本就会被执行了。

### 可重复执行的脚本

只要脚本内容发生了变化，项目启动时脚本就会被重新执行。文件名以大写`R`字开头，后面再以两个下划线分隔后面为文件名，最后以`.sql`结尾。

接着在 `db/migration/1.0.0/`下创建`R__init_user.sql`

```
insert into tb_user(username, nickname, age) VALUES ('admin','admin',18);
insert into tb_user(username, nickname, age) VALUES ('sys','sys',28);
```

重新启动项目，运行成功。  
数据库`tb_user`表中新增两条数据。  
`flyway_schema_history`新增了一条记录，但 version 字段上为 null。

![flyway_schema_history](https://rcbb-blog.oss-cn-guangzhou.aliyuncs.com/2022/04/20220426151708-8b7aab.png?x-oss-process=style/yuantu_shuiyin)

重新运行程序第二遍，并未发生变化。

重新修改`R__init_user.sql` 脚本。
```
insert into tb_user(username, nickname, age) VALUES ('test','test',18);
```
重新运行程序，脚本执行成功，数据变更成功。

![tb_user](https://rcbb-blog.oss-cn-guangzhou.aliyuncs.com/2022/04/20220426152833-f7de31.png?x-oss-process=style/yuantu_shuiyin)


## 错误验证

### 刻意修改仅需被执行一次的脚本

当直接在`V1.0.0__init.sql`脚本上进行修改时。重新启动程序会进行报错。

![报错](https://rcbb-blog.oss-cn-guangzhou.aliyuncs.com/2022/04/20220426153132-e37ecd.png?x-oss-process=style/yuantu_shuiyin)

原因：会检查数据库中对应脚本的`checksum`字段，发现不对应则说明脚本进行了修改，所以阻止运行。

将修改的内容还原，则可重新运行。  
或者删除`flyway_schema_history`表中`V1.0.0__init.sql`脚本对应的记录，则可重新执行`V1.0.0__init.sql`脚本。


### 可重复执行的脚本若存在错误

刻意在`R__init_user.sql`脚本中添加一条错误的脚本，让其`id`与库中已存在的数据冲突。

顺便验证数据是否会回滚。

```
insert into tb_user(username, nickname, age) VALUES ('test1','test1',18);
insert into tb_user(username, nickname, age) VALUES ('test2','test2',28);
insert into tb_user(id, username, nickname, age) VALUES (1, 'test2','test2',28);
```

从报错信息中能很明显的看出来问题，直接给出了具体的语句和错误原因。  
然后上面的语句也未成功添加。

![报错信息](https://rcbb-blog.oss-cn-guangzhou.aliyuncs.com/2022/04/20220426154032-2e57e6.png?x-oss-process=style/yuantu_shuiyin)


## 总结

`V`开头的脚本，内容不变运行，语句并不会执行，内容修改再运行，程序启动失败！

`R`开头的脚本，内容不变运行，语句并不会执行，内容修改过再运行，**会将文件中的所有语句执行**！

每次程序启动会检验目前项目中的 sql 文件名称和文件夹名是否与数据库中存储的对应。  
如果不对应则抛出相应的错误，程序启动失败。

`R`开头的脚本，更像是**当前版本的一个临时补充的区域，为了及时修改当下紧急更新的内容**。