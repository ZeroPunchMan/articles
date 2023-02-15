<script type="text/javascript" async="" src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-MML-AM_CHTML"> </script>

# Quaternion.Slerp 关于四元数的球形插值
很久之前就看到这个算法了,当年写过程序验证了插值,确实与我所理解的另一种算法结果相同,一直没有想清楚原理,最近看了PBRT上的说明,其中阐述了三维或二维的slerp,再扩展到四维并应用于四元数上,再结合网上搜索资料,个人感觉总无法把slerp与四元数的旋转插值对应起来,也许这是从更抽象的角度来看待这个问题吧.
为了自己能够理解这个算法为什么会得到正确的插值,最近用自己能理解的数学知识证明了这个算法,在此记录下来,也希望能帮助到那些像我一样数学不太好的朋友.

## 基于四元数的旋转
首先,四元数表示旋转的公式:

$$q = cos\alpha + sin\alpha*\vec{v}$$

其中$$\vec{v}$$为单位向量,则四元数q为单位四元数,对于一点$$p$$,$${p}'=qpq^{-1}$$ 代表$$p$$绕 $$\vec{v}$$ 旋转$$2\alpha$$角度后的位置

## 标准的Slerp

$$slerp(q_{1}, q_{2}, t) = \frac{sin((1-t)\theta)q_{1}+sin(t\theta)q_{2}}{sin\theta}$$

其中$$\theta$$为$$q_{1}$$和$$q_{2}$$的夹角,即$$cos\theta = dot(q_{1}, q_{2})$$

## 我理解的Slerp

那么对于从$$q_{1}pq_{1}^{-1}$$到$$q_{2}pq_{2}^{-1}$$,需要经过一个中间四元数为$$q_{m}$$,使得

$$q_{m}(q_{1}pq_{1}^{-1})q_{m}^{-1} = q_{2}pq_{2}^{-1}$$

所以这个中间四元数为

$$q_{m} = q_{2} * q_{1}^{-1}$$

把$$q_{m}$$写成旋转形式

$$q_{m} = cos\theta + sin\theta*\vec{v_{m}}$$

即从$$q_{1}$$到$$q_{2}$$需要绕$$v_{m}$$旋转$$2\theta$$角度,按照我的理解,$$slerp(q_{1}, q_{2}, t)$$就是绕$$v_{m}$$旋转$$t*2\theta$$角度,即如下形式:

$$slerp(q_{1}, q_{2}, t) = cos(t\theta) + sin(t\theta)*\vec{v_{m}}$$

为了方便区分,写成如下单位四元数的指数形式

$$cos(t\theta) + sin(t\theta)*\vec{v_{m}} = q_{m}^{t}$$

这样写的好处在于,$$\vec{v}$$相同的单位四元数相乘,就等于角度相加的结果.

## 证明

需要证明的是,以上两种算法,其实结果相同,即

$$slerp(q_{1}, q_{2}, t) = q_{m}^{t} * q_{1}$$

很容易验证(展开即可),$$q_{m}$$中的$$cos\theta$$与$$dot(q_{1}, q_{2})$$相等(所以Slerp和$$q_{m}$$都是写作$$\theta$$),且上式可以变形如下

$$slerp(q_{1}, q_{2}, t) * q_{1}^{-1} = q_{m}^{t}$$

把式子左边展开即可

$$left=\frac{sin((1-t)\theta)q_{1}+sin(t\theta)q_{2}}{sin\theta} * q_{1}^{-1}$$

$$=\frac{sin((1-t)\theta)}{sin\theta} + \frac{sin(t\theta)}{sin\theta}q_{2}*q_{1}^{-1}$$

带入$$q_{m} = q_{2} * q_{1}^{-1}$$

$$=\frac{sin((1-t)\theta)}{sin\theta} + \frac{sin(t\theta)}{sin\theta}q_{m}$$

$$=\frac{sin((1-t)\theta)}{sin\theta} + \frac{sin(t\theta)}{sin\theta}(cos\theta + sin\theta*\vec{v_{m}})$$

$$=\frac{sin((1-t)\theta)}{sin\theta} + \frac{sin(t\theta)cos\theta}{sin\theta} + sin(t\theta)*\vec{v_{m}}$$

$$=cos(t\theta) + sin(t\theta)*\vec{v_{m}} = q_{m}^{t}$$

证毕.

## 总结

很简单的计算，主要在于我理解的球形插值定义为$$q_{m}^{t}q_{1}$$,这里证明了与$$\frac{sin((1-t)\theta)q_{1}+sin(t\theta)q_{2}}{sin\theta}$$相等.