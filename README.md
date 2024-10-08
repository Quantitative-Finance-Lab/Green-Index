# Green Index Data Records   
<img src="https://img.shields.io/badge/Google Colab-F9ABOO?style=for-the-badge&logo=Google Colab&logoColor=white" link='https://colab.google/'> <img src="https://img.shields.io/badge/python-3776AB?style=for-the-badge&logo=python&logoColor=white">  
We introduce an indicator called 'green index', based on the google street view (GSV) images in Busan. 

To derive the green index, we collect GSV images, convert to HSV, calculate green index and implement spatial interpolation.   
<p align="center">
  <img src = "README_image/green index process.png" width = "30%"> <br>
  Figure 1. Steps to obtain green index.
</p>
        
This four-step process is necessary to effectively compute the green index, and for a detailed explanation, please refer to the [paper](https://doi.org/10.1038/s41598-023-49845-0), and sample data was stored in the *'DATA'* folder to replicate this calculation.   

Data in this repository consists of Excel and CSV files:   

- *Property Price and Green Index.xlsx*: Aggregated hedonic dataset of 52,644 observations with 27 variables
- *Calculated Greenness.csv*: Converted value data of GSV images in the *GSV IMAGE* folder
- *Data.csv*: Location of transaction sample data
- *Street Greenness.csv*: Calculated street greenness and its location
- *Green Index_Spatial Interpolation.csv*: Adjusted green index by implementing spatial interpolation
- *Roadside trees.xlsx*: Location information of roadside trees

## Image Preprocessing and Calculating Green Index
In order to calculate the green index, it is necessary to convert red, green, and blue color space to hue, satuation, and value color space.    
Street view image obtained from GSV download tool should contain latitude and longitude tokens in file name; thus, the saved image file name is  ‘_latitude_ _longitude_.jpg’.    
These two tokens are required for employing spatial interpolation method. So, the target property should also include the location information, i.e., latitude and longitude.   
After preprocessing, the green index is calculated as follows:   
$$Green \ index_{i} = pixel_{non-zero}/pixel_{total} * 100$$   

The following code is to perform above step:   
```python
import os
import pandas as pd
import numpy as np
import cv2
import matplotlib.image as mpimg
import warnings

warnings.filterwarnings('ignore')

# Setting upper and lower boundaries
lower_green = (40, 45, 30)
upper_green = (177, 177, 177)

green_indices = []

os.chdir('Your path')

for i, n in enumerate(os.listdir()):
  name = os.path.splitext(n)[0]
  print(name)
  lng, lat = name.split(sep=' ')
  img = mpimg.imread(n, cv2.IMREAD_COLOR)

  img_copy = img.copy()
  
  img_hsv = cv2.cvtColor(img, cv2.COLOR_BGR2HSV)
  img_mask = cv2.inRange(img_hsv, lower_green, upper_green)
  img_result = cv2.bitwise_and(img, img, mask=img_mask)

  nonmasked_index = np.where((img_result[:,:,0] != 0) & (img_result[:,:,1] != 0) & (img_result[:, :, 2] != 0))

  green_pixels = len(img_result[nonmasked_index])
  total_pixels = img_result.shape[0] * img_result.shape[1]

  #Calculate Green Index
  green_index = (green_pixels/total_pixels) * 100

  green_indices.append([lng, lat, green_index])

green_indices = pd.DataFrame(green_indices, columns = ['Longitude', 'Latitude', 'Green Index'])
green_indices.to_csv('Calculated Greenness.csv',index=False,encoding='utf-8-sig')
```   
From this step, we can obtain the street greenness in the view of pedestrian.     
It can be tested with images from the *'GSV IMAGE'* folder, and the resulting image is stored in the *'CONVERTED IMAGE'* folder.   

<p align="center">
  <img src = "/CONVERTED IMAGE/128.831857 35.090245.jpg" width = "100%"> <br>
  Figure 2. Visualization of street greenness in the view of pedestrian.
</p>

## Spatial Interpolation
Spatial interpolation step can be utilized to remedy the uneven spatial distribution of GSV images.   
To implement the spatial interpolation method, refer to the sample data file named *'Data.csv'* and *Street Greenness.csv*.    
The columns required to effectively manage the green index are as follows:   

*Data.csv*
- x: Longitude in the Cartesian coordinate system of the transaction point
- y: Latitude in the Cartesian coordinate system of the transaction point
   
*Street Greenness.csv*
- Longitude: Longitude of GSV image
- Latitude: Latitude of GSV image
- Green Index: Calculated street greenness

Spatial interpolation requires the distance between two objects based on longitude and latitude. It can be obtained by using haversine formula as follows:
$$d_{\text{haversine}} = 2 \times R \times \arcsin\left(\sqrt{\sin^2\left(\frac{\Delta \text{lat}}{2}\right) + \cos(\text{lat}_p) \cos(\text{lat}_g) \sin^2\left(\frac{\Delta \text{lng}}{2}\right)}\right)$$
   
<p align="center">
  <img src = "/README_image/spatial interpolation.png" width = "60%"> <br>
  Figure 3. Graphical description of spatial interpolation.
</p>   

The following code uses above mathematical form and aggregates the green index with 50 images closest to the transaction point. The final result file is in *Green Index_Spatial Interpolation.csv*.
```python
import pandas as pd
from haversine import haversine

data_df = pd.read_('Write your path\Data.csv')
green_df = pd.read_csv('Write your path\Street Greenness.csv')

Aggregated_Green_Index = []
Aggregated_Green_Index_Distance = []

num = 1
for y, x, ind in zip(data_df['y'], data_df['x'], data_df.index):
  distance = []

  for gr_y, gr_x, hgvi in zip(green_df['Latitude'], green_df['Longitude'], green_df['Green Index']):
    dis = haversine([y,x], [gr_y, gr_x], unit='km')
    distance.append([x,y,gr_x,gr_y,dis,hgvi])
  dis_df = pd.DataFrame(distance)
  dis_df.columns = ['x','y','gr_x','gr_y','distance','HGVI']
  dis_df = dis_df.sort_values('distance', ascending=True)

  # Extract the 50 nearest green indices
  dis_df_50 = dis_df.iloc[:50]

  mean_hgvi_50 = dis_df_50['HGVI'].mean()
  mean_dis_50 = dis_df_50['distance'].mean()

  Aggregated_Green_Index.append(mean_hgvi_50)
  Aggregated_Green_Index_Distance.append(mean_dis_50)

data_df['Green Index'] = Aggregated_Green_Index
data_df['Green Index_d'] = Aggregated_Green_Index_Distance
data_df.to_csv('Green Index_Spatial Interpolation.csv',index=False,encoding='utf-8-sig')
```
Through this process, we can get the green index for all points of transaction and all information of hedonic variables including green index is in *Hedonic Dataset.xlsx*.


## Visualization
Using *Property Price and Green Index.xlsx* and *Roadside trees.xlsx* files, we visulize the aggregated green index and the location of roadside trees. Each variable can be visualized through different layers, allowing potential readers to use various visualization methods supported by Pydeck library (https://pydeck.gl/) to visualize not only the green index but also other variables together.

The following code is visualization code using green index and location data.
```python
mapbox_key =   # Write your mapbox key of pydeck library

import pydeck as pdk
import mapboxgl
import pandas as pd
import csv
import json
from IPython.display import HTML
import colorsys

# Roadside trees
file_path_busan_street = 'Roadside trees.xlsx'
busan_street = pd.read_excel(file_path_busan_street)

data = []
for _, row in busan_street.iterrows():
    d = {'latitude': row[0], 'longitude': row[1]}
    data.append(d)

json_string = json.dumps(data, ensure_ascii=False, indent=2)
txt_file_path = 'data.txt'

with open(txt_file_path, 'w', encoding='utf-8') as f:
    f.write(json_string)

with open(txt_file_path, 'r', encoding='utf-8') as f:
    geo_street = json.load(f)

# Green index
file_path_busan_property = 'Property Price and Green Index.xlsx'
busan = pd.read_excel(file_path_busan_property)

data=[]
for _, row in busan.iterrows():
    d = {
        'latitude': row[2],  # latitude
        'longitude': row[1],  # longitude
        'properties': {
            'green index': row[12]  # green index
        }
    }
    data.append(d)

json_string = json.dumps(data, ensure_ascii=False, indent=2)
txt_file_path = 'busan_data.txt'

with open(txt_file_path, 'w', encoding='utf-8') as f:
    f.write(json_string)

with open(txt_file_path, 'r', encoding='utf-8') as f:
    geo = json.load(f)

busan_mini = busan[['Longitude', 'Latitude', 'Green Index']].copy()

# Calculate the maximum and minimum values of the green index to scale the data
max_index_value = max(float(item["properties"]["green index"]) for item in geo)
min_index_value = min(float(item["properties"]["green index"]) for item in geo)

pdk.settings.mapbox_key = mapbox_key  # mapbox API key

geo_street_transformed_2 = [
    {"longitude": float(item["longitude"]), "latitude": float(item["latitude"])}
    for item in geo_street
]

def minmax(value, min_value, max_value):
    return (value - min_value) / (max_value - min_value)

def calculate_color(item):
    index_value = float(item["properties"]["green index"])
    minmax_value = minmax(index_value, min_index_value, max_index_value)
    return [0, 255 * minmax_value, 255 * (1 - minmax_value), 255]

def calculate_elevation(item):
    index_value = float(item["properties"]["green index"])
    minmax_value = minmax(index_value, min_index_value, max_index_value)
    return minmax_value * 3000 

geo_transformed_2 = [
    {
        "longitude": float(item["longitude"]),
        "latitude": float(item["latitude"]),
        "color": calculate_color(item),
        "elevation": calculate_elevation(item)
    }
    for item in geo
]

elevation_values = [int(item['elevation']) for item in geo_transformed_2]
max_elevation = max(elevation_values)

color_values = [item['color'] for item in geo_transformed_2]

busan_mini['elevation'] = elevation_values
busan_mini['color'] = color_values

# The center coordinates when visualized
lon, lat = 129.0708802, 35.1153616

# Roadside trees visualization
layer11 = pdk.Layer(
    'ScatterplotLayer',
    geo_street_transformed_2,
    get_position='[longitude, latitude]',
    get_color='[255, 255, 255, 255]',
    get_radius=100
)

# Green index visualization
layer22 = pdk.Layer(
    'ColumnLayer',
    busan_mini,
    extruded=True,
    get_position='[Longitude,Latitude]',
    get_fill_color = 'color',
    # get_color = '[255,255,255]',
    get_elevation='elevation',
    elevation_scale=1,
    elevation_range=[0, max_elevation],
    pickable=True,
    auto_highlight=True,
    radius=100,
    opacity= 0.01
)

view_state = pdk.ViewState(
    longitude=lon,
    latitude=lat,
    zoom=11,
    pitch=70,
    bearing=-27.36
)

r = pdk.Deck(layers=[layer11, layer22], initial_view_state=view_state)

data_result = r.to_html('result.html', as_string=True)
```

Figure 4 illustrates the visualization results. White circles indicate the location of roadside trees retrieved from the Busan Open Data Portal (https://data.busan.go.kr/dataSet/detail.nm?contentId=10&publicdatapk=15040363). The cuboids present the level of greenness assigned to each property; thus they indicate the aggregated green index through spatial interpolation. The height of a cuboid denotes the degree of greenness, that is, the higher the degree of greenness, the higher the height of the cuboid.

<p align="center">
  <img src = "README_image/Visualization.png" width = "70%"> <br>
  Figure 4. Visualization of interpolated green indices and roadside trees in Busan.
</p>

