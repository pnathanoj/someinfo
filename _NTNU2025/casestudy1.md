---
title: 1 Coldwave and mangroves
parent: Case Studies
layout: home
nav_order: 1
---

# Mapping low temperatures in relation to mangroves

## Coldwaves and mangroves

### Dynamics

Explain here

### Kang et al. (2025) study

Explain here 

## Mapping low temperatures in GEE

### Importing data on climate and state borders

The figure of Kang et al. (2025) includes two major datalayers in addition to their study sites: the outlines of the US borders, and the minimum temperature from November 2022 to March 2023. The internal borders of the USA can easily be obtained from the [FAO GAUL Level 1](https://developers.google.com/earth-engine/datasets/catalog/FAO_GAUL_2015_level1) dataset that includes all countries' internal borders. In this FeatureCollection, each Feature represents an individual level 1 (sub-national) administrative unit (such as a USA state). As shown in *Table Schema* tab of the dataset, it is possible to identify all Features that belong to the USA with *property* "ADM0_NAME". Therefore, we first import the FeatureCollection using ``ee.FeatureCollection()`` function. Then, we filter it using ``filter()`` of type ``ee.Filter.eq()`` (eq: equal). ``ee.Filter.eq`` will select all Feature for which the property "ADM0_NAME" value is "United States of America". Note that it is important to first check how the USA are designated in the "ADM0_NAME" property because it could have been "USA", "United_States_of_America", "Unites States Of America", etc.

```js
var FAOGAULL1 = ee.FeatureCollection('FAO/GAUL/2015/level1')
  .filter(ee.Filter.eq('ADM0_NAME','United States of America'));
```

Kang et al. (2025) have used the PRISM daily dataset to identify minimum temperature over the study period. [This dataset is thanksfully available in Earth Engine catalog](https://developers.google.com/earth-engine/datasets/catalog/OREGONSTATE_PRISM_AN81d). The *Bands* tab of the dataset shows that each daily Image has multiple bands: ``ppt`` for total daily precipitation (in mm), ``tmin`` for minimum daily temperature, etc. The spatial resolution is 4638.3 m and it is produced daily (1 Day). We import the ImageCollection using ``ee.ImageCollection()`` as the new variable ``PRISM_Temp`` and we will use a filter, ``ee.Filter.date()`` to filter all Images that were produced for dates between ``2022-11-01`` and ``2023-03-31``. Finally, we use the ``select()`` function that allows to select a subset of bands for all Images.

```js
var PRISM_Temp = ee.ImageCollection('OREGONSTATE/PRISM/AN81d')
  .filter(ee.Filter.date('2022-11-01', '2023-03-31'))
  .select('tmin');

print('PRISM_Temp:', PRISM_Temp);
```

### Reducing temperature data

The variable ``PRISM_Temp`` contains several Images corresponding to all days between November 2022 and March 2023. However, we seek to measured the minimum measured temperature across that period for each pixel. This can be easily carried out using a *reducer* that works across all Images of an ImageCollection to produce a single Image. Here, the reducer is ``ee.Reducer.min()`` that is applied using ``reduce()`` to the ImageCollection ``PRISM_Temp``.

```js
var Temp_min = PRISM_Temp.reduce(ee.Reducer.min());

print('Temp_min:', Temp_min);
```

### Visualization

To visualize the produced Image, we will ask the map panel to automatically move toward a coordinate in Southeast USA with a zoom level of 7 in order to see the whole state of Florida. This is carried out by the ``Map.setCenter()`` function that takes three arguments: longitude, latitude, and the zoom level.

```js
Map.setCenter(-82.295, 29.085, 7);
```

A color palette should be created before displaying the Image ``Temp_min``. The color palette is created as the variable ``TVis`` and will range from -7 to 0 (°C) as in Kang et al. (2025) figure. Accordingly, eight colors are provided in a List object using ``[`` and ``]``. Here, the colors are provided as hexadecimal values based on those used by Kang et al. (2025).

```js
var TVis = {
  min: -7,
  max: 0,
  palette: ['#1475B9', '#6E9ABD', '#AABEBD', '#E2E9BF’,  '#FFE3A4', '#FFA672', '#FE6F47', '#F72B22'],
};
```

To display the ``Temp_min`` Image, we use the ``Map.addLayer()`` function that includes several arguments. The first one is the Image name, followed by the palette, title, and whether it should be automatically displayed (**``false``** means that it will not be automatically displayed). Once the script is ran, the Image will be displayed if it is toogled on in the Map panel.

```js
Map.addLayer(Temp_min, TVis, 'Min temperature gradient', false);
```

{: .note }
Why do the colors not exactly match those in Kang et al. (2025)?

However, the displayed Image does not exactly match Kang et al. (2025) figure. In fact, that figure was based on categorized minimum temperature (*e.g.*, -6.5 and -6.31 are the same category, -0.3 and -0.9 are in another category) whereas the displayed Image use continuous temperature values between -7 and 0 (*e.g.*, -6.5 and -6.31 have different colors). To obtain a similar visualization as Kang et al. (2025) figure, we simply have to convert the decimal values (*e.g.*, -6.5 and -6.31) to integer values (*e.g., -6 and -6). This is carried out using the function ``int()``. Note that it is not necessary to create a new variable and that this can be done within ``Map.addLayer()``.


```js
Map.addLayer(Temp_min.int(), TVis, 'Min temperature categories', false);
```

{: .note }
Try other reducers such as max and mean.

Finally, we seek to display the states' borders. A new variable is created with the visualization parameters (black outline, transparent fill color) for the borders of the US states including a value of ``#00000000`` for ``fillColor`` to obtain a transparent fill color. It is then applied to ``FAOGAULL1`` FeatureCollection using ``style()`` function instead than using it as the second argument as done previously for ``Temp_min``. The second argument is an empty one ``{}`` that has to be nonetheless specified here because we need to match the order of the arguments of ``Map.addLayer``.

```js
var BorderVis = {
  color: 'black',
  width: 1.0,
  fillColor: "#00000000"
};

Map.addLayer(FAOGAULL1.style(BorderVis), {}, 'States', false);
```

### Raster export
The produced Image can be exported as a *raster* with ``Export.image.toDrive()`` function. However, we have to define the extent of the desired output (the *region* argument in ``Export.image.toDrive()``). If the region of interest (ROI) is a state of the USA, then it can be filtered from the ``FAOGAUL1`` FeatureCollection. Here, we will instead use the geometry editor in the map panel. Use the last button, the rectangle polygon, to draw a single rectangle including all of Florida, and rename it *ROI*.

Using the export function, we will define the following arguments. The *folder* will be the name of the Google Drive folder in which the raster will be exported, if that folder does not exist yet, it will be automatically created. The region is the *ROI* polygon, and the scale is the spatial resolution of the raster (here defined as PRISM spatial resolution).

```js
Export.image.toDrive({
  image: Temp_min.int(),
  description: 'Tempmin',
  folder: 'NTNU2025',
  region: ROI,
  scale: 4638,
  crs: 'EPSG:4326'
});
```

Once the script is ran, a new task appears in the *Tasks* tab of the *right* panel. The new task is titled "Tempmin" and may take 1-2 minutes to be processed.

## Relating temperature and vegetation change

### Importing multispectral data


### Production of a vegetation index


### Sampling
