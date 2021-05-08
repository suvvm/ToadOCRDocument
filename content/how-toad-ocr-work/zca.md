---
title: "对图像进行ZCA白化"
date: 2021-05-07T17:57:33+08:00
draft: false
---

## 一、理论讲解

### 一、PCA解决的问题

从一个简单的数据集开始

#### 1、当只有属性1时

|           | SampleA | SampleB | SampleC | SampleD | SampleE | SampleF |
| --------- | ------- | ------- | ------- | ------- | ------- | ------- |
| **Attr1** | 10      | 11      | 8       | 3       | 2       | 1       |

可以将数据绘制在一个一维的坐标轴上

![img](/images/PCA_1.png)

这里可以发现针对属性1样本A、B、C有相对较高的值，样本D、E、F有相对较低的值。通过上图可以发现，样本A、B、C彼此之间的相似性要比它们与D、E、F高。

#### 2、如果测量两个属性

|           | SampleA | SampleB | SampleC | SampleD | SampleE | SampleF |
| --------- | ------- | ------- | ------- | ------- | ------- | ------- |
| **Attr1** | 10      | 11      | 8       | 3       | 2       | 1       |
| **Attr2** | 6       | 4       | 5       | 3       | 2.8     | 1       |

可以将数据绘制在一个二维的坐标轴上

![img](/images/PCA_2.png)

在张图中，属性1是x轴，属性2是y轴。通过观察上图依旧可以得出，样本A、B、C在右上侧聚集，样本D、E、F在左下侧聚集。

#### 3、如果测量三个属性

|           | SampleA | SampleB | SampleC | SampleD | SampleE | SampleF |
| --------- | ------- | ------- | ------- | ------- | ------- | ------- |
| **Attr1** | 10      | 11      | 8       | 3       | 2       | 1       |
| **Attr2** | 6       | 4       | 5       | 3       | 2.8     | 1       |
| **Attr3** | 12      | 9       | 10      | 2.5     | 1.3     | 2       |

则需要在上图的基础上添加z轴，使之成为三维坐标轴

![img](/images/PCA_3.png)

#### 4、如果要测量四个属性

|           | SampleA | SampleB | SampleC | SampleD | SampleE | SampleF |
| --------- | ------- | ------- | ------- | ------- | ------- | ------- |
| **Attr1** | 10      | 11      | 8       | 3       | 2       | 1       |
| **Attr2** | 6       | 4       | 5       | 3       | 2.8     | 1       |
| **Attr3** | 12      | 9       | 10      | 2.5     | 1.3     | 2       |
| **Attr4** | 5       | 7       | 6       | 2       | 4       | 7       |

则无法再绘制数据。

​	四个属性需要四个维度，而PCA则是为了解决上述问题而出现的，使用PCA可以将三维、四维及其以上的更多数据降维绘制二维的PCA图，并告知使用者哪个属性对数据聚类最有价值。

![img](/images/PCA_4.png)

### 二、PCA原理解析

​	如果要了解PCA功能以及工作原理，需要先回归只有两个属性的数据集。

|           | SampleA | SampleB | SampleC | SampleD | SampleE | SampleF |
| --------- | ------- | ------- | ------- | ------- | ------- | ------- |
| **Attr1** | 10      | 11      | 8       | 3       | 2       | 1       |
| **Attr2** | 6       | 4       | 5       | 3       | 2.8     | 1       |

#### 1、计算中心点

​	首先，计算样本绘制的二维图像的中心点 ，即针对对应坐标轴所代表的每个样本的属性值求和后除以样本数量。
$$
m_x = \frac{\sum_{S=1}^nA_{1i}}{n} \\\\ m_y = \frac{\sum_{S=1}^nA_{2i}}{n}
$$


可以得到G(5.83,3.63)

![img](/images/PCA_6.png)

移动数据，使得中心点G落在坐标轴原点上。

![img](/images/PCA_7.png)



|           | SampleA | SampleB | SampleC | SampleD | SampleE | SampleF |
| --------- | ------- | ------- | ------- | ------- | ------- | ------- |
| **Attr1** | 4.17    | 5.17    | 2.17    | -2.83   | -3.83   | -4.83   |
| **Attr2** | 2.37    | 0.37    | 1.37    | -0.63   | -0.83   | -2.63   |



#### 2、计算PC1

​	绘制一条穿过原点的随机直线，并旋转这条直线，使其在仍然穿过原点的情况下尽可能的拟合数据。

![img](/images/PCA_8.png)

判定拟合度高低，为了量化穿过原点的直线拟合数据的程度，首先将所有数据投影到目标直线上

![img](/images/PCA_9.png)

​	之后便可以测量各个样本点到目标直线的距离，并尝试找到使所有样本点到直线距离之和**最小**的直线，或者也可以尝试寻找所有投影点到原点的距离最之和大的直线，原因是当直线与数据更加拟合的时候，样本点到直线距离整体上会减小，而投影点到原点的整体距离则会增加。

​	针对每个单独的样本点来说，可以以原点、样本点、投影点，三点组成一个直角三角形，通过勾股定理可以得知，样本点到直线距离(边a)和投影点到原点的距离(边b)是逆相关的且$$ c^2 = a^2 + b^2 $$。样本点到原点的距离(边c)是不变的。所以PCA可以选择最小化样本与直线的距离或最大化投影到原点的距离。

​	不过显然投影点到原点更加方便计算，所以可以采用最大化投影点到原点的距离(d)的平方和($ SS_d $)。
$$
SS_d = \sum_{s=1}^nd_s^2
$$


​	这条线可以看作一条特殊的拟合直线。

##### PCA拟合直线

我们将输入数据  $ \left( \begin{array} {ccc} 4.17 & 5.17 & 2.17 & -2.83 & -3.83 & -4.83  \\\\  2.37 & 0.37 & 1.37 & -0.63 & -0.83 & -2.63 \end{array} \right) $  记为矩阵P

因为直线过原点只需考虑计算直线斜率，那么如果知道了直线的单位法向量$ \vec n(n_1, n_2) $   ，对于任意样本点 $ (x_s, y_s) $ 该点的坐标向量与直线法向量  的内积正好等于该点到直线的距离(同向为正, 反向为负)，即
$$
d_s =  \left(
\begin{array}
{ccc}
x_s\\\\
y_s
\end{array}
\right) \cdot \vec n
$$


将距离带入距离公式 $ SS_d = \sum_{s=1}^nd_s^2 $

得到新的公式
$$
SS_d = \sum_{s=1}^n \left|\left( \begin{array} {ccc} x_s \\\\ y_s \end{array} \right) \cdot \vec n \right|^2 = \sum_{s=1}^n \vec n^T \left( \begin{array} {ccc} x_s \\\\ y_s \end{array} \right) （\begin{array} {ccc} x_s & y_s \end{array}）\vec n = \vec n^T \sum_{s=1}^n \left( \begin{array} {ccc} x_s \\\\ y_s \end{array} \right) （\begin{array} {ccc} x_s & y_s \end{array}）\vec n
$$


其中 $ \sum_{s=1}^n \left( \begin{array} {ccc} x_s \\\\ y_s \end{array} \right) （\begin{array} {ccc} x_s & y_s \end{array}）= PP^T $   所以 $ SS_d = \vec n^TPP^T\vec n $  而 $ \frac{1}{n - 1}  PP^T $ 就是原始数据的协方差矩阵。



对矩阵P进行SVD奇异值分解
$$
P = U\Sigma V^T
$$

$$
PP^T	= U\Sigma V^T(U\Sigma V^T)^T
$$

$$
PP^T = U\Sigma V^TV\Sigma U^T
$$

$$
PP^T = V\Sigma \Sigma V^T \qquad or \qquad PP^T = U\Sigma \Sigma U^T
$$

$$
P^TPV = V\Sigma \Sigma \qquad or \qquad PP^TU = U\Sigma \Sigma
$$

$ PP^T $ 的特征向量
$$
P^TP \vec v_1 = \lambda_1 \vec v_1 \qquad P^TP \vec v_2 = \lambda_2 \vec v_2
$$
 所以
$$
V = \vec v_1 \vec v_2
$$
而
$$
\Sigma = \left( \begin{array} {ccc} \sigma_1 & 0 \\\\ 0 & \sigma_2 \end{array} \right)
$$

$$
\Sigma \Sigma = \left( \begin{array} {ccc} \lambda_1 & 0 \\\\ 0 & \lambda_2 \end{array} \right)
$$

所以
$$
\lambda_1 = \sigma_1^2 \qquad \lambda_2 = \sigma_2^2
$$
直线的法向方向应该为$ PP^T $  最小特征值对应的特征方向。

得到了直线的法向量，便可以获得直线的斜率
$$
PP^T = \left( \begin{array} {ccc} 4.17 & 5.17 & 2.17 & -2.83 & -3.83 & -4.83  \\\\  2.37 & 0.37 & 1.37 & -0.63 & -0.83 & -2.63 \end{array} \right)
\left( \begin{array} {ccc} 2.37 & 4.17 \\\\ 0.37 & 5.17 \\\\ 1.37 & 2.17 \\\\ -0.63 & -2.83 \\\\ -0.83 & -3.83 \\\\ -2.63 & -4.83 \end{array} \right) =\left( \begin{array} {ccc} 32.43 & 94.83 \\\\ 15.63 & 32.43 \end{array} \right)
$$

$$
\lambda_1 = 65.52 对应特征向量 = \left( \begin{array} {ccc} 2.64 \\\\ 1 \end{array} \right)  \qquad \lambda_2 = -0.66 对应特征向量 = \left( \begin{array} {ccc} -2.64 \\\\ 1 \end{array} \right)
$$

所以斜率为0.264

###### 最小二乘数法无法计算PCA拟合直线

​	使用截距式表示上述直线，直线为ax + by = 0，由于上方直线过原点，所以截距b为0
$$
d = \frac{|ax_s + by_s|}{\sqrt {a^2 + b^2}}
$$


​	我们要使表达式 $ \sum_{s=1}^n[\frac{|ax_s + by_s|}{\sqrt {a^2 + b^2}}]^2 = \sum_{s=1}^n\frac{(ax_s + by_s)^2}{a^2 + b^2} $  的值最小

​	首先对a和b求偏导：
$$
\frac{\sigma}{\sigma a}\sum_{s=1}^n\frac{(ax_s + by_s)^2}{a^2 + b^2} \\\\
= \sum_{s=1}^n\frac{2x_s(ax_s + by_s)(a^2 + b^2) - 2a(ax_s + by_s)^2}{(a^2 + b^2)^2}\\\\
= \sum_{s=1}^n\frac{2x_s^2a^3 + 2bx_sy_sa^2 + 2x_s^2ab^2 + 2b^3x_sy_s - 2a^3x_s^2 - 4a^2bx_sy_s - 2ab^2y_s^2}{a^4 + 2a^2b^2 + b^4} \\\\
= \sum_{s=1}^n\frac{-2bx_sy_s a^2 + 2b^2(x_s^2 - y_s^2)a + 2b^3x_sy_s}{a^4 + 2a^2b^2 + b^4}
$$

$$
\frac{\sigma}{\sigma b}\sum_{s=1}^n\frac{(ax_s + by_s)^2}{a^2 + b^2} \\\\
= \sum_{s=1}^n\frac{2y_s(ax_s + by_s)(a^2 + b^2) - 2b(ax_s + by_s)^2}{(a^2 + b^2)^2} \\\\
= \sum_{s=1}^n\frac{2a^3x_sy_s + 2a^2y_s^2b + 2ax_sy_sb^2 + 2y_s^2b^3 - 2a^2x_s^2b - 4ax_sy_sb^2 - 2y_s^2b^3}{a^4 + 2a^2b^2 + b^4} \\\\
= \sum_{s=1}^n\frac{-2ax_sy_sb^2 + 2a^2(y_s^2 - x_s^2)b + 2a^3x_sy_s}{a^4 + 2a^2b^2 + b^4}
$$



得到下方方程组
$$
\begin{cases}
\sum_{s=1}^n\frac{-2ax_sy_sb^2 + 2a^2(y_s^2 - x_s^2)b + 2a^3x_sy_s}{a^4 + 2a^2b^2 + b^4} = 0 \\\\
\sum_{s=1}^n\frac{-2bx_sy_s a^2 + 2b^2(x_s^2 - y_s^2)a + 2b^3x_sy_s}{a^4 + 2a^2b^2 + b^4} = 0
\end{cases}
$$
xy共轭，无法根据将上述方程组得到a与b的表达式，所以无法继续解得拟合直线。

​	最终得到的这条直线被称为主成份1(PC1) y = 0.264x，斜率为0.264

![img](/images/PCA_10.png)

​	可以通过PC1的斜率得知属性1和属性2的线性组合为2.64:1，可以根据勾股定理求的属性1为3.1，属性2位1的样本点在PC1上，其到原点距离为，使用SVD计算PCA时将上述PC1方向样本缩放做标准化，使3.1作为单位长度等于1。最终得到属性1为 $ 2.64 \div 2.82 = 0.94 $属性2为 $ 1 \div 2.82 = 0.35 $ 最终得到PC1的特征向量$ \vec a = (0.94, 0.35) $，0.94与0.35则作为属性1与属性2的荷载得分。而上方的 $ SS_d $ 也就是PC1的特征值。

#### 3、计算PC2

​	经过原点做一条垂直于PC1的直线

![img](/images/PCA_11.png)

​	同样适用SVD计算得到PC2的特征向量$ \vec b = (-0.35, 0.94) $ -0.35与0.94则作为属性1与属性2的荷载得分。

#### 4、获得最终PCA图

​	将PC1与PC2旋转至水平与垂直，作为新的坐标系

![img](/images/PCA_12.png)

得到新的一组样本点



|           | SampleA' | SampleB' | SampleC' | SampleD' | SampleE' | SampleF' |
| --------- | -------- | -------- | -------- | -------- | -------- | -------- |
| **Attr1** | 4.74     | 4.97     | 2.51     | -2.87    | -3.88    | -5.45    |
| **Attr2** | 0.74     | -1.49    | 0.51     | 0.41     | 0.58     | -0.75    |

$$
SS_{dpc1} = \sum_{S'=1}^nd_{a1S'}^2 \\\\
					= 4.74^2 + 4.97^2 + 2.51^2 + (-2.87)^2 + (-3.88)^2 + (-5.45)^2 \\\\
					= 106.46 \\\\
SS_{dpc1} = \sum_{S'=1}^nd_{a2S'}^2 \\\\
					= 0.74^2 + (-1.49)^2 + 0.51^2 + 0.41^2 + 0.58^2 + (-0.75)^2 \\\\
					= 4.09
$$

可以用上方的平方和除以样本点数量减一获得PC1与PC2围绕原点的的差异
$$
V_{PC1} = \frac{SS_{dpc1}}{6 - 1} = \frac{93.25}{5} = 21.29 \\\\
V_{PC1} = \frac{SS_{dpc2}}{6 - 1} = \frac{4.05}{5} = 0.82
$$
最终通过计算得到碎石图所需数值
$$
PC1 = \frac{18.65}{18.65 + 0.81} = 0.962 = 96.2\% \\\\
PC2 = \frac{18.65}{18.65 + 0.81} = 0.038 = 3.8\% \\\\
$$


![img](/images/PCA_13.png)

可以发现PC1可以96.2%的表示特征数据，可以使用PC1作为主轴将上述数据降低为一维矩阵

### 三、ZCA白化

​	**ZCA白化必须先进行PCA**

​	首先将目标n维特征映射到k维上，但此时得到的数据已经被降为k纬度，但是对于图像处理来说，输入的特征矩阵就是像素矩阵，我们希望原图像维度不变，这时则需要对PCA处理完的数据进行旋转，将维度旋转回原先的n维，使得最终得到的数据尽可能的接近原始数据。

​	即  $ X_{ZCA} = UX_{PCA} $

## 二、代码实现

``` go
// ZCA 零相位成分分析(zca)函数
// zca函数作用类似与主成分分析(pca)
// 首先进行pca主成分分析，将图像像素点张量作为特征进行数据降维
// 之后在将pca降维后的数据进行旋转，使得最终数据维度与原始数据维度相同
// 具体理论见toadOCRDocument中关于ZCA白化的描述
//
// 入参
//	data tensor.Tensor	// 图像像素点张量
//
// 返回
//	tensor.Tensor		// ZCA白化后的张量
//	error				// 错误信息
func ZCA(data tensor.Tensor) (tensor.Tensor, error) {
	var dataTranspose, dataClone, sigma tensor.Tensor	// data转置 data备份 矩阵Σ(定义详见上述链接)
	dataClone = data.Clone().(tensor.Tensor)
	if err := Centralization(dataClone); err != nil {	// 对备份数据张量进行中心化操作
		return nil, err
	}
	var err error
	if dataTranspose, err =  tensor.T(dataClone); err != nil {	// 求得中心化完成后的矩阵的转置
		return nil, err
	}
	if sigma, err = tensor.MatMul(dataTranspose, dataClone); err != nil {	// 求矩阵Σ
		return nil, err
	}
	cols := sigma.Shape()[1]	// 获取矩阵Σ列数
	// 对矩阵Σ逐元素处以列数-1，并覆盖原矩阵Σ，最终得到协方差矩阵
	if _, err := tensor.Div(sigma, float64(cols-1), tensor.UseUnsafe()); err != nil {
		return nil, err
	}
	// S特征值矩阵 U特征向量矩阵 V=U‘ 奇异值分解后的V这里用不到
	S, U, _, err := sigma.(*tensor.Dense).SVD(true,true)	// 对协方差矩阵执行奇异值分解
	if err != nil {
		return nil, err
	}
	var diag, UTranspose, tmp tensor.Tensor
	// 规则化项epsilon取0.1 求平方根倒数得对角阵diag
	if diag, err = S.Apply(InverseSqrt(0.1), tensor.UseUnsafe()); err != nil {
		return nil, err
	}
	diag = tensor.New(tensor.AsDenseDiag(diag))
	if UTranspose, err = tensor.T(U); err != nil {	// 计算U的转置
		return nil, err
	}
	if tmp, err = tensor.MatMul(U, diag); err != nil {
		return nil, err
	}
	if tmp, err = tensor.MatMul(tmp, UTranspose); err != nil {
		return nil, err
	}
	if err = tmp.T(); err != nil {
		return nil, err
	}
	return tensor.MatMul(data, tmp)
}
```

