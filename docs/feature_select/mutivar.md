

> 数据

```python
from kivi.Dataset import *
from kivi.FeatureAnalysis import VIF
# 测试数据
df_bank = Dataset.BankData()
df_bank = df_bank.select_dtypes(['int64', 'float64']).copy()
```

### 1. 相关系数 `spearman`

> 说明

Spearman 秩相关系数是两个数据集之间关系单调性的非参数度量。
与 Pearson 相关性不同，Spearman 相关性不假设两个数据集都是正态分布的。
Spearman 系数在 -1 和 +1 之间变化，0 表示没有相关性。
-1 或 +1 的相关性意味着精确的单调关系。正相关意味着随着 x 的增加，y 也会增加。
负相关意味着随着 x 增加，y 减少。

p 值粗略地表示不相关系统生成具有 Spearman 相关性的数据集的概率，
p 值并不完全可靠，但对于大于 500 左右的数据集可能是合理的。

> 参数

- `:param df:` `DataFrame` 需要统计相关系数的数据。
- `:param columns:` 需要统计相关系数的数据列名。

> 示例

```python
df_corr = spearmanr(df, columns)
```

### 2. 多重共线性检测与处理

> 理论分析

- 不论是一般回归，逻辑回归，或生存分析，都要同时考虑多个预测因子，如果各个自变量x之间有很强的线性关系，就无法固定其他变量，
也就找不到x和y之间真实的关系了，因此多重共线性是不可避免需要面对的。
- 在构造预测模型时如何处理多重共线性是一个比较微妙的议题。既不能不加控制，又不能一刀切，认为凡是多重共线性就应该消除。

假设有$k$个自变量的多元线性回归模型: 
$$
\begin{aligned}
y & = \theta_0 + \theta_1 x_1 + ... + \theta_k x_k +\varepsilon \\\\
& = X\theta + \varepsilon \\\\
\varepsilon & = N(0, \sigma^2) \\\\
\hat{\theta} & = (X^T X)^{-1}X^Ty 
\end{aligned}
$$
该求解公式唯一的条件是矩阵$X$是列满秩的，不然会有无穷多解，
当各变量之间存在共线性问题，即各变量之间存在部分线性相关时，例如：
$$x_3 = x_2 + x_1 + \varepsilon$$
易知此时$X$近乎是不满秩的，$X^TX$近乎是奇异的，$X$的最小奇异值会非常小。

> 多重共线性解决手段

多重共线性是普遍存在的，通常情况下，如果共线性情况不严重（VIF<5），不需要做特别的处理。
如存在严重的多重共线性问题，可以考虑使用以下几种方法处理：

**1. 手动移除出共线性的变量**

先做相关分析，如果发现某两个自变量$X$（解释变量）的相关系数值大于$0.7$，则移除掉一个自变量（解释变量），然后再做回归分析。
此方法是最直接的方法，但有的时候我们不希望把某个自变量从模型中剔除，这样就要考虑使用其他方法。

**2. 逐步回归法**

让系统自动进行自变量的选择剔除，使用逐步回归将共线性的自变量自动剔除出去。此种解决办法有个问题是，
可能算法会剔除掉本不想剔除的自变量，如果有此类情况产生，此时最好是使用岭回归进行分析。

**3. 增加样本容量**

增加样本容量是解释共线性问题的一种办法，但在实际操作中可能并不太适合，原因是样本量的收集需要成本时间等。

**4. 岭回归**

上述第1和第2种解决办法在实际研究中使用较多，但问题在于，如果实际研究中并不想剔除掉某些自变量，某些自变量很重要，
不能剔除。此时可能只有岭回归最为适合了。岭回归是当前解决共线性问题最有效的解释办法。

> 其他问题

**多重共线性与相关性的区别**

相关性，是指两个变量的关联程度。一般地，从散点图上可以观察到两个变量有以下三种关系之一：两变量正相关、负相关、不相关。
如果一个变量高的值对应于另一个变量高的值，相似地，低的值对应低的值，那么这两个变量正相关。

**多重共线性不需要处理的场景**

1. 多重共线性是普遍存在的，轻微的多重共线性问题可不采取措施，如果`VIF`值大于`10`说明共线性很严重，这种情况需要处理，
如果`VIF`值在`5`以下不需要处理，如果`VIF`介于`5~10`之间视情况而定。

2. 如果模型仅用于预测，则只要拟合程度好，可不处理多重共线性问题，存在多重共线性的模型用于预测时，往往不影响预测结果。

#### 2.1 方差膨胀因子 VIF

##### 2.1.1 `VIF` 方差分析

> `VIF` 检测

从统计学的角度来看共线性。可以证明参数$\theta$的协方差矩阵为

$$
Var(\hat{\theta}) = Var(\hat{\theta} - \theta) = Var[(X^TX)^{-1}X^T \varepsilon]
$$

又对任意的常数矩阵A和随机变量x有

$$
Var(Ax) = A \cdot Var(x) \cdot A^T
$$

代入上式即可得

$$
Var(\hat{\theta}) = \sigma^2(X^TX)^{-1}
$$

具体到每个参数，有：

$$
Var(\hat{\theta}_i) = \frac{\sigma^2}{(n-1)Var(x_j)} \cdot \frac{1}{1-R_i^2}
$$

其中$Ri2$是将第i个变量$x_i$作为因变量，其他$k-1$个变量作为自变量进行线性回归获得的$R2$，且令

$$
VIF_i = \frac{1}{1-R_i^2}
$$

为方差膨胀因子(variance inflation factor，VIF)。当$R_i^2 \sim 1$时，
即当第$i$个变量和其他变量之间存在线性关系时，`VIF`趋于无穷大。
所以`VIF`的大小反应了变量的共线性程度。一般地，当`VIF`大于`5`或`10`时，
认为模型存在严重的共线性问题。

> 共线性的影响

同时考虑参数显著性检验的`t`统计量：

$$
t = \frac{\hat{\theta}_i}{std(\hat{\theta}_i)} \sim t(n-k-1)
$$

当存在共线性时，参数的标准差偏大，相应的`t 统计量`会偏小，
这样容易淘汰一些不应淘汰的解释变量，使统计检验的结果失去可靠性。

另外考虑线性回归的残差

$$
\hat{\varepsilon} = y - X\hat{\theta} = M\varepsilon
$$

其中$M$是一个投影矩阵，且满足

$$
M = I - X(X^TX)^{-1}X^T
$$

易证明

$$
||\hat{\varepsilon}||^2_2 = \varepsilon^T M \varepsilon \le ||M||_F^2 \cdot ||\varepsilon||_2^2 = (n-k) ||\varepsilon||^2_2
$$

而矩阵$M$的范数与$X$的条件数毫无关系，
于是可以得出共线性并不影响模型的训练精度。
但是对于泛化精度，由于参数的估计已经不准确啦，
所以泛化误差肯定要差些，具体差多少，很难用公式表示出来。

共线性问题对线性回归模型有如下影响：
- 参数的方差增大；
- 难以区分每个解释变量的单独影响；
- 变量的显著性检验失去意义；
- 回归模型缺乏稳定性。样本的微小扰动都可能带来参数很大的变化；
- 影响模型的泛化误差。

##### 2.1.2 `VIF.vif`: 计算某变量与其他变量的`VIF`

> 参数

- `:param exog:` DataFrame。
- `:param exog_idx:` 需要计算变量`VIF`的`ID`。

> 示例

```python
VIF.vif(df_bank.values, 0)
```

> 结果

```bash
>>> output Data:
>>> 5.0928601913135125
```

##### 2.1.3 `VIF.StatsTableVIF`: 依次计算全表`VIF`

> 参数

- `:param df[pandas.DataFrame]:` 全量需要计算`VIF`的特征`DataFrame`

> 示例

```python
vif_dict = VIF.StatsTableVIF(df_bank)
pd.DataFrame(vif_dict)
```

> 结果

```bash
>>> output Data:
>>>     feature  vif_value
>>> 0       age   5.092860
>>> 1   balance   1.232178
>>> 2       day   4.058272
>>> 3  duration   2.292100
>>> 4  campaign   1.832987
>>> 5     pdays   1.737051
>>> 6  previous   1.663161
>>> 7    target   1.369417
```

##### 2.1.4 `VIF.LowVIFFeatures`: 依据`VIF`设定阈值，选择最小`VIF`特征组合

> 参数

- `:param df[pandas.DataFrame]:` 特征
- `:param thresh[int]:` `VIF`阈值
- `:return [pandas.DataFrame]:` 指标名称列表，筛选明细

> 示例

```python
df_vif = VIF.LowVIFFeatures(df_bank, 3)
df_vif
```

> 结果

```bash
>>> output Data:
>>>     feature    step_1    step_2
>>> 0       age  5.092860       NaN
>>> 1   balance  1.232178  1.180269
>>> 2       day  4.058272  2.425762
>>> 3  duration  2.292100  2.040717
>>> 4  campaign  1.832987  1.738374
>>> 5     pdays  1.737051  1.704357
>>> 6  previous  1.663161  1.661306
>>> 7    target  1.369417  1.369348
```

##### 2.1.5 `VIF.SparkVIF():`: PySpark 计算变量`VIF`

##### 2.1.6 `VIF.SparkTableVIF():`: PySpark 计算全表`VIF`

#### 2.2 扰动分析

> **扰动分析**: 对于一个方程或者系统而言，当输入有一个非常微小的扰动时，
希望方程或系统的输出变化也非常微小，如果输出的变化非常大，且不能被控制，
那这个系统的预测就无效了。在矩阵计算中，这叫做扰动分析。

**【扰动分析定理】** 
设非奇异方阵$A$满足方程
$$
Ax = y
$$
它的精确解为 $x^\*$，当$A$存在一个小扰动时，假设$\hat{x}$是新方程的解：
$$
(A+\delta A)\hat{x} = y
$$
可以证明$x^*$的扰动满足：
$$
\frac{||\delta x||}{||\hat{x}||} \le k(A) \frac{||\delta A||}{||A||}
$$
其中$k(A) = ||A^{-1}|| \cdot ||A||$
是非奇异方阵的条件数，且此时矩阵范数等价于矩阵最大的奇异值，
即矩阵的条件数等价于`最大奇异值/最小奇异值`。

可以看到矩阵的条件数越大，扰动就越大，即$x$的求解值会变得非常不准确。
回到上面讲的线性回归问题，容易证明最小二乘法的解满足下面的正定方程：
$$
X^TX\hat{\theta} = X^T y
$$

此时

$$
k(X^TX) = \frac{\lambda_max(X^T X)}{\lambda_max(X^T X)} = \frac{\sigma_{max}^2(X)}{\sigma_{min}^2(X)}
$$

当方程有共线性问题时，$X$的最小特征值非常小，相应的，
上述的条件数会非常大。也就是说机器学习中的共线性问题实际上就是矩阵计算中的条件数问题。

从实际应用的角度:
- 一般若`K<100`，则认为多重共线性的程度很小
- `100<=K<=1000`，则认为存在一般程度上的多重共线性
- `K>1000`，则就认为存在严重的多重共线性

- 条件数`condition number`

```python
lr.cond_num
```

- 相关性分析: 检验变量之间的相关系数，参考: [相关系数](./descriptive_statistics/descriptive_statistics?id=相关系数示例)

----