# 项目复盘

---

## 1. 基于价格维度进行商品召回
### 入手点

1. 通过 `order_sku_all` 拉取近两个月内热销商品；
2. 通过 `dwd_backend_day` `order_sku_all` 拉取热销商品日销量和每日价格相关字段；
3. 通过对比价格和销量，确定四个字段用于做价格召回，包括：`predictprice` `discount` `order_promotion_price` `order_coupon_price`。

### 价格字段图解

![价格字段图解](C:\Users\Work-PC\Desktop\2021春招\figures\价格字段图解.jpg)

### 建立中间表

1. `order_promotion_price` 和 `order_coupon_price` 代表每一单的具体价格，考虑到异常值，取每日所有订单的中值，单独构建一张中间表；
2. 考虑到价格是本次召回的核心，因此建立价格和日销量的中间表。

### 初步召回策略

1. 近两个月内 `med_promotion_price` 和 `med_coupon_price` 为最大值的商品，即促销和发券力度达到最大；
2. 近两个月内 `predict_price` 和 `discount` 为最小值的商品，即价格达到最低点。

### 存在问题

1. 因为**平台大促（如双十一、双十二）等活动**的存在，部分优惠商品因促销力度略低于大促时段而未被召回；
2. 有些商品可能只有**一个或者两个**价格，这会导致无论何时都会被召回；
3. 召回结果只包括 `itemid` ，无法知晓某一商品是通过何种策略进行召回，以及无法对召回商品进行排序。

### 优化方案

1. 召回促销力度最大和第二大的商品，采用 `dense_rank()` 实现；
2. 统计每个商品价格数目，仅针对价格数目大于2的商品进行召回；
3. 建立推荐系统点击率中间表，最终召回的商品需要同时包括商品名称、品牌、3日点击率、7日点击率、日销量等信息；
4. 根据销量进行筛选，去除销量极低的商品；
5. 引入实时数仓数据表 `ads_item_rec_price_realtime_day` ，该表基于 `promomtion_item_new` 中的 `promosellingprice` 字段进行召回；`promomtion_item_new` 表中的 `promotiontype` 的含义见下表：

| Promotion type | 含义                            |
| -------------- | ------------------------------- |
| banner         | 类似于predict_price，越低越优惠 |
| deduction      | 价格减免，越大越优惠            |
| preSale        | 类似于origin_price，越低越优惠  |
| price          | 类似于sell_price，越低越优惠    |

6. 没有必要考虑是否是长期销售的商品，只要价格数量大于等于3个即满足初步召回筛选条件。

 ### 价格召回所用表

![价格召回所用表](C:\Users\Work-PC\Desktop\2021春招\figures/价格召回所用表.jpg)

### 踩到的坑

1. `dwd_backend_day` 和 `order_sku_all` 中的 `itemid` 的数据类型分别为 `string` 和 `int`，在做连接时可能会出现匹配不到的问题，导致漏掉很多数据；
2. 在连接 `dwd_backend_day` 和 `dws_item_med_pro_cou_price_day` 时，应采用 `full join`，取到所有有效的价格字段，哪个表有 `itemid` 就用哪个表的 `itemid` （`case when` 语句实现）。

### 反思

1. 建立中间表时，需要考虑表的复用性，初步建立的点击率表只针对每日价格召回商品，无法用于其它业务；
2. 需要多看召回结果，寻找bad cases，线上工具网址：http://m.wanwustore.cn//o/uc/m/uiww/testfe?command=recall&userId=15029792232323&type=bigDiscountItem&page=home；
3. 引入实时数仓表时，未过多关注表的结构，实际上，不同 `promotiontype` 对应的 `prosellingprice` 是存在很大差别的，有的是越大越优惠，而另外的则是越低越优惠。

### 实时价格召回

+ 实时价格召回使用 `promotion_item_new` ，需要关注的字段包括如下：

| 字段名称          | 含义                          |
| ----------------- | ----------------------------- |
| promotiontype     | 促销类型，如 rebate、banner等 |
| promosellingprice | 促销价格                      |
| effecttime        | 促销策略的生效时间            |
| expiretime        | 促销策略的失效时间            |

1. 通过使用如下代码，获取当日内有效的促销价格

```sql
cast(effecttime as date) <= '$dt'
and cast(expiretime as date) >= '$dt'
```

## 2. 商品价格特征构建

### 入手点

1. 拉取近两个月热销商品的**价格、3日点击率、7日点击率、销量信息**；
2. 寻找与点击率、销量关系密切的价格字段；
3. 上网查阅资料，~~是否有类似案例~~（未查到相关案例）；
4. 确定结果呈现形式：itemid + 特征A + 特征B + 特征B + ...。

### 商品价格特征

- 通过分析价格、点击率、销量数据，确定如下特征

1. 二值化特征

> isHighPromotion_60days：是否为60天内**最高、次高**的 `med_order_promotion_price`
>
> isHighCoupon_60days：是否为60天内**最高、次高**的 `med_order_coupon_price`
>
> isLowPredict_60days：是否为60天内**最低、次低**的 `predict_price`
>
> isLowDiscount_60days：是否为60天内**最低、次低**的 `discount`
>
> isHighPromotion_30days：是否为30天内**最高、次高**的 `med_order_promotion_price`
>
> isHighCoupon_30days：是否为30天内**最高、次高**的 `med_order_coupon_price`
>
> isLowPredict_30days：是否为30天内**最低、次低**的 `predict_price`
>
> isLowDiscount_30days：是否为30天内**最低、次低**的 `discount`
>
> isHighPromotionAppear_3days：**最高、次高**的 `med_order_promotion_price`是否在3天内出现
>
> isHighCouponAppear_3days：**最高、次高**的 `med_order_coupon_price`是否在3天内出现

1. 统计类特征

> predict_price_percentiles：`predict_price` 分位数（在某一商品历史价格的排名）
>
> discount_percentiles：`discount` 分位数（在某一商品历史价格的排名）
>
> med_promotion_price_percentiles：`med_promotion_price` 分位数（在某一商品历史价格的排名）
>
> med_coupon_price_percentiles：`med_coupon_price` 分位数（在某一商品历史价格的排名）

### 所用中间表

![商品价格特征中间表](C:\Users\Work-PC\Desktop\2021春招\figures\/商品价格特征中间表.jpg?lastModify=1609748638)

### 优化过程

1. 第一次构建过程只统计商品是否是60天的低价或者高促销，这会导致特征覆盖率过低；
2. 加入统计30天内商品是否是60天的低价或者高促销，因为在短时间内商品处在低价或者高促销的概率更大；
3. 加入中间表 `dws_item_price_binary_features_day` ，记录商品低价、高促销的具体值以及对应出现日期。

### 后续策略

1. 考虑进行特征交叉；
2. 优化SQL语句；
3. 拉取每一个字段的前100个商品的具体信息；
4. 建立价格、销量、优惠中间表 `dws_item_price_sales_promotion_day`，方便后续做特征交叉，具体字段包括：

| 字段名称                   | 含义                                             |
| -------------------------- | ------------------------------------------------ |
| itemid                     | 商品id                                           |
| origin_price               | 商品原始售价                                     |
| sell_price                 | 商品销售价格（如打折后）                         |
| predict_price              | 商品预测售价（考虑各种优惠策略）                 |
| discount                   | 折扣数                                           |
| sales_1day                 | 昨天销量                                         |
| sales_3days                | 3天总销量                                        |
| sales_7days                | 7天总销量                                        |
| sales_30days               | 30天总销量                                       |
| sales_60days               | 60天总销量                                       |
| med_promotion_price_7days  | 7天 order_promotion_price 中值                   |
| med_promotion_price_30days | 30天 order_promotion_price 中值                  |
| med_coupon_price_7days     | 7天 order_coupon_price 中值                      |
| med_coupon_price_30days    | 30天 order_coupon_price 中值                     |
| banner_price               | 实时价格表 promotion_item_new 中的 predict_price |

## 3. 用户、商品特征交叉（基础特征）

### 入手点

1. 拉取用户每日点击商品的价格；
2. 统计不同时间尺度用户点击商品价格的最大值、最小值、均值以及中位数。

### 所用中间表

![用户点击商品价格基础特征](C:\Users\Work-PC\Desktop\2021春招\figures\用户点击商品价格基础特征.jpg)

## 4. 用户、商品特征交叉（分桶特征） 

### 入手点

1. 拉取用户点击商品的价格，点击行为包括商详页点击和开团页点击，价格具体包括 `origin_price` 、 `sell_price` 、 `predict_price` ，并应用 `matplotlib` 可视化价格分布；
2. 基于分布，对商品根据价格进行分桶，阈值设定为：0-50、50-100、100-200、200-500、500+；

### 建立中间表

1. 首先从 `dws_user_event_type_day` 拉取用户点击行为，从 `dws_item_price_sales_promotion_day` 拉取商品价格；
2. 基于分桶设定，计算用户每日在各个桶的点击数和点击比例，构建表 `dws_user_click_item_price_bucket_day` ；
3. 计算1、3、7、30、60天的用户每日在各个桶的点击数和点击比例，构建表 `dws_user_click_item_price_bucket_multi_time_day` 。

### 所用中间表

![用户、商品特征交叉所用表](C:\Users\Work-PC\Desktop\2021春招\figures\用户、商品特征交叉所用表.jpg)

### 用户、商品交叉特征

**price**

+ origin_price
+ sell_price
+ predict_price
+ banner_price

**type**

+ click_num
+ click_ratio

**bucket**

+ 0-50
+ 50-100
+ 100-200
+ 200-500
+ 500+

**time_span**

+ 1day
+ 3days
+ 7days
+ 30days
+ 60days

## 5. 基于用户、商品交叉特征的个性化推荐

### 入手点

1. 首先需要判断不同用户点击商品价格的特征（如中位数是否存在个体上差异）；
2. 同时结合之前构建基础交叉特征和分桶交叉特征设定召回策略。

### 个性化推荐策略

1. 基于 `predict_price` ，通过之前的工作，可以发现其与商品销量存在较大关联；

2. 拉出每个用户点击商品价格的中位数，初步设定 `time_span` 为7天；

3. 针对每个用户，计算每日价格召回商品的价格与其中位数的差值，并进行最大最小归一化，得到 `diff_norm` ；

4. 针对每个用户，判断价格召回商品的价格是否在其点击最多的 bucket 内，得到 `isInBucket` ，在为1，不在为0；

5. 同时沿用之前根据点击率和销量所得到的 `click_score`；

6. 对于每一用户，每一个价格召回商品可以计算一个对应的得分，计算公式为：
   $$
   item\_score = (1 - diff\_norm) * 0.4 + isInBucket * 0.4 + click\_score * 0.2
   $$

### 优化过程

1. 通过计算 `banner_price` 的覆盖率为 73%，优于 `predict_price` 的覆盖率 68%，因此将二者进行替换。

### 所用中间表

![用户、价格交叉特征的个性化推荐](C:\Users\Work-PC\Desktop\2021春招\figures\用户、价格交叉特征的个性化推荐.jpg)

### 最后结果格式

1. 针对每一个 `userid` ，将价格召回商品按照 `item_score` 进行降序排列，使用`collect` 和 `concat_ws` 对 `itemid` 进行收集和使用逗号进行合并，并写入 Redis。

### 现存问题

1. 当前召回覆盖率较低，会将点击率摊平（对于那些不在最终结果的 `userid` ，会出现停留，但没有点击）；
2. 当前有价格信息的商品仅占所有商品的 2/3 左右，导致不能完整获取用户点击商品的信息；
3. 价格召回商品是否有效、合理有待验证。

### 解决方案

1. 将用户点击商品的信息的时间跨度扩展到 30天 ，以获取更多 `userid` ；
2. 补全价格信息可尝试使用 `redshift` 中的 `promotion_item_new` ;
3. 需要进一步确认价格召回的的商品是否可靠。

## 6. 淘宝爬虫数据解析（基于 Scala 写 UDF）

### 入手点

1. 每日的爬虫数据存放在 `MySQL` 中；  
2. 商品信息存放在 `item` 表中，该表位于 `redshift` ；
3. 编写 UDF 解析淘宝爬虫json数据。

### UDF 初版（TaobaoSpiderParser）代码逻辑

1. 使用 `JsonHelper.getJsonObj` 读取json数据；

2. 根据 `key` 逐层进入到 `groupProps`;

3. 解析 `groupProps`时，因为字段为中文，需要使用 `asOpt` 将数据转换为 `Array[JsObject]` ；

4. 采用 `foreach` 读 `itemList` 中的每个 `item` ;

5. 每个 `item` 继续采用 `foreach` 读 `item.fields` ，当 `f._1` 不为 `基本信息` 直接返回空串；

6. `f._2` 为 values ，继续采用 `foreach` 读；

7. key 和 value 之间采用 `\u0001` 连接，不同的 kv 对采用 `\u0002` 连接；

8. 相关代码如下：

   ```scala
   if (null != itemList) {
     itemList.foreach(item => {
       item.fields.foreach(f => {
         if (!f._1.equals("基本信息")) {
           return ""
         }
         val values = f._2.asOpt[Array[JsObject]].getOrElse(null)
         values.foreach(v => {
           v.fields.foreach(l => {
             val k = l._1.trim
             val vv = l._2.toString().replaceAll("\"", "").trim
             val res_val = k+"\u0001"+vv
             resultList.append(res_val)
           })
         })
       })
     })
   }
   ```

### 初版代码存在问题

1. 传入参数写死，只能解析 `item_prop` ；

### UDF 新版（GeneralTaobaoSpiderParser）

1. 新引入参数 `targetKeys` ，允许用户自定义需要提取的字段；

2. 将 value 传入结果集时，对其类型进行判断，代码如下：

   ```scala
   if (tmpJson.getClass.getSimpleName == "JsString") {
   	val tmpKey = tmpJson.asOpt[String].getOrElse("")
   	resultList.append(tmpKey.trim)
   } else if (tmpJson.getClass.getSimpleName == "JsNumber") {
   	val tmpKey = tmpJson.toString
   	resultList.append(tmpKey.trim)
   }
   ```

### SQL 语句编写

1. 因 `item_props` 包含多个 value，因此采用 `lateral view explode` 结合 `split` 提取数据；
2. 其它字段正常读取;
3. 拆分 key 和 value 采用 `split` ，分隔符在 SparkSQL 中为 `\u001` ，取 `[0]` 和 `[1]` 分别得到 key 和 value。

### 后续步骤

1. 针对拆分出的数据进行建表（DWD层），并配置任务，使其每日更新且可以通过Hue查询；
2. 针对DWD层的表进行二次处理，包括拆分，根据映射规则对数据规范化，然后二次拆分（DWS层）。

### 爬虫数据规范化处理思路

1. 统计每个 `prop_key` 在其对应二级分类的**覆盖率**，以此为依据挑选有价值的 `prop_key`；
2. 针对 `prop_value` 进行规范化。

## 7. 提取点击数据及邻近非点击数据（基于 Scala 写 UDF）

### 入手点

1. 明确输入、输出数据（输入数据、输出数据：多条数据拼成一条）；
2. 明确传入参数，是否需要写死，或者允许自定义窗口大小。

### SQL 部分

1. 针对数据进行去重，确保同一时刻只存在一个点击/非点击数据；
2. 将 `itemid` 、 `click` 、 `index`  、 `timestamp` 使用 `concat` 进行连接，通过 `collect` 收集用户在一次操作过程中的所有点击、非点击行为；
3. 使用 `concat_ws` 连接一次用户的所有行为；
4. 针对 UDF 输出的数据，使用 `lateral view explode` 进行炸裂，并使用 `split` 进行拆分；（PS：SparkSQL 的下标从0开始）。

### Scala 部分

1. 传入数据和窗口上下限；
2. 找到点击数据之后，再进行非点击数据的寻找，核心代码如下：

```scala
val isclick = list(1).toInt
if (isclick == 1) {

  // 点击数据加入结果集
  results += fields(i)

  for (j <- (i - preceding) to (i + following)) {

    breakable{
      if (j == i || j < 0 || j > (n - 1)) {
        break
      }

      val tmp = fields(j).split("-").toList

      // 跳过点击数据
      val isclick_tmp = tmp(1).toInt
      if (isclick_tmp == 1) {
        break
      }

      // 非点击数据加入结果集
      results += fields(j)
    }
  }
}
```

3. 对于点击/非点击数据，采用 `:` 进行连接。

### 踩到的坑

1. 某一次点击行为，可能在对应时间内存在非点击数据，通过 `group by` 并取 `max(isclick)` 可解决该问题；
2. 对于下标的使用务必严谨，谨防下标越界的问题。

## 8. 种草社区浏览转化需求

### 需求说明

```
为监控社区种草内容对用户购买行为的作用，提出以下数据需求：

一、商详页进入种草页点击情况

1）（有种草的商品的）商详页pv，uv；

2） 商品种草页的 pv，uv；

uv定义：1个用户浏览1个商品算1（uid+itemid）

【计算1个点击率】

二、浏览种草行为转化为加车、下单行为的情况

【1】 种草页uv（各种途径进入种草页都算：搜索-搜种草；种草tab-各feed流；商详页-大家的种草列表）（uid）

【2】 （【1】这个用户池子里）24小时内 将（浏览过的种草里的任一关联商品）加入购物车的人数（uid）

【3】 （【1】这个用户池子里）24小时内 购买（浏览过的种草里的任一关联商品），生成订单的人数（uid）

【可视化 1个漏斗图】
```

### 入手点

1. 明确需要哪些表： `analysis_ww_ugc_item` 商品种草信息表， `order_sku_all` 用户订单信息表， `dwd_user_event_day` 用户行为表；
2. 确定埋点 key ，如下：

| key  | 含义                       |
| ---- | -------------------------- |
| 1036 | 商详点击                   |
| 1044 | 种草点击                   |
| 1051 | 加车点击                   |
| 1315 | 种草feed点击（种草的商品） |
| 1169 | feed内页点击（种草的商品） |

### SQL 代码逻辑

1. 通过 `analysis_ww_ugc_item` 拉取 `itemid` 和对应的 `ugcid`；
2. 通过 `order_sku_all` 拉取 `userid` 和购买的 `itemid`；
3. 1036 为进入商详页的埋点，通过该埋点可拉取对应的 `itemid` ，与带有 `ugcid`的 `itemid` 进行 `join` 操作并进行  `count(1) `  即可得到商详页 pv，通过  `count(distinct (concat_ws('-', userid, itemid)))` 可得到商详页 uv；
4. 类似的，将埋点改为 `1044` 即可改为种草页 pv、uv；
5. 通过  `count(distinct (concat_ws('-', userid, itemid)))` 从 `1315` 和 `1169` 提取的数据可以得到种草页uv；
6. 将种草页商品和加车商品连接并统计可以得到加车 uv，加车的埋点为1051;
7. 在上述条件上，继续和购买商品连接可以得到购买 uv。

### 图解需求

![种草需求](C:\Users\Work-PC\Desktop\2021春招\figures\种草需求.jpg)

### 踩到的坑

1. Transform 部分的代码为顺序执行，可先将需要的数据一次性提取，如提取1036、1044、1051埋点对应数据；
2. 在计算种草页 uv 时， `itemid` 应该与 `ugcid` 进行连接，在此需要事先连接每一个埋点的具体含义。