#### R Retrievability 可检索性
首先考虑遗忘曲线，即 $R$ 表明学过一些东西之后，可被记忆检索的部分占总体学习资料的比值随时间的变化曲线。
下面这是 FSRS 各版本做出的一些公式：
$$
\begin{align}
R &= 0.9^{\frac{t}{S}} \\
R &= (1 + \frac{t}{9S})^{-1} \\
R &= (1 + \frac{19}{81} \cdot \frac{t}{S})^{-0.5}
\end{align}

$$
发现在 v4 之后，我们使用了一个幂函数而不是指数函数来拟合 $R$，我们的研究知道 $R$ 是指数变化的。但其实我们发现用一个幂函数来拟合一些复杂指数函数的和，比直接用指数函数更好。
#### S, Stability 稳定性
最主要计算稳定性的公式为 
$$
S'(D,S,R,G) = S \cdot (1 + w_{15} \cdot w_{16} \cdot e^{w_{8}} \cdot (11 - D) \cdot S^{-w_{9}} \cdot (e^{w_{10} \cdot (1 - R)} - 1))
$$
$G$ 是分数的意思，它会影响 $w_{15}, w_{16}$
简化一下该公式，就变成了：
$$
S'(D,S,R) = S \cdot SInc
$$
其中 SInc 是更新参数，它代表了如果本次复习比较成功，那么就让 $S$ 比原来更大，反之亦然。
先不考虑 $G$，则 SInc 为：
$$
SInc = 1 + f(D) \cdot g(S) \cdot h(R)
$$
首先，$f(D) = 11 - D$，这是指难度越大，记忆的稳定性增长的就越慢。
$g(S) = S^{-w_{9}}$，这是指记忆稳定后，就很难再变的更稳定。
$h(R) = (e^{w_{10} \cdot (1 - R)} - 1)$
w_{15}，这是指忘记的越多，复习效果就越好。
而 $e^{w_{8}}$ 是一个放缩系数，而 $w_{15}$ 为 1 时，表明学习结果为 Good 或 Easy，小于 1 时，表明学习结果为 Hard。而 $w_{16}$ 为 1 时，表明学习结果为 Good 或 Hard，大于 1 时，表明学习结果为 Easy.
当用户点击再学一次后，就会出现如下情况：
$$
S'(D,S,R) = \min(S, w_{11} \cdot D^{-w_{12}} \cdot ((S + 1)^{w_{13}} - 1) \cdot e^{w_{14}(1 - R)})
$$
注意这里的需要和原来的 S 取一个最小值，防止其超过原来的 S。
#### D, Difficulty 难度
一个卡片的首次难度为：
$$
D_{0}(G) = w_{4} - w_{5} \cdot (G - 3)
$$
这里的 G 是分数，Again = 1, Hard = 2, Good = 3, Easy = 4，我们使用移动均值来更新 $D$，即：
$$
D'(D, G) = w_{7} \cdot w_{4} + (1 - w_{7}) \cdot (D - w_{6} \cdot (G - 3))
$$

#### 库
`due`: 表示卡片的下次复习日期。这个字段用于确定卡片何时需要再次复习。

`stability`: 表示卡片的稳定性，即记忆的持久性。稳定性越高，表示记忆越牢固，复习间隔可以越长。

`difficulty`: 表示卡片的难度，即记忆的难易程度。难度越高，表示记忆越困难，复习间隔需要越短。

`elapsed_days`: 表示自上次复习以来经过的天数。这个字段用于计算卡片的遗忘曲线和复习间隔。

`scheduled_days`: 表示计划的复习间隔天数。这个字段用于确定卡片在未来的哪一天需要进行复习。

`reps`: 表示卡片的复习次数。这个字段用于记录卡片已经复习了多少次。

`lapses`: 表示卡片的遗忘次数。这个字段用于记录卡片在复习过程中被遗忘了多少次。

`state`: 表示卡片的状态。状态可以是新卡片（New）、学习中（Learning）、复习中（Review）或重新学习中（Relearning）。

`last_review`: 表示卡片的上次复习日期。这个字段用于计算卡片的遗忘曲线和复习间隔。

#### Forgetting Curve and Spacing Effect
考虑 $e_{i}$ 表明第 $i$ 次的记忆行为：
$$
e_{i} := (w, \boldsymbol{\Delta t}_{1:i-1}, \boldsymbol{r}_{1:i-1}, \Delta t_{i}, p_{i}, N)
$$
其中 $w$ 表明单词，$\boldsymbol{\Delta t}_{1:i-1}$ 表明前 $i - 1$ 次的记忆间隔，$\boldsymbol{r}_{1:i-1}$ 表明前 $i - 1$ 次是否回忆起了该单词，$\Delta t_{i}$ 表明第 $i$ 次的记忆间隔，$p_{i}$ 表明这一次回忆起该单词的概率，$N$ 为样本总数。
我们可以通过第一次学习该单词后，有能回忆起该单词人的比例作为该单词的难度系数，划分了 10 个难度等级 $d$：
$$
e_{i} := (d, \boldsymbol{\Delta t}_{1:i-1}, \boldsymbol{r}_{1:i-1}, \Delta t_{i}, p_{i}, N)
$$
我们使用指数遗忘曲线来拟合记忆过程 $p_{i} = 2^{-\frac{\Delta t_{i}}{h_{i}}}$，最终得到一个记忆过程的半衰期 $h_{i}$。（即遗忘概率达到 0.5 时，需要间隔多久）
$$
e_{i} := (d, \boldsymbol{\Delta t}_{1:i-1}, \boldsymbol{r}_{1:i-1}, \Delta t_{i}, h_{i}, N)
$$
最终根据数据计算出观察给定 $d, \boldsymbol{\Delta t}_{1:i-1}, \boldsymbol{r}_{1:i-1}, N$，控制 $\Delta t_{i}$ 得到的 $h_{i}$，我们观察到，一次成功的回忆就会延长半衰期。记忆巩固的效果随着复习间隔的延长而增强。
##### DHP Difficulty-Halflife-P(recall)
由于新时代的神经网络模型不具备可解释性，我们选择了更为原始的马尔可夫链来建模。
我们将记忆时间序列压缩成状态变量和状态方程。有四个状态变量：$h, p, r, d$。
我们发现当 $r_{i} = 1$ 时，$h_{i} > h_{i-1}$，而总有 $h_{i} > 0$。我们将这两个约束考虑进来，有：
$$
h_{i}  = [h_{i-1} \cdot (\exp(\boldsymbol{\theta}_{1}^{\mathrm{T}} \boldsymbol{x}_{i}) + 1),\exp(\boldsymbol{\theta}_{2}^{\mathrm{T}} \boldsymbol{x}_{i}))] \cdot [r_{i}, 1-r_{i}]^{\mathrm{T}}
$$
其中 $x_{i} = [\log d_{i-1}, \log h_{i-1}, \log(1 - p_{i})]$.
我们将一次成功的回忆就会延长半衰期体现在难度的变化上，每回忆成功一次，难度就会减小。
$$
d_{i} = [d_{i-1}, d_{i-1} + \theta_{3}] \cdot [r_{i}, 1 - r_{i}]^{\mathrm{T}}
$$
最终我们可以将其化为：
$$
\begin{bmatrix}
h_{i} \\
d_{i}
\end{bmatrix} =
\begin{bmatrix}
h_{i-1} \cdot (\exp(\boldsymbol{\theta}_{1}^{\mathrm{T}} \boldsymbol{x}_{i}) + 1) & \exp(\boldsymbol{\theta}_{2}^{\mathrm{T}} \boldsymbol{x}_{i})) \\
d_{i-1} & d_{i-1} + \theta_{3}
\end{bmatrix}
\begin{bmatrix}
r_{i} \\
1 - r_{i}
\end{bmatrix}
$$其中 $x_{i} = [\log d_{i-1}, \log h_{i-1}, \log(1 - p_{i})], r_{i} \sim B(p_{i}), p_{i} = 2^{\frac{\Delta t_{i}}{h_{i - 1}}}, h_{1} = -1 / \log_{2}(0.925-0.05 \cdot d_{0})$.
##### optimal scheduling
$h$ 表明了记忆的强度，复习的次数和每次复习的间隔即为学习的代价。
目标：以最小的记忆成本获取一定量的记忆资料以达到目标半衰期。我们只需要考虑单个记忆资料即可。
不难发现 $h_{i}$ 和 $d_{i}$ 只和 $h_{i-1}, d_{i-1}, p_{i}$ 有关