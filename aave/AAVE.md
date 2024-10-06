* 此篇文章将从以下几方面讲解 AAVE 的核心原理：
  * 利率
    * 资金利用率
    * 浮动利率
      * 借款利率
      * 存款利率
    * 固定利率
  * 存款取款
  * 借款还款
  * 清算
  * V3



# 利率

## 资金利用率

* 计算公式：
  $$
  U=\frac {Total\; borrowed} {Total\; supplied}
  $$

* `U` 资金利用率

* `Total borrowed` 资产总借出数量

* `Total supplied` 资产总存入数量

* 可以从 AAVE [详情页](https://app.aave.com/reserve-overview/?underlyingAsset=0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2&marketName=proto_mainnet_v3) 看到这些数据，如下图所示：

  * ![img2024-09-19 07.35.22](/Users/zhouwei/Desktop/contract/aave/img/:Users:zhouwei:Desktop:img2024-09-19 07.35.22.png)

  * ![img2024-09-19 07.31.09](/Users/zhouwei/Desktop/contract/aave/img/:Users:zhouwei:Desktop:img2024-09-19 07.31.09.png)

  * 可以从上面的数据计算出 `U`：差值是与合约计算精度问题

    * $$
      U= \frac {1.06M} {1.18M}=0.8983
      $$

## 浮动利率

### 借贷利率

* 计算公式：

  * $$
    \left\{
    \begin{aligned}
    & R_t=R_0+\frac {U_t}{U_{optimal}} R_{slope1},\quad U <= U_{optimal} \\
    & R_t=R_0+R_{slope1}+\frac {U_t-U_{optimal}}{1-U_{optimal}} R_{slope2},\quad U > U_{optimal} \\
    \end{aligned}
    \right.
    $$

  * `R_0` ：初始利率，固定值

  * `U_t` ：资金利用率

  * `U_optimal`：最佳资金利用率，固定值

  * `R_slope1`：变化率，固定值

  * `R_slope2`：变化率，固定值

* 以上参数数值可参考 AAVE 官网 [borrow-interest-rate](https://docs.aave.com/risk/liquidity-risk/borrow-interest-rate)，最新数据可从 AAVE [详情页](https://app.aave.com/reserve-overview/?underlyingAsset=0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2&marketName=proto_mainnet_v3) 最下方 `Interest rate model` 的`INTEREST RATE STRATEGY`按钮获取，可以看到最新值如下图所示：

  * ![img2024-09-19 08.21.52](/Users/zhouwei/Desktop/contract/aave/img/img2024-09-19 08.21.52.png)

  * 可以得到最新数值如下，注意需要将返回值除以 10^27
    * `R_0` ：初始利率，getBaseVariableBorrowRate，为 0
    * `U_optimal`：最佳资金利用率，getOptimalUsageRatio，为 0.9
    * `R_slope1`：变化率，getVariableRateSlope1，为 0.027
    * `R_slope2`：变化率，getVariableRateSlope2，为 0.8

* 将参数带入公式：

  * $$
    & R_t=R_0+\frac {U_t}{U_{optimal}} R_{slope1}\\
    & =0+\frac {0.8998} {0.9}0.027\\
    & =0.026994
    $$

* 由于利率计算是复利，而上述计算结果 `R_t` 为单利，所以出现数据不一致，可用以下公式计算复利：

  * $$
    & ActualAPY=(1+TheoreticalAPY/secsperyear)^{secsperyear}−1 \\
    & =(1+0.026994/31536000)^{31536000}-1 \\
    & =0.02736
    $$

  * **secsperyear**：每年秒数，为 31536000

  * **TheoreticalAPY**：单利利率

* 可以看出结果与**借贷利率 2**.74% 基本一致：

  * ![img2024-09-19 08.48.42](/Users/zhouwei/Desktop/contract/aave/img/:Users:zhouwei:Desktop:img2024-09-19 08.48.42.png)

### 存款利率

* 我们可以从资产详情页面看到存款利率为 2.09%
  * ![img2024-09-19 08.49.34](/Users/zhouwei/Desktop/contract/aave/img/img2024-09-19 08.49.34.png)

* 还可以看到另一个参数 **Reserve factor**

  * ![img2024-09-19 09.19.13](/Users/zhouwei/Desktop/contract/aave/img/:Users:zhouwei:Desktop:img2024-09-19 09.19.13.png)

  * 是将借款利息中的 15% 交给协议

* 现在来看一下存款利率计算公式：

  * $$
    & LR_t=\bar R_tU_t \\
    & \bar R_t=\frac {VariableDebt} {TotalDebt}R_{variable}+\frac {StableDebt} {TotalDebt}R_{stable}
    $$

    

  * LR_t：存款利率
  * R_t：浮动利率贷款与固定利率贷款的利率加权平均值(以占总贷款数量的比值为权数)
  * U_t：资金利用率

* 由于 AAVE 固定利率已不支持固定利率，所以新公式为：

  * $$
    & LR_t=R_{variable}U_t \\
    & \bar R_t=R_{variable}
    $$

* 根据公式计算可得：

  * $$
    LR_t=R_{variable}U_t \\
    =0.0274*0.8998 \\
    =0.02465
    $$

* 然后在减去 **Reserve factor**，可得最终存款利息：

  * $$
    rate=0.02465*(1-0.15)=0.0209
    $$

* 可以看出计算结果与页面展示基本一致

## 固定利率

* 计算公式：

  * $$
    R_{base}=
    \left\{
    \begin{aligned}
    & R_t=R_0+\frac {U_t}{U_{optimal}} R_{slope1},\qquad U <= U_{optimal} \\
    & R_t=R_0+R_{slope1}+\frac {U_t-U_{optimal}}{1-U_{optimal}} R_{slope2},\quad U > U_{optimal} \\
    \end{aligned}
    \right.\\
    R=
    \left\{
    \begin{aligned}
    & R_{base},\qquad ratio<O_{ratio} \\
    & R_{base}+\frac {ratio-O_{ratio}}{1-O_{ratio}}offfset,\quad ratio>=O_{ratio} \\
    \end{aligned}
    \right.
    $$

  * **R_base** 为浮动利率，**R** 为固定利率

  * ***ratio*** 固定利率负债与总负债之比，

  * **O_ratio** 最佳比例

  * **offset** 标准偏差





# 存款取款

* 现在需要新增一个参数，cumulated liquidity index，计算公式如下：

  * $$
    LI_t=(LR_t \Delta year+1)LI_{t-1}
    $$

    * LI_t：cumulated liquidity index
    * LR_t：平均存款利率
    * year：时间间隔

* 假设在  `t-1` 时刻用户存入 b 个代币，那么记录的存入数量为：

  * $$
    \frac {b} {LI_{t-1}}
    $$

* 用户在  `t` 时刻取出时，可以获得的代币数量：

  * $$
    \frac {b} {LI_{t-1}}LI_t
    $$

    

# 借款还款

* 计算方式与存款取款类似，公式也是用同一个，其中  `LR_t` 为平均借款利率  

* 假设在  `t-1` 时刻用户借 b 个代币，那么记录的借款数量为：

  * $$
    \frac {b} {LI_{t-1}}
    $$

* 用户在  `t` 时刻还款，需要代币数量：

  * $$
    \frac {b} {LI_{t-1}}LI_t
    $$

* 其中差值就是 利息





# 清算

* 现在需要几个新增参数：
  * `Max LTV` 贷出资产价值与质押品价值的最大比值
  * `Liquidation threshold` 清算阈值，如果抵押资产与贷出资产的价值比率低于此值，会触发清算
  * `Liquidation penalty` 资产被清算时，清算人使用被清算人贷出资产购买抵押品的折扣

* 下面是 WETH 几个参数的具体值：

  * ![img2024-09-19 16.49.04](/Users/zhouwei/Desktop/contract/aave/img/:Users:zhouwei:Desktop:img2024-09-19 16.49.04.png)

  * `Max LTV` ：假设存入了 1WETH，那么最多贷出 0.805WETH 的等价资产
  * `Liquidation threshold` ：假设存入 1WETH，贷出 0.5WETH，由于 WETH 的下跌，导致抵押资产的价值小于 **0.5/0.83=0.6024** 时，将触发清算
  * `Liquidation penalty` ：假设我清算价值 1ETH 资产，那么我将获得价值 1.05 ETH 的资产，用于偿还价值 1ETH 债务

* 现在再新增一个参数 `Health factor`，此数值一旦低于 1 ，将触发清算，公式如下：

  * $$
    H_f= \frac {\sum (Collateral_i \;in \;ETH×Liquidation \;Threshold_i)}{Total \;Borrows \;in \;ETH}
    $$

  * **H_f**：Health factor，健康因子

  * **Collateral_i in ETH**：资产 i 以 ETH计价的价值

  * **Liquidation Threshold_i**：资产 i 的清算阈值

  * **Total Borrows in ETH**：用户以 ETH 计价借出的总资产

* 资产清算规则：

  * 当贷款的`Health factor`小于 1 时，清算启动，但此时清算人只能最多清算 50% 的贷款
  * 当贷款的`Health factor`小于 0.95 时，清算人可以清算贷款人 100% 的贷款

* 举例说明一个完整的清算过程：

  * 假设当前 WETH 价格为 2000USDT，现在存入 2WETH，3000USDT，价值 3.5ETH 或 7000USDT

    * WETH:

      * ![img2024-09-19 16.49.04](/Users/zhouwei/Desktop/contract/aave/img/:Users:zhouwei:Desktop:img2024-09-19 16.49.04.png)

      * USDT：
        * ![img2024-09-19 18.38.56](/Users/zhouwei/Desktop/contract/aave/img/img2024-09-19 18.38.56.png)

  * 按照最大可借贷金额为

    * $$
      2*0.805*2000+3000*0.75=5470USDT \\
      5470/2000=2.735ETH
      $$

  * 贷出 **2.735ETH**

    * 健康度`H_f`为：

      * $$
        H_f=\frac {2*0.83+3000/2000*0.78} {2.735}=1.034
        $$

      

    * 假设现在 ETH 价格上涨 10%，为 2200，此时：

      * $$
        H_f=\frac {2*0.83+3000/2200*0.78} {2.735}=0.9958
        $$

    * 现在需要清算借贷人 50% 的资产，也就是 **2.735/2=1.3725ETH**。现在清算人需要向 AAVE 偿还 **1.3725ETH**，并得到 **1.3725*(1+0.05)=1.4411ETH**，此次清算利润为 **1.3725*0.05=0.0686ETH**

    * 清算完后：

      * 资产：

        * $$
          & 2-1.4411/2=1.2794ETH \\
          & 3000-1.4411/2*2200=1414.79USDT
          $$

        

      * 借贷金额 **1.3725ETH**

      * 新健康度为：

        * $$
          H_f=\frac {1.2794*0.83+1414.79/2200*0.78} {1.3725}=1.139
          $$

        

  * 贷出 **5470USDT**:

    * 健康度`H_f`为：

      * $$
        H_f=\frac {2*0.83+3000/2000*0.78} {5470/2000}=1.034
        $$

    * 假设现在 ETH 价格下跌 30%，为 1700，此时：

      * $$
        H_f=\frac {2*0.83+3000/1700*0.78} {5470/1700}=0.9436
        $$

    * 现在需要清算借贷人 100% 的资产，也就是 **5470USDT**。现在清算人需要向 AAVE 偿还 **5470USDT**，并得到 **5470*(1+0.05)=5743.5USDT**，此次清算利润为 **5470*0.05=273.5USDT**

    * 清算完后：

      * 资产：

        * $$
          & 2-5743.5/2/1700=0.3107ETH \\
          & 3000-5743.5/2=128.25USDT
          $$

        

    * 假设现在 ETH 价格下跌 20%，为 1800，此时：

      * $$
        H_f=\frac {2*0.83+3000/1800*0.78} {5470/1800}=0.9740
        $$

    * 现在需要清算借贷人 50% 的资产，也就是 **5470/2=2735USDT**。现在清算人需要向 AAVE 偿还 **2735USDT**，并得到 **2735*(1+0.05)=2871.75USDT**，此次清算利润为 **2735*0.05=136.75USDT**

    * 清算完后：

      * 资产：

        * $$
          & 2-2735/2/1800=1.24ETH \\
          & 3000-2735/2=1632.5USDT
          $$

        

      * 借贷金额 **2735USDT**

      * 新健康度为：

        * $$
          H_f=\frac {1.24*0.83+1632.5/1800*0.78} {2735/1800}=1.14
          $$

        

# V3

* potal：允许不同区块链网络上的 AAVE V3 的流动性代币 atoken 进行转移，在源网络上燃烧 aTokens，同时在目标网络上铸造 aTokens。
* 效率模式（E-Mode）：当质押物和借出资产在价格上有相关性，甚至是同一种底层资产的衍生物时，stETH, sETH, alETH，以及 USDT，USDC，DAI等，用户拥有更高的质押系数，例如；稳定币 97% LTV、98% 清算阈值， 2% 清算奖金。最多可支持 255 种币种。
  * ![img2024-09-20 08.15.38](/Users/zhouwei/Desktop/contract/aave/img/:Users:zhouwei:Desktop:img2024-09-20 08.15.38.png)
* 隔离模式：当管理者（可以是DAO）将抵押资产设置为隔离模式后这一抵押资产就不能用作其他资产的抵押物，并且这一抵押资产只能借贷稳定币。这一模式主要针对的是一些不是主流币的次级项目，由于次级项目具有较大的波动性，所以aave不允许将多个次级资产同时作为隔离模式的抵押物。AAVE V2 中支持抵押的资产都是主流资产，但是用户受众可能会有很多次级资产，V2 中这部分资金目前是没有被利用起来的，如果将次级资产和主流资产同等对待作为抵押资产，AAVE 将会承担更大的风险，与时 V3 更新了这个隔离模式，用户只能将一种次级资产作为隔离模式的抵押资产，无法和主流资产混合，两者之间必须隔离。如果你使用了一个次级资产作为抵押，则其他资产不能作为抵押品，只能当做存款。如果使用了其他资产作为抵押物，则任意一个次级资产都无法作为抵押品。
* 稳定利率：todo



# 其他

* 其他借贷协议，如 Maker，Compound的原理基本和 AAVE 一致。如 Compound 利率计算和 AAVE 有些不同，Maker 清算使用荷兰拍卖。

# 参考文章

* [Aave调研报告 - s3cunda's blog](https://s3cunda.github.io/2022/03/13/AAVE调研报告.html#抵押借贷逻辑)
* [AAVE交互指南 | Wong's Blog](https://blog.wssh.trade/posts/aave-interactive/)
* [深入解析AAVE智能合约:存款 | Wong's Blog](https://blog.wssh.trade/posts/aave-contract-part1/)
* [DeFi 概念：借贷 - smlXL | smlXL](https://medium.com/smlxl/defi-lending-concepts-part-1-lending-and-borrowing-f646d6a08dd7)
* [un.Block 区块链周报 #23：Secret Network、DeFi 借贷之清算 - un.Block 周报](https://unblock256.substack.com/p/unblock-23secret-networkdefi-)
* [AAVE V2 学习笔记 - ripwu's blog](https://godorz.info/2021/10/aave-v2/)
* [Aave V2分析(一) | 登链社区 | 区块链技术社区](https://learnblockchain.cn/article/7914)
* [Dapp-Learning/defi/Aave/whitepaper/AAVE_V3_Techpaper.md at main · Dapp-Learning-DAO/Dapp-Learning](https://github.com/Dapp-Learning-DAO/Dapp-Learning/blob/main/defi/Aave/whitepaper/AAVE_V3_Techpaper.md)