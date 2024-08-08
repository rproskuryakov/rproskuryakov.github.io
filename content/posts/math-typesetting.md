---
author: rodion proskuriakov
date: "2019-03-08"
description: A brief guide to setup KaTeX
math: true
title: Math Typesetting
---

Mathematical notation in a Hugo project can be enabled by using third party JavaScript libraries.

<!--more-->

In this example we will be using [KaTeX](https://katex.org/)


**Note:** Use the online reference of [Supported TeX Functions](https://katex.org/docs/supported.html)

{{< math.inline >}}

{{</ math.inline >}}

### Examples

{{< math.inline >}}

<p>
Inline math: \(\varphi = \dfrac{1+\sqrt5}{2}= 1.6180339887…\)
</p>

{{</ math.inline >}}

Block math:

```katex
 \varphi = 1+\frac{1} {1+\frac{1} {1+\frac{1} {1+\cdots} } }

```

Inline formulas: {{< katex formula="a^n" inline=true />}}, {{< katex "a^2+b^2=c^2" true />}}.



[//]: # ($${a}^{b} - \overbrace{c}^{d}$$)

[//]: # ()
[//]: # ()
[//]: # (```katex)

[//]: # (\tag*{&#40;1&#41;} P&#40;E&#41; = {n \choose k} p^k &#40;1-p&#41;^{n-k})

[//]: # (```)

[//]: # (```katex)

[//]: # (f&#40;x&#41; = \int_{-\infty}^\infty\hat f&#40;\xi&#41;\,e^{2 \pi i \xi x}\,d\xi)

[//]: # (```)