# beancount学习和使用 一

一直使用excel进行记账, 2013年工作后用一个晚上手写公式做了个账本, 之后大修了二次, 2014年末基本就不再变化了, 剩下就是小修小补了.

excel问题在于我想要的功能已经不好改了, 很多都是固定范围很难扩展, 最想要的是标签但我的模板已经无法支持了, 买卖有价证券也无法进行记录, 开始寻找替代方案. 最重要的一个要求就是能结构化导出或本地运行,最后靠着RSS发现了一个好用的工具[beancount](https://github.com/beancount/beancount). 可以解决核心要求, 剩下的功能基本都是满足的.

本文就是自己学习内容

## [复式记账法](https://zh.wikipedia.org/zh-hans/%E5%A4%8D%E5%BC%8F%E7%B0%BF%E8%AE%B0)

账本设计采用复式记账法, 所有账务都涉及两个以上的账户. 从多个账户转入多个账户. 始终保证所有账户加和的钱是0

`(收入 + 负债) + (资产 + 费用) + 权益 = 0`

`(Income + Liabilities) + (Assets + Expenses) + Equity = 0`

> 用大白话来说就是：你赚的钱（Income），加上你借来的钱（Liabilities），最终要么变成你自己的钱（Assets），要么就是花掉了（Expenses），最终得到的是个零。这就是人的一辈子……

借方和贷方实例:下表演示了借方和贷方的增减如何影响会计要素。

| 会计要素 | 借方 | 贷方 |
| --- | --- | --- |
| 资产 | ▲ | ▼ |
| 费用 | ▲ | ▼ |
| 负债 | ▼ | ▲ |
| 业主权益 | ▼ | ▲ |
| 收入 | ▼ | ▲ |

> 我个人习惯, 所有记录, 都是一对一, 一对多, 多对一, 不要出现多对多情况.

## beancount账本结构

以`;`开头是注释

* 建立账户

```
; 时间 open/close 类型:一级账户名:二级账户名:... 货币类型
2013-07-01 open Assets:Bank:ICBC CNY
```
> 特别说明, 账户名称是英文的话必须是大写字母开头

账户类型:

* 资产 Assets —— 现金、银行存款、有价证券等；
* 负债 Liabilities —— 信用卡、房贷、车贷等；
* 收入 Income —— 工资、奖金等；
* 费用 Expenses —— 外出就餐、购物、旅行等；可以用来做分类会归总
* 权益 Equity —— 用于「存放」某个时间段开始前已有的豆子，下文详述。

* 交易/账单

```
; * 确定的, !表示不确定的, 可以没有收款人
; 时间 */! "收款人" "说明" #tag #tag ^link
2016-01-01 * "捡到钱了" #surprise
  id: ""            ; 原信息, 账单id等
  Income:Windfall                            -100.00 CNY
  Assets:Cash                                +100.00 CNY
```

> tag不支持中文

* 全局设置

```
;【一、账本信息】
option "title" "我的账本" ;账本名称
option "operating_currency" "CNY" ;账本主货币
```

* 管理

通过`include "***.bean"`导入其他账本. 方便切分文件

```
include "2021-03-01.bean" ; 3月1日
include "2021-03-02.bean" ; 3月2日
include "2021-03-03.bean" ; 3月3日
include "2021-03-04.bean" ; 3月4日
```

* 定期平账

以周或月为单位进行平账, 让所有账务和为零, 防止错账.

```
2021-03-01 balance 账户 金额 CNY
```

## 部署和安装

推荐docker部署, 简单暴力.

我使用的两个docker镜像
```
# linux/amd64/mac
docker pull yegle/fava
# 启动
docker run -p 5000:5000 -v $PWD:/bean -e BEANCOUNT_FILE=/bean/example.bean yegle/fava

# pi
docker pull lykhee/fava-docker
# 启动
docker run  -p 5000:5000 -v $PWD:/bean -e BEANCOUNT_FILE=/bean/main.bean lykhee/fava-docker
```

python直接安装
```shell
python -m venv BEANCOUNT
source BEANCOUNT/bin/active
pip install beancount # 命令行程序
pip install fava    #界面展示
```

## 参考

### 官网
* [beancount](https://github.com/beancount/beancount)
* [Beancount – Language Syntax](https://docs.google.com/document/d/1wAMVrKIA2qtRGmoVDSUBJGmYZSygUaR0uOMW1GV3YE0/)
* [Beancount – Getting Started](https://docs.google.com/document/d/1P5At-z1sP8rgwYLHso5sEy3u4rMnIUDDgob9Y_BYuWE/)
* [Beancount – Syntax Cheat Sheet](https://docs.google.com/document/d/1M4GwF6BkcXyVVvj4yXBJMX7YFXpxlxo95W6CpU3uWVc/)

### blog

* [Beancount —— 命令行复式簿记](https://wzyboy.im/post/1063.html)
* [使用 Beancount 记录证券投资](https://wzyboy.im/post/1317.html)
* [记账神器 Beancount 教程](https://www.skyue.com/19101819.html)
* [握着你的手写最简单的 BEANCOUNT 账本](https://lyric.im/c/beancount-tutorial/beancount-tutorial-1)
* [beancount总结](https://byvoid.com/zhs/tags/beancount/)
