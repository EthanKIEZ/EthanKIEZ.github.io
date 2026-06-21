---
title: "Archimedes' Hat-Box Theorem 学习笔记：球面面积公式与多元微积分推导"
date: 2026-06-21 14:00:00 +0800
categories:
  - 学习笔记
tags:
  - 微积分
  - 几何
  - 阿基米德
  - 球面几何
  - 多元微积分
toc: true
toc_label: "目录"
toc_icon: "list"
excerpt: "用简单的多元微积分推导球面面积元公式，并证明阿基米德帽盒定理。"
mathjax: true
---

## 前言

今天学习了一个非常优美的几何定理——**阿基米德帽盒定理**（Archimedes' Hat-Box Theorem）。它告诉我们：

> 球面上任意一个球带的表面积，等于其外切圆柱上对应带区的侧面积。

换句话说，若球半径为 $R$，球带位于两个平行平面之间，垂直高度为 $h$，则球带表面积为：

$$A = 2\pi R h$$

这与半径为 $R$、高为 $h$ 的圆柱侧面积完全相同。这篇文章记录完整的推导过程。

---

## 1. 球面的参数化

**知识点**：球坐标、参数化曲面

设球面半径为 $R$，用经度 $\lambda$ 和余纬度 $\phi$（从北极点起算）参数化：

$$x = R\sin\phi\cos\lambda, \quad y = R\sin\phi\sin\lambda, \quad z = R\cos\phi$$

其中 $\phi \in [0,\pi]$，$\lambda \in [0,2\pi)$。

> **一句话总结**：用 $\phi$ 和 $\lambda$ 两个角度，可以描述球面上任意一点。

---

## 2. 球面面积元的推导

**知识点**：多元微积分、叉积求面积元

### 2.1 计算偏导数

$$\mathbf{r}_\phi = \frac{\partial \mathbf{r}}{\partial \phi} = \begin{pmatrix} R\cos\phi\cos\lambda \\ R\cos\phi\sin\lambda \\ -R\sin\phi \end{pmatrix}$$

$$\mathbf{r}_\lambda = \frac{\partial \mathbf{r}}{\partial \lambda} = \begin{pmatrix} -R\sin\phi\sin\lambda \\ R\sin\phi\cos\lambda \\ 0 \end{pmatrix}$$

### 2.2 计算叉积

面积元由两个切向量的叉积模长给出：

$$dA = \left| \mathbf{r}_\phi \times \mathbf{r}_\lambda \right| \, d\phi\, d\lambda$$

展开并化简后：

$$\boxed{dA = R^2 \sin\phi \, d\phi \, d\lambda}$$

> **一句话总结**：球面面积元等于 $R^2\sin\phi$，这是后续所有公式的起点。

---

## 3. 由经线和纬线围成的区域面积

**知识点**：二重积分、球面矩形

### 3.1 面积公式

| 已知条件 | 面积公式 |
|---------|---------|
| 经线夹角 $\Delta\lambda$，余纬度 $\phi_1,\phi_2$ | $A = R^2\Delta\lambda \left| \cos\phi_1 - \cos\phi_2 \right|$ |
| 经线夹角 $\Delta\lambda$，地理纬度 $\theta_1,\theta_2$ | $A = R^2\Delta\lambda \left| \sin\theta_2 - \sin\theta_1 \right|$ |

### 3.2 为什么有两种形式？

地理纬度 $\theta$ 与余纬度 $\phi$ 的关系为：

$$\theta = \frac{\pi}{2} - \phi$$

因此 $\cos\phi = \sin\theta$。两种形式本质相同，只是坐标选择不同。

---

## 4. 证明 Archimedes' Hat-Box Theorem

**知识点**：球带、圆柱侧面积

### 4.1 定理内容

> 球面上任意球带的表面积，等于其外切圆柱对应带区的侧面积。

### 4.2 证明过程

设球带位于高度 $z_1$ 和 $z_2$ 之间，垂直高度 $h = |z_2 - z_1|$。

由于 $z = R\sin\theta$，有：

$$h = R \left| \sin\theta_2 - \sin\theta_1 \right|$$

球带绕球一周，因此 $\Delta\lambda = 2\pi$。代入面积公式：

$$A_{sphere} = R^2 \cdot 2\pi \cdot \left| \sin\theta_2 - \sin\theta_1 \right| = 2\pi R h$$

而半径为 $R$、高为 $h$ 的圆柱侧面积为：

$$A_{cylinder} = 2\pi R h$$

| 几何体 | 面积 |
|--------|------|
| 球带 | $2\pi R h$ |
| 外切圆柱带 | $2\pi R h$ |

因此 $A_{sphere} = A_{cylinder}$，证毕。

> **一句话总结**：球带面积只与高度 $h$ 有关，与球带位置无关。

---

## 5. 直观理解

帽盒定理最令人惊讶的地方在于：球带的面积只与高度 $h$ 有关，而与球带在球面上的具体位置无关。

- 靠近赤道的宽带 和 靠近两极的窄带，只要高度相同，面积就相同
- 一个半球（$h = R$）的表面积为 $2\pi R^2$，正好是整球的一半
- 整个球面（$h = 2R$）的表面积为 $4\pi R^2$，与经典公式一致

---

## 6. 核心公式速查

| 公式 | 含义 |
|------|------|
| $dA = R^2\sin\phi \, d\phi \, d\lambda$ | 球面面积元 |
| $A = R^2\Delta\lambda \left| \cos\phi_1 - \cos\phi_2 \right|$ | 球面矩形面积（余纬度形式） |
| $A = R^2\Delta\lambda \left| \sin\theta_2 - \sin\theta_1 \right|$ | 球面矩形面积（地理纬度形式） |
| $A = 2\pi R h$ | 帽盒定理：球带面积 = 圆柱侧面积 |
| $S = 4\pi R^2$ | 整球表面积 |

---

## 总结

今天通过三个步骤证明了 Archimedes' Hat-Box Theorem：

1. **参数化球面**，用叉积推导出面积元
2. **积分得到球面矩形面积公式**
3. **令 $\Delta\lambda = 2\pi$**，证明球带面积等于圆柱侧面积

这个定理展示了多元微积分在几何中的强大威力，也揭示了球面与圆柱之间一个深刻而优美的联系。

---

## 附录：公式显示说明

本文使用 LaTeX 语法 `$...$` 和 `$$...$$` 书写公式。如果网页上公式没有正常渲染，说明你的 Jekyll 主题尚未启用 MathJax。可以在主题模板中加入以下代码：

```html
<script src="https://polyfill.io/v3/polyfill.min.js?features=es6"></script>
<script id="MathJax-script" async src="https://cdn.jsdelivr.net/npm/mathjax@3/es5/tex-mml-chtml.js"></script>
```
