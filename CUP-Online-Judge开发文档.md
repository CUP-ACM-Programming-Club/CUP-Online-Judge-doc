**CUP Online Judge**是一个以Vue.js作为前端框架，ExpressJS 作为后端框架开发的在线评测系统。

*注: CUP Virtual Judge属于较为独立的、技术栈未成熟的另外一个程序，因此在本程序中不会涉及到与Virtual Judge系统相关的接口开发问题。*

阅读本文档需要了解以下预备知识:
* Node.js(v10.0+)
* JavaScript(ES~~5~~6)
* PHP(v7.0+)(**Deprecated***)
* Linux
* C++(11+)
* HTML(HTML5 is optional)
* Vue.js
* CSS3
* SQL

*: PHP即将在`去PHP`计划中被删除。
## 程序架构

本程序参照了HUSTOJ与SYZOJ的架构，采用了MySQL + Node.js + Linux C/C++ ~~+ PHP~~技术栈进行开发。

下列列出的本程序的项目已经开源。
* <a href="https://github.com/CUP-ACM-Programming-Club/CUP-Online-Judge-Express" target="_blank">基于Node.js Express开发的后端</a>
* <a href="https://github.com/CUP-ACM-Programming-Club/CUP-Online-Judge-Judger" target="_blank">基于简易沙箱的判题机</a>
* <a href="https://github.com/CUP-ACM-Programming-Club/CUP-Online-Judge-docker-judger" target="_blank">基于Docker的判题机</a>
* ~~<a href="https://github.com/CUP-ACM-Programming-Club/CUP_Online_Judge_Semantic_UI" target="_blank">基于Semantic UI构建的、使用Vue.js作为框架的前端</a>~~<a href="https://github.com/ryanlee2014/CUP-Online-Judge-NG-FrontEnd" target="_blank">基于Vue-Cli 3.0构建的Vue.js单页应用程序</a>
* <a href="https://github.com/CUP-ACM-Programming-Club/CUP-Online-Judge-Problem-Packer" target="_blank">使用Electron Vue开发的跨平台题目打包管理器</a>
* ~~<a href="https://github.com/ryanlee2014/HUSTOJ-Flat-UI-Theme" target="_blank">基于Flat UI开发的PHP事务下的主题</a>~~

下列部分~~暂未开源，或者~~由于即将被替代/与其他开源项目的功能重复决定不做开源处理。
* 基于Medoo编写的旧版的PHP事务部分
* 基于Semantic UI与Bootstrap开发的主题
* 管理员管理后台

关于本程序采用的开源软件，请访问[开放源代码声明](https://www.cupacm.com/opensource.php)查看。

## 写在开发前

为了更好的方便内部模块间的协同工作，请认真理解本程序的工作流程，即使它非常的混乱、冗余。

在为本程序进行开发时请时刻牢记**稳定**与**必要**是该程序功能增改的一大原则。

以下是本程序在用户登录后模块间通信的图示。
![](/img/program_structure.png)

可以分为以下部分。
### ~~Node.js Runtime~~基于Node.js Express开发的后端
使用ExpressJS框架，主要功能：
* 使用Socket.IO与用户进行双向通信
* CRUD
* 维护判题队列
* 用户权限管理
* 题目文件部署
* WebSocket转发
* 信息广播

#### 程序文件夹
\- root
\--- static 静态文件文件夹(js库/css库)
\--- route 路由文件夹(API接口)
\--- bin 启动文件夹(程序启动文件)
\--- module 模块文件夹(被其他文件调用)
\--- middleware 中间件文件夹(中间件)
\--- views 模版文件夹(pug文件模板)
\--- logs 日志文件夹(日志)

#### 程序启动(Daemon)
`npm start`

#### 程序启动(Debug)
`npm test`

#### 程序重启(Daemon)
`npm restart`

### C++ Judger

使用EasyWebsocket、HUSTOJ Judger进行重构、开发
主要功能:
判题:

## 一个提交的执行流程

### 客户端
1. 用户点击`提交`按钮
2. 代码、token被打包，委托Socket.IO模块推送到Node.js Runtime
3. Socket.IO模块接受`result`事件，调用挂在`window`上的`problemsubmitter`Vue实例，Vue实例执行相关操作，显示判题情况。

### 服务器 
##### Node.js Runtime
1. (/bin/main.js)收到`submit`事件响应的数据，将数据移交`submitControl`模块，判断提交合法性并解包数据，存储至数据库中
2. 等待模块返回完成信号，建立socket连接与solution_id的映射，将判题任务移交到判题机
3. 判题机维护判题队列。
   - 若队列空闲则直接唤起空闲判题机移交判题任务
   - 若队列非空，则赋予高优先权移交队列，等待判题机结束后询问队列，入队判题
4. 通过WebSocket模块接收来自判题机的判题信息，通过查阅solution_id对应的socket连接，将数据转发给对应socket
5. WebSocket模块通过检测完成信号，释放相关资源
6. 判题机判题结束，检查队列任务，若队列非空，则获取队列头的任务进行判题，若队列为空，则判题机入空闲队列

#### C++ Judger
1. 被Node.js Runtime唤醒，建立简易沙箱环境，获取MySQL连接
2. 初始化系统调用，载入数据文件
3. 判断提交类型
   - 测试运行：不进行答案比对，直接返回评测数据，程序结束
   - 正常提交：根据题目内容进行评测，根据数据库取出的数据决定判题方法为`文本对比`或`Special Judge`
     - 文本对比：对比标准输出与用户输出的内容，返回`Wrong Answer`/`Presentation Error`/`Accept`信息
     - Special Judge: 将输入文件、输出文件、用户输出、用户代码传入程序，根据程序退出时返回的值决定判题结果。理论上可以返回所有的判题结果。

4. 数据保存，更新用户数据与题目数据
5. 退出程序

## PHP与Node.js之间如何共享用户数据

在每次访问PHP页面时，`/include/db_info.inc.php`文件作为所有PHP文件的头文件被引入到PHP文件中，该文件包括了数据库相关的函数初始化以及缓存数据库的初始化。同时该文件会生成一个和用户`user_id`相关的`token`,并将`token`和`user_id`一同保存在cookie中，在页面进行XHR时cookie会一同发送到Node.js运行时环境 

## Node.js Router开发

要使前端能够调用接口，访问数据，就需要开发对应的Router响应时间。所有的Router文件均放置于文件夹`/router`下。编写Router的方式与Express.js官网的模板相似，只需导出数组
```javascript
module.exports = ["router_path", middleware, Router]
```
就能够被app.js自动在启动时载入。

## Node.js Module开发

所有的后台功能若是能够抽象的，应该开发成`Module`以供使用。开发的Module存放在`/module`文件夹下即可

## Node.js Socket.IO开发

该模块是后台程序诞生的原因~~显示在线人数~~,因此这里的代码是所有代码中历史最久远的部分。目前还没有整理方便置入中间件或绑定响应时间的接口。请等待版本更新。

## 前端开发

前端是本系统中一个非常重要的部分。考虑到服务器本身对CPU的依赖，于是所有的非保密数据的运算均由客户端(即用户浏览器)运算得出。

本平台界面采用Semantic UI进行构建，因此能够支持~~Semantic UI~~Fomantic UI最新文档下的所有模块及特效。

大多数的页面由最新版本的Vue.js驱动。由于`去PHP计划`还没有全面完成，部分页面仍将使用旧版的PHP模块驱动。
目前还使用PHP模块驱动的页面有(此处不考虑CUP Virtual Judge):
- ~~竞赛及作业~~
- 管理员后台*
- ~~编译错误页面~~
- ~~运行错误页面~~

删除在`3.0.0-alpha`版本中已重构的部分
*: 正在重构中

以上的页面将在不久的将来被Vue页面替代，因此在这些页面上进行功能增删需要三思。

当前所有的前端请求均通过**AJAX**与后端保持通信，~~考虑到Semantic UI仍然基于jQuery进行开发，为了减少过多第三方库的引入，建议直接使用jQuery的接口进行XHR请求~~请直接在`CUP-Online-Judge-NG-Frontend`中使用`this.axios`进行AJAX操作。

有少数的功能不能够使用AJAX通信，而必须使用Socket.IO接口与后端通信。
- 历史页面最新提交的更新
- 题目提交页面评测题目
- 黑板
- 全局信息推送
- 建立WebSocket时提交的环境数据
- 在线用户情况
- 评测队列

## Git
### 版本
开发版本格式为：`${major-version}.${maintain-version}.${bugfix-version}`

#### 如何针对代码的改变选择适合的版本号变化
当出现以下变化的，大版本增加:
* 核心代码完全重构
* 环境更改

当出现以下变化的，小版本增加:
* 增加前/后端模块链
* 修改前/后端模块链
* 新增/修改Plugin
* 新增非重要功能
* 删除模块/UI更改

当出现以下变化时，修复版本增加:
* 修复Bugs
* 更新依赖
* 更正typo
* 新增/修改测试用例
