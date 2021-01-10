# Geomarketing {#location}

## Prerequisites {-}

- This chapter requires the following packages (**revgeo** must also be installed):


```r
library(sf)
library(dplyr)
library(purrr)
library(raster)
library(osmdata)
library(spDataLarge)
```

- Required data, that will be downloaded in due course

As a convenience to the reader and to ensure easy reproducibility, we have made available the downloaded data in the **spDataLarge** package.

## Introduction

This chapter demonstrates how the skills learned in Parts I and II can be applied to a particular domain: geomarketing\index{geomarketing} (sometimes also referred to as location analysis\index{location analysis} or location intelligence).
This is a broad field of research and commercial application.
A typical example is where to locate a new shop.
The aim here is to attract most visitors and, ultimately, make the most profit.
There are also many non-commercial applications that can use the technique for public benefit, for example where to locate new health services [@tomintz_geography_2008].

People are fundamental to location analysis\index{location analysis}, in particular where they are likely to spend their time and other resources.
Interestingly, ecological concepts and models are quite similar to those used for store location analysis.
Animals and plants can best meet their needs in certain 'optimal' locations, based on variables that change over space (@muenchow_review_2018; see also chapter \@ref(eco)).
This is one of the great strengths of geocomputation and GIScience in general.
Concepts and methods are transferable to other fields.
<!-- add reference!! -->
Polar bears, for example, prefer northern latitudes where temperatures are lower and food (seals and sea lions) is plentiful.
Similarly, humans tend to congregate in certain places, creating economic niches (and high land prices) analogous to the ecological niche of the Arctic.
The main task of location analysis is to find out where such 'optimal locations' are for specific services, based on available data.
Typical research questions include:

- Where do target groups live and which areas do they frequent?
- Where are competing stores or services located?
- How many people can easily reach specific stores?
- Do existing services over- or under-exploit the market potential?
- What is the market share of a company in a specific area?

This chapter demonstrates how geocomputation can answer such questions based on a hypothetical case study based on real data.

## Case study: bike shops in Germany {#case-study}

Imagine you are starting a chain of bike shops in Germany.
The stores should be placed in urban areas with as many potential customers as possible.
Additionally, a hypothetical survey (invented for this chapter, not for commercial use!) suggests that single young males (aged 20 to 40) are most likely to buy your products: this is the *target audience*.
You are in the lucky position to have sufficient capital to open a number of shops.
But where should they be placed?
Consulting companies (employing geomarketing\index{geomarketing} analysts) would happily charge high rates to answer such questions.
Luckily, we can do so ourselves with the help of open data\index{open data} and open source software\index{open source software}.
The following sections will demonstrate how the techniques learned during the first chapters of the book can be applied to undertake common steps in service location analysis:

- Tidy the input data from the German census (Section \@ref(tidy-the-input-data))
- Convert the tabulated census data into raster\index{raster} objects (Section \@ref(create-census-rasters))
- Identify metropolitan areas with high population densities (Section \@ref(define-metropolitan-areas))
- Download detailed geographic data (from OpenStreetMap\index{OpenStreetMap}, with **osmdata**\index{osmdata (package)}) for these areas (Section \@ref(points-of-interest))
- Create rasters\index{raster} for scoring the relative desirability of different locations using map algebra\index{map algebra} (Section \@ref(identifying-suitable-locations))

Although we have applied these steps to a specific case study, they could be generalized to many scenarios of store location or public service provision.

## Tidy the input data

The German government provides gridded census data at either 1 km or 100 m resolution.
The following code chunk downloads, unzips and reads in the 1 km data.
Please note that `census_de` is also available from the **spDataLarge** package (`data("census_de", package = "spDataLarge"`).



```r
download.file("https://tinyurl.com/ybtpkwxz", 
              destfile = "census.zip", mode = "wb")
unzip("census.zip") # unzip the files
census_de = readr::read_csv2(list.files(pattern = "Gitter.csv"))
```

The `census_de` object is a data frame containing 13 variables for more than 300,000 grid cells across Germany.
For our work, we only need a subset of these: Easting (`x`) and Northing (`y`), number of inhabitants (population; `pop`), mean average age (`mean_age`), proportion of women (`women`) and average household size (`hh_size`).
These variables are selected and renamed from German into English in the code chunk below and summarized in Table \@ref(tab:census-desc). 
Further, `mutate_all()` is used to convert values -1 and -9 (meaning unknown) to `NA`.


```r
# pop = population, hh_size = household size
input = dplyr::select(census_de, x = x_mp_1km, y = y_mp_1km, pop = Einwohner,
                      women = Frauen_A, mean_age = Alter_D,
                      hh_size = HHGroesse_D)
# set -1 and -9 to NA
input_tidy = mutate_all(input, list(~ifelse(. %in% c(-1, -9), NA, .)))
```



Table: (\#tab:census-desc)Categories for each variable in census data from Datensatzbeschreibung...xlsx located in the downloaded file census.zip (see Figure 13.1 for their spatial distribution).

| class | Population | % female | Mean age | Household size |
|:-----:|:----------:|:--------:|:--------:|:--------------:|
|   1   |   3-250    |   0-40   |   0-40   |      1-2       |
|   2   |  250-500   |  40-47   |  40-42   |     2-2.5      |
|   3   |  500-2000  |  47-53   |  42-44   |     2.5-3      |
|   4   | 2000-4000  |  53-60   |  44-47   |     3-3.5      |
|   5   | 4000-8000  |   >60    |   >47    |      >3.5      |
|   6   |   >8000    |          |          |                |

## Create census rasters
 
After the preprocessing, the data can be converted into a raster stack\index{raster!stack} or brick\index{raster!brick} (see Sections \@ref(raster-classes) and \@ref(raster-subsetting)).
`rasterFromXYZ()` makes this really easy.
It requires an input data frame where the first two columns represent coordinates on a regular grid.
All the remaining columns (here: `pop`, `women`, `mean_age`, `hh_size`) will serve as input for the raster brick layers (Figure \@ref(fig:census-stack); see also `code/13-location-jm.R` in our github repository).


```r
input_ras = rasterFromXYZ(input_tidy, crs = st_crs(3035)$proj4string)
#> Warning in showSRID(uprojargs, format = "PROJ", multiline = "NO", prefer_proj =
#> prefer_proj): Discarded datum Unknown based on GRS80 ellipsoid in CRS definition
```


```r
input_ras
#> class : RasterBrick
#> dimensions : 868, 642, 557256, 4 (nrow, ncol, ncell, nlayers)
#> resolution : 1000, 1000 (x, y)
#> extent : 4031000, 4673000, 2684000, 3552000 (xmin, xmax, ymin, ymax)
#> coord. ref. : +proj=laea +lat_0=52 +lon_0=10
#> names       :  pop, women, mean_age, hh_size 
#> min values  :    1,     1,        1,       1 
#> max values  :    6,     5,        5,       5
```

\BeginKnitrBlock{rmdnote}<div class="rmdnote">Note that we are using an equal-area projection (EPSG:3035; Lambert Equal Area Europe), i.e., a projected CRS\index{CRS!projected} where each grid cell has the same area, here 1000 x 1000 square meters. 
Since we are using mainly densities such as the number of inhabitants or the portion of women per grid cell, it is of utmost importance that the area of each grid cell is the same to avoid 'comparing apples and oranges'.
Be careful with geographic CRS\index{CRS!geographic} where grid cell areas constantly decrease in poleward directions (see also Section \@ref(crs-intro) and Chapter \@ref(reproj-geo-data)). </div>\EndKnitrBlock{rmdnote}

<div class="figure" style="text-align: center">
<img src="figures/08_census_stack.png" alt="Gridded German census data of 2011 (see Table 13.1 for a description of the classes)." width="100%" />
<p class="caption">(\#fig:census-stack)Gridded German census data of 2011 (see Table 13.1 for a description of the classes).</p>
</div>

<!-- find out about new lines in headings + blank cells-->
The next stage is to reclassify the values of the rasters stored in `input_ras` in accordance with the survey mentioned in Section \@ref(case-study), using the **raster** function `reclassify()`, which was introduced in Section \@ref(local-operations)\index{map algebra!local operations}.
In the case of the population data, we convert the classes into a numeric data type using class means. 
Raster cells are assumed to have a population of 127 if they have a value of 1 (cells in 'class 1' contain between 3 and 250 inhabitants) and 375 if they have a value of 2 (containing 250 to 500 inhabitants), and so on (see Table \@ref(tab:census-desc)).
A cell value of 8000 inhabitants was chosen for 'class 6' because these cells contain more than 8000 people.
Of course, these are approximations of the true population, not precise values.^[
The potential error introduced during this reclassification stage will be explored in the exercises.
]
However, the level of detail is sufficient to delineate metropolitan areas (see next section).

In contrast to the `pop` variable, representing absolute estimates of the total population, the remaining variables were re-classified as weights corresponding with weights used in the survey.
Class 1 in the variable `women`, for instance, represents areas in which 0 to 40% of the population is female;
these are reclassified with a comparatively high weight of 3 because the target demographic is predominantly male.
Similarly, the classes containing the youngest people and highest proportion of single households are reclassified to have high weights.


```r
rcl_pop = matrix(c(1, 1, 127, 2, 2, 375, 3, 3, 1250, 
                   4, 4, 3000, 5, 5, 6000, 6, 6, 8000), 
                 ncol = 3, byrow = TRUE)
rcl_women = matrix(c(1, 1, 3, 2, 2, 2, 3, 3, 1, 4, 5, 0), 
                   ncol = 3, byrow = TRUE)
rcl_age = matrix(c(1, 1, 3, 2, 2, 0, 3, 5, 0),
                 ncol = 3, byrow = TRUE)
rcl_hh = rcl_women
rcl = list(rcl_pop, rcl_women, rcl_age, rcl_hh)
```

Note that we have made sure that the order of the reclassification matrices in the list is the same as for the elements of `input_ras`.
For instance, the first element corresponds in both cases to the population.
Subsequently, the `for`-loop\index{loop!for} applies the reclassification matrix to the corresponding raster layer.
Finally, the code chunk below ensures the `reclass` layers have the same name as the layers of `input_ras`.


```r
reclass = input_ras
for (i in seq_len(nlayers(reclass))) {
  reclass[[i]] = reclassify(x = reclass[[i]], rcl = rcl[[i]], right = NA)
}
names(reclass) = names(input_ras)
```


```r
reclass
#> ... (full output not shown)
#> names       :  pop, women, mean_age, hh_size 
#> min values  :  127,     0,        0,       0 
#> max values  : 8000,     3,        3,       3
```


## Define metropolitan areas

We define metropolitan areas as pixels of 20 km^2^ inhabited by more than 500,000 people.
Pixels at this coarse resolution can rapidly be created using `aggregate()`\index{aggregation}, as introduced in Section \@ref(aggregation-and-disaggregation).
The command below uses the argument `fact = 20` to reduce the resolution of the result twenty-fold (recall the original raster resolution was 1 km^2^):


```r
pop_agg = aggregate(reclass$pop, fact = 20, fun = sum)
```

The next stage is to keep only cells with more than half a million people.


```r
summary(pop_agg)
#>             pop
#> Min.        127
#> 1st Qu.   39886
#> Median    66008
#> 3rd Qu.  105696
#> Max.    1204870
#> NA's        447
pop_agg = pop_agg[pop_agg > 500000, drop = FALSE] 
```

Plotting this reveals eight metropolitan regions (Figure \@ref(fig:metro-areas)).
Each region consists of one or more raster cells.
It would be nice if we could join all cells belonging to one region.
**raster**'s\index{raster} `clump()` command does exactly that.
Subsequently, `rasterToPolygons()` converts the raster object into spatial polygons, and `st_as_sf()` converts it into an `sf`-object.


```r
polys = pop_agg %>% 
  clump() %>%
  rasterToPolygons() %>%
  st_as_sf()
```

`polys` now features a column named `clumps` which indicates to which metropolitan region each polygon belongs and which we will use to dissolve\index{dissolve} the polygons into coherent single regions (see also Section \@ref(geometry-unions)):


```r
metros = polys %>%
  group_by(clumps) %>%
  summarize()
#> `summarise()` ungrouping output (override with `.groups` argument)
```

Given no other column as input, `summarize()` only dissolves the geometry.

<!-- maybe a good if advanced exercise
This requires finding the nearest neighbors (`st_intersects()`), and some additional processing.
Do not worry too much about the following code.
There is probably a better way to do it. 
Nevertheless, it finds all pixels belonging to one region in a generic way.
We use this information to assign each polygon (pixel) to a region.
Subsequently, we can use the region information to dissolve the pixels into region polygons.


```r
# dissolve on spatial neighborhood
nbs = st_intersects(polys, polys)
# nbs = over(polys, polys, returnList = TRUE)

fun = function(x, y) {
  tmp = lapply(y, function(i) {
  if (any(x %in% i)) {
   union(x, i)
  } else {
   x
    }
  })
  Reduce(union, tmp)
}
# call function recursively
fun_2 = function(x, y) {
  out = fun(x, y)
  while (length(out) < length(fun(out, y))) {
    out = fun(out, y)
  }
  out
}

cluster = map(nbs, ~ fun_2(., nbs) %>% sort)
# just keep unique clusters
cluster = cluster[!duplicated(cluster)]
# assign the cluster classes to each pixel
for (i in seq_along(cluster)) {
  polys[cluster[[i]], "region_id"] = i
}
# dissolve pixels based on the the region id
polys = group_by(polys, region_id) %>%
  summarize(pop = sum(layer, na.rm = TRUE))
# polys_2 = aggregate(polys, list(polys$region_id), sum)
plot(polys[, "region_id"])

# Another approach, can be also be part of an excercise

coords = st_coordinates(polys_3) %>% 
  as.data.frame
ls = split(coords, f = coords$L2)
ls = lapply(ls, function(x) {
  dplyr::select(x, X, Y) %>%
    as.matrix %>%
    list %>%
    st_polygon
})
metros = do.call(st_sfc, ls)
metros = st_set_crs(metros, 3035)
metros = st_sf(data.frame(region_id = 1:9), geometry = metros)
st_intersects(metros, metros)
plot(metros[-5,])
st_centroid(metros) %>%
  st_coordinates
```
-->


<div class="figure" style="text-align: center">
<img src="figures/08_metro_areas.png" alt="The aggregated population raster (resolution: 20 km) with the identified metropolitan areas (golden polygons) and the corresponding names." width="100%" />
<p class="caption">(\#fig:metro-areas)The aggregated population raster (resolution: 20 km) with the identified metropolitan areas (golden polygons) and the corresponding names.</p>
</div>

The resulting eight metropolitan areas suitable for bike shops (Figure \@ref(fig:metro-areas); see also `code/13-location-jm.R` for creating the figure) are still missing a name.
A reverse geocoding\index{geocoding} approach can settle this problem.
Given a coordinate, reverse geocoding finds the corresponding address.
Consequently, extracting the centroid\index{centroid} coordinate of each metropolitan area can serve as an input for a reverse geocoding API\index{API}.
The **revgeo** package provides access to the open source Photon geocoder for OpenStreetMap\index{OpenStreetMap}, Google Maps and Bing. 
By default, it uses the Photon API\index{API}.
`revgeo::revgeo()` only accepts geographical coordinates (latitude/longitude); therefore, the first requirement is to bring the metropolitan polygons into an appropriate coordinate reference system\index{CRS} (Chapter \@ref(reproj-geo-data)).


```r
metros_wgs = st_transform(metros, 4326)
coords = st_centroid(metros_wgs) %>%
  st_coordinates() %>%
  round(4)
```

<!-- It is a great feature of `revgeo::revgeo()` that it accepts also more than one coordinate pair as input (parameters `longitude` and `latitude`), which spares us the need for writing a loop. -->
Choosing `frame` as `revgeocode()`'s `output` option will give back a `data.frame` with several columns referring to the location including the street name, house number and city.


```r
library(revgeo)
metro_names = revgeo(longitude = coords[, 1], latitude = coords[, 2], 
                     output = "frame")
```

To make sure that the reader uses the exact same results, we have put them into **spDataLarge** as the object `metro_names`.


Table: (\#tab:metro-names)Result of the reverse geocoding.

|city              |state                  |
|:-----------------|:----------------------|
|Hamburg           |Hamburg                |
|Berlin            |Berlin                 |
|Wülfrath          |North Rhine-Westphalia |
|Leipzig           |Saxony                 |
|Frankfurt am Main |Hesse                  |
|Nuremberg         |Bavaria                |
|Stuttgart         |Baden-Württemberg      |
|Munich            |Bavaria                |

Overall, we are satisfied with the `city` column serving as metropolitan names (Table \@ref(tab:metro-names)) apart from one exception, namely Wülfrath which belongs to the greater region of Düsseldorf.
Hence, we replace Wülfrath with Düsseldorf (Figure \@ref(fig:metro-areas)).
Umlauts like `ü` might lead to trouble further on, for example when determining the bounding box of a metropolitan area with `opq()` (see further below), which is why we avoid them.


```r
metro_names = dplyr::pull(metro_names, city) %>% 
  as.character() %>% 
  ifelse(. == "Wülfrath", "Duesseldorf", .)
```


## Points of interest
\index{point of interest}
The **osmdata**\index{osmdata (package)} package provides easy-to-use access to OSM\index{OpenStreetMap} data (see also Section \@ref(retrieving-data)).
Instead of downloading shops for the whole of Germany, we restrict the query to the defined metropolitan areas, reducing computational load and providing shop locations only in areas of interest.
The subsequent code chunk does this using a number of functions including:

- `map()`\index{loop!map} (the **tidyverse** equivalent of `lapply()`\index{loop!lapply}), which iterates through all eight metropolitan names which subsequently define the bounding box\index{bounding box} in the OSM\index{OpenStreetMap} query function `opq()` (see Section \@ref(retrieving-data)).
<!-- Alternatively, we could have provided the bounding box in the form of coordinates ourselves. -->
- `add_osm_feature()` to specify OSM\index{OpenStreetMap} elements with a key value of `shop` (see [wiki.openstreetmap.org](http://wiki.openstreetmap.org/wiki/Map_Features) for a list of common key:value pairs).
- `osmdata_sf()`, which converts the OSM\index{OpenStreetMap} data into spatial objects (of class `sf`).
- `while()`\index{loop!while}, which tries repeatedly (three times in this case) to download the data if it fails the first time.^[The OSM-download will sometimes fail at the first attempt.
]
Before running this code: please consider it will download almost 2GB of data.
To save time and resources, we have put the output named `shops` into **spDataLarge**.
To make it available in your environment ensure that the **spDataLarge** package is loaded, or run `data("shops", package = "spDataLarge")`.


```r
shops = map(metro_names, function(x) {
  message("Downloading shops of: ", x, "\n")
  # give the server a bit time
  Sys.sleep(sample(seq(5, 10, 0.1), 1))
  query = opq(x) %>%
    add_osm_feature(key = "shop")
  points = osmdata_sf(query)
  # request the same data again if nothing has been downloaded
  iter = 2
  while (nrow(points$osm_points) == 0 & iter > 0) {
    points = osmdata_sf(query)
    iter = iter - 1
  }
  points = st_set_crs(points$osm_points, 4326)
})
```

It is highly unlikely that there are no shops in any of our defined metropolitan areas.
The following `if` condition simply checks if there is at least one shop for each region.
If not, we recommend to try to download the shops again for this/these specific region/s.


```r
# checking if we have downloaded shops for each metropolitan area
ind = map(shops, nrow) == 0
if (any(ind)) {
  message("There are/is still (a) metropolitan area/s without any features:\n",
          paste(metro_names[ind], collapse = ", "), "\nPlease fix it!")
}
```

To make sure that each list element (an `sf`\index{sf} data frame) comes with the same columns, we only keep the `osm_id` and the `shop` columns with the help of another `map` loop.
This is not a given since OSM contributors are not equally meticulous when collecting data.
Finally, we `rbind` all shops into one large `sf`\index{sf} object.


```r
# select only specific columns
shops = map(shops, dplyr::select, osm_id, shop)
# putting all list elements into a single data frame
shops = do.call(rbind, shops)
```

It would have been easier to simply use `map_dfr()`\index{loop!map\_dfr}. 
Unfortunately, so far it does not work in harmony with `sf` objects.
Note: `shops` is provided in the `spDataLarge` package.

The only thing left to do is to convert the spatial point object into a raster (see Section \@ref(rasterization)).
The `sf` object, `shops`, is converted into a raster\index{raster} having the same parameters (dimensions, resolution, CRS\index{CRS}) as the `reclass` object.
Importantly, the `count()` function is used here to calculate the number of shops in each cell.

\BeginKnitrBlock{rmdnote}<div class="rmdnote">If the `shop` column were used instead of the `osm_id` column, we would have retrieved fewer shops per grid cell. 
This is because the `shop` column contains `NA` values, which the `count()` function omits when rasterizing vector objects.</div>\EndKnitrBlock{rmdnote}

The result of the subsequent code chunk is therefore an estimate of shop density (shops/km^2^).
`st_transform()`\index{sf!st\_transform} is used before `rasterize()`\index{raster!rasterize} to ensure the CRS\index{CRS} of both inputs match.


```r
shops = st_transform(shops, proj4string(reclass))
# create poi raster
poi = rasterize(x = shops, y = reclass, field = "osm_id", fun = "count")
#> Warning in showSRID(uprojargs, format = "PROJ", multiline = "NO", prefer_proj =
#> prefer_proj): Discarded datum Unknown based on GRS80 ellipsoid in CRS definition
#> Warning in showSRID(SRS_string, format = "PROJ", multiline = "NO", prefer_proj =
#> prefer_proj): Discarded datum Unknown based on GRS80 ellipsoid in CRS definition
```

As with the other raster layers (population, women, mean age, household size) the `poi` raster is reclassified into four classes (see Section \@ref(create-census-rasters)). 
Defining class intervals is an arbitrary undertaking to a certain degree.
One can use equal breaks, quantile breaks, fixed values or others.
Here, we choose the Fisher-Jenks natural breaks approach which minimizes within-class variance, the result of which provides an input for the reclassification matrix.


```r
# construct reclassification matrix
int = classInt::classIntervals(values(poi), n = 4, style = "fisher")
int = round(int$brks)
rcl_poi = matrix(c(int[1], rep(int[-c(1, length(int))], each = 2), 
                   int[length(int)] + 1), ncol = 2, byrow = TRUE)
rcl_poi = cbind(rcl_poi, 0:3)  
# reclassify
poi = reclassify(poi, rcl = rcl_poi, right = NA) 
names(poi) = "poi"
```

## Identifying suitable locations

The only steps that remain before combining all the layers are to add `poi` to the `reclass` raster stack and remove the population layer from it.
The reasoning for the latter is twofold.
First of all, we have already delineated metropolitan areas, that is areas where the population density is above average compared to the rest of Germany.
Second, though it is advantageous to have many potential customers within a specific catchment area\index{catchment area}, the sheer number alone might not actually represent the desired target group.
For instance, residential tower blocks are areas with a high population density but not necessarily with a high purchasing power for expensive cycle components.
This is achieved with the complementary functions `addLayer()` and `dropLayer()`:


```r
# add poi raster
reclass = addLayer(reclass, poi)
# delete population raster
reclass = dropLayer(reclass, "pop")
```

In common with other data science projects, data retrieval and 'tidying' have consumed much of the overall workload so far.
With clean data, the final step --- calculating a final score by summing all raster\index{raster} layers --- can be accomplished in a single line of code.


```r
# calculate the total score
result = sum(reclass)
```

For instance, a score greater than 9 might be a suitable threshold indicating raster cells where a bike shop could be placed (Figure \@ref(fig:bikeshop-berlin); see also `code/13-location-jm.R`).


```
#> Warning in showSRID(uprojargs, format = "PROJ", multiline = "NO", prefer_proj
#> = prefer_proj): Discarded ellps WGS 84 in CRS definition: +proj=merc +a=6378137
#> +b=6378137 +lat_ts=0 +lon_0=0 +x_0=0 +y_0=0 +k=1 +units=m +nadgrids=@null
#> +wktext +no_defs +type=crs
#> Warning in showSRID(uprojargs, format = "PROJ", multiline = "NO", prefer_proj =
#> prefer_proj): Discarded datum World Geodetic System 1984 in CRS definition
#> Warning in showSRID(uprojargs, format = "PROJ", multiline = "NO", prefer_proj
#> = prefer_proj): Discarded ellps WGS 84 in CRS definition: +proj=merc +a=6378137
#> +b=6378137 +lat_ts=0 +lon_0=0 +x_0=0 +y_0=0 +k=1 +units=m +nadgrids=@null
#> +wktext +no_defs +type=crs
#> Warning in showSRID(uprojargs, format = "PROJ", multiline = "NO", prefer_proj =
#> prefer_proj): Discarded datum World Geodetic System 1984 in CRS definition
```

<div class="figure" style="text-align: center">
<!--html_preserve--><div id="htmlwidget-841de324d41155df19a0" style="width:100%;height:415.296px;" class="leaflet html-widget"></div>
<script type="application/json" data-for="htmlwidget-841de324d41155df19a0">{"x":{"options":{"crs":{"crsClass":"L.CRS.EPSG3857","code":null,"proj4def":null,"projectedBounds":null,"options":{}}},"calls":[{"method":"addTiles","args":["//{s}.tile.openstreetmap.org/{z}/{x}/{y}.png",null,null,{"minZoom":0,"maxZoom":18,"tileSize":256,"subdomains":"abc","errorTileUrl":"","tms":false,"noWrap":false,"zoomOffset":0,"zoomReverse":false,"opacity":1,"zIndex":1,"detectRetina":false,"attribution":"&copy; <a href=\"http://openstreetmap.org\">OpenStreetMap<\/a> contributors, <a href=\"http://creativecommons.org/licenses/by-sa/2.0/\">CC-BY-SA<\/a>"}]},{"method":"addRasterImage","args":["data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAACgAAAAoCAYAAACM/rhtAAAAM0lEQVRYhe3SMQ0AMAwEsQcb/hTSuQSaVLIR3HAJAADfqfR0wm1dEC9VevcCq+MAAIBhB20vBvBK3JZrAAAAAElFTkSuQmCC",[[52.696600857292,13.0820026147986],[52.3210740886184,13.6981591380976]],0.8,null,null,null]},{"method":"addLegend","args":[{"colors":["darkgreen"],"labels":["potential locations"],"na_color":null,"na_label":"NA","opacity":0.5,"position":"bottomright","type":"unknown","title":"Legend","extra":null,"layerId":null,"className":"info legend","group":null}]}],"limits":{"lat":[52.3210740886184,52.696600857292],"lng":[13.0820026147986,13.6981591380976]}},"evals":[],"jsHooks":[]}</script><!--/html_preserve-->
<p class="caption">(\#fig:bikeshop-berlin)Suitable areas (i.e., raster cells with a score > 9) in accordance with our hypothetical survey for bike stores in Berlin.</p>
</div>

## Discussion and next steps

The presented approach is a typical example of the normative usage of a GIS\index{GIS} [@longley_geographic_2015].
We combined survey data with expert-based knowledge and assumptions (definition of metropolitan areas, defining class intervals, definition of a final score threshold).
This approach is less suitable for scientific research than applied analysis that provides an evidence based indication of areas suitable for bike shops that should be compared with other sources of information.
A number of changes to the approach could improve the analysis:

- We used equal weights when calculating the final scores but other factors, such as the household size, could be as important as the portion of women or the mean age
- We used all points of interest\index{point of interest} but only those related to bike shops, such as do-it-yourself, hardware, bicycle, fishing, hunting, motorcycles, outdoor and sports shops (see the range of shop values available on the  [OSM Wiki](http://wiki.openstreetmap.org/wiki/Map_Features#Shop)) may have yielded more refined results
- Data at a higher resolution may improve the output (see exercises)
- We have used only a limited set of variables and data from other sources, such as the [INSPIRE geoportal](http://inspire-geoportal.ec.europa.eu/discovery/) or data on cycle paths from OpenStreetMap, may enrich the analysis (see also Section \@ref(retrieving-data))
- Interactions remained unconsidered, such as a possible relationships between the portion of men and single households
<!-- However, to find out about such an interaction we would need customer data. -->

In short, the analysis could be extended in multiple directions.
Nevertheless, it should have given you a first impression and understanding of how to obtain and deal with spatial data in R\index{R} within a geomarketing\index{geomarketing} context.

Finally, we have to point out that the presented analysis would be merely the first step of finding suitable locations.
So far we have identified areas, 1 by 1 km in size, representing potentially suitable locations for a bike shop in accordance with our survey.
Subsequent steps in the analysis could be taken:

- Find an optimal location based on number of inhabitants within a specific catchment area\index{catchment area}.
For example, the shop should be reachable for as many people as possible within 15 minutes of traveling bike distance (catchment area\index{catchment area} routing\index{routing}).
Thereby, we should account for the fact that the further away the people are from the shop, the more unlikely it becomes that they actually visit it (distance decay function).
- Also it would be a good idea to take into account competitors. 
That is, if there already is a bike shop in the vicinity of the chosen location, one has to distribute possible customers (or sales potential) between the competitors [@huff_probabilistic_1963; @wieland_market_2017].
- We need to find suitable and affordable real estate, e.g., in terms of accessibility, availability of parking spots, desired frequency of passers-by, having big windows, etc.

## Exercises

1. We have used `raster::rasterFromXYZ()` to convert a `input_tidy` into a raster brick\index{raster!brick}.
Try to achieve the same with the help of the `sp::gridded()` function.
<!--
input = st_as_sf(input, coords = c("x", "y"))
# use the correct projection (see data description)
input = st_set_crs(input, 3035)
# convert into an sp-object
input = as(input, "Spatial")
gridded(input) = TRUE
# convert into a raster stack
input = stack(input)
-->

1. Download the csv file containing inhabitant information for a 100-m cell resolution (https://www.zensus2011.de/SharedDocs/Downloads/DE/Pressemitteilung/DemografischeGrunddaten/csv_Bevoelkerung_100m_Gitter.zip?__blob=publicationFile&v=3).
Please note that the unzipped file has a size of 1.23 GB.
To read it into R\index{R}, you can use `readr::read_csv`.
This takes 30 seconds on my machine (16 GB RAM)
`data.table::fread()` might be even faster, and returns an object of class `data.table()`.
Use `as.tibble()` to convert it into a tibble\index{tibble}.
Build an inhabitant raster\index{raster}, aggregate\index{aggregation!spatial} it to a cell resolution of 1 km, and compare the difference with the inhabitant raster (`inh`) we have created using class mean values.

1. Suppose our bike shop predominantly sold electric bikes to older people. 
Change the age raster\index{raster} accordingly, repeat the remaining analyses and compare the changes with our original result.
