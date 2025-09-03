---
title: 2 Species distribution model
parent: Case Studies
layout: home
nav_order: 2
---

# Modeling potential species distribution based on species presence-absence data

## Presence-absence data

### What is it?

Information on species **presence-absence** (PA) is based on field observations that record whether a species is present or absent at a given location. The information is is typically derived from ground surveys or other methods, such as camera traps. This data generally include precise coordinates of the PA observation (spatial points) along with additional information such as species name, site characteristics, observer, and date of observation. 

![SpeciesPA](https://pnathanoj.github.io/someinfo/NTNU2025/speciespa.png)

In addition to PA data, there are datasets containing only presence information (presence only, PO). However the Maximum Entropy (Maxent) function used in GEE requires PA information to model a species distribution probabilities. Note that there are several discussions on the use of PA data such as in [Royle & Nichols (2003)](https://doi.org/10.1890/0012-9658(2003)084[0777:EAFRPA]2.0.CO;2), [Kent & Carmel (2011)](https://doi.org/10.1111/j.1472-4642.2011.00755.x), and [Brotons et al. (2004)](https://doi.org/10.1111/j.0906-7590.2004.03764.x), including in relation to Maxent ([Guillera-Arroita et al., 2014](https://doi.org/10.1111/2041-210X.12252)).

### Downloading PA data

This study case is based on a dataset provided by [Elith et al. (2020)](https://doi.org/10.17161/bi.v15i2.13384) on the distribution of 30 plant species across northern South America. The dataset can be downloaded at:

[https://osf.io/kwc4v/files/osfstorage](https://osf.io/kwc4v/files/osfstorage) > data > Records > test_pa > SAtest_pa.csv

### File structure

The file is a .csv that is structured as follow:

| group   | siteid      | x         | y        | sa01 | sa02 | sa03 | ... | sa30 |
|:--------|:------------|:----------|:---------|:-----|:-----|:-----|:----|:-----|
| plant   | allpahua    | -73.4167  | -3.95    | 0    | 0    | 1    | ... | 0    |
| plant   | alterdoc    | -54.9667  | -2.5     | 0    | 0    | 0    | ... | 0    |
| plant   | amotape     | -80.6167  | -4.15    | 0    | 0    | 0    | ... | 0    |
| ...     | ...         | ...       | ...      | ...  | ...  | ...  | ... | ...  |

Where *siteid* represents the name of the site, *x* is the longitude, *y* is the latitude, and the remaining columns *sa01* to *sa30* contain the PA information for the 30 species. In these columns, *0* denotes the absence of the species, whereas *1* indicates its presence.

## Distribution modeling in GEE

Open the [code editor](https://code.earthengine.google.com/).

### Importing PA data

To import the SAtest_pa.csv file, go to the *left panel* under **Assets** and click on New > CSV file (.csv).

In the new window, select the *Source file* and create a new **Asset ID** such as:
```
others/SA_spPA
```
where *others/* is a new folder that will contain the *SA_spPA* asset you are importing.

Do not upload yet, and specify "x" as the X column and "y" as the Y column (longitude and latitude). You can then click the UPLOAD button.

This new task is now running in the Task tab of *right* panel, initially as an Unsubmitted task (file uploading) and then as a Submitted task (server processing). Once the task reaches the submitted status, it is not necessary to stay connected. When the task is complete, it will turn blue. Errors will be clearly indicated in red. After completion, go to the Assets tab in the *left* panel to refresh it, and verify that the new asset is here.

Finally, hover over the new asset with the cursor to reveal an arrow symbol, which allows you to import the asset into the new script in the Code Editor. Rename it ``testPA``.

### Visualize PA data

For species 10:

```js
Map.addLayer(speciesPA.filter('sa10 == 0'), {color: 'red'}, 'Training data (sp. absent)');

Map.addLayer(speciesPA.filter('sa10 == 1'), {color: 'blue'}, 'Training data (sp. present)');

Map.centerObject(speciesPA, 4);
```

### Training data

We first create a new variable, ``trainingData``, which will be used to sample various parameters that may explain the species distribution.

```js
var trainingData = ee.FeatureCollection(speciesPA);
```

Several parameters affect species distributions including climate, topography, barriers, and the presence of other organisms, such as predators or humans.

{: .note }
What datasets can you find in Earth Engine Catalog?


To provide information on climate characteristics, we use the [Worldclim BIO dataset](https://developers.google.com/earth-engine/datasets/catalog/WORLDCLIM_V1_BIO). The dataset includes several bands corresponding to different parameters, such as annual mean temperature (``bio01``), annual precipitation  (``bio12``), or the precipitation measured during the driest month  (``bio14``). The dataset page provides the units and a **scale** that must be applied to each band to obtain the actual values. For example, the band ``bio01`` has a scale of 0.1.

```js
var worldclim = ee.Image('WORLDCLIM/V1/BIO');

var annualMeanTemperature = worldclim.select('bio01').multiply(0.1);

var annualPrecipitation = worldclim.select('bio12');

var driestMonthPrecipitation = worldclim.select('bio14');

var temperatureSeasonality = worldclim.select('bio04').multiply(0.01);
```

![Selectmultiply](https://pnathanoj.github.io/someinfo/NTNU2025/selectmultiply.png)

To inform on elevation, we use the digital elevation model (DEM) from [CGIAR, based on the SRTM](https://developers.google.com/earth-engine/datasets/catalog/CGIAR_SRTM90_V4).

```js
var elevation = ee.Image('USGS/SRTMGL1_003').select('elevation');
```

All the explanatory variables are combined into a single Image with 5 bands:

```js
var allData = ee.Image([elevation, annualMeanTemperature, annualPrecipitation, driestMonthPrecipitation, temperatureSeasonality]);
```

### Sampling and training

The ``sampleRegions()`` function allows sampling the values of the 5 bands of ``allData`` at the spatial points corresponding to the species PA data (``trainingData``). We specify the scale (in meters). 

It is clear that the resulting variable ``trainingSample`` contains several Features representing the species PA spatial points, along with the sampled climate and elevation data.

```js
var trainingSample = allData.sampleRegions({collection: trainingData, scale: 500});

print(trainingSample)
```

In order to use Maxent, we first specify the function ``ee.Classifier.amnhMaxent()`` in order to inform on the selected classifier, and then use the ``train`` function to train the species distribution model for species 10 (based on the values of ``sa10``, 0 or 1) using the climate and elevation parameters' values in ``trainingSample``. The names of the selected parameters (as written in ``trainingSample``) need to be provided as a List in ``inputProperties``: these are the same names as the band names in ``allData`` Image.

The variable ``classifier`` stores the distribution model produced by Maxent using the training data.

```js
var classifier = ee.Classifier.amnhMaxent().train({
  features: trainingSample,
  classProperty: 'sa10',
  inputProperties: allData.bandNames()
});
```

### Distribution probability map

The distribution model ``classifier`` is then applied to the ``allData`` Image (containing climate and elevation parameters) to produce a map of the predicted distribution of species 10. This is done using the ``classify()`` function.

```js
var imageClassified = allData.classify(classifier);

print(imageClassified, 'Classified');
```

Finally, the distribution probability can be visualized:

```js
Map.addLayer(imageClassified, {bands: 'probability', min: 0, max: 1}, 'Probability’);
```


{: .note }
Try with other species such as species 01 and species 25.
