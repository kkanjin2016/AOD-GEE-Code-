# AOD-GEE-Code-
# GEE code for calculating Optical Aerosol Depth from MODIS Data

//Boundary
var table = Study_Area

// Specify the start and end dates for the data extraction period.

var startDate = '2022-01-01'; // Start date of the period you're interested in
var endDate = '2022-12-31'; // End date of the period you're interested in

// Load the MODIS AOD dataset.
// Replace 'MODIS/006/MCD19A2_GRANULES' with the specific AOD dataset you're interested in.
// Ensure the dataset and variable names are correct.

var dataset = ee.ImageCollection('MODIS/006/MCD19A2_GRANULES')
                .filterDate(startDate, endDate)
                .filterBounds(table);

// Apply the scale factor and mask for the AOD values.

var scaledAOD = dataset.map(function(image) {
  var aod = image.select('Optical_Depth_047').multiply(0.001).clip(table); // Scaling down AOD values
    return aod.copyProperties(image, ['system:time_start']);
});

// Visualizing the AOD data over Ohio.
// Define visualization parameters.

var visParams = {
  min: 0.0,
  max: 0.6,
  palette: ['blue', 'green', 'yellow', 'orange', 'red']
};

// Add the scaled AOD layer to the map.

Map.centerObject(table, 6); // Adjust the zoom level to your preference.
Map.addLayer(scaledAOD.mean(), visParams, 'Mean AOD');

// Calculate monthly averages

var months = ee.List.sequence(1, 12);  // A list of months from 1 to 12
var monthlyAverages = ee.FeatureCollection(months.map(function(month) {
  var filteredMonth = scaledAOD.filter(ee.Filter.calendarRange(month, month, 'month'));
  var mean = filteredMonth.mean();
  var meanValue = mean.reduceRegion({
    reducer: ee.Reducer.mean(),
    geometry: table,
    scale: 1000, // Adjust scale based on the resolution you need and computational limits
    maxPixels: 1e9
  });
  return ee.Feature(null, meanValue.set('month', month));
}));

// Export the monthly averages as a CSV file

Export.table.toDrive({
  collection: monthlyAverages,
  description: 'Monthly_AOD_Averages_Ohio_2022',
  fileFormat: 'CSV'
});

// Optionally, export the AOD data for Ohio.

Export.image.toDrive({
  image: scaledAOD.mean(),
  description: 'AOD_Over_Ohio20220101_20221230',
  scale: 1000, 
  region: table,
  fileFormat: 'GeoTIFF',
  maxPixels: 1e9
});






