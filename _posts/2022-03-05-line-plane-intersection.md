---
layout: post
title: Checking if Line and Plane Intersect
tags: [maths, computational geometry, computer graphics]
mathjax: true
---

The post is compiled with information on [Wikipedia page on Line-plane Interaction](https://en.wikipedia.org/wiki/Line%E2%80%93plane_intersection) and [this notes on finding the normal to a plane](https://web.ma.utexas.edu/users/m408m/Display12-5-4.shtml).

## What is the question?

This post is about checking if a line intersect with a plane, given 2 different coordinates on the line and 3 different coordinates on the plane.

## How to solve it?

Suppose having a line with coordinates $$\mathbf{a}$$ and $$\mathbf{b}$$ on the line, the vector equation representing the line with the set of $$\mathbf{p}$$ consist the line is:
\begin{equation}
\label{eq:line}
\mathbf{p} = \mathbf{a} + d \mathbf{(b - a)}
\end{equation}

Suppose having a plane with coordinates $$\mathbf{e}$$, $$\mathbf{f}$$ and $$\mathbf{g}$$ on the plane, then the normal to the plane is $$\mathbf{n = (f - e) \times (g - e)}$$. If $$\mathbf{p_0}$$ is a point on the plane (e.g. $$\mathbf{p_0}$$ can be $$\mathbf{e}$$, $$\mathbf{f}$$ or $$\mathbf{g}$$), the plane can be expressed as the set of points $$\mathbf{p}$$ for which:
\begin{equation}
\label{eq:plane}
\mathbf{(p - p_0) \cdot n} = 0
\end{equation}

The point(s) where the line and the plane intersect, the points have the same coordinates. Substitute \eqref{eq:line} into \eqref{eq:plane}:

$$
\begin{align*}
((\mathbf{a} + d \mathbf{(b - a)}) - \mathbf{p_0}) \cdot \mathbf{n} &= 0\\
d \mathbf{(b - a)} \cdot \mathbf{n} + \mathbf{(a - p_0)} \cdot \mathbf{n} &= 0\\
d &= \frac{\mathbf{(p_0 - a)} \cdot \mathbf{n}}{\mathbf{(b - a)} \cdot \mathbf{n}}
\end{align*}
$$

If $$\mathbf{(b - a)} \cdot \mathbf{n} = 0$$ then the line and the plane are parallel. In this case, if $$\mathbf{(p_0 - a)} \cdot \mathbf{n} = 0$$ then the line is on the plane. Otherwise the line and plane have no intersection.

If $$\mathbf{(b - a)} \cdot \mathbf{n} \neq 0$$ there is a single point of intersection.
