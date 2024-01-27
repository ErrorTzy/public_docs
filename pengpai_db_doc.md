# 前置条件

## 程序

**SQLite3**：

使用（最？）广泛的轻量级数据库

官网： https://www.sqlite.org/index.html

下载： https://www.sqlite.org/download.html

安装说明：见本文档的**前置条件-知识-SQL-中文SQLite SQL教程-SQLite安装**





**simple**：

SQLite FTS5中文分词插件，用于加速检索数据库中的中文

Github： https://github.com/wangfenjin/simple

下载： https://github.com/wangfenjin/simple/releases/tag/v0.3.0

安装说明：选择系统对应的压缩包，并解压至任意一个文件夹





**Python3**（optional）

一个运行python代码的程序，如果需要使用pengpai_data_export.py辅助数据导出，那么需要安装python3

官网： https://www.python.org/

下载： https://www.python.org/downloads/

安装说明：见本文档的**前置条件-知识-Python3语法-基础教程-python3环境搭建**

## 知识

尽管本文档将简单地对这些知识进行最基础的介绍，因此**可以在不具备以下知识的情况下对数据库进行操作**；但完全实现对数据库中的数据的掌控需要以下知识：

**SQL语法**：

操作数据库所使用的语言

中文SQL教程（比较全面）： https://www.runoob.com/sql/sql-tutorial.html

中文SQLite SQL教程（比较简练）： https://www.runoob.com/sqlite/sqlite-tutorial.html


**SQLite FTS5 Extension语法**:

对FTS5虚拟表进行进行字符串查询时所使用的查询语法

教程： https://www.sqlitetutorial.net/sqlite-full-text-search/



**Python3语法**（optional）：

一种编程语言，可以对数据库中的数据进行更复杂的操作

基础教程： https://www.runoob.com/python3/python3-tutorial.html

python3操作sqlite3数据库： https://www.runoob.com/sqlite/sqlite-python.html

## 硬件

尽管使用机械硬盘/使用低速接口依然能够对数据库进行操作，并且数据库已经进行了许多优化使得能够事先毫秒级查询。但是当不可避免地出现全表扫描式的操作时，那么你需要一块速度组够快的固态硬盘。基本的下限是，在执行查询时，读取速度能够达到200MB/s。

# 使用说明

## 数据表结构

本数据库内的数据以**表**的形式存储。除了`pengpai_optimizer`和`pengpai_raw`这两个表外，还应当存在以下五张表：
- `pengpai_optimizer_docsize`
- `pengpai_optimizer_config`
- `pengpai_optimizer_content`
- `pengpai_optimizer_data`
- `pengpai_optimizer_idx`

但这五张表并不实际存储可操作的数据，而是创建虚拟表`pengpai_optimizer`所自动生成的，因此忽视它们就可以了

### pengpai_optimizer

`pengpai_optimizer`实际上就是将`pengpai_raw`中`number`，`title`和`content`复制了出来。但是`pengpai_optimizer`对字符串的全文搜索进行了优化，大大加快了速度。例如，对`pengpai_raw`中的`content`字段进行查询需要数分钟，而`pengpai_optimizer`只需要几毫秒。

| 字段名 | 说明                                                                      | 例                                       |
| ------ | ------------------------------------------------------------------------- | ---------------------------------------- |
| number | `pengpai_raw`中的`number`字段，用于存储一条新闻的唯一id，但是是字符串类型 | 16575845                                 |
| title  | 新闻的标题                                                                | 冬奥会运动员最易受伤的身体部位，它排第一 |
| content       | 新闻的正文（不包含图片/视频等多媒体信息）                                                                          | 有来阅读原文                                         |

### pengpai_raw

| 字段名         | 说明                                                   | 例                                                  |
| -------------- | ------------------------------------------------------ | --------------------------------------------------- |
| link           | 新闻文章链接                                           | https://www.thepaper.cn/newsDetail_forward_16575845 |
| number         | 链接中的最后那一串数字，是每一篇新闻的独一无二的标识id | 16575845                                            |
| title          | 新闻的标题                                             | 冬奥会运动员最易受伤的身体部位，它排第一            |
| author         | 新闻的作者                                             | 有来医生                                            |
| date           | 以YYYY-mm-dd格式的字符串标识的新闻发布日期             | 2022-02-08                                          |
| timestamp      | 新闻的准确发布时间，单位为毫秒                         | 1644275832085                                       |
| content        | 新闻的正文（不包含图片/视频等多媒体信息）              | 有来阅读原文                                        |
| content_length | 新闻正文的长度                                         | 6                                                   |
| source_info    | 新闻的来源                                             | 澎湃新闻·澎湃号·湃客                                |
| is_repetitive               | 这篇文章是否是洗稿/转发文章；如果是，为1；如果不是，为0                                                       | 0                                                    |

## 操作

### 心理准备

如果你已经用惯了各种软件，如Office/WPS/Adobe全家桶等，你可能会认为任何一个软件，都有一个交互界面，而用户这个界面上点击一些按钮，在一些窗口中输入，等等。对于面向普通用户的程序而言确实如此，因为普通用户并不会用别的方式操作程序。但对于面向程序员的程序而言，交互界面并不是一个好的选择。我认为原因主要是两方面：

- 第一，程序员懒得开发这样的界面，因为这种界面本身就是一个额外开发出来的程序，开发和设计它需要大量时间。并且是程序就会出bug，程序员不希望自己的核心代码没有出错，但是界面出错而无法使用自己的程序，而界面类的程序因为操作系统的原因，很容易出bug。
- 第二，如果一个程序能够接受一些字符串（即文字）作为输入而不是操作界面，这意味着它可以很方便地自动化。例如我可以提前创建一个文本文档，这个文本文档中存储了操作一个程序的指令。这样一来，我可以直接让程序去读取这个文本文档的指令来执行命令。此外，这样能大幅度提高效率。并且假设一共有一万条指令，人的反应是0.14秒，那么人使用界面进行操作，不算程序运行时间，人的反应时间就需要1400秒；但计算机的反应在微秒级别，大约只需要十几秒。

而由于本数据过大（共1300万篇新闻），无法使用日常办公软件操作，必须使用专业的工具进行数据处理，这就必须要使用命令行。告诉自己：在命令行上敲命令是正常的，习惯它。

### 定义
当本文档中使用以下任意一个名称时，它们都可以被替换它们实际的值。例如：

- `$EXAMPLE`: 一个任意的文件夹路径，例如`C:\example folder\` 

那么当本文档中提到在命令行输入命令：`cd "$EXAMPLE"`，那么它就可以被代换为`cd "C:\example folder\"`

- `$PATH_TO_DB`: 
	- `pengpai.db`所在路径，例如`F:/pengpai/pengpai.db`。在本文档中为`/media/scott/Seagate Basic/pengpai/pengpai.db`。
- `$PATH_TO_SIMPLE`: 
	- 解压simple至一个文件夹后，名为`libsimple`的文件路径，例如`F:/simple/libsimple.so`。在本文档中为`/media/scott/Seagate Basic/pengpai/libsimple.so`
- `$PATH_TO_CSV`:
	- 如果需要导出数据库的结果到csv文件，那么你就需要假定一个csv文件的路径（此时这个文件还不能存在）。例如：`F:/pengpai/output.csv`，此时pengpai文件夹下没有文件`output.csv`。在本文档中为`/media/scott/Seagate Basic/pengpai/output.csv`。
- `PATH_TO_PY`:
	- 如果需要运行`pengpai_data_export.py`，那么你也需要得到这个文件的路径，例如`F:/pengpai/pengpai_data_export.py`。在本文档中为`python3 "/media/scott/Seagate Basic/pengpai/pengpai_data_export.py`




### 打开数据库

打开命令行（Windows: cmd/powershell Mac: terminal Linux: bash)，输入`sqlite3 "$PATH_TO_DB"`（如果路径中有空格，那么必须有双引号；如果没有，可以不加。下同。），应当得到类似的结果：

```bash
scott@debian:~$ sqlite3 "/media/scott/Seagate Basic/pengpai/pengpai.db"
SQLite version 3.40.1 2022-12-28 14:03:47
Enter ".help" for usage hints.
sqlite> 
```

此时可以在`sqlite>`后方输入指令，对这个数据库进行操作。本文档以下内容都默认已打开了数据库，并在`sqlite>`后输入指令。
由于`pengpai_optimizer`使用了simple插件，此时还需要导入插件。输入`.load "$PATH_TO_SIMPLE$"`，如果没有任何输出并跳出了下一行则成功：

```bash
scott@debian:~$ sqlite3 "/media/scott/Seagate Basic/pengpai/pengpai.db"
SQLite version 3.40.1 2022-12-28 14:03:47
Enter ".help" for usage hints.
sqlite> .load "/media/scott/Seagate Basic/pengpai/libsimple.so"
sqlite> 
```


### 关闭数据库

如果你已经打开了一个数据库，那么你可以输入`.quit`关闭它，回到正常的命令行界面。以下例子打开了数据库，然后关闭了它：
```bash
scott@debian:~$ sqlite3 "/media/scott/Seagate Basic/pengpai/pengpai.db"
SQLite version 3.40.1 2022-12-28 14:03:47
Enter ".help" for usage hints.
sqlite> .quit
scott@debian:~$ 
```


### 查询数据库
很不幸，查询数据库需要使用SQL语句，而它可以变得极其复杂。但它有一些简单的运用，而这些简单的运用并不需要超出小学生智力水平就能够理解的知识。

#### Level I：先看一眼数据长什么样
现在向已经打开了的数据库中输入`SELECT number,date,title FROM pengpai_raw ORDER BY timestamp ASC LIMIT 10;`并回车，你将会看到以下结果：

```sql
sqlite> SELECT number,date,title FROM pengpai_raw ORDER BY timestamp ASC LIMIT 10;
1243574|2014-03-31|王石、冯仑、周其仁继续“在商言商”
1243818|2014-04-02|3小时闭门家属会，马航只公布了六次握手航迹
1243929|2014-04-10|奉化一街道干部自杀身亡
1244041|2014-04-13|国务院发文要求遏止“官谣”，专家称设专门机构处置
1244042|2014-04-14|黑龙江K7034次列车事故调查
1244054|2014-04-14|环保部：兰州自来水苯超标与监管不力有关
1244078|2014-04-14|重庆开县高考移民案调查：公安局政委被查， 涉逾90名河南考生
1244356|2014-04-23|陆家嘴：国企改革和B股改革都在抓紧研究
1244367|2014-04-23|澳大利亚欲购58架F-35战机
1244426|2014-04-23|伶人三部曲：一次成功的“勾引”
sqlite> 
```

现在来解释`SELECT number,date,title FROM pengpai_raw ORDER BY timestamp ASC LIMIT 10;`的含义：
- 第一部分：`SELECT number,title FROM pengpai_raw`
	- `FROM`后是**表名**，如`pengpai_raw`或`pengpai_optimizer`。
	- `SELECT` 和 `FROM`之间的是这个表中的一个或多个**字段名**。如果有多个，那么用逗号隔开。
	- `pengpai.db`中表名和字段名，见本文档的**使用说明-数据表结构-你要查询的表名**
- 第二部分：`ORDER BY timestamp ASC` （optional）
	- `ORDER BY` 和 `ASC` 之间是FROM后的表名中的一个字段。这一部分对整个表中数据根据timestamp字段的值进行**升序排列**的操作。与之类似的还有`ORDER BY timestamp DESC`，这将会进行降序排列。（ASC是ascending的首字母，DESC是descending的首字母）
	- 可以没有这个部分（试一试）
- 第三部分： `LIMIT 10`（optional）
	- LIMIT限制了显示的结果数量。鉴于整个表有一千多万行数据，正常情况下没有人希望这一千多万行数据全部倾泻到命令行上，所以我在这里作出了限制。
	- 可以没有这个部分（不建议试一试，但是也可以试一试）
- 第四部分：`;`
	- **重要！** 分号标识了一个SQL语句的结束。如果没有分号但是敲了回车，那么SQLite3会认为这个语句没有结束，因此不会执行任何操作而是等待用户输入`;`。这也意味着在SQL中，命令可以分行输入以增加可读性，例如如下语句和之前的命令等价，但显然更美观：

```sql
sqlite> SELECT
   ...>   number,date,title
   ...> FROM
   ...>   pengpai_raw
   ...> ORDER BY
   ...>   timestamp ASC
   ...> LIMIT
   ...>   10;
1243574|2014-03-31|王石、冯仑、周其仁继续“在商言商”
1243818|2014-04-02|3小时闭门家属会，马航只公布了六次握手航迹
1243929|2014-04-10|奉化一街道干部自杀身亡
1244041|2014-04-13|国务院发文要求遏止“官谣”，专家称设专门机构处置
1244042|2014-04-14|黑龙江K7034次列车事故调查
1244054|2014-04-14|环保部：兰州自来水苯超标与监管不力有关
1244078|2014-04-14|重庆开县高考移民案调查：公安局政委被查， 涉逾90名河南考生
1244356|2014-04-23|陆家嘴：国企改革和B股改革都在抓紧研究
1244367|2014-04-23|澳大利亚欲购58架F-35战机
1244426|2014-04-23|伶人三部曲：一次成功的“勾引”
sqlite> 
```


#### Level II：通过筛选时间来获取数据

现在我可以通过`SELECT title FROM pengpai_raw`来获取所有新闻的标题。但我要怎么获取某一个特定时期的新闻，例如特定时间范围内的所有新闻？
首先思考：我应该针对哪一个表和字段进行筛选？关于时间的字段有两个，它们都在`pengpai_raw`中，因此关于时间的查询都应该是`SELECT ... FROM pengpai_raw`。那么在pengpai_raw中，关于时间的字段是date和timestamp。date精确到日期，并且是字符串；timestamp精确到毫秒，并且是数字。先说结论：
**所有关于时间的条件都应当对timestamp进行筛选，而不是date。**
原因：
1. 我已经在timestamp上建立了索引，因此查询timestamp的速度远快于date。在一千多万的数据量面前，速度很重要：对date查询要几分钟，而timestamp是毫秒级的。
2. timestamp是数字，因此可以支持比大小，加减等计算；date是字符串，不能（在数值意义上）进行比较小和加减，只能判断字符串是否相等。
3. timestamp精确到毫秒，而date只精确到天

现在假定我的需求是：查询2022年2月22日20点00分00秒-2022年2月22日22点00分00秒的所有新闻，我该怎么做？
有无数种生成时间戳的办法，但最简单的办法是打开这个网站 https://tool.lu/timestamp/ ，生成**单位为毫秒**的时间戳。例如：
2022年2月22日22点00分00秒： 1645538400000
2022年2月22日22点02分22秒： 1645538542000
那么我们需要查询的范围就是timestamp处于1645538400000和1645538542000之间的所有数据。尝试输入语句：`SELECT number,title,date FROM pengpai_raw WHERE timestamp BETWEEN 1645538400000 AND 1645538542000;` 这时候，会得到数据：

```sql
sqlite> SELECT number,title,date FROM pengpai_raw WHERE timestamp BETWEEN 1645538400000 AND 1645538542000;
16808305|紧急！太原市疫情防控办最新提示|2022-02-22
16811363|【难忘的五年】数据|2022-02-22
16814558|毕节打造具有市域特点的社会治理新模式：百姓平安生活再添“安全锁”|2022-02-22
16814561|贵州：校企合作成立首个跨境电商学院|2022-02-22
16814560|黄粑香 日子甜丨新春走基层|2022-02-22
16814562|毕节8条高速封路！|2022-02-22
16815382|事发长沙！男子推电动车进电梯伤及孕妇|2022-02-22
16814444|威县人社局“直播带岗”取得显著成效|2022-02-22
16814447|威县一老乡将征战北京冬残奥会！认识吗？|2022-02-22
16812212|建设儿童友好城市，请你来发声！|2022-02-22
16812226|儿童友好(24)丨人间至味是团圆 古韵风雅过元宵|2022-02-22
16814046|关于武汉市武昌区凯莱熙酒店来菏人员主动报备的公告|2022-02-22
16814051|菏泽市第二十届人大第一次会议高新区代表团分团会议召开|2022-02-22
16814048|菏泽高新区办公室系统2022年工作会议召开|2022-02-22
16814056|菏泽高新区：生产要素聚集 生物医药产业成为发展“主引擎”|2022-02-22
16808877|【疫情防控】当阿尔兹海默症“遇上”新冠 医务人员用温情治愈身心|2022-02-22
16808878|20220222 瑞丽有爱|2022-02-22
16814362|今天，也太有爱了吧！|2022-02-22
sqlite> 
```

现在来解释这个语句的含义。`SELECT`和`FROM`都是在Level I中已经介绍过了的关键词。这里新的是`WHERE timestamp BETWEEN 1645538400000 AND 1645538542000`
`WHERE`后跟随的是一个“条件表达式”。例如：
```sql
... WHERE timestamp BETWEEN 1645538400000 AND 1645538542000`
-- 在sql中，between a and b是包含a和b的。例如，BETWEEN 1 AND 3的自然数是1，2，3
... WHERE timestamp = 1645538542000
-- 判断相等
... WHERE timestamp > 1645538542000
... WHERE timestamp < 1645538542000
... WHERE timestamp >= 1645538542000
... WHERE timestamp <= 1645538542000
--比较大小
... WHERE (timestamp >= 1645538400000) AND (timestamp <= 1645538542000)
--多个条件之间的链接。这个和BETWEEN 1645538400000 AND 1645538542000完全等价
```

更多的运算符，见 https://www.runoob.com/sqlite/sqlite-operators.html

#### Level III: 通过筛选内容来获取数据

在Level II中，基本上所有对于数值的筛选都可以完成了。但是我要怎么通过对字符串（文本）的条件来进行筛选呢？例如我想要知道所有正文中包含"缅甸"且包含"诈骗"的新闻，我该怎么做？
首先，所有的对文本内容进行查询都要在pengai_optimizer中进行。尽管在pengpai_raw中完全可以进行等价的文本查询，但是会非常慢非常慢。

##### III.1 单表查询
如果我只需要获得这些新闻的number字段（例如，我只需要统计总共有多少篇新闻包含这个字段），那么这个查询只需要在pengpai_optimizer中进行。输入`SELECT number FROM pengpai_optimizer WHERE content MATCH '缅甸 AND 诈骗' LIMIT 10;`，会得到：

```sql
sqlite> SELECT number FROM pengpai_optimizer WHERE content MATCH '缅甸 AND 诈骗' LIMIT 10;
1257326
1258158
1292442
1311542
1321900
1334257
1364283
1388159
1393500
1435213
sqlite> 
```

在这里，新的关键词是`MATCH`，`MATCH`是FTS5的语法。
例如，'泰国 AND 诈骗'将会匹配所有同时具有 "泰国" 和 "诈骗" 关键字的文章内容
例如，'电诈 OR 诈骗'将会匹配所有具有 "电诈" 或 "诈骗" 关键字的文章内容
具体教程见本文档**前置条件-知识-SQLite FTS5 Extension语法-教程**

##### III.2 多表查询
但很不幸，一些需求是筛选一段时间内包含特定内容的新闻。这意味着，我既要用到pengpai_optimizer进行字符串查询，又要用到pengpai_raw进行时间查询。那么我要怎么做到这个需求？
假如我现在要查询的是2023年6月28日（时间戳范围1687881600000-1687967999999），所有包含"缅甸"和"诈骗"，或包含"缅甸"和"电诈"的新闻。我们已经能够对其分开来进行查询：

```sql
SELECT number,...其它字段 FROM pengpai_raw WHERE timestamp BETWEEN 1687881600000 AND 1687967999999;
SELECT number FROM pengpai_optimizer WHERE content MATCH '(缅甸 AND 诈骗) OR (缅甸 AND 电诈)';
```

如果把number看作为一个对象，而把所有其它字段看作number对象的属性，那么我们所需要的是这两个筛选出的数据的**交集**。那么要怎么在SQL中取交集？

输入：
```sql
SELECT 
  pengpai_raw.number,
  pengpai_raw.title,
  pengpai_raw.date,
  pengpai_raw.is_repetitive
FROM 
  pengpai_raw 
  JOIN
    (SELECT number FROM pengpai_optimizer WHERE content MATCH '(缅甸 AND 诈骗) OR (缅甸 AND 电诈)') AS optimizer_number
  ON
    optimizer_number.number = pengpai_raw.number
WHERE 
  timestamp BETWEEN 1687881600000 AND 1687967999999;
```

此外，我在这条命令之前输入了`.timer on`回车开启了计时（这不是一个SQL语句而是一个SQLite3指令，所以不需要分号结尾），于是我得到了：
```sql
sqlite> .timer on
sqlite> SELECT 
  pengpai_raw.number,
  pengpai_raw.title,
  pengpai_raw.date,
  pengpai_raw.is_repetitive
FROM 
  pengpai_raw 
  JOIN
    (SELECT number FROM pengpai_optimizer WHERE content MATCH '(缅甸 AND 诈骗) OR (缅甸 AND 电诈)') AS optimizer_number
  ON
    optimizer_number.number = pengpai_raw.number
WHERE 
  timestamp BETWEEN 1687881600000 AND 1687967999999;
23650946|“枪都顶到嘴里了！”捡回一条命的他，讲述噩梦般的经历......|2023-06-28|0
23651058|同住人出借车辆给第三人，第三人无证驾驶发生交通事故，实际车主赔不赔？|2023-06-28|0
23656645|起底电诈⑬丨毒打视频曝光！境外电诈回流人员讲述亲身经历 这是一条恐怖的黑色产业链……|2023-06-28|0
23665949|普法强基在行动 | 元阳警方成功规劝6名犯罪嫌疑人投案|2023-06-28|0
23648417|全民反诈 | 毒打视频曝光！境外电诈回流人员讲述亲身经历 这是一条恐怖的黑色产业链……|2023-06-28|1
23653537|全民反诈 | 毒打视频曝光！境外电诈回流人员讲述亲身经历 这是一条恐怖的黑色产业链……|2023-06-28|1
23661266|全民反诈 | 毒打视频曝光！境外电诈回流人员讲述亲身经历 这是一条恐怖的黑色产业链……|2023-06-28|1
Run Time: real 146.169 user 1.188153 sys 1.136403
sqlite> 
```
（这条指令在缓慢的机械硬盘上共花费了146秒）

这里引入的取交集的新句法是：
```sql
...
表a 
JOIN 
  表b 
ON 
  一些条件 
...
```

值得注意的是：
1. 由于这里引入了多张表，所以每一个字段必须有一个表的归属，以免出现重名的字段时SQLite3不知道用哪个表的字段。因此，这里的SELECT后跟的字段为`pengpai.number`, `pengpai.title`, ...。
2. 所有SELECT语句产生的东西都是一张临时表，它没有名字。那么我们在join这样没有名字的临时表时，我们就需要使用AS关键字来给它取名字，以方便后续操作
3. ON后面的条件是判断两行是否相等的条件。数据库所查询出来的东西可以理解为一个**列**作为元素的集合，而列具有多个字段的属性。当我们取交集的时候，无非就是取出两个集合中相等的元素，而ON后面就是定义列元素相等的条件。
4. 事实上这里并不是直接对两个查询结果取交集，而是先把optimizer中取出的number和总表取了交集，然后把交集的总表做了一次筛选。也可以把总表筛选一次，再把optimizer中的number筛选出来，最后两个取交集（试一试）。

#### Level IV: 统计数量

##### IV.1 count(\*)

我们可以在SELECT后使用`count(*)`来统计将会筛选出多少行。例如：
```sql
sqlite> SELECT count(*) FROM pengpai_raw;
13689303
sqlite> SELECT count(*) FROM pengpai_raw WHERE content_length != 0 AND is_repetitive=0;
11123213
```
这说明pengpai_raw这个表格共有13689303篇新闻数据，且有11123213篇是正文不为空且不是洗稿/转发文章的新闻数据。

##### IV.2 GROUP BY

但在IV.1中，我只能一个个地查询。但如果我对需求是分别统计有多少篇非重复文章以及有多少篇重复文章，有没有更自动化的办法而不需要手动输入？
```sql
sqlite> SELECT count(*),is_repetitive FROM pengpai_raw GROUP BY is_repetitive;
11129056|0
2560247|1
```

这意味着在所有的数据中，共有11129056是不重复的，而有2560247被表极为重复数据。`GROUP BY`会对它后面跟的值进行判断，并把相同的分为一组。这样一来`count(*)`的行为就会变为对每一个组进行统计。
在`pengpai.db`中，`GROUP BY`后通常可以接`date`或者`is_repetitive`。

### 导出数据库

#### 内置方法
SQLite3可以对任意一个查询导出为csv。这里涉及了一系列指令：
```sql
sqlite> .headers on --如果不适用这个指令，那么csv就会没有第一行标题；如果要导出多个，这个指令只需要开始执行一次就可以了
sqlite> .mode csv --将输出调整为csv
sqlite> .output "/media/scott/Seagate Basic/pengpai/output.csv" --这里的文件路径就是$PATH_TO_CSV，不能事先存在
sqlite> SELECT number,title,date FROM pengpai_raw WHERE timestamp BETWEEN 1645538400000 AND 1645538542000; --执行这个查询，这个查询的结果就会导入到$PATH_TO_CSV文件内
sqlite> .output stdout --保存$PATH_TO_CSV文件
```


#### 现成的Python程序
说明：这个python程序用于处理一个特殊的需求，即按照日期批量导出多个匹配特定规则的数据，并且区分is_repetitive的值。这个程序在速度上进行了优化，能够较为快速地进行查询。
依赖：已安装python3

使用任意一个文本编辑器（notepad/nano/vim...）打开`pengpai_data_export.py`。它的前几行内容如下：

```python
import csv
from datetime import datetime
from time import gmtime, strftime
import sqlite3
import pytz

kws = ['缅甸']
DB_FILE = "/home/scott/Desktop/WorkingFiles/storage/pengpai/pengpai.db"
EXTENTION_FILE = "/home/scott/Desktop/WorkingFiles/storage/pengpai/libsimple.so"
OUTPUT_PATH = "./output/"

RAW_TABLE_NAME = "pengpai_raw"
OPTIMIZED_TABLE_NAME = "pengpai_optimizer"
AN_HOUR = 3600 # seconds
A_DAY = 24 * AN_HOUR
A_DAY_IN_MS = A_DAY * 1000
TIMEZONE = pytz.timezone('Asia/Shanghai')
...
```

一般地，需要修改的四个参数是：
```python
kws = ['一些符合FTS5 MATCH语法的query','用逗号隔开','用单引号或双引号把它们围起来']
DB_FILE = $PATH_TO_DB
EXTENTION_FILE = $PATH_TO_SIMPLE
OUTPUT_PATH = '' # 这里的参数类似于$PATH_TO_CSV，但是不需要指定文件名，而是指定输出的文件夹。请保证这个输出的文件夹存在。
```

然后打开命令行，不需要打开sqlite3数据库，输入`python $PATH_TO_PY`或`python3 $PATH_TO_PY`（不同的系统会用不同的名字。在linux上使用python3，在windows上使用python等，具体看python --version有反应还是python3 --version有反应）并回车，例如:
```bash
python3 "/media/scott/Seagate Basic/pengpai/pengpai_data_export.py"
```
它就会在OUTPUT_PATH文件夹下创建相关文件了
