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

##### 系统变量表达式

`$$<variable>` 使用 `$$` 来指示系统变量

```shell
# 管道中当前操作的文档
$$CURRENT
```

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