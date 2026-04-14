---
layout: default
---

<h1 align="center"> Shor 算法的 order finding 问题</h1>

#### 1. Problem definition

设整数 $x, N$ 满足 $1 \le x < N$ 且 $\gcd(x, N) = 1$。**$x$ 模 $N$ 的阶 (Order)** 定义为满足 $$x^r \equiv 1 \pmod N$$ 的小正整数 $r$ 。Order finding 被认为是经典计算中的困难问题，而 Shor 算法通过将其转化为量子相位估计，实现了多项式时间内的求解。

#### 2. Unitary operator & eigenstructure

定义受控模乘酉算符 $U_x$：

$$
U_x |y\rangle := |xy \bmod N\rangle, \quad y \in \{0, 1\}^n
$$

该算符的本征态 $\vert u_s\rangle$ 及其对应的本征值可构造如下：

$$
|u_s\rangle
=
\frac{1}{\sqrt r}
\sum_{k=0}^{r-1}
e^{-2\pi i sk/r}\,
|x^k \bmod N\rangle,
\qquad s=0,1,\dots,r-1.
$$

验证如下：

$$
\begin{aligned}
U_x\left|u_s\right\rangle
&= \frac{1}{\sqrt{r}} \sum_{k=0}^{r-1} \exp \left[\frac{-2 \pi i s k}{r}\right]\left|x^{k+1} \bmod N\right\rangle \\
&= \frac{1}{\sqrt{r}} \sum_{k^{\prime}=1}^r \exp \left[\frac{-2 \pi i s\left(k^{\prime}-1\right)}{r}\right]\left|x^{k^{\prime}} \bmod N\right\rangle \\
&= \exp \left[\frac{2 \pi i s}{r}\right] \frac{1}{\sqrt{r}} \sum_{k^{\prime}=1}^r \exp \left[\frac{-2 \pi i s k^{\prime}}{r}\right]\left|x^{k^{\prime}} \bmod N\right\rangle \\
&= \exp \left[\frac{2 \pi i s}{r}\right]\left|u_s\right\rangle
\end{aligned}
$$

因此，算符 $U_x$ 的相位信息 $\phi = s/r$ 包含了我们需要的阶 $r$。

#### 3. Quantum phase estimation

在实际操作中，制备单个本征态 $\vert u_s\rangle$ 需要预知 $r$，这构成了逻辑循环。Shor 算法利用了所有本征态的**均匀叠加态**恰好为初态 $\vert 1\rangle$ 的特性：

$$
\frac{1}{\sqrt r}\sum_{s=0}^{r-1}|u_s\rangle
=
|x^0 \bmod N\rangle
=
|1\rangle.
$$

通过 QPE 线路，系统的联合状态演化为：

$$
\begin{aligned}
\frac{1}{\sqrt{2^t}}
\sum_{j=0}^{2^t-1}
|j\rangle (U_x)^j |1\rangle 
&= \frac{1}{\sqrt{r2^t}}
\sum_{j=0}^{2^t-1}\sum_{s=0}^{r-1}
|j\rangle (U_x)^j |u_s\rangle \\
&= \frac{1}{\sqrt{r2^t}}
\sum_{s=0}^{r-1}\sum_{j=0}^{2^t-1}
e^{2\pi i js/r}
|j\rangle |u_s\rangle.
\end{aligned}
$$

此时，施加IQFT后，第一寄存器（$t$ 个辅助比特）对每个 $\vertu_s\rangle$ 分量编码了接近 $2^t\cdot s/r$ 的位置，测量第一寄存器将以高概率获得相位 $s/r$ 的最佳 $t$ 位二进制近似值。

#### 4. Classical processing：continued fractions

测量得到结果后，通过 Continued fractions algorithm  在多项式时间内恢复最接近的有理数 $s/r$：

- **理想情况：** 若 $\gcd(s, r) = 1$，则直接恢复出分母 $r$。
- **一般情况：** 若 $\gcd(s, r) > 1$，则仅能恢复出 $r$ 的一个因子。此时需重复运行算法，利用多次测得的分母取**最小公倍数 (LCM)** 来恢复真实的 $r$。



在容错量子计算资源估算中，Order finding 的开销主要集中在受控幂次算符的实现上：

$$
U_x^{2^0}, U_x^{2^1}, \dots, U_x^{2^{t-1}},
$$

这些算符最终都要分解成大量的 controlled modular multiplications。  这正是后续资源估算中 Toffoli count 和 T count 的主要来源。



---
<h1 align="center">资源优化的 RSA-2048 破解算法</h1>

*基于 RNS、截断累加与高度并行的硬件映射 (Gidney 2025 & Pinnacle 2026)*

#### 1. 传统 Shor 算法的资源瓶颈
在传统的 Shor 算法中，破解 RSA-2048 的核心算力消耗在于**受控模幂运算**（Controlled Modular Exponentiation, $a^x \pmod{N}$），其中 $N \approx 2^{2048}$。传统线路需要在模 $N$ 下直接维护和更新大整数，这导致必须分配若干个位宽为 $O(\log N)$ 的量子工作寄存器（Working Registers），带来了极其庞大的空间开销（Spatial Overhead）。

#### 2. Gidney (2025) 的核心突破：RNS 与截断累加
Gidney 算法放弃了直接在模 $N$ 下计算大整数乘法链，而是引入了**剩余数系统（Residue Number System, RNS）**。其核心 Insight 在于：
1. 利用中国剩余定理（CRT）将复杂的“大数乘法链”转化为“许多小贡献项的求和”。
2. 由于“求和”操作不会像乘法那样产生灾难性的误差放大，算法允许在累加时**丢弃低位，只保留高 $f$ 位**。
3. 这使得主工作寄存器的位宽从 $O(\log N)$ 骤降至常数量级的 $f = \Theta(\log \log N)$。
4. 最终通过近似周期发现（Approximate period finding）和频率基测量（Frequency basis measurement）提取所需周期。

#### 3. 算法推导：从模幂到截断求和

**Step 3.1: 构建剩余数系统 (RNS)**
我们将目标运算 $V = a^x \pmod{N}$ 改写为受控乘法链的形式：

$$
V = g^e \pmod{N} = \left( \prod_{k=0}^{m-1} M_k^{e_k} \right) \pmod{N}
$$

其中 $e_k$ 是指数寄存器 $e$ 的第 $k$ 位，$M_k = g^{2^k} \pmod{N}$，而 $m = O(\log N)$ 是受控乘法的总次数。

为了避免直接处理大数 $X := \prod_{k=0}^{m-1} M_k^{e_k}$，我们选取一组互质的小质数集合 $P = \{p_1, p_2, \dots, p_{\vert P\vert}\}$。为了确保 RNS 系统不溢出，要求这组质数的乘积满足：

$$
L = \prod_{i=1}^{|P|} p_i \geq N^m > X.
$$

**Step 3.2: 转化为求和形式 (CRT Contribution)**
计算 $X$ 在每个小质数下的余数 $r_j = X \bmod p_j$。同时定义 CRT 的正交基：

$$
u_j = \left(\frac{L}{p_j}\right) \cdot \text{MultiplicativeInverse}_{p_j}\left(\frac{L}{p_j}\right)
$$

满足 $u_j \equiv 1 \pmod{p_j}$ 且对其他质数 $u_j \equiv 0 \pmod{p_i}$ $(i \neq j)$。根据中国剩余定理，有 $X \equiv \sum_j r_j u_j \pmod{L}$。因此，目标值 $V$ 可表示为：$$V = \left( \sum_{j=1}^{|P|} r_j u_j \right) \bmod L \bmod N$$。将每个小余数展成二进制形式 $r_j = \sum_{k=0}^{\ell-1} r_{j, k} 2^k$（其中 $\ell$ 为质数位宽），我们得到：

$$
V = \left( \sum_{j=1}^{|P|} \sum_{k=0}^{\ell-1} r_{j, k} \left[ u_j \cdot 2^k \right] \right) \bmod L \bmod N
$$

> **Insight**：这一步极其关键。原本深度极长、极易累积误差的乘法链，被成功展平为了针对 $\vert P\vert$ 个小余数的线性求和问题。

**Step 3.3: 截断累加 (Truncated Accumulation)**
由于总的 Modular deviation 只要保持在常数量级，就不会影响最后的 Period finding，我们不需要精确算出 $V$ 的每一位。
设 $t = \text{len}(N) - f$ 为被丢弃的低位长度。我们只保留累加器的高 $f$ 位，得到近似值 $\widetilde{V} \approx V \gg t$：

$$
\widetilde{V} = \left( \sum_{j=1}^{|P|} \sum_{k=0}^{\ell-1} r_{j, k} \left( \left[ \left( u_j \cdot 2^k \right) \bmod L \bmod N \right] \gg t \right) \right) \bmod (N \gg t)
$$

可以进一步简写为：$$\widetilde{V} = \left( \sum_{j=1}^{\vert P\vert} \sum_{k=0}^{\ell-1} r_{j, k} C_{j, k} \right) \bmod (N \gg t)$$。其中 $C_{j, k}$ 是完全可以由经典计算机提前算好的常数。至此，量子计算机只需要维护一个宽度为 $f = O(\log \log N)$ 的小型截断累加器即可完成整个模幂过程的近似。

#### 4. 硬件映射与高度并行化 (Pinnacle 2026)
在 Pinnacle 架构的物理实现中，Gidney 算法被进一步从串行扫描转化为极限的硬件级并行：
* **分布式计算**：算法外层循环（遍历 $\vert P\vert$ 个质数）被分配给 $\rho$ 个独立的“微型工作寄存器（Working Registers）”并行执行。
* **完全解耦**：在漫长的求和过程中，这 $\rho$ 个处理单元完全独立运行，互不通信。
* **二叉树归并**：直到算法最后一步，才通过代码手术（Code Surgery）将这 $\rho$ 个截断累加器的结果以二叉树形式（Parallel Reduction）两两合并，完成最后的频率提取。
