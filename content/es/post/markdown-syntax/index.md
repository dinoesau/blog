---
title: Guía de sintaxis de Markdown
date: 2023-09-07
description: Artículo de muestra que muestra la sintaxis básica de Markdown y el formato para elementos HTML.
tags:
    - markdown
    - css
    - html
    - themes
categories:
    - themes
    - syntax
---

Este artículo ofrece una muestra de la sintaxis básica de Markdown que se puede usar en los archivos de contenido de Hugo, y también muestra si los elementos HTML básicos están decorados con CSS en un tema de Hugo.

<!--more-->

## Encabezados

Los siguientes elementos HTML `<h1>`—`<h6>` representan seis niveles de encabezados de sección. `<h1>` es el nivel de sección más alto mientras que `<h6>` es el más bajo.

# H1
## H2
### H3
#### H4
##### H5
###### H6

## Párrafo

Xerum, quo qui aut unt expliquam qui dolut labo. Aque venitatiusda cum, voluptionse latur sitiae dolessi aut parist aut dollo enim qui voluptate ma dolestendit peritin re plis aut quas inctum laceat est volestemque commosa as cus endigna tectur, offic to cor sequas etum rerum idem sintibus eiur? Quianimin porecus evelectur, cum que nis nust voloribus ratem aut omnimi, sitatur? Quiatem. Nam, omnis sum am facea corem alique molestrunt et eos evelece arcillit ut aut eos eos nus, sin conecerem erum fuga. Ri oditatquam, ad quibus unda veliamenimin cusam et facea ipsamus es exerum sitate dolores editium rerore eost, temped molorro ratiae volorro te reribus dolorer sperchicium faceata tiustia prat.

Itatur? Quiatae cullecum rem ent aut odis in re eossequodi nonsequ idebis ne sapicia is sinveli squiatum, core et que aut hariosam ex eat.

## Citas en bloque

El elemento de cita en bloque representa contenido que se cita de otra fuente, opcionalmente con una cita que debe estar dentro de un elemento `footer` o `cite`, y opcionalmente con cambios en línea como anotaciones y abreviaturas.

### Cita en bloque sin atribución

> Tiam, ad mint andaepu dandae nostion secatur sequo quae.
> **Nota:** puedes usar *sintaxis de Markdown* dentro de una cita en bloque.

### Cita en bloque con atribución

> No te comuniques compartiendo memoria, comparte memoria comunicándote.<br>
> — <cite>Rob Pike[^1]</cite>

[^1]: La cita anterior está extraída de la [charla](https://www.youtube.com/watch?v=PAAkCSZUG1c) de Rob Pike durante Gopherfest, 18 de noviembre de 2015.

## Tablas

Las tablas no forman parte de la especificación principal de Markdown, pero Hugo las soporta de fábrica.

   Nombre | Edad
--------|------
    Bob | 27
  Alice | 23

### Markdown en línea dentro de tablas

| Cursiva   | Negrita     | Código   |
| --------  | -------- | ------ |
| *cursiva* | **negrita** | `código` |

| A                                                        | B                                                                                                             | C                                                                                                                                    | D                                                 | E                                                          | F                                                                    |
|----------------------------------------------------------|---------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------|------------------------------------------------------------|----------------------------------------------------------------------|
| Lorem ipsum dolor sit amet, consectetur adipiscing elit. | Phasellus ultricies, sapien non euismod aliquam, dui ligula tincidunt odio, at accumsan nulla sapien eget ex. | Proin eleifend dictum ipsum, non euismod ipsum pulvinar et. Vivamus sollicitudin, quam in pulvinar aliquam, metus elit pretium purus | Proin sit amet velit nec enim imperdiet vehicula. | Ut bibendum vestibulum quam, eu egestas turpis gravida nec | Sed scelerisque nec turpis vel viverra. Vivamus vitae pretium sapien |

## Bloques de código
### Bloque de código con comillas invertidas

```html
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <title>Documento HTML5 de ejemplo</title>
</head>
<body>
  <p>Prueba</p>
</body>
</html>
```

### Bloque de código indentado con cuatro espacios

    <!doctype html>
    <html lang="en">
    <head>
      <meta charset="utf-8">
      <title>Documento HTML5 de ejemplo</title>
    </head>
    <body>
      <p>Prueba</p>
    </body>
    </html>

### Bloque de código diff

```diff
[dependencies.bevy]
git = "https://github.com/bevyengine/bevy"
rev = "11f52b8c72fc3a568e8bb4a4cd1f3eb025ac2e13"
- features = ["dynamic"]
+ features = ["jpeg", "dynamic"]
```

### Bloque de código de una línea

```html
<p>Un párrafo</p>
```

## Tipos de listas

### Lista ordenada

1. Primer elemento
2. Segundo elemento
3. Tercer elemento

### Lista desordenada

* Elemento de lista
* Otro elemento
* Y otro elemento

### Lista anidada

* Fruta
  * Manzana
  * Naranja
  * Plátano
* Lácteos
  * Leche
  * Queso

## Otros elementos — abbr, sub, sup, kbd, mark

<abbr title="Graphics Interchange Format">GIF</abbr> es un formato de imagen de mapa de bits.

H<sub>2</sub>O

X<sup>n</sup> + Y<sup>n</sup> = Z<sup>n</sup>

Presiona <kbd>CTRL</kbd> + <kbd>ALT</kbd> + <kbd>Supr</kbd> para finalizar la sesión.

La mayoría de las <mark>salamandras</mark> son nocturnas y cazan insectos, gusanos y otras criaturas pequeñas.
