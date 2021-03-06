> ##### 作者介绍
> 王喆，毕业于清华大学计算机系，现在美国最大的 smartTV 公司 Roku 任 senior machine learning engineer，曾任 hulu senior research SDE，7 年计算广告、推荐系统领域业界经验，相关专利 3 项，论文 7 篇，《机器学习实践指南》、《百面机器学习》作者之一。知乎专栏 / 微信公众号：王喆的机器学习笔记。<br /><br />
> [知乎专栏 - 王喆的机器学习笔记](https://zhuanlan.zhihu.com/wangzhenotes)、[作者InfoQ主页](https://www.infoq.cn/profile/1372410)

### [深度学习CTR预估模型凭什么成为互联网增长的关键？](https://www.infoq.cn/article/3WK8*7x0avuuh3CJK6Pw)

开篇伊始，有两个问题是应该澄清的，一是该专栏的主题选择，二是该专栏的目标受众。

- **为什么着重讲深度 CTR 模型这个主题**？ 除了跟我的计算广告、推荐系统的背景有关之外，更重要的是 CTR 预估模型以及 CTR 预估模型衍生出的泛效果预测的模型，已经成为了互联网当之无愧的“增长之心”。自 2012 年以来，站在 Yann LeCun、Geoffrey Hinton 和 Yoshua Bengio 等巨人肩膀上的 Alex Krizhevsky 凭借 AlexNet 一举引爆新一轮的深度学习浪潮以来，深度学习席卷各个计算机应用领域，作为广告、搜索、推荐业务核心的 CTR 预估模型也借助深度学习得到效果上的显著提升，成为几乎所有主流互联网公司的标准配置。
- **这个专栏希望哪些受众从中受益**？我希望把专栏的受众分为两类，一是互联网行业相关方向，特别是广告、推荐、搜索领域的从业者，希望这些同学能够熟悉深度 CTR 模型的发展脉络，清楚每个关键模型的技术细节，进而能够在工作中应用甚至改进这些模型；二是有一定机器学习基础，想进入这个领域的爱好者、在校生。我会尽量用平实的语言从细节出发介绍每个 CTR 预估模型，希望大家能够从零开始构建深度 CTR 模型的知识体系。

对于互联网从业者来说，“增长”这个词就像是插在心中的一只矛，无时无刻不被其刺激和激励着。我对“增长”这个词的理解还是来源于上大学时实验室的一段经历。清华计算机系跟搜狗一直是长期的伙伴，因此实验室的师兄师姐也经常谈起与搜狗合作的项目，我一直记忆至今的一句话是“如果我们能把搜狗搜索引擎广告的点击率提升 1%，那就能为公司带来上千万的利润”。从那时，“点击率（Click Through Rate， CTR）”这个词深深的烙在我心中，可能也在潜意识中指引我走上计算广告工程师的职业道路。那么到底是怎样的一个指标能对公司的增长起到如此至关重要的效果，为什么 CTR 预估模型能够被称为互联网“增长之心”，下面我尝试用两个场景给出答案。

#### CTR 预估模型与计算广告的利润增长
CTR 预估模型在计算广告领域的关键地位来源于计算广告利润增长的需求。CTR 预估的准确与否，直接影响计算广告公司的收入。

假设我们是一个 DSP（Demand side platform）公司，需要对接第三方的流量资源，通过出价的方式竞得该流量，从而赢得这个广告曝光（impression）机会。

对于一个以效果为核心目标的中小广告主来说，往往会选择 CPC 的结算方式，也就是每带来一次点击，我为你支付 x 元。那么这时，CTR 模型的关键就体现出来了，因为只有拥有了准确的 CTR 模型，DSP 公司才能够正确的估计某次流量的成本价。

例如 CTR 预估模型预测某流量投放某广告的点击率是 0.5%（即 CTR=0.5%），广告主愿意为一次点击支付 1 元（即 CPC=1），那么我只有用少于 CPC*CTR = 1 * 0.5% = 0.005 元的价格竞得该流量，我才不会亏钱。如果 CTR 模型预测的 CTR 偏高，我将极可能以高于成本价的价格竞得该流量，这样的情况下，竞得越多这样的流量，公司的亏损也越大。另一方面，如果 CTR 模型预测的 CTR 过低，进而出价过低，很有可能损失大量竞得机会，导致客户的广告预算花不完，从而无法获得后续订单，也使公司利润受损。

因此，精准的 CTR 预估模型是计算广告系统的基础和核心，也是计算广告公司进行利润最大化的核心模块，所以说 CTR 预估模型是计算广告利润的增长之心丝毫不为过。

#### CTR 预估模型与推荐系统的用户使用时长增长
广义上来讲，计算广告和推荐系统的界限并不那么严格，比如淘宝的直通车广告，应该属于计算广告的范畴，但它又完全符合商品推荐的场景。这里我倾向于把一切跟“钱”直接相关的模型归为计算广告的范畴，把一切跟“用户体验”直接相关的模型归为推荐系统的范畴，虽然提高用户体验更本质的目标也是为了最终实现产品利润的增长，但这与计算广告时刻跟出价、转化率、投资回报等“钱”相关的模型还是有业务场景上的较大区别。

所谓提高“用户体验”，可以进一步做这样的解释——“推荐系统的优化目标应该是为了在不损害用户长期兴趣的基础上增加用户的使用时长”。以 YouTube 为例，其商业目标就是为了通过提高用户观看总时长，实现广告 inventory 的增长，进而增加公司利润。

实现这一目标的关键就在于预测用户 U（user）在某场景 C（context）下是否会观看某视频 V（video），以及观看该视频的观看时长是多少。这与计算广告中的 CTR 预估模型的区别仅在于将 A（ad）换成了 V（video），将构建 CTR=g(A, U, C) 的问题换成了构建 watch time=g（V, U, C）的问题。

事实上，YouTube 的工程师们在那篇著名的工程论文“Deep Neural Networks for YouTube Recommendations”也非常明确提出了改造 CTR 预估模型为预测用户时长的深度学习推荐模型的方法。由 CTR 预估模型衍生出的泛效果模型共同构成了驱动互利网场景下的用户增长、使用时长增长、转化效果增长等一系列的关键商业指标的增长之心。

#### 深度学习 CTR 预估模型专栏的结构
专栏的正式内容将会分为两大部分，一是深度 CTR 模型的理论和技术发展脉络；二是深度 CTR 模型的系统设计和工程实践。

理论部分又将会分为“前深度学习时代“和”深度学习时代“两部分，这里仍要强调”前深度学习时代“CTR 预估模型的原因，是希望大家能够建立完整的学习框架，并打牢深度学习模型的理论基础，二者本质上是不可分割的。

而实践部分将分为”模型实现与部署“和”模型业界应用“两部分。能够掌握从”模型理论“到”模型实现“再到”上线部署“的一整套技术栈对于算法工程师来说是重要的。最终的”业界应用”部分包括了 Google，Airbnb，Facebook，Alibaba 等业界知名互利网公司的 CTR 预估模型的设计和应用案例，希望读者能够从应用中学到更多实践中应该注意的技术细节。

### [前深度学习时代CTR预估模型的演化之路：从LR到FFM](https://www.infoq.cn/article/wEdZsdQZ2pIKjs_x3JGJ)

在互联网永不停歇的增长需求的驱动下，CTR 预估模型（以下简称 CTR 模型）的发展也可谓一日千里，从 2010 年之前千篇一律的逻辑回归（Logistic Regression，LR），进化到因子分解机（Factorization Machine，FM）、梯度提升树（Gradient Boosting Decision Tree，GBDT），再到 2015 年之后深度学习的百花齐放，各种模型架构层出不穷。

我想所有从业者谈起深度学习 CTR 预估模型都有一种莫名的兴奋，但在这之前，认真的回顾前深度学习时代的 CTR 模型仍是非常必要的。原因有两点：

1. 即使是深度学习空前流行的今天，LR、FM 等传统 CTR 模型仍然凭借其可解释性强、轻量级的训练部署要求、便于在线学习等不可替代的优势，拥有大量适用的应用场景。模型的应用不分新旧贵贱，熟悉每种模型的优缺点，能够灵活运用和改进不同的算法模型是算法工程师的基本要求。
2. 传统 CTR 模型是深度学习 CTR 模型的基础。深度神经网络（Deep Nerual Network，DNN）从一个神经元生发而来，而 LR 模型正是单一神经元的经典结构；此外，影响力很大的 FNN，DeepFM，NFM 等深度学习模型更是与传统的 FM 模型有着千丝万缕的联系；更不要说各种梯度下降方法的一脉相承。所以说传统 CTR 模型是深度学习模型的地基和入口。

下面，我们用传统 CTR 模型演化的关系图来正式开始技术部分的内容。

<img alt="图2-1" src="https://richie-leo.github.io/ydres/img/10/170/1007/2-01.png" style="max-width:471px;width:99%" />

看到上面的关系图，有经验的同学可能已经对各模型的细节和特点如数家珍了。中间位置的 LR 模型向四个方向的延伸分别代表了传统 CTR 模型演化的四个方向。
1. 向下为了解决特征交叉的问题，演化出 PLOY2，FM，FFM 等模型； 
2. 向右为了使用模型化、自动化的手段解决之前特征工程的难题，Facebook 将 LR 与 GBDT 进行结合，提出了 GBDT+LR 组合模型； 
3. 向左 Google 从 online learning 的角度解决模型时效性的问题，提出了 FTRL； 
4. 向上阿里基于样本分组的思路增加模型的非线性，提出了 LS-PLM（MLR）模型。

#### LR——CTR 模型的基础

位于正中央的是当之无愧的 Logistic Regression。仍记得 2012 年我刚进入计算广告这个行业的时候，各大中小公司的主流 CTR 模型无一例外全都是 LR 模型。LR 模型的流行是有三方面原因的，一是数学形式和含义上的支撑；二是人类的直觉和可解释性的原因；三是工程化的需要。

##### 1. 逻辑回归的数学基础

逻辑回归作为广义线性模型的一种，它的假设是因变量 y 服从伯努利分布。那么在点击率预估这个问题上，“点击”这个事件是否发生就是模型的因变量 y。而用户是否点击广告这个问题是一个经典的掷偏心硬币问题，因此 CTR 模型的因变量显然应该服从伯努利分布。所以采用 LR 作为 CTR 模型是符合“点击”这一事件的物理意义的。

与之相比较，线性回归（Linear Regression）作为广义线性模型的另一个特例，其假设是因变量 y 服从高斯分布，这明显不是点击这类二分类问题的数学假设。
 在了解 LR 的数学基础后，其目标函数的形式就不再是空中楼阁了，具体的形式如下：

<img alt="图2-2" src="https://richie-leo.github.io/ydres/img/10/170/1007/2-02.png" style="max-width:200px;width:99%" />

其中 x 是输入向量，θ 是我们要学习的参数向量。结合 CTR 模型的问题来说，x 就是输入的特征向量，h(x) 就是我们最终希望得到的点击率。

##### 2. 人类的直觉和可解释性

直观来讲，LR 模型目标函数的形式就是各特征的加权和，再套上 sigmoid 函数。我们忽略其数学基础（虽然这是其模型成立的本质支撑），仅靠人类的直觉认知也可以一定程度上得出使用 LR 作为 CTR 模型的合理性。
 使用各特征的加权和是为了综合不同特征对 CTR 的影响，而由于不同特征的重要程度不一样，所以为不同特征指定不同的权重来代表不同特征的重要程度。最后要套上 sigmoid 函数
 
 <img alt="图2-3" src="https://richie-leo.github.io/ydres/img/10/170/1007/2-03.png" style="max-width:150px;width:99%" />
 
 正是希望其值能够映射到 0-1 之间，使其符合 CTR 的物理意义。

<img alt="图2-4" src="https://richie-leo.github.io/ydres/img/10/170/1007/2-04.png" style="max-width:300px;width:99%" />

 LR 如此符合人类的直觉认知显然有其他的好处，就是模型具有极强的可解释性，算法工程师们可以轻易的解释哪些特征比较重要，在 CTR 模型的预测有偏差的时候，也可以轻易找到哪些因素影响了最后的结果，如果你有跟运营、产品一起工作的经验的话，更会知道可解释性强是一个模型多么优秀的“品质”。

##### 3. 工程化的需要

在互联网公司每天动辄 TB 级别的数据面前，模型的训练开销就异常重要了。在 GPU 尚未流行开来的 2012 年之前，LR 模型也凭借其易于并行化、模型简单、训练开销小等特点占据着工程领域的主流。囿于工程团队的限制，即使其他复杂模型的效果有所提升，在没有明显 beat LR 之前，公司也不会贸然加大计算资源的投入升级 CTR 模型，这是 LR 持续流行的另一重要原因。

#### POLY2——特征交叉的开始

但 LR 的表达能力毕竟是非常初级的。在“辛普森悖论”现象的存在下，只用单一特征进行判断，甚至会得出错误的结论。我们用一个简单的“辛普森悖论”的例子来解释一下为什么特征交叉是重要的。

假设下面是某 APP 男性用户和女性用户点击广告的数据：

男性用户

广告位 | 点击 | 曝光 | 点击率
---|---|---|---
广告位 A |  8 | 530 | 1.51%
广告位 B | 51 | 1520 | 3.36%

女性用户

广告位 | 点击 | 曝光 | 点击率
---|---|---|---
广告位 A | 201 | 2510 | 8.01%
广告位 B | 92 | 1010 | 9.11%

透过上面两个表格的数据来看，广告位 B 无论在男性用户还是女性用户中的点击率都高于广告位 A，如果我们作为广告需求方构建的 CTR 模型够精准，理所应当会把广告预算分配给广告位 B，而不是广告位 A。

那么我们如果去掉性别这个维度，把数据汇总后会得出什么结论呢？

广告位 | 点击 | 曝光 | 点击率
---|---|---|---
广告位 A | 209 | 3040 | 6.88%
广告位 B | 143 | 2530 | 5.65%

在汇总结果中，广告位 A 的点击率居然比广告位 B 高。我们如果据此进行广告预算的分配，将得出完全相反的结论。可这个结论明显是错误的，广告位 A 的综合点击率高仅仅是因为其女性用户更多，提高了整体点击率。如果你构建的 CTR 模型的表达能力不够，很可能被数据“欺骗”。

因此，从“辛普森悖论”中我们能够得出一个结论，低维特征由于对高维特征进行了合并，丢失掉了大量信息。而 LR 由于只是对单一特征做简单加权，不具备进行特征交叉生成高维特征的能力，所以表达能力是非常初级的。

针对这个问题，当时的算法工程师们经常采用手动组合特征，再通过各种分析手段筛选特征的方法。但这个方法无疑是残忍的，完全不符合“懒惰是程序员的美德”这一金科玉律。更遗憾的是，人类的经验往往有局限性，程序员的时间和精力也无法支撑其找到最优的特征组合。因此采用 PLOY2 模型进行特征的“暴力”组合成为了可行的选择。

<img alt="图2-5" src="https://richie-leo.github.io/ydres/img/10/170/1007/2-05.png" style="max-width:420px;width:99%" />

在上面 POLY2 二阶部分的目标函数中（上式省略一阶部分和 sigmoid 函数的部分），我们可以看到 POLY2 对所有特征进行了两两交叉，并对所有的特征组合赋予了权重 wh(j1, j2)。POLY2 无疑通过暴力组合特征的方式一定程度上解决了特征组合的问题。并且由于本质上仍是线性模型，其训练方法与 LR 并无区别，便于工程上的兼容。

<img alt="图2-6" src="https://richie-leo.github.io/ydres/img/10/170/1007/2-06.png" style="max-width:650px;width:99%" />

但 POLY2 这一模型同时存在着两个巨大的缺陷：
1. 由于在处理互联网数据时，经常采用 one-hot 的方法处理 id 类数据，致使特征向量极度稀疏，POLY2 进行无选择的特征交叉使原本就非常稀疏的特征向量更加稀疏，使得大部分交叉特征的权重缺乏有效的数据进行训练，无法收敛。 
2. 权重参数的数量由 n 直接上升到 n2，极大增加了训练复杂度。

#### FM——隐向量特征交叉

为了解决 POLY2 模型的缺陷，2010 年 Rendle 提出了 FM（Factorization Machine）。

<img alt="图2-7" src="https://richie-leo.github.io/ydres/img/10/170/1007/2-07.png" style="max-width:460px;width:99%" />

从 FM 的目标函数的二阶部分中我们可以看到，相比 POLY2，主要区别是用两个向量的内积（wj1 · wj2）取代了单一的权重 wh(j1, j2)。具体来说，FM 为每个特征学习了一个隐权重向量（latent vector），在特征交叉时，使用两个特征隐向量的内积作为交叉特征的权重。

通过引入特征隐向量的方式，直接把原先 n2 级别的权重参数数量减低到了 nk（k 为隐向量维度，n>>k）。在训练过程中，又可以通过转换目标函数形式的方法，使 FM 的训练复杂度减低到 nk 级别。相比 POLY2 极大降低训练开销。

隐向量的引入还使得 FM 比 POLY2 能够更好的解决数据稀疏性的问题。举例来说，我们有两个特征，分别是 channel 和 brand，一个训练样本的 feature 组合是 (ESPN, Adidas)，在 POLY2 中，只有当 ESPN 和 Adidas 同时出现在一个训练样本中时，模型才能学到这个组合特征对应的权重。而在 FM 中，ESPN 的隐向量也可以通过 (ESPN, Gucci) 这个样本学到，Adidas 的隐向量也可以通过 (NBC, Adidas) 学到，这大大降低了模型对于数据稀疏性的要求。甚至对于一个从未出现过的特征组合 (NBC, Gucci)，由于模型之前已经分别学习过 NBC 和 Gucci 的隐向量，FM 也具备了计算该特征组合权重的能力，这是 POLY2 无法实现的。也许 FM 相比 POLY2 丢失了某些信息的记忆能力，但是泛化能力大大提高，这对于互联网的数据特点是非常重要的。

工程方面，FM 同样可以用梯度下降进行学习的特点使其不失实时性和灵活性。相比之后深度学习模型复杂的网络结构导致难以线上 serving 的问题，FM 比较容易实现的 inference 过程也使其没有 serving 的难题。因此 FM 在 2012-2014 年前后逐渐成为业界 CTR 模型的主流。

#### FFM——引入特征域概念

2015 年，基于 FM 提出的 FFM（Field-aware Factorization Machine ，简称 FFM）在多项 CTR 预估大赛中一举夺魁，并随后被 Criteo、美团等公司深度应用在 CTR 预估，推荐系统领域。相比 FM 模型，FFM 模型主要引入了 Field-aware 这一概念，使模型的表达能力更强。

<img alt="图2-7" src="https://richie-leo.github.io/ydres/img/10/170/1007/2-07.png" style="max-width:460px;width:99%" />

上式是 FFM 的目标函数的二阶部分。其与 FM 目标函数的区别就在于隐向量由原来的 wj1 变成了 wj1,f2，这就意味着每个特征对应的不是一个隐向量，而是对应着不同域的一组隐向量，当 xj1 特征与 xj2 特征进行交叉时，xj1 特征会从 xj1 的一组隐向量中挑出与特征 xj2 的域 f2 对应的隐向量 wj1,f2 进行交叉。同理 xj2 也会用与 xj1 的域 f1 对应的隐向量进行交叉。

那么这里所说的“域”代表着什么呢？

简单来讲“域”代表着特征域，域内的特征一般会采用 one-hot 编码形成 one-hot 特征向量。

我们通过 Criteo FFM 论文的中一个例子来更具体的说明 FFM 的过程。假设我们在训练 CTR 模型过程中接收到下面这个样本。

<img alt="图2-9" src="https://richie-leo.github.io/ydres/img/10/170/1007/2-09.png" style="max-width:600px;width:99%" />

其中，Publisher，Advertiser，Gender 就是三个特征域。ESPN、NIKE、Male 分别是这三个特征域的特征。那么，如果按照原 FM 的原理，ESPN 与 NIKE、以及 ESPN 与 Male 做交叉的权重应该是

<img alt="图2-10" src="https://richie-leo.github.io/ydres/img/10/170/1007/2-10.png" style="max-width:520px;width:99%" />

大家肯定已经注意到，由于交叉特征域的改变，ESPN 的隐向量由 wESPN,A 换成了 wESPN,G 。

FFM 模型学习每个特征在 f 个域上的 k 维隐向量，交叉特征的权重由特征在对方特征域上的隐向量内积得到，权重数量共 nkf 个。在训练方面，由于 FFM 的二次项并不能够像 FM 那样简化，因此其复杂度为 kn2。

相比 FM，FFM 由于引入了 field 这一概念，为模型引入了更多有价值信息，使模型表达能力更强，但与此同时，FFM 的计算复杂度上升到 kn2，远远大于 FM 的 k*n。

#### CTR 模型特征交叉方向的演化

本章我们沿着传统 CTR 模型演化图中黄色的部分，朝着特征交叉的演化方向，依次介绍了 LR、POLY2，FM 和 FFM 四个模型。

<img alt="图2-11" src="https://richie-leo.github.io/ydres/img/10/170/1007/2-11.png" style="max-width:471px;width:99%" />

我们再用图示方法回顾一下从 POLY2 到 FM，再到 FFM 进行特征交叉方法的不同。

<img alt="图2-12" src="https://richie-leo.github.io/ydres/img/10/170/1007/2-12.png" style="max-width:550px;width:99%" />

POLY2 模型直接学习每个交叉特征的权重，权重数量共 n2 个。

<img alt="图2-13" src="https://richie-leo.github.io/ydres/img/10/170/1007/2-13.png" style="max-width:550px;width:99%" />

FM 模型学习每个特征的 k 维隐向量，交叉特征由相应特征隐向量的内积得到，权重数量共 n*k 个。

<img alt="图2-14" src="https://richie-leo.github.io/ydres/img/10/170/1007/2-14.png" style="max-width:550px;width:99%" />

FFM 模型引入了特征域这一概念，在做特征交叉时，每个特征选择与对方域对应的隐向量做内积运算得到交叉特征的权重。参数数量共 nkf 个。

但无论怎样 FFM 只能够做到二阶的特征交叉，如果要继续提高特征交叉的维度，不可避免的会发生组合爆炸和计算复杂度过高的情况。

下一节的专栏文章将回顾模型演化图中蓝色部分的模型。我将沿着特征工程的角度带大家一起回顾 Facebook 的 CTR 模型 GBDT+LR，并从时效性的角度看看 Google 是如何使用 FTRL 解决模型 online learning 问题的，最后介绍阿里的 LS-PLM，看其是如何从样本聚类的动机出发为传统线性模型引入非线性能力的。期待与大家继续探讨 CTR 预估模型的内容。

### [盘点前深度学习时代阿里、谷歌、Facebook的CTR预估模型](https://www.infoq.cn/article/Q8Cv-4vq6jD8974w9MxJ)

前面我们沿着特征交叉这条发展路线，回顾了从 LR 到 POLY2，FM，再到 FFM 的 CTR 模型演化过程。但任何问题的观察角度都不是唯一的，CTR 模型更是如此，本文将继续探索前深度学习时代的 CTR 模型，从特征工程模型化的角度回顾 Facebook 的 CTR 模型——GBDT+LR，从时效性的角度学习 Google 是如何使用 FTRL 解决模型 online learning 问题的，最后从样本聚类的动机出发介绍阿里的 LS-PLM CTR 模型。

在介绍细节之前，我们再次回到前深度学习时代的 CTR 模型关系图，熟悉各个模型在演化过程中的位置，其中黄色的部分是上篇专栏文章介绍的模型，蓝色部分将是这篇文章的主要内容。

<img alt="图2-11" src="https://richie-leo.github.io/ydres/img/10/170/1007/2-11.png" style="max-width:471px;width:99%" /><br />传统 CTR 模型演化关系图

#### GBDT+LR——特征工程模型化的开端

上篇文章介绍的 FFM 模型采用引入特征域的方式增强了模型的表达能力，但无论如何，FFM 只能够做二阶的特征交叉，如果要继续提高特征交叉的维度，不可避免的会发生组合爆炸和计算复杂度过高的情况。那么有没有其他的方法可以有效的处理高维特征组合和筛选的问题？2014 年，Facebook 提出了基于 GBDT+LR 组合模型的解决方案。

简而言之，Facebook 提出了一种利用 GBDT 自动进行特征筛选和组合，进而生成新的离散特征向量，再把该特征向量当作 LR 模型输入，预估 CTR 的模型结构。

<img alt="图3-2 GBDT+LR 的模型结构" src="https://richie-leo.github.io/ydres/img/10/170/1007/3-02.png" style="max-width:450px;width:99%" /><br />GBDT+LR 的模型结构

需要强调的是，用 GBDT 构建特征工程，和利用 LR 预估 CTR 两步是独立训练的。所以自然不存在如何将 LR 的梯度回传到 GBDT 这类复杂的问题，而利用 LR 预估 CTR 的过程在上篇文章中已经有所介绍，在此不再赘述，下面着重讲解如何利用 GBDT 构建新的特征向量。

大家知道，GBDT 是由多棵回归树组成的树林，后一棵树利用前面树林的结果与真实结果的残差做为拟合目标。每棵树生成的过程是一棵标准的回归树生成过程，因此每个节点的分裂是一个自然的特征选择的过程，而多层节点的结构自然进行了有效的特征组合，也就非常高效的解决了过去非常棘手的特征选择和特征组合的问题。

利用训练集训练好 GBDT 模型之后，就可以利用该模型完成从原始特征向量到新的离散型特征向量的转化。具体过程是这样的，一个训练样本在输入 GBDT 的某一子树后，会根据每个节点的规则最终落入某一叶子节点，那么我们把该叶子节点置为 1，其他叶子节点置为 0，所有叶子节点组成的向量即形成了该棵树的特征向量，把 GBDT 所有子树的特征向量连接起来，即形成了后续 LR 输入的特征向量。

<img alt="图3-3 GBDT 生成特征向量的过程" src="https://richie-leo.github.io/ydres/img/10/170/1007/3-03.png" style="max-width:650px;width:99%" /><br />GBDT 生成特征向量的过程

举例来说，如上图所示，GBDT 由三颗子树构成，每个子树有 4 个叶子节点，一个训练样本进来后，先后落入“子树 1”的第 3 个叶节点中，那么特征向量就是 [0,0,1,0]，“子树 2”的第 1 个叶节点，特征向量为 [1,0,0,0]，“子树 3”的第 4 个叶节点，特征向量为 [0,0,0,1]，最后连接所有特征向量，形成最终的特征向量 [0,0,1,0,1,0,0,0,0,0,0,1]。

由于决策树的结构特点，事实上，决策树的深度就决定了特征交叉的维度。如果决策树的深度为 4，通过三次节点分裂，最终的叶节点实际上是进行了 3 阶特征组合后的结果，如此强的特征组合能力显然是 FM 系的模型不具备的。但由于 GBDT 容易产生过拟合，以及 GBDT 这种特征转换方式实际上丢失了大量特征的数值信息，因此我们不能简单说 GBDT 由于特征交叉的能力更强，效果就比 FFM 好，在模型的选择和调试上，永远都是多种因素综合作用的结果。

GBDT+LR 比 FM 重要的意义在于，它大大推进了特征工程模型化这一重要趋势，某种意义上来说，之后深度学习的各类网络结构，以及 embedding 技术的应用，都是这一趋势的延续。

在之前所有的模型演化过程中，实际上是从特征工程这一角度来推演的。接下来，我们从时效性这个角度出发，看一看模型的更新频率是如何影响模型效果的。

<img alt="图3-4 模型实效性实验" src="https://richie-leo.github.io/ydres/img/10/170/1007/3-04.png" style="max-width:600px;width:99%" /><br />模型实效性实验

在模型更新这个问题上，我们的直觉是模型的训练时间和 serving 时间之间的间隔越短，模型的效果越好，为了证明这一点，facebook 的工程师还是做了一组实效性的实验（如上图），在结束模型的训练之后，观察了其后 6 天的模型 loss（这里采用 normalized entropy 作为 loss）。可以看出，模型的 loss 在第 0 天之后就有所上升，特别是第 2 天过后显著上升。因此 daily update 的模型相比 weekly update 的模型效果肯定是有大幅提升的。

如果说日更新的模型比周更新的模型的效果提升显著，我们有没有方法实时引入模型的效果反馈数据，做到模型的实时更新从而进一步提升 CTR 模型的效果呢？Google 2013 年应用的 FTRL 给了我们答案。

#### FTRL——天下武功，唯快不破

FTRL 的全称是 Follow-the-regularized-Leader，是一种在线实时训练模型的方法，Google 在 2010 年提出了 FTRL 的思路，2013 年实现了 FTRL 的工程化，之后快速成为 online learning 的主流方法。与模型演化图中的其他模型不同，FTRL 本质上是模型的训练方法。虽然 Google 的工程化方案是针对 LR 模型的，但理论上 FTRL 可以应用在 FM，NN 等任何通过梯度下降训练的模型上。

为了更清楚的认识 FTRL，这里对梯度下降方法做一个简要的介绍。从训练样本的规模角度来说，梯度下降可以分为：batch，mini-batch，SGD（随机梯度下降）三种，batch 方法每次都使用全量训练样本计算本次迭代的梯度方向，mini-batch 使用一小部分样本进行迭代，而 SGD 每次只利用一个样本计算梯度。对于 online learning 来说，为了进行实时得将最新产生的样本反馈到模型中，SGD 无疑是最合适的训练方式。

但 SGD 对于互利网广告和推荐的场景来说，有比较大的缺陷，就是难以产生稀疏解。为什么稀疏解对于 CTR 模型如此重要呢？

之前我们已经多次强调，由于 one hot 等 id 类特征处理方法导致广告和推荐场景下的样本特征向量极度稀疏，维度极高，动辄达到百万、千万量级。为了不割裂特征选择和模型训练两个步骤，如果能够在保证精度的前提下尽可能多的让模型的参数权重为 0，那么我们就可以自动过滤掉这些权重为 0 的特征，生成一个“轻量级”的模型。“轻量级”的模型不仅会使样本部署的成本大大降低，而且可以极大降低模型 inference 的计算延迟。这就是模型稀疏性的重要之处。

而 SGD 由于每次迭代只选取一个样本，梯度下降的方向虽然总体朝向全局最优解，但微观上的运动的过程呈现布朗运动的形式，这就导致 SGD 会使几乎所有特征的权重非零。即使加入 L1 正则化项，由于 CPU 浮点运算的结果很难精确的得到 0 的结果，也不会完全解决 SGD 稀疏性差的问题。就是在这样的前提下，FTRL 几乎完美地解决了模型精度和模型稀疏性兼顾的训练问题。

<img alt="图3-5 FTRL 的发展过程" src="https://richie-leo.github.io/ydres/img/10/170/1007/3-05.png" style="max-width:500px;width:99%" /><br />FTRL 的发展过程

但 FTRL 的提出也并不是一蹴而就的。如上图所示，FTRL 的提出经历了下面几个关键的过程：
1. 从最近简单的 SGD 到 OGD（online gradient descent），OGD 通过引入 L1 正则化简单解决稀疏性问题； 
2. 从 OGD 到截断梯度法，通过暴力截断小数值梯度的方法保证模型的稀疏性，但损失了梯度下降的效率和精度； 
3. FOBOS（Forward-Backward Splitting），google 和伯克利对 OGD 做进一步改进，09 年提出了保证精度并兼顾稀疏性的 FOBOS 方法； 
4. RDA：微软抛弃了梯度下降这条路，独辟蹊径提出了正则对偶平均来进行 online learning 的方法，其特点是稀疏性极佳，但损失了部分精度。 
5. Google 综合 FOBOS 在精度上的优势和 RDA 在稀疏性上的优势，将二者的形式进行了进一步统一，提出并应用 FTRL，使 FOBOS 和 RDA 均成为了 FTRL 在特定条件下的特殊形式。

FTRL 的算法细节对于初学者来说仍然是晦涩的，建议非专业的同学仅了解其特点和应用场景即可。对算法的数学形式和实现细节感兴趣的同学，我强烈推荐微博 冯扬 写的“在线最优化求解”一文，希望能够帮助大家进一步熟悉 FTRL 的技术细节。

#### LS-PLM——阿里曾经的主流 CTR 模型

本文的第三个模型，我们从样本 pattern 本身来入手，介绍阿里的的 LS-PLM（Large Scale Piece-wise Linear Model），它的另一个更广为人知的名字是 MLR（Mixed Logistic Regression）。MLR 模型虽然在 2017 年才公之于众，但其早在 2012 年就是阿里主流的 CTR 模型，并且在深度学习模型提出之前长时间应用于阿里的各类广告场景。

本质上，MLR 可以看做是对 LR 的自然推广，它在 LR 的基础上采用分而治之的思路，先对样本进行分片，再在样本分片中应用 LR 进行 CTR 预估。在 LR 的基础上加入聚类的思想，其动机其实来源于对计算广告领域样本特点的观察 。

举例来说，如果 CTR 模型要预估的是女性受众点击女装广告的 CTR，显然我们并不希望把男性用户点击数码类产品的样本数据也考虑进来，因为这样的样本不仅对于女性购买女装这样的广告场景毫无相关性，甚至会在模型训练过程中扰乱相关特征的权重。为了让 CTR 模型对不同用户群体，不用用户场景更有针对性，其实理想的方法是先对全量样本进行聚类，再对每个分类施以 LR 模型进行 CTR 预估。MLR 的实现思路就是由该动机产生的。

<img alt="图3-6" src="https://richie-leo.github.io/ydres/img/10/170/1007/3-06.png" style="max-width:640px;width:99%" />

MLR 目标函数的数学形式如上式，首先用聚类函数π对样本进行分类（这里的π采用了 softmax 函数，对样本进行多分类），再用 LR 模型计算样本在分片中具体的 CTR，然后将二者进行相乘后加和。

其中超参数分片数 m 可以较好地平衡模型的拟合与推广能力。当 m=1 时 MLR 就退化为普通的 LR，m 越大模型的拟合能力越强，但是模型参数规模随 m 线性增长，相应所需的训练样本也随之增长。在实践中，阿里给出了 m 的经验值为 12。

下图中 MLR 模型用 4 个分片可以完美地拟合出数据中的菱形分类面。

<img alt="图3-7" src="https://richie-leo.github.io/ydres/img/10/170/1007/3-07.png" style="max-width:640px;width:99%" />

MLR 算法适合于工业级的广告、推荐等大规模稀疏数据场景问题。主要是由于表达能力强、稀疏性高等两个优势：
1. 端到端的非线性学习：从模型端自动挖掘数据中蕴藏的非线性模式，省去了大量的人工特征设计，这使得 MLR 算法可以端到端地完成训练，在不同场景中的迁移和应用非常轻松。 
2. 稀疏性：MLR 在建模时引入了 L1 和 L2,1 范数，可以使得最终训练出来的模型具有较高的稀疏度，模型的学习和在线预测性能更好。

如果我们用深度学习的眼光来看待 MLR 这个模型，其在结构上已经很接近由输入层、单隐层、输出层组成的神经网络。所以某种意义上说，MLR 也在用自己的方式逐渐逼近深度学习的大门了。

#### 深度学习 CTR 模型的前夜

2010 年 FM 被提出，特征交叉的概念被引入 CTR 模型；2012 年 MLR 在阿里大规模应用，其结构十分接近三层神经网络；2014 年 Facebook 用 GBDT 处理特征，揭开了特征工程模型化的篇章。这些概念都将在深度学习 CTR 模型中继续应用，持续发光。

另一边，Alex Krizhevsky 2012 年提出了引爆整个深度学习浪潮的 AlexNet，深度学习的大幕正式拉开，其应用逐渐从图像扩展到语音，再到 NLP 领域，推荐和广告也必然会紧随其后，投入深度学习的大潮之中。

2016 年，随着 FNN、Deep&Wide、Deep crossing 等一大批优秀的 CTR 模型框架的提出，深度学习 CTR 模型逐渐席卷了推荐和广告领域，成为新一代 CTR 模型当之无愧的主流。下一篇文章，我们将继续探讨深度学习 CTR 模型，从模型演化的角度揭开所有主流深度学习 CTR 模型之间的关系和每个模型的特点，期待继续与你一同学习讨论。
