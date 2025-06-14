# RUSLE-Based Soil Erosion Estimation Using Google Earth Engine

This repository contains a Google Earth Engine (GEE) script for estimating annual soil loss using the Revised Universal Soil Loss Equation (RUSLE). The model integrates precipitation, soil characteristics, terrain features, vegetation cover, and land use practices. The script also computes class-wise area distribution and sub-basin-wise statistics. This specific implementation is applied to the **Umran basin of Meghalaya**, but you can modify the Area of Interest (AOI) to suit your own study region.

## 📌 Model Equation

The core equation used is the **RUSLE (Revised Universal Soil Loss Equation)**:

```
A = R × K × LS × C × P
```

Where:

* **A** = Estimated annual soil loss (tonnes per hectare per year)
* **R** = Rainfall erosivity factor
* **K** = Soil erodibility factor
* **LS** = Topographic factor (slope length and steepness)
* **C** = Cover management factor
* **P** = Support practice factor

*Source: Renard et al., 1997*

---

## 🌍 Study Area

The area of interest (AOI) in this project is the **Umran basin of Meghalaya**, defined using HydroSHEDS Level 12 basin boundaries. You can change the `mainID` to extract different major river basins or upload your own shapefile or geometry.

```javascript
var dataset = ee.FeatureCollection("WWF/HydroSHEDS/v1/Basins/hybas_12")
var mainID = 4120031730 // Example basin ID (replace for your own AOI)
var aoi = table;
```

---

## 📈 R Factor: Rainfall Erosivity

Derived using CHIRPS precipitation data:

```
R = 0.363 × P + 79
```

Where `P` is the total annual precipitation.

```javascript
var current = CHIRPS.filterDate(date1, date2).select('precipitation').sum().clip(aoi);
var R = ee.Image(current.multiply(0.363).add(79)).rename('R');
```

*Source: Roose, 1977*

---

## 🧪 K Factor: Soil Erodibility

Based on soil raster classification and a lookup formula for K values:

```
K = f(soil_code) using conditionals based on soil raster codes
```

```javascript
var K = soil.expression("(b('soil') > 11) ? 0.0053 : ... : 0").rename('K');
```

*Source: FAO, 2003; Wischmeier and Smith, 1978*

---

## ⛰️ LS Factor: Slope Length and Steepness

The LS factor is calculated as:

```
LS = (0.76 + 0.53 × S + 0.076 × S²) × √(L / 22.13)
```

Where:

* `S` = slope in percentage
* `L` = slope length (fixed as 500 m in this case)

```javascript
var slope = ee.Terrain.slope(elevation).clip(aoi).divide(180).multiply(Math.PI).tan().multiply(100);
var LS4 = Math.sqrt(500/100);
var LS = ((slope.multiply(0.53)).add(slope.pow(2).multiply(0.076)).add(0.76)).multiply(LS4).rename("LS");
```

*Source: Desmet and Govers, 1996*

---

## 🌿 C Factor: Vegetative Cover

Based on NDVI (from Sentinel-2):

```
C = exp[-2 × NDVI / (1 - NDVI)]
```

C values are normalized between 0 and 1.

```javascript
var image_ndvi = s2.normalizedDifference(['B8','B4']).rename("NDVI");
var C1 = image_ndvi.multiply(-2);
var C = (C1.divide(ee.Image(1).subtract(image_ndvi))).exp().normalize().rename('C');
```

*Source: Durigon et al., 2014*

---

## 🚜 P Factor: Land Use Management

P is assigned based on LULC class and slope as per FAO recommendations:

```
P = f(LULC, slope) using nested conditionals
```

```javascript
var P = lulc_slope.expression("(b('lulc') < 11) ? 0.8 : ... : 1").rename('P');
```

*Source: Morgan, 2005; FAO, 2003*

---

## 🧼 Soil Loss Computation

Once all factors are derived, final soil loss is computed as:

```
A = R × K × LS × C × P
```

```javascript
var soil_loss = ee.Image(R.multiply(K).multiply(LS).multiply(C).multiply(P)).rename("Soil Loss")
```

---

## 📊 Output

* **Soil Loss Map** with visual classification
* **Pie chart** for erosion class distribution
* **Tabular export** (CSV) of sub-basin wise mean soil loss and area statistics
* **Map Legend Panel** for UI visualization

---

## 📄 Export

Results can be exported to Google Drive as CSV files using `Export.table.toDrive` for class-wise and sub-basin-wise summaries.

---

## 🧐 Notes

* All calculations are scale-dependent; use appropriate spatial resolution (\~500 m).
* Modify parameters (e.g., slope length, alpha-beta for NDVI-to-C conversion) to match regional characteristics.

---

## 📁 Files

* `soil_loss_model.js`: Full GEE script implementing the RUSLE workflow
* `README.md`: This documentation

---

## 📧 Contact

For questions or collaboration, connect with [Suman Bhowmick](https://github.com/Suman1801).

---

## 📜 License

MIT License — you are free to use, share, and modify with attribution.

---

## 📚 References

* Desmet, P. J. J., & Govers, G. (1996). A GIS procedure for automatically calculating the USLE LS factor on topographically complex landscape units. *Journal of Soil and Water Conservation*, 51(5), 427–433.

* Durigon, V. L., Carvalho, D. F., Antunes, M. A. H., Oliveira, P. T. S., & Fernandes, M. M. (2014). NDVI time series for monitoring RUSLE cover management factor in a tropical watershed. *International Journal of Remote Sensing*, 35(2), 441–453. [https://doi.org/10.1080/01431161.2013.871081](https://doi.org/10.1080/01431161.2013.871081)

* FAO. (2003). *Manual for Soil Description*. Food and Agriculture Organization of the United Nations, Rome.

* Morgan, R. P. C. (2005). *Soil erosion and conservation* (3rd ed.). Wiley-Blackwell.

* Renard, K. G., Foster, G. R., Weesies, G. A., McCool, D. K., & Yoder, D. C. (1997). *Predicting soil erosion by water: A guide to conservation planning with the Revised Universal Soil Loss Equation (RUSLE)*. USDA Agricultural Handbook No. 703.

* Roose, E. (1977). Application of the universal soil loss equation of Wischmeier and Smith in West Africa. In *Soil Conservation and Management in the Humid Tropics* (pp. 177–187). Wiley.

* Wischmeier, W. H., & Smith, D. D. (1978). *Predicting rainfall erosion losses: A guide to conservation planning*. USDA Agricultural Handbook No. 537.
