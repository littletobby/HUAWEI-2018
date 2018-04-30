# 2018 华为软件精英挑战赛 （上合赛区决赛：227/300分）

## 说明
1. 运行
进入com.file.tool.main 运行Main.java
2. 本地数据集（/HUAWEI-2018/newData）

Input data

- smallCase: 2015.12 - 2016.01 ECS历史请求数据
- largerCase: 2015.01 - 2015.05 ECS历史请求数据

Input param:

- General 56 128 1200
- Large-Memory 84 256 2400
- High-Performance 112 192 3600

Predict Date:

- smallCase: 2016-01-08 ~ 2016-01-14
- largerCase: 2015-05-18 ~ 2015-05-30

## 前言
些许遗憾，没能进入上合赛区前四强，不过从中学习到的一些trick，以及比赛套路值得回味，当然还认识了许多赛区大佬，和他们交流的过程中，总能发掘一些新思路来弥补现有模型的不足。对于我们而言，这次比赛，我们被模型给限制住了思路，没能从全局的角度审视数据集中出现的问题，数据集本身存在的漏洞信息并没有被我们有效地捕捉到，相反进入了构建稳定预测模型的死胡同。多点变通，情况或许就不同了。

## 赛题回顾（参赛选手可直接跳过本节）

**背景:**
云平台为了满足不同租户的需求,提供了一种可随时自助获取、可弹性伸缩的云服务器,即弹性云服务器(Elastic Cloud Server,ECS)。为容纳更多的租户请求、并尽可能提高资源利用率、降低成本,自动化、智能化的资源调度管理系统非常关键。

由于租户对ECS实例(虚拟机,VM)请求的行为具有一定规律,可以通过对历史ECS实例请求的分析,预测到未来一段时间的ECS实例请求,然后对预测的请求分配资源(如图1所示),这样可以找到一个接近最优的分配策略,实现资源最大化利用,同时也能参考预测的结果制定云数据中心的建设计划。

![这里写图片描述](https://img-blog.csdn.net/20180430105657640?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTQ2ODgxNDU=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

**物理服务器:**
云数据中心通常由大规模的物理服务器集群所组成,通过虚拟化技术,可以对每个物理节点的资源(如CPU、内存、硬盘等)进行隔离,使得每台物理机上可以同时容纳多个虚拟机,这些虚拟机即共享该物理服务器上的所有资源。为了保证每台虚拟机的运行性能,通常物理资源不能“超分”,即每台物理服务器上所有虚拟机的虚拟资源总量不能超过其物理资源总量。这里假设物理服务器只有一种规格大小。

**虚拟机规格:**
云平台通常预先定义了各种类型虚拟机的规格(flavor),便于租户选择购买。例如,如果租户对计算要求低,而内存要求相对高,可以选择4个vCPU、16GB内存的配置。由于不考虑物理资源的“超分”,虚拟机的资源占用可与物理资源进行一比一的映射。当然,不同规格的虚拟机有不同的性能,价格也不一。

**资源维度:**
如上述,虚拟机正常工作需要多个维度的资源同时配合,如CPU、内存、硬盘、网络等等,每种资源都可能成为瓶颈,并且每种资源分配不合理都可能产生碎片。这里假设只需要考虑单个维度资源的使用,即尽可能最大化每台物理服务器的CPU或者内存(Mem)的利用率,但在考虑某个维度资源优化的同时,要保证其他资源不能超分。

**历史请求数据:**
租户在云平台上每申请一台虚拟机都会在后台的数据库产生一条数据,每条数据包含了虚拟机的ID、虚拟机的规格大小以及创建时间等。如果我们可以获取到云平台在过去一段时间的所有虚拟机请求数据,通过训练这些数据特征,可以预测下一个时间段可能到来的虚拟机请求分布。

**比赛程序内容:**
请你设计一个程序能够精确预测未来某个时间段内的虚拟机的请求情况,并寻找最佳的资源分配方案:对输入的虚拟机历史请求数据进行建模分析以及训练,确定系数给出预测模型,然后对给定预测时间段内不同规格的虚拟机数量进行预测,最后根据预测结果把虚拟机部署到物理服务器上,使得服务器的资源利用率最大化。

**比赛胜负规则**
比较参赛者程序输出的预测结果的精度以及部署的资源利用率的乘积,得分较大者胜出。如果出现得分相同的情况,则比较程序运行时间,时间短者胜出。若运行时间也相同,则根据提交时间先后来区分排名。如输出结果不满足约束条件,得分为零。 (备注:资源利用率只需要考虑单个维度,如CPU或内存的利用率)


## 基本解决方案

首先，该问题可以分为以下两部分：

- 根据每个flavor的历史请求数据，建立预测模型进行预测。
- 根据预测得到每种flavor的请求数后，进行放置分配算法。

### 预测方案

**备用模型**

时间序列上的预测模型有很多，在前期参考了大量资料后，我们总共实现了以下模型：

- 自回归移动平均差分模型（ARIMA）模型，详见com/elasticcloudservice/arima/ARIMA.java
- 单元线性回归模型，详见com/elasticcloudservice/ml/LinearRegression.java
- 单元多项式拟合模型，详见com/elasticcloudservice/ml/LeastSquareMethod.java
- 局部线性加权，详见com/elasticcloudservice/ml/LocalWeightedLinearRegression.java
- 基于决策树的GBDT模型，详见com/elasticcloudservice/ml/GBM.java
- 深度学习RNN模型之LSTM，详见com/elasticcloudservice/ml/LSTM.java

虽然实现了多种预测模型，但实际预测效果随着模型复杂度上升而递减。在练习阶段，arima模型能稳定在70分上下，而一些线性模型却只能徘徊在65分左右，GBDT和LSTM则只有50分上下。实际上，模型能否发挥功效，取决于数据集的数量和质量。华为所提供的数据集有以下两特点：每个flavor数据稀疏，一周内有多天的实际请求数为0。其次线上数据集最多不超过2万条样本，总共包含了20种flavor的请求量，最多每个flavor只有1000条数据可供训练。因此，复赛阶段，我们弃用了LSTM和GBDT模型。选了局部线性加权模型和ARIMA模型作为线上模型。

从模型的角度来解决上述问题时，还需要前期的数据预处理，比如去噪（数据缺失，异常点过滤），特征提取（是否节假日）等，因此我们也采用了一些常规的去噪，数据预处理手段。

**数据缺失：**

华为有多天的数据是空缺的，这并不代表华为当天没有flavor的请求数，因此我们采用附近N天的数据取平均来弥补这些缺失数据，只能说效果还不错。

**数据去噪（手段一）：**

采用一级指数平滑：一阶指数平滑实际就是对历史数据的加权平均，它可以用于任何一种没有明显函数规律但确实存在某种前后关联的时间序列的短期预测。公式为：

S_{t + 1} = alpha * y_t + (1 - alpha) * S_t

其中，y_t表示第t个时刻的真实预测值，S_t表示第t时刻平滑后的值。不过此处，我们并没有将指数平滑用于预测，而是对公式做了如下修改：

if t = 1: S_1 = y_1

if t > 1: S_t = alpha * y_t + (1 - alpha) * S_{t - 1}

当前第t时刻的平滑值，由两部分组成：t时刻的真实值和前t-1时刻平滑值的加权平均。即当alpha = 1时，不产生平滑效果。当alpha = 0时，是一条恒为y_1的直线。

举个例子：（针对flavor 5）

**alpha = 0.2**

![这里写图片描述](https://img-blog.csdn.net/20180430180906446?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTQ2ODgxNDU=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

**alpha = 0.5**
![这里写图片描述](https://img-blog.csdn.net/20180430181545477?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTQ2ODgxNDU=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

**alpha = 0.9**

![这里写图片描述](https://img-blog.csdn.net/20180430181744472?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTQ2ODgxNDU=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

显然，当alpha = 0.5时，对局部极大值起到了一定的约束作用，并且能较好的拟合原序列。针对线上不同flavor的alpha，我们没有特别好的方法，只能通过线上不断测试得到一个较好结果的alpha。

**数据去噪（手段二）：**

从聚类的角度来看，去噪是为了舍弃异常点（outlier），我们直接采用了一种简单粗暴的思路。核心思路是假设序列中的值符合高斯分布，因此可以去除高斯分布边缘两侧的离群点。如下图：

![这里写图片描述](https://img-blog.csdn.net/20180430182717167?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTQ2ODgxNDU=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

可以去除大于mu + 3 * delta和小于mu - 3 * delta的离群点，保留99%的数据用于训练。

**模型选择**

我们测试很多种模型，始终得不到分数较高且稳定的模型，ARIMA模型和局部线性加权模型在线上表现不错，但稳定性依旧不够，因此我们最终还是采用均值大法来预测。这里需要提一点，复赛阶段，预测可以分为两个阶段，官方给出的预测时间是可能存在间隙的，如下图：

![这里写图片描述](https://img-blog.csdn.net/20180430184509998?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTQ2ODgxNDU=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

线上训练集和预测天数之间可能存在间隙，因此如果直接拿训练集去预测而不考虑间隔会存在较大误差。于是间隔预测采用局部线性加权，原因如下：

- 局部加权模型能够根据近期的趋势来预测未来，从序列中能够得到未来时间序列是否增长or降低的趋势。
- 我们大胆假设离预测起始点越远的历史数据对预测的贡献越小，而局部线性加权可以通过设置sigma很容易做到这点，同时多一个超参意味着能够拟合一个更准确的未来函数。通过对每个flavor过度调参，完全可以做到准确预测。

出于对线上数据的无知考虑，我们希望实际要预测flavor的模型尽可能稳定， 预测模型采用了均值大法。这也是我们此次比赛最大的遗憾，考虑会出现过拟合问题，而挑选了较稳定且简单的模型。结果当官方切换数据集时，发现复赛数据分布和练习阶段分布近乎一致，甚至怀疑没有更换数据集。这种稳定模型，自然很难通过调参得到较高的分数。不过不必再纠结这些细节了，结局已定，且我们始终相信好的模型一定能够防止过拟合，且即使切换数据集也能得到不错的结果。没能做到这点，是我们的模型依旧不够完美。

均值大法很简单，由于我们已经得到了间隔的预测值和实际训练集中的值，把它们合并后，求得最近N天（如10天）每个flavor平均每天的请求数，预测值即为平均请求数 * 预测天数。不过还需要注意一点，均值大法无法得到每种flavor的上升or下降趋势，因此需要乘以一个修正系数来调整一下。经过线上测试，这修正系数大概在1.3左右，这也就意味着大多数flavor，每过一段时间都有一定程度的增长（业务增长）。

此处，考虑为何需要修正系数，我们队处于两点考虑：
 
 - 确实业务存在稳定增长的可能，可以考虑根据预测天数来确定修正系数，我们考虑了利滚利模型。
 - 根据评分公式，能够得到一个很有意思的结论，公式对预测结果偏大的结果惩罚较小。

评分公式如下：
![这里写图片描述](https://img-blog.csdn.net/20180430191406391?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTQ2ODgxNDU=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

举个例子：某个flavor的真实值是100，那么如果模型预测为50时，得到Accuracy = 66.7%，如果模型预测为150，那么得到的Accuracy  = 80%，所以我们宁可让模型多预测也不能让模型少预测。

**关于预测的一些建议及对此次比赛的思考：**

**针对比赛**

如果单纯考虑此次比赛，那没什么好说的了，因为正式赛和练习赛分布一致，在练习阶段可以通过100次提交机会，训练一个过拟合模型，练习赛10分的优势完全可以累积到正式赛，甚至可能会被放大。要做的就是去猜测复赛高级用例长什么样。提分的手段很简单，增加超参数量，比如每种flavor一种指数平滑alpha，每种flavor有自己的局部线性加权超参sigma。除了对flavor进行调参，也可以面向用例调参， 分初，中高，分别得到一组参数。

对于华为提供的两年数据，可以做一个大胆的猜测，线下数据集缺少6，7，8，9，10，11月份的数据，用于线上评测的数据估计就是这些缺少的数据集，通过二分可以尽可能爬取线上数据做验证。假设预测9， 10， 11月份的每种flavor的个数，我们甚至可以利用线下提供的12月，1月的数据，结合6， 7， 8线上数据集进行反向预测。

**防过拟合的思考**

如若考虑过拟合问题，在理想情况下，华为选择不同数据分布来验证预测模型的稳定能力时，可以有如下做法：我们认为，参数反而越少越好，因为在实际比赛中只有5次提交机会，那么可以考虑建立一些自适应模型，来让参数拟合线上结果。有点类似于猜测线上分布函数，但该函数要猜的尽可能准确及需要一定的变动能力，最后根据线上反馈得到猜测误差，用于调节参数和确定参数调节方向。

实际上，上述过程是基于反馈的人工梯度下降。比如考虑猜测函数：y = kx，线上数据为x，预测值为y，真实值我们不得而知，但官方会给我们一个分数，根据该分数就能得到k的变动方向和变动范围。为了完成这件事，我们做了如下操作：

- 建立复杂函数（模型）
- 寻找唯一变动的超参，此处我们选取了修正系数
- 线上验证，参数的调节与官方反馈的分数是否存在相关性

前期考虑到每种flavor会存在业务增长的可能性，所以我们让修正系数根据预测天数动态变化，公式如下：

fixed_factor =  alpha^{(预测天数 + 间隔) / 7 + 1}

该修正因子解决了均值模型无法预测增长趋势的问题，当要预测第一周的flavor时，业务可能增长10%，那么预测值为：原来的预测数 * 1.10。同理可以得到第二周，第三周增长后的预测值。

最终我们可以通过调节alpha来得到一个不错的分数。显然，当alpha越大，分数会先上升再下降。有了这种相关性，通过二分，我们能够方便地确定系数alpha。在有限次数下，能够让模型迅速逼近最优解。

### 放置方案

复赛阶段，提供了三种物理机，分别为：

- General 56 128 1200
- Large-Memory 84 256 2400
- High-Performance 112 192 3600

分别计算它们的MEM / CPU 比为：

- General 2.29;
- Large-Memory 3.05;
- High-Performance 1.71;

再考虑复赛提供的18种flavor的MEM / CPU 比，总共就三种情况：

- flavor1, 4, 7, 10, 13, 16比值为1； 
- flavor2, 5, 8, 11, 14, 17比值为2；
- flavor3, 8, 9, 12, 15, 18比值为4；

现在考虑如何让General 的 CPU 和 MEM的利用率同时最高？很明显，如果有这么一种flavor，MEM / CPU = 2.29，一定能够得到k个flavor完全塞满General。关键就在于这个比值，一种intuition的做法，尽量让MEM / CPU接近2.29，越接近，利用率一定是最高的。

很可惜，现在给定的flavor只存在1，2，4的比值，不过我们可以通过组合flavor的方式降低比值，比如flavor2和flavor3组合在一块，就能得到（2 + 4）/ 2 = 3的MEM/CPU比。嘿，这样就能较大程度上缓解背包的自由组合导致利用率的急速下降。

所以我们前期采取三种贪心策略来预先组合一波flavor，实际测试提升了近3个百分点：

1. flavor的组合需尽量降低MEM/CPU的比值
2. 尽量选择同级别的flavor进行组合（flavor1，2，3之间组合， flavor4，5，6之间的组合，依此类推）
3. 大于Large-Memory的flavor组合还需要进一步降低，小于High-Performance的flavor组合需要进一步提高

上述group的方案采用三组优先队列完成，如下图所示：

![这里写图片描述](https://img-blog.csdn.net/20180430202303107?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTQ2ODgxNDU=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

黄色为队首待组合的区域，Q4表示为MEM/CPU比为4的flavor队列，同理Q2和Q1分别是MEM/CPU比为2和1的flavor队列，优先队列按照flavor的core数排序，队首的三个flavor组合时，先选择1.5的组合方式，如果不存在1.5组合方式，选择2.5组合方式，否则选择3.0组合方式。

根据上述组合后，再采用背包组合方式暴力搜索三个个物理机的最大CPU利用率和最大MEM利用率。比如，针对flavor队列，选择一组物品，预先放置到General，High-performance， Large-Memory中，得到CPU和MEM的平均利用率，谁的平均利用率最大，就把这组物品放到该物理机下，且该组物品不会出现在下次的放置当中。

期间我们还尝试了FF贪心法，模拟退火等手段，都有不错的利用率，具体测试数据如下（总利用率为：18种flavor全开100的预测结果，100次随机测试为18种flavor，random(0,50)的测试结果）：

![这里写图片描述](https://img-blog.csdn.net/20180430203605236?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTQ2ODgxNDU=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

**放置trick**

如果不考虑预测的准确性，对利用率不足的物理机可以选择填塞操作，无非牺牲一点预测的准确性（实际可以通过调参自动拟合，影响微乎其微，却对利用率有较大的提升）。

思路一：每种flavor最多塞1个，以保证预测的准确性。这样，我们贪心选择利用率最差的物理机，尝试塞每种flavor，如果对利用率有所提升，则塞，否则尝试其他物理机。线下随机100次测试能够提升到96%。

思路二：在暴力三种物理机时，采用01背包选取利用率最高的物品组合，这里可以分成两个队列，一个为必须要塞的队列，第二个为或许要填塞的队列。作为第二种队列，每种flavor都设置上限20个，01背包组合优先选择第一队列中的物品，如果第一队列的物品无法满足利用率最大，则从第二队列中选择多种flavor进行填塞。这种策略可以让利用率理论上达到100%，但由于限制了上限，利用率取决于flavor的上限。设置为20时，随机100次测试平均利用率可达99%，近乎完美（当确定了虚拟机flavor的填塞个数后，再反向操作根据调参拟合线上数据或许是一种不错的策略）。

## 后记
最后感谢队友们为此次比赛的努力和付出。当我们得知线上分布基本不变时，依旧没有放弃任何博得头彩的机会，努力地算出每一个参数，虽然结局不如人意，过程依旧精彩。当然，还有在比赛过程中认识的战友们，和你们交流激发了很多灵感，模型不断地被完善。最后预祝进决赛的大佬们，努力加油干，再创佳绩。如有问题，随时联系：daimenSong@gmail.com




