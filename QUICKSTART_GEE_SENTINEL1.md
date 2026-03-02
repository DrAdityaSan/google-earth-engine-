# Quickstart: Remote Sensing Project with Google Earth Engine (GEE) using Sentinel-1 SAR

This guide helps you build a practical starter workflow in **Google Earth Engine** with **Sentinel-1 SAR** imagery.

## 1) Project goal (example)
Build a simple flood/water-change detector by comparing Sentinel-1 backscatter before vs. after an event.

## 2) Prerequisites
- A Google account
- Access to Earth Engine Code Editor: <https://code.earthengine.google.com>
- Basic JavaScript familiarity (for GEE scripting)

---

## 3) Open GEE and define your area of interest (AOI)
In the Code Editor, start with:

```javascript
// Example AOI: replace with your own geometry
var aoi = ee.Geometry.Rectangle([90.30, 23.60, 90.55, 23.85]);
Map.centerObject(aoi, 11);
Map.addLayer(aoi, {color: 'red'}, 'AOI');
```

> Tip: You can draw geometry directly in the map and rename it (e.g., `geometry`).

---

## 4) Load and filter Sentinel-1 GRD
Use IW mode, dual polarization, and orbit direction consistency.

```javascript
var s1 = ee.ImageCollection('COPERNICUS/S1_GRD')
  .filterBounds(aoi)
  .filter(ee.Filter.eq('instrumentMode', 'IW'))
  .filter(ee.Filter.listContains('transmitterReceiverPolarisation', 'VV'))
  .filter(ee.Filter.listContains('transmitterReceiverPolarisation', 'VH'))
  .filter(ee.Filter.eq('orbitProperties_pass', 'DESCENDING'))
  .filter(ee.Filter.eq('resolution_meters', 10));

print('Sentinel-1 collection size:', s1.size());
```

---

## 5) Split into pre-event and post-event windows
Choose dates for your use case.

```javascript
var preStart  = '2021-07-01';
var preEnd    = '2021-07-20';
var postStart = '2021-07-21';
var postEnd   = '2021-08-10';

var pre = s1.filterDate(preStart, preEnd).median().clip(aoi);
var post = s1.filterDate(postStart, postEnd).median().clip(aoi);

print('Pre image bands:', pre.bandNames());
print('Post image bands:', post.bandNames());
```

---

## 6) Speckle reduction (simple and fast)
Apply focal median smoothing to VV and VH.

```javascript
var smooth = function(img) {
  return img
    .select(['VV', 'VH'])
    .focal_median({radius: 30, units: 'meters'})
    .copyProperties(img, img.propertyNames());
};

var preSmooth = smooth(pre);
var postSmooth = smooth(post);
```

---

## 7) Create change metrics
Typical flood/water response: backscatter drops in VV/VH.

```javascript
var dVV = postSmooth.select('VV').subtract(preSmooth.select('VV')).rename('dVV');
var dVH = postSmooth.select('VH').subtract(preSmooth.select('VH')).rename('dVH');

// Ratio in linear scale can be useful; convert dB -> linear first
var toLinear = function(dbImg) {
  return ee.Image(10.0).pow(dbImg.divide(10.0));
};

var preVVlin = toLinear(preSmooth.select('VV'));
var postVVlin = toLinear(postSmooth.select('VV'));
var vvRatio = postVVlin.divide(preVVlin).rename('VV_ratio');
```

---

## 8) Basic water/flood mask (threshold approach)
This is a starter rule; tune thresholds to local conditions.

```javascript
// Candidate water/flood pixels: large VV drop and low post-event VV
var floodMask = dVV.lt(-1.5)
  .and(postSmooth.select('VV').lt(-17))
  .selfMask();

// Optional cleanup
floodMask = floodMask
  .focal_min({radius: 20, units: 'meters'})
  .focal_max({radius: 20, units: 'meters'});
```

---

## 9) Visualize outputs

```javascript
var sarViz = {min: -25, max: 0};
var dViz = {min: -5, max: 5, palette: ['blue', 'white', 'red']};

Map.addLayer(preSmooth.select('VV'), sarViz, 'Pre VV');
Map.addLayer(postSmooth.select('VV'), sarViz, 'Post VV');
Map.addLayer(dVV, dViz, 'dVV (post-pre)');
Map.addLayer(floodMask, {palette: ['00FFFF']}, 'Flood mask');
```

---

## 10) Area statistics
Estimate flooded area in square kilometers.

```javascript
var pixelArea = ee.Image.pixelArea().divide(1e6); // km²
var floodArea = floodMask.multiply(pixelArea).reduceRegion({
  reducer: ee.Reducer.sum(),
  geometry: aoi,
  scale: 10,
  maxPixels: 1e10
});

print('Estimated flood area (km²):', floodArea);
```

---

## 11) Export results

```javascript
Export.image.toDrive({
  image: floodMask,
  description: 'GEE_S1_FloodMask',
  folder: 'GEE_exports',
  fileNamePrefix: 'flood_mask_s1',
  region: aoi,
  scale: 10,
  maxPixels: 1e13
});
```

---

## 12) Recommended next improvements
1. **Terrain correction / slope masking** in mountainous regions.
2. **Permanent water masking** using JRC Global Surface Water to isolate event-driven flood.
3. **Object-based cleanup** (connected components filtering).
4. **Validation** with optical imagery (Sentinel-2 where cloud-free) or reference flood maps.
5. **Time-series baseline** using monthly/seasonal median SAR to reduce false positives.

---

## Common pitfalls
- Mixing ascending and descending passes can introduce geometry-related differences.
- Comparing very different incidence angles can bias change detection.
- Thresholds vary by land cover and season; always calibrate locally.

---

## Minimal end-to-end script (copy/paste)

```javascript
var aoi = ee.Geometry.Rectangle([90.30, 23.60, 90.55, 23.85]);
Map.centerObject(aoi, 11);

var s1 = ee.ImageCollection('COPERNICUS/S1_GRD')
  .filterBounds(aoi)
  .filter(ee.Filter.eq('instrumentMode', 'IW'))
  .filter(ee.Filter.listContains('transmitterReceiverPolarisation', 'VV'))
  .filter(ee.Filter.listContains('transmitterReceiverPolarisation', 'VH'))
  .filter(ee.Filter.eq('orbitProperties_pass', 'DESCENDING'))
  .filter(ee.Filter.eq('resolution_meters', 10));

var pre = s1.filterDate('2021-07-01', '2021-07-20').median().clip(aoi);
var post = s1.filterDate('2021-07-21', '2021-08-10').median().clip(aoi);

var smooth = function(img) {
  return img.select(['VV', 'VH']).focal_median({radius: 30, units: 'meters'});
};

var preS = smooth(pre);
var postS = smooth(post);

var dVV = postS.select('VV').subtract(preS.select('VV')).rename('dVV');
var floodMask = dVV.lt(-1.5).and(postS.select('VV').lt(-17)).selfMask();

Map.addLayer(preS.select('VV'), {min: -25, max: 0}, 'Pre VV');
Map.addLayer(postS.select('VV'), {min: -25, max: 0}, 'Post VV');
Map.addLayer(dVV, {min: -5, max: 5, palette: ['blue', 'white', 'red']}, 'dVV');
Map.addLayer(floodMask, {palette: ['00FFFF']}, 'Flood mask');

Export.image.toDrive({
  image: floodMask,
  description: 'GEE_S1_FloodMask',
  region: aoi,
  scale: 10,
  maxPixels: 1e13
});
```

If you want, I can also provide:
- a **crop monitoring** version (VH/VV seasonal dynamics),
- a **rice mapping** starter pipeline, or
- the same workflow in the **Python Earth Engine API**.
