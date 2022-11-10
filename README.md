# Evaluating the efficiency of coarser to finer resolution multispectral satellites in mapping paddy rice fields using GEE implementation



![https://doi.org/10.1038/s41598-022-17454-y](/data/img/paper.png)

# Paper Link

- PDF: https://www.nature.com/articles/s41598-022-17454-y.pdf
- DOI: https://doi.org/10.1038/s41598-022-17454-y

See my other work at my [Portfolio Website](https://waleedgeo.github.io/)
# Data used

 - ## [S2 Rice Classification Raster Geotiff (10-20m)](/data/rasters/S2_Final.tif)

 - ## [Landsat-8 Rice Classification Raster Geotiff (30m)](/data/rasters/L8_Final.tif)
 - ## [MODIS Rice Classification Raster Geotiff (250m)](/data/rasters/MODIS_Final.tif)


# GEE Codes for Paddy Rice Classification

## Rice Classification using Sentinel-2
```javascript
    // Applied to all

    // Samples Buffer
    /*
    var buffercheck = rice.buffer(20)
    var buffercheck1  = vegetation.buffer(20)
    var buffercheck2  = water.buffer(20)
    Map.addLayer(buffercheck, {}, 'rice buffer', false)
    Map.addLayer(buffercheck1, {}, 'veg buffer', false)
    Map.addLayer(buffercheck2, {}, 'water buffer', false)

    */
    var bufferPoly = function(feature) {
    return feature.buffer(20);   // substitute in your value of Z here
    }; 

    //merge training samples
    var training_merge = water.merge(vegetation).merge(rice).merge(builtup);
    //create buffer
    var training_buffer = training_merge.map(bufferPoly);
    Map.addLayer(training_buffer, {}, 'training buffer')

    Map.setCenter(70.94, 29.728, 7);

    //--------------------Sentinel-2-------------------------

    //Cloud Masking Function
    function maskS2clouds(image) {
    var qa = image.select('QA60');

    // Bits 10 and 11 are clouds and cirrus, respectively.
    var cloudBitMask = 1 << 10;
    var cirrusBitMask = 1 << 11;

    // Both flags should be set to zero, indicating clear conditions.
    var mask = qa.bitwiseAnd(cloudBitMask).eq(0)
        .and(qa.bitwiseAnd(cirrusBitMask).eq(0));

    return image.updateMask(mask).divide(10000);
    }

    // Load Sentinel-2 L2A imagery and filter it to June/July to Oct/Nov.
    var s2 = ee.ImageCollection('COPERNICUS/S2_SR')
                                    .filterDate('2020-07-01', '2020-10-01')
                                    .filter(ee.Filter.lt('CLOUDY_PIXEL_PERCENTAGE',20))
                                    .map(maskS2clouds)
                                    .median()
                                    .clipToCollection(SA);
    var s2vp = {
    bands: ['B8', 'B4', 'B3'],
    min: 0.0,
    max: 0.3
    }
    Map.addLayer(s2, s2vp, 'Sentinel-2 FCC', false);

    /////////////// classification starts here//////////////////////

    var bands_s2 = ['B2', 'B3', 'B4', 'B8', 'B11', 'B12'];


    var r_training_s2 = s2.select(bands_s2).sampleRegions({
        collection :training_buffer,
        properties: ['LC','random'],
        scale: 10
    });
    //random
    var train_all_s2 = r_training_s2.randomColumn('random',2015);

    var training_s2 = train_all_s2.filterMetadata('random','less_than', 0.7);

    var testing_s2= train_all_s2.filterMetadata('random','not_less_than', 0.7);

    // Train the classifier
    var trainc_s2 = ee.Classifier.smileRandomForest(115).train({
    features:training_s2,
    classProperty: 'LC',
    inputProperties: bands_s2
    });
    // training classifier
    var c_s2 = s2.select(bands_s2).classify(trainc_s2);
    // validation classifier using testing samples

    // -------------- Accuracy S2

    // validation classifier using testing samples
    var valc_s2 = testing_s2.classify(trainc_s2);
    // accuracy matrix
    var em_s2 = valc_s2.errorMatrix('LC','classification');// Error Matrix
    var oa_s2 = em_s2.accuracy(); // Overall accuracy
    var ks_s2 = em_s2.kappa(); // Kappa statistic
    var ua_s2 = em_s2.consumersAccuracy().project([1]); // Consumer's accuracy
    var pa_s2 = em_s2.producersAccuracy().project([0]); // Producer's accuracy
    var f1_s2 = (ua_s2.multiply(pa_s2).multiply(2.0)).divide(ua_s2.add(pa_s2)); // F1-statistic
    print('Error Matrix, Sentinel-2: ', em_s2);
    print('Overall Accuracy, Sentinel-2Sentinel-2: ', oa_s2);
    print('Kappa Statistic, Sentinel-2: ', ks_s2);
    print('User\'s Accuracy (rows), Sentinel-2:', ua_s2);
    print('Producer\'s Accuracy (cols), Sentinel-2:', pa_s2);
    print('F1 Score, Sentinel-2: ', f1_s2);

    Map.addLayer(c_s2,
    {min :0, max: 1, palette: ['F7F7F7','0EE000']},
    'classified rice S2');

    //Export
    Export.image.toDrive({
            image: c_s2.clip(SA).multiply(100).uint8(),
            description: "S2_Final",
            scale: 10,
            region: SA,
            maxPixels:1e12
            })
```

[GEE Link for S2](https://code.earthengine.google.com/20e54dbb3fa1eca561cc54b10a595b6c) | [JavaScript Code for S2](/data/javascript_codes/s2_full.js)


## Rice Classification using Landsat-8
```javascript
    // Applied to all
 
    // Samples Buffer
    /*
    var buffercheck = rice.buffer(20)
    var buffercheck1  = vegetation.buffer(20)
    var buffercheck2  = water.buffer(20)
    Map.addLayer(buffercheck, {}, 'rice buffer', false)
    Map.addLayer(buffercheck1, {}, 'veg buffer', false)
    Map.addLayer(buffercheck2, {}, 'water buffer', false)

    */
    var bufferPoly = function(feature) {
    return feature.buffer(20);   // substitute in your value of Z here
    }; 

    //merge training samples
    var training_merge = water.merge(vegetation).merge(rice).merge(builtup);
    //create buffer
    var training_buffer = training_merge.map(bufferPoly);

    Map.setCenter(70.94, 29.728, 7);

    //------------------- Landsat-8--------------------------

    function maskL8sr(image) {
    // Bits 3 and 5 are cloud shadow and cloud, respectively.
    var cloudShadowBitMask = (1 << 3);
    var cloudsBitMask = (1 << 5);
    // Get the pixel QA band.
    var qa = image.select('pixel_qa');
    // Both flags should be set to zero, indicating clear conditions.
    var mask = qa.bitwiseAnd(cloudShadowBitMask).eq(0)
                    .and(qa.bitwiseAnd(cloudsBitMask).eq(0));
    return image.updateMask(mask);
    }

    var l8 = ee.ImageCollection('LANDSAT/LC08/C01/T1_SR')
                                    .filterDate('2020-07-01', '2020-10-01')
                                    .map(maskL8sr)
                                    .median()
                                    .clipToCollection(SA);
    var l8vp = {
    bands: ['B5', 'B4', 'B3'],
    min: 0,
    max: 3000,
    
    };

    Map.addLayer(l8, l8vp, 'Landsat8 FCC', false);


    var bands_l8 = ['B2', 'B3', 'B4', 'B5', 'B6', 'B7'];


    var r_training_l8 = l8.select(bands_l8).sampleRegions({
        collection :training_buffer,
        properties: ['LC','random'],
        scale: 10
    });
    //random
    var train_all_l8 = r_training_l8.randomColumn('random',2015);

    var training_l8 = train_all_l8.filterMetadata('random','less_than', 0.7);

    var testing_l8= train_all_l8.filterMetadata('random','not_less_than', 0.7);

    // Train the classifier
    var trainc_l8 = ee.Classifier.smileRandomForest(100).train({
    features:training_l8,
    classProperty: 'LC',
    inputProperties: bands_l8
    });

    // training classifier
    var c_l8 = l8.select(bands_l8).classify(trainc_l8);

    //-----------Accuracy L8

    // validation classifier using testing samples
    var valc_l8 = testing_l8.classify(trainc_l8);
    // accuracy matrix
    // accuracy matrix
    var em_l8 = valc_l8.errorMatrix('LC','classification');// Error Matrix
    var oa_l8 = em_l8.accuracy(); // Overall accuracy
    var ks_l8 = em_l8.kappa(); // Kappa statistic
    var ua_l8 = em_l8.consumersAccuracy().project([1]); // Consumer's accuracy
    var pa_l8 = em_l8.producersAccuracy().project([0]); // Producer's accuracy
    var f1_l8 = (ua_l8.multiply(pa_l8).multiply(2.0)).divide(ua_l8.add(pa_l8)); // F1-statistic
    print('Error Matrix, Landsat-8: ', em_l8);
    print('Overall Accuracy, Landsat-8: ', oa_l8);
    print('Kappa Statistic, Landsat-8: ', ks_l8);
    print('User\'s Accuracy (rows), Landsat-8:', ua_l8);
    print('Producer\'s Accuracy (cols), Landsat-8:', pa_l8);
    print('F1 Score, Landsat-8: ', f1_l8);

    Map.addLayer(c_l8,
    {min :0, max: 1, palette: ['F7F7F7','0EE000']},
    'classified rice L8');

    //Export
    Export.image.toDrive({
            image: c_l8.clip(SA).multiply(100).uint8(),
            description: "L8_Final",
            scale: 30,
            region: SA,
            maxPixels:1e12
            })
```
[GEE Link for L8](https://code.earthengine.google.com/2a1f97945ae3a874eb9feeefd47fbec6) | [JavaScript Code for L8](/data/javascript_codes/l8_full.js)

## Rice Classification using MODIS

```javascript
    // Applied to all
    
    // Samples Buffer
    /*
    var buffercheck = rice.buffer(20)
    var buffercheck1  = vegetation.buffer(20)
    var buffercheck2  = water.buffer(20)
    Map.addLayer(buffercheck, {}, 'rice buffer', false)
    Map.addLayer(buffercheck1, {}, 'veg buffer', false)
    Map.addLayer(buffercheck2, {}, 'water buffer', false)

    */
    var bufferPoly = function(feature) {
    return feature.buffer(20);   // substitute in your value of Z here
    }; 

    //merge training samples
    var training_merge = water.merge(vegetation).merge(rice).merge(builtup);
    //create buffer
    var training_buffer = training_merge.map(bufferPoly);

    Map.setCenter(70.94, 29.728, 7);

    var modis = ee.ImageCollection('MODIS/006/MOD09GQ')
                                    .filterDate('2020-07-01', '2020-10-01')
                                    .median()
                                    .clipToCollection(SA);
    var falseColorVis = {
    min: -100.0,
    max: 8000.0,
    bands: ['sur_refl_b02', 'sur_refl_b02', 'sur_refl_b01'],
    };


    Map.addLayer(modis, falseColorVis, 'False Color', false);

    /////////////// classification starts here//////////////////////
    var bands_modis = ['sur_refl_b02', 'sur_refl_b01'];


    var r_training_modis = modis.select(bands_modis).sampleRegions({
        collection :training_buffer,
        properties: ['LC','random'],
        scale: 10
    });
    //random
    var train_all_modis = r_training_modis.randomColumn('random',2015);

    var training_modis = train_all_modis.filterMetadata('random','less_than', 0.7);

    var testing_modis= train_all_modis.filterMetadata('random','not_less_than', 0.7);

    // Train the classifier
    var trainc_modis = ee.Classifier.smileRandomForest(100).train({
    features:training_modis,
    classProperty: 'LC',
    inputProperties: bands_modis
    });

    // training classifier
    var c_modis = modis.select(bands_modis).classify(trainc_modis);

    // validation classifier using testing samples
    var valc_modis = testing_modis.classify(trainc_modis);

    // accuracy matrix
    var em_modis = valc_modis.errorMatrix('LC','classification');// Error Matrix
    var oa_modis = em_modis.accuracy(); // Overall accuracy
    var ks_modis = em_modis.kappa(); // Kappa statistic
    var ua_modis = em_modis.consumersAccuracy().project([1]); // Consumer's accuracy
    var pa_modis = em_modis.producersAccuracy().project([0]); // Producer's accuracy
    var f1_modis = (ua_modis.multiply(pa_modis).multiply(2.0)).divide(ua_modis.add(pa_modis)); // F1-statistic
    print('Error Matrix, modis: ', em_modis);
    print('Overall Accuracy, modis: ', oa_modis);
    print('Kappa Statistic, modis: ', ks_modis);
    print('User\'s Accuracy (rows), modis:', ua_modis);
    print('Producer\'s Accuracy (cols), modis:', pa_modis);
    print('F1 Score, modis: ', f1_modis);

    Map.addLayer(c_modis,
    {min :0, max: 1, palette: ['F7F7F7','0EE000']},
                'classified rice MODIS');


    //Export
    Export.image.toDrive({
            image: c_modis.clip(SA).multiply(100).uint8(),
            description: "MODIS_Final",
            scale: 250,
            region: SA,
            maxPixels:1e12
            })
```
[GEE Link for MODIS](https://code.earthengine.google.com/b991eb4d42c084b33629b6329b77bc18) | [JavaScript Code for MODIS](/data/javascript_codes/modis_full.js)