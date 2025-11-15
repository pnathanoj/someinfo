---
title: 3 Supervised land cover classification
parent: Case Studies
layout: home
nav_order: 3
---



```js
var landsat = ee.ImageCollection('LANDSAT/LC08/C02/T1_L2')
   .filterBounds(study_site)
   .filterDate('2023-01-01', '2023-05-31')
   .sort('CLOUD_COVER')
   .first();
```


```js
var landsat_vis = {
   bands: ['SR_B4', 'SR_B3', 'SR_B2'],
   min: 7000,
   max: 15000
}; 
```

```js
Map.centerObject(study_site, 12);
Map.addLayer(landsat, landsat_vis, 'L8 image');
Map.addLayer(study_site);
```

```js
var training_points = ee.FeatureCollection([WATER, BUILTUP, FARMS, NATURALVEG, BARESOILSAND]).flatten();
print(training_points);
```

```js
var bands = ['SR_B1', 'SR_B2', 'SR_B3', 'SR_B4', 'SR_B5', 'SR_B6', 'SR_B7','ST_B10'];
```

```js
var sample = landsat.select(bands)
   .sampleRegions({
      collection: training_points,
      properties: ['type'],
      scale: 30
});
print(sample);
```

```js
var classifier = ee.Classifier.smileCart().train({
   features: sample,
   classProperty: 'type',
   inputProperties: bands
});
```

```js
var classification = landsat.select(bands).classify(classifier); 
var classification_vis = {
   min: 0,
   max: 4,
   palette: ['green', 'red', 'blue', 'yellow', 'pink']
};
Map.addLayer(classification, classification_vis, 'classificationsupervCART');
```

```js
var traintest = sample.randomColumn();
var train_group = traintest.filter(ee.Filter.lt('random', 0.8));
var test_group = traintest.filter(ee.Filter.gte('random', 0.8));
print(train_group);
print(test_group);
```

```js
var classifier2 = ee.Classifier.smileCart().train({
   features: train_group,
   classProperty: 'type',
   inputProperties: bands
});
```

```js
var confusion_matrix =
test_group.classify(classifier2)
   .errorMatrix({
      actual: 'type',
      predicted: 'classification'
});
print('Confusion Matrix:', confusion_matrix);
print('Accuracy:', confusion_matrix.accuracy());
print('Producer Accuracy:', confusion_matrix.producersAccuracy());
print('User Accuracy:', confusion_matrix.consumersAccuracy());
print('Kappa:', confusion_matrix.kappa());
```

```js
var L8filtre = ee.ImageCollection("NASA/HLS/HLSL30/v002")
   .filterDate('2020-01-01', '2024-01-01')
   .filterBounds(study_site)
   .select(['B4', 'B5'])
   .filter(ee.Filter.lt('CLOUD_COVERAGE', 30));
```

```js
var farmsfun = function(image) {
  var imagemasked = image.updateMask(classification.eq(2));
  var ndvi = imagemasked.normalizedDifference(['B5', 'B4'])
   .rename('ndvi');
  return imagemasked.addBands(ndvi);
};
```

```js
var L8filtre_mask = L8filtre.map(farmsfun);
print(L8filtre_mask, 'L8filtre_mask');
```

```js
var graph = ui.Chart.image.series({
   imageCollection: L8filtre_mask,
   region: study_site,
   reducer: ee.Reducer.first(),
   scale: 30
});
print(graph);
```

----

[^1]: [It can take up to 10 minutes for changes to your site to publish after you push the changes to GitHub](https://docs.github.com/en/pages/setting-up-a-github-pages-site-with-jekyll/creating-a-github-pages-site-with-jekyll#creating-your-site).

[Just the Docs]: https://just-the-docs.github.io/just-the-docs/
[GitHub Pages]: https://docs.github.com/en/pages
[README]: https://github.com/just-the-docs/just-the-docs-template/blob/main/README.md
[Jekyll]: https://jekyllrb.com
[GitHub Pages / Actions workflow]: https://github.blog/changelog/2022-07-27-github-pages-custom-github-actions-workflows-beta/
[use this template]: https://github.com/just-the-docs/just-the-docs-template/generate
