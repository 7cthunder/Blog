---
title: SWAD | Homework4
date: 2019-05-22 15:16:29
categories:
  - SWAD
tags:
  - SWAD
---
## 简答题

* **用例的概念**
  * 用例是描述一个参与者使用一个系统来实现一个目标的成功或失败场景的集合
* **用例和场景的关系？什么是主场景或 happy path？**
  * 场景就是参与者和系统的交互过程，由若干行为和会话组成的特定序列构成，也被称为用例的实例。
  * 用例事实上就是一系列场景的集合，例如：主场景，正零路径或者是其他备选流
  * 主场景就是最常用的一个业务场景，反映的是用户最为基本的目的，简单的说就是这个系统所实现的基本业务
* **用例有哪些形式？**
  * 摘要 (Brief)：一段总结，通常是主要的成功场景的建立，只需要几分钟的时间
  * 简便格式 (Casual)：不是很正式的形式，并且包括了不同的场景
  * 详述 (Fully)：完整的场景建立，完整的步骤以及用例的细节，以及一些可能发生的情况的应对以及如何确保一个成功的场景
* **对于复杂业务，为什么编制完整用例非常难？**
  * 因为复杂业务的子用例非常多，流程复杂，且需要处理的场景很多，很难考虑完全所有子用例和场景，绘制的用例图繁杂，容易出错
* **什么是用例图？**
  * 用例图是表示系统上下文的一张图片，它显示了系统的边界，展示了与系统交互的外部对象，描述了系统的使用方法
* **用例图的基本符号与元素？**
  * 参与者 (Actor)：表示一个系统用户，包括与应用程序进行交互的用户、组织或外部系统
  * 用例 (Use Case)：表示一个用例，通常用作对系统提供的功能、服务的一种描述
  * 包含关系 (Includes)
  * 扩展关系 (Extends)
  * 关联关系 (Association)
* **用例图的画法与步骤：**
  * 确定系统边界
  * 识别 Actors
    * 识别使用系统的主要参与者 (primary actors) / 角色(roles)
      * 使用用例图 actor符号表示，通常放在系统的左边
      * 企业应用可以通过企业组织架构，业务角色与职责识别
      * 互联网应用则必须通过市场分析，确定受众范围
      * **千万不要用“用户”代表系统使用者，以避免过于通用导致缺乏用户体验**。例如，你的系统服务对象是程序员，但你必须明白 c/c++ 程序员、java 程序员、 PHP 程序员之间的相同与不同
    * 识别系统依赖的外部系统
      - 使用用例图 Neighboursystem框 表示用例依赖的外部系统、服务、设备，并使用构造型（Stereotype）识别
        - <<system>> 例如：Account System（财务系统），教务系统
        - <<service>> 例如：第三方身份认证、地理信息服务、短信服务等第三方在线服务
        - <<device>> 或 <<sensor>> 例如：GPS 等等
  * 识别用例 (服务)
    * 识别用户级别用例（user goal level）
      - 以主要参与者目标驱动
      - 收集主要参与者的业务事件
      - 必须满足以下准则
        - boss test
        - EBP test
        - Size Test
      - manage 用例。特指管理一些事物的 CRUD 操作，例如管理文件、管理用户等
    * 识别子功能级别的用例（sub function level）
      - 子用例特征
        - 业务复用。例如：现金支付
        - 复杂业务分解。将业务分解为若干步，便于交互设计与迭代实现
        - 强调技术或业务创新。例如：“识别人脸”，尽管从用户角度是单步操作，但背后涉及技术解决方案是比较复杂的
      - 正确使用用例与子用例之间的关系
        - <<include>> 表示子用例是父用例的一部分，通常强调离开这个特性，父用例无法达成目标或失去意义！
        - <<extend>> 表示子用例是父用例的可选场景或技术特征。
        - <<include>> **箭头指向子用例**；<<extend>> **箭头指向父用例**。箭头表示的依赖关系！
  * 建立 Actor 和 Use Cases 之间的关联
    - 请使用 **无方向连线**，表示两间之间是双向交互的协议
* **用例图给利益相关人与开发者的价值有哪些？**
  * 用例图对于利益相关者来说，他们可以非常直观的了解到自己所要实现的功能是否被很好的体现，开发者是否理解了自己的需求；而对于开发者来说，这不仅是向客户传递自己对需求的理解，也是方便开发者进行系统设计和开发

## 建模练习题（用例模型）
![Ctrip Use Case](https://github.com/7cthunder/Blog/raw/master/source/assets/Ctrip-UseCase.png)
* **然后，回答下列问题**

  * **为什么相似系统的用例图是相似的？**

    * 因为相似系统提供的服务是相似或相同的，面对的用户和用例是相似的，而用例之间的关系是由服务确定的，也是相似的，即使有特色的扩展服务，也是在基本业务上的扩展，都是为了满足相同的需求而提出的，所以最终相似系统的用例图是相似的

  * **如果是定旅馆业务，请对比 Asg_RH 用例图，简述如何利用不同时代、不同地区产品的用例图，展现、突出创新业务和技术**

    * 在现在，可以使用大数据的方法分析每个用户对旅馆类型、价格、位置等的偏好，给每个用户推荐最适合他们的旅馆
    * 对于不同地区的旅馆，可以结合当地特色，在用户选择时为用户介绍，帮助用户做出最佳的选择

  * **如何利用用例图定位创新思路（业务创新、或技术创新、或商业模式创新）在系统中的作用**

    * 在用例图中对创新用例使用某种颜色进行高亮标记，可以很方便地让需求方、开发人员快速了解该系统的创新功能以及该模块相关依赖和输入输出结果

  * **请使用 SCRUM 方法，选择一个用例图，编制某定旅馆开发的需求（backlog）开发计划表**

|  ID  | Title      | Est  | Imp  |             Demo              |
| :--: | :--------- | :--: | :--: | :---------------------------: |
|  1   | 登录       |  2   |  20  |   官方、微信、阿里账号登录    |
|  2   | 预定酒店   |  5   |  30  |      搜索酒店、管理订单       |
|  3   | 搜索酒店   |  2   |  40  |       多种方式搜索酒店        |
|  4   | 城市搜索   |  2   |  30  |          按城市搜索           |
|  5   | 地图搜索   |  10  |  20  | 调用第三方地图API来按地图搜索 |
|  6   | 标志物搜索 |  5   |  10  |        按标志物来搜索         |
|  7   | 订单管理   |  3   |  30  |  添加、删除、修改、查询订单   |
|  8   | 支付       |  3   |  40  | 使用银行卡、微信、支付宝支付  |

  * **根据任务4，给出项目用例点的估算**

| 用例       | # 事务 | # 计算 | 原因 | UC 权重 |
| ---------- | :----: | :----: | ---- | :-----: |
| 登录       |   3    |   2    |      |  简单   |
| 预定酒店   |   8    |   8    |      | 复杂性  |
| 搜索酒店   |   4    |   3    |      | 复杂性  |
| 城市搜索   |   1    |   1    |      |  简单   |
| 地图搜索   |   1    |   1    |      |  简单   |
| 标志物搜索 |   1    |   1    |      |  简单   |
| 订单管理   |   4    |   4    |      |  平均   |
| 支付       |   3    |   3    |      |  平均   |