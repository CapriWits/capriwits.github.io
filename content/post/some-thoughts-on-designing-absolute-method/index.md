---
title: "关于设计绝对值 abs() 的一些思考"
slug: "some-thoughts-on-designing-absolute-method"
date: 2021-09-18T17:34:24+08:00
author: CapriWits
image: java-programming-cover.png
tags:
  - Java
  - Design
categories:
  - Backend
hidden: false
math: true
comments: true
draft: false
---
# 关于设计绝对值abs的一些思考

![](https://img.shields.io/badge/JDK-1.8-red)

「**取绝对值**」对于 `Integer` 毫无疑问直接判断正负

* `Math::abs(int)`

```Java
public static int abs(int a) {
    return (a < 0) ? -a : a;
}
```

**注意到双精度浮点数** `Double` 官方使用以下实现

* `Math::abs(double)`

```Java
public static double abs(double a) {
    return (a <= 0.0D) ? 0.0D - a : a;
}
```

Java 遵循 [IEEE-754](https://en.wikipedia.org/wiki/IEEE_754-2008) 标准，因此实现上存在 `+0.0` & `-0.0`，两者除了文本表示不同，在计算过程中也不同。如：`1 / +- 0.0` 得到的结果是 `+Infinity` & `-Infinity`

`abs` 计算结果仍然是负数，出现**错误**，原因既是 `+0.0 == -0.0`

```Java
public class Solution {
    public static void main(String[] args) {
        double x = -0.0;
        if (1 / abs(x) < 0) {
            System.out.println("abs(x) < 0");
        }
    }
    public static double abs(double a) {
        return (a < 0) ? -a : a;
    }
}
```

**尝试解决问题，添加判断条件：**`if (val < 0 || val == -0.0)` 对 `-0.0` 单独考虑，进行双重判断，这里采用 `Double::compare(double, double)` 实现

**成功实现**

```Java
public static double abs(double value) {
    if (value < 0 || Double.compare(value, -0.0) == 0) {
        return -value;
    }
    return value;
}
```

再追求极致的优化。查看 [Double::compare](https://github.com/openjdk/jdk/blob/36e2ddad4d2ef3ce27475af6244d0246a8315c0c/src/java.base/share/classes/java/lang/Double.java#L1117) 实现。

对于正数进行额外的**两次**比较, 对于 `-0.0` 进行额外的 **三次** 比较, 对于 `+0.0` 进行额外的 **四次** 比较

```Java
public static int compare(double d1, double d2) {
    if (d1 < d2)
        return -1;           // Neither val is NaN, thisVal is smaller
    if (d1 > d2)
        return 1;            // Neither val is NaN, thisVal is larger

    // Cannot use doubleToRawLongBits because of possibility of NaNs.
    long thisBits    = Double.doubleToLongBits(d1);
    long anotherBits = Double.doubleToLongBits(d2);

    return (thisBits == anotherBits ?  0 : // Values are equal
            (thisBits < anotherBits ? -1 : // (-0.0, 0.0) or (!NaN, NaN)
             1));                          // (0.0, -0.0) or (NaN, !NaN)
}
```

**而实际上，要想实现只需要** `Double::doubleToLongBits` 方法，将 `Double` 转 `Long`

```Java
private static final long MINUS_ZERO_LONG_BITS =
        Double.doubleToLongBits(-0.0);

public static double abs(double value) {
    if (value < 0 ||
            Double.doubleToLongBits(value) == MINUS_ZERO_LONG_BITS) {
        return -value;
    }
    return value;
}
```

不过 `Double::doubleToLongBits` 也只有微不足道的性能提升，因为它会对 `NaN` 进行约束，`NaN` 会赋值为 `0x7ff8000000000000L` ，如果确保 `abs` 入参肯定是 `double`，则只需要取出 `Double::doubleToRawLongBits`

```Java
public static long doubleToLongBits(double value) {
    if (!isNaN(value)) {
        return doubleToRawLongBits(value);
    }
    return 0x7ff8000000000000L;
}
```

**于是就变成这样实现**

```Java
private static final long MINUS_ZERO_LONG_BITS = Double.doubleToRawLongBits(-0.0);

public static double abs(double value) {
  if (value < 0 ||
      Double.doubleToRawLongBits(value) == MINUS_ZERO_LONG_BITS) {
    return -value;
  }
  return value;
}
```

到 `JDK8` 就结束了，而 `JDK9` 开始引入 `@HotSpotIntrinsicCandidate` 注解，即 `HotSpot JVM` 内部的 `JIT compiler` 会移除 `JDK` 中的实现方法，采用 `CPU` 指令直接实现，这会比高级语言转汇编转机器语言要快很多，毕竟 CPU 并不会在乎数据类型的问题，只需要重新解释(reinterpreting) 储存在 CPU 寄存器的一组位的问题，以便于与 Java 数据类型一致。

```Java
@HotSpotIntrinsicCandidate
public static native long doubleToRawLongBits(double value);
```

**但这么实现，仍然有条件分支，如果 CPU 分支预测(branch predictor) 失误，性能开销就会增大。接下来考虑减少条件分支。**

**利用** `0.0` 与 `+/0.0` 作差，都会使正负零转化为正零

```Java
System.out.println(0.0 - (-0.0)); // 0.0
System.out.println(0.0 - (+0.0)); // 0.0
```

**对方法进行改写**

```Java
public static double abs(double value) {
  if (value == 0) {
    return 0.0 - value;
  }
  if (value < 0) {
    return -value;
  }
  return value;
}
```

**注意到对于普通负数而言，**`0.0 - value` 与 `-value` 的结果相同，所以合并分支

```Java
public static double abs(double value) {
  if (value <= 0) {
    return 0.0 - value;
  }
  return value;
}
```

**AKA**

```Java
public static double abs(double a) {
    return (a <= 0.0) ? 0.0 - a : a;
}
```

会发现，JDK `Math::abs(double,double)` 实现相同（逃

**遵循** `IEEE-754` 的双精度浮点数二进制表达形式，只需要将二进制在高位符号位改成 0 即可实现转正数(abs)，需要掩码 `0x7fffffffffffffffL` == 63位 1 bit

```Java
System.out.println(Long.bitCount(0x7fffffffffffffffL));  // 63
```

## 最终实现

```Java
public static double abs(double value) {
  return Double.longBitsToDouble(
    Double.doubleToRawLongBits(value) & 0x7fffffffffffffffL);
}
```

**📌此版本不存在分支，在某些条件下的吞吐量增加** **10%**，单分支实现在 Java 标准库存在多年，在随即到来的 JDK 18 中，改进版本已经提交「From: 2021/9/18」

然而在许多情况下，这些改进并没有太大意义，因为 JIT 编译器会适当使用汇编指令(if available) 会完全替代 Java code，所以这种改动并 **不能** 使程序显著性提升很多（逃

## 🔗Reference

> [One does not simply calculate the absolute value](https://habr.com/en/post/574082/)
>
> [OpenJDK Double::compare](https://github.com/openjdk/jdk/blob/36e2ddad4d2ef3ce27475af6244d0246a8315c0c/src/java.base/share/classes/java/lang/Double.java#L1117)
>
> [JavaSE 8 doubleToLongBits](https://docs.oracle.com/javase/8/docs/api/java/lang/Double.html#doubleToLongBits-double-)
