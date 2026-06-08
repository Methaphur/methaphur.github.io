---
title: "The Remarkable Theorem of Gauss"
date: 2026-05-16
description: "A visual and mathematical tour of Gauss's Theorema Egregium, the theorem that says Gaussian curvature belongs to the surface itself."
summary: "Gauss's Theorema Egregium reveals that Gaussian curvature is intrinsic: it can be read from distances measured on a surface, without looking at the surrounding 3D space."
tags: ["differential geometry", "gaussian curvature", "gauss"]
categories: ["Mathematics"]
showHero: true
heroStyle: "big"
featureimagecaption: "Curvature can look extrinsic, but Gauss showed it is encoded in the surface's own metric."
alt: "An illustrated curved surface and a flat grid showing intrinsic curvature"
---

{{< katex >}}

## Introduction

Most geometric properties seem to require an outside observer. To tell whether a wire is bent, you look at it from the side. To tell whether a road curves, you look at a map. Curvature feels like something that can only be seen from the outside.

Gauss discovered that, for surfaces, this intuition is wrong.

Imagine you are a 2-dimensional ant living on a surface with no access to the surrounding space. You can walk around, measure distances, and study the geometry of your world, but you can never step outside and look at it from above.

Could you determine whether your world is curved?

In 1827, Gauss proved that you can. The Gaussian curvature of a surface is completely determined by measurements made within the surface itself. No external viewpoint is needed.

He called the result *Theorema Egregium*, the Remarkable Theorem.

That is the heart of the theorem:

$$
\text { Gaussian Curvature is determined entirely by the First Fundamental Form.}
$$

## Why It Looks Extrinsic At First

Classically, Gaussian curvature first appears through the First and Second Fundamental Forms. For a parametrized surface \(\boldsymbol{\sigma}(u,v)\), the First Fundamental Form is

$$
I = E\,du^2 + 2F\,du\,dv + G\,dv^2,
$$

where

$$
E = \boldsymbol{\sigma}_u \cdot \boldsymbol{\sigma}_u,\quad
F = \boldsymbol{\sigma}_u \cdot \boldsymbol{\sigma}_v,\quad
G = \boldsymbol{\sigma}_v \cdot \boldsymbol{\sigma}_v.
$$

The Second Fundamental Form is

$$
\Pi = L\,du^2 + 2M\,du\,dv + N\,dv^2.
$$

where

$$
L = \boldsymbol{\sigma}_{uu}\cdot\bar{N},\quad
M = \boldsymbol{\sigma}_{uv}\cdot\bar{N},\quad
N = \boldsymbol{\sigma}_{vv}\cdot\bar{N}.
$$

and \(\bar{N}\) is the unit normal vector to the surface. 

The Gaussian curvature is then

$$
K = \frac{LN - M^2}{EG - F^2}.
$$

At first, this looks extrinsic. The coefficients \(L\), \(M\), and \(N\) are computed using the unit normal vector to the surface. They measure how the surface bends in ambient three-dimensional space. So it seems natural to expect \(K\) to depend on the embedding.

Gauss proved that this expectation is wrong.

## Christoffel Symbols

The bridge from extrinsic curvature to intrinsic geometry is built from the Christoffel symbols. Let the surface patch be \(\boldsymbol{\sigma}(u_1,u_2)\), where \(u_1 = u\) and \(u_2 = v\). Write

$$
\boldsymbol{\sigma}_1 = \boldsymbol{\sigma}_u,\qquad
\boldsymbol{\sigma}_2 = \boldsymbol{\sigma}_v.
$$

The coefficients of the First Fundamental Form are

$$
g_{ij} = \boldsymbol{\sigma}_i \cdot \boldsymbol{\sigma}_j.
$$

Thus \(g_{11} = E\), \(g_{12} = g_{21} = F\), and \(g_{22} = G\). For \(1 \leq i,j,k \leq 2\), the Christoffel symbols \(\Gamma^k_{ij}\) are scalar functions defined by

$$
\begin{pmatrix}
\Gamma^1_{ij} \\
\Gamma^2_{ij}
\end{pmatrix}
{}=
\begin{pmatrix}
g_{11} & g_{12} \\
g_{21} & g_{22}
\end{pmatrix}^{-1}
\begin{pmatrix}
\boldsymbol{\sigma}_{ij} \cdot \boldsymbol{\sigma}_1 \\
\boldsymbol{\sigma}_{ij} \cdot \boldsymbol{\sigma}_2
\end{pmatrix}.
$$

In words, they give the tangent part of the second derivative \(\boldsymbol{\sigma}_{ij}\). They can also be computed directly from the metric:

$$
\Gamma^k_{ij}
{}=
\frac{1}{2}
\sum_{l=1}^{2}
g^{kl}
\left(
\frac{\partial g_{il}}{\partial u_j}
+
\frac{\partial g_{lj}}{\partial u_i}
{}-
\frac{\partial g_{ij}}{\partial u_l}
\right),
$$

where \(g^{kl}\) denotes the \((k,l)\)-entry of the inverse matrix \((g_{ij})^{-1}\).

This is the crucial intrinsic point: Christoffel symbols are determined entirely by the first fundamental form and its derivatives. A surface-dweller could compute them by measuring lengths on the surface. They are not, however, intrinsic invariants: their numerical values depend on the coordinate system.

With these symbols, the second derivatives of the surface patch split into tangent and normal pieces:

$$
\boldsymbol{\sigma}_{ij}
{}=
\Gamma^1_{ij}\boldsymbol{\sigma}_1
+
\Gamma^2_{ij}\boldsymbol{\sigma}_2
+
b_{ij}\bar{N}.
$$

This is the Formula of Gauss. The first two terms are tangent to the surface, and the final term is normal to it. Here \(b_{ij}\) are the coefficients of the Second Fundamental Form and \(\bar{N}\) is the unit normal.

## How They Enter Theorema Egregium

The Gaussian curvature can be written as

$$
K
{}=
\frac{\det(b_{ij})}{\det(g_{ij})}.
$$

Since \(\det(g_{ij}) = EG - F^2\) is already intrinsic, Gauss only had to show that \(\det(b_{ij})\) can also be expressed intrinsically.

Start with the Formula of Gauss,

$$
\boldsymbol{\sigma}_{ij}
{}=
\sum_{m=1}^{2}\Gamma^m_{ij}\boldsymbol{\sigma}_m
+
b_{ij}\bar{N},
$$

and differentiate with respect to \(u_k\). This gives expressions for the third derivatives \(\boldsymbol{\sigma}_{ijk}\). Since mixed partial derivatives commute, we have

$$
\boldsymbol{\sigma}_{ijk} = \boldsymbol{\sigma}_{ikj}.
$$

Taking dot products with a tangent vector \(\boldsymbol{\sigma}_l\) and expanding introduces the Riemann symbol:

$$
\begin{aligned}
R^l_{jki}
={}&
\sum_{m=1}^{2}
\left(
\frac{\partial \Gamma^m_{ij}}{\partial u_k}g_{ml}
+
\Gamma^m_{ij}\,\boldsymbol{\sigma}_{mk}\cdot\boldsymbol{\sigma}_l
\right) \\
&{}-
\sum_{m=1}^{2}
\left(
\frac{\partial \Gamma^m_{ik}}{\partial u_j}g_{ml}
+
\Gamma^m_{ik}\,\boldsymbol{\sigma}_{mj}\cdot\boldsymbol{\sigma}_l
\right).
\end{aligned}
$$

This quantity is built from Christoffel symbols, their partial derivatives, and first fundamental form data. The symmetry of mixed partial derivatives yields

$$
R^l_{jki}
{}-
b_{ij}b_{lk}
+
b_{ik}b_{lj}
= 0.
$$

Choosing \(i=1\), \(j=2\), \(k=1\), and \(l=2\), we get

$$
R^2_{121}
{}=
b_{11}b_{22}
{}-
b_{12}b_{21}
{}=
\det(b_{ij}).
$$

Therefore \(\det(b_{ij})\) is expressible in terms of intrinsic data. Consequently,

$$
K
{}=
\frac{\det(b_{ij})}{\det(g_{ij})}
$$

is intrinsic. This is the remarkable content of Gauss's theorem: the curvature that first appeared to depend on bending in 3D space is completely determined by measurements made on the surface itself.

## A Cleaner Formula In Orthogonal Coordinates

When the coordinate curves meet at right angles, \(F = 0\). In that case, the intrinsic expression becomes much more readable:

$$
K =
-\frac{1}{2\sqrt{EG}}
\left[
\frac{\partial}{\partial u}
\left(
\frac{G_u}{\sqrt{EG}}
\right)
+
\frac{\partial}{\partial v}
\left(
\frac{E_v}{\sqrt{EG}}
\right)
\right].
$$

This formula is beautiful because it contains no reference to a normal vector or to 3D space. The curvature is extracted from the coefficients \(E\) and \(G\) alone.

## Why The Theorem Is So Powerful

The biggest consequence is that Gaussian curvature is preserved by local isometries.

A local isometry is a transformation that preserves distances. Since distances determine \(E\), \(F\), and \(G\), and since \(K\) is determined by those coefficients, curvature must be preserved as well.

This explains a familiar impossibility: there is no perfect flat map of the Earth.

A plane has

$$
K = 0,
$$

while a sphere of radius \(R\) has

$$
K = \frac{1}{R^2}.
$$

Because their Gaussian curvatures differ, no map from the sphere to the plane can preserve all distances, even locally, without distortion. Something must stretch, tear, or compress.

The same idea explains why paper can be rolled into a cylinder. A sheet of paper and a cylinder both have \(K = 0\). The cylinder looks curved from outside, but intrinsically it remains flat.

## The Remarkable Moral

Gauss revealed that curvature has an internal life. It is not merely how a surface sits in the world around it. It is encoded in the surface's own measurements.

That insight changed geometry. It prepared the ground for Riemannian geometry, where one studies curved spaces without needing a larger space to contain them. It also echoes through general relativity, where curvature is not something observed from outside the universe, but something measured from within it.

The theorem is remarkable because it turns a question about shape into a question about distance.
