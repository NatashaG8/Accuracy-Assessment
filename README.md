# Accuracy-Assessment
#To find the country boundaries, use this link: https://gadm.org/download_country.html 

# To have an appreciation of what one should expect from the code, a pdf document with results is provided #

                                                   ##### CHECKING THE COUNTRY SHAPEFILE #####
                                                                ## PARAGUAY ##

import geopandas as gpd

# Define the path to the ZIP archive and the shapefile inside it -for Paraguay-
shapefile_path = "C:\\Users\\49176\\Downloads\\ParaguayShape.zip"  #use the path to the zip file you downloaded from the github path provided
shapefile_inside_zip = "/vsizip/{}/ParaguayShape.shp".format(shapefile_path)

# Load the shapefile
aoi = gpd.read_file(shapefile_inside_zip)

# Perform checks
print(f"Number of features: {len(aoi)}")
print("First feature:")
print(aoi.iloc[0])

                                                ##### CHECKING THE COUNTRY SHAPEFILE #####
                                                                ## ZAMBIA ##

import geopandas as gpd

# Define the path to the ZIP archive and the shapefile inside it
shapefile_path = "C:\\Users\\49176\\Downloads\\ZambiaShape.zip"  #use the path to the zip file you downloaded from the github path provided
shapefile_inside_zip = "/vsizip/{}/ZambiaShape.shp".format(shapefile_path)

# Load the shapefile
aoi = gpd.read_file(shapefile_inside_zip)

# Perform checks
print(f"Number of features: {len(aoi)}")
print("First feature:")
print(aoi.iloc[0])

                                             ##### CHECKING THE COUNTRY SHAPEFILE #####
                                                                ## ZIMBABWE##

import geopandas as gpd

# Define the path to the ZIP archive and the shapefile inside it
shapefile_path = "C:\\Users\\49176\\Downloads\\ZimbabweShape.zip"  #use the path to the zip file you downloaded from the github path provided
shapefile_inside_zip = "/vsizip/{}/ZimbabweShape.shp".format(shapefile_path)

# Load the shapefile
aoi = gpd.read_file(shapefile_inside_zip)

# Perform checks
print(f"Number of features: {len(aoi)}")
print("First feature:")
print(aoi.iloc[0])

                                   ####  RAW DATASET IMAGES BEFORE RE-CLASSIFICATION ####
                                                        #### PARAGUAY ####

import geopandas as gpd
import ee
import folium
import geemap

# Authenticate and Initialize Earth Engine
ee.Authenticate()
ee.Initialize()

# Load the shapefile as AOI
shapefile_path = "C:\\Users\\49176\\Downloads\\ParaguayShape.zip"
shapefile_inside_zip = f"/vsizip/{shapefile_path}/ParaguayShape.shp"
aoi_gdf = gpd.read_file(shapefile_inside_zip)
print(f"Number of features in AOI: {len(aoi_gdf)}")
print("First feature of AOI:")
print(aoi_gdf.iloc[0])

# Convert AOI to Earth Engine FeatureCollection
aoi = geemap.gdf_to_ee(aoi_gdf)

# Dynamically compute AOI centroid for map centering
aoi_centroid = aoi.geometry().centroid().coordinates().getInfo()
center = [aoi_centroid[1], aoi_centroid[0]]  # Convert to [latitude, longitude]

# Create a folium map
map = folium.Map(location=center, zoom_start=6)

# Define a function to add Earth Engine layers to folium map
def add_ee_layer(self, ee_object, vis_params, name):
    """Function to add Earth Engine layers to a folium map."""
    if isinstance(ee_object, ee.ImageCollection):
        ee_object = ee_object.mosaic()  # Combine into a single image
    map_id_dict = ee.Image(ee_object).getMapId(vis_params)
    folium.TileLayer(
        tiles=map_id_dict['tile_fetcher'].url_format,
        attr="Map Data &copy; Google Earth Engine",
        name=name,
        overlay=True,
        control=True,
    ).add_to(self)

# Bind the method to the folium map instance
folium.Map.add_ee_layer = add_ee_layer

# Add AOI boundary to the map
map.add_ee_layer(aoi.style(color="red", width=2), {}, "AOI Boundary")

# Define visualization function with fixes
def visualize_reclassification(dataset_name):
    """Adds dataset layers to the folium map."""
    if dataset_name == "DynamicWorld":
        dynamic_world = ee.ImageCollection('GOOGLE/DYNAMICWORLD/V1') \
            .filterBounds(aoi) \
            .filterDate('2020-01-01', '2020-12-31')
        raw_image = dynamic_world.select('trees').mean().clip(aoi).reproject(crs='EPSG:4326')
        vis_params = {'min': 0, 'max': 1, 'palette': ['white', 'green']}
        map.add_ee_layer(raw_image, vis_params, "DynamicWorld - Trees")
    elif dataset_name == "PALSAR":
        palsar = ee.ImageCollection("JAXA/ALOS/PALSAR/YEARLY/FNF4") \
            .filterBounds(aoi) \
            .filterDate('2020-01-01', '2020-12-31') \
            .mosaic()
        raw_image = palsar.select('fnf').clip(aoi).reproject(crs='EPSG:4326', scale=100)
        vis_params = {
            'min': 1,
            'max': 4,
            'palette': ['#00b200', '#83ef62', '#ffff99', '#0000ff']
        }
        map.add_ee_layer(raw_image, vis_params, "PALSAR - Forest/Non-Forest")
    elif dataset_name == "GFW":
        hansen_data = ee.Image('UMD/hansen/global_forest_change_2022_v1_10')
        treecover2000 = hansen_data.select('treecover2000').clip(aoi)
        loss = hansen_data.select('lossyear').clip(aoi)
        
        # Create a loss mask for the year 2020 only
        loss_mask_2020 = loss.eq(20)
        
        # Calculate forest cover after 2020 loss by excluding areas of 2020 loss
        forest_cover_after_loss_2020 = treecover2000.updateMask(loss_mask_2020.Not())
        
        vis_params = {'min': 0, 'max': 100, 'palette': ['white', 'green']}
        map.add_ee_layer(forest_cover_after_loss_2020, vis_params, "GFW - Forest Cover After 2020 Loss")
    elif dataset_name == "ESA":
        esa_image = ee.ImageCollection('ESA/WorldCover/v100') \
            .filterDate('2020-01-01', '2020-12-31') \
            .first().select('Map').clip(aoi).reproject(crs='EPSG:4326', scale=100)
        vis_params = {
            'min': 10,
            'max': 100,
            'palette': [
                '#006400', '#ffbb22', '#ffff4c', '#f096ff', '#fa0000',
                '#b4b4b4', '#f0f0f0', '#0064c8', '#0096a0', '#00cf75', '#fae6a0'
            ]
        }
        map.add_ee_layer(esa_image, vis_params, "ESA - WorldCover")

# Visualize datasets
datasets = ["DynamicWorld", "PALSAR", "GFW", "ESA"]
for dataset in datasets:
    visualize_reclassification(dataset)

# Add layer control and save the map
folium.LayerControl().add_to(map)
#map.save("visualized_map_with_loss_2020.html")
map

                                        ####  RAW DATASET IMAGES BEFORE RE-CLASSIFICATION ######
                                                        #### ZAMBIA #####

import geopandas as gpd
import ee
import folium

# Authenticate and Initialize Earth Engine
ee.Authenticate()
ee.Initialize()

# Load the shapefile as AOI
shapefile_path = "C:\\Users\\49176\\Downloads\\ZambiaShape.zip"
shapefile_inside_zip = f"/vsizip/{shapefile_path}/ZambiaShape.shp"
aoi_gdf = gpd.read_file(shapefile_inside_zip)
print(f"Number of features in AOI: {len(aoi_gdf)}")
print("First feature of AOI:")
print(aoi_gdf.iloc[0])

# Convert AOI to Earth Engine FeatureCollection
aoi = geemap.gdf_to_ee(aoi_gdf)

# Dynamically compute AOI centroid for map centering
aoi_centroid = aoi.geometry().centroid().coordinates().getInfo()
center = [aoi_centroid[1], aoi_centroid[0]]  # Convert to [latitude, longitude]

# Create a folium map
map = folium.Map(location=center, zoom_start=6)

# Define a function to add Earth Engine layers to folium map
def add_ee_layer(self, ee_object, vis_params, name):
    """Function to add Earth Engine layers to a folium map."""
    if isinstance(ee_object, ee.ImageCollection):
        ee_object = ee_object.mosaic()  # Combine into a single image
    map_id_dict = ee.Image(ee_object).getMapId(vis_params)
    folium.TileLayer(
        tiles=map_id_dict['tile_fetcher'].url_format,
        attr="Map Data &copy; Google Earth Engine",
        name=name,
        overlay=True,
        control=True,
    ).add_to(self)

# Bind the method to the folium map instance
folium.Map.add_ee_layer = add_ee_layer

# Add AOI boundary to the map
map.add_ee_layer(aoi.style(color="red", width=2), {}, "AOI Boundary")

# Define visualization function with fixes
def visualize_reclassification(dataset_name):
    """Adds dataset layers to the folium map."""
    if dataset_name == "DynamicWorld":
        dynamic_world = ee.ImageCollection('GOOGLE/DYNAMICWORLD/V1') \
            .filterBounds(aoi) \
            .filterDate('2020-01-01', '2020-12-31')
        raw_image = dynamic_world.select('trees').mean().clip(aoi).reproject(crs='EPSG:4326')
        vis_params = {'min': 0, 'max': 1, 'palette': ['white', 'green']}
        map.add_ee_layer(raw_image, vis_params, "DynamicWorld - Trees")
    elif dataset_name == "PALSAR":
        palsar = ee.ImageCollection("JAXA/ALOS/PALSAR/YEARLY/FNF4") \
            .filterBounds(aoi) \
            .filterDate('2020-01-01', '2020-12-31') \
            .mosaic()
        raw_image = palsar.select('fnf').clip(aoi).reproject(crs='EPSG:4326', scale=100)
        vis_params = {
            'min': 1,
            'max': 4,
            'palette': ['#00b200', '#83ef62', '#ffff99', '#0000ff']
        }
        map.add_ee_layer(raw_image, vis_params, "PALSAR - Forest/Non-Forest")
    elif dataset_name == "GFW":
        hansen_data = ee.Image('UMD/hansen/global_forest_change_2022_v1_10')
        treecover2000 = hansen_data.select('treecover2000').clip(aoi)
        loss = hansen_data.select('lossyear').clip(aoi)
        
        # Create a loss mask for the year 2020 only
        loss_mask_2020 = loss.eq(20)
        
        # Calculate forest cover after 2020 loss by excluding areas of 2020 loss
        forest_cover_after_loss_2020 = treecover2000.updateMask(loss_mask_2020.Not())
        
        vis_params = {'min': 0, 'max': 100, 'palette': ['white', 'green']}
        map.add_ee_layer(forest_cover_after_loss_2020, vis_params, "GFW - Forest Cover After 2020 Loss")
    elif dataset_name == "ESA":
        esa_image = ee.ImageCollection('ESA/WorldCover/v100') \
            .filterDate('2020-01-01', '2020-12-31') \
            .first().select('Map').clip(aoi).reproject(crs='EPSG:4326', scale=100)
        vis_params = {
            'min': 10,
            'max': 100,
            'palette': [
                '#006400', '#ffbb22', '#ffff4c', '#f096ff', '#fa0000',
                '#b4b4b4', '#f0f0f0', '#0064c8', '#0096a0', '#00cf75', '#fae6a0'
            ]
        }
        map.add_ee_layer(esa_image, vis_params, "ESA - WorldCover")

# Visualize datasets
datasets = ["DynamicWorld", "PALSAR", "GFW", "ESA"]
for dataset in datasets:
    visualize_reclassification(dataset)

# Add layer control and save the map
folium.LayerControl().add_to(map)
#map.save("visualized_map_with_loss_2020.html")
map

                                      ####  RAW DATASET IMAGES BEFORE RE-CLASSIFICATION ######
                                                         #### ZIMBABWE #####

import geopandas as gpd
import ee
import folium

# Authenticate and Initialize Earth Engine
ee.Authenticate()
ee.Initialize()

# Load the shapefile as AOI
shapefile_path = "C:\\Users\\49176\\Downloads\\ZimbabweShape.zip"
shapefile_inside_zip = f"/vsizip/{shapefile_path}/ZimbabweShape.shp"
aoi_gdf = gpd.read_file(shapefile_inside_zip)
print(f"Number of features in AOI: {len(aoi_gdf)}")
print("First feature of AOI:")
print(aoi_gdf.iloc[0])

# Convert AOI to Earth Engine FeatureCollection
aoi = geemap.gdf_to_ee(aoi_gdf)

# Dynamically compute AOI centroid for map centering
aoi_centroid = aoi.geometry().centroid().coordinates().getInfo()
center = [aoi_centroid[1], aoi_centroid[0]]  # Convert to [latitude, longitude]

# Create a folium map
map = folium.Map(location=center, zoom_start=6)

# Define a function to add Earth Engine layers to folium map
def add_ee_layer(self, ee_object, vis_params, name):
    """Function to add Earth Engine layers to a folium map."""
    if isinstance(ee_object, ee.ImageCollection):
        ee_object = ee_object.mosaic()  # Combine into a single image
    map_id_dict = ee.Image(ee_object).getMapId(vis_params)
    folium.TileLayer(
        tiles=map_id_dict['tile_fetcher'].url_format,
        attr="Map Data &copy; Google Earth Engine",
        name=name,
        overlay=True,
        control=True,
    ).add_to(self)

# Bind the method to the folium map instance
folium.Map.add_ee_layer = add_ee_layer

# Add AOI boundary to the map
map.add_ee_layer(aoi.style(color="red", width=2), {}, "AOI Boundary")

# Define visualization function with fixes
def visualize_reclassification(dataset_name):
    """Adds dataset layers to the folium map."""
    if dataset_name == "DynamicWorld":
        dynamic_world = ee.ImageCollection('GOOGLE/DYNAMICWORLD/V1') \
            .filterBounds(aoi) \
            .filterDate('2020-01-01', '2020-12-31')
        raw_image = dynamic_world.select('trees').mean().clip(aoi).reproject(crs='EPSG:4326')
        vis_params = {'min': 0, 'max': 1, 'palette': ['white', 'green']}
        map.add_ee_layer(raw_image, vis_params, "DynamicWorld - Trees")
    elif dataset_name == "PALSAR":
        palsar = ee.ImageCollection("JAXA/ALOS/PALSAR/YEARLY/FNF4") \
            .filterBounds(aoi) \
            .filterDate('2020-01-01', '2020-12-31') \
            .mosaic()
        raw_image = palsar.select('fnf').clip(aoi).reproject(crs='EPSG:4326', scale=100)
        vis_params = {
            'min': 1,
            'max': 4,
            'palette': ['#00b200', '#83ef62', '#ffff99', '#0000ff']
        }
        map.add_ee_layer(raw_image, vis_params, "PALSAR - Forest/Non-Forest")
    elif dataset_name == "GFW":
        hansen_data = ee.Image('UMD/hansen/global_forest_change_2022_v1_10')
        treecover2000 = hansen_data.select('treecover2000').clip(aoi)
        loss = hansen_data.select('lossyear').clip(aoi)
        
        # Create a loss mask for the year 2020 only
        loss_mask_2020 = loss.eq(20)
        
        # Calculate forest cover after 2020 loss by excluding areas of 2020 loss
        forest_cover_after_loss_2020 = treecover2000.updateMask(loss_mask_2020.Not())
        
        vis_params = {'min': 0, 'max': 100, 'palette': ['white', 'green']}
        map.add_ee_layer(forest_cover_after_loss_2020, vis_params, "GFW - Forest Cover After 2020 Loss")
    elif dataset_name == "ESA":
        esa_image = ee.ImageCollection('ESA/WorldCover/v100') \
            .filterDate('2020-01-01', '2020-12-31') \
            .first().select('Map').clip(aoi).reproject(crs='EPSG:4326', scale=100)
        vis_params = {
            'min': 10,
            'max': 100,
            'palette': [
                '#006400', '#ffbb22', '#ffff4c', '#f096ff', '#fa0000',
                '#b4b4b4', '#f0f0f0', '#0064c8', '#0096a0', '#00cf75', '#fae6a0'
            ]
        }
        map.add_ee_layer(esa_image, vis_params, "ESA - WorldCover")

# Visualize datasets
datasets = ["DynamicWorld", "PALSAR", "GFW", "ESA"]
for dataset in datasets:
    visualize_reclassification(dataset)

# Add layer control and save the map
folium.LayerControl().add_to(map)
#map.save("visualized_map_with_loss_2020.html")
map

                                                ####### RE-CLASSIFICATION OF IMAGES #########
                                                              ### PARAGUAY #####

import geopandas as gpd
import ee
import folium

# Authenticate and Initialize Earth Engine
ee.Authenticate()
ee.Initialize()

# Load the shapefile as AOI
shapefile_path = "C:\\Users\\49176\\Downloads\\ParaguayShape.zip"
shapefile_inside_zip = f"/vsizip/{shapefile_path}/ParaguayShape.shp"
aoi_gdf = gpd.read_file(shapefile_inside_zip)
print(f"Number of features in AOI: {len(aoi_gdf)}")
print("First feature of AOI:")
print(aoi_gdf.iloc[0])

# Convert AOI to Earth Engine FeatureCollection
aoi = geemap.gdf_to_ee(aoi_gdf)

# Dynamically compute AOI centroid for map centering
aoi_centroid = aoi.geometry().centroid().coordinates().getInfo()
center = [aoi_centroid[1], aoi_centroid[0]]  # Convert to [latitude, longitude]

# Create a folium map
map = folium.Map(location=center, zoom_start=6)

# Define a function to add Earth Engine layers to folium map
def add_ee_layer(self, ee_object, vis_params, name):
    """Function to add Earth Engine layers to a folium map."""
    if isinstance(ee_object, ee.ImageCollection):
        ee_object = ee_object.mosaic()  # Combine into a single image
    map_id_dict = ee.Image(ee_object).getMapId(vis_params)
    folium.TileLayer(
        tiles=map_id_dict['tile_fetcher'].url_format,
        attr="Map Data &copy; Google Earth Engine",
        name=name,
        overlay=True,
        control=True,
    ).add_to(self)

# Bind the method to the folium map instance
folium.Map.add_ee_layer = add_ee_layer

# Add AOI boundary to the map
map.add_ee_layer(aoi.style(color="red", width=2), {}, "AOI Boundary")

# Reclassify each dataset to forest/non-forest maps
def reclassify_to_forest(dataset_name):
    """Reclassifies dataset to binary forest/non-forest and visualizes on the map."""
    if dataset_name == "DynamicWorld":
        dynamic_world = ee.ImageCollection('GOOGLE/DYNAMICWORLD/V1') \
            .filterBounds(aoi) \
            .filterDate('2020-01-01', '2020-12-31')
        forest = dynamic_world.select('trees').mean().clip(aoi).gt(0.3)  # Threshold for forest
        vis_params = {'min': 0, 'max': 1, 'palette': ['white', 'green']}
        map.add_ee_layer(forest, vis_params, "DynamicWorld - Forest/Non-Forest")
    elif dataset_name == "PALSAR":
        palsar = ee.ImageCollection("JAXA/ALOS/PALSAR/YEARLY/FNF4") \
            .filterBounds(aoi) \
            .filterDate('2020-01-01', '2020-12-31') \
            .mosaic()
        forest = palsar.select('fnf').clip(aoi).eq(1).Or(palsar.select('fnf').eq(2))  # Class 1 and 2 are forests
        vis_params = {'min': 0, 'max': 1, 'palette': ['white', 'green']}
        map.add_ee_layer(forest, vis_params, "PALSAR - Dense and Non-Dense Forest")
    elif dataset_name == "GFW":
        hansen_data = ee.Image('UMD/hansen/global_forest_change_2022_v1_10')
        treecover2000 = hansen_data.select('treecover2000').clip(aoi)
        loss = hansen_data.select('lossyear').clip(aoi)
        
        # Create a loss mask for 2020 only
        loss_mask_2020 = loss.eq(20)
        
        # Exclude 2020 loss from forest cover
        forest_cover_after_loss = treecover2000.updateMask(loss_mask_2020.Not())
        
        # Reclassify remaining forest (>15% tree cover) and non-forest
        forest = forest_cover_after_loss.gt(10)
        
        vis_params = {'min': 0, 'max': 1, 'palette': ['white', 'green']}
        map.add_ee_layer(forest, vis_params, "GFW - Forest Cover After 2020 Loss")
    elif dataset_name == "ESA":
        esa_image = ee.ImageCollection('ESA/WorldCover/v100') \
            .filterDate('2020-01-01', '2020-12-31') \
            .first().select('Map').clip(aoi)
        forest = esa_image.eq(10)  # Class 10 represents tree cover in ESA WorldCover
        vis_params = {'min': 0, 'max': 1, 'palette': ['white', 'green']}
        map.add_ee_layer(forest, vis_params, "ESA - Forest/Non-Forest")

# Apply reclassification for each dataset
datasets = ["DynamicWorld", "PALSAR", "GFW", "ESA"]
for dataset in datasets:
    reclassify_to_forest(dataset)

# Add layer control and save the map
folium.LayerControl().add_to(map)
#map.save("reclassified_map_with_gfw_2020.html")
map

                                                    ##### RE-CLASSIFICATION OF IMAGES ####
                                                          #### ZAMBIA #######

import geopandas as gpd
import ee
import folium
import geemap

# Authenticate and Initialize Earth Engine
ee.Authenticate()
ee.Initialize()

# Load the shapefile as AOI
shapefile_path = "C:\\Users\\49176\\Downloads\\ZambiaShape.zip"
shapefile_inside_zip = f"/vsizip/{shapefile_path}/ZambiaShape.shp"
aoi_gdf = gpd.read_file(shapefile_inside_zip)
print(f"Number of features in AOI: {len(aoi_gdf)}")
print("First feature of AOI:")
print(aoi_gdf.iloc[0])

# Convert AOI to Earth Engine FeatureCollection
aoi = geemap.gdf_to_ee(aoi_gdf)

# Dynamically compute AOI centroid for map centering
aoi_centroid = aoi.geometry().centroid().coordinates().getInfo()
center = [aoi_centroid[1], aoi_centroid[0]]  # Convert to [latitude, longitude]

# Create a folium map
map = folium.Map(location=center, zoom_start=6)

# Define a function to add Earth Engine layers to folium map
def add_ee_layer(self, ee_object, vis_params, name):
    """Function to add Earth Engine layers to a folium map."""
    if isinstance(ee_object, ee.ImageCollection):
        ee_object = ee_object.mosaic()  # Combine into a single image
    map_id_dict = ee.Image(ee_object).getMapId(vis_params)
    folium.TileLayer(
        tiles=map_id_dict['tile_fetcher'].url_format,
        attr="Map Data &copy; Google Earth Engine",
        name=name,
        overlay=True,
        control=True,
    ).add_to(self)

# Bind the method to the folium map instance
folium.Map.add_ee_layer = add_ee_layer

# Add AOI boundary to the map
map.add_ee_layer(aoi.style(color="red", width=2), {}, "AOI Boundary")

# Reclassify each dataset to forest/non-forest maps
def reclassify_to_forest(dataset_name):
    """Reclassifies dataset to binary forest/non-forest and visualizes on the map."""
    if dataset_name == "DynamicWorld":
        dynamic_world = ee.ImageCollection('GOOGLE/DYNAMICWORLD/V1') \
            .filterBounds(aoi) \
            .filterDate('2020-01-01', '2020-12-31')
        forest = dynamic_world.select('trees').mean().clip(aoi).gt(0.3)  # Threshold for forest
        vis_params = {'min': 0, 'max': 1, 'palette': ['white', 'green']}
        map.add_ee_layer(forest, vis_params, "DynamicWorld - Forest/Non-Forest")
    
    elif dataset_name == "PALSAR":
        palsar = ee.ImageCollection("JAXA/ALOS/PALSAR/YEARLY/FNF4") \
            .filterBounds(aoi) \
            .filterDate('2020-01-01', '2020-12-31') \
            .mosaic()
        forest = palsar.select('fnf').clip(aoi).eq(1).Or(palsar.select('fnf').eq(2))  # Class 1 and 2 are forests
        vis_params = {'min': 0, 'max': 1, 'palette': ['white', 'green']}
        map.add_ee_layer(forest, vis_params, "PALSAR - Dense and Non-Dense Forest")
    
    elif dataset_name == "GFW":
        hansen_data = ee.Image('UMD/hansen/global_forest_change_2022_v1_10')
        treecover2000 = hansen_data.select('treecover2000').clip(aoi)
        loss = hansen_data.select('lossyear').clip(aoi)

        # Mask areas that experienced loss in 2020
        loss_mask_2020 = loss.eq(20)
        forest_cover_after_loss = treecover2000.updateMask(loss_mask_2020.Not())

        # Reclassify values: below 10 -> 0 (non-forest), above or equal to 10 -> 1 (forest)
        forest = forest_cover_after_loss.gte(10)  # Greater than or equal to 10 becomes 1 (forest)

        # Ensure non-forest areas are explicitly 0
        forest_binary = forest.updateMask(forest).unmask(0)

        vis_params = {'min': 0, 'max': 1, 'palette': ['white', 'green']}
        map.add_ee_layer(forest_binary, vis_params, "GFW - Reclassified Forest/Non-Forest")
    
    elif dataset_name == "ESA":
        esa_image = ee.ImageCollection('ESA/WorldCover/v100') \
            .filterDate('2020-01-01', '2020-12-31') \
            .first().select('Map').clip(aoi)
        forest = esa_image.eq(10)  # Class 10 represents tree cover in ESA WorldCover
        vis_params = {'min': 0, 'max': 1, 'palette': ['white', 'green']}
        map.add_ee_layer(forest, vis_params, "ESA - Forest/Non-Forest")

# Apply reclassification for each dataset
datasets = ["DynamicWorld", "PALSAR", "GFW", "ESA"]
for dataset in datasets:
    reclassify_to_forest(dataset)

# Add layer control and save the map
folium.LayerControl().add_to(map)
#map.save("reclassified_map_with_gfw_2020.html")
map         

                                            ####  RE-CLASSIFICATION OF IMAGES ######
                                                    #### ZIMBABWE #####
                                                           

import geopandas as gpd
import ee
import folium
import geemap

# Authenticate and Initialize Earth Engine
ee.Authenticate()
ee.Initialize()

# Load the shapefile as AOI
shapefile_path = "C:\\Users\\49176\\Downloads\\ZimbabweShape.zip"
shapefile_inside_zip = f"/vsizip/{shapefile_path}/ZimbabweShape.shp"
aoi_gdf = gpd.read_file(shapefile_inside_zip)
print(f"Number of features in AOI: {len(aoi_gdf)}")
print("First feature of AOI:")
print(aoi_gdf.iloc[0])

# Convert AOI to Earth Engine FeatureCollection
aoi = geemap.gdf_to_ee(aoi_gdf)

# Dynamically compute AOI centroid for map centering
aoi_centroid = aoi.geometry().centroid().coordinates().getInfo()
center = [aoi_centroid[1], aoi_centroid[0]]  # Convert to [latitude, longitude]

# Create a folium map
map = folium.Map(location=center, zoom_start=6)

# Define a function to add Earth Engine layers to folium map
def add_ee_layer(self, ee_object, vis_params, name):
    """Function to add Earth Engine layers to a folium map."""
    if isinstance(ee_object, ee.ImageCollection):
        ee_object = ee_object.mosaic()  # Combine into a single image
    map_id_dict = ee.Image(ee_object).getMapId(vis_params)
    folium.TileLayer(
        tiles=map_id_dict['tile_fetcher'].url_format,
        attr="Map Data &copy; Google Earth Engine",
        name=name,
        overlay=True,
        control=True,
    ).add_to(self)

# Bind the method to the folium map instance
folium.Map.add_ee_layer = add_ee_layer

# Add AOI boundary to the map
map.add_ee_layer(aoi.style(color="red", width=2), {}, "AOI Boundary")

# Reclassify each dataset to forest/non-forest maps
def reclassify_to_forest(dataset_name):
    """Reclassifies dataset to binary forest/non-forest and visualizes on the map."""
    if dataset_name == "DynamicWorld":
        dynamic_world = ee.ImageCollection('GOOGLE/DYNAMICWORLD/V1') \
            .filterBounds(aoi) \
            .filterDate('2020-01-01', '2020-12-31')
        forest = dynamic_world.select('trees').mean().clip(aoi).gt(0.3)  # Threshold for forest
        vis_params = {'min': 0, 'max': 1, 'palette': ['white', 'green']}
        map.add_ee_layer(forest, vis_params, "DynamicWorld - Forest/Non-Forest")
    
    elif dataset_name == "PALSAR":
        palsar = ee.ImageCollection("JAXA/ALOS/PALSAR/YEARLY/FNF4") \
            .filterBounds(aoi) \
            .filterDate('2020-01-01', '2020-12-31') \
            .mosaic()
        forest = palsar.select('fnf').clip(aoi).eq(1).Or(palsar.select('fnf').eq(2))  # Class 1 and 2 are forests
        vis_params = {'min': 0, 'max': 1, 'palette': ['white', 'green']}
        map.add_ee_layer(forest, vis_params, "PALSAR - Dense and Non-Dense Forest")
    
    elif dataset_name == "GFW":
        hansen_data = ee.Image('UMD/hansen/global_forest_change_2022_v1_10')
        treecover2000 = hansen_data.select('treecover2000').clip(aoi)
        loss = hansen_data.select('lossyear').clip(aoi)

        # Mask areas that experienced loss in 2020
        loss_mask_2020 = loss.eq(20)
        forest_cover_after_loss = treecover2000.updateMask(loss_mask_2020.Not())

        # Reclassify values: below 10 -> 0 (non-forest), above or equal to 10 -> 1 (forest)
        forest = forest_cover_after_loss.gte(10)  # Greater than or equal to 10 becomes 1 (forest)

        # Ensure non-forest areas are explicitly 0
        forest_binary = forest.updateMask(forest).unmask(0)

        vis_params = {'min': 0, 'max': 1, 'palette': ['white', 'green']}
        map.add_ee_layer(forest_binary, vis_params, "GFW - Reclassified Forest/Non-Forest")
    
    elif dataset_name == "ESA":
        esa_image = ee.ImageCollection('ESA/WorldCover/v100') \
            .filterDate('2020-01-01', '2020-12-31') \
            .first().select('Map').clip(aoi)
        forest = esa_image.eq(10)  # Class 10 represents tree cover in ESA WorldCover
        vis_params = {'min': 0, 'max': 1, 'palette': ['white', 'green']}
        map.add_ee_layer(forest, vis_params, "ESA - Forest/Non-Forest")

# Apply reclassification for each dataset
datasets = ["DynamicWorld", "PALSAR", "GFW", "ESA"]
for dataset in datasets:
    reclassify_to_forest(dataset)

# Add layer control and save the map
folium.LayerControl().add_to(map)
#map.save("reclassified_map_with_gfw_2020.html")
map

                                                #### CALCULATING FOREST COVER IN HECTARES ####
                                                          ### PARAGUAY###

import geopandas as gpd
import ee
import folium
import geemap

# Authenticate and Initialize Earth Engine
ee.Authenticate()
ee.Initialize()

# Load the shapefile as AOI
shapefile_path = "C:\\Users\\49176\\Downloads\\ParaguayShape.zip"
shapefile_inside_zip = f"/vsizip/{shapefile_path}/ParaguayShape.shp"
aoi_gdf = gpd.read_file(shapefile_inside_zip)
print(f"Number of features in AOI: {len(aoi_gdf)}")
print("First feature of AOI:")
print(aoi_gdf.iloc[0])

# Convert AOI to Earth Engine FeatureCollection
aoi = geemap.gdf_to_ee(aoi_gdf)

# Dynamically compute AOI centroid for map centering
aoi_centroid = aoi.geometry().centroid().coordinates().getInfo()
center = [aoi_centroid[1], aoi_centroid[0]]  # Convert to [latitude, longitude]

# Create a folium map
map = folium.Map(location=center, zoom_start=6)

# Define a function to add Earth Engine layers to folium map
def add_ee_layer(self, ee_object, vis_params, name):
    """Function to add Earth Engine layers to a folium map."""
    if isinstance(ee_object, ee.ImageCollection):
        ee_object = ee_object.mosaic()  # Combine into a single image
    map_id_dict = ee.Image(ee_object).getMapId(vis_params)
    folium.TileLayer(
        tiles=map_id_dict['tile_fetcher'].url_format,
        attr="Map Data &copy; Google Earth Engine",
        name=name,
        overlay=True,
        control=True,
    ).add_to(self)

# Bind the method to the folium map instance
folium.Map.add_ee_layer = add_ee_layer

# Add AOI boundary to the map
map.add_ee_layer(aoi.style(color="red", width=2), {}, "AOI Boundary")

# Function to reclassify datasets and calculate forest area
def reclassify_and_calculate_area(dataset_name, scale=500):
    """Reclassifies dataset to binary forest/non-forest, calculates total forest area, and visualizes on the map."""
    forest = None
    if dataset_name == "DynamicWorld":
        dynamic_world = ee.ImageCollection('GOOGLE/DYNAMICWORLD/V1') \
            .filterBounds(aoi) \
            .filterDate('2020-01-01', '2020-12-31')
        forest = dynamic_world.select('trees').mean().gt(0.4)  # Threshold for forest
        map.add_ee_layer(forest, {'min': 0, 'max': 1, 'palette': ['white', 'green']}, "DynamicWorld Forest")
        # Calculate forest area
        forest_area = ee.Image.pixelArea().updateMask(forest).reduceRegion(
            reducer=ee.Reducer.sum(),
            geometry=aoi.geometry(),
            scale=scale,
            maxPixels=1e13,
            bestEffort=True
        ).getInfo()
        forest_area_ha = forest_area.get('area', 0) / 10000
        print(f"DynamicWorld Total Forest Area (ha): {forest_area_ha}")

    elif dataset_name == "PALSAR":
        palsar = ee.ImageCollection("JAXA/ALOS/PALSAR/YEARLY/FNF4") \
            .filterBounds(aoi) \
            .filterDate('2020-01-01', '2020-12-31') \
            .mosaic()
        forest = palsar.select('fnf').eq(1).Or(palsar.select('fnf').eq(2))  # Class 1 and 2 are forests
        map.add_ee_layer(forest, {'min': 0, 'max': 1, 'palette': ['white', 'green']}, "PALSAR Forest")
        # Calculate forest area
        forest_area = ee.Image.pixelArea().updateMask(forest).reduceRegion(
            reducer=ee.Reducer.sum(),
            geometry=aoi.geometry(),
            scale=scale,
            maxPixels=1e13,
            bestEffort=True
        ).getInfo()
        forest_area_ha = forest_area.get('area', 0) / 10000
        print(f"PALSAR Total Forest Area (ha): {forest_area_ha}")

    elif dataset_name == "GFW":
        hansen_data = ee.Image('UMD/hansen/global_forest_change_2022_v1_10')
        treecover2000 = hansen_data.select('treecover2000').clip(aoi)
        loss = hansen_data.select('lossyear').clip(aoi)
        loss_mask_2020 = loss.eq(20)
        forest_cover_after_loss = treecover2000.updateMask(loss_mask_2020.Not())
        forest = forest_cover_after_loss.gt(2)
        map.add_ee_layer(forest, {'min': 0, 'max': 1, 'palette': ['white', 'green']}, "GFW Forest")
        # Calculate forest area
        forest_area = ee.Image.pixelArea().updateMask(forest).reduceRegion(
            reducer=ee.Reducer.sum(),
            geometry=aoi.geometry(),
            scale=scale,
            maxPixels=1e13,
            bestEffort=True
        ).getInfo()
        forest_area_ha = forest_area.get('area', 0) / 10000
        print(f"GFW Total Forest Area (ha): {forest_area_ha}")

    elif dataset_name == "ESA":
        esa_image = ee.ImageCollection('ESA/WorldCover/v100') \
            .filterDate('2020-01-01', '2020-12-31') \
            .first().select('Map')
        forest = esa_image.eq(10)  # Class 10 represents tree cover in ESA WorldCover
        map.add_ee_layer(forest, {'min': 0, 'max': 1, 'palette': ['white', 'green']}, "ESA Forest")
        # Calculate forest area
        forest_area = ee.Image.pixelArea().updateMask(forest).reduceRegion(
            reducer=ee.Reducer.sum(),
            geometry=aoi.geometry(),
            scale=scale,
            maxPixels=1e13,
            bestEffort=True
        ).getInfo()
        forest_area_ha = forest_area.get('area', 0) / 10000
        print(f"ESA Total Forest Area (ha): {forest_area_ha}")

# Apply reclassification and area calculation for each dataset
datasets = ["DynamicWorld", "PALSAR", "GFW", "ESA"]
for dataset in datasets:
    reclassify_and_calculate_area(dataset)

# Add layer control and save the map
folium.LayerControl().add_to(map)
map.save("fixed_reclassified_map.html")
print("Fixed map saved as fixed_reclassified_map.html")

                                                #### CALCULATING FOREST COVER IN HECTARES ####
                                                         ##### ZAMBIA #####
import ee
import folium
import geemap

# Authenticate and Initialize Earth Engine
ee.Authenticate()
ee.Initialize()

# Load the shapefile as AOI
shapefile_path = "C:\\Users\\49176\\Downloads\\ZambiaShape.zip"
shapefile_inside_zip = f"/vsizip/{shapefile_path}/ZambiaShape.shp"
aoi_gdf = gpd.read_file(shapefile_inside_zip)
print(f"Number of features in AOI: {len(aoi_gdf)}")
print("First feature of AOI:")
print(aoi_gdf.iloc[0])

# Convert AOI to Earth Engine FeatureCollection
aoi = geemap.gdf_to_ee(aoi_gdf)

# Dynamically compute AOI centroid for map centering
aoi_centroid = aoi.geometry().centroid().coordinates().getInfo()
center = [aoi_centroid[1], aoi_centroid[0]]  # Convert to [latitude, longitude]

# Create a folium map
map = folium.Map(location=center, zoom_start=6)

# Define a function to add Earth Engine layers to folium map
def add_ee_layer(self, ee_object, vis_params, name):
    """Function to add Earth Engine layers to a folium map."""
    if isinstance(ee_object, ee.ImageCollection):
        ee_object = ee_object.mosaic()  # Combine into a single image
    map_id_dict = ee.Image(ee_object).getMapId(vis_params)
    folium.TileLayer(
        tiles=map_id_dict['tile_fetcher'].url_format,
        attr="Map Data &copy; Google Earth Engine",
        name=name,
        overlay=True,
        control=True,
    ).add_to(self)

# Bind the method to the folium map instance
folium.Map.add_ee_layer = add_ee_layer

# Add AOI boundary to the map
map.add_ee_layer(aoi.style(color="red", width=2), {}, "AOI Boundary")

# Function to reclassify datasets and calculate forest area
def reclassify_and_calculate_area(dataset_name, scale=500):
    """Reclassifies dataset to binary forest/non-forest, calculates total forest area, and visualizes on the map."""
    forest = None
    if dataset_name == "DynamicWorld":
        dynamic_world = ee.ImageCollection('GOOGLE/DYNAMICWORLD/V1') \
            .filterBounds(aoi) \
            .filterDate('2020-01-01', '2020-12-31')
        forest = dynamic_world.select('trees').mean().gt(0.4)  # Threshold for forest
        map.add_ee_layer(forest, {'min': 0, 'max': 1, 'palette': ['white', 'green']}, "DynamicWorld Forest")
        # Calculate forest area
        forest_area = ee.Image.pixelArea().updateMask(forest).reduceRegion(
            reducer=ee.Reducer.sum(),
            geometry=aoi.geometry(),
            scale=scale,
            maxPixels=1e13,
            bestEffort=True
        ).getInfo()
        forest_area_ha = forest_area.get('area', 0) / 10000
        print(f"DynamicWorld Total Forest Area (ha): {forest_area_ha}")

    elif dataset_name == "PALSAR":
        palsar = ee.ImageCollection("JAXA/ALOS/PALSAR/YEARLY/FNF4") \
            .filterBounds(aoi) \
            .filterDate('2020-01-01', '2020-12-31') \
            .mosaic()
        forest = palsar.select('fnf').eq(1).Or(palsar.select('fnf').eq(2))  # Class 1 and 2 are forests
        map.add_ee_layer(forest, {'min': 0, 'max': 1, 'palette': ['white', 'green']}, "PALSAR Forest")
        # Calculate forest area
        forest_area = ee.Image.pixelArea().updateMask(forest).reduceRegion(
            reducer=ee.Reducer.sum(),
            geometry=aoi.geometry(),
            scale=scale,
            maxPixels=1e13,
            bestEffort=True
        ).getInfo()
        forest_area_ha = forest_area.get('area', 0) / 10000
        print(f"PALSAR Total Forest Area (ha): {forest_area_ha}")

    elif dataset_name == "GFW":
        hansen_data = ee.Image('UMD/hansen/global_forest_change_2022_v1_10')
        treecover2000 = hansen_data.select('treecover2000').clip(aoi)
        loss = hansen_data.select('lossyear').clip(aoi)
        loss_mask_2020 = loss.eq(20)
        forest_cover_after_loss = treecover2000.updateMask(loss_mask_2020.Not())
        forest = forest_cover_after_loss.gt(2)
        map.add_ee_layer(forest, {'min': 0, 'max': 1, 'palette': ['white', 'green']}, "GFW Forest")
        # Calculate forest area
        forest_area = ee.Image.pixelArea().updateMask(forest).reduceRegion(
            reducer=ee.Reducer.sum(),
            geometry=aoi.geometry(),
            scale=scale,
            maxPixels=1e13,
            bestEffort=True
        ).getInfo()
        forest_area_ha = forest_area.get('area', 0) / 10000
        print(f"GFW Total Forest Area (ha): {forest_area_ha}")

    elif dataset_name == "ESA":
        esa_image = ee.ImageCollection('ESA/WorldCover/v100') \
            .filterDate('2020-01-01', '2020-12-31') \
            .first().select('Map')
        forest = esa_image.eq(10)  # Class 10 represents tree cover in ESA WorldCover
        map.add_ee_layer(forest, {'min': 0, 'max': 1, 'palette': ['white', 'green']}, "ESA Forest")
        # Calculate forest area
        forest_area = ee.Image.pixelArea().updateMask(forest).reduceRegion(
            reducer=ee.Reducer.sum(),
            geometry=aoi.geometry(),
            scale=scale,
            maxPixels=1e13,
            bestEffort=True
        ).getInfo()
        forest_area_ha = forest_area.get('area', 0) / 10000
        print(f"ESA Total Forest Area (ha): {forest_area_ha}")

# Apply reclassification and area calculation for each dataset
datasets = ["DynamicWorld", "PALSAR", "GFW", "ESA"]
for dataset in datasets:
    reclassify_and_calculate_area(dataset)

# Add layer control and save the map
folium.LayerControl().add_to(map)
map.save("fixed_reclassified_map.html")
print("Fixed map saved as fixed_reclassified_map.html") 

                                           ##### CALCULATING FOREST COVER IN HECTARES ####
                                                   ##### ZIMBABWE #####
import ee
import folium
import geemap

# Authenticate and Initialize Earth Engine
ee.Authenticate()
ee.Initialize()

# Load the shapefile as AOI
shapefile_path = "C:\\Users\\49176\\Downloads\\ZimbabweShape.zip"
shapefile_inside_zip = f"/vsizip/{shapefile_path}/ZimbabweShape.shp"
aoi_gdf = gpd.read_file(shapefile_inside_zip)
print(f"Number of features in AOI: {len(aoi_gdf)}")
print("First feature of AOI:")
print(aoi_gdf.iloc[0])

# Convert AOI to Earth Engine FeatureCollection
aoi = geemap.gdf_to_ee(aoi_gdf)

# Dynamically compute AOI centroid for map centering
aoi_centroid = aoi.geometry().centroid().coordinates().getInfo()
center = [aoi_centroid[1], aoi_centroid[0]]  # Convert to [latitude, longitude]

# Create a folium map
map = folium.Map(location=center, zoom_start=6)

# Define a function to add Earth Engine layers to folium map
def add_ee_layer(self, ee_object, vis_params, name):
    """Function to add Earth Engine layers to a folium map."""
    if isinstance(ee_object, ee.ImageCollection):
        ee_object = ee_object.mosaic()  # Combine into a single image
    map_id_dict = ee.Image(ee_object).getMapId(vis_params)
    folium.TileLayer(
        tiles=map_id_dict['tile_fetcher'].url_format,
        attr="Map Data &copy; Google Earth Engine",
        name=name,
        overlay=True,
        control=True,
    ).add_to(self)

# Bind the method to the folium map instance
folium.Map.add_ee_layer = add_ee_layer

# Add AOI boundary to the map
map.add_ee_layer(aoi.style(color="red", width=2), {}, "AOI Boundary")

# Function to reclassify datasets and calculate forest area
def reclassify_and_calculate_area(dataset_name, scale=500):
    """Reclassifies dataset to binary forest/non-forest, calculates total forest area, and visualizes on the map."""
    forest = None
    if dataset_name == "DynamicWorld":
        dynamic_world = ee.ImageCollection('GOOGLE/DYNAMICWORLD/V1') \
            .filterBounds(aoi) \
            .filterDate('2020-01-01', '2020-12-31')
        forest = dynamic_world.select('trees').mean().gt(0.4)  # Threshold for forest
        map.add_ee_layer(forest, {'min': 0, 'max': 1, 'palette': ['white', 'green']}, "DynamicWorld Forest")
        # Calculate forest area
        forest_area = ee.Image.pixelArea().updateMask(forest).reduceRegion(
            reducer=ee.Reducer.sum(),
            geometry=aoi.geometry(),
            scale=scale,
            maxPixels=1e13,
            bestEffort=True
        ).getInfo()
        forest_area_ha = forest_area.get('area', 0) / 10000
        print(f"DynamicWorld Total Forest Area (ha): {forest_area_ha}")

    elif dataset_name == "PALSAR":
        palsar = ee.ImageCollection("JAXA/ALOS/PALSAR/YEARLY/FNF4") \
            .filterBounds(aoi) \
            .filterDate('2020-01-01', '2020-12-31') \
            .mosaic()
        forest = palsar.select('fnf').eq(1).Or(palsar.select('fnf').eq(2))  # Class 1 and 2 are forests
        map.add_ee_layer(forest, {'min': 0, 'max': 1, 'palette': ['white', 'green']}, "PALSAR Forest")
        # Calculate forest area
        forest_area = ee.Image.pixelArea().updateMask(forest).reduceRegion(
            reducer=ee.Reducer.sum(),
            geometry=aoi.geometry(),
            scale=scale,
            maxPixels=1e13,
            bestEffort=True
        ).getInfo()
        forest_area_ha = forest_area.get('area', 0) / 10000
        print(f"PALSAR Total Forest Area (ha): {forest_area_ha}")

    elif dataset_name == "GFW":
        hansen_data = ee.Image('UMD/hansen/global_forest_change_2022_v1_10')
        treecover2000 = hansen_data.select('treecover2000').clip(aoi)
        loss = hansen_data.select('lossyear').clip(aoi)
        loss_mask_2020 = loss.eq(20)
        forest_cover_after_loss = treecover2000.updateMask(loss_mask_2020.Not())
        forest = forest_cover_after_loss.gt(2)
        map.add_ee_layer(forest, {'min': 0, 'max': 1, 'palette': ['white', 'green']}, "GFW Forest")
        # Calculate forest area
        forest_area = ee.Image.pixelArea().updateMask(forest).reduceRegion(
            reducer=ee.Reducer.sum(),
            geometry=aoi.geometry(),
            scale=scale,
            maxPixels=1e13,
            bestEffort=True
        ).getInfo()
        forest_area_ha = forest_area.get('area', 0) / 10000
        print(f"GFW Total Forest Area (ha): {forest_area_ha}")

    elif dataset_name == "ESA":
        esa_image = ee.ImageCollection('ESA/WorldCover/v100') \
            .filterDate('2020-01-01', '2020-12-31') \
            .first().select('Map')
        forest = esa_image.eq(10)  # Class 10 represents tree cover in ESA WorldCover
        map.add_ee_layer(forest, {'min': 0, 'max': 1, 'palette': ['white', 'green']}, "ESA Forest")
        # Calculate forest area
        forest_area = ee.Image.pixelArea().updateMask(forest).reduceRegion(
            reducer=ee.Reducer.sum(),
            geometry=aoi.geometry(),
            scale=scale,
            maxPixels=1e13,
            bestEffort=True
        ).getInfo()
        forest_area_ha = forest_area.get('area', 0) / 10000
        print(f"ESA Total Forest Area (ha): {forest_area_ha}")

# Apply reclassification and area calculation for each dataset
datasets = ["DynamicWorld", "PALSAR", "GFW", "ESA"]
for dataset in datasets:
    reclassify_and_calculate_area(dataset)

# Add layer control and save the map
folium.LayerControl().add_to(map)
map.save("fixed_reclassified_map.html")
print("Fixed map saved as fixed_reclassified_map.html") 

                                              ###### CHECKING THE NICFI POINTS ####
                                                      #### PARAGUAY #####
import geopandas as gpd

# Define the path to the ZIP file and the shapefile inside it
shapefile_path = "C:\\Users\\49176\\Downloads\\ParaguayNICFIPoints.zip"
shapefile_inside_zip = "/vsizip/{}/ParaguayNICFIPoints.shp".format(shapefile_path)

# Load the shapefile
actual_points_NICFI = gpd.read_file(shapefile_inside_zip)

# Perform checks
print(f"Number of features: {len(actual_points_NICFI)}")
print("First feature:")
print(actual_points_NICFI.iloc[0])

# Count occurrences of each class
class_counts = actual_points_NICFI['id'].value_counts()

# Display class distribution with mapping
print("\nClass Mapping: 0 = Non-Forest, 1 = Forest")
print("\nClass Distribution in the Validation Dataset:")
for cls, count in class_counts.items():
    label = "Non-Forest" if cls == 0 else "Forest"
    print(f"Class {cls} ({label}): {count} points")

                                               ###### CHECKING THE NICFI POINTS ####
                                                    #### ZAMBIA #####
import geopandas as gpd

# Define the path to the ZIP file and the shapefile inside it
shapefile_path = "C:\\Users\\49176\\Downloads\\Zambia_NICFI_Points.zip"
shapefile_inside_zip = "/vsizip/{}/Points_NICFI_Zambia.shp".format(shapefile_path)

# Load the shapefile
actual_points_NICFI = gpd.read_file(shapefile_inside_zip)

# Perform checks
print(f"Number of features: {len(actual_points_NICFI)}")
print("First feature:")
print(actual_points_NICFI.iloc[0])

# Count occurrences of each class
class_counts = actual_points_NICFI['id'].value_counts()

# Display class distribution with mapping
print("\nClass Mapping: 0 = Non-Forest, 1 = Forest")
print("\nClass Distribution in the Validation Dataset:")
for cls, count in class_counts.items():
    label = "Non-Forest" if cls == 0 else "Forest"
    print(f"Class {cls} ({label}): {count} points")

                                               ###### CHECKING THE NICFI POINTS ####
                                                    #### ZIMBABWE #####
import geopandas as gpd

# Define the path to the ZIP file and the shapefile inside it
shapefile_path = "C:\\Users\\49176\\Downloads\\ZimNICFIPoints.zip"
shapefile_inside_zip = "/vsizip/{}/ZimNICFIPoints.shp".format(shapefile_path)

# Load the shapefile
actual_points_NICFI = gpd.read_file(shapefile_inside_zip)

# Perform checks
print(f"Number of features: {len(actual_points_NICFI)}")
print("First feature:")
print(actual_points_NICFI.iloc[0])

# Count occurrences of each class
class_counts = actual_points_NICFI['id'].value_counts()

# Display class distribution with mapping
print("\nClass Mapping: 0 = Non-Forest, 1 = Forest")
print("\nClass Distribution in the Validation Dataset:")
for cls, count in class_counts.items():
    label = "Non-Forest" if cls == 0 else "Forest"
    print(f"Class {cls} ({label}): {count} points")

                                                 #### ACCURACY ASSESSMENT FOR ALL THREE COUNTRIES ###
import ee
import geopandas as gpd
import geemap
import pandas as pd

# Authenticate and Initialize Earth Engine
ee.Authenticate()
ee.Initialize()

# Function to load data for a given country
def load_country_data(country_name):
    if country_name == "Paraguay":
        aoi_path = "C:\\Users\\49176\\Downloads\\ParaguayShape.zip"
        points_path = "C:\\Users\\49176\\Downloads\\ParaguayNICFIPoints.zip"
        aoi_file = "ParaguayShape.shp"
        points_file = "ParaguayNICFIPoints.shp"
    elif country_name == "Zambia":
        aoi_path = "C:\\Users\\49176\\Downloads\\ZambiaShape.zip"
        points_path = "C:\\Users\\49176\\Downloads\\Zambia_NICFI_Points.zip"
        aoi_file = "ZambiaShape.shp"
        points_file = "Points_NICFI_Zambia.shp"
    elif country_name == "Zimbabwe":
        aoi_path = "C:\\Users\\49176\\Downloads\\ZimbabweShape.zip"
        points_path = "C:\\Users\\49176\\Downloads\\ZimNICFIPoints.zip"
        aoi_file = "ZimbabweShape.shp"
        points_file = "ZimNICFIPoints.shp"
    else:
        raise ValueError("Unsupported country name.")

    aoi_gdf = gpd.read_file(f"/vsizip/{aoi_path}/{aoi_file}")
    points_gdf = gpd.read_file(f"/vsizip/{points_path}/{points_file}")
    return geemap.gdf_to_ee(aoi_gdf), geemap.gdf_to_ee(points_gdf)

# Function to process each dataset
def process_dataset(dataset_name, aoi_fc, actual_points_fc, country_name, results):
    classification_image = None
    resolution = None

    if dataset_name == "DynamicWorld":
        image = ee.ImageCollection('GOOGLE/DYNAMICWORLD/V1') \
            .filterBounds(aoi_fc) \
            .filterDate('2020-01-01', '2020-12-31') \
            .select(['trees']).mean()
        classification_image = image.gte(0.5).rename('classification')
        resolution = 10
    elif dataset_name == "PALSAR":
        palsar = ee.ImageCollection("JAXA/ALOS/PALSAR/YEARLY/FNF4") \
            .filterBounds(aoi_fc) \
            .filterDate('2020-01-01', '2020-12-31') \
            .mosaic()
        classification_image = palsar.expression(
            'fnf == 1 || fnf == 2 ? 1 : 0',
            {'fnf': palsar.select('fnf')}
        ).rename('classification')
        resolution = 25
    elif dataset_name == "GFW":
        image = ee.Image('UMD/hansen/global_forest_change_2022_v1_10').select('treecover2000')
        classification_image = image.gte(10).rename('classification')
        resolution = 30
    elif dataset_name == "ESA":
        image = ee.ImageCollection('ESA/WorldCover/v100') \
            .filterDate('2020-01-01', '2020-12-31') \
            .first()
        classification_image = image.expression(
            'b("Map") == 10 ? 1 : 0',
            {'Map': image.select('Map')}
        ).rename('classification')
        resolution = 10

    samples = classification_image.sampleRegions(
        collection=actual_points_fc,
        properties=['id'],
        scale=resolution
    ).map(lambda f: f.set('classification', ee.Number(f.get('classification')).toInt()))

    cm = samples.errorMatrix('id', 'classification')
    matrix = cm.getInfo()

    def safe_val(r, c):
        try:
            return matrix[r][c]
        except:
            return 0

    TN = safe_val(0, 0)
    FP = safe_val(0, 1)
    FN = safe_val(1, 0)
    TP = safe_val(1, 1)

    precision = TP / (TP + FP) if (TP + FP) > 0 else 0
    recall = TP / (TP + FN) if (TP + FN) > 0 else 0
    f1 = (2 * precision * recall) / (precision + recall) if (precision + recall) > 0 else 0
    accuracy = cm.accuracy().getInfo()
    kappa = cm.kappa().getInfo()

    squared_diff = samples.map(lambda f: f.set(
        'squared_diff', ee.Number(f.get('id')).subtract(f.get('classification')).pow(2)
    ))
    mean_squared = squared_diff.reduceColumns(
        reducer=ee.Reducer.mean(),
        selectors=['squared_diff']
    ).get('mean')
    rmse = ee.Number(mean_squared).sqrt().getInfo()

    tpr = TP / (TP + FN) if (TP + FN) > 0 else 0
    fpr = FP / (FP + TN) if (FP + TN) > 0 else 0
    auc = tpr / (tpr + fpr) if (tpr + fpr) > 0 else 0

    results.append({
        "Country": country_name,
        "Dataset": dataset_name,
        "TP": TP,
        "TN": TN,
        "FP": FP,
        "FN": FN,
        "Precision": precision,
        "Recall": recall,
        "F1 Score": f1,
        "Accuracy": accuracy,
        "Kappa": kappa,
        "RMSE": rmse,
        "AUC": auc
    })

def run_all_countries():
    results = []
    countries = ["Paraguay", "Zambia", "Zimbabwe"]
    datasets = ["DynamicWorld", "PALSAR", "GFW", "ESA"]

    for country in countries:
        aoi_fc, points_fc = load_country_data(country)
        for dataset in datasets:
            process_dataset(dataset, aoi_fc, points_fc, country, results)

    df = pd.DataFrame(results)
    df = df.sort_values(by=["Country", "Dataset"])

    print("\nCombined Classification Results:\n")
    for country in df["Country"].unique():
        print("=" * 80)
        print(f"{country}".center(80))
        print("=" * 80)
        print(df[df["Country"] == country].drop(columns="Country").to_string(index=False))
        print("\n")

    df.to_csv("combined_classification_results.csv", index=False)
    print("Results saved to 'combined_classification_results.csv'")

# Execute
run_all_countries()

        ### TO SHOW THE ACCURACY RANKINGS; MAKE SURE TO COPY ACQUIRED ACCURACY ASSESSMENT RESULTS ###
     ##IN THE CASE OF THIS STUDY AREAS;THESE WERE THE RESULTS WHICH WERE COPIED AND PRODUCED HEATMAPS##

    import pandas as pd
import seaborn as sns
import matplotlib.pyplot as plt

# Real metrics from the classification results
data = {
    "Country": ["Paraguay"]*4 + ["Zambia"]*4 + ["Zimbabwe"]*4,
    "Dataset": ["Dynamic World", "PALSAR", "GFW", "ESA WorldCover"]*3,
    "F1 Score": [
        0.777049, 0.759740, 0.783862, 0.777600,  # Paraguay
        0.784091, 0.853470, 0.867102, 0.804408,  # Zambia
        0.476744, 0.899384, 0.904000, 0.736342   # Zimbabwe
    ],
    "Kappa": [
        0.456000, 0.401682, 0.295907, 0.428924,  # Paraguay
        0.673427, 0.761146, 0.755844, 0.697202,  # Zambia
        0.302499, 0.804936, 0.808441, 0.563624   # Zimbabwe
    ],
    "Overall Accuracy": [
        0.728, 0.704, 0.700, 0.722,  # Paraguay
        0.848, 0.886, 0.878, 0.858,  # Zambia
        0.640, 0.902, 0.904, 0.778   # Zimbabwe
    ]
}

# Create DataFrame
df = pd.DataFrame(data)

# Create a plot for each country
fig, axes = plt.subplots(3, 1, figsize=(8, 16))
countries = ["Paraguay", "Zambia", "Zimbabwe"]
titles = ["(a) Heatmap of Metrics: Paraguay", "(b) Heatmap of Metrics: Zambia", "(c) Heatmap of Metrics: Zimbabwe"]

for i, country in enumerate(countries):
    ax = axes[i]
    country_df = df[df["Country"] == country].set_index("Dataset")[["F1 Score", "Kappa", "Overall Accuracy"]]
    
    sns.heatmap(
        country_df,
        annot=True,
        fmt=".2f",
        cmap="Blues",
        linewidths=0.5,
        cbar=True,
        ax=ax
    )
    ax.set_title(titles[i], fontsize=14)
    ax.set_xlabel("")
    ax.set_ylabel("Dataset")

plt.tight_layout()
plt.show()


                    ### TO SHOW THE RANKING OF THE RMSE AND AUC;COPY RMSE AND AUC RESULTS TO SUIT THE FOLLOWING CODE ###
                      ### PLEASE NOTE THAT LOWER RMSE INDICATES HIGHER ACCURACY AND HIGHER AUC VALUES INDICATES HIGHER ACCURACY ###
import matplotlib.pyplot as plt
import numpy as np

# Dataset labels and countries
datasets = ['GFW', 'Dynamic World', 'ESA WorldCover', 'Global-4 class PALSAR-2']
countries = ['Paraguay', 'Zambia', 'Zimbabwe']

# Updated RMSE values
rmse_data = {
    'GFW': [0.547723, 0.349285, 0.309839],
    'Dynamic World': [0.521536, 0.389872, 0.600000],
    'ESA WorldCover': [0.527257, 0.376829, 0.471169],
    'Global-4 class PALSAR-2': [0.544059, 0.337639, 0.313050]
}

# Updated AUC values
auc_data = {
    'GFW': [0.630464, 0.846064, 0.944776],
    'Dynamic World': [0.876387, 0.979442, 1.000000],
    'ESA WorldCover': [0.811159, 0.966446, 0.972376],
    'Global-4 class PALSAR-2': [0.805310, 0.946334, 0.970723]
}

# Set up the figure and axes
fig, axs = plt.subplots(1, 2, figsize=(14, 6))

# Colors
colors = {
    'GFW': 'green',
    'Dynamic World': 'gold',
    'ESA WorldCover': 'gray',
    'Global-4 class PALSAR-2': 'red'
}

# Bar plot for RMSE
bar_width = 0.2
x = np.arange(len(countries))

for i, dataset in enumerate(datasets):
    values = rmse_data[dataset]
    bars = axs[0].bar(x + i * bar_width, values, width=bar_width, label=dataset, color=colors[dataset])
    for j, val in enumerate(values):
        axs[0].text(x[j] + i * bar_width, val + 0.005, f"{val:.2f}", 
                    ha='center', va='bottom', fontsize=12, fontweight='bold', color='black')  # <- Enlarged and bold

axs[0].set_title('(a) RMSE (Lower Value Represents Greater Accuracy)')
axs[0].set_ylabel('RMSE Value')
axs[0].set_xticks(x + bar_width * 1.5)
axs[0].set_xticklabels(countries)
axs[0].legend(title='Datasets')
axs[0].invert_yaxis()

# Line plot for AUC
for dataset in datasets:
    axs[1].plot(countries, auc_data[dataset], marker='o', label=dataset, color=colors[dataset])

axs[1].set_title('(b) AUC (Higher Value Represents Greater Accuracy)')
axs[1].set_ylabel('AUC Value')
axs[1].legend(title='Datasets')

plt.tight_layout()
plt.show()

                     # Example code if doing accuracy assessment for one country e.g Zambia ##

import ee
import geopandas as gpd
import geemap
import pandas as pd

# Authenticate and Initialize Earth Engine
ee.Authenticate()
ee.Initialize()

# Function to load data for one country (Zambia in this example)
def load_country_data():
    aoi_path = "C:\\Users\\49176\\Downloads\\ZambiaShape.zip"
    points_path = "C:\\Users\\49176\\Downloads\\Zambia_NICFI_Points.zip"
    aoi_file = "ZambiaShape.shp"
    points_file = "Points_NICFI_Zambia.shp"

    aoi_gdf = gpd.read_file(f"/vsizip/{aoi_path}/{aoi_file}")
    points_gdf = gpd.read_file(f"/vsizip/{points_path}/{points_file}")
    return geemap.gdf_to_ee(aoi_gdf), geemap.gdf_to_ee(points_gdf)

# Function to process each dataset
def process_dataset(dataset_name, aoi_fc, actual_points_fc, country_name, results):
    classification_image = None
    resolution = None

    if dataset_name == "DynamicWorld":
        image = ee.ImageCollection('GOOGLE/DYNAMICWORLD/V1') \
            .filterBounds(aoi_fc) \
            .filterDate('2020-01-01', '2020-12-31') \
            .select(['trees']).mean()
        classification_image = image.gte(0.5).rename('classification')
        resolution = 10
    elif dataset_name == "PALSAR":
        palsar = ee.ImageCollection("JAXA/ALOS/PALSAR/YEARLY/FNF4") \
            .filterBounds(aoi_fc) \
            .filterDate('2020-01-01', '2020-12-31') \
            .mosaic()
        classification_image = palsar.expression(
            'fnf == 1 || fnf == 2 ? 1 : 0',
            {'fnf': palsar.select('fnf')}
        ).rename('classification')
        resolution = 25
    elif dataset_name == "GFW":
        image = ee.Image('UMD/hansen/global_forest_change_2022_v1_10').select('treecover2000')
        classification_image = image.gte(10).rename('classification')
        resolution = 30
    elif dataset_name == "ESA":
        image = ee.ImageCollection('ESA/WorldCover/v100') \
            .filterDate('2020-01-01', '2020-12-31') \
            .first()
        classification_image = image.expression(
            'b("Map") == 10 ? 1 : 0',
            {'Map': image.select('Map')}
        ).rename('classification')
        resolution = 10

    # Sample points and calculate confusion matrix
    samples = classification_image.sampleRegions(
        collection=actual_points_fc,
        properties=['id'],
        scale=resolution
    ).map(lambda f: f.set('classification', ee.Number(f.get('classification')).toInt()))

    cm = samples.errorMatrix('id', 'classification')
    matrix = cm.getInfo()

    def safe_val(r, c):
        try:
            return matrix[r][c]
        except:
            return 0

    TN = safe_val(0, 0)
    FP = safe_val(0, 1)
    FN = safe_val(1, 0)
    TP = safe_val(1, 1)

    precision = TP / (TP + FP) if (TP + FP) > 0 else 0
    recall = TP / (TP + FN) if (TP + FN) > 0 else 0
    f1 = (2 * precision * recall) / (precision + recall) if (precision + recall) > 0 else 0
    accuracy = cm.accuracy().getInfo()
    kappa = cm.kappa().getInfo()

    # RMSE
    squared_diff = samples.map(lambda f: f.set(
        'squared_diff', ee.Number(f.get('id')).subtract(f.get('classification')).pow(2)
    ))
    mean_squared = squared_diff.reduceColumns(
        reducer=ee.Reducer.mean(),
        selectors=['squared_diff']
    ).get('mean')
    rmse = ee.Number(mean_squared).sqrt().getInfo()

    # AUC
    tpr = TP / (TP + FN) if (TP + FN) > 0 else 0
    fpr = FP / (FP + TN) if (FP + TN) > 0 else 0
    auc = tpr / (tpr + fpr) if (tpr + fpr) > 0 else 0

    results.append({
        "Country": country_name,
        "Dataset": dataset_name,
        "TP": TP,
        "TN": TN,
        "FP": FP,
        "FN": FN,
        "Precision": precision,
        "Recall": recall,
        "F1 Score": f1,
        "Accuracy": accuracy,
        "Kappa": kappa,
        "RMSE": rmse,
        "AUC": auc
    })

# Run for Zambia only
def run_zambia():
    country = "Zambia"
    datasets = ["DynamicWorld", "PALSAR", "GFW", "ESA"]
    results = []

    aoi_fc, points_fc = load_country_data()
    for dataset in datasets:
        process_dataset(dataset, aoi_fc, points_fc, country, results)

    df = pd.DataFrame(results)
    print("\nZambia Classification Results:")
    print(df.to_string(index=False))
    df.to_csv("zambia_classification_results.csv", index=False)
    print("\nSaved to 'zambia_classification_results.csv'")

# Execute for Zambia
run_zambia()

