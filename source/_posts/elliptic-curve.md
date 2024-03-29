---
title: 椭圆曲线加密
date: 2020-03-18 17:23:45
tags:
- cryptology
- ecdsa
- elliptic curve
---

# 椭圆曲线应用原理

## 离散对数问题

ECC算法是在有限域Fp定义公式:Q=kP,已知大数k和点P的情况下,很容易求点Q,但是已知的点P、点Q,却很难求得k,这就是经典的离散对数问题,ECC算法正是利用该特点进行加密,点Q为公钥,大数k为私钥,点P为基点

## 数字签名算法

1、选择一条椭圆曲线Ep(a,b),和基点G
2、选择私有密钥da（da < n, n为G的阶),利用基点G计算公开密钥 Qa = da * G
3、产生一个随机整数k (k < n),计算点(x, y) = k * G,取x坐标为R
4、对原数据做hash为z
5、利用方程 S = k^-1 * (z + da * R) mod p 计算S
6、R和S做为签名值, 如果R和S其中一个为0, 重新从第3步开始执行

## 数字签名验证

1、接受方在收到消息(m)和签名值(R,S)后,进行以下运算
2、对原数据做hash为z
3、计算 P = S^-1 * z * G + S^-1 * R * Qa
4、如果P的 x = R, 则签名有效

## 椭圆曲线非对称加解密

1、椭圆曲线上的两对公私钥 Qa = da * G, Qb = db * G
2、设用户密钥为 [da, Qa]
3、临时生成密钥为 [db, Qb]
4、运用用户公钥及临时私钥计算加密密钥: Key = Qa * db,运用Key对原文做加密,返回密文及临时公钥Qb,返回信息中可选包含Key及原文的hash
5、解密时提取临时公钥Qb,使用用户私钥计算加密密钥: Key = da * Qb,运用Key对密文做解密,可选对hash信息做验证
6、运用Key对原文做加解密本质是对称加密,只要能通过Key进行加解密即可,由于Key的长度有限,需要考虑Key的加解密算法安全性

# 椭圆曲线

## 曲线方程

```bash
y^2 = x^3 + ax + b
4a^3 + 27b^2 ≠ 0
```
其中：4a^3 + 27b^2 ≠ 0 这个限定条件是为了保证曲线不包含奇点（在数学中是指曲线上任意一点都存在切线）

## 加法

```bash
C = A + B
```
椭圆曲线上的两点A、B直线与椭圆曲线的交点,该交点关于x轴对称位置的点,定义为 A + B

## 数乘

```bash
C = 2A = A + A
```
椭圆曲线在A点的切线与椭圆曲线的交点,该焦点关于x轴对称位置的点,定义为 2A

## 正负取反

椭圆曲线上的点A关于x轴对称位置的点定义为-A

## 无穷远点

将A与-A相加,过A与-A的直线平行于y轴,可以认为直线与椭圆曲线相交于无穷远点

# 椭圆曲线中的阿贝尔群

## 阿贝尔群

一个集合以及一个二元运算所组成的结构满足如下条件(其中 “+” 表示某二院运算):

1、封闭性(closure),如果a和b被包含于群,那么 a + b 也一定是群的元素
2、结合律(associativity)
3、存在一个单位元(identity element)0, 0与任意元素运算不改变其值的元素,即 a + 0 = 0 + a = a
4、每个元素都存在一个逆元(inverse)
5、交换律(commutativity),即 a + b = b + a

## 椭圆曲线中的阿贝尔群

在椭圆曲线上定义一个群:

1、群中的元素就是椭圆曲线上的点
2、单位元就是无穷处的点0
3、相反数P,是关于X轴对称的另一边的点
4、二元运算规则定义如下:取一条直线上的三点(这条直线和椭圆曲线相交的三点)P, Q, R（皆非零）,他们的总和等于0,即: P + Q + R = 0, 如果P,Q,R在一条直线上的话,他们满足: P + ( Q + R ) = Q + ( P + R ) = R + ( P + Q ) = ⋯ = 0,当P,Q点为同一点时,P = Q,满足: P + Q = -R

推出:因为阿贝尔群满足交换律和结合律,所以点P和点-R的二元运算结果必会在曲线上,即 P + P + P 的结果必会在曲线上的另一点Q,以此类推,可以得出得出: Q = kP (k个相同的点P进行二元运（数乘),记做kP)
