---
layout: post
title: "backtrader broker"
subtitle: "回测模拟券商行为"
date: 2023-01-27 20:21:17
author: "dimi"
header-img: "img/post-bg-alitrip.jpg"
catalog: true
tags: [coding,quant,backtrader]
---

##  Backbroker（模拟券商）

由于国内暂未发现有真实（live）券商，咱们只能用模拟Broker进行回测，Backtrader提供了BackBroker类用来模拟券商的行为。如果要实时交易，需要手动挂单，有些券商提供很好的条件单功能。

## BackBroker类的继承关系

Backtrader的继承关系比较简单，就先不提供图了。

```python
class BackBroker(bt.BrokerBase)
class BrokerBase(with_metaclass(MetaBroker, object))
class MetaBroker(MetaParams)
```

从类定义来看，BackBroker也继承了元类，所以其实例化也会受元类控制，下面在实例化和初始化过程进行说明。



## 佣金方案 （CommInfoBase）

针对国内券商的佣金，需要单独开发佣金方案；一般情况下，实例化Cerebro之后，就可以直接设置佣金方案了

```python
cerebro.broker.setcommission(commission=0.001)  #设定交易费用0.1%（买卖都收）
```

Cerebro获取broker，然后调用setcommission设置佣金：

```python
def setcommission(self,commission=0.0, margin=None, mult=1.0,
                  commtype=None, percabs=True, stocklike=False,
                  interest=0.0, interest_long=False, leverage=1.0,
                  automargin=False, name=None):

                comm = CommInfoBase(commission=commission, margin=margin, mult=mult,
                                    commtype=commtype, stocklike=stocklike,
                                    percabs=percabs,
                                    interest=interest, interest_long=interest_long,
                                    leverage=leverage, automargin=automargin)
    self.comminfo[name] = comm
```

Broker首先会实例化一个CommInfoBase，然后记录到commoninfo字典中，这里要注意，一个Broker可以拥有多个CommInfoBase，通过name来区分，name和Data类创建时候使用的名字一致，也就是不同的数据源（对应不同的资产）佣金方案可以不同。在相同的券商中，针对不同的资产不同的佣金，例如对于股票和ETF佣金不同，这里就可以设置不同佣金方案。CommInfoBase就是设置相关参数，关键参数具体解释如下表：

| 参数            | 缺省值   | 含义                    |
| ------------- | ----- | ----------------------------- |
| commission    | 0.0   | 就是基本佣金数据，按照百分比或者绝对值。百分比就是按照交易的金额（size*price）的百分比提取佣金。绝对值，就是针对size提取固定佣金。比如一笔交易（或者一个合约）收取2元钱。至于具体是采取百分比还是固定值，参见下面参数说明。           |
| commtype      | None  | 就是用来指定佣金数据是百分比还是固定值。<br>ommInfoBase.COMM_PERC：按照百分比解释。<br>CommInfoBase.COMM_FIXED：按照固定值解释。<br>None：这种情况下系统根据资产情况自动判决，判决方法参见参数margin。为啥搞得这么复杂，主要是为了兼容原来老的类（CommissionInfo），原来是通过margin来判断是否百分比。                |
| percabs       | False | 当 commtype 设置为 COMM_PERC 时，指定百分比数字的填写方法。比如说，你要设置0.1%，这个值设置为True，那么commission参数填写为0.001，为False的话，只要填写为0.1.为啥搞得怎么麻烦？还是兼容性问题。因为老版本中commission填写值不包括百分号，还记得咱们之前的例子要求“0.1% ，需要除以100然后去掉%”,这个就是老的处理方式，在新版本中，如果保持老的处理方式，percaps需要填写为TRUE。 |
| stocklike     | False | 这里用来指示股票类资产还是期权类资产，在老的CommissionInfo，是通过margin来确定是否期权。在新的类中，当commtype为None的时候，可以通过这个参数来指示是否股票类资产。                |
| margin        | None  | 保证金，港股称为孖展。Margin为0或者None，那么commission就是按照百分比解释。否则按照固定值解释。主要是因为Margin主要针对的期货/期权产品，通常按合约计价。这个是之前类的使用方法，新版本不再采用。      |
| mult          | 1.0   | 杠杆比，这个主要应用于期权等可以加杠杆的资产，Backtrader据此计算盈利和亏损。      |
| name          | 空     | 前面已叙及。             |
| interest      | 0.0   | 利息。有些情况下，需要对所持资产计算利息，例如借入证券卖空，由于这些对于我们个人交易者使用不多，咱们先忽略。          |
| interest_long | False | 是否对多头仓位收取利息，和上面一样，是否对做多的资产收取利息，先忽略。                |

### 佣金方案的定制

首先，继承CommInfoBase类。然后根据你的需求修改参数和重载函数。

1. 简单的自定义方法，修改参数

    针对百分比的股票佣金方案

    ```python
    class CommInfo_Stocks_Perc(CommInfoBase):
        params = (
            ('stocklike', True),
            ('commtype', CommInfoBase.COMM_PERC),
        )
    ```
    
    假如你希望按照老方法输入0.001表示0.1%,那么还可以这样定义：

    ```python
    class CommInfo_Stocks_Perc(CommInfoBase):
        params = (
            ('stocklike', True),
            ('commtype', CommInfoBase.COMM_PERC),
            ('percabs', True),
        )
    ```

    针对固定值的期权佣金方案

    ```python
    class CommInfo_Futures_Fixed(CommInfoBase):
        params = (
            ('stocklike', False),
            ('commtype', CommInfoBase.COMM_FIXED),
        )
    ```

    这样的话，使用的时候只需要输入commision就行了，其他参数就不要关注了。

2. 如果需要修改佣金的计算方法，那么你可以重载 _getcommission 函数，然后通过Cerebro调用broker.addcommissioninfo添加这个方案。**注意不是使用setcommission**，因为这个函数只是针对原有的CommissionInfo对象。注意，可以设定佣金方案对应的资产，通过data的name来指定。

    ```python
    def _getcommission(self, size, price, pseudoexec):
    '''根据规模以及价格计算佣金'''

    def addcommissioninfo(self, comminfo, name=None):
        '''Adds a ``CommissionInfo`` object that will be the default for all assets if
        ``name`` is ``None``'''
        self.comminfo[name] = comminfo

    comminfo = CommInfo_Stocks_PercAbs(commission=0.001)  # 0.1%
    cerebro.broker.addcommissioninfo(comminfo)
    ```

3. 一个实现案例

    - 佣金按照百分比。
    - 每一笔交易有一个最低值，比如5块，当然有些券商可能会免5.
    - 卖出股票还需要收印花税。 成交金额*0.1% 仅卖出
    - 可能有的平台还需要收平台费。
    - 沪市过户费 成交金额*0.001% 买卖双向
    - 股息税

        ```python
        # 设置交易费用
        class MyStockCommissionScheme(bt.CommInfoBase):
            '''
            1.佣金按照百分比。
            2.每一笔交易有一个最低值，比如5块，当然有些券商可能会免5.
            3.卖出股票还需要收印花税。 成交金额*0.1% 仅卖出
            4.可能有的平台还需要收平台费。
            5.沪市过户费 成交金额*0.001% 买卖双向
            6.股息税
            '''
        params = (
            ('stampduty', 0.001),  # 印花税率
            ('commission', 0.00025),  # 佣金率
            ('stocklike', True),#股票类资产，不考虑保证金
            ('commtype', bt.CommInfoBase.COMM_PERC),#按百分比
            ('minCommission', 5),#最小佣金
            # ('maxCommissionPct', 0.003),#最大佣金比例

            ('platFee', 0),#平台费用
        )

        def _getcommission(self, size, price, pseudoexec):
            '''
            size>0，买入操作。
            size<0，卖出操作。
            '''
            if size > 0:  # 买入，不考虑印花税，需要考虑最低收费
                return max(size * price * self.p.commission, self.p.minCommission) + self.p.platFee
            elif size < 0:  # 卖出，考虑印花税。
                return max(abs(size) * price * self.p.commission,self.p.minCommission)+abs(size) * price * self.p.stampduty + self.p.platFee
            else:
                return 0  # 防止特殊情况下size为0.
        ```




