# MongoDB 学习笔记

## Linux Mac 下安装

### 下载

https://www.mongodb.com/download-center/community

### 安装

将压缩包解压到目录 `/usr/local`

```shell
# 解压
tar -zxvf mongodb-osx-ssl-x86_64-enterprise-4.0.10.tgz

# 将解压包拷贝到指定目录
mv mongodb-linux-x86_64-3.0.6/ /usr/local/mongodb 
```

### 确保二进制文件位于`PATH`环境变量中

```shell
sudo vim /etc/profile

# 在 profile 新增下面配置
export PATH=/usr/local/mongodb/bin:$PATH
```

### 创建数据库目录

```shell
# 创建数据存储目录
mkdir -p /var/lib/mongo

# 创建日志目录
mkdir -p /var/log/mongodb
```

### 配置

```shell
# 生成配置文件
vim /usr/local/mongodb/bin/mongodb.conf
```

```shell
# where to write logging data.
systemLog:
	# 日志写到文件
  destination: file
  # 默认值：False, 日志写入模式, 
  # false: 将备份现有日志并创建新文件
  # true: 将日志追加到现有日志文件的末尾
  logAppend: true
  # 日志存储路径
  path: /var/log/mongodb/mongod.log

# Where and how to store data.
storage:
  dbPath: /var/lib/mongo
  journal:
    enabled: true
#  engine:
#  mmapv1:
#  wiredTiger:

# how the process runs
processManagement:
  fork: true  # fork and run in background
  pidFilePath: /var/run/mongodb/mongod.pid  # location of pidfile
  timeZoneInfo: /usr/share/zoneinfo

# network interfaces
net:
	# 数据库端口
  port: 27017
  # 访问数据库的 ip, 多个ip配置使用逗号分隔
  # 0.0.0.0: 任何ip都可访问
  # 127.0.0.1: 只有本地可访问
  bindIp: 0.0.0.0

# 权限配置
security:
	# 启用或禁用基于角色的访问控制（RBAC）以管理每个用户对数据库资源和操作的访问
	# 默认值是：disabled
	# 取值：
	#		enabled: 用户只能访问已被授予权限的数据库资源和操作
	# 	disabled: 用户可以访问任何数据库并执行任何操作
	authorization: enabled
```

### 启动

```shell
mongod –f /usr/local/mongodb/bin/mongodb.conf

# 如果没有创建全局路径 PATH，需要进入以下目录
cd /usr/local/mongodb/bin
sudo ./mongod
```

### 配置角色

1. 数据库用户角色：read、readWrite;

2. 数据库管理角色：dbAdmin、dbOwner、userAdmin；

3. 集群管理角色：clusterAdmin、clusterManager、clusterMonitor、hostManager；

4. 备份恢复角色：backup、restore；

5. 所有数据库角色：readAnyDatabase、readWriteAnyDatabase、userAdminAnyDatabase、dbAdminAnyDatabase

6. 超级用户角色：root 
  

//这里还有几个角色间接或直接提供了系统超级用户的访问（dbOwner 、userAdmin、userAdminAnyDatabase）

其中MongoDB默认是没有开启用户认证的，也就是说游客也拥有超级管理员的权限。

userAdminAnyDatabase：有分配角色和用户的权限，但没有查写的权限

进入`mongo shell`

```shell
$ mongo
```

创建用户，注意，账号是跟随数据库的。

```shell
$ use admin
$ db.createUser({
	user: 'userName',	# 用户名
	pwd: '123456',		# 密码
	roles:[{role:'dbAdmin',db:'admin'}]	# role: 角色，db: 数据库
})
```

此时使用 `MongoDB` 前需要进行认证

```shell
$ db.auth("userName", "123456")
```

## 用户管理

切换到 products 数据库，并给 accountUser 用户 readWrite 和 dbAdmin 权限

```shell
$ use products
$ db.createUser(
   {
     user: "accountUser",
     pwd: "password",
     roles: [ "readWrite", "dbAdmin" ]
   }
)

# 修改用户权限
db.revokeRolesFromUser(
	"testUser",
	[{role: "readWrite", db: "test"}]
)

# 显示当前数据库上的所有用户
show users
```

## 基本使用

### 查询正在使用的数据库

```shell
$ db
```

### 切换/创建数据库

```shell
$ use 数据库名称
```

如果该数据库不存在，则该命令会创建一个新的

### 列出当前用户所有可用数据库

```shell
$ show dbs
```

### 集合操作

```shell
# 查看所有集合
$ db.getCollectionNames()

# 集合操作
$ db.集合名称.<insert/find>()

# 如果这里的集合名称包含空格、短横线等特殊符号或者与系统命令冲突，则使用下面的形式执行集合命令
$ db.getCollection("3 test").find()
$ db.getCollection("3-test").find()
$ db.getCollection("stats").find()
```

### 格式化打印结果

可以将其添加`.pretty()`到操作中，如下所示：

```shell
$ db.myCollection.find().pretty()
```

您可以在`mongo shell`中使用以下显式打印方法 ：

- `print()` 打印没有格式化
- `print(tojson(<obj>))`使用`JSON`格式打印并等效于`printjson()`
- `printjson()`使用`JSON`格式打印并等效于`print(tojson(<obj>))`

### 退出 mongo shell

```
要退出shell，请键入quit()或使用<Ctrl-C>快捷方式。
```

## BSON 数据类型

BSON是一种二进制序列化格式，用于存储文档并在MongoDB中进行远程过程调用

| type                    | Number | String Alias          | Notes               |
| :---------------------- | :----- | :-------------------- | :------------------ |
| Double                  | 1      | “double”              |                     |
| String                  | 2      | “string”              |                     |
| Object                  | 3      | “object”              |                     |
| Array                   | 4      | “array”               |                     |
| Binary data             | 5      | “binData”             |                     |
| Undefined               | 6      | “undefined”           | Deprecated.         |
| ObjectId                | 7      | “objectId”            |                     |
| Boolean                 | 8      | “bool”                |                     |
| Date                    | 9      | “date”                |                     |
| Null                    | 10     | “null”                |                     |
| Regular Expression      | 11     | “regex”               |                     |
| DBPointer               | 12     | “dbPointer”           | Deprecated.         |
| JavaScript              | 13     | “javascript”          |                     |
| Symbol                  | 14     | “symbol”              | Deprecated.         |
| JavaScript (with scope) | 15     | “javascriptWithScope” |                     |
| 32-bit integer          | 16     | “int”                 |                     |
| Timestamp               | 17     | “timestamp”           |                     |
| 64-bit integer          | 18     | “long”                |                     |
| Decimal128              | 19     | “decimal”             | New in version 3.4. |
| Min key                 | -1     | “minKey”              |                     |
| Max key                 | 127    | “maxKey”              |                     |

可以将这些类型与 `$type` 运算符一起使用，以按 `BSON` 类型查询文档。`$type` 聚合运算符使用列出的 `BSON` 类型字符串返回运算符表达式的类型

### Date

`Date` 对象存储为带符号的64位整数，表示自Unix纪元（1970年1月1日）以来的毫秒数

`mongo shell` 提供了各种方法来返回日期对象或者日期字符串：

```shell
# 以字符串形式返回当前日期
Date()

# 构造函数，它使用 ISODate() 包装器返回Date对象
new Date()

# 构造函数，它使用 ISODate() 包装器返回Date对象
ISODate()
```

```shell
# 创建一个当前日期字符串
var myDateString = Date();

# 下面操作返回 myDateString 的类型：string
typeof myDateString
```

### ObjectId

要生成新的 `ObjectId` 请在 `mongo shell` 中使用以下操作：

```
new ObjectId
```

`ObjectId` 有以下属性/方法:

| Attribute/Method                                             | Description                                               |
| :----------------------------------------------------------- | :-------------------------------------------------------- |
| `str`                                                        | 返回对象的十六进制字符串表示形式                          |
| [`ObjectId.getTimestamp()`](https://docs.mongodb.com/manual/reference/method/ObjectId.getTimestamp/#ObjectId.getTimestamp) | 以 `Date` 作为对象类型的时间戳部分                        |
| [`ObjectId.toString()`](https://docs.mongodb.com/manual/reference/method/ObjectId.toString/#ObjectId.toString) | 以字符串"`ObjectId(…)`”的形式返回 `JavaScript` 表示形式   |
| [`ObjectId.valueOf()`](https://docs.mongodb.com/manual/reference/method/ObjectId.valueOf/#ObjectId.valueOf) | 返回对象的十六进制字符串表示形式，这里返回就是 `str` 属性 |

### NumberLong

默认情况下，`mongo shell` 将所有数字视为64位浮点数

`mongo shell` 提供 `NumberLong()` 包装器来处理64位整数

```shell
# NumberLong() 接受一个字符串类型的整数值
NumberLong("2090845886852")

# 将 NumberLong 类型的值插入数据库
db.collection.insertOne( { _id: 10, calc: NumberLong("2090845886852") } )
```

### NumberInt

`NumberInt(string)` 构造函数来显式指定32位整数

### NumberDecimal

`NumberDecimal(string)` 构造函数，显式指定128位基于十进制的浮点数，能够以精确的精度模拟十进制舍入。此功能适用于处理货币数据的应用程序

```shell
NumberDecimal("1000.55")
```

### Timestamps

`MongoDB` 中的时间戳是按照以下方式生成 64 位整数：

```
前 32 位是自 Unix 纪元以来的秒数
后 32 位是当前给定的秒内递增数值
```

因此，在单个独立的 `mongo` 数据库上，`timestamps` 是唯一的

```shell
# 生成的时间戳: Timestamp(1412180887, 1)
# 第一个32位值是秒数，第二个32位值是递增整数
var a = new Timestamp();
```

`Timestamp` 一般用与 `MongoDB` 内部，开发人员使用 `Date` 类型

## CRUD ( 增/删/改/查)

### 新增

创建或插入操作将新文档添加到集合中，如果集合当前不存在，则插入操作将创建集合。

在`MongoDB`中，插入操作以单个集合为目标。

`MongoDB`中的所有写入操作都是**`单个文档级别的原子`**操作。

`MongoDB`中每个文档都必须有一个唯一的字段`_id`作为主键，如果文档未指定`_id`字段，`MongoDB`会将`_id`自动将字段添加到新文档中，其的值类型是: ObjectId

`ObjectId`值是12个字节：

- 前4个字节的值：表示自Unix纪元以来的秒数

- 中间5个字节的随机值

- 最后3个字节的计数器，以随机值开始

在默认情况下，多个文档顺序插入文档时，`MongoDB`是一条一条写入集合的，如果遇到其中一个文档插入错误，则插入操作终止，后面的语句也不会执行

#### `db.collection.insertOne()`

将单个文档插入到集合中

```
db.students.insertOne(
   { name: "canvas", age: 100 }
)
```

#### `db.collection.insertMany()`

将*多个* 文档插入到集合中。将一组文档传递给该方法。

```
db.students.insertMany([
   { name: "canvas", age: 100 },
   { name: "ximing", age: 200 }
])
```
#### `db.collection.insert()`

将单个文档或多个文档插入集合中

```
db.students.insert(
   { name: "canvas", age: 100 }
)
```

```
db.students.insert([
   { name: "canvas", age: 100 },
   { name: "ximing", age: 200 }
])
```

### 删除

#### `db.collection.deleteMany()`

删除操作不会删除索引，即使从集合中删除所有文档也是如此

删除所有文档

```
db.students.deleteMany({})
```

删除与条件匹配的所有文档

要删除与删除条件匹配的所有文档，请将筛选参数传递给 `deleteMany()`方法。

```
db.students.deleteMany(
	{age: 100}
)
```

#### `db.collection.deleteOne()`

删除与条件匹配的一个文档

删除一个文档时，如果筛选条件匹配了多个结果，那么则删除`_id`排序第一个文档

```
db.students.deleteOne(
	{name: "xiaoming"}
)
```

#### `db.collection.remove()` 

删除单个文档或与指定过滤器匹配的所有文档

```
db.collection.remove(
   <query>,
   {
     justOne: <boolean>,
     writeConcern: <document>,
     collation: <document>
   }
)
```

`justOne` 默认值 false，删除多有匹配的文档，true 仅删除一个文档

```shell
# 删除所有文档
db.students.remove({})

# 删除与删除条件匹配的所有文档
db.students.remove({age: 100})

# 删除与删除条件匹配的一个文档
db.students.remove({age: 100}, {justOne: true})
```

#### `db.collection.drop()`

从数据库中删除集合或视图

该方法还会删除与已删除集合关联的所有索引

```
db.collection.drop( { writeConcern: <document> } )
```

此方法在受影响的数据库上获取写锁定，并将阻止其他操作，直到完成为止。

`db.collection.drop（）` 方法和 `drop命令` 为已删除集合上打开的任何更改流创建invalidate事件

从MongoDB 4.0.2开始，删除集合将删除其关联的区域/标记范围

```
db.students.drop()
```

### 修改

选项：upsert: true

默认值为`false`，未找到匹配项时不插入新文档。

如果`updateOne()`， `updateMany()`或 `replaceOne()` 没有与指定过滤器匹配的文档，则该操作将创建一个新文档并将其插入。如果存在匹配的文档，则操作修改或替换匹配的文档。

**`_id` 字段的值是不可修改的**

如果多个客户端同时发出带有 `upsert: true` 的更新相同内容的操作，则可能会导致多条语句被插入数据库，为防止这种情况，则需要给 `<filter> 加上唯一索引`

使用**点表示法**更新嵌套字段或者数组中值

#### `db.collection.updateOne(<filter>, <update>, <options>)`

更新单个文档

```shell
db.students.updateOne(
   { name: "xiaoming" },	# 更新条件
   {
     $set: { name: "cm", age: 101 },	# 修改 name 和 age 字段的值
     $set: {"obj.id": 111},	# 修改 obj.id 字段的值
     $currentDate: { updateTime: true} 	# 将 updateTime 字段的值更新为当前时间
   }
)
```

#### db.collection.updateMany(\<filter>, \<update>, \<options>)

```shell
db.students.updateMany(
   { age: 10 },	# 将满足条件的所有文档更新
   {
     $set: { name: "cm", age: 101 },	# 修改 name 和 age 字段的值
     $set: {"obj.id": 111},	# 修改 obj.id 字段的值
     $currentDate: { updateTime: true} 	# 将 updateTime 字段的值更新为当前时间
   }
)
```

#### db.collection.replaceOne(\<filter>, \<update>, \<options>)

替换一个文档中除`_id`之外的所有内容

替换文档可以具有与原始文档不同的字段。在替换文档中，您可以省略该`_id`字段，因为该`_id`字段是不可变的; 但是，如果包含该 `_id`字段，则它必须与当前值具有相同的值

```shell
db.students.replaceOne(
	{name: "xiaoming"},
	{name: "xiaohua", age: 10}
)
```

#### `db.collection.update()`

```shell
db.collection.update(
   <query>,
   <update>,
   {
     upsert: <boolean>,
     multi: <boolean>,
     writeConcern: <document>,
     collation: <document>,
     arrayFilters: [ <filterdocument1>, ... ]
   }
)
```

更新或替换与指定过滤器匹配的单个文档，或更新与指定过滤器匹配的所有文档。

##### 参数说明

`upsert` 

​	可选的。默认值为false

​	true：没有文档与查询条件匹配时创建新文档。false：如果未找到匹配项，则不会插入新文档

`multi` 

​	可选的。默认值为false

​	true: 更新符合查询条件的所有文档。false: 只更新一个文档

`writeConcern`

​	可选的

​	`writeConcern` 描述了 `MongoDB` 请求的确认级别，用于对独立 `MongoDB` 或副本集或分片集群的写入操作。在分片群集中，mongos实例会将 `writeConcern` 传递给分片

​	**注意**: 对于多文档事务，您可以在事务级别设置写入关注点，而不是在单个操作级别。不要在事务中明确设置_单个写操作_的写入问题

```
writeConcern: { w: <value>, j: <boolean>, wtimeout: <number> }
```

* w: 选项请求确认写操作已传播到指定数量的mongod实例或具有指定标记的mongod实例

* j: 选项请求确认写操作已写入磁盘日志

* timeout: 选项指定一个时间限制，以防止写操作无限期地阻塞

`collation`

​	指定要用于操作的排序规则。排序规则允许用户为字符串比较指定特定于语言的规则

```
collation: {
   locale: <string>,
   caseLevel: <boolean>,
   caseFirst: <string>,
   strength: <int>,
   numericOrdering: <boolean>,
   alternate: <string>,
   maxVariable: <string>,
   backwards: <boolean>
}
```

`arrayFilters`

​	可选的。

​	一组过滤器文档，用于确定要为数组字段上的更新操作修改哪些数组元素。在更新文档中，使用`$ [<identifier>]`过滤的位置运算符来定义标识符，然后在数组过滤器文档中引用该标识符。如果标识符未包含在更新文档中，则不能有标识符的数组过滤器文档。

```shell
# 筛选出集合中 grades 数组大于等于100 的文档
# 将大于等于 100 的 grades 数组元素修改为 100
db.students.update(
   { grades: { $gte: 100 } },
   { $set: { "grades.$[element]" : 100 } },
   {
     multi: true,
     arrayFilters: [ { "element": { $gte: 100 } } ]
   }
)
```

默认情况下，`db.collection.update()` 方法更新单个文档。要更新多个文档，请使用`multi`选项。

```shell
# 替换单个文档，如果不存在则不插入
db.students.update(
	{name: "xiaoming"},
	{name: "xiaohua", age: 10}
)

# 修改单个文档，如果不存在则不插入
db.students.update(
	{name: "xiaoming"},
	{
		$set: {name: "xiaohua", age: 10}
  }
)

# 修改匹配的多个文档，如果不存在则不插入
db.students.update(
	{name: "xiaoming"},
	{
		$set: {name: "xiaohua", age: 10}
  },
  {
  	multi: true
  }
)
```

```shell
# 更新单个文档，如果不存在则插入
db.students.update(
	{name: "xiaoming"},
	{name: "xiaohua", age: 10},
	{upsert: true}
)
```

#### 更新操作符

更新操作符可用于更新操作

更新操作符语法

```
{
   <operator1>: { <field1>: <value1>, ... },
   <operator2>: { <field2>: <value2>, ... },
   ...
}
```

##### $set

用指定的值替换字段的值

如果修改的字段不存在，则会创建新的字段

```
{ $set: { <field1>: <value1>, ... } }
```

##### $unset

删除指定字段，如果字段不存在则不执行任何操作

```
{ $unset: { <field1>: "", ... } }
```

```shell
# 删除 age 和 love 字段
db.students.update(
	{name: "xiaowang"},
	{$unset: {age: "", love: ""}}
)
```

如果删除数组中的一个元素，则是将数组中指定的元素设置为 `null` ，而不是从数组中删除匹配元素。以保持数组大小和元素位置保持一致。

```shell
# 如果 love = ["a", "b", "c"]
# 修改后则是: love = ["a", null, "c"]
db.students.update(
	{name: "xiaoming"},
	{$unset: {"love.1": ""}}
)
```

##### $rename

`$rename` 更新字段的名称

```
{$rename: { <field1>: <newName1>, <field2>: <newName2>, ... } }
```

`$rename` 运算符在逻辑上执行旧名称和新名称的 `$unset`，然后使用新名称执行 `$set` 操作。因此，操作可能不保留文档中字段的顺序，即重命名的字段可以在文档内移动

如果文档已经包含 `<newName>` 的字段，则 `$rename` 运算符将删除该字段，并将指定的 `<field>` 重命名为`<newName>`

如果需要重命名的字段不存在，则不进行任何操作

对于嵌套文档中的字段，`$rename` 运算符可以重命名这些字段，以及将字段移入和移出嵌入文档

如果这些字段在数组元素中，则 `$rename` 不起作用

```shell
# 将 nmae 字段的名称修改为 name
db.students.updateMany( {}, { $rename: { "nmae": "name" } } )
```

```shell
# 如果存在下面的文档
{
    "_id": ObjectId("5d33b642ec904777066a9290"),
    "name": {first: "xiao", last: "wang"},
    "age": 444
}

# 修改 name 字段中的 first 字段的名称为 fname
db.students.updateMany({}, {$rename: {"name.first": "name.fname"}})

# 执行之后文档变成
{
    "_id": ObjectId("5d33b642ec904777066a9290"),
    "name": {fname: "xiao", last: "wang"},
    "age": 444
}
```

```shell
# 如果存在下面的文档
{
    "_id": ObjectId("5d33b642ec904777066a9290"),
    "name": {first: "xiao", last: "wang"},
    "age": 444,
    "balance": 200
}

# 将 balance 字段移动到 name 字段中
db.students.updateMany({}, {$rename: {"balance": "name.balance"}})

# 执行结果
{
    "_id": ObjectId("5d33b642ec904777066a9290"),
    "name": {first: "xiao", last: "wang", "balance": 200},
    "age": 444
}
```

##### $inc 和 \$mul

`$inc`  运算符先现有的值加上指定的值，如果指定的值时负数则就是减法运算

```
注意
1. 如果该字段不存在，$inc 将创建该字段并将该字段设置为指定值
2. 在具有空值的字段上使用 $inc 运算符将生成错误
3. $inc 在单个文档中是原子操作
```

`$mul` 运算符将现有的值乘以指定的数字

```
注意
1. 如果该字段不存在，$mul 将创建该字段并将其值设置为与乘数相同的数字类型的零
2. $inc 在单个文档中是原子操作
```

```
{ $inc: { <field1>: <amount1>, <field2>: <amount2>, ... } }
{ $mul: { <field1>: <number1>, ... } }
```

```shell
db.students.update({_id: 1}, {$inc: {age: 1}})
db.students.update({_id: 1}, {$mul: {price: 1.2}})
```

##### \$min 和 \$max

`$min` 运算符将比较比较指定值与字段当前的值，如果指定的**小于**字段当前的值，则将字段的值修改为指定的值

`$max` 运算符将比较比较指定值与字段当前的值，如果指定的**大于**字段当前的值，则将字段的值修改为指定的值

```
{ $min: { <field1>: <value1>, ... } }
{ $max: { <field1>: <value1>, ... } }
```

```shell
# 假设有下面的文档
{ _id: 1, highScore: 800, lowScore: 200 }

# 下面操作比较指定的值 100 和 字段值 200, 将最小值 100 更新到 lowScorce 字段
# 执行结果 { _id: 1, highScore: 800, lowScore: 100 }
db.students.update({_id: 1}, {$min: {lowScorce: 100}})
```

如果指定的字段不存在，则插入该字段并将值设置为指定的值

对于不同类型之间的比较，`$min` 和 `$max` 使用 `BSON` 比较顺序

```shell
# 下面的 BSON 是从低到高的顺序
MinKey (internal type)
Null
Numbers (ints, longs, doubles, decimals)
Symbol, String
Object
Array
BinData
ObjectId
Boolean
Date
Timestamp
Regular Expression
MaxKey (internal type)
```

#### 数组更新操作符

##### $addToSet

如果数组中不存在指定的值，则将该值添加到数组中

```
{ $addToSet: { <field1>: <value1>, ... } }
```

`$addToSet` 仅确保没有重复项添加到集合中，并且不会影响现有的重复元素。`$addToSet` 不保证修改集中元素的特定排序

如果在不是数组的字段上使用 `$addToSet`，则操作将失败

如果指定的值是数组，则 `$addToSet` 将整个数组作为单个元素追加到数组字段中

```shell
# 假设文档 { _id: 1, letters: ["a", "b"] }
db.students.update({_id: 1}, {$addToSet: {letters: ["c", "d"]}})
# 上面语句执行后，会将 ["c", "d"] 作为一个单独的追加到数组的末尾
# { _id: 1, letters: ["a", "b", ["c", "d"]] }
```

如果要将指定数组中的每个元素添加到数组中，则需要使用 `$each` 修饰符和 `$addToSet` 

```shell
# 假设文档 { _id: 1, letters: ["a", "b"] }
db.students.update({_id: 1}, {$addToSet: {letters: {$each: ["c", "d"]}})
# 上面语句执行后，会将 ["c", "d"] 中的每个当作一个元素分别追加到数组字段的末尾
# { _id: 1, letters: ["a", "b", "c", "d"] }
```

##### $pop

`$pop`运算符删除数组的第一个或最后一个元素

传递`$pop` 值为 `-1` 删除数组的第一个元素，传递`1`删除数组中的最后一个元素

```
{ $pop: { <field>: <-1 | 1>, ... } }
```

如果 `<field>` 不是数组，`$pop` 操作将失败。

如果 `$pop` 运算符删除 `<field>` 中的最后一项，则 `<field>` 将保留一个空数组

```shell
# 删除第一个元素
db.students.update( { _id: 1 }, { $pop: { love: -1 } } )
```

##### $pull

`$pull` 运算符从现有数组中删除与指定条件匹配的所有元素

```
{ $pull: { <field1>: <value|condition>, <field2>: <value|condition>, ... } }
```

如果要删除 `<value>` 是一个数组，`$pull` 将仅删除数组中与指定的 `<value>` 完全匹配的元素，包括顺序

如果要删除 `<value>` 是文档，`$pull` 将仅删除数组中具有完全相同的字段和值的元素。字段的顺序可以不同

```
db.students.update({}, {$pull: {love: {$in: ["yinyue", "zuqiu"]}}})
```

##### $pullAll

`$pullAll` 运算符从现有数组中删除指定值的所有元素

```
{ $pullAll: { <field1>: [ <value1>, <value2> ... ], ... } }
```

如果要删除的 `<value>` 是文档或数组，`$pullAll` 将仅删除数组中与指定的 `<value>` 完全匹配的元素，包括顺序

```
db.survey.update( { _id: 1 }, { $pullAll: { scores: [ 0, 5 ] } } )
```

##### $push

`$push`运算符将指定值追加到数组

```
{ $push: { <field1>: <value1>, ... } }
```

如果要更新的文档中没有该字段，`$push` 会添加数组字段，并将值作为其元素

如果值是数组，则 `$push` 将整个数组作为单个元素追加。要分别添加值的每个元素，请使用 `$each` 修饰符和 `$push`

```
db.students.update(
   { _id: 1 },
   { $push: { scores: 89 } }
)
```

##### `$` 

充当占位符以更新与查询条件匹配的第一个元素

位置 `$` 运算符标识要更新的数组中的元素，而不显式指定元素在数组中的位置

```
{ "<array>.$" : value }
```

不要将位置运算符 `$` 与 `upsert` 操作一起使用，因为 `insert` 将在插入的文档中使用$作为字段名称

与 `$unset`运算符一起使用时，位置 `$` 运算符不会从数组中删除匹配元素，而是将其设置为 `null`

##### `$[]`

充当占位符以更新数组中与查询条件匹配的文档中的所有元素

```
{ <update operator>: { "<array>.$[]" : value } 
```

### 查询

查询所有文档

```shell
db.students.find({})
```

#### 相等条件查询

```shell
# 查询 name = "xiaoming" 的文档
# 相当于sql: select * from students where name = "xiaoming"
db.students.find(
	{name: "xiaoming"}
)
```

#### 查询运算符

`find()`可以使用`查询运算符`，指定以下形式的条件：

```
{  < field1 >： {  < operator1 >： < value1 >  }， ...  }
```

`$in`

从集合查询name="xiaoming" 或者 name="xiaoli"记录

```shell
# 相当于SQL: select * from students where name in ('xiaoming', 'xiaoli')
db.students.find({name: {$in: ["xiaoming", "xiaoli"]}})

# 查询 age < 123 同时 name = "xiaoming" 的所有记录
db.students.find({name: "xiaoming", age: {$lt: 123}})
```

**注意：虽然此处使用`$or`运算符也可以，但在对`同一字段`执行`相等性`检查时，请使用`$in`运算符而不是`$or`运算符**

`$or`

查询选择集合中至少与一个条件匹配的文档

```shell
# 查询 age = 101 或者 name = "xiaoming" 的所有记录
db.students.find({$or: [{age: 101}, {name: "xiaoming"}]})
```

#### 嵌套查询

这里的嵌套指下面记录中的`point`字段类型的

```json
{
    "_id": ObjectId("5d3470a384ceb411ed074457"),
    "name": "em_test_rev",
    "age": 21,
    "point": {
      	"x": 3,
        "y": 4
    }
}
```

##### 相等查询

```shell
# 查询 point.x = 3 and point.y = 4 的记录
# 整个嵌套文档的等式匹配需要 指定的文档完全匹配<value>，包括字段顺序, 例如：
# db.students.find({poin: {y: 4， x: 3}}) 
# 上面这条查询就不会匹配到任何记录，因为顺序和插入时不同
db.students.find({poin: {x: 3, y: 4}})
```

##### 查询嵌套字段

在嵌套文档中的字段上指定查询条件，请使用`点表示法`（"field.nestedField"）。

```
注意: 使用点表示法查询时，字段和嵌套字段必须在引号内
```

##### 在嵌套字段上指定等式匹配

```shell
# 查询嵌套在 point 字段中的 x 的值等于 3 的所有文档
db.students.find({"point.x": 3})
```

#### 数组查询

##### 匹配数组

要在数组上指定相等条件，要匹配确切的数组，包括元素的顺序

例如有以下的文档

```shell
{
	_id: "ObjectId(fadfadfadfd234)",
	name: "ali",
	age: 12,
	love: ["足球", "乒乓球", "篮球"]
}

{
	_id: "ObjectId(fadfadfadfd234)",
	name: "xiaowang",
	age: 12,
	love: ["乒乓球", "足球", "篮球"]
}
```

如果要查询出 love 的值是 `["足球", "乒乓球", "篮球"]` 的文档

```shell
db.students.find({love: ["足球", "乒乓球", "篮球"]})
```

则结果只会匹配到第一条数据，因为第二条的顺序不对

如果只看值不考虑顺序的话，就要使用操作符 `$all`

例如，查询出 `love`  的值包含 `"足球", "篮球"` 的文档

```shell
db.students.find({love: {$all: ["足球", "篮球"]}})

# 查询出数组中只要包含 "篮球" 的所有文档
db.students.find({love: "篮球"})
```

##### 通过数组索引位置查询元素

使用点表示法，可以为数组的特定索引或位置处的元素指定查询条件。该数组使用从零开始的索引。

```shell
# 查询数组中第二个元素的值时 "a" 的文档
db.sutdents.find({"love.1", "a"})
```

##### 按照数组长度查询

```shell
# 查询出数组长度等于3个所有文档
db.students.find({love: {$size: 3}})
```

##### $slice

`$slice` 运算符控制查询返回的数组的条数

```shell
# 返回存储在数组字段中 count 值指定的数组元素数量
# 如果count的值大于数组长度，则返回该数组的所有元素
db.collection.find( { field: value }, { array: {$slice: count } } );

# count 也可接受负值， 负值表示返回数组后面的元素
# 下面查询返回数组字段中的最后两个元素
db.students.find({}, love: {$slice: -2})
```

`$slice` 运算符也可接受数组参数 `[skip，limit]` 的形式，其中第一个值表示要跳过的数组中的项数，第二个值表示要返回的项数

```shell
# 跳过2个数组元素，返回紧接着的5个元素
db.students.find({}, love: {$slice: [2, 5]})

# 从数组的末尾跳过2个元素，返回从后数紧接着的5个元素
db.students.find({}, love: {$slice: [-2, 5]})
```

#### 投影 — 限制查询返回的字段

如果未指定**投影**文档，则 [`db.collection.find()`](https://docs.mongodb.com/manual/reference/method/db.collection.find/#db.collection.find)方法将返回匹配文档中的所有字段

```
db.collection.find(查询条件, 投影)
```

1. 设置过滤字段`<field>` 为 `1`，可返回文档的指定字段，`_id`默认都会返回

   将过滤字段`<field>` 为 `0` 则返回该字段

```shell
# 相当于SQL：select _id, name, age from students where age = 10
db.students.find({age: 10}, {name: 1, age: 1})

# 相当于SQL：select name, age from students where age = 10
db.students.find({age: 10}, {name: 1, age: 1, _id: 0})
```

返回排除的字段外的所有字段

```shell
# 返回除 name 和 age 字段外的所有字段
db.students.find({age: 10}, {name: 0, age: 0})
```

**注意：除`_id`字段外，无法在投影文档中同时包含和排除语句。**

2. 使用**点操作**限制嵌套字段返回的字段

   ```shell
   # 返回字段: name, point.x
   db.students.find({}, {name: 1, "point.x": 1})
   ```

#### 查询值为 null 或者 没有字段的文档

查询某字段的值为 null 或者 没有该字段的文档

```shell
# 查询出 name = null 或者 没有 name 字段的所有文档
db.students.find({name: null})
```

#### 游标 Cursor

`db.collection.find()`返回的其实是一个游标

如果未将返回的游标分配给变量，则游标会自动迭代最多20次，即打印结果中的前20个文档

如果将返回的游标分配给了变量，那么游标就不会自动迭代了

##### 手动迭代游标

在shell中调用游标变量来迭代最多20次 并打印匹配的文档

```shell
var myCursor = db.students.find({})
myCursor
```

使用游标的`hasNext()`和 `next()` 循环

```shell
var myCursor = db.students.find({})
while (myCursor.hasNext()) {
	printjson(myCursor.next())
}
```

使用游标的 `forEach()` 方法循环

`forEach()` 方法接受一个方法作为参数

```shell
var myCursor = db.students.find({})
myCursor.forEach(printjson)
```

##### 将游标中的所有文档转换为数组

`toArray()` 方法迭代游标并将文档以数组的形式返回

```shell
var myCursor = db.students.find({})
var docArray = myCursor.toArray()
printjson(docArray[0])
```

##### 关闭非活动游标

默认情况下，服务器将在10分钟关闭不活动的游标，或者客户端已用尽光标

可以使用 `cursor.noCursorTimeout()` 方法阻止服务器关闭不活动游标

```shell
db.sutdents.find({}).noCursorTimeout()
```

设置 `noCursorTimeout()` 选项后，必须手动调用方法 `cursor.close()` 关闭光标，或者通过耗尽游标的结果

#### 排序

`cursor.sort({})` 在从数据库查询之前对文档进行排序

语法: `{ feild: value }`

`value` 取值 `1` 升序，`-1` 降序

```shell
# 先按照 name 升序，name 的值相同的时候再按照 age 降序
db.students.find({}).sort({name: 1, age: -1})
```

#### 查询文档条数

`db.document.count()` 使用元数据返回数量，如果数据库分片，那么返回的文档数量就不正确

```shell
db.document.countDocuments() 执行文档聚合返回精确的数量

语法: db.documents.countDocuments(<query>, <option>)

# 查询集合中文档的总数
db.students.countDocuments({})

# 查询 age > 10 的前10条数据的总数
db.students.countDocuments(
	{
		age: {$gt: 10}
	},
	{
		limit: 10
	}
)
```

#### 分页查询

在**集合的数据量比较少**的时候可使用 `cursor.sort(<1/-1>)` `cursor.skip(<number>)` `cursor.limit(<number>)` 这3个方法进行分页查询，因为`cursor.skip()` 是一条一条数着跳过的

```shell
# pageSize: 10, pageNum: 2
db.students.find({}).sort({name: 1}).skip(10).limit(10)
```

推荐使用**范围查询**进行进行分页，范围查询不需要扫描整个文档

1. 选择一个字段，例如_id，它通常会随着时间的推移以一致的方向变化，并具有唯一的索引以防止重复值

2. 使用 `$gt/$lt` 和 `cursor.sort()` 运算符查询其字段大于起始值的文档

3. 存储上次查询的最后一个字段值

```js
// 按照 _id 递增分页查询
function printStudents(startValue, nPerPage) {
  let endValue = null;
  db.students.find( { _id: { $gt: startValue } } )
             .sort( { _id: 1 } )
             .limit( nPerPage )
             .forEach( student => {
               print( student.name );
               endValue = student._id;
             } );

  return endValue;
}

// 使用下面调用方法，分页查出所有结果
// 如果是降序此处需要使用 MaxKey
let currentKey = MinKey;
while (currentKey !== null) {
  currentKey = printStudents(currentKey, 10);
}
```

### 批量写入操作

`db.collection.bulkWrite()` 方法提供了执行批量插入、更新和删除操作的功能

语法: 

```shell
db.collection.bulkWrite(
   [ <operation 1>, <operation 2>, ... ],
   {
      writeConcern : <document>,
      ordered : <boolean>
   }
)
```

#### 有序/无序操作

批量写入操作可以是有序的也可以是无序的

通过**有序**的操作列表，MongoDB以串行方式执行操作。如果在处理其中一个写操作期间**发生错误**，MongoDB将**返回**而不处理列表中的剩余写操作

```shell
# 有序写入
db.collection.bulkWrite(
   [
      { insertOne : <document> },
      { updateOne : <document> },
      { updateMany : <document> },
      { replaceOne : <document> },
      { deleteOne : <document> },
      { deleteMany : <document> }
   ]
)
```

使用**无序**的操作列表，MongoDB可以并行执行操作。如果在处理其中一个写操作期间**发生错误**，MongoDB将**继续处理**列表中的剩余写操作

```shell
# 无序写入
db.collection.bulkWrite(
   [
      { insertOne : <document> },
      { updateOne : <document> },
      { updateMany : <document> },
      { replaceOne : <document> },
      { deleteOne : <document> },
      { deleteMany : <document> }
   ],
   { ordered : false }
)
```

在分片集合上执行有序操作列表通常比执行无序列表慢，因为对于有序列表，每个操作必须等待上一个操作完成

默认情况下，`bulkWrite()` 执行有序操作。要指定无序写入操作，请在选项文档中设置 `ordered: false`

#### bulkWrite() 方法

`bulkWrite()` 支持以下写操作：

```shell
insertOne
updateOne
updateMany
replaceOne
deleteOne
deleteMany
```

## Aggregation 聚合

聚合操作处理数据记录并返回计算结果。聚合操作将来自多个文档的值组合在一起，并且可以对分组数据执行各种操作以返回单个结果。MongoDB提供了三种执行聚合的方式：`聚合管道`、`map-reduce函数` 和 `单用途聚合方法`

### 聚合管道

`MongoDB` 的聚合框架以数据处理流水线的概念为蓝本。文档进入多阶段管道，将文档转换为聚合结果

聚合管道的 `stage` 是按照顺序一步一步进行的

```
db.collection.aggregate( [ { <stage> }, ... ], <options>)
```

`<options>` 常用参数如下：

| Field          | Type       | Description                                                  |
| :------------- | :--------- | :----------------------------------------------------------- |
| `explain`      | boolean    | 可选，指定返回管道处理过程中的信息                           |
| `allowDiskUse` | boolean    | 可选，true 会将管道处理过程中的临时数据写入临时文件，主要用于管道数据过大的情况 |
| `maxTimeMS`    | 无符号整型 | 可选，指定游标操作处理的时间限制（单位：毫秒），设置为 0 或者不设置则操作不超时 |

```shell
db.orders.aggregate([
   { $match: { status: "A" } },
   { $group: { _id: "$cust_id", total: { $sum: "$amount" } } }
])

# 上面的聚合操作分为2个阶段
# 第一阶段：$match 阶段按状态字段过滤文档，并将状态等于“A”的文档传递给下一阶段
# 第二阶段：$group 阶段按 cust_id 字段对文档进行分组，以计算每个唯一cust_id的金额总和
```

最基本的管道阶段提供过滤器，其操作类似于查询和文档转换

其他管道操作提供了按特定字段或字段对文档进行分组和排序的工具，以及用于聚合数组内容（包括文档数组）的工具。此外，管道阶段可以使用运算符执行任务，例如计算平均值或连接字符串

聚合管道可以在分片集合上运行

#### 聚合管道使用要点

* 当[`$match`](https://docs.mongodb.com/manual/reference/operator/aggregation/match/#pipe._S_match)和[`$sort`](https://docs.mongodb.com/manual/reference/operator/aggregation/sort/#pipe._S_sort) 出现在管道操作符时，尽量在前面的阶段（最好从第一个阶段开始）使用，以便更好的使用索引，避免全集合扫描

* 如果聚合管道中要使用到 `$geoNear` 操作符，那么 [`$geoNear`](https://docs.mongodb.com/manual/reference/operator/aggregation/geoNear/#pipe._S_geoNear)管道操作符必须为聚合管道中的第一个阶段

* 管道最大返回 16M 的数据，超出报错

#### 聚合表达式

##### 字段路径表达式

`$<feild>` 使用 `$` 来指示字段路径

`$<feild>.<sub-feild>` 使用 `$` 和 `.` 指示内嵌文档字段路径

```shell
# 某文档中 name 字段
$name

# 内嵌文档 info 的 createTime 字段
$info.createTime
```

##### 聚合表达式中的变量

聚合表达式变量分为系统变量和用户自定义变量

`$$<variable>` 使用 `$$` 来访问聚合表达式中的变量

如果变量引用了一个对象，要访问该对象中的特定字段，请使用点表示法 `$$<variable>.<field>`

```shell
# 管道中当前操作的文档
$$CURRENT
```

`MongoDB` 支持以下系统变量

| Variable  | Description                                                  |
| :-------- | :----------------------------------------------------------- |
| `ROOT`    | 引用当前正在聚合管道阶段处理的根文档，即顶级文档             |
| `CURRENT` | 引用聚合管道阶段中正在处理的字段路径的开始位置。除非另有说明，否则所有阶段都以`ROOT` 和 `CURRENT` 相同的开始 |
| `REMOVE`  | 一个计算缺失值的变量。允许条件排除字段。在 `$projection` 中，将字段设置为 `REMOVE` 该字段将从输出中排除 |

##### 常量表达式

`$literal: <value>` 指示常量 `<value>`

```shell
# 如果不使用 #literal 则 mongodb 认为 $name 是个字段路径表达式
# 使用 $literal 后 mongodb 就把 $name 作为一个字符串处理
$literal: "$name"
```

#### 聚合管道阶段操作符

##### $project

对文档进行投影（限制传递到下个阶段的文档字段）

将包含请求字段的文档传递到管道中的下一个阶段。指定的字段可以是输入文档或新计算字段中的现有字段

```
{ $project: { <specification(s)> } }
```

specification 使用的形式如下：

| 格式                    | 描述                         |
| ----------------------- | ---------------------------- |
| \<field>: <1 or true>   | 包含指定字段                 |
| _id: <0 or false>       | 指定_id字段不传递            |
| \<field>: \<expression> | 添加新字段或重置现有字段的值 |
| \<field>:\<0 or false>  | 指定排除的字段               |

**要点**

- `_id`默认情况下，该字段包含在输出文档中

- 要在输出文档中包含输入文档中的任何其他字段，必须明确指出

```shell
# 下面的管道输出 title、author 字段的值
db.books.aggregate( [ { $project : { _id: 0, title : 1 , author : 1 } } ] )
```

##### $match

过滤文档将符合指定条件的文档传递到下一个管道阶段

```
{ $match: { <query> } }
```

[`$match`](https://docs.mongodb.com/manual/reference/operator/aggregation/match/#pipe._S_match)获取指定查询条件的文档。查询语法与[读操作查询](https://docs.mongodb.com/manual/tutorial/query-documents/#read-operations-query-argument)语法相同

**使用要点**

* `$match` 使用越早要好（减少管道之间数据传递的数量）

```
db.articles.aggregate(
    [ { $match : { author : "dave" } } ]
);
```

##### $group

按指定的表达式对文档进行分组，并把每个不同的分组输出到下一个阶段

输出文档包含一个 `_id` 字段，该字段是不同分组的key。输出文档还可以包含计算字段，这些字段包含按 `$group` 的 `_id` 字段分组的某些累加器表达式的值。`$group` 不会对其输出文档进行排序

```shell
{
    $group: {
        _id: < expression > ,
        < field1 >: {
            < accumulator1 >: < expression1 >
        },
        ...
    }
}
```

`_id` 是必须的，但是可以给他 `null` 值，此时就把所有输入的文档分为一组，其他字段都是可选的

`accumulator` 操作符如下：

| Name                                                         | Description                                                  |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| [`$addToSet`](https://docs.mongodb.com/manual/reference/operator/aggregation/addToSet/#grp._S_addToSet) | Returns an array of *unique* expression values for each group. Order of the array elements is undefined. |
| [`$avg`](https://docs.mongodb.com/manual/reference/operator/aggregation/avg/#grp._S_avg) | Returns an average of numerical values. Ignores non-numeric values. |
| [`$first`](https://docs.mongodb.com/manual/reference/operator/aggregation/first/#grp._S_first) | Returns a value from the first document for each group. Order is only defined if the documents are in a defined order. |
| [`$last`](https://docs.mongodb.com/manual/reference/operator/aggregation/last/#grp._S_last) | Returns a value from the last document for each group. Order is only defined if the documents are in a defined order. |
| [`$max`](https://docs.mongodb.com/manual/reference/operator/aggregation/max/#grp._S_max) | Returns the highest expression value for each group.         |
| [`$mergeObjects`](https://docs.mongodb.com/manual/reference/operator/aggregation/mergeObjects/#exp._S_mergeObjects) | Returns a document created by combining the input documents for each group. |
| [`$min`](https://docs.mongodb.com/manual/reference/operator/aggregation/min/#grp._S_min) | Returns the lowest expression value for each group.          |
| [`$push`](https://docs.mongodb.com/manual/reference/operator/aggregation/push/#grp._S_push) | Returns an array of expression values for each group.        |
| [`$stdDevPop`](https://docs.mongodb.com/manual/reference/operator/aggregation/stdDevPop/#grp._S_stdDevPop) | Returns the population standard deviation of the input values. |
| [`$stdDevSamp`](https://docs.mongodb.com/manual/reference/operator/aggregation/stdDevSamp/#grp._S_stdDevSamp) | Returns the sample standard deviation of the input values.   |
| [`$sum`](https://docs.mongodb.com/manual/reference/operator/aggregation/sum/#grp._S_sum) | Returns a sum of numerical values. Ignores non-numeric values. |

示例：

```shell
# 按照 月、年 进行分组
db.sales.aggregate(
   [
      {
        $group : {
           _id : { month: { $month: "$date" }, year: { $year: "$date" } },
           totalPrice: { $sum: { $multiply: [ "$price", "$quantity" ] } },
           averageQuantity: { $avg: "$quantity" },
           count: { $sum: 1 }
        }
      }
   ]
)
```

```shell
# 将所有输入的文档分为一个文档
db.sales.aggregate(
   [
      {
        $group : {
           _id : null,
           totalPrice: { $sum: { $multiply: [ "$price", "$quantity" ] } },
           averageQuantity: { $avg: "$quantity" },
           count: { $sum: 1 }
        }
      }
   ]
)
```

```shell
# 去重
db.sales.aggregate( [ { $group : { _id : "$item" } } ] )
```

```shell
# 按照 author 分组，并将 title 的值添加到数组字段 books 中
db.books.aggregate(
   [
     { $group : { _id : "$author", books: { $push: "$title" } } }
   ]
)

# 其执行结果如下
{ "_id" : "Homer", "books" : [ "The Odyssey", "Iliad" ] }
{ "_id" : "Dante", "books" : [ "The Banquet", "Divine Comedy", "Eclogues" ] 
```

##### $skip

跳过指定数量的[文档](https://docs.mongodb.com/manual/reference/glossary/#term-document)（传入），并将剩余的文档传递到管道中的下一个阶段

```
{ $skip: <正整数> }
```

```shell
# 此操作会跳过管道传递给它的前5个文档。$skip对通过管道的文件内容没有影响
db.article.aggregate(
    { $skip : 5 }
);
```

##### $limit

限制传递到管道中下一个阶段的文档数

筛选出前几个文档传递给下个阶段

```
{ $limit: <正整数> }
```

##### $sort

对输入文档进行排序，并按排序顺序将它们返回到管道

```shell
{ $sort: { <field1>: <sort order>, <field2>: <sort order> ... } }
# sort order: 1 升序，-1 降序
```

```
db.users.aggregate(
   [
     { $sort : { age : -1, posts: 1 } }
   ]
)
```

##### $unwind

从输入文档展开数组字段以输出每个元素的文档。每个输出文档中数组字段的值由数组元素替换

```shell
# 操作数是一个字段路径
{ $unwind: <field path> }
```

```shell
# 操作数是一个文档
{
  $unwind:
    {
      path: <field path>,
      includeArrayIndex: <string>,
      preserveNullAndEmptyArrays: <boolean>
    }
}
```

参数说明：

`path`: 数组字段的字段路径。要指定字段路径，请在字段名称前加上美元符号$，并用引号括起来

`includeArrayIndex`: 可选的。新字段名字（值是元素在数组中的索引）。该名称不能以美元符号 `$` 开头

`preserveNullAndEmptyArrays`: 可选的，默认值为 `false`

​	`true`: 指定的数组字段的值为null、数组字段不存在或为空数组，`$unwind` 将输出文档

​	`false`: 则 `$unwind` 指定的数组字段的值为null、数组字段不存在或为空数组时不输出文档

**使用要点:**

* 如果不能展开 `path` 指定的数组或者 `path` 指定的不是一个数组，则将该字段作为一个单独的元素处理

```shell
# 假设有如下文档
{ "_id" : 1, "item" : "ABC", "sizes": [ "S", "M", "L"] }
{ "_id" : 2, "item" : "EFG", "sizes" : [ ] }
{ "_id" : 3, "item" : "IJK", "sizes": "M" }
{ "_id" : 4, "item" : "LMN" }
{ "_id" : 5, "item" : "XYZ", "sizes" : null }
```

```shell
# 1. 使用 $unwind 展开该文档的 sizes 数组
db.inventory.aggregate( [ {$match: {"_id": 1}}, { $unwind : "$sizes" } ] )

# 执行结果如下所示
{ "_id" : 1, "item" : "ABC1", "sizes" : "S" }
{ "_id" : 1, "item" : "ABC1", "sizes" : "M" }
{ "_id" : 1, "item" : "ABC1", "sizes" : "L" }
# 除了 sizes 字段的值外其他字段的值不变，sizes 为数组的每个元素值

# 2. 展开文档中的所有记录
db.inventory.aggregate( [ { $unwind : "$sizes" } ] )

# 执行结果如下
{ "_id" : 1, "item" : "ABC", "sizes" : "S" }
{ "_id" : 1, "item" : "ABC", "sizes" : "M" }
{ "_id" : 1, "item" : "ABC", "sizes" : "L" }
{ "_id" : 3, "item" : "IJK", "sizes" : "M" }

# 3. 使用 includeArrayIndex 指定将数组的下标输出到 arrayIndex 字段
db.inventory.aggregate([{
    $unwind: {
        path: "$sizes",
        includeArrayIndex: "arrayIndex"
    }
}])

# 执行结果
{ "_id" : 1, "item" : "ABC", "sizes" : "S", "arrayIndex" : NumberLong(0) }
{ "_id" : 1, "item" : "ABC", "sizes" : "M", "arrayIndex" : NumberLong(1) }
{ "_id" : 1, "item" : "ABC", "sizes" : "L", "arrayIndex" : NumberLong(2) }
{ "_id" : 3, "item" : "IJK", "sizes" : "M", "arrayIndex" : null }
# 如果 sizes 字段时单个值，则 arrayIndex 为 null
```

##### $lookup

对未分片的同一个数据的集合执行**左外链接**

对于每个输入文档，`$lookup` 阶段添加一个新的数组字段，其值是来自“已连接”集合的匹配文档。`$lookup` 阶段将这些重新组合的文档传递给下一个阶段

在 `$lookup` 阶段，无法对 `from` 集合进行分片。但是，可以对运行 `aggregate` 方法的集合进行分片

1. 相等匹配语法

   对输入文档的字段和连接文档的字段进行相等匹配

   ```
   {
      $lookup:
        {
          from: <collection to join>,
          localField: <field from the input documents>,
          foreignField: <field from the documents of the "from" collection>,
          as: <output array field>
        }
   }
   ```

   `$lookup` 参数说明: 

   | Field          | Description                                                  |
   | :------------- | :----------------------------------------------------------- |
   | `from`         | 指定相同数据库进行连接的集合名称                             |
   | `localField`   | 指定输入文档要进行匹配得字段，`$lookup` 使用 `from` 指定集合的文档的  `foreignField` 字段和输入文档的指定字段 `localField` 进行相等匹配<br />如果输入文档没有 `localField` 指定的字段，则 `$lookup` 认为 `localField` 指定字段的值是 `null` 然后再进行匹配 |
   | `foreignField` | 指定 `from` 集合的文档用于和 `localField` 指定字段进行匹配的字段 |
   | `as`           | 指定添加到输出文档中的数组字段名称<br />新的数组字段包含从 `from` 指定的集合匹配的文档<br />如果输入文档中已经有了指定的数组字段名称，则会覆盖现有的字段的值 |

2. 要在两个集合之间执行不相关的子查询以及允许除单个相等匹配之外的其他连接条件

   ```
   {
      $lookup:
        {
          from: <collection to join>,
          let: { <var_1>: <expression>, …, <var_n>: <expression> },
          pipeline: [ <pipeline to execute on the collection to join> ],
          as: <output array field>
        }
   }
   ```

   `$lookup` 参数说明: 

   | Field      | Description                                                  |
   | :--------- | :----------------------------------------------------------- |
   | `from`     | 指定相同数据库进行连接的集合名称                             |
   | `let`      | 可选<br />定义在 `pipeline` 阶段使用的变量<br />使用 `$expr` 操作符可访问 `let` 定义的变量<br />`let` 变量可由管道中的各个阶段访问，包括嵌套在管道中的其他 `$lookup` 阶段 |
   | `pipeline` | 指定在连接集合上执行的管道<br />指定空的 `pipeline: []` 则返回所有的连接文档<br />要访问 `let` 定义的变量，则需要使用 `$expr` 操作符，在 `$expr` 中使用 `$$变量名` 即可访问 |
   | `as`       | 指定添加到输出文档中的数组字段名称<br />新的数组字段包含从 `from` 指定的集合匹配的文档<br />如果输入文档中已经有了指定的数组字段名称，则会覆盖现有的字段的值 |

示例：

```shell
# orders.item == inventory.sku 进行左外链接
db.orders.aggregate([
   {
     $lookup:
       {
         from: "inventory",
         localField: "item",
         foreignField: "sku",
         as: "inventory_docs"
       }
  }
])
```

`MongoDB 3.6` 添加了 `$mergeObjects` 运算符，将多个文档合并到一个文档中

```shell
db.orders.aggregate([
   {
      $lookup:
         {
           from: "warehouses",
           let: { order_item: "$item", order_qty: "$ordered" },
           pipeline: [
              { $match:
                 { $expr:
                    { $and:
                       [
                         { $eq: [ "$stock_item",  "$$order_item" ] },
                         { $gte: [ "$instock", "$$order_qty" ] }
                       ]
                    }
                 }
              },
              { $project: { stock_item: 0, _id: 0 } }
           ],
           as: "stockdata"
         }
    }
])
```

##### $out

将聚合管道返回的文档写入到指定的集合

`$out` 必须是管道的最后一个阶段

```shell
# $out 接收一个字符串类型的集合名称（out 写入的集合）
{ $out: "<output-collection>" }
```

使用要点:

* 如果写入的集合不存在，则 `MongoDB` 会在当前数据库上创建该集合，聚合操作完成之前该集合是不可见的，如果聚合操作失败，则 `MongoDB` 不会创建该集合
* 如果写入的集合已经存在，那么在 `out` 阶段将使用新集合**替换**掉旧集合（旧的数据将会被清除）
* 不会清除旧集合中的索引
* 管道输出将会校验所有的索引（旧文档索引和新文档索引），如果校验失败则当前管道输出失败

### Map-Reduce



















## 查询选择器

### 比较运算符

|  Name  | Description              | 语法                                                   |
| :----: | :----------------------- | ------------------------------------------------------ |
| `$eq`  | 相等                     | { field: { $eq: value } }                              |
| `$gt`  | 大于                     | { field: { $gt: value } }                              |
| `$gte` | 大于等于                 | { field: { $gte: value} }                              |
| `$in`  | 在指定的集合中的所有记录 | { field: { $in:  [ value1, value2,  ...  valueN ] } }  |
| `$lt`  | 小于                     | { field: { $lt: value} }                               |
| `$lte` | 小于等于                 | { field: { $lte: value} }                              |
| `$ne`  | 不等于                   | { field: { $ne: value} }                               |
| `$nin` | 不在指定集合中的所有记录 | { field: { $nin:  [ value1, value2,  ...  valueN ] } } |

### 逻辑运算符

| 名称                                                         | 描述                               | 语法                                                         |
| :----------------------------------------------------------- | :--------------------------------- | ------------------------------------------------------------ |
| [`$and`](https://docs.mongodb.com/manual/reference/operator/query/and/#op._S_and) | 返回与多个子句的条件匹配的所有文档 | { $and: [ { <expression1> }, { <expression2> } , ... , { <expressionN> }] } |
| [`$not`](https://docs.mongodb.com/manual/reference/operator/query/not/#op._S_not) | 返回与查询表达式不匹配的文档       | { field: { $not: { <operator-expression> } } }               |
| [`$nor`](https://docs.mongodb.com/manual/reference/operator/query/nor/#op._S_nor) | 返回所有无法匹配多个子句的文档     | {  $nor: [ { <expression1> }, { <expression2>}, …, { <expressionN> } ] } |
| [`$or`](https://docs.mongodb.com/manual/reference/operator/query/or/#op._S_or) | 返回与任一子句的条件匹配的所有文档 | {  $or: [ { <expression1> }, { <expression2>}, ...   { <expressionN> } ] } |

### 元素

| 名称                                                         | 描述                             | 语法                                                         |
| :----------------------------------------------------------- | :------------------------------- | ------------------------------------------------------------ |
| [`$exists`](https://docs.mongodb.com/manual/reference/operator/query/exists/#op._S_exists) | 匹配具有指定字段的文档           | { field: { $exists: <boolean> } }                            |
| [`$type`](https://docs.mongodb.com/manual/reference/operator/query/type/#op._S_type) | 如果字段是指定类型，则选择文档。 | 单个表达式语法：{ field: { \$type: <BSON type> } }，<br/>多个表达式语法：{ field: { \$type: [ <BSON type1>, <BSON type2>, ... ] } } |

### 评价

| 名称                                                         | 描述                                             | 语法                                                         |
| :----------------------------------------------------------- | :----------------------------------------------- | ------------------------------------------------------------ |
| [`$expr`](https://docs.mongodb.com/manual/reference/operator/query/expr/#op._S_expr) | 允许在查询语言中使用聚合表达式。                 |                                                              |
| [`$jsonSchema`](https://docs.mongodb.com/manual/reference/operator/query/jsonSchema/#op._S_jsonSchema) | 根据给定的JSON模式验证文档。                     |                                                              |
| [`$mod`](https://docs.mongodb.com/manual/reference/operator/query/mod/#op._S_mod) | 对字段的值执行模运算，并选择具有指定结果的文档。 |                                                              |
| [`$regex`](https://docs.mongodb.com/manual/reference/operator/query/regex/#op._S_regex) | 返回与指定正则表达式匹配的文档                   | { <field>: { $regex: /pattern/, $options: '<options>' } }<br/>{ <field>: { $regex: 'pattern', $options: '<options>' } }<br/>{ <field>: { $regex: /pattern/<options> } } |
| [`$text`](https://docs.mongodb.com/manual/reference/operator/query/text/#op._S_text) | 执行文本搜索                                     |                                                              |
| [`$where`](https://docs.mongodb.com/manual/reference/operator/query/where/#op._S_where) | 匹配满足JavaScript表达式的文档                   |                                                              |

### 地理空间

| 名称                                                         | 描述                                                         |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| [`$geoIntersects`](https://docs.mongodb.com/manual/reference/operator/query/geoIntersects/#op._S_geoIntersects) | 选择与[GeoJSON](https://docs.mongodb.com/manual/reference/glossary/#term-geojson)几何体相交的几何。该[2dsphere](https://docs.mongodb.com/manual/core/2dsphere/)索引支持 [`$geoIntersects`](https://docs.mongodb.com/manual/reference/operator/query/geoIntersects/#op._S_geoIntersects)。 |
| [`$geoWithin`](https://docs.mongodb.com/manual/reference/operator/query/geoWithin/#op._S_geoWithin) | 选择边界[GeoJSON](https://docs.mongodb.com/manual/reference/geojson/#geospatial-indexes-store-geojson)几何体内的[几何](https://docs.mongodb.com/manual/reference/geojson/#geospatial-indexes-store-geojson)。该[2dsphere](https://docs.mongodb.com/manual/core/2dsphere/)和[2D](https://docs.mongodb.com/manual/core/2d/)索引支持 [`$geoWithin`](https://docs.mongodb.com/manual/reference/operator/query/geoWithin/#op._S_geoWithin)。 |
| [`$near`](https://docs.mongodb.com/manual/reference/operator/query/near/#op._S_near) | 返回点附近的地理空间对象。需要地理空间索引。该[2dsphere](https://docs.mongodb.com/manual/core/2dsphere/)和[2D](https://docs.mongodb.com/manual/core/2d/)索引支持 [`$near`](https://docs.mongodb.com/manual/reference/operator/query/near/#op._S_near)。 |
| [`$nearSphere`](https://docs.mongodb.com/manual/reference/operator/query/nearSphere/#op._S_nearSphere) | 返回球体上某点附近的地理空间对象。需要地理空间索引。该[2dsphere](https://docs.mongodb.com/manual/core/2dsphere/)和[2D](https://docs.mongodb.com/manual/core/2d/)索引支持[`$nearSphere`](https://docs.mongodb.com/manual/reference/operator/query/nearSphere/#op._S_nearSphere)。 |

### 数组

| 名称                                                         | 描述                                                   | 语法                                                     |
| :----------------------------------------------------------- | :----------------------------------------------------- | -------------------------------------------------------- |
| [`$all`](https://docs.mongodb.com/manual/reference/operator/query/all/#op._S_all) | 返回数组中包含所有指定值的记录（数组的值的顺序不考虑） | { <field>: { $all: [ <value1> , <value2> ... ] } }       |
| [`$elemMatch`](https://docs.mongodb.com/manual/reference/operator/query/elemMatch/#op._S_elemMatch) | 返回数组中至少有一个匹配所有条件的文档                 | { <field>: { $elemMatch: { <query1>, <query2>, ... } } } |
| [`$size`](https://docs.mongodb.com/manual/reference/operator/query/size/#op._S_size) | 如果数组字段是指定大小，则选择文档。                   |                                                          |

### 按位

| 名称                                                         | 描述                                                         |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| [`$bitsAllClear`](https://docs.mongodb.com/manual/reference/operator/query/bitsAllClear/#op._S_bitsAllClear) | 匹配数值或二进制值，其中一组位位置*都*具有值`0`。            |
| [`$bitsAllSet`](https://docs.mongodb.com/manual/reference/operator/query/bitsAllSet/#op._S_bitsAllSet) | 匹配数值或二进制值，其中一组位位置*都*具有值`1`。            |
| [`$bitsAnyClear`](https://docs.mongodb.com/manual/reference/operator/query/bitsAnyClear/#op._S_bitsAnyClear) | 匹配数值或二进制值，其中来自一组位位置的*任何*位的值都为`0`。 |
| [`$bitsAnySet`](https://docs.mongodb.com/manual/reference/operator/query/bitsAnySet/#op._S_bitsAnySet) | 匹配数值或二进制值，其中来自一组位位置的*任何*位的值都为`1`。 |

### 评论

| 名称                                                         | 描述                 |
| :----------------------------------------------------------- | :------------------- |
| [`$comment`](https://docs.mongodb.com/manual/reference/operator/query/comment/#op._S_comment) | 向查询谓词添加注释。 |

### 投影操作员

| 名称                                                         | 描述                                                         |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| [`$`](https://docs.mongodb.com/manual/reference/operator/projection/positional/#proj._S_) | 投影数组中与查询条件匹配的第一个元素。                       |
| [`$elemMatch`](https://docs.mongodb.com/manual/reference/operator/projection/elemMatch/#proj._S_elemMatch) | 投影数组中与指定[`$elemMatch`](https://docs.mongodb.com/manual/reference/operator/projection/elemMatch/#proj._S_elemMatch)条件匹配的第一个元素。 |
| [`$meta`](https://docs.mongodb.com/manual/reference/operator/projection/meta/#proj._S_meta) | 投影在[`$text`](https://docs.mongodb.com/manual/reference/operator/query/text/#op._S_text)操作期间分配的文档分数。 |
| [`$slice`](https://docs.mongodb.com/manual/reference/operator/projection/slice/#proj._S_slice) | 限制从数组投射的元素数量。支持跳过和限制切片。               |

## 索引

如果没有索引 `MongoDB` 在查找的时候必须进行全集合扫描，如果创建了合适的索引 `MongoDB` 就可以使用索引来限制扫描的文档数量

索引是特殊的数据结构，它以易于遍历的形式存储集合数据集的一小部分。索引存储特定字段或字段集的值，按字段的值排序。索引条目的排序支持有效的等式匹配和基于范围的查询操作。此外，MongoDB可以使用索引中的顺序返回排序结果

索引以升序（1）或降序（-1）排序顺序存储对字段的引用

### _id 默认索引

`MongoDB` 在创建集合时在 `_id` 字段上创建唯一索引。此索引不能被删除

### 创建索引

```
db.collection.createIndex( <keys>, <options> )
```

参数说明：

| Parameter | Type     | Description                                                  |
| :-------- | :------- | :----------------------------------------------------------- |
| `keys`    | document | 包含 `field` 和 `value` 的文档，其中 `field` 是索引 `key`，值是 `field` 的索引类型<br />`value` 等于 `1` 表示升序索引，`-1` 表示降序索引 |
| `options` | document | 可选的，一组选项文档，控制如何创建索引                       |

`optons` 文档常用选项说明：

| Parameter          | Type    | Description                                                  |
| :----------------- | :------ | :----------------------------------------------------------- |
| unique             | boolean | 可选的，创建一个唯一索引，`true` 创建唯一索引，默认值为 `false` |
| name               | string  | 可选的，索引名称，不指定则由 `MongoDB` 生成                  |
| sparse             | boolean | 可选的，默认值 false，true 创建稀疏索引，稀疏索引引用 field 字段存在的文档，如果文档的 field 不存在，则不会添加到索引中 |
| expireAfterSeconds | integer | 可选的，指定一个值（以秒为单位）作为TTL来控制 `MongoDB` 在此集合中保留文档的时间。即超过设置的时间后该文档被删除 |

```shell
# 在 collection 上创建 name 字段的降序索引
db.collection.createIndex( { name: -1 } )
```

当索引包含查询扫描的所有字段时，索引支持查询





### 索引类型

#### 单字段索引

除 `MongoDB` 定义的 `_id` 索引外，`MongoDB` 还支持在文档的单个字段上创建用户定义的升序/降序索引

```shell
# 在 score 字段上查询时将会使用该索引
db.collection.createIndex( { score: 1 } )
```

单字段索引升序/降序都可以

#### 复合索引

多个字段的用户定义索引，即复合索引。

复合索引的排序非常重要

对于复合索引和排序操作，索引键的排序顺序（即升序或降序）可以确定索引是否可以支持排序操作

```
db.collection.createIndex( { <field1>: <type>, <field2>: <type2>, ... } )
```

除了支持在所有索引字段上匹配的查询之外，复合索引还可以支持与索引字段的前缀匹配的查询

```
假如创建复合索引：{A: 1, B: 1, C: 1}
那么下面的查询将被该索引支持：
	{A}
	{A, B}
	{A, B, C}

下面的查询该索引不支持
	{C}
	{B}
	{B, C}
	{A, C}

也就是说查询的字段的顺序必须从第一个字段开始且必须是连续的，同时查询的排序也要和定义的相同或者逆向
```

```shell
# 创建一个复合索引
db.products.createIndex( { "item": 1, "stock": 1 } )

# 下面查询都被上面创建的索引支持
db.products.find( { item: "Banana" } )
db.products.find( { item: "Banana", stock: { $gt: 5 } } )

# 不被支持
db.products.find( { stock: 9 } )
```

##### 前缀

索引前缀是索引字段的起始子集

```shell
# 假设下面的索引
{ "item": 1, "location": 1, "stock": 1 }

# 那么他的前缀如下
{ item: 1 }
{ item: 1, location: 1 }
```

对于复合索引，MongoDB可以使用索引来支持对索引前缀的查询。因此，MongoDB可以在以下字段中使用索引进行查询：

- the `item` field,
- the `item` field *and* the `location` field,
- the `item` field *and* the `location` field *and* the `stock` field.

### 查看索引

#### 查询集合中的所有索引

```
db.collection.getIndexes()
```

#### 查询数据库中的所有索引

```shell
db.getCollectionNames().forEach(function(collection) {
   indexes = db[collection].getIndexes();
   print("Indexes for " + collection + ":");
   printjson(indexes);
});
```

### 删除索引

#### 删除单个索引

从集合删除指定索引

不能删除 `_id` 字段上的默认索引

```shell
db.collection.dropIndex(<string or document>)
# <string or document> 指定要删除的索引。可以通过索引名称或索引定义文档指定索引
# 索引的名称或者索引定义文档可通过 db.collection.getIndexes() 获得

# 删除除 _id 之外的所有的索引
db.accounts.dropIndexes()
```

**注意** 此命令在受影响的数据库上获取写锁定，并将阻止其他操作，直到完成为止

```shell
# 假如调用方法获得集合上的所有索引
db.collection.getIndexes()

# 获得结果如下
[
   {  "v" : 1,
      "key" : { "_id" : 1 },
      "ns" : "test.pets",
      "name" : "_id_"
   },
   {
      "v" : 1,
      "key" : { "cat" : -1 },
      "ns" : "test.pets",
      "name" : "catIdx"
   }
]

# 删除 catIdx 索引
db.collection.getIndexes("catIdx")
db.collection.getIndexes({ "cat" : -1 })
```

### 修改索引

要修改现有索引，需要删除并重新创建索引

### 索引使用情况（执行计划）

#### 聚合管道执行计划

使用 `$indexStats` 返回有关集合的每个索引的使用统计信息。如果使用访问控制运行，则用户必须具有包含`indexStats` 操作的权限

```shell
{ $indexStats: { } }

# 聚合管道也可使用 explain() 方法查询执行计划
db.collection.explain().aggregate()
```

返回信息说明：

| Output Field | Description                                                  |
| :----------- | :----------------------------------------------------------- |
| `name`       | 索引名称                                                     |
| `key`        | 索引定义文档                                                 |
| `host`       | `mongod` 进程的主机名和端口。                                |
| `accesses`   | 索引使用统计：<br />        `ops` 使用索引的操作符数量<br />        `since` MongoDB 收集统计数据的时间 |

`accessses` 字段报告的统计信息仅包括由用户请求的索引访问。 

`$indexStats` 必须是聚合管道中的第一个阶段

`$indexStats` 不允许在事物中使用

#### 查询执行计划

`db.collection.explain()` 或 `cursor.explain()` 方法返回有关查询过程的统计信息，包括使用的索引，扫描的文档数以及查询处理的时间（以毫秒为单位）

##### db.collection.explain()

返回以下操作的查询计划信息：

```
aggregate()
count()
find()
group()
remove()
update()
distinct()
findAndModify()
```

```shell
# method 是上面列出的方法
db.collection.explain().<method(...)>
```

```shell
db.products.explain().remove( { category: "apparel" }, { justOne: true } )
```

对于写入操作 `db.collection.explain()` 返回有关将要执行但不实际修改数据库的写入操作的信息

查看 `explain()` 支持的方法列表:

```
db.collection.explain().help()
```

```shell
# 例子
db.products.explain().count( { quantity: { $gt: 50 } } )
```

## Spring Boot 操作 MongoDB

1. 在 `MongoDB` 中新建用户（拥有 `readWrite` 权限）

2. 在 `application.yml` 新增 `MongoDB` 配置

   ```yaml
   spring:
     data:
       mongodb:
         host: 47.75.110.73		# mongodb 主机ip
         port: 27017						# mongodb 主机端口号
         username: testUser		# 需要操作的数据库的用户名（该用户需要拥有 readWrite 权限）
         password: 1qaz				# 密码
         database: test				# 连接的数据库
   ```

3. 在 `Dao` 中注入 `MongoTemplate`

   ```java
   @Autowired
   private MongoTemplate mongoTemplate;
   ```

4. 创建 bean

   ```java
   /**
    * @Document注解把一个java类声明为mongodb的文档，可以通过collection参数指定这个类对应的文档
    * @Document(collection="mongodb 对应 collection 名")
    */
   @Document(collection = "customer")
   public class Customer implements Serializable {
   
       // @Id 标注主键，不可重复，自带索引，可以在定义的列名上标注，需要自己生成并维护不重复的约束
       // @Id 标注的字段与 mongodb 的 _id 关联
       @Id
       public String id;
   
       public String firstName;
       public String lastName;
   
       public Customer() {}
   
       public Customer(String firstName, String lastName) {
           this.firstName = firstName;
           this.lastName = lastName;
       }
   }
   ```

5. 使用 `MongoTemplate` 操作数据库

   ```java
   mongoTemplate.insert(new Customer("aa", "bb"));
   ```

   



























