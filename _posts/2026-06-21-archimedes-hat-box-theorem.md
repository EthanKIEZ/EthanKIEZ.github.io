---
title: "Archimedes' Hat-Box Theorem 的证明：球面面积公式与多元微积分推导"
date: 2026-06-21 14:00:00 +0800
categories:
  - 数学
tags:
  - 微积分
  - 几何
  - 阿基米德
  - 球面几何
  - 多元微积分
toc: true
toc_label: "目录"
toc_icon: "list"
excerpt: "使用简单的多元微积分推导球面面积元公式，并由此证明阿基米德帽盒定理（Archimedes' Hat-Box Theorem）。"
---

## 1. 引言

**阿基米德帽盒定理**（Archimedes' Hat-Box Theorem）是古希腊数学家阿基米德发现的一个美妙结论：

> 球面上任意一个"球带"（spherical zone）的表面积，等于其外切圆柱上对应带宽的侧面积。

换句话说，若球半径为 $R$，球带位于两个平行平面之间，垂直高度为 $h$，则球带表面积为：

$$A = 2\pi R h$$

这与半径为 $R$、高为 $h$ 的圆柱侧面积完全相同。

本文先用多元微积分推导球面面积元，再给出由两条经线和两条纬线围成的球面区域面积公式，最后用它证明帽盒定理。

---

## 2. 球面的参数化

考虑半径为 $R$ 的球面。采用球坐标参数化，设：

- $\phi$：**余纬度**（co-latitude），从北极点 $N$ 起算，$\phi \in [0, \pi]$
- $\lambda$：**经度**（longitude），$\lambda \in [0, 2\pi)$

则球面上任意一点的位置向量为：

$$
\mathbf{r}(\phi, \lambda) = \begin{pmatrix}
x \\
 y \\
 z
\end{pmatrix}
= \begin{pmatrix}
R\sin\phi\cos\lambda \\
 R\sin\phi\sin\lambda \\
 R\cos\phi
\end{pmatrix}
$$

---

## 3. 球面面积元的推导

### 3.1 计算偏导数

对 $\mathbf{r}$ 分别关于 $\phi$ 和 $\lambda$ 求偏导：

$$
\mathbf{r}_\phi = \frac{\partial \mathbf{r}}{\partial \phi}
= \begin{pmatrix}
R\cos\phi\cos\lambda \\
 R\cos\phi\sin\lambda \\
 -R\sin\phi
\end{pmatrix}
$$

$$
\mathbf{r}_\lambda = \frac{\partial \mathbf{r}}{\partial \lambda}
= \begin{pmatrix}
-R\sin\phi\sin\lambda \\
 R\sin\phi\cos\lambda \\
 0
\end{pmatrix}
$$

### 3.2 计算叉积

面积元由两个切向量的叉积模长给出：

$$
dA = \left| \mathbf{r}_\phi \times \mathbf{r}_\lambda \right| \, d\phi \, d\lambda
$$

计算叉积：

$$
\mathbf{r}_\phi \times \mathbf{r}_\lambda
= \begin{vmatrix}
\mathbf{i} & \mathbf{j} & \mathbf{k} \\
 R\cos\phi\cos\lambda & R\cos\phi\sin\lambda & -R\sin\phi \\
 -R\sin\phi\sin\lambda & R\sin\phi\cos\lambda & 0
\end{vmatrix}
$$

展开得：

$$
\mathbf{r}_\phi \times \mathbf{r}_\lambda
= \begin{pmatrix}
R^2 \sin^2\phi \cos\lambda \\
 R^2 \sin^2\phi \sin\lambda \\
 R^2 \sin\phi\cos\phi
\end{pmatrix}
$$

### 3.3 叉积的模长

$$
\left| \mathbf{r}_\phi \times \mathbf{r}_\lambda \right|
= R^2 \sin\phi \sqrt{\sin^2\phi\cos^2\lambda + \sin^2\phi\sin^2\lambda + \cos^2\phi}
$$

由于 $\cos^2\lambda + \sin^2\lambda = 1$，根号内化简为：

$$
\sin^2\phi + \cos^2\phi = 1
$$

因此：

$$
\boxed{dA = R^2 \sin\phi \, d\phi \, d\lambda}
$$

这就是球面上的面积元公式。

---

## 4. 由经线和纬线围成的区域面积

### 4.1 一般公式

考虑球面上由以下四条曲线围成的区域：

- 两条经线：$\lambda = \lambda_1$ 和 $\lambda = \lambda_2$，夹角 $\Delta\lambda = \lambda_2 - \lambda_1$
- 两条纬线：余纬度 $\phi = \phi_1$ 和 $\phi = \phi_2$

面积为：

$$
A = \int_{\lambda_1}^{\lambda_2} \int_{\phi_1}^{\phi_2} R^2 \sin\phi \, d\phi \, d\lambda
$$

先对 $\phi$ 积分：

$$
\int_{\phi_1}^{\phi_2} \sin\phi \, d\phi = -\cos\phi \Big|_{\phi_1}^{\phi_2} = \cos\phi_1 - \cos\phi_2
$$

再对 $\lambda$ 积分：

$$
A = R^2 \Delta\lambda \left( \cos\phi_1 - \cos\phi_2 \right)
$$

取绝对值：

$$
\boxed{A = R^2 \Delta\lambda \left| \cos\phi_1 - \cos\phi_2 \right|}
$$

### 4.2 用地理纬度表示

地理纬度 $\theta$ 与余纬度 $\phi$ 的关系为 $\theta = \frac{\pi}{2} - \phi$，因此 $\cos\phi = \sin\theta$。

代入得：

$$
\boxed{A = R^2 \Delta\lambda \left| \sin\theta_2 - \sin\theta_1 \right|}
$$

其中 $\theta_1, \theta_2$ 为两条纬线的地理纬度。

---

## 5. 证明 Archimedes' Hat-Box Theorem

### 5.1 球带的定义

设球面上一个**球带**（spherical zone）由两个平行平面截得，这两个平面垂直于 $z$ 轴，分别位于高度 $z_1$ 和 $z_2$。

由参数化 $z = R\cos\phi = R\sin\theta$，可得：

$$
z_1 = R\sin\theta_1, \quad z_2 = R\sin\theta_2
$$

球带的垂直高度为：

$$
h = |z_2 - z_1| = R \left| \sin\theta_2 - \sin\theta_1 \right|
$$

### 5.2 计算球带面积

球带绕整个球面一周，因此 $\Delta\lambda = 2\pi$。代入面积公式：

$$
A_{\text{sphere}} = R^2 \cdot 2\pi \cdot \left| \sin\theta_2 - \sin\theta_1 \right|
$$

整理得：

$$
A_{\text{sphere}} = 2\pi R \cdot R\left| \sin\theta_2 - \sin\theta_1 \right| = 2\pi R h
$$

### 5.3 计算对应圆柱侧面积

考虑半径为 $R$、高为 $h$ 的圆柱，其侧面积为：

$$
A_{\text{cylinder}} = 2\pi R h
$$

### 5.4 结论

因此：

$$
\boxed{A_{\text{sphere}} = A_{\text{cylinder}} = 2\pi R h}
$$

这就是 **Archimedes' Hat-Box Theorem**：球面上任意球带的表面积等于其外切圆柱对应带区的侧面积。

---

## 6. 直观理解

帽盒定理最令人惊讶的地方在于：球带的面积只与球带的高度 $h$ 有关，而与球带在球面上的具体位置无关。

- 靠近赤道的宽带和靠近两极的窄带，只要高度 $h$ 相同，面积就相同
- 一个半球（$h = R$）的表面积为 $2\pi R^2$，恰好是整个球面面积的一半
- 整个球面（$h = 2R$）的表面积为 $4\pi R^2$，与经典公式一致

---

## 7. 总结

本文通过三个步骤证明了 Archimedes' Hat-Box Theorem：

1. **参数化球面**并计算切向量的叉积，得到面积元 $dA = R^2\sin\phi \, d\phi \, d\lambda$
2. **积分得到球面矩形面积公式**：$A = R^2 \Delta\lambda \left| \cos\phi_1 - \cos\phi_2 \right|$
3. **令 $\Delta\lambda = 2\pi$**，将球带面积与圆柱侧面积联系起来，得到 $A = 2\pi R h$

这个证明展示了多元微积分在几何中的强大威力，也揭示了球面与圆柱之间一个深刻而优美的联系。
