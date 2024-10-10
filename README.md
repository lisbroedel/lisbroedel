- 👋 Hi, I’m @lisbroedel
- 👀 I’m interested in ...
- 🌱 I’m currently learning ...
- 💞️ I’m looking to collaborate on ...
- 📫 How to reach me ...
- 😄 Pronouns: ...
- ⚡ Fun fact: ...

<!---
lisbroedel/lisbroedel is a ✨ special ✨ repository because its `README.md` (this file) appears on your GitHub profile.
You can click the Preview link to take a look at your changes.
--->from pathlib import Path
import geopandas
import numpy as np
import xarray as  xr
from geocube.api.core import make_geocube

# Path to the .nc file
CFS_NC = 'C:/Users/lisbr/Documents/Cemaden/Merge/CFS.LaggedEnsembleAccumulated.LastSixICns.202410.1x1.sa.nc'

def make_mask(shapefile, dataarray):
    """Creates the mask for the basin."""
    shp = geopandas.read_file(shapefile).to_crs("EPSG:4326")
    shp["mask"] = 1
    geocube = make_geocube(
        vector_data=shp, measurements=["mask"], like=dataarray, fill=0
    )
    return (("lat", "lon"), geocube.mask.data)

def extract_mean_value(dataarray, shapefile):
    """Extract the mean precipitation value for the basin for a single time point."""
    basin_name = shapefile.stem
    
    # Extract the 'prate' variable for the single time step (time=0) and apply the mask
    masked_precipitation = dataarray['prate'].isel(time=0).where(dataarray[basin_name])
    
    # Calculate the mean value across the spatial dimensions (lat, lon)
    mean_value = masked_precipitation.mean(dim=("lat", "lon"))
    mean_value_np = mean_value.values

    # Debugging: Check for NaN values in the mean
    if np.isnan(mean_value_np):
        print(f"Warning: Mean value is NaN for {basin_name}. Check the mask and data alignment.")
        return
    
    # Save the mean value to a text file
    np.savetxt(f"{basin_name}-mean-value.txt", [mean_value_np])  # Single value array for savetxt
    print(f"File saved: {basin_name}-mean-value.txt")

    return mean_value_np
          
def main(shapefile):
    """Process one basin and extract the mean precipitation value."""
    basin_name = shapefile.stem

    with xr.open_dataset(CFS_NC).rio.write_crs("EPSG:4326") as cfs:
        cfs.coords["lon"] = (cfs.coords["lon"] + 180) % 360 - 180
        # print(cfs)
        cfs.coords[basin_name] = make_mask(shapefile, cfs)
        mean_value = extract_mean_value(cfs, shapefile)
        print(f"Mean value for {basin_name}: {mean_value}")        

if __name__ == "__main__":
    for basin_shapefile in Path("C:/Users/lisbr/Documents/Cemaden/Chirps/Basins_RI").glob("*.shp"):
        main(basin_shapefile)

