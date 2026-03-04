                                     # ACCURACY ASSESSMENT FOR PARAGUAY, ZAMBIA AND ZIMBABWE
# 1. Define the relative path for the shapefiles, Paraguay, Zambia and Zimbabwe and check the AOI features
#    The country boundaries can be accessed here: https://gadm.org/download_country.html 

import zipfile
from pathlib import Path
import geopandas as gpd

def read_first_shp_from_zip(zip_path):
    """
    Reads the first .shp found inside a ZIP using GDAL /vsizip/.
    If there are multiple .shp files, it prints them so you can choose.
    """
    zip_path = Path(zip_path)

    if not zip_path.exists():
        raise FileNotFoundError(f"ZIP not found: {zip_path}")

    with zipfile.ZipFile(zip_path, "r") as z:
        shp_files = [n for n in z.namelist() if n.lower().endswith(".shp")]
        if not shp_files:
            raise FileNotFoundError(f"No .shp found inside ZIP: {zip_path.name}")

        print(f"\n{zip_path.name} -> SHP candidates:")
        for s in shp_files:
            print("  ", s)

        shp_in_zip = shp_files[0]  # pick the first by default

    vsi_path = f"/vsizip/{zip_path.as_posix()}/{shp_in_zip}"
    print("Reading:", vsi_path)

    gdf = gpd.read_file(vsi_path)
    return gdf

# ---- Paths (same user folder) ----
# Ensure to include the path to your local folder after downloading the shapefile zip from Github
paraguay_zip = r"C:\Users\gapnat\Downloads\ParaguayShape.zip"
zambia_zip   = r"C:\Users\gapnat\Downloads\ZambiaShape.zip"
zimbabwe_zip = r"C:\Users\gapnat\Downloads\ZimbabweShape.zip"

# ---- Load all three ----
paraguay = read_first_shp_from_zip(paraguay_zip)
zambia   = read_first_shp_from_zip(zambia_zip)
zimbabwe = read_first_shp_from_zip(zimbabwe_zip)

# ---- Checks (as in your example) ----
print("\n===== PARAGUAY =====")
print(f"Number of features: {len(paraguay)}")
print("First feature:")
print(paraguay.iloc[0])

print("\n===== ZAMBIA =====")
print(f"Number of features: {len(zambia)}")
print("First feature:")
print(zambia.iloc[0])

print("\n===== ZIMBABWE =====")
print(f"Number of features: {len(zimbabwe)}")
print("First feature:")
print(zimbabwe.iloc[0])

# 2. One-time environment setup (Restart kennel after setup)
import sys

# Confirm which Python this notebook is using
print("Notebook Python:", sys.executable)

# Install everything into THIS exact Python environment (same kernel)
!"{sys.executable}" -m pip install -U folium geemap earthengine-api ipywidgets

print("Done. Now restart the kernel (Kernel -> Restart), then re-import the packages.")

# 3. Import libraries
import folium
import geemap
import ee
print("folium:", folium.__version__)
print("geemap:", geemap.__version__)
print("ee:", ee.__version__)

# 4. Visualise the raw dataset images for (Dynamic World, PALSAR, Hansen and ESA WorldCover) before reclassification
#    (i) PARAGUAY
import zipfile
from pathlib import Path

import geopandas as gpd
import ee
import folium
import geemap

# Authenticate and Initialize Earth Engine
ee.Authenticate()
ee.Initialize()

# ----------------------------
# Helper: find the correct .shp inside the ZIP (no guessing)
# ----------------------------
def get_shp_in_zip(zip_path_str, preference_keywords=None):
    zip_path = Path(zip_path_str)
    if not zip_path.exists():
        raise FileNotFoundError(f"ZIP not found: {zip_path}")

    with zipfile.ZipFile(zip_path, "r") as z:
        shp_files = [n for n in z.namelist() if n.lower().endswith(".shp")]
        if not shp_files:
            raise FileNotFoundError(f"No .shp found inside ZIP: {zip_path.name}")

        chosen = None
        if preference_keywords:
            lower_files = [(n, n.lower()) for n in shp_files]
            for kw in preference_keywords:
                kw = kw.lower()
                for n, nl in lower_files:
                    if kw in nl:
                        chosen = n
                        break
                if chosen:
                    break

        shp_in_zip = chosen if chosen else shp_files[0]

        print(f"\n{zip_path.name} -> SHP candidates:")
        for s in shp_files:
            print("  ", s)
        print("Chosen SHP:", shp_in_zip)

    return shp_in_zip

# ----------------------------
# Load the shapefile as AOI (UPDATED PATH)
# ----------------------------
shapefile_path = r"C:\Users\gapnat\Downloads\ParaguayShape.zip"

# pick the right shp if multiple exist (adjust keywords if needed)
shp_in_zip = get_shp_in_zip(shapefile_path, preference_keywords=["paraguay", "pry", "_0", "adm0"])

shapefile_inside_zip = f"/vsizip/{Path(shapefile_path).as_posix()}/{shp_in_zip}"
aoi_gdf = gpd.read_file(shapefile_inside_zip)

print(f"Number of features in AOI: {len(aoi_gdf)}")
print("First feature of AOI:")
print(aoi_gdf.iloc[0])

# Convert AOI to Earth Engine FeatureCollection
aoi = geemap.gdf_to_ee(aoi_gdf)

# Dynamically compute AOI centroid for map centering
aoi_centroid = aoi.geometry().centroid().coordinates().getInfo()
center = [aoi_centroid[1], aoi_centroid[0]]  # [lat, lon]

# Create a folium map
map = folium.Map(location=center, zoom_start=6)

# Define a function to add Earth Engine layers to folium map
def add_ee_layer(self, ee_object, vis_params, name):
    """Function to add Earth Engine layers to a folium map."""
    if isinstance(ee_object, ee.ImageCollection):
        ee_object = ee_object.mosaic()
    map_id_dict = ee.Image(ee_object).getMapId(vis_params)
    folium.TileLayer(
        tiles=map_id_dict["tile_fetcher"].url_format,
        attr="Map Data &copy; Google Earth Engine",
        name=name,
        overlay=True,
        control=True,
    ).add_to(self)

# Bind the method to the folium map instance
folium.Map.add_ee_layer = add_ee_layer

# Add AOI boundary to the map
map.add_ee_layer(aoi.style(color="red", width=2), {}, "AOI Boundary")

# Define visualization function
def visualize_reclassification(dataset_name):
    """Adds dataset layers to the folium map."""
    if dataset_name == "DynamicWorld":
        dynamic_world = (
            ee.ImageCollection("GOOGLE/DYNAMICWORLD/V1")
            .filterBounds(aoi)
            .filterDate("2020-01-01", "2020-12-31")
        )
        raw_image = dynamic_world.select("trees").mean().clip(aoi).reproject(crs="EPSG:4326")
        vis_params = {"min": 0, "max": 1, "palette": ["white", "green"]}
        map.add_ee_layer(raw_image, vis_params, "DynamicWorld - Trees")

    elif dataset_name == "PALSAR":
        palsar = (
            ee.ImageCollection("JAXA/ALOS/PALSAR/YEARLY/FNF4")
            .filterBounds(aoi)
            .filterDate("2020-01-01", "2020-12-31")
            .mosaic()
        )
        raw_image = palsar.select("fnf").clip(aoi).reproject(crs="EPSG:4326", scale=100)
        vis_params = {"min": 1, "max": 4, "palette": ["#00b200", "#83ef62", "#ffff99", "#0000ff"]}
        map.add_ee_layer(raw_image, vis_params, "PALSAR - Forest/Non-Forest")

    elif dataset_name == "GFW":
        hansen_data = ee.Image("UMD/hansen/global_forest_change_2022_v1_10")
        treecover2000 = hansen_data.select("treecover2000").clip(aoi)
        loss = hansen_data.select("lossyear").clip(aoi)

        # Create a loss mask for the year 2020 only
        loss_mask_2020 = loss.eq(20)

        # Forest cover after excluding 2020 loss
        forest_cover_after_loss_2020 = treecover2000.updateMask(loss_mask_2020.Not())

        vis_params = {"min": 0, "max": 100, "palette": ["white", "green"]}
        map.add_ee_layer(forest_cover_after_loss_2020, vis_params, "GFW - Forest Cover After 2020 Loss")

    elif dataset_name == "ESA":
        esa_image = (
            ee.ImageCollection("ESA/WorldCover/v100")
            .filterDate("2020-01-01", "2020-12-31")
            .first()
            .select("Map")
            .clip(aoi)
            .reproject(crs="EPSG:4326", scale=100)
        )
        vis_params = {
            "min": 10,
            "max": 100,
            "palette": [
                "#006400", "#ffbb22", "#ffff4c", "#f096ff", "#fa0000",
                "#b4b4b4", "#f0f0f0", "#0064c8", "#0096a0", "#00cf75", "#fae6a0"
            ],
        }
        map.add_ee_layer(esa_image, vis_params, "ESA - WorldCover")

# Visualize datasets
datasets = ["DynamicWorld", "PALSAR", "GFW", "ESA"]
for dataset in datasets:
    visualize_reclassification(dataset)

# Add layer control and display map
folium.LayerControl().add_to(map)
# map.save("Paraguay_visualized_map_with_loss_2020.html")
map

# 4.Visualise the raw dataset images for (Dynamic World, PALSAR, Hansen and ESA WorldCover) before reclassification
# (ii) ZAMBIA

import zipfile
from pathlib import Path

import geopandas as gpd
import ee
import folium
import geemap

# Authenticate and Initialize Earth Engine
ee.Authenticate()
ee.Initialize()

# ----------------------------
# Helper: find the correct .shp inside the ZIP (no guessing)
# ----------------------------
def get_shp_in_zip(zip_path_str, preference_keywords=None):
    zip_path = Path(zip_path_str)
    if not zip_path.exists():
        raise FileNotFoundError(f"ZIP not found: {zip_path}")

    with zipfile.ZipFile(zip_path, "r") as z:
        shp_files = [n for n in z.namelist() if n.lower().endswith(".shp")]
        if not shp_files:
            raise FileNotFoundError(f"No .shp found inside ZIP: {zip_path.name}")

        chosen = None
        if preference_keywords:
            lower_files = [(n, n.lower()) for n in shp_files]
            for kw in preference_keywords:
                kw = kw.lower()
                for n, nl in lower_files:
                    if kw in nl:
                        chosen = n
                        break
                if chosen:
                    break

        shp_in_zip = chosen if chosen else shp_files[0]

        print(f"\n{zip_path.name} -> SHP candidates:")
        for s in shp_files:
            print("  ", s)
        print("Chosen SHP:", shp_in_zip)

    return shp_in_zip

# ----------------------------
# Load the shapefile as AOI (UPDATED PATH)
# ----------------------------
shapefile_path = r"C:\Users\gapnat\Downloads\ZambiaShape.zip"

# pick the right shp if multiple exist (adjust keywords if needed)
shp_in_zip = get_shp_in_zip(shapefile_path, preference_keywords=["zambia", "zmb", "_0", "adm0"])

shapefile_inside_zip = f"/vsizip/{Path(shapefile_path).as_posix()}/{shp_in_zip}"
aoi_gdf = gpd.read_file(shapefile_inside_zip)

print(f"Number of features in AOI: {len(aoi_gdf)}")
print("First feature of AOI:")
print(aoi_gdf.iloc[0])

# Convert AOI to Earth Engine FeatureCollection
aoi = geemap.gdf_to_ee(aoi_gdf)

# Dynamically compute AOI centroid for map centering
aoi_centroid = aoi.geometry().centroid().coordinates().getInfo()
center = [aoi_centroid[1], aoi_centroid[0]]  # [lat, lon]

# Create a folium map
map = folium.Map(location=center, zoom_start=6)

# Define a function to add Earth Engine layers to folium map
def add_ee_layer(self, ee_object, vis_params, name):
    """Function to add Earth Engine layers to a folium map."""
    if isinstance(ee_object, ee.ImageCollection):
        ee_object = ee_object.mosaic()
    map_id_dict = ee.Image(ee_object).getMapId(vis_params)
    folium.TileLayer(
        tiles=map_id_dict["tile_fetcher"].url_format,
        attr="Map Data &copy; Google Earth Engine",
        name=name,
        overlay=True,
        control=True,
    ).add_to(self)

# Bind the method to the folium map instance
folium.Map.add_ee_layer = add_ee_layer

# Add AOI boundary to the map
map.add_ee_layer(aoi.style(color="red", width=2), {}, "AOI Boundary")

# Define visualization function
def visualize_reclassification(dataset_name):
    """Adds dataset layers to the folium map."""
    if dataset_name == "DynamicWorld":
        dynamic_world = (
            ee.ImageCollection("GOOGLE/DYNAMICWORLD/V1")
            .filterBounds(aoi)
            .filterDate("2020-01-01", "2020-12-31")
        )
        raw_image = dynamic_world.select("trees").mean().clip(aoi).reproject(crs="EPSG:4326")
        vis_params = {"min": 0, "max": 1, "palette": ["white", "green"]}
        map.add_ee_layer(raw_image, vis_params, "DynamicWorld - Trees")

    elif dataset_name == "PALSAR":
        palsar = (
            ee.ImageCollection("JAXA/ALOS/PALSAR/YEARLY/FNF4")
            .filterBounds(aoi)
            .filterDate("2020-01-01", "2020-12-31")
            .mosaic()
        )
        raw_image = palsar.select("fnf").clip(aoi).reproject(crs="EPSG:4326", scale=100)
        vis_params = {"min": 1, "max": 4, "palette": ["#00b200", "#83ef62", "#ffff99", "#0000ff"]}
        map.add_ee_layer(raw_image, vis_params, "PALSAR - Forest/Non-Forest")

    elif dataset_name == "GFW":
        hansen_data = ee.Image("UMD/hansen/global_forest_change_2022_v1_10")
        treecover2000 = hansen_data.select("treecover2000").clip(aoi)
        loss = hansen_data.select("lossyear").clip(aoi)

        # Create a loss mask for the year 2020 only
        loss_mask_2020 = loss.eq(20)

        # Forest cover after excluding 2020 loss
        forest_cover_after_loss_2020 = treecover2000.updateMask(loss_mask_2020.Not())

        vis_params = {"min": 0, "max": 100, "palette": ["white", "green"]}
        map.add_ee_layer(forest_cover_after_loss_2020, vis_params, "GFW - Forest Cover After 2020 Loss")

    elif dataset_name == "ESA":
        esa_image = (
            ee.ImageCollection("ESA/WorldCover/v100")
            .filterDate("2020-01-01", "2020-12-31")
            .first()
            .select("Map")
            .clip(aoi)
            .reproject(crs="EPSG:4326", scale=100)
        )
        vis_params = {
            "min": 10,
            "max": 100,
            "palette": [
                "#006400", "#ffbb22", "#ffff4c", "#f096ff", "#fa0000",
                "#b4b4b4", "#f0f0f0", "#0064c8", "#0096a0", "#00cf75", "#fae6a0"
            ],
        }
        map.add_ee_layer(esa_image, vis_params, "ESA - WorldCover")

# Visualize datasets
datasets = ["DynamicWorld", "PALSAR", "GFW", "ESA"]
for dataset in datasets:
    visualize_reclassification(dataset)

# Add layer control and display map
folium.LayerControl().add_to(map)
# map.save("Zambia_visualized_map_with_loss_2020.html")
map

# 4. Visualise the raw dataset images for (Dynamic World, PALSAR, Hansen and ESA WorldCover) before reclassification
#    (iii) ZIMBABWE

import zipfile
from pathlib import Path

import geopandas as gpd
import ee
import folium
import geemap

# Authenticate and Initialize Earth Engine
ee.Authenticate()
ee.Initialize()

# ----------------------------
# Helper: find the correct .shp inside the ZIP (no guessing)
# ----------------------------
def get_shp_in_zip(zip_path_str, preference_keywords=None):
    zip_path = Path(zip_path_str)
    if not zip_path.exists():
        raise FileNotFoundError(f"ZIP not found: {zip_path}")

    with zipfile.ZipFile(zip_path, "r") as z:
        shp_files = [n for n in z.namelist() if n.lower().endswith(".shp")]
        if not shp_files:
            raise FileNotFoundError(f"No .shp found inside ZIP: {zip_path.name}")

        chosen = None
        if preference_keywords:
            lower_files = [(n, n.lower()) for n in shp_files]
            for kw in preference_keywords:
                kw = kw.lower()
                for n, nl in lower_files:
                    if kw in nl:
                        chosen = n
                        break
                if chosen:
                    break

        shp_in_zip = chosen if chosen else shp_files[0]

        print(f"\n{zip_path.name} -> SHP candidates:")
        for s in shp_files:
            print("  ", s)
        print("Chosen SHP:", shp_in_zip)

    return shp_in_zip

# ----------------------------
# Load the shapefile as AOI (UPDATED PATH)
# ----------------------------
shapefile_path = r"C:\Users\gapnat\Downloads\ZimbabweShape.zip"

# pick the right shp if multiple exist (adjust keywords if needed)
shp_in_zip = get_shp_in_zip(shapefile_path, preference_keywords=["zimbabwe", "zwe", "_0", "adm0"])

shapefile_inside_zip = f"/vsizip/{Path(shapefile_path).as_posix()}/{shp_in_zip}"
aoi_gdf = gpd.read_file(shapefile_inside_zip)

print(f"Number of features in AOI: {len(aoi_gdf)}")
print("First feature of AOI:")
print(aoi_gdf.iloc[0])

# Convert AOI to Earth Engine FeatureCollection
aoi = geemap.gdf_to_ee(aoi_gdf)

# Dynamically compute AOI centroid for map centering
aoi_centroid = aoi.geometry().centroid().coordinates().getInfo()
center = [aoi_centroid[1], aoi_centroid[0]]  # [lat, lon]

# Create a folium map
map = folium.Map(location=center, zoom_start=6)

# Define a function to add Earth Engine layers to folium map
def add_ee_layer(self, ee_object, vis_params, name):
    """Function to add Earth Engine layers to a folium map."""
    if isinstance(ee_object, ee.ImageCollection):
        ee_object = ee_object.mosaic()
    map_id_dict = ee.Image(ee_object).getMapId(vis_params)
    folium.TileLayer(
        tiles=map_id_dict["tile_fetcher"].url_format,
        attr="Map Data &copy; Google Earth Engine",
        name=name,
        overlay=True,
        control=True,
    ).add_to(self)

# Bind the method to the folium map instance
folium.Map.add_ee_layer = add_ee_layer

# Add AOI boundary to the map
map.add_ee_layer(aoi.style(color="red", width=2), {}, "AOI Boundary")

# Define visualization function
def visualize_reclassification(dataset_name):
    """Adds dataset layers to the folium map."""
    if dataset_name == "DynamicWorld":
        dynamic_world = (
            ee.ImageCollection("GOOGLE/DYNAMICWORLD/V1")
            .filterBounds(aoi)
            .filterDate("2020-01-01", "2020-12-31")
        )
        raw_image = dynamic_world.select("trees").mean().clip(aoi).reproject(crs="EPSG:4326")
        vis_params = {"min": 0, "max": 1, "palette": ["white", "green"]}
        map.add_ee_layer(raw_image, vis_params, "DynamicWorld - Trees")

    elif dataset_name == "PALSAR":
        palsar = (
            ee.ImageCollection("JAXA/ALOS/PALSAR/YEARLY/FNF4")
            .filterBounds(aoi)
            .filterDate("2020-01-01", "2020-12-31")
            .mosaic()
        )
        raw_image = palsar.select("fnf").clip(aoi).reproject(crs="EPSG:4326", scale=100)
        vis_params = {"min": 1, "max": 4, "palette": ["#00b200", "#83ef62", "#ffff99", "#0000ff"]}
        map.add_ee_layer(raw_image, vis_params, "PALSAR - Forest/Non-Forest")

    elif dataset_name == "GFW":
        hansen_data = ee.Image("UMD/hansen/global_forest_change_2022_v1_10")
        treecover2000 = hansen_data.select("treecover2000").clip(aoi)
        loss = hansen_data.select("lossyear").clip(aoi)

        # Create a loss mask for the year 2020 only
        loss_mask_2020 = loss.eq(20)

        # Forest cover after excluding 2020 loss
        forest_cover_after_loss_2020 = treecover2000.updateMask(loss_mask_2020.Not())

        vis_params = {"min": 0, "max": 100, "palette": ["white", "green"]}
        map.add_ee_layer(forest_cover_after_loss_2020, vis_params, "GFW - Forest Cover After 2020 Loss")

    elif dataset_name == "ESA":
        esa_image = (
            ee.ImageCollection("ESA/WorldCover/v100")
            .filterDate("2020-01-01", "2020-12-31")
            .first()
            .select("Map")
            .clip(aoi)
            .reproject(crs="EPSG:4326", scale=100)
        )
        vis_params = {
            "min": 10,
            "max": 100,
            "palette": [
                "#006400", "#ffbb22", "#ffff4c", "#f096ff", "#fa0000",
                "#b4b4b4", "#f0f0f0", "#0064c8", "#0096a0", "#00cf75", "#fae6a0"
            ],
        }
        map.add_ee_layer(esa_image, vis_params, "ESA - WorldCover")

# Visualize datasets
datasets = ["DynamicWorld", "PALSAR", "GFW", "ESA"]
for dataset in datasets:
    visualize_reclassification(dataset)

# Add layer control and display map
folium.LayerControl().add_to(map)
# map.save("Zimbabwe_visualized_map_with_loss_2020.html")
map

# 5. Reclassifcation of raw dataset images (Dynamic World, PALSAR, Hansen and ESA WorldCover) to forest/non-forest maps
#    (i) PARAGUAY


import zipfile
from pathlib import Path

import geopandas as gpd
import ee
import folium
import geemap

# Authenticate and Initialize Earth Engine
ee.Authenticate()
ee.Initialize()

# ----------------------------
# Helper: find the correct .shp inside the ZIP (no guessing)
# ----------------------------
def get_shp_in_zip(zip_path_str, preference_keywords=None):
    zip_path = Path(zip_path_str)
    if not zip_path.exists():
        raise FileNotFoundError(f"ZIP not found: {zip_path}")

    with zipfile.ZipFile(zip_path, "r") as z:
        shp_files = [n for n in z.namelist() if n.lower().endswith(".shp")]
        if not shp_files:
            raise FileNotFoundError(f"No .shp found inside ZIP: {zip_path.name}")

        chosen = None
        if preference_keywords:
            lower_files = [(n, n.lower()) for n in shp_files]
            for kw in preference_keywords:
                kw = kw.lower()
                for n, nl in lower_files:
                    if kw in nl:
                        chosen = n
                        break
                if chosen:
                    break

        shp_in_zip = chosen if chosen else shp_files[0]

        print(f"\n{zip_path.name} -> SHP candidates:")
        for s in shp_files:
            print("  ", s)
        print("Chosen SHP:", shp_in_zip)

    return shp_in_zip

# ----------------------------
# Load the shapefile as AOI (UPDATED PATH)
# ----------------------------
shapefile_path = r"C:\Users\gapnat\Downloads\ParaguayShape.zip"

# Pick the right shp if multiple exist (adjust keywords if needed)
shp_in_zip = get_shp_in_zip(shapefile_path, preference_keywords=["paraguay", "pry", "_0", "adm0"])

shapefile_inside_zip = f"/vsizip/{Path(shapefile_path).as_posix()}/{shp_in_zip}"
aoi_gdf = gpd.read_file(shapefile_inside_zip)

print(f"Number of features in AOI: {len(aoi_gdf)}")
print("First feature of AOI:")
print(aoi_gdf.iloc[0])

# Convert AOI to Earth Engine FeatureCollection
aoi = geemap.gdf_to_ee(aoi_gdf)

# Dynamically compute AOI centroid for map centering
aoi_centroid = aoi.geometry().centroid().coordinates().getInfo()
center = [aoi_centroid[1], aoi_centroid[0]]  # [lat, lon]

# Create a folium map
map = folium.Map(location=center, zoom_start=6)

# Define a function to add Earth Engine layers to folium map
def add_ee_layer(self, ee_object, vis_params, name):
    """Function to add Earth Engine layers to a folium map."""
    if isinstance(ee_object, ee.ImageCollection):
        ee_object = ee_object.mosaic()
    map_id_dict = ee.Image(ee_object).getMapId(vis_params)
    folium.TileLayer(
        tiles=map_id_dict["tile_fetcher"].url_format,
        attr="Map Data &copy; Google Earth Engine",
        name=name,
        overlay=True,
        control=True,
    ).add_to(self)

# Bind the method to the folium map instance
folium.Map.add_ee_layer = add_ee_layer

# Add AOI boundary to the map
map.add_ee_layer(aoi.style(color="red", width=2), {}, "AOI Boundary")

# ----------------------------
# Reclassify each dataset to forest/non-forest maps
# ----------------------------
def reclassify_to_forest(dataset_name):
    """Reclassifies dataset to binary forest/non-forest and visualizes on the map."""
    if dataset_name == "DynamicWorld":
        dynamic_world = (
            ee.ImageCollection("GOOGLE/DYNAMICWORLD/V1")
            .filterBounds(aoi)
            .filterDate("2020-01-01", "2020-12-31")
        )
        forest = dynamic_world.select("trees").mean().clip(aoi).gt(0.3)  # Threshold for forest
        vis_params = {"min": 0, "max": 1, "palette": ["white", "green"]}
        map.add_ee_layer(forest, vis_params, "DynamicWorld - Forest/Non-Forest")

    elif dataset_name == "PALSAR":
        palsar = (
            ee.ImageCollection("JAXA/ALOS/PALSAR/YEARLY/FNF4")
            .filterBounds(aoi)
            .filterDate("2020-01-01", "2020-12-31")
            .mosaic()
        )
        fnf = palsar.select("fnf").clip(aoi)
        forest = fnf.eq(1).Or(fnf.eq(2))  # Class 1 and 2 are forests
        vis_params = {"min": 0, "max": 1, "palette": ["white", "green"]}
        map.add_ee_layer(forest, vis_params, "PALSAR - Dense and Non-Dense Forest")

    elif dataset_name == "GFW":
        hansen_data = ee.Image("UMD/hansen/global_forest_change_2022_v1_10")
        treecover2000 = hansen_data.select("treecover2000").clip(aoi)
        loss = hansen_data.select("lossyear").clip(aoi)

        # Create a loss mask for 2020 only
        loss_mask_2020 = loss.eq(20)

        # Exclude 2020 loss from forest cover
        forest_cover_after_loss = treecover2000.updateMask(loss_mask_2020.Not())

        # Reclassify remaining forest (>10% tree cover) and non-forest
        forest = forest_cover_after_loss.gt(10)

        vis_params = {"min": 0, "max": 1, "palette": ["white", "green"]}
        map.add_ee_layer(forest, vis_params, "GFW - Forest Cover After 2020 Loss")

    elif dataset_name == "ESA":
        esa_image = (
            ee.ImageCollection("ESA/WorldCover/v100")
            .filterDate("2020-01-01", "2020-12-31")
            .first()
            .select("Map")
            .clip(aoi)
        )
        forest = esa_image.eq(10)  # Class 10 represents tree cover in ESA WorldCover
        vis_params = {"min": 0, "max": 1, "palette": ["white", "green"]}
        map.add_ee_layer(forest, vis_params, "ESA - Forest/Non-Forest")

# Apply reclassification for each dataset
datasets = ["DynamicWorld", "PALSAR", "GFW", "ESA"]
for dataset in datasets:
    reclassify_to_forest(dataset)

# Add layer control and display map
folium.LayerControl().add_to(map)
# map.save("Paraguay_reclassified_map_with_gfw_2020.html")
map

# 5. Reclassifcation of raw dataset images (Dynamic World, PALSAR, Hansen and ESA WorldCover) to forest/non-forest maps
#    (ii) ZAMBIA

import zipfile
from pathlib import Path

import geopandas as gpd
import ee
import folium
import geemap

# Authenticate and Initialize Earth Engine
ee.Authenticate()
ee.Initialize()

# ----------------------------
# Helper: find the correct .shp inside the ZIP (no guessing)
# ----------------------------
def get_shp_in_zip(zip_path_str, preference_keywords=None):
    zip_path = Path(zip_path_str)
    if not zip_path.exists():
        raise FileNotFoundError(f"ZIP not found: {zip_path}")

    with zipfile.ZipFile(zip_path, "r") as z:
        shp_files = [n for n in z.namelist() if n.lower().endswith(".shp")]
        if not shp_files:
            raise FileNotFoundError(f"No .shp found inside ZIP: {zip_path.name}")

        chosen = None
        if preference_keywords:
            lower_files = [(n, n.lower()) for n in shp_files]
            for kw in preference_keywords:
                kw = kw.lower()
                for n, nl in lower_files:
                    if kw in nl:
                        chosen = n
                        break
                if chosen:
                    break

        shp_in_zip = chosen if chosen else shp_files[0]

        print(f"\n{zip_path.name} -> SHP candidates:")
        for s in shp_files:
            print("  ", s)
        print("Chosen SHP:", shp_in_zip)

    return shp_in_zip

# ----------------------------
# Load the shapefile as AOI (UPDATED PATH)
# ----------------------------
shapefile_path = r"C:\Users\gapnat\Downloads\ZambiaShape.zip"

# Pick the right shp if multiple exist (adjust keywords if needed)
shp_in_zip = get_shp_in_zip(shapefile_path, preference_keywords=["zambia", "zmb", "_0", "adm0"])

shapefile_inside_zip = f"/vsizip/{Path(shapefile_path).as_posix()}/{shp_in_zip}"
aoi_gdf = gpd.read_file(shapefile_inside_zip)

print(f"Number of features in AOI: {len(aoi_gdf)}")
print("First feature of AOI:")
print(aoi_gdf.iloc[0])

# Convert AOI to Earth Engine FeatureCollection
aoi = geemap.gdf_to_ee(aoi_gdf)

# Dynamically compute AOI centroid for map centering
aoi_centroid = aoi.geometry().centroid().coordinates().getInfo()
center = [aoi_centroid[1], aoi_centroid[0]]  # [lat, lon]

# Create a folium map
map = folium.Map(location=center, zoom_start=6)

# Define a function to add Earth Engine layers to folium map
def add_ee_layer(self, ee_object, vis_params, name):
    """Function to add Earth Engine layers to a folium map."""
    if isinstance(ee_object, ee.ImageCollection):
        ee_object = ee_object.mosaic()
    map_id_dict = ee.Image(ee_object).getMapId(vis_params)
    folium.TileLayer(
        tiles=map_id_dict["tile_fetcher"].url_format,
        attr="Map Data &copy; Google Earth Engine",
        name=name,
        overlay=True,
        control=True,
    ).add_to(self)

# Bind the method to the folium map instance
folium.Map.add_ee_layer = add_ee_layer

# Add AOI boundary to the map
map.add_ee_layer(aoi.style(color="red", width=2), {}, "AOI Boundary")

# ----------------------------
# Reclassify each dataset to forest/non-forest maps
# ----------------------------
def reclassify_to_forest(dataset_name):
    """Reclassifies dataset to binary forest/non-forest and visualizes on the map."""
    if dataset_name == "DynamicWorld":
        dynamic_world = (
            ee.ImageCollection("GOOGLE/DYNAMICWORLD/V1")
            .filterBounds(aoi)
            .filterDate("2020-01-01", "2020-12-31")
        )
        forest = dynamic_world.select("trees").mean().clip(aoi).gt(0.3)  # Threshold for forest
        vis_params = {"min": 0, "max": 1, "palette": ["white", "green"]}
        map.add_ee_layer(forest, vis_params, "DynamicWorld - Forest/Non-Forest")

    elif dataset_name == "PALSAR":
        palsar = (
            ee.ImageCollection("JAXA/ALOS/PALSAR/YEARLY/FNF4")
            .filterBounds(aoi)
            .filterDate("2020-01-01", "2020-12-31")
            .mosaic()
        )
        fnf = palsar.select("fnf").clip(aoi)
        forest = fnf.eq(1).Or(fnf.eq(2))  # Class 1 and 2 are forests
        vis_params = {"min": 0, "max": 1, "palette": ["white", "green"]}
        map.add_ee_layer(forest, vis_params, "PALSAR - Dense and Non-Dense Forest")

    elif dataset_name == "GFW":
        hansen_data = ee.Image("UMD/hansen/global_forest_change_2022_v1_10")
        treecover2000 = hansen_data.select("treecover2000").clip(aoi)
        loss = hansen_data.select("lossyear").clip(aoi)

        # Create a loss mask for 2020 only
        loss_mask_2020 = loss.eq(20)

        # Exclude 2020 loss from forest cover
        forest_cover_after_loss = treecover2000.updateMask(loss_mask_2020.Not())

        # Reclassify remaining forest (>10% tree cover) and non-forest
        forest = forest_cover_after_loss.gt(10)

        vis_params = {"min": 0, "max": 1, "palette": ["white", "green"]}
        map.add_ee_layer(forest, vis_params, "GFW - Forest Cover After 2020 Loss")

    elif dataset_name == "ESA":
        esa_image = (
            ee.ImageCollection("ESA/WorldCover/v100")
            .filterDate("2020-01-01", "2020-12-31")
            .first()
            .select("Map")
            .clip(aoi)
        )
        forest = esa_image.eq(10)  # Class 10 represents tree cover in ESA WorldCover
        vis_params = {"min": 0, "max": 1, "palette": ["white", "green"]}
        map.add_ee_layer(forest, vis_params, "ESA - Forest/Non-Forest")

# Apply reclassification for each dataset
datasets = ["DynamicWorld", "PALSAR", "GFW", "ESA"]
for dataset in datasets:
    reclassify_to_forest(dataset)

# Add layer control and display map
folium.LayerControl().add_to(map)
# map.save("Zambia_reclassified_map_with_gfw_2020.html")
map

# 5. Reclassifcation of raw dataset images (Dynamic World, PALSAR, Hansen and ESA WorldCover) to forest/non-forest maps
#    (iii) ZIMBABWE

import zipfile
from pathlib import Path

import geopandas as gpd
import ee
import folium
import geemap

# Authenticate and Initialize Earth Engine
ee.Authenticate()
ee.Initialize()

# ----------------------------
# Helper: find the correct .shp inside the ZIP (no guessing)
# ----------------------------
def get_shp_in_zip(zip_path_str, preference_keywords=None):
    zip_path = Path(zip_path_str)
    if not zip_path.exists():
        raise FileNotFoundError(f"ZIP not found: {zip_path}")

    with zipfile.ZipFile(zip_path, "r") as z:
        shp_files = [n for n in z.namelist() if n.lower().endswith(".shp")]
        if not shp_files:
            raise FileNotFoundError(f"No .shp found inside ZIP: {zip_path.name}")

        chosen = None
        if preference_keywords:
            lower_files = [(n, n.lower()) for n in shp_files]
            for kw in preference_keywords:
                kw = kw.lower()
                for n, nl in lower_files:
                    if kw in nl:
                        chosen = n
                        break
                if chosen:
                    break

        shp_in_zip = chosen if chosen else shp_files[0]

        print(f"\n{zip_path.name} -> SHP candidates:")
        for s in shp_files:
            print("  ", s)
        print("Chosen SHP:", shp_in_zip)

    return shp_in_zip

# ----------------------------
# Load the shapefile as AOI (UPDATED PATH)
# ----------------------------
shapefile_path = r"C:\Users\gapnat\Downloads\ZimbabweShape.zip"

# Pick the right shp if multiple exist (adjust keywords if needed)
shp_in_zip = get_shp_in_zip(shapefile_path, preference_keywords=["zimbabwe", "zwe", "_0", "adm0"])

shapefile_inside_zip = f"/vsizip/{Path(shapefile_path).as_posix()}/{shp_in_zip}"
aoi_gdf = gpd.read_file(shapefile_inside_zip)

print(f"Number of features in AOI: {len(aoi_gdf)}")
print("First feature of AOI:")
print(aoi_gdf.iloc[0])

# Convert AOI to Earth Engine FeatureCollection
aoi = geemap.gdf_to_ee(aoi_gdf)

# Dynamically compute AOI centroid for map centering
aoi_centroid = aoi.geometry().centroid().coordinates().getInfo()
center = [aoi_centroid[1], aoi_centroid[0]]  # [lat, lon]

# Create a folium map
map = folium.Map(location=center, zoom_start=6)

# Define a function to add Earth Engine layers to folium map
def add_ee_layer(self, ee_object, vis_params, name):
    """Function to add Earth Engine layers to a folium map."""
    if isinstance(ee_object, ee.ImageCollection):
        ee_object = ee_object.mosaic()
    map_id_dict = ee.Image(ee_object).getMapId(vis_params)
    folium.TileLayer(
        tiles=map_id_dict["tile_fetcher"].url_format,
        attr="Map Data &copy; Google Earth Engine",
        name=name,
        overlay=True,
        control=True,
    ).add_to(self)

# Bind the method to the folium map instance
folium.Map.add_ee_layer = add_ee_layer

# Add AOI boundary to the map
map.add_ee_layer(aoi.style(color="red", width=2), {}, "AOI Boundary")

# ----------------------------
# Reclassify each dataset to forest/non-forest maps
# ----------------------------
def reclassify_to_forest(dataset_name):
    """Reclassifies dataset to binary forest/non-forest and visualizes on the map."""
    if dataset_name == "DynamicWorld":
        dynamic_world = (
            ee.ImageCollection("GOOGLE/DYNAMICWORLD/V1")
            .filterBounds(aoi)
            .filterDate("2020-01-01", "2020-12-31")
        )
        forest = dynamic_world.select("trees").mean().clip(aoi).gt(0.3)  # Threshold for forest
        vis_params = {"min": 0, "max": 1, "palette": ["white", "green"]}
        map.add_ee_layer(forest, vis_params, "DynamicWorld - Forest/Non-Forest")

    elif dataset_name == "PALSAR":
        palsar = (
            ee.ImageCollection("JAXA/ALOS/PALSAR/YEARLY/FNF4")
            .filterBounds(aoi)
            .filterDate("2020-01-01", "2020-12-31")
            .mosaic()
        )
        fnf = palsar.select("fnf").clip(aoi)
        forest = fnf.eq(1).Or(fnf.eq(2))  # Class 1 and 2 are forests
        vis_params = {"min": 0, "max": 1, "palette": ["white", "green"]}
        map.add_ee_layer(forest, vis_params, "PALSAR - Dense and Non-Dense Forest")

    elif dataset_name == "GFW":
        hansen_data = ee.Image("UMD/hansen/global_forest_change_2022_v1_10")
        treecover2000 = hansen_data.select("treecover2000").clip(aoi)
        loss = hansen_data.select("lossyear").clip(aoi)

        # Create a loss mask for 2020 only
        loss_mask_2020 = loss.eq(20)

        # Exclude 2020 loss from forest cover
        forest_cover_after_loss = treecover2000.updateMask(loss_mask_2020.Not())

        # Reclassify remaining forest (>10% tree cover) and non-forest
        forest = forest_cover_after_loss.gt(10)

        vis_params = {"min": 0, "max": 1, "palette": ["white", "green"]}
        map.add_ee_layer(forest, vis_params, "GFW - Forest Cover After 2020 Loss")

    elif dataset_name == "ESA":
        esa_image = (
            ee.ImageCollection("ESA/WorldCover/v100")
            .filterDate("2020-01-01", "2020-12-31")
            .first()
            .select("Map")
            .clip(aoi)
        )
        forest = esa_image.eq(10)  # Class 10 represents tree cover in ESA WorldCover
        vis_params = {"min": 0, "max": 1, "palette": ["white", "green"]}
        map.add_ee_layer(forest, vis_params, "ESA - Forest/Non-Forest")

# Apply reclassification for each dataset
datasets = ["DynamicWorld", "PALSAR", "GFW", "ESA"]
for dataset in datasets:
    reclassify_to_forest(dataset)

# Add layer control and display map
folium.LayerControl().add_to(map)
# map.save("Zimbabwe_reclassified_map_with_gfw_2020.html")
map

# 6.  Forest Cover Estimates

import ee
ee.Initialize()

# Hansen GFC
gfc = ee.Image("UMD/hansen/global_forest_change_2024_v1_12")
treecover2000 = gfc.select("treecover2000")
lossyear = gfc.select("lossyear")  # 1 = 2001 ... 22 = 2022
pixel_area = ee.Image.pixelArea().divide(10000)  # m² → ha

# Countries to process
countries = ["Paraguay", "Zambia", "Zimbabwe"]

# Canopy threshold for defining 2000 forest
CANOPY_THRESHOLD = 10   # change to 30 for GFW-style forest

# Loop over countries
for country_name in countries:

    # Load boundary
    country = ee.FeatureCollection("FAO/GAUL/2015/level0") \
        .filter(ee.Filter.eq("ADM0_NAME", country_name))

    # Forest in 2000
    forest2000 = treecover2000.gt(CANOPY_THRESHOLD)

    # Compute 2000 forest area
    area2000_dict = forest2000.multiply(pixel_area) \
        .reduceRegion(
            reducer=ee.Reducer.sum(),
            geometry=country.geometry(),
            scale=30,
            maxPixels=1e13
        ).getInfo()

    forest2000_area = area2000_dict.get('treecover2000', 0)

    print(f"\n==================== {country_name} ====================")
    print(f"Forest in 2000 (> {CANOPY_THRESHOLD}% canopy): {forest2000_area:,.2f} ha\n")

    # For storing cumulative loss
    cumulative_loss = 0

    print("Year | Annual Loss (ha) | Cumulative Loss (ha) | Remaining Forest (ha)")
    print("-----------------------------------------------------------------------")

    for year in range(2015, 2023):   # 2015–2022
        loss_index = year - 2000     # Hansen index (15 = 2015)

        # Annual loss mask
        annual_loss_mask = lossyear.eq(loss_index)

        # Annual loss area
        loss_area_dict = annual_loss_mask.multiply(pixel_area) \
            .reduceRegion(
                reducer=ee.Reducer.sum(),
                geometry=country.geometry(),
                scale=30,
                maxPixels=1e13
            ).getInfo()

        annual_loss_area = loss_area_dict.get('lossyear', 0)

        # Update cumulative loss
        cumulative_loss += annual_loss_area

        # Remaining forest = baseline 2000 forest − cumulative loss
        remaining_forest = forest2000_area - cumulative_loss

        # Print formatted
        print(f"{year} | {annual_loss_area:,.2f} | {cumulative_loss:,.2f} | {remaining_forest:,.2f}")

        # 7. Plot graph of 2020 forest estimates 
import matplotlib.pyplot as plt

# Updated years (2015–2022)
years = [2015, 2016, 2017, 2018, 2019, 2020, 2021, 2022]

# Updated GFW values (in million ha)
paraguay_gfw = [
    18.016606, 17.666802, 17.267822, 16.992154,
    16.636968, 16.365407, 16.053072, 15.800307
]

zambia_gfw = [
    57.901367, 57.637866, 57.262845, 56.951167,
    56.689298, 56.343695, 55.928173, 55.567610
]

zimbabwe_gfw = [
    17.553340, 17.528325, 17.460919, 17.439084,
    17.412315, 17.394730, 17.361824, 17.339020
]

# Existing other datasets (unchanged)
paraguay_palsar = [11, 13, 19, 20.5, 18.9, 17.8, None, None]
paraguay_dw = [15, 22, 20, 22, 20.5, 19.5, 20, 18.7]
paraguay_esa = [None, None, None, None, None, 16.5, None, None]

zambia_palsar = [36, 34, 41, 45, 40, 41, None, None]
zambia_dw = [10, 29, 33, 34, 30, 29, 33, 35]
zambia_esa = [None, None, None, None, None, 31, None, None]

zimbabwe_palsar = [8.2, 7.8, 14.3, 16, 13, 13.2, None, None]
zimbabwe_dw = [1, 3, 5.8, 6, 3.3, 3.5, 6.3, 6.5]
zimbabwe_esa = [None, None, None, None, None, 4.5, None, None]

# Create subplots
fig, axes = plt.subplots(3, 1, figsize=(8, 10), sharex=True)

def plot_forest_cover(ax, country, gfw, palsar, dw, esa):
    ax.plot(years, gfw, marker='o', color='green', label='GFW')
    ax.plot(years, palsar, marker='o', color='blue', label='PALSAR')
    ax.plot(years, dw, marker='o', color='purple', label='DynamicWorld')
    ax.scatter(years, esa, color='red', s=50, label='ESA WorldCover')
    ax.set_ylabel('Forest Cover (ha)')
    ax.set_title(f"({country}) Forest Cover")
    ax.set_yticklabels([f"{int(x)} million" for x in ax.get_yticks()])
    ax.grid(True, linestyle='--', alpha=0.5)

# Plot each country
plot_forest_cover(axes[0], 'a) Paraguay', paraguay_gfw, paraguay_palsar, paraguay_dw, paraguay_esa)
plot_forest_cover(axes[1], 'b) Zambia', zambia_gfw, zambia_palsar, zambia_dw, zambia_esa)
plot_forest_cover(axes[2], 'c) Zimbabwe', zimbabwe_gfw, zimbabwe_palsar, zimbabwe_dw, zimbabwe_esa)

axes[-1].set_xlabel('Year')

axes[0].legend(loc='upper center', bbox_to_anchor=(0.5, 1.25), ncol=4)

plt.tight_layout()
plt.show()

# 8. Checking the Planet- NICFI points for PARAGUAY + ZAMBIA + ZIMBABWE 
#  A Key aspect of the Accuracy Assessment

import zipfile
from pathlib import Path
import geopandas as gpd

def read_shp_from_zip(zip_path_str, preference_keywords=None):
    """
    Reads a shapefile from a ZIP using GDAL /vsizip/.
    - If multiple .shp exist, optionally chooses one using keywords.
    - Otherwise picks the first .shp found.
    """
    zip_path = Path(zip_path_str)
    if not zip_path.exists():
        raise FileNotFoundError(f"ZIP not found: {zip_path}")

    with zipfile.ZipFile(zip_path, "r") as z:
        shp_files = [n for n in z.namelist() if n.lower().endswith(".shp")]
        if not shp_files:
            raise FileNotFoundError(f"No .shp found inside ZIP: {zip_path.name}")

        print(f"\n{zip_path.name} -> SHP candidates:")
        for s in shp_files:
            print("  ", s)

        chosen = None
        if preference_keywords:
            lower_files = [(n, n.lower()) for n in shp_files]
            for kw in preference_keywords:
                kw = kw.lower()
                for n, nl in lower_files:
                    if kw in nl:
                        chosen = n
                        break
                if chosen:
                    break

        shp_in_zip = chosen if chosen else shp_files[0]
        print("Chosen SHP:", shp_in_zip)

    vsi_path = f"/vsizip/{zip_path.as_posix()}/{shp_in_zip}"
    print("Reading:", vsi_path)
    return gpd.read_file(vsi_path)

def summarize_points(points_gdf, country_name, class_field="id"):
    print(f"\n===== {country_name.upper()} NICFI POINTS =====")
    print(f"Number of features: {len(points_gdf)}")
    print("First feature:")
    print(points_gdf.iloc[0])

    if class_field not in points_gdf.columns:
        print(f"\n⚠️ Column '{class_field}' not found. Available columns: {list(points_gdf.columns)}")
        return

    class_counts = points_gdf[class_field].value_counts(dropna=False)

    print("\nClass Mapping: 0 = Non-Forest, 1 = Forest")
    print("Class Distribution in the Validation Dataset:")
    for cls, count in class_counts.items():
        if cls == 0:
            label = "Non-Forest"
        elif cls == 1:
            label = "Forest"
        else:
            label = "Other/Unknown"
        print(f"Class {cls} ({label}): {count} points")

# ---- ZIP paths (your PC) ----
paraguay_zip = r"C:\Users\gapnat\Downloads\ParaguayNICFIPoints.zip"
zambia_zip   = r"C:\Users\gapnat\Downloads\Zambia_NICFI_Points.zip"
zimbabwe_zip = r"C:\Users\gapnat\Downloads\ZimNICFIPoints.zip"

# ---- Read points (keywords help pick the right shp if there are multiple) ----
paraguay_points = read_shp_from_zip(paraguay_zip, preference_keywords=["paraguay", "nicfi", "points"])
zambia_points   = read_shp_from_zip(zambia_zip,   preference_keywords=["zambia", "nicfi", "points"])
zimbabwe_points = read_shp_from_zip(zimbabwe_zip, preference_keywords=["zim", "zimbabwe", "nicfi", "points"])

# ---- Summaries ----
summarize_points(paraguay_points, "Paraguay")
summarize_points(zambia_points, "Zambia")
summarize_points(zimbabwe_points, "Zimbabwe")

#8. Accuracy Assessment for Paraguay, Zambia and Zimbabwe

import ee
import geopandas as gpd
import geemap
import pandas as pd
from sklearn.metrics import roc_auc_score
import zipfile
from pathlib import Path

# Authenticate and Initialize Earth Engine
ee.Authenticate()
ee.Initialize()

# ----------------------------
# Robust SHP reader from ZIP (no unzip + no guessing internal file name)
# ----------------------------
def read_shp_from_zip(zip_path_str, preference_keywords=None):
    zip_path = Path(zip_path_str)
    if not zip_path.exists():
        raise FileNotFoundError(f"ZIP not found: {zip_path}")

    with zipfile.ZipFile(zip_path, "r") as z:
        shp_files = [n for n in z.namelist() if n.lower().endswith(".shp")]
        if not shp_files:
            raise FileNotFoundError(f"No .shp found inside ZIP: {zip_path.name}")

        chosen = None
        if preference_keywords:
            lower_files = [(n, n.lower()) for n in shp_files]
            for kw in preference_keywords:
                kw = kw.lower()
                for n, nl in lower_files:
                    if kw in nl:
                        chosen = n
                        break
                if chosen:
                    break

        shp_in_zip = chosen if chosen else shp_files[0]

    vsi_path = f"/vsizip/{zip_path.as_posix()}/{shp_in_zip}"
    return gpd.read_file(vsi_path)

def ensure_wgs84(gdf):
    if gdf.crs is None:
        gdf = gdf.set_crs("EPSG:4326")
    if gdf.crs.to_string().upper() != "EPSG:4326":
        gdf = gdf.to_crs("EPSG:4326")
    return gdf

# ----------------------------
# Load data for a given country (UPDATED PATHS + robust ZIP reading)
# ----------------------------
def load_country_data(country_name):
    base_path = r"C:\Users\gapnat\Downloads"

    if country_name == "Paraguay":
        aoi_zip = Path(base_path) / "ParaguayShape.zip"
        pts_zip = Path(base_path) / "ParaguayNICFIPoints.zip"
        aoi_kw = ["paraguay", "adm0", "_0", "pry"]
        pts_kw = ["paraguay", "nicfi", "point"]
    elif country_name == "Zambia":
        aoi_zip = Path(base_path) / "ZambiaShape.zip"
        pts_zip = Path(base_path) / "Zambia_NICFI_Points.zip"
        aoi_kw = ["zambia", "adm0", "_0", "zmb"]
        pts_kw = ["zambia", "nicfi", "point"]
    elif country_name == "Zimbabwe":
        aoi_zip = Path(base_path) / "ZimbabweShape.zip"
        pts_zip = Path(base_path) / "ZimNICFIPoints.zip"
        aoi_kw = ["zimbabwe", "adm0", "_0", "zwe"]
        pts_kw = ["zim", "zimbabwe", "nicfi", "point"]
    else:
        raise ValueError("Unsupported country.")

    aoi_gdf = ensure_wgs84(read_shp_from_zip(aoi_zip, aoi_kw))
    pts_gdf = ensure_wgs84(read_shp_from_zip(pts_zip, pts_kw))

    return geemap.gdf_to_ee(aoi_gdf), geemap.gdf_to_ee(pts_gdf), pts_gdf

# ----------------------------
# Process each dataset (UPDATED to match your earlier rules)
# ----------------------------
def process_dataset(dataset_name, aoi_fc, actual_points_fc, pts_gdf, country_name, results):
    resolution = None

    if dataset_name == "DynamicWorld":
        image = (ee.ImageCollection('GOOGLE/DYNAMICWORLD/V1')
                 .filterBounds(aoi_fc)
                 .filterDate('2020-01-01', '2020-12-31')
                 .select(['trees']).mean().clip(aoi_fc))
        # MATCH YOUR OLD WORKFLOW:
        classification_image = image.gt(0.3).rename('classification')  # 0.3 threshold
        resolution = 10

    elif dataset_name == "PALSAR":
        palsar = (ee.ImageCollection("JAXA/ALOS/PALSAR/YEARLY/FNF4")
                  .filterBounds(aoi_fc)
                  .filterDate('2020-01-01', '2020-12-31')
                  .mosaic().clip(aoi_fc))
        classification_image = palsar.expression(
            'fnf == 1 || fnf == 2 ? 1 : 0',
            {'fnf': palsar.select('fnf')}
        ).rename('classification')
        resolution = 25

    elif dataset_name == "GFW":
        hansen = ee.Image('UMD/hansen/global_forest_change_2022_v1_10')
        treecover2000 = hansen.select('treecover2000').clip(aoi_fc)
        lossyear = hansen.select('lossyear').clip(aoi_fc)

        # MATCH YOUR TABLE WORKFLOW:
        # remove ONLY 2020 loss (lossyear == 20), set those pixels to 0
        treecover_after_2020_loss = treecover2000.where(lossyear.eq(20), 0)

        classification_image = treecover_after_2020_loss.gt(10).rename('classification')
        resolution = 30

    elif dataset_name == "ESA":
        image = (ee.ImageCollection('ESA/WorldCover/v100')
                 .filterDate('2020-01-01', '2020-12-31')
                 .first()
                 .select('Map')
                 .clip(aoi_fc))
        classification_image = image.eq(10).rename('classification')
        resolution = 10

    else:
        raise ValueError(f"Unknown dataset: {dataset_name}")

    # Ensure every sampled point gets a value (keeps Sum=500)
    classification_image = ee.Image(classification_image).unmask(0)

    # Sample image using point labels
    samples = (classification_image.sampleRegions(
        collection=actual_points_fc,
        properties=['id'],
        scale=resolution,
        geometries=False
    )
    .map(lambda f: f.set('classification', ee.Number(f.get('classification')).toInt())))

    # Confusion matrix in EE
    cm = samples.errorMatrix('id', 'classification')
    matrix = cm.getInfo()

    # Safely extract TN, FP, FN, TP from 2x2 matrix
    TN = matrix[0][0] if len(matrix) > 0 and len(matrix[0]) > 0 else 0
    FP = matrix[0][1] if len(matrix) > 0 and len(matrix[0]) > 1 else 0
    FN = matrix[1][0] if len(matrix) > 1 and len(matrix[1]) > 0 else 0
    TP = matrix[1][1] if len(matrix) > 1 and len(matrix[1]) > 1 else 0

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

    # AUC (binary AUC is coarse; still computed)
    sample_info = samples.getInfo()
    y_true = []
    y_pred = []
    for feature in sample_info['features']:
        props = feature['properties']
        y_true.append(int(props['id']))
        y_pred.append(int(props['classification']))

    try:
        auc_score = roc_auc_score(y_true, y_pred)
    except:
        auc_score = 0

    results.append({
        "Country": country_name,
        "Dataset": dataset_name,
        "TP": TP,
        "TN": TN,
        "FP": FP,
        "FN": FN,
        "Sum": TP + TN + FP + FN,
        "Precision": precision,
        "Recall": recall,
        "F1 Score": f1,
        "Accuracy": accuracy,
        "Kappa": kappa,
        "RMSE": rmse,
        "AUC": auc_score
    })

def run_all_countries():
    results = []
    countries = ["Paraguay", "Zambia", "Zimbabwe"]
    datasets = ["GFW", "DynamicWorld", "PALSAR", "ESA"]  # keep names consistent with logic

    for country in countries:
        aoi_fc, pts_fc, pts_gdf = load_country_data(country)
        for dataset in datasets:
            print(f"Processing {dataset} for {country}...")
            process_dataset(dataset, aoi_fc, pts_fc, pts_gdf, country, results)

    df = pd.DataFrame(results).sort_values(by=["Country", "Dataset"])
    print("\n=== Combined Classification Results ===\n")
    print(df.to_string(index=False))

    df.to_csv("combined_classification_results.csv", index=False)
    print("\n✅ Results saved to 'combined_classification_results.csv'")

run_all_countries()

import pandas as pd
import seaborn as sns
import matplotlib.pyplot as plt

# Final metrics from your classification results
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

# Create heatmaps per country
fig, axes = plt.subplots(3, 1, figsize=(8, 16))
countries = ["Paraguay", "Zambia", "Zimbabwe"]
titles = [
    "(a) Heatmap of Accuracy Metrics: Paraguay",
    "(b) Heatmap of Accuracy Metrics: Zambia",
    "(c) Heatmap of Accuracy Metrics: Zimbabwe"
]

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
