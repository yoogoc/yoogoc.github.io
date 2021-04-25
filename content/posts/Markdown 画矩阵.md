---
title: Markdown 画矩阵
tags:
- markdown
date: 2018-11-23 14:10:01
---
画普通矩阵，不带括号的
```math
 \begin{matrix}
   a & b & c & d & e\\
   f & g & h & i & j \\
   k & l & m & n & o \\
   p & q & r & s & t
  \end{matrix}
```

画带中括号的矩阵

```math
\left[
 \begin{matrix}
   a & b & c & d & e\\
   f & g & h & i & j \\
   k & l & m & n & o \\
   p & q & r & s & t
  \end{matrix}
\right]
```

画带大括号的矩阵

```math
\left\{
 \begin{matrix}
   a & b & c & d & e\\
   f & g & h & i & j \\
   k & l & m & n & o \\
   p & q & r & s & t
  \end{matrix}
\right\}
```

矩阵前加个参数
```math
A=
\left\{
 \begin{matrix}
   a & b & c & d & e\\
   f & g & h & i & j \\
   k & l & m & n & o \\
   p & q & r & s & t
  \end{matrix}
\right\}
```

矩阵中间有省略号

```
\cdots为水平方向的省略号
\vdots为竖直方向的省略号
\ddots为斜线方向的省略号

```

```math
A=
\left\{
 \begin{matrix}
   a & b & \cdots & e\\
   f & g & \cdots & j \\
   \vdots & \vdots & \ddots & \vdots \\
   p & q & \cdots & t
  \end{matrix}
\right\}
```

矩阵中间加根横线

```
array必须为array
{cccc|c}中的c表示矩阵元素，可以控制|的位置
```

```math
A=
\left\{
 \begin{array}{cccc|c}
     a & b & c & d & e\\
     f & g & h & i & j \\
     k & l & m & n & o \\
     p & q & r & s & t
  \end{array}
\right\}
```

