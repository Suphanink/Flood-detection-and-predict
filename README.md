# Flood-detection-and-predict
Flooded Areas (สีฟ้า) – แสดงบริเวณที่ตรวจพบน้ำท่วมในช่วงเวลานั้น 
High-Risk Flood Zones (สีแดง) – เป็นพื้นที่ที่น้ำท่วมและอยู่ในที่ลุ่มต่ำ ซึ่งมีความเสี่ยงว่าน้ำจะขยายวงกว้างได้อีก
// 1. กำหนดพื้นที่(เปลี่ยนตามพื้นที่ๆสนใจ)
var aoi = ee.Geometry.Rectangle([100.3, 13.6, 100.9, 14.2]);

// 2. Sentinel-1 SAR
var sar = ee.ImageCollection('COPERNICUS/S1_GRD')
  .filterBounds(aoi)
  .filterDate('2024-10-01', '2024-10-10')
  .filter(ee.Filter.listContains('transmitterReceiverPolarisation', 'VV'))
  .filter(ee.Filter.eq('instrumentMode', 'IW'))
  .select('VV')
  .median();

// 3. แปลงเป็น dB
var sar_db = sar.log10().multiply(10);

// 4. Flood mask จาก threshold
var threshold = -17;
var flood = sar_db.lt(threshold);

// 5. Remove permanent water (JRC)
var jrc = ee.Image('JRC/GSW1_3/GlobalSurfaceWater').select('occurrence');
var permanentWater = jrc.gt(90);
var flood_mask = flood.updateMask(permanentWater.not());

// 6. DEM elevation
var dem = ee.Image("USGS/SRTMGL1_003").clip(aoi);
var lowland = dem.lt(5);  // เปลี่ยนเป็น <8 ก็ได้ถ้าอยากกว้างขึ้น

// 7. จุดสีแดง = น้ำท่วม + พื้นที่ต่ำ
var highRisk = flood_mask.and(lowland);
var highRiskBuffered = highRisk.focal_max(1);  // ขยายจุดแดงให้ดูง่ายขึ้น

// 8. แสดงผล
Map.centerObject(aoi, 10);
Map.addLayer(sar_db, {min: -25, max: 0}, 'SAR dB');
Map.addLayer(flood_mask.updateMask(flood_mask), {palette: ['blue']}, 'Flooded Area');
Map.addLayer(highRiskBuffered.updateMask(highRiskBuffered), {palette: ['red']}, 'High Risk Flood Zones');

// 9. ตรวจสอบว่ามีจุดแดงหรือไม่
print('High Risk Pixel Count:', highRisk.reduceRegion({
  reducer: ee.Reducer.sum(),
  geometry: aoi,
  scale: 30,
  maxPixels: 1e9
}));
