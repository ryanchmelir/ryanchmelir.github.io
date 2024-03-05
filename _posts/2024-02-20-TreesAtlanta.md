---
title: "City in a Forest: Trees Atlanta"
date: 2024-02-20 19:08:32 -0500
categories: [Analytics, Visualization]
tags: [sql, tableau, GIS, python, pandas, postgres, psql, docker]
image:
  path: https://e1.nmcdn.io/treesatl/wp-content/uploads/2019/08/treesatlanta-social-placeholder-2.jpg/v:1-width:1200-height:630-fit:cover/treesatlanta-social-placeholder-2.jpg?signature=7b26396d
  alt: Trees Atlanta Logo
---

An analysis of Trees Atanta's planting locations, survival rates, and tree types.

> <a href="https://public.tableau.com/views/CityinaForestTreesAtlanta/Dashboard?:language=en-US&:sid=&:display_count=n&:origin=viz_share_link" target="_blank">__Click here for an interactive Tableau dashboard!__</a>
{: .prompt-info }

## About

Trees Atlanta is a nonprofit community group with a wide range of programs in Atlanta centered around the protection, care, and improvement of Atlanta's urban forest.

You can learn more about the organization at their site [here](https://treesatlanta.org/), but for now here's a quick excerpt from their About page: "Founded in 1985, Trees Atlanta works to mitigate Atlanta’s tree loss, protect its forests, and increase its tree canopy. Empowered by a community of volunteers, Trees Atlanta serves the metro Atlanta area and has grown to become one of Atlanta’s most widely known and supported non-profit organizations."

While moving to Atlanta, I quickly fell in love with the city's abundance of trees, something I was not used to seeing in major cities. At the time, I was living on the eastside beltline, a location Trees Atlanta manages as an Arborterum. Noticing the Trees Atlanta work trucks, volunteers, and employees caring for plants along the beltline, I decided to begin volunteering for the organization in 2023.

After finding Tree Atlanta's [Tree Inventory](https://www.treesatlanta.org/resources/tree-inventory/), I wanted see if I could pull the data and run my own analysis on it.

## Data Preparation

### Gathering Data

Trees Atlanta uses ArcGIS, a geographic information system (GIS) maintained by Esri, to track trees they plant. Reading through their documentation led me to their [querying feature](https://developers.arcgis.com/rest/services-reference/enterprise/query-feature-service-layer-.htm). Using this, I was able to query their Feature Layer "[Trees Atlanta Plant Inventory](https://services6.arcgis.com/BZn8g8tzu5WfMokL/ArcGIS/rest/services/Trees_Atlanta_Plant_Inventory_Public_View/FeatureServer/0)" for the full dataset of 97,041 trees, along with a long list of fields:

> OBJECTID, PlantingSeason, Genus, Species, Cultivar, PlantedSize, ForestLayer, Neighborhood, SpecialProject, ParcelNUM, StreetPark, Quadrant, City, County, District, NPU, Utilities, GrowthSpace, Program, PlantType, PlantedDate, NotesaboutTree, Status, StatusDateUpdate, StatusComment, ReplacementNeeded, GlobalID, x, y

### Converting ESPG Projections

Thankfully, there wasn't too much necessary cleansing to be done. Knowing I would eventually map the trees in Tableau, I wanted to convert the x, y coordinates used by ArcGIS to the more commonly used Latitude, Longitude format. The data description included in the Feature Layer showed the coordinates were projected using the "Spatial Reference: 102100 (3857)". A quick google shows 3857 is an ESPG code in reference to a specific type of coordinate projection, and that the standard Lat, Long projection was code 4326.

This can be accomplished using PyProj's [Transformer function](https://pyproj4.github.io/pyproj/stable/api/transformer.html) and pandas' dataframe to apply the function over the dataset:

```python
from pyproj import Transformer
import pandas as pd
                                                
#reading file into pandas dataframe
df = pd.read_csv('treeinventory.csv')
                                            
#defining transformation function
def transform_function(row):
    transformer = Transformer.from_crs("EPSG:3857", "EPSG:4326")
    result = transformer.transform(row['x'],row['y'])
    return result
                                            
#applying the function to each row in the dataframe
df['LatLong'] = df.apply(transform_function, axis=1)
                                            
#writing the dataframe to a new CSV file
df.to_csv('projectedtreeinventory.csv', index=False)
```

### Creating a Container & Database

This section covers setting up the Postgres database within Docker. Feel free to scroll ahead to queries and results!

Copying the data to a docker container and creating a PSQL database and user:

```zsh
docker cp ./projectedtreeinventory.csv postgres-container:/projectedtreeinventory.csv
docker exec -it postgres-container psql -U postgres
```
```sql 
CREATE USER trees WITH PASSWORD '1234';
CREATE DATABASE treesdb WITH OWNER 'trees';
```
```zsh
docker exec -it postgres-container psql -U trees treesdb;
```
                                            
### Importing to Database

In this section I import the data into the database. I first copy the data to a staging table using all TEXT fields, and then insert the data into a final table casted as the correct datatypes.

```sql
CREATE TABLE treeInventoryStaging (
    OBJECTID TEXT, PlantingSeason TEXT, Genus TEXT, Species TEXT, Cultivar TEXT, PlantedSize TEXT, ForestLayer TEXT, Neighborhood TEXT, SpecialProject TEXT, ParcelNUM TEXT, StreetPark TEXT, Quadrant TEXT, City TEXT, County TEXT, District TEXT, NPU TEXT, Utilities TEXT, GrowthSpace TEXT, Program TEXT, PlantType TEXT, PlantedDate TEXT, NotesaboutTree TEXT, Status TEXT, StatusDateUpdate TEXT, StatusComment TEXT, ReplacementNeeded TEXT, GlobalID TEXT, x TEXT, y TEXT, Latitude TEXT, Longitude TEXT
);

--Copying the CSV into our staging table.
\copy treeInventoryStaging(OBJECTID,PlantingSeason,Genus,Species,Cultivar,PlantedSize,ForestLayer,Neighborhood,SpecialProject,ParcelNUM,StreetPark,Quadrant,City,County,District,NPU,Utilities,GrowthSpace,Program,PlantType,PlantedDate,NotesaboutTree,Status,StatusDateUpdate,StatusComment,ReplacementNeeded,GlobalID,x,y,Latitude,Longitude)
FROM 'projectedtreeinventory.csv' WITH DELIMITER ',' CSV;

--Creating final table with fixed datatypes.
--Most are kept as text. Changes include OBJECTID as an int primary key, and various date, integer, and numeric fields.
CREATE TABLE treeInventory (
    OBJECTID INTEGER PRIMARY KEY, PlantingSeason TEXT, Genus TEXT, Species TEXT, Cultivar TEXT, PlantedSize TEXT, ForestLayer TEXT, Neighborhood TEXT, SpecialProject TEXT, ParcelNUM SMALLINT, StreetPark TEXT, Quadrant TEXT, City TEXT, County TEXT, District TEXT, NPU TEXT, Utilities TEXT, GrowthSpace TEXT, Program TEXT, PlantType TEXT, PlantedDate DATE, NotesaboutTree TEXT, Status TEXT, StatusDateUpdate DATE, StatusComment TEXT, ReplacementNeeded TEXT, GlobalID TEXT, x NUMERIC, y NUMERIC, Latitude NUMERIC, Longitude NUMERIC
);
    
--Casting and inserting staged data into final table.
INSERT INTO treeInventory
    SELECT 
        CAST(OBJECTID AS INTEGER), PlantingSeason TEXT, Genus, Species, Cultivar, PlantedSize, ForestLayer, Neighborhood, SpecialProject TEXT, CAST(ParcelNUM AS SMALLINT), StreetPark, Quadrant, City, County, District, NPU, Utilities, GrowthSpace, Program, PlantType, TO_DATE(PlantedDate, 'MM/DD/YY'),  NotesaboutTree, Status, TO_DATE(StatusDateUpdate, 'MM/DD/YY'), StatusComment, ReplacementNeeded, GlobalID, CAST(x AS NUMERIC), CAST(y AS NUMERIC), CAST(Latitude AS NUMERIC), CAST(Longitude AS NUMERIC)
    FROM treeInventoryStaging;

--Confirming record count.
SELECT 
    COUNT(DISTINCT OBJECTID) AS treesPlanted
FROM treeInventory;

--Dropping staging table.
DROP TABLE treeInventoryStaging;
```

Alright! The data is cleaned and in our database.

## Analysis

### Tableau Queries

> <a href="https://public.tableau.com/views/CityinaForestTreesAtlanta/Dashboard?:language=en-US&:sid=&:display_count=n&:origin=viz_share_link" target="_blank">__Click here for an interactive Tableau dashboard!__</a>
{: .prompt-info }

The first thing I wanted to accomplish was creating a Tableau dashboard for identifying planting trends. After playing around with the dataset, I decided the planting trends I felt best represented visually would be those related to survival rates among genuses, amounts planted within decades, and planting locations. 

The first two SQL queries below, along with a query of planting coordinates and statuses, were used to build the dashboard, which you can interact with by following the link above.

#### Grouping by Decade Planted, Total Living & Dead

```sql
WITH InventoryWithDecades AS (
SELECT
    Genus,
    Status,
    PlantedDate,
    CONCAT(
        MIN(DATE_PART('YEAR', PlantedDate)) OVER(PARTITION BY DATE_PART('DECADE', PlantedDate)), --Minimum Year Present Within a Decade
        ' - ',
        MAX(DATE_PART('YEAR', PlantedDate)) OVER(PARTITION BY DATE_PART('DECADE', PlantedDate))  --Maximum Year Present Within a Decade
        ) AS DecadePlanted 
FROM treeinventory
WHERE PlantedDate IS NOT NULL
)

SELECT
    DecadePlanted,
    COUNT(*) FILTER(WHERE Status = 'Alive') AS AliveCount,
    COUNT(*) FILTER(WHERE Status LIKE '%Dead%' OR Status LIKE '%Not%Found%') AS DeadCount,
    COUNT(*) AS TotalPlanted
FROM InventoryWithDecades
GROUP BY DecadePlanted
ORDER BY DecadePlanted;
```

In the query above, I first set a CTE, 'InventoryWithDecades', whose purpose is to create a readable and variable column 'DecadePlanted'. This column uses window functions to partition by decade and pull in the first and last year currently present in the data within a decade.

For example, the below output shows 2020 - 2023 as the most recent decade, but once trees planted in 2024 are added to the dataset, the output will show 2020 - 2024.

Next, I pull in the DecadePlanted column, and generate counts of the number of alive and dead trees within each decade. The dataset includes four different statuses we are considering as dead: "Dead", "Vandalized Dead", "VandalizedDead", and "NotFound". We use the LIKE operator to group the different types of "dead" into one bucket. 

Out of the 97,041 trees in the dataset, only 67 are missing a status and excluded from our results here.

Query output:

| decadeplanted | alivecount | deadcount | totalplanted |
|---------------|------------|-----------|--------------|
| 1990 - 1999   |       1295 |       337 |         1632 |
| 2000 - 2009   |      13837 |      4183 |        18022 |
| 2010 - 2019   |      33866 |      7382 |        41304 |
| 2020 - 2023   |      29114 |      1865 |        30979 |

#### Breaking out the results by Genus

Next, we will create a similar table as above, but including groups by their genus as well as calculating the percentage that are alive.

The previous CTE is reused here to create the decade grouping.

```sql
WITH InventoryWithDecades AS (
SELECT
    Genus,
    Status,
    PlantedDate,
    CONCAT(
        MIN(DATE_PART('YEAR', PlantedDate)) OVER(PARTITION BY DATE_PART('DECADE', PlantedDate)), --Minimum Year Present Within a Decade
        ' - ',
        MAX(DATE_PART('YEAR', PlantedDate)) OVER(PARTITION BY DATE_PART('DECADE', PlantedDate))  --Maximum Year Present Within a Decade
        ) AS DecadePlanted 
FROM treeinventory
WHERE PlantedDate IS NOT NULL
)

SELECT
    Genus,
    DecadePlanted,
    COUNT(*) FILTER(WHERE Status = 'Alive') AS AliveCount,
    COUNT(*) FILTER(WHERE Status LIKE '%Dead%' OR Status LIKE '%Not%Found%') AS DeadCount,
    COUNT(*) AS TotalPlanted,
    ROUND(100*COALESCE(COUNT(*) FILTER(WHERE Status = 'Alive')::DECIMAL / NULLIF(COUNT(*),0),0),2) AS PercentAlive
FROM InventoryWithDecades
GROUP BY Genus, DecadePlanted
ORDER BY TotalPlanted DESC;
```

Limiting the results to the first five, here is the query output:

| genus         | decadeplanted | alivecount | deadcount | totalplanted | percentalive |
|---------------|---------------|------------|-----------|--------------|--------------|
| Quercus       | 2010 - 2019   |       4597 |       596 |         5214 |        88.17 |
| Quercus       | 2020 - 2023   |       4303 |       355 |         4658 |        92.38 |
| Lagerstroemia | 2010 - 2019   |       3721 |       432 |         4154 |        89.58 |
| Cercis        | 2010 - 2019   |       2672 |      1356 |         4028 |        66.34 |
| Quercus       | 2000 - 2009   |       2914 |       634 |         3549 |        82.11 |

> The queries above, along with one to pull the geographical data, were all needed to build the dashboard linked above.
{: .prompt-tip }

### More Analysis

#### Lifespans before Death

The query below shows us the average number of days before trees are updated as dead, grouped by genus, along with the percentage of those deaths that are caused by vandalization ('vandalized dead', 'vandalizeddead') vs unknown reasons ('dead', 'notfound').

```sql
SELECT
    Genus,
    COUNT(*) AS TotalDead,
    ROUND(AVG(StatusDateUpdate - PlantedDate), 0) AS AvgDaysAlive,
    ROUND(COUNT(*) FILTER(WHERE Status LIKE 'Vandalized%Dead')::DECIMAL
        / COUNT(*) FILTER(WHERE Status LIKE '%Dead%' OR Status LIKE '%Not%Found') * 100.0, 2) AS VandalizedPercent,
    ROUND(COUNT(*) FILTER(WHERE Status = 'Dead' OR Status LIKE '%NotFound%')::DECIMAL 
        / COUNT(*) FILTER(WHERE Status LIKE '%Dead%' OR Status LIKE '%Not%Found') * 100.0, 2) AS UnknownDeathPercent
FROM treeInventory
WHERE 
    Genus IN ( --Subquery to pull top fiveteen planted genuses
        SELECT Genus 
        FROM treeInventory 
        GROUP BY Genus 
        ORDER BY Count(Genus) DESC 
        LIMIT 15)
    AND StatusDateUpdate IS NOT NULL --Only including trees with status update dates for a more accurate average days alive
    AND Status LIKE '%Dead%' 
    OR Status LIKE '%Not%Found%'
GROUP BY Genus
HAVING COUNT(*) > 100 -- Filtering for genuses with more than 100 dead or not found trees
ORDER BY AvgDaysAlive DESC;
```

| genus         | totaldead | avgdaysalive | vandalizedpercent | unknowndeathpercent |
|---------------|-----------|--------------|-------------------|---------------------|
| Acer          |       921 |         3754 |             13.68 |               86.32 |
| Ulmus         |       313 |         3697 |             17.89 |               82.11 |
| Quercus       |      1341 |         3351 |             12.98 |               87.02 |
| Lagerstroemia |       686 |         3066 |             28.43 |               71.57 |
| Nyssa         |       464 |         2948 |             14.44 |               85.56 |
| Ilex          |       128 |         2841 |             15.63 |               84.38 |
| Carpinus      |       246 |         2800 |             16.67 |               83.33 |
| Parrotia      |       181 |         2746 |             16.57 |               83.43 |
| Chionanthus   |       305 |         2628 |             25.90 |               74.10 |
| Taxodium      |       166 |         2513 |             16.87 |               83.13 |
| Amelanchier   |       455 |         2406 |             13.41 |               86.59 |
| Cercis        |      1853 |         2336 |             17.16 |               82.84 |
| Cornus        |      1038 |         2316 |              8.86 |               91.14 |
| Liriodendron  |       207 |         2121 |              7.25 |               92.75 |
| Magnolia      |       747 |         1962 |             25.70 |               74.30 |

The results are limited to the top 15 genuses by planting volume with more than 100 dead trees. We can see Lagerstroemia is the most likely of that group to die by vandlization.

>  Magnolia has the lowest average lifespan before death, coupled with a high likelihood of vandlization.
{: .prompt-tip }

#### Seasons of Year & Days of Week Analysis

Here, we do an analysis on the day of week and time of year Trees Atlanta plants trees.

```sql
SELECT
	CASE 
		WHEN EXTRACT(MONTH FROM planteddate) BETWEEN 3 AND 5 THEN 'Spring' --Case statement groups the planted month within seasons
		WHEN EXTRACT(MONTH FROM planteddate) BETWEEN 6 AND 8 THEN 'Summer'
		WHEN EXTRACT(MONTH FROM planteddate) BETWEEN 9 AND 11 THEN 'Fall'
		ELSE 'Winter' --Winter is left last as it's the only season where the months reset (12, 1, 2).
	END AS Season,
	TO_CHAR(PlantedDate,'Day') AS PlantedDay, --Pulling the day of the week in text from the planted date.
	COUNT(*)
FROM treeinventory
WHERE PlantedDate IS NOT NULL
GROUP BY Season, PlantedDay;
```

The results look something like this (limiting to the first couple of rows):

| season | plantedday | count |
|--------|------------|-------|
| Spring | Monday     |  1027 |
| Fall   | Tuesday    |  1327 |

> Visualizing the results below, we can see Trees Atlanta primarily plants in the Winter, and as a volunteer organization, plants mostly on Saturdays!
{: .prompt-tip }

![Desktop View](/assets/img/projects/TreeAtlanta/dayofweekdistlight.png){: .light width="5532" height="3226" w-25}
![Desktop View](/assets/img/projects/TreeAtlanta/dayofweekdistdark.png){: .dark width="5532" height="3226" w-25}

#### Neighborhood Plantings - Bar Chart Race

Finally, we'll take a look at plantings by neighborhood since 1996. To start, we'll need to pull a running total of trees planted grouped by neighborhood and ordered by year.

```sql
SELECT
	EXTRACT('YEAR' FROM PlantedDate) AS PlantedYear,
	Neighborhood,
	SUM(COUNT(*)) OVER(PARTITION BY Neighborhood ORDER BY EXTRACT('YEAR' FROM PlantedDate)) AS PlantedAmount --Running total of plantings, paritioned by neighborhood and ordered by year planted.
FROM treeinventory
WHERE 
	PlantedDate IS NOT NULL
	AND Neighborhood IS NOT NULL
GROUP BY PlantedYear, Neighborhood;
```

This data is then fed into a python library, [bar_chart_race](https://www.dexplo.org/bar_chart_race/). This library reformats our data from it's current long format into a wide format and then creates a bar chart race.

```python
import bar_chart_race as bcr
import pandas as pd

df = pd.read_csv('LongDataYears.csv')

df_values, df_ranks = bcr.prepare_long_data(df, index='plantedyear', columns='neighborhood',
                                            values='plantedamount', steps_per_period=10, 
                                            orientation='h', sort='desc')

bcr.bar_chart_race(
    df=df_values,
    filename='NeighborhoodBarRace.mp4',
    orientation='h',
    sort='desc',
    n_bars=10,
    fixed_order=False,
    fixed_max=False,
    steps_per_period=8,
    interpolate_period=False,
    label_bars=True,
    bar_size=.95,
    period_label={'x': .99, 'y': .1, 'ha': 'right'},
    period_length=300,
    figsize=(6, 3.5),
    dpi=288,
    cmap='tab20',
    title='Tree Planting by Neighborhood',
    title_size='',
    bar_label_size=7,
    tick_label_size=7,
    shared_fontdict={'family' : 'Helvetica', 'color' : '.1'},
    scale='linear',
    writer=None,
    fig=None,
    bar_kwargs={'alpha': .7},
    filter_column_colors=True)
```

{% include embed/youtube.html id='e8IrdrLMXx8' %}

> Take a look at the rapid growth of the Beltline arborterum beginning in 2017 (starts at 0:59). The increase in planting coincides with the [opening of the Beltline's Westside Trail and Eastside Trail Extension](https://beltline.org/2017/02/22/2017-the-biggest-year-yet-for-the-atlanta-beltline/)!
{: .prompt-tip }