ERA5 Data Extraction and Processing
This repository provides a comprehensive Python script for extracting, processing, and analyzing ERA5 hourly reanalysis data. It demonstrates how to load GRIB files, filter specific meteorological variables, perform necessary unit conversions, merge datasets, visualize time series, and efficiently save the processed data into the cloud-optimized Zarr format.

Features
GRIB File Loading: Seamlessly load ERA5 data from GRIB files using xarray with the cfgrib engine.

Variable Filtering: Select specific variables (e.g., 2-meter temperature, total precipitation) from multi-message GRIB files.

Data Cleaning: Handle potential coordinate conflicts and drop unnecessary variables.

Unit Conversions: Convert raw data units (e.g., Kelvin to Celsius, meters of precipitation to millimeters).

Dataset Merging: Combine different meteorological variables into a single, cohesive xarray.Dataset.

Time Series Visualization: Generate plots to visualize trends and patterns of extracted data over time.

Zarr Export: Save processed data to the Zarr format for efficient, chunked, and compressed storage, ideal for large datasets and cloud-based analysis.

Installation
To get started, clone this repository and set up your Python environment.

Clone the repository:

git clone https://github.com/your-username/ERA5-Data-Processing.git
cd ERA5-Data-Processing

Create a virtual environment (recommended):

python -m venv venv
source venv/bin/activate  # On Windows: `venv\Scripts\activate`

Install the required libraries:

pip install xarray cfgrib matplotlib folium numcodecs zarr

cfgrib dependency: cfgrib requires the ECMWF ecCodes library. Installation can be complex depending on your OS. For more detailed instructions, refer to the cfgrib documentation. On some systems, conda install cfgrib might be easier if you use Anaconda/Miniconda.

Usage
This script is designed to be run in a Jupyter Notebook or as a Python script. Ensure you have your ERA5 GRIB file (e.g., 22e7a5177aa05425bfc8e4399e9a4fc.grib) in the same directory or provide the correct path.

1. Load the GRIB File
The first step involves loading your ERA5 GRIB file using xarray and the cfgrib engine. Error handling is included to catch common issues.

import xarray as xr
import cfgrib

file_path = "22e7a5177aa05425bfc8e4399e9a4fc.grib"

try:
    ds = xr.open_dataset(file_path, engine='cfgrib')
    print("GRIB File successfully loaded ✅")
except Exception as e:
    print(f"Failed to load GRIB file: {e}")
    raise

# Print basic info and variables
print(ds)
print("Variables in dataset:", list(ds.data_vars))

2. Isolate and Process Specific Variables
GRIB files can contain multiple messages, sometimes leading to coordinate conflicts (e.g., for time). To handle this, we load specific variables (2t for 2-meter temperature and tp for total precipitation) separately and then merge them. This section also performs essential unit conversions.

# Load temperature
ds_t2m = xr.open_dataset(
    file_path,
    engine="cfgrib",
    backend_kwargs={"filter_by_keys": {"shortName": "2t"}}
)

# Load precipitation
ds_tp = xr.open_dataset(
    file_path,
    engine="cfgrib",
    backend_kwargs={"filter_by_keys": {"shortName": "tp"}}
)

# Drop 'valid_time' if present to avoid merge conflicts
if 'valid_time' in ds_t2m.variables:
    ds_t2m = ds_t2m.drop_vars('valid_time')
if 'valid_time' in ds_tp.variables:
    ds_tp = ds_tp.drop_vars('valid_time')

# Rename variables for clarity
ds_t2m = ds_t2m.rename({'t2m': 'temperature_2m'})
ds_tp = ds_tp.rename({'tp': 'total_precipitation'})

# Unit conversions: Kelvin to Celsius, meters to millimeters
ds_t2m['temperature_2m'] = ds_t2m['temperature_2m'] - 273.15
ds_t2m['temperature_2m'].attrs['units'] = '°C'

if ds_tp['total_precipitation'].attrs.get('units', '') == 'm':
    ds_tp['total_precipitation'] *= 1000
    ds_tp['total_precipitation'].attrs['units'] = 'mm'

# Merge the processed datasets
ds_combined = xr.merge([ds_t2m, ds_tp], compat='override')

# Collapse the 'step' dimension for total precipitation
precip_total = ds_combined['total_precipitation'].sum(dim='step', skipna=True)
ds_combined['precip_collapsed'] = precip_total
ds_combined['precip_collapsed'].attrs['long_name'] = "Total Precipitation (Collapsed over step)"
ds_combined['precip_collapsed'].attrs['units'] = ds_combined['total_precipitation'].attrs.get('units', 'mm')

print(ds_combined)

3. Visualize Data (Time Series Plotting)
Generate plots to visualize the time series of temperature and precipitation for the extracted location.

import matplotlib.pyplot as plt

def plot_time_series(var_name):
    data = ds_combined[var_name].squeeze()  # remove lat/lon dims for single point
    plt.figure(figsize=(15, 4))
    plt.plot(data['time'], data, label=ds_combined[var_name].attrs.get('long_name', var_name), color='tab:blue')
    plt.xlabel("Time")
    plt.ylabel(f"{ds_combined[var_name].attrs.get('units', '')}")
    plt.title(f"Time Series of {var_name}")
    plt.grid(True)
    plt.legend()
    plt.tight_layout()
    plt.show()

plot_time_series("temperature_2m")
plot_time_series("precip_collapsed")

4. Save to Zarr Format
Saving your processed data to Zarr is highly recommended for large datasets. Zarr is a cloud-optimized format that allows for efficient reading of data subsets without loading the entire dataset into memory.

import zarr
import os
from numcodecs import Blosc

zarr_path = "2_Temp_Precip_dynamic_data_2.zarr"

# Select variables to save
ds_zarr = ds_combined[["temperature_2m", "precip_collapsed"]]

# Define chunking strategy
chunks = {"time": 1000}

# Define compression using Blosc
compressor = Blosc(cname='zstd', clevel=3, shuffle=Blosc.SHUFFLE)
encoding = {
    var: {"compressor": compressor}
    for var in ds_zarr.data_vars
}

# Remove existing Zarr store if it exists
if os.path.exists(zarr_path):
    import shutil
    shutil.rmtree(zarr_path)

# Save to Zarr with specified chunking and compression
ds_zarr.chunk(chunks).to_zarr(
    zarr_path,
    mode='w',
    consolidated=True,
    encoding=encoding,
    zarr_format=2
)

print(f"✅ Zarr dataset successfully saved to: {zarr_path}")

5. (Optional) Plot Location on a Map
If your dataset contains latitude and longitude, you can easily plot the point on an interactive map using folium.

import folium

# Extract the single point location from your dataset
lat = float(ds.latitude.values)
lon = float(ds.longitude.values)

print(f"Plotting marker at Latitude: {lat}, Longitude: {lon}")

# Create a folium map centered at the point
m = folium.Map(location=[lat, lon], zoom_start=10)

# Add a marker
folium.Marker(
    location=[lat, lon],
    popup=f"ERA5 Point\nLat: {lat}, Lon: {lon}",
    tooltip="ERA5 Time Series Location 📍",
    icon=folium.Icon(color="blue", icon="info-sign")
).add_to(m)

# Display the map (will render inline in Jupyter)
m

The Importance and Utility of ERA5 Data
ERA5 is the fifth generation ECMWF reanalysis for the global climate and weather, providing a comprehensive and consistent record of atmospheric, land, and oceanic climate variables since 1940. It is a cornerstone for scientific research and practical applications due to its:

High Resolution: Offers hourly data at a fine spatial resolution (e.g., 0.25 degrees), enabling detailed analysis of atmospheric and surface processes.

Extensive Temporal Coverage: With data available back to 1940, it's invaluable for long-term climate trend analysis and historical event reconstruction.

Comprehensive Variable Set: Includes a vast array of atmospheric, land, and oceanic parameters, supporting multi-disciplinary studies.

Data Assimilation: As a reanalysis product, ERA5 combines observations with a sophisticated numerical weather prediction model using advanced data assimilation techniques, resulting in a globally complete and consistent dataset.

Accessibility: Publicly available through platforms like the Copernicus Climate Change Service (C3S), making it widely accessible to the research community.

Diverse Domains of Utilization:
ERA5 data is a crucial resource across a multitude of fields:

Climate Change Research: Essential for trend analysis of climate indicators, validation of climate models, and attribution studies to understand the causes of observed climate changes.

Hydrology and Water Resources Management: Drives hydrological models for river flow simulation, drought monitoring, and reservoir management optimization.

Agriculture and Food Security: Used for crop yield prediction, irrigation scheduling, and understanding the impact of weather on pest and disease outbreaks.

Renewable Energy: Critical for wind and solar energy assessment, including site selection and forecasting power generation.

Disaster Risk Reduction: Provides crucial inputs for flood forecasting, storm surge modeling, and wildfire risk assessment.

Atmospheric Sciences and Meteorology: Supports research into atmospheric phenomena, extreme weather events, and numerical weather prediction model improvements.

Oceanography: Offers atmospheric forcing for ocean circulation models and helps in understanding sea ice dynamics.

Public Health: Enables heatwave impact assessment and disease vector modeling by linking weather conditions to health outcomes.

By leveraging ERA5 data, researchers and practitioners can gain deep insights into Earth's climate system and develop solutions for critical environmental and societal challenges.

Contact
For any questions or collaborations, feel free to reach out:

Name: Sayantan Mandal

Email: sayantanonfire@gmail.com