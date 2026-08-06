---
title: Composiciones matemáticas
description: Composición matemática usando KaTeX
date: 2023-08-24 00:00:00+0000
math: true
---

Stack tiene soporte incorporado para la composición matemática usando [KaTeX](https://katex.org/).

**No está habilitado de forma predeterminada para todo el sitio,** pero puedes activarlo para publicaciones individuales agregando `math: true` a los front matter. O puedes activarlo para todo el sitio agregando `math = true` a la sección `params.article` en `config.toml`.

## Matemáticas en línea

Esta es una expresión matemática en línea: $\varphi = \dfrac{1+\sqrt5}{2}= 1.6180339887…$

```markdown
$\varphi = \dfrac{1+\sqrt5}{2}= 1.6180339887…$
```

## Matemáticas en bloque

$$
    \varphi = 1+\frac{1} {1+\frac{1} {1+\frac{1} {1+\cdots} } } 
$$

```markdown
$$
    \varphi = 1+\frac{1} {1+\frac{1} {1+\frac{1} {1+\cdots} } } 
$$
```

$$
    f(x) = \int_{-\infty}^\infty\hat f(\xi)\,e^{2 \pi i \xi x}\,d\xi
$$

```markdown
$$
    f(x) = \int_{-\infty}^\infty\hat f(\xi)\,e^{2 \pi i \xi x}\,d\xi
$$
```
