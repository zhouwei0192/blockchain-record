* 本篇文章以 Uniswap V2为例子，讲解其核心概念，并举例说明，包括：
  * AMM自动做市商
  * 流动性计算
  * 手续费计算
  * 滑点
  * 无常损失
  * 价格预言机
  * **闪电贷 todo**

# AMM自动做市商

* Uniswap 链上交易的基础叫恒定乘积做市商，它的核心就是一个公式： 

  * $$
    X*Y=K
    $$

  * 恒定乘积做市商的原理就是，在交易前后保证 K 值不变，其中 X和 Y 就是两种交易资产的数目，K为一个不变的数值。

  * 在交易时发生的公式如下：不考虑手续费

    * $$
      （x-\Delta	x）*(y+\Delta	y)=k
      $$

    * 举例说明，假设 tokenX=3，tokenY=100，K=300，我用 50个tokenY 去交易，经过如下计算，可以得到 1个tokenX

      * ```
        （3-tokenX）*（100+50）= 300
        							 tokenX = 1
        ```



# 流动性

* 在 Uniswap 中流动性就是恒定乘积做市商公式中 X和Y的数量，添加流动性就是增加X和Y的数量，移除流动性就是减少 X和Y的数量

* 在 uniswap v2中流动性计算代码如下：

  * 添加流动性：

    * ```solidity
      if (_totalSupply == 0) { 
          liquidity = Math.sqrt(amount0.mul(amount1)).sub(MINIMUM_LIQUIDITY);
      } else { 
          liquidity = Math.min(amount0.mul(_totalSupply) / _reserve0, amount1.mul(_totalSupply) / _reserve1);
      }
      ```

  * 移除流动性：

    * ```solidity
      uint _totalSupply = totalSupply; 
      amount0 = liquidity.mul(balance0) / _totalSupply; 
      amount1 = liquidity.mul(balance1) / _totalSupply; 
      ```

* **在下面我会举例说明以上公式如何计算**

  * 第一次添加流动性（初次添加，流动性 lp=0），amount0 = 4e，amount1 = 400u，那么

    * ```
      lp = sqrt(amount0 * amount1) = 40
      _reserve0 = amount0 = 4
      _reserve1 = amount1 = 400
      k = _reserve0 * _reserve1 = 1600
      totalSupply = lp = 40
      ```

  * 第二次添加流动性（非初次添加，流动性 lp>0），amount0 = 1e，amount1 = 100u，那么

    * ```
      lp = min(amount0/_reserve0*totalSupply, amount1/_reserve1*totalSupply) 
      	 = min(1/4*40, 100/400*40) = min(10, 10) = 10
      totalSupply = totalSupply + lp = 40 + 10 = 50
      _reserve0 = reserve0 + amount0 = 4 + 1 = 5
      _reserve1 = reserve1 + amount1 = 400 + 100 = 500
      ```

* **我们现在思考两个问题：**

  1. 我以异常的比例添加流动性会出现什么情况，比如现在池子价格为（1e，100u），高价添加（1e，160u）和低价添加（1e，80u）会出现什么情况呢？
  2. 计算 lp 为什么取最小值而不是最大值？

* 我们现在看第一个问题，异常的比例添加流动性会出现什么情况？

  * 高价添加（1e，160u）：

    * amount0 = 1e，amount1 = 160u，totalSupply = 50， _reserve0 = 5，_reserve1 = 500

    * ```
      lp = min(1/5*50, 160/500*50) = min(10, 16) = 10
      totalSupply = 50 + 10 = 60
      _reserve0 = 5 + 1 = 6
      _reserve1 = 500 + 160 = 660
      ```

    * 此时移除流动性，此人 lp = 10

      * ```
        amount0 = lp / totalSupply * _reserve0 = 10 / 60 * 6 = 1
        amount1 = lp / totalSupply * _reserve1 = 10 / 60 * 660 = 110
        ```

        * 可以看到移除流动性获得了 1e 和 110u，而其他 50u 作为惩罚平分给了其他lp提供者，所以在提供流动性时，需要合理范围内，否则就要承受损失

  * 低价添加（1e，80u）：

    * amount0 = 1e，amount1 = 80u，totalSupply = 50， _reserve0 = 5，_reserve1 = 500

    * ```
      lp = min(1/5*50, 80/500*50) = min(10, 8) = 8
      totalSupply = 50 + 8 = 58
      _reserve0 = 5 + 1 = 6
      _reserve1 = 500 + 80 = 580
      ```

    * 此时移除流动性

      * ```
        amount0 = lp / totalSupply * _reserve0 = 8 / 58 * 6 = 0.8275
        amount1 = lp / totalSupply * _reserve1 = 8 / 58 * 580 = 80
        ```

        * 可以看到移除流动性获得了 0.8275e 和 80u，而减少的 0.1724e 作为惩罚平分给了其他lp提供者，所以在提供流动性时，需要合理范围内

* 现在看第二个问题，lp 为什么取最小值而不是最大值，因为取最大值会破坏整个系统，额外获取不属于自己的利益

  * 高价添加（1e，160u）：

    * amount0 = 1e，amount1 = 160u，totalSupply = 50， _reserve0 = 5，_reserve1 = 500

      * ```
        lp = max(1/5*50, 160/500*50) = max(10, 16) = 16
        totalSupply = 50 + 16 = 66
        _reserve0 = 5 + 1 = 6
        _reserve1 = 500 + 160 = 660
        ```

      * 此时移除流动性

        * ```
          amount0 = lp / totalSupply * _reserve0 = 16 / 66 * 6 = 1.4545
          amount1 = lp / totalSupply * _reserve1 = 16 / 66 * 660 = 160
          ```

        * 可以看到移除流动性时拿到了 1.4545e 和 160u，额外获得不属于他的 0.4545e，

  * 低价添加（1e，80u）：

    * amount0 = 1e，amount1 = 80u，totalSupply = 50， _reserve0 = 5，_reserve1 = 500

    * ```
      lp = min(1/5*50, 80/500*50) = max(10, 8) = 10
      totalSupply = 50 + 10 = 60
      _reserve0 = 5 + 1 = 6
      _reserve1 = 500 + 80 = 580
      ```

      * 此时移除流动性，

        * ```
          amount0 = lp / totalSupply * _reserve0 = 10 / 60 * 6 = 1
          amount1 = lp / totalSupply * _reserve1 = 10 / 60 * 580 = 96.6666 
          ```

          * 可以看到移除流动性时拿到了 1e 和 96.6666u，额外获得不属于他的 16.6666u，





# 手续费

* 在Uniswap中交易是有手续费，每笔交易为 0.3%，默认全返还给流动性提供者，但是有一个开关可以控制其中1/6，也就是0.05%，是否给项目方。

* 在加入手续费后，新的公式如下所示：

  * $$
    \theta=1-\alpha
    $$

    

  * $$
    （x-\Delta	x）*(y+\theta\Delta	y)=k
    $$

* 现在我会举例说明该公式如何使用：

  * 假设现在有一个流动性池 ETH-USDT，其中池子比例为 ETH：USDT = 5：2000，现在通过两次花费100USDT来买入ETH，其中变量变化如下表格所示：

  * ![截屏2024-09-08 08.52.30](/Users/zhouwei/Desktop/contract/picture/截屏2024-09-08 08.52.30.png)

    * 第一次花费100USDT来买入ETH：

      * ```
        （5-x）*（200+10*0.997）= 5*200
        									   x = 0.2374
        ```

        

    * 第二次花费100USDT来买入ETH：

      * ```
        （4.7626-x）*（2100+100*0.997）= 10001.46
           													 x = 0.2158
        ```

  * 可以看到在 totalSupply（LP总量）不变的情况下，每一次交易，K值在增加，其中增加的就是手续费，这部分手续费会平分给所有流动性提供者，并在移除流动性时实现收益



# 滑点

* 我们以**上面**手续费计算为例子进行说明：
  * 我们知道流动性池 ETH：USDT = 5：2000，所以在此池子中 1ETH=400USDT
  * 而花费 100USDT，应该得到 0.25ETH，但是根据计算我们只得到了 0.2374ETH，其中的差值就是**滑点**，我们在 Uniswap Swap时，会显示的兑换数量是 0.2374 而不是 0.25，滑点为 5.04%

# 无常损失

* 我们还以手续费计算为例子进行说明：**（以下计算忽略手续费）**
  * 我们知道流动性池 ETH * USDT = 5 * 2000，所以在此池子中 1ETH=400USDT，假设小明拥有这个流动性池的20%，那么在此时他拥有 1ETH+400USDT
  * 现在假设ETH价格突然发生了大幅变化：
    * 假设此时 ETH 价格突然上涨400%，此时 1ETH=1600USDT，此时流动性池 ETH * USDT=2.5 * 4000，这时移除流动性，小明将获得 0.5ETH+800USDT=**1600USDT=1ETH**
      * 如果小明当时没有添加流动性池，而是一直持有 1ETH+400USDT，那么此时价值 **2000USDT或1.2ETH**
    * 假设此时 ETH 价格突然下跌75%，此时 1ETH=100USDT，此时流动性池 ETH：USDT = 10：1000，这时移除流动性，小明将获得 2ETH+200USDT=**400USDT=4ETH**
      * 如果小明当时没有添加流动性池，而是一直持有 1ETH+400USDT，那么此时价值 **500USDT或5ETH**
* 可以从上述计算看出无论价格上涨还是下跌，小明都要承受损失，这个就是**无常损失**，所以我们在添加流动性时需要考虑手续费收入能否覆盖代币价格波动带来的无常损失
* 计算公式：r 为价格比值

  * ![截屏2024-09-12 18.16.39](/Users/zhouwei/Desktop/contract/picture/:Users:zhouwei:Library:Application Support:typora-user-images:截屏2024-09-12 18.16.39.png)

  	* 推导过程：https://myself659.github.io/post/blockchain/uniswap-impermanent-loss-formula/





# 价格预言机

* UniswapV2 使用的价格预言机称为 **TWAP（Time-Weighted Average Price）**，即**时间加权平均价格**，该累计价格将是此合约历史上每秒的现货价格之和。在每个区块第一笔交易前记录累计价格实现预言机，累计价格每个价格会以时间权重记录（基于当前区块与上一次更新价格的区块的时间差）。具体可看下图：

  * ![截屏2024-09-10 18.14.00](/Users/zhouwei/Desktop/contract/picture/:Users:zhouwei:Library:Application Support:typora-user-images:截屏2024-09-10 18.14.00.png)
  * ![截屏2024-09-10 20.47.18](/Users/zhouwei/Desktop/contract/picture/:Users:zhouwei:Library:Application Support:typora-user-images:截屏2024-09-10 20.47.18.png)

* 实现 TWAP 官方提供了两种方式：固定时间窗口，滑动时间窗口

  * **固定时间窗口：**假设窗口大小为 30

    * 我们只需取对应时间区间的两个累积价格，然后在除以时间间隔即可
    * 【0，30】：（88.8-0）/（30-0）= 2.96
      * 那么在【30，60】这段时间内，我们得到的价格都为 2.96
    * 【30，60】：（284.8-88.8）/（60-30）= 6.53
      * 那么在【60，90】这段时间内，我们得到的价格都为 6.53

  * **滑动时间窗口：**假设窗口 windowSize=30，分隔时间片段数量 granularity=3，间隔 periodSize=10

    * 我们在任意时间点 time 得到的价格：

      * **当前时间区间** 到 **第 time / periodSize % granularity 个时间区间**的加权平均价格

    * 例如：

      * 当前时间区间：【40，50】

      * 第 time / periodSize % granularity 个时间区间：

        * ```
           time / periodSize % granularity = 45 / 10 % 3 = 1
          ```

        * 也就是第一个时间区间【10，20】，所以可以得到价格为（130.8-24.4）/（40-10）= 3.54

      * 同理，如果当前时间区间为【30，40】，那么对应区间为第0个，为【10，20】，计算可得：

        * （88.8-0）/（30-0）= 2.96

* 能否操控价格？

  * 操纵这个价格会比操纵区块中任意时间点的价格要困难。如果攻击者通过在区块的最后阶段提交一笔交易来操纵价格，其他套利者（发现价格差异后）可以在同一区块中提交另一笔交易来将价格恢复正常。矿工（或者支付了足够gas费用填充整个区块的攻击者）可以在区块的末尾操控价格，但是除非他们同时挖出了下一个区块，否则他们没有特殊的优势可以进行套利。
  * 以太坊的POS选定矿工能否提前知道？提前知道就可以操控价格？

# 闪电贷

