// Importar la geometría
var geometry = ee.Geometry.Polygon(
  [[[-69.86235293925914, -13.01289455800908],
    [-69.8874155002943, -12.789344373103333],
    [-69.90183505595836, -12.776286636498533],
    [-70.03332767070445, -12.834204265228196],
    [-70.12877139629039, -12.962713582932887],
    [-70.0741830784193, -13.00888045322229],
    [-70.00208530009898, -13.00152109237729],
    [-69.86475619853648, -13.028615842479189],
    [-69.86235293925914, -13.01289455800908]]]);

// Función para filtrar imágenes por fecha y nubosidad
function getFilteredImages(year) {
  return ee.ImageCollection('COPERNICUS/S2')
    .filterBounds(geometry)
    .filter(ee.Filter.calendarRange(year, year, 'year'))
    .filter(ee.Filter.lt('CLOUDY_PIXEL_PERCENTAGE', 15))
    .map(function(image) {
      return image.clip(geometry);
    });
}

// Obtener las imágenes filtradas para 2019 y 2023
var images2019 = getFilteredImages(2019).median();
var images2023 = getFilteredImages(2023).median();

// Función para calcular el NDVI
function calculateNDVI(image) {
  var ndvi = image.normalizedDifference(['B8', 'B4']).rename('NDVI');
  return image.addBands(ndvi);
}

// Calcular el NDVI para las imágenes de 2019 y 2023
var ndvi2019 = calculateNDVI(images2019);
var ndvi2023 = calculateNDVI(images2023);

// Función para detectar cambios entre dos imágenes
function detectChange(image1, image2) {
  var ndvi1 = image1.select('NDVI');
  var ndvi2 = image2.select('NDVI');
  var diff = ndvi2.subtract(ndvi1).rename('NDVI_Change');
  return diff;
}

// Detectar cambios entre las imágenes de 2019 y 2023
var changeImage = detectChange(ndvi2019, ndvi2023);

// Función para calcular el área deforestada en hectáreas
function calculateDeforestedArea(changeImage) {
  // Consideramos una disminución en el NDVI mayor a un umbral como deforestación
  var threshold = -0.2; // Ajusta este valor según sea necesario
  var deforested = changeImage.lt(threshold);
  
  // Calcular el área de píxeles deforestados
  var pixelArea = ee.Image.pixelArea();
  var deforestedArea = deforested.multiply(pixelArea).divide(10000); // Convertir a hectáreas
  
  // Sumar el área total deforestada dentro de la geometría
  var stats = deforestedArea.reduceRegion({
    reducer: ee.Reducer.sum(),
    geometry: geometry,
    scale: 10,
    maxPixels: 1e13
  });

  // Manejar casos donde el resultado sea null
  var area = ee.Number(stats.get('NDVI_Change', 0)); // Usar 0 si es null
  return area;
}

// Calcular el área deforestada entre 2019 y 2023
var deforestedArea = calculateDeforestedArea(changeImage);

// Imprimir el área deforestada en la consola
print('Deforested Area between 2019 and 2023 (ha):', deforestedArea);

// Función para calcular el área no vegetada en hectáreas
function calculateNonVegetatedArea(ndviImage) {
  // Consideramos un valor de NDVI menor a 0.2 como no vegetación
  var nonVegetated = ndviImage.lt(0.2);
  
  // Calcular el área de píxeles no vegetados
  var pixelArea = ee.Image.pixelArea();
  var nonVegetatedArea = nonVegetated.multiply(pixelArea).divide(10000); // Convertir a hectáreas
  
  // Sumar el área total no vegetada dentro de la geometría
  var stats = nonVegetatedArea.reduceRegion({
    reducer: ee.Reducer.sum(),
    geometry: geometry,
    scale: 10,
    maxPixels: 1e13
  });

  // Manejar casos donde el resultado sea null
  var area = ee.Number(stats.get('NDVI', 0)); // Usar 0 si es null
  return area;
}

// Calcular el área no vegetada para 2019 y 2023
var nonVegetatedArea2019 = calculateNonVegetatedArea(ndvi2019);
var nonVegetatedArea2023 = calculateNonVegetatedArea(ndvi2023);

// Imprimir el área no vegetada en la consola
print('Non-Vegetated Area in 2019 (ha):', nonVegetatedArea2019);
print('Non-Vegetated Area in 2023 (ha):', nonVegetatedArea2023);

// Añadir el mapa de cambio al mapa para visualización
Map.centerObject(geometry, 10);
Map.addLayer(changeImage, {min: -0.5, max: 0.5, palette: ['red', 'white', 'green']}, 'NDVI Change 2019-2023');

// Visualizar las imágenes originales y el NDVI
var visParams = {'bands':['B4','B3','B2'],'min':0,'max':3000,'gamma':1.4};
Map.addLayer(images2019, visParams, '2019 Mosaic');
Map.addLayer(images2023, visParams, '2023 Mosaic');

// Exportar el mapa de cambio
Export.image.toDrive({
  image: changeImage,
  description: 'NDVI_Change_2019_2023',
  scale: 10,
  region: geometry,
  maxPixels: 1e13,
  fileFormat: 'GeoTIFF'
});
