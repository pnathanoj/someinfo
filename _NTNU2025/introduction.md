---
title: Introduction to GEE
layout: home
nav_order: 1
---

# Introduction to Google Earth Engine

The following sections are intended to support the short introduction to Google Earth Engine (GEE) delivered to students of the Department of Life Sciences at NTNU (Fall 2025).

This half-day lecture is partly based on the week-long introduction to GEE delivered to the students at University of Parakou in Spring 2025 as part of the [OBSYDYA project](https://www.obsydya.org/).

{: .links }
> Code Editor: [https://code.earthengine.google.com/](https://code.earthengine.google.com/)
> 
> GEE Data Catalog: [developers.google.com/earth-engine/datasets/catalog](https://developers.google.com/earth-engine/datasets/catalog)
> 
> API Docs: [https://developers.google.com/earth-engine/apidocs/ee-image](https://developers.google.com/earth-engine/apidocs/ee-image)

# Lists, Arrays

```js
var a = ee.Number(10);
var b = ee.Number(20);
var c = ee.Number(530);
var List1 = ee.List([a, b, b, a, c]);
var Array1 = ee.Array([a, b, b, a, c]);

print(List1); print(Array1);
```


```js
var List2 = ee.List([[a, b, b, a, c], [a, c, c, c, a]]);
var Array2 = ee.Array([[a, b, b, a, c], [a, c, c, c, a]]);

print(List2); print(Array2);
```

```js
var List2 = ee.List([[a, b, b, a, c], [a, c, c]]);
var Array2 = ee.Array([[a, b, b, a, c], [a, c, c]]);

print(List2); print(Array2);
```

```js
var Array2 = ee.Array([[a, b, b, a, c], [a, c, c, c, a]]);
var Arraytf = Array2.gte([[1, 2, 3, 11, 40],[10, 15, 45, 0, 9]]);

print(Arraytf);
```


----

[Just the Docs]: https://just-the-docs.github.io/just-the-docs/
[GitHub Pages]: https://docs.github.com/en/pages
[README]: https://github.com/just-the-docs/just-the-docs-template/blob/main/README.md
[Jekyll]: https://jekyllrb.com
[GitHub Pages / Actions workflow]: https://github.blog/changelog/2022-07-27-github-pages-custom-github-actions-workflows-beta/
[use this template]: https://github.com/just-the-docs/just-the-docs-template/generate
