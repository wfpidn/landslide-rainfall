# Half-hourly IMERG rainfall during landslide event

During 2018, landslide dominated natural disasters occurred in Central Java. The Regional Disaster Management Agency (BPBD) of Central Java Province recorded that there were about 2,000 landslides in this area.

Most landslide is preceeded by continuous extreme rainfall for few days, as most of the area isn't located near ground weather station, therefore rainfall records are often not available.

Recent development in opendata allows more access to high resolution climate/weather data and to computing platform that allows users to run geospatial analysis in the cloud. This leads to further exploration like never before.

Open access to 30 mins temporal rainfall data at [Google Earth Engine](https://earthengine.google.com/) platform provided detail information on rainfall intensity prior to the landslide. Such information help advancing research on rainfall model or identification of threshold for extreme rainfall that could trigger a landslide.


## Data Source

Half hourly IMERG at Earth Engine Data Catalogue - [https://developers.google.com/earth-engine/datasets/catalog/NASA_GPM_L3_IMERG_V06](https://developers.google.com/earth-engine/datasets/catalog/NASA_GPM_L3_IMERG_V06)

Landslide event in Magelang, Central Java - Indonesia during 2018. Compiled by Department of Environmental Geography, Faculty of Geography - Universitas Gadjah Mada. Available in CSV format with column structure: ID, Lon, Lat, Day, DD, MM, YYYY, TimeWIB

[Landslide](./data/idn_nhr_ls_3308_magelang_2018_p_example.csv)


## Script

The script below describe how to extract 30-minute rainfall from NASA [GPM-IMERG](https://gpm.nasa.gov/GPM), based on points location and convert it into CSV file using Google Earth Engine code editor.

``` js
// List of coordinates from Landslide events
var LS = ee.FeatureCollection("users/bennyistanto/datasets/table/idn_nhr_ls_3308_magelang_2018_p_example");
// LS = LS.filter(ee.Filter.lte('ID', 2)); // lite version

// Load table to layer
Map.addLayer(LS, {color: "#ff0000"}, "Landlside location", true);
Map.centerObject(LS, 11);

// Import NASA GPM IMERG 30 minute data 
var imerg = ee.ImageCollection('NASA/GPM_L3/IMERG_V06');

// Parsing date from CSV
function parseDate(p) {
  var d = p.getNumber('DD');
  var m = p.getNumber('MM');
  var y = p.getNumber('YYYY');
  var dt = y.format('%d')
    .cat('-').cat(m.format('%02d'))
    .cat('-').cat(d.format('%02d'));
  return ee.Date(dt);
}

var result = LS.map(function(p) {
  var id = p.getNumber('ID');
  var point = p.geometry();
  var coord = point.coordinates();
  var dt = parseDate(p);
  var start = dt.advance(-10, 'days'); // 10-days before
  var end = dt.advance(1, 'days'); // 1-day after
  var precipHH = imerg.filterBounds(point)
    .filterDate(start, end)
    .select('precipitationCal');
  var timeSeries = precipHH.map(function(image) {
    var date = image.date().format('yyyy-MM-dd hh:mm');
    var value = image
      .clip(point)
      .reduceRegion({
        reducer: ee.Reducer.mean(),
        scale: 30
      }).get('precipitationCal'); 
    return ee.Feature(null, { 
      coord_id: id, 
      lon: coord.get(0), 
      lat: coord.get(1),
      date: date, 
      value: value
    });
  });
  return timeSeries;
});

var flatResult = ee.FeatureCollection(result).flatten();

// Export the result as CSV in Google Drive
Export.table.toDrive({
  collection: flatResult,
  description:'landslide_rainfall',
  folder:'GEE',
  selectors: 'coord_id, lon, lat, date, value', 
  fileFormat: 'CSV'
});
```

GEE [link](https://code.earthengine.google.com/f55d6b0417351e0582f7558d1de5715b)

![GEE](./img/landslide_rainfall.png)

GEE script for 1 point simulation available via this [link](https://code.earthengine.google.com/47dede746d68d7d1a603d49540aa4805). You need to fill coordinate (line 12) and set the landslide date (line 16)


## Output

30-minutes of rainfall that occurred 10-days before landslide. Generated using GEE [Landslide Rainfall](./data/landslide_rainfall.csv)


## About

This is part of research and development of threshold for extreme rainfall that could trigger a landslide event in Indonesia. Reference: [https://wfpidn.github.io/ERM/ls/#extreme-rainfall-triggered-landslide-alert](https://wfpidn.github.io/ERM/ls/#extreme-rainfall-triggered-landslide-alert)


## Contact

For further information about Extreme Rainfall triggering a Landslide research and development, please contact:

- Benny Istanto and Ridwan Mulyadi
	
	Vulnerability Analysis and Mapping Unit<br>
	UN World Food Programme - Indonesia

- Guruh Samodra, Dr. Eng.

	Department of Environmental Geography<br>
	Faculty of Geography, Universitas Gadjah Mada.
