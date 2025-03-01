+++
title = 'Jane Street 2020 QTC'
date = 2020-01-13
draft = false
+++

一直说“条条专业通金融”，每年贵系都有不少人去做金融相关行业，许多前辈也成立了金融初创公司回校招人。不过因为抗拒心理，我很抵触为了财富之类去做自己不喜欢的事业，即使自己对这些行业的具体内容还一无所知。正巧在期中季结束后的一天，辅导员在年级群转发了 Jane Street QTC 的活动介绍。如果能借此机会去一家非常专业的公司深入了解量化交易，很有可能会给我带来非常大的受获。**事实也的确如此。**

## 申请

申请分三（两）个部分，英文简历、一个游戏以及简短的电话聊天。简历会有初步筛选，大概两周后通知我进入第二轮，要做一道博弈论的题目（Blotto）；提交之后又过了两三周告知我入围，并准备安排一场电话。我以为会是电话面试还看了一会儿面经，没想到只是随意地聊了十分钟为什么想来这个项目，有什么期待之类的 :) 然后就坐等 HR 帮忙安排行程之类的了。

## 出行

机酒都是公司安排（公费出游 XD），住的地方在中环，离 Jane Street 大概十分钟路程。早上八点半到，吃完早饭聊一会天大概九点开始活动，五六点下班。在公司里主要活动是讲座和游戏，一般听一场讲座了解一些原理之后玩两三个小时相关的游戏加强理解，很好玩也非常有意义。在游戏的间歇 Jane Street 员工会跟我们讨论我们的想法，以及他们想让我们在其中观察学习到什么。全程英文，听懂不是很难，表达会慢慢适应，不过说多了之后就不会说中文了（？）

![](/log/qtc/1.jpg)

### Day 1

They first introduced what Jane Street is and is not:

1. Jane Street is not a bank / high-frequency trading firm.
2. Jane Street trades on its own money.
3. Jane Street helps to regulate the market.

Jane Street consists of three main departments: **traders**, **developers** and **researchers**.
About 30% are traders.

We played a game helping us know how trades happen by giving people different colored coins, and only coins that matched your paper cup counted for you. For example, each blue coin was worth $10 inside a blue cup, and $0 in other cups. Black coins were worth $1 for everyone. By only allowing exchanges between neighbors, trades began naturally. Nearly all people exchanged with toll fees, but there were still profits to earn.

![](/log/qtc/2.jpg)

The next game was the estimathon. See https://estimathon.com/ for reference. Actually, I've played it before in various places and it was amazing to know that it was Jane Street that first created the game. In this game, we mainly estimated the upper & lower bound of the answer to tricky questions like *How many emails have been sent last year?*.

Then we started another building game. People were divided into two groups: one group was called miners who auctioned for a bunch of materials and sold pieces of them to another group of builders. Builders then built towers using materials they purchased from miners and earned $1 for each centimeter high their building was. Both sides had no idea what the others' rules were. In the second round, the policy was made public, and in the third round, special rewards like doubling the height were introduced. Trading patterns were totally not the same.

![](/log/qtc/3.jpg)

In the lecture section, we were taught the Bayes formula and then practiced with a new game in which you had 2/3 probs to win $1000 and 1/3 to lose $2000. You could choose to gather information by $20, which gave you a signal of + or - that was related to win or lose in probability. You were required to decide whether you bet, stopped, or gather information (no limitation in each round). We also learned some basics of machine learning.

These games were super exciting and demonstrated an overview of real markets excellently.

### Day 2

On the second day of QTC, we were first shown a series of heuristics & biases. A simple example was that you were less likely to solve a question marked as 'bonus', even if it shared the same score as those compulsory ones. Biases existed everywhere and we should try to eliminate them.

We then played a special card game called Figgie which I considered the most interesting one among all the games. See https://www.janestreet.com/figgie/ for the rules. We needed to gather information (in most of the time there was very little) from others' bids and offers to make our own trades. Trading terms like *34 bid for 100* were also taught.

After having a lecture on arbitrages, we started a mock game where gold, silver, and bronze coins could be exchanged at a constant ratio, while the bid & offer price of each coin would change either in response to trades or randomly. To make money, we need to monitor all the prices and make arbitrage tradings. That gave us ideas of how Jane Street worked.

![](/log/qtc/4.jpg)

### Day 3

Deluges of information and terms flooded towards us on day 3.

- Capital, Trades, Markets, Orders, Liquidity
- Capital seekers, Capital lenders, Equity, Bond, Financial professionals, Regulators
- IPO
- AAPL, Trade tape, Lit exchange, Dark pool
- Continuous order book, Auctions
- Limit orders, Marketable orders
- Electronic, OTC
- Investment Banks, Analysts, Arbitrageurs, Market makers, Activist investors, Private equity funds, Product issuers, Regulators
- Stocks, Futures, Currencies, Mutual funds, ETF, Options, Bonds, Depository receipts, Swaps

The lecturer also moved deeper into our recognition systems: intuition and reasoning. We learned all these terms in approximately 1.5 hours. However, as it began from the very beginning, there's no need to prepare, but concentration was highly required. That explained more into what was happening behind our games in the past few days.

![](/log/qtc/5.jpg)

Nearly all of the day was spent learning markets, trading, machine learning, and biases. Games like Hi-Lo Mock took place for an hour or so. After the exhausting day, I formed a more detailed understanding of the market, tradings, and how Jane Street was participating in this huge real-life game.

We were encouraged to apply for internships at Jane Street, and by participating in QTC we also gained a brief knowledge of what Jane Street interviews emphasized.

## 收获

去 Jane Street QTC 之前我对几乎市场一无所知，不知道它存在的意义更不知道这些公司如何赚取巨额财富。以前会觉得这只是资本家在利用他们手中的金钱，是一种不劳而获。但是深入了解之后才知道 markets 和 traders 本身就在创造价值。正如讲座中提到过的例子，一个地区的人会捕鱼但是常年露宿野外，另一个地区的人精通建筑但是每天摘果子为生。如果有人在两个地区之间建一条路，双方的生活质量都会提高，他们也会乐于支付通行费。包括 Jane Street 在内很多量化公司在赚取利润的同时都致力于将市场控制在一个公平的价格区间内，也就是在稳定市场的运作，在创造价值的同时也是市场运作不可或缺的部分。我以后可能会去计算机相关领域工作，也有可能尝试在 Jane Street 等公司实习，但无论如何都对这些公司、这个领域有了更加深入的了解，这也势必会成为自己未来人生很重要的经历。

## 花絮

好不容易出来玩一圈当然不能放弃机会，香港比想象中和平很多，正常运作，完全看不出来曾经暴动的痕迹。

1. 公司 bar 包场，从西装革履的人群中穿过，像送外卖的

![](/log/qtc/6.jpg)

2. 遇到两个同样很爱浪的同学，晚上去摩天轮，乘船去九龙，在海边聊人生

![](/log/qtc/7.jpg)
<!-- ![](/log/qtc/8.jpg) -->

3. 坐小火车上太平山观景，结果全是雾什么都看不见

![](/log/qtc/9.jpg)

4. 想着来吃海鲜结果吃了好多天咖喱牛腩，不过都很好吃

## 小结

看到宣传时离申请 ddl 已经只有一天了。熬夜肝了一篇英文简历出来绝对是一个明智的决定。在 QTC 的这三天我从无到有建立起了对 trading 运作方式、Jane Street 工作内容等的认知，极大地丰富了视野与知识储备，也认识到这个领域所需要的技能以及做出的贡献。如果你正在犹豫是否报名，可以大胆一试，绝对不会令你失望。

![](/log/qtc/10.jpg)

## 后记

在 Jane Street QTC gaming 的过程中对自己的总结是「低天赋高成长」。其实很早就发现我对概念性的说明理解很慢，一个算法如果只给我代码可能要想比较长时间才能理解透彻，但是如果有一些例子帮助思考速度会提升非常快。这次活动的游戏中也发现每种类型游戏的第一轮我都会输得比较惨，往往结束时会亏损 20% 或者 30%。但是经过一轮观察和分析讨论，第二轮第三轮往往就能取得一个比较好的结果。这提醒我无论如何都不能冲动，我对包括数字在内很多事物的第一感觉很有可能是不准确的，唯有稳定、细致观察后才可以准备行动。
