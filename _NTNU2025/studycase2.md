---
title: Study Case 2
parent: Study Cases
layout: home
nav_order: 2
---

# Modeling potential species distribution based on species presence-absence data

## Presence-absence data

### What is it?

Information on species **presence-absence** (PA) is based on field observations that provide data relative to the presence or absence of a species at a given location. The information is derived from ground surveys or other means such as camera traps. This data is generally provided with precise coordinates of the PA observation (spatial point) and other additional information on the species name, site characteristics, observer, and date of observation. Besides PA data, there are also datasets containing only presence information (presence only, PO), however the function (Maximum Entropy, Maxent) used in GEE requires PA information to model a species distribution probability. Note that there are several discussions on the use of PA data such as in [Royle & Nichols (2003)](https://doi.org/10.1890/0012-9658(2003)084[0777:EAFRPA]2.0.CO;2), [Kent & Carmel (2011)](https://doi.org/10.1111/j.1472-4642.2011.00755.x), and [Brotons et al. (2004)](https://doi.org/10.1111/j.0906-7590.2004.03764.x), including in relation to Maxent ([Guillera-Arroita et al., 2014](https://doi.org/10.1111/2041-210X.12252)).

### Downloading PA data

This study case is based on one dataset provided by [Elith et al. (2020)](https://doi.org/10.17161/bi.v15i2.13384) on the distribution of 30 plant species across upper South America. The dataset can be downloaded at:

[https://osf.io/kwc4v/files/osfstorage](https://osf.io/kwc4v/files/osfstorage) > data > Records > test_pa > SAtest_pa.csv

### File structure

The file is a .csv that is structured as follow:

| group   | siteid      | x         | y        | sa01 | sa02 | sa03 | ... | sa30 |
|:--------|:------------|:----------|:---------|:-----|:-----|:-----|:----|:-----|
| plant   | allpahua    | -73.4167  | -3.95    | 0    | 0    | 1    | ... | 0    |
| plant   | alterdoc    | -54.9667  | -2.5     | 0    | 0    | 0    | ... | 0    |
| plant   | amotape     | -80.6167  | -4.15    | 0    | 0    | 0    | ... | 0    |
| ...     | ...         | ...       | ...      | ...  | ...  | ...  | ... | ...  |

Where *siteid* is the name of the site, *x* is the longitude, *y* is the latitude, and the remaining columns *sa01* to *sa30* are the PA information about the 30 species. In these columns, *0* denotes the absence of the species, whereas *1* indicates that the species was present.

## Distribution modeling in GEE

Open the [code editor](https://code.earthengine.google.com/).

### Importing PA data

To import the SAtest_pa.csv file, go to the *left panel* in **Assets** and click on New > CSV file (.csv).

In the new window, select the Source file and create a new **Asset ID** such as:
```
others/SA_spPA
```
where *others/* is a new folder that will contain the *SA_spPA* asset that you are importing.

Do not upload yet, and write *x* as the X column, and *y* as the Y column (longitude and latitude). You can then click on the UPLOAD button.

This new task is now running in the *right* panel's Tasks tab, first as an Unsubmitted task (file uploading) and then as a submitted task (server processing). Once the task is at the submitted task status, it is not needed to stay connected. Once the task has been completed, it will become blue. Errors will be clearly indicated with a red color. Once the task is completed, go to the *left* panel's Assets tab to refresh it and verify that the new asset is here.

Finally, hover the new asset with the cursor to show an arrow symbol that will allow you to import the asset in the new script in the Code Editor. Rename it ``testPA``.

### Visualize PA data

For species 10:

```js
Map.addLayer(speciesPA.filter('sa10 == 0'), {color: 'red'}, 'Training data (sp. absent)');

Map.addLayer(speciesPA.filter('sa10 == 1'), {color: 'blue'}, 'Training data (sp. present)');

Map.centerObject(speciesPA, 4);
```

### Training data

We first create a new variable ``trainingData`` that will be used to sample various parameters that could explain the species distribution.

```js
var trainingData = ee.FeatureCollection(speciesPA);
```

Several parameters affect species distributions such as climate, topography, barriers, or the presence of other organisms such as predators or humans.

To inform on climate characteristics, we use the [Worldclim BIO dataset](https://developers.google.com/earth-engine/datasets/catalog/WORLDCLIM_V1_BIO). The dataset includes several bands refering to different parameters such as Annual mean temperature (``bio01``), annual precipitation  (``bio12``), or the precipitation measured for the driest month  (``bio14``). The page of the dataset provides the units and a **scale** that need to be applied to the band to obtain the actual value. Here, the band ``bio01`` has a scale of 0.1.

```js
var worldclim = ee.Image('WORLDCLIM/V1/BIO');

var annualMeanTemperature = worldclim.select('bio01').multiply(0.1);

var annualPrecipitation = worldclim.select('bio12');

var driestMonthPrecipitation = worldclim.select('bio14');

var temperatureSeasonality = worldclim.select('bio04').multiply(0.01);
```

To inform on elevation, we use the digital elevation model (DEM) from the [CGIAR that is based on the SRTM](https://developers.google.com/earth-engine/datasets/catalog/CGIAR_SRTM90_V4).

```js
var elevation = ee.Image('USGS/SRTMGL1_003').select('elevation');
```

All the explanatory variables are combined in a single Image with 5 bands:

```js
var allData = ee.Image([elevation, annualMeanTemperature, annualPrecipitation, driestMonthPrecipitation, temperatureSeasonality]);
```

### Sampling and training

The ``sampleRegions()`` function allows to sample the values of the 5 bands of ``allData`` at the spatial points corresponding to the species PA data (``trainingData``). We precise the scale (in meters). It is clear that the variable ``trainingSample`` contains several Features that are the species PA spatial points that also includes the sampled climate and elevation data.

```js
var trainingSample = allData.sampleRegions({collection: trainingData, scale: 500});

print(trainingSample)
```

In order to use Maxent, we first need to specify the function ``ee.Classifier.amnhMaxent()`` in order to inform on the selected Classifier, and then use the function ``train`` that train the model of species 10 distribution (sa10's value 0 or 1) based on the climate and elevation parameters' values in the ``trainingSample``. These names of the selected parameters (as written in ``trainingSample``) need to be listed in ``inputProperties``: those are the same names as the band names of the ``allData`` Image.

The variable ``classifier`` is the distribution model produced based on Maxent and the training data.

```js
var classifier = ee.Classifier.amnhMaxent().train({
  features: trainingSample,
  classProperty: 'sa10',
  inputProperties: allData.bandNames()
});
```

### Distribution probability map

The distribution model ``classifier`` is then applied to the ``allData`` Image (climate and elevation parameters) in order to produce a map of the predicted distribution of species 10. This is carried out using the ``classify()`` function.

```js
var imageClassified = allData.classify(classifier);

print(imageClassified, 'Classified');
```

Finally, it is possible to visualize the distribution probability:

```js
Map.addLayer(imageClassified, {bands: 'probability', min: 0, max: 1}, 'Probability’);
```


{: .practice }
Try with other species such as species 01 and species 25.
