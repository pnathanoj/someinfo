---
title: 1 Coldwave and mangroves
parent: Case Studies
layout: home
nav_order: 1
---

# Mapping low temperatures in relation to mangroves

## Coldwaves and mangroves

### Dynamics

Mangroves are tropical to subtropical ecosystems that are found at the ecotone between the land and the sea. At the edge of their distribution, winter temperatures can occasionally approach 0°C and <0°C, leading to physiological stress and mortality in mangrove trees. These coldwaves do not occur every year, allowing mangroves to establish and grow until the next cold event. These events play an important role in explaining shifts between mangroves and saltmarsh ecosystems. Saltmarshes are less affected by low temperatures than mangroves, but they may gradually be replaced by mangroves if no coldwaves occur.

### Kang et al. (2025) study

[Kang and colleagues](https://doi.org/10.1111/1365-2745.14440) studied the responses of different mangrove tree species to low temperatures across Florida to identify their resistance and resilience to cold, and to predict future mangrove expansion under climate change.

## Mapping low temperatures in GEE

### Importing data on climate and state borders

The figure 2 of Kang et al. (2025) includes two major data layers in addition to their study sites: the outlines of U.S. borders and the minimum temperature from November 2022 to March 2023. The internal borders of the United States can be obtained from the [FAO GAUL Level 1](https://developers.google.com/earth-engine/datasets/catalog/FAO_GAUL_2015_level1) dataset which includes the internal borders of all countries. In this FeatureCollection, each Feature represents an individual level 1 (subnational) administrative unit (such as a U.S. state). 

As shown in the *Table Schema* tab of the dataset, it is possible to identify all Features belonging to the United States using the *property* "ADM0_NAME". Therefore, we first import the FeatureCollection with the ``ee.FeatureCollection()`` function. Then, we filter it using ``filter()`` of type ``ee.Filter.eq()`` (eq: equal). The ``ee.Filter.eq`` operation will select all Features whose "ADM0_NAME" property is equal to "United States of America". Note that it is important to first check how the United States are designated in the "ADM0_NAME" property, as it could appear as "USA", "United_States_of_America", "United States Of America", etc.

```js
var FAOGAULL1 = ee.FeatureCollection('FAO/GAUL/2015/level1')
  .filter(ee.Filter.eq('ADM0_NAME','United States of America'));
```
![Reducers](https://pnathanoj.github.io/someinfo/NTNU2025/filter.png)

Kang et al. (2025) used the PRISM daily dataset to identify minimum temperatures over the study period. [This dataset is conveniently available in Earth Engine catalog](https://developers.google.com/earth-engine/datasets/catalog/OREGONSTATE_PRISM_ANd). The *Bands* tab of the dataset shows that each daily Image has multiple bands: ``ppt`` for total daily precipitation (in mm), ``tmin`` for minimum daily temperature, etc. The spatial resolution is 4638.3 m and it is produced daily (1 Day). We import the ImageCollection using ``ee.ImageCollection()`` as a new variable ``PRISM_Temp`` and we use a filter, ``ee.Filter.date()`` to filter all Images produced for dates between ``2022-11-01`` and ``2023-03-31``. Finally, we use the ``select()`` function that allows to select a subset of bands from all Images.

```js
var PRISM_Temp = ee.ImageCollection('OREGONSTATE/PRISM/ANd')
  .filter(ee.Filter.date('2022-11-01', '2023-03-31'))
  .select('tmin');

print('PRISM_Temp:', PRISM_Temp);
```

### Reducing temperature data

The variable ``PRISM_Temp`` contains several Images representing all days between November 2022 and March 2023. However, we seek to measure the minimum temperature recorded across that period for each pixel. This can be easily achieved using a *reducer*, which operates across all Images of an ImageCollection to produce a single Image. Here, the reducer is ``ee.Reducer.min()`` that is applied using ``reduce()`` to the ImageCollection ``PRISM_Temp``.

```js
var Temp_min = PRISM_Temp.reduce(ee.Reducer.min());

print('Temp_min:', Temp_min);
```
![Reducers](https://pnathanoj.github.io/someinfo/NTNU2025/reducer.png)


### Visualization

To visualize the resulting Image, we set the map panel to automatically center on a coordinate in the southeastern USA with a zoom level of 7 in order to see the entire state of Florida. This is done using the ``Map.setCenter()`` function, which takes three arguments: longitude, latitude, and zoom level.

```js
Map.setCenter(-82.295, 29.085, 7);
```

A color palette should be created before displaying the Image ``Temp_min``. The color palette is stored as the variable ``TVis`` and is set to range from -7 to 0 (°C), following the figure in Kang et al. (2025). Accordingly, eight colors are specified in a List object using ``[`` and ``]``. Here, these colors are provided as hexadecimal values based on those used by Kang et al. (2025).

```js
var TVis = {
  min: -7,
  max: 0,
  palette: ['#1475B9', '#6E9ABD', '#AABEBD', '#E2E9BF', '#FFE3A4', '#FFA672', '#FE6F47', '#F72B22'],
};
```

To display the ``Temp_min`` Image, we use the ``Map.addLayer()`` function, which takes several arguments. The first one is the Image name, followed by the visualization parameters (palette), the title, and a *flag* indicating whether it should be automatically displayed (**``false``** means that it will not be automatically displayed). Once the script is ran, the Image will be displayed when toogled on in the Map panel.

```js
Map.addLayer(Temp_min, TVis, 'Min temperature gradient', false);
```

{: .note }
Why do the colors not exactly match those in Kang et al. (2025)?

However, the displayed Image does not exactly match Kang et al. (2025) figure 2. In fact, that figure was based on categorized minimum temperatures (*e.g.*, -6.5 and -6.31 fall into the same category, -0.3 and -0.9 fall into another category), whereas the displayed Image uses continuous temperature values between -7 and 0 (*e.g.*, -6.5 and -6.31 are shown with different colors). 

![Reducers](https://pnathanoj.github.io/someinfo/NTNU2025/gradient.png)

To obtain a visualization similar to Kang et al. (2025), we simply need to convert decimal values (*e.g.*, -6.5 and -6.31) to integer values (*e.g., -6 and -6). This is carried out using the ``int()`` function. Note that it is not necessary to create a new variable and that this conversion can be applied directly within ``Map.addLayer()``.


```js
Map.addLayer(Temp_min.int(), TVis, 'Min temperature categories', false);
```

![Reducers](https://pnathanoj.github.io/someinfo/NTNU2025/minimumtemperature.png)

{: .note }
Try other reducers such as max and mean.

Finally, we want to display the states borders. A new variable is created to store the visualization parameters (black outline, transparent fill color) for the borders of the US states including a value of ``#00000000`` assigned to ``fillColor`` to obtain a transparent fill color. The parameters are then applied to ``FAOGAULL1`` FeatureCollection using the ``style()`` function, rather than passing them as the second argument as done previously for ``Temp_min`` ([see this thread and reply by Gorelick N.](https://gis.stackexchange.com/questions/470500/set-fill-color-of-vector-polygon-layer-to-transparent-in-google-earth-engine)). Thus, the second argument is left as an empty object ``{}`` that has to be nonetheless specified to maintain the correct argument order in ``Map.addLayer``.

```js
var BorderVis = {
  color: 'black',
  width: 1.0,
  fillColor: "#00000000"
};

Map.addLayer(FAOGAULL1.style(BorderVis), {}, 'States', false);
```

### Raster export
The resulting Image can be exported as a *raster* with the ``Export.image.toDrive()`` function. However, we must first define the extent of the desired output (the *region* argument in ``Export.image.toDrive()``). If the region of interest (ROI) is a state of the USA, it can be filtered from the ``FAOGAUL1`` FeatureCollection. Here, we will instead use the geometry editor in the map panel. Select the rectangle polygon tool (last button) to draw a single rectangle that covers all of Florida, and rename it *ROI*.

Using the export function, we define the following arguments. The *folder* specifies the name of the Google Drive folder where the raster will be exported, if that folder does not exist yet, it will be automatically created. The *region* is the "ROI" polygon, and the *scale* is the spatial resolution of the raster (here set to match the PRISM spatial resolution).

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

Once the script is run, a new task appears in the *Tasks* tab of the *right* panel. The new task is titled "Tempmin" and may take 1-2 minutes to be process.

## Relating temperature and vegetation change

### Importing multispectral data


### Production of a vegetation index


### Sampling
