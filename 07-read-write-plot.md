# Geographic data I/O {#read-write}

## Prerequisites {-}

This chapter requires the following packages:


```r
library(sf)
library(raster)
library(dplyr)
library(spData)
```

## Introduction

This chapter is about reading and writing geographic data.
Geographic data *import* is essential for geocomputation\index{geocomputation}: real-world applications are impossible without data.
For others to benefit from the results of your work, data *output* is also vital.
Taken together, we refer to these processes as I/O, short for input/output.

Geographic data I/O is almost always part of a wider process.
It depends on knowing which datasets are *available*, where they can be *found* and how to *retrieve* them.
These topics are covered in Section \@ref(retrieving-data), which describes various *geoportals*, which collectively contain many terabytes of data, and how to use them.
To further ease data access, a number of packages for downloading geographic data have been developed.
These are described in Section \@ref(geographic-data-packages).

There are many geographic file formats, each of which has pros and cons.
These are described in Section \@ref(file-formats).
The process of actually reading and writing such file formats efficiently is not covered until Sections \@ref(data-input) and \@ref(data-output), respectively.
The final Section \@ref(visual-outputs) demonstrates methods for saving visual outputs (maps), in preparation for Chapter \@ref(adv-map) on visualization.

## Retrieving open data {#retrieving-data}

\index{open data}
A vast and ever-increasing amount of geographic data is available on the internet, much of which is free to access and use (with appropriate credit given to its providers).
In some ways there is now *too much* data, in the sense that there are often multiple places to access the same dataset.
Some datasets are of poor quality.
In this context, it is vital to know where to look, so the first section covers some of the most important sources.
Various 'geoportals' (web services providing geospatial datasets such as [Data.gov](https://catalog.data.gov/dataset?metadata_type=geospatial)) are a good place to start, providing a wide range of data but often only for specific locations (as illustrated in the updated [Wikipedia page](https://en.wikipedia.org/wiki/Geoportal) on the topic).

\index{geoportals}
Some global geoportals overcome this issue.
The [GEOSS portal](http://www.geoportal.org/) and the [Copernicus Open Access Hub](https://scihub.copernicus.eu/), for example, contain many raster datasets with global coverage.
A wealth of vector datasets can be accessed from the National Aeronautics and Space Administration agency (NASA), [SEDAC](http://sedac.ciesin.columbia.edu/) portal and the European Union's [INSPIRE geoportal](http://inspire-geoportal.ec.europa.eu/), with global and regional coverage.

Most geoportals provide a graphical interface allowing datasets to be queried based on characteristics such as spatial and temporal extent, the United States Geological Services' [EarthExplorer](https://earthexplorer.usgs.gov/) being a prime example.
*Exploring* datasets interactively on a browser is an effective way of understanding available layers.
*Downloading* data is best done with code, however, from reproducibility and efficiency perspectives.
Downloads can be initiated from the command line using a variety of techniques, primarily via URLs and APIs\index{API} (see the [Sentinel API](https://scihub.copernicus.eu/twiki/do/view/SciHubWebPortal/APIHubDescription) for example).
Files hosted on static URLs can be downloaded with `download.file()`, as illustrated in the code chunk below which accesses US National Parks data from: [catalog.data.gov/dataset/national-parks](https://catalog.data.gov/dataset/national-parks):


```r
download.file(url = "http://nrdata.nps.gov/programs/lands/nps_boundary.zip",
              destfile = "nps_boundary.zip")
unzip(zipfile = "nps_boundary.zip")
usa_parks = st_read(dsn = "nps_boundary.shp")
```

## Geographic data packages

\index{data packages}
A multitude of R packages have been developed for accessing geographic data, some of which are presented in Table \@ref(tab:datapackages).
These provide interfaces to one or more spatial libraries or geoportals and aim to make data access even quicker from the command line.

<!-- add sentinel2 package as soon as it is published on CRAN https://github.com/IVFL-BOKU/sentinel2-->
<table class="table" style="margin-left: auto; margin-right: auto;">
<caption>(\#tab:datapackages)Selected R packages for geographic data retrieval.</caption>
 <thead>
  <tr>
   <th style="text-align:left;"> Package </th>
   <th style="text-align:left;"> Description </th>
  </tr>
 </thead>
<tbody>
  <tr>
   <td style="text-align:left;"> getlandsat </td>
   <td style="text-align:left;"> Provides access to Landsat 8 data. </td>
  </tr>
  <tr>
   <td style="text-align:left;"> osmdata </td>
   <td style="text-align:left;"> Download and import of OpenStreetMap data. </td>
  </tr>
  <tr>
   <td style="text-align:left;"> raster </td>
   <td style="text-align:left;"> getData() imports administrative, elevation, WorldClim data. </td>
  </tr>
  <tr>
   <td style="text-align:left;"> rnaturalearth </td>
   <td style="text-align:left;"> Access to Natural Earth vector and raster data. </td>
  </tr>
  <tr>
   <td style="text-align:left;"> rnoaa </td>
   <td style="text-align:left;"> Imports National Oceanic and Atmospheric Administration (NOAA) climate data. </td>
  </tr>
  <tr>
   <td style="text-align:left;"> rWBclimate </td>
   <td style="text-align:left;"> Access World Bank climate data. </td>
  </tr>
</tbody>
</table>

It should be emphasised that Table \@ref(tab:datapackages) represents only a small number of available geographic data packages.
Other notable packages include **GSODR**, which provides Global Summary Daily Weather Data in R (see the package's [README](https://github.com/ropensci/GSODR) for an overview of weather data sources);
**tidycensus** and **tigris**, which provide socio-demographic vector data for the USA; and **hddtools**, which provides access to a range of hydrological datasets.

Each data package has its own syntax for accessing data.
This diversity is demonstrated in the subsequent code chunks, which show how to get data using three packages from Table \@ref(tab:datapackages).
Country borders are often useful and these can be accessed with the `ne_countries()` function from the **rnaturalearth** package as follows:


```r
library(rnaturalearth)
usa = ne_countries(country = "United States of America") # United States borders
class(usa)
#> [1] "SpatialPolygonsDataFrame"
#> attr(,"package")
#> [1] "sp"
# alternative way of accessing the data, with raster::getData()
# getData("GADM", country = "USA", level = 0)
```

By default **rnaturalearth** returns objects of class `Spatial`.
The result can be converted into an `sf` objects with `st_as_sf()` as follows:


```r
usa_sf = st_as_sf(usa)
```

A second example downloads a series of rasters containing global monthly precipitation sums with spatial resolution of ten minutes.
The result is a multilayer object of class `RasterStack`.


```r
library(raster)
worldclim_prec = getData(name = "worldclim", var = "prec", res = 10)
class(worldclim_prec)
#> [1] "RasterStack"
#> attr(,"package")
#> [1] "raster"
```

A third example uses the **osmdata** package [@R-osmdata] to find parks from the OpenStreetMap (OSM) database\index{OpenStreetMap}.
As illustrated in the code-chunk below, queries begin with the function `opq()` (short for OpenStreetMap query), the first argument of which is bounding box, or text string representing a bounding box (the city of Leeds in this case).
The result is passed to a function for selecting which OSM elements we're interested in (parks in this case), represented by *key-value pairs*. Next, they are passed to the function `osmdata_sf()` which does the work of downloading the data and converting it into a list of `sf` objects (see `vignette('osmdata')` for further details):


```r
library(osmdata)
parks = opq(bbox = "leeds uk") %>% 
  add_osm_feature(key = "leisure", value = "park") %>% 
  osmdata_sf()
```

OpenStreetMap is a vast global database of crowd-sourced data and it is growing daily.
Although the quality is not as spatially consistent as many official datasets, OSM data have many advantages: they are globally available free of charge and using crowd-source data can encourage 'citizen science' and contributions back to the digital commons.
Further examples of **osmdata** in action are provided in Chapters \@ref(gis), \@ref(transport) and \@ref(location).

Sometimes, packages come with inbuilt datasets.
These can be accessed in four ways: by attaching the package (if the package uses 'lazy loading' as **spData** does), with `data(dataset)`, by referring to the dataset with `pkg::dataset` or with `system.file()` to access raw data files.
The following code chunk illustrates the latter two options using the `world` dataset (already loaded by attaching its parent package with `library(spData)`):^[
For more information on data import with R packages, see Sections 5.5 and 5.6 of @gillespie_efficient_2016.
]


```r
world2 = spData::world
world3 = st_read(system.file("shapes/world.gpkg", package = "spData"))
```

## Geographic web services

\index{geographic web services}
In an effort to standardize web APIs for accessing spatial data, the Open Geospatial Consortium (OGC) has created a number of specifications for web services (collectively known as OWS, which is short for OGC Web Services).
These specifications include the Web Feature Service (WFS)\index{geographic web services!WFS}, Web Map Service (WMS)\index{geographic web services!WMS}, Web Map Tile Service (WMTS)\index{geographic web services!WMTS}, the Web Coverage Service (WCS)\index{geographic web services!WCS} and even a Web Processing Service (WPS)\index{geographic web services!WPS}.
Map servers such as PostGIS have adopted these protocols, leading to standardization of queries.
Like other web APIs, OWS APIs use a 'base URL', an 'endpoint' and 'URL query arguments' following a `?` to request data (see the [`best-practices-api-packages`](https://httr.r-lib.org/articles/api-packages.html) vignette in the **httr** package).

There are many requests that can be made to a OWS service.
One of the most fundamental is `getCapabilities`, demonstrated with **httr** below.
The code chunk demonstrates how API\index{API} queries can be constructed and dispatched, in this case to discover the capabilities of a service run by the Food and Agriculture Organization of the United Nations (FAO):


```r
base_url = "http://www.fao.org"
endpoint = "/figis/geoserver/wfs"
q = list(request = "GetCapabilities")
res = httr::GET(url = httr::modify_url(base_url, path = endpoint), query = q)
res$url
#> [1] "http://www.fao.org/figis/geoserver/wfs?request=GetCapabilities"
```

The above code chunk demonstrates how API\index{API} requests can be constructed programmatically with the `GET()` function, which takes a base URL and a list of query parameters which can easily be extended.
The result of the request is saved in `res`, an object of class `response` defined in the **httr** package, which is a list containing information of the request, including the URL.
As can be seen by executing `browseURL(res$url)`, the results can also be read directly in a browser.
One way of extracting the contents of the request is as follows:


```r
txt = httr::content(res, "text")
xml = xml2::read_xml(txt)
```


```r
xml
#> {xml_document} ...
#> [1] <ows:ServiceIdentification>\n  <ows:Title>GeoServer WFS...
#> [2] <ows:ServiceProvider>\n  <ows:ProviderName>Food and Agr...
#> ...
```

Data can be downloaded from WFS services with the `GetFeature` request and a specific `typeName` (as illustrated in the code chunk below).



Available names differ depending on the accessed web feature service.
One can extract them programmatically using web technologies [@nolan_xml_2014] or scrolling manually through the contents of the `GetCapabilities` output in a browser.


```r
qf = list(request = "GetFeature", typeName = "area:FAO_AREAS")
file = tempfile(fileext = ".gml")
httr::GET(url = base_url, path = endpoint, query = qf, httr::write_disk(file))
fao_areas = sf::read_sf(file)
```

Note the use of `write_disk()` to ensure that the results are written to disk rather than loaded into memory, allowing them to be imported with **sf**.
This example shows how to gain low-level access to web services using **httr**, which can be useful for understanding how web services work.
For many everyday tasks, however, a higher-level interface may be more appropriate, and a number of R packages, and tutorials, have been developed precisely for this purpose.

Packages **ows4R**, **rwfs** and **sos4R** have been developed for working with OWS services in general, WFS and the sensor observation service (SOS) respectively.
As of October 2018, only **ows4R** is on CRAN.
The package's basic functionality is demonstrated below, in commands that get all `FAO_AREAS` as we did in the previous code chunk:^[
To filter features on the server before downloading them, the argument `cql_filter` can be used. Adding `cql_filter = URLencode("F_CODE= '27'")` to the command, for example, would instruct the server to only return the feature with values in the `F_CODE` column equal to 27.
]


```r
library(ows4R)
wfs = WFSClient$new("http://www.fao.org/figis/geoserver/wfs",
                      serviceVersion = "1.0.0", logger = "INFO")
fao_areas = wfs$getFeatures("area:FAO_AREAS")
```



There is much more to learn about web services and much potential for development of R-OWS interfaces, an active area of development.
For further information on the topic, we recommend examples from European Centre for Medium-Range Weather Forecasts (ECMWF) services at [github.com/OpenDataHack](https://github.com/OpenDataHack/data_service_catalogue) and reading-up on OCG Web Services at [opengeospatial.org](http://www.opengeospatial.org/standards).





## File formats

\index{file formats}
Geographic datasets are usually stored as files or in spatial databases.
File formats can either store vector or raster data, while spatial databases such as [PostGIS](https://trac.osgeo.org/postgis/wiki/WKTRaster) can store both (see also Section \@ref(postgis)).
Today the variety of file formats may seem bewildering but there has been much consolidation and standardization since the beginnings of GIS software in the 1960s when the first widely distributed program ([SYMAP](https://news.harvard.edu/gazette/story/2011/10/the-invention-of-gis/)) for spatial analysis was created at Harvard University [@coppock_history_1991].

\index{GDAL}
GDAL (which should be pronounced "goo-dal", with the double "o" making a reference to object-orientation), the Geospatial Data Abstraction Library, has resolved many issues associated with incompatibility between geographic file formats since its release in 2000.
GDAL provides a unified and high-performance interface for reading and writing of many raster and vector data formats.
Many open and proprietary GIS programs, including GRASS, ArcGIS\index{ArcGIS} and QGIS\index{QGIS}, use GDAL\index{GDAL} behind their GUIs\index{graphical user interface} for doing the legwork of ingesting and spitting out geographic data in appropriate formats.

GDAL\index{GDAL} provides access to more than 200 vector and raster data formats.
Table \@ref(tab:formats) presents some basic information about selected and often used spatial file formats.

<table>
<caption>(\#tab:formats)Selected spatial file formats.</caption>
 <thead>
  <tr>
   <th style="text-align:left;"> Name </th>
   <th style="text-align:left;"> Extension </th>
   <th style="text-align:left;"> Info </th>
   <th style="text-align:left;"> Type </th>
   <th style="text-align:left;"> Model </th>
  </tr>
 </thead>
<tbody>
  <tr>
   <td style="text-align:left;"> ESRI Shapefile </td>
   <td style="text-align:left;width: 7em; "> .shp (the main file) </td>
   <td style="text-align:left;width: 14em; "> Popular format consisting of at least three files. No support for: files &gt; 2GB;  mixed types; names &gt; 10 chars; cols &gt; 255. </td>
   <td style="text-align:left;"> Vector </td>
   <td style="text-align:left;width: 7em; "> Partially open </td>
  </tr>
  <tr>
   <td style="text-align:left;"> GeoJSON </td>
   <td style="text-align:left;width: 7em; "> .geojson </td>
   <td style="text-align:left;width: 14em; "> Extends the JSON exchange format by including a subset of the simple feature representation. </td>
   <td style="text-align:left;"> Vector </td>
   <td style="text-align:left;width: 7em; "> Open </td>
  </tr>
  <tr>
   <td style="text-align:left;"> KML </td>
   <td style="text-align:left;width: 7em; "> .kml </td>
   <td style="text-align:left;width: 14em; "> XML-based format for spatial visualization, developed for use with Google Earth. Zipped KML file forms the KMZ format. </td>
   <td style="text-align:left;"> Vector </td>
   <td style="text-align:left;width: 7em; "> Open </td>
  </tr>
  <tr>
   <td style="text-align:left;"> GPX </td>
   <td style="text-align:left;width: 7em; "> .gpx </td>
   <td style="text-align:left;width: 14em; "> XML schema created for exchange of GPS data. </td>
   <td style="text-align:left;"> Vector </td>
   <td style="text-align:left;width: 7em; "> Open </td>
  </tr>
  <tr>
   <td style="text-align:left;"> GeoTIFF </td>
   <td style="text-align:left;width: 7em; "> .tif/.tiff </td>
   <td style="text-align:left;width: 14em; "> Popular raster format. A TIFF file containing additional spatial metadata. </td>
   <td style="text-align:left;"> Raster </td>
   <td style="text-align:left;width: 7em; "> Open </td>
  </tr>
  <tr>
   <td style="text-align:left;"> Arc ASCII </td>
   <td style="text-align:left;width: 7em; "> .asc </td>
   <td style="text-align:left;width: 14em; "> Text format where the first six lines represent the raster header, followed by the raster cell values arranged in rows and columns. </td>
   <td style="text-align:left;"> Raster </td>
   <td style="text-align:left;width: 7em; "> Open </td>
  </tr>
  <tr>
   <td style="text-align:left;"> R-raster </td>
   <td style="text-align:left;width: 7em; "> .gri, .grd </td>
   <td style="text-align:left;width: 14em; "> Native raster format of the R-package raster. </td>
   <td style="text-align:left;"> Raster </td>
   <td style="text-align:left;width: 7em; "> Open </td>
  </tr>
  <tr>
   <td style="text-align:left;"> SQLite/SpatiaLite </td>
   <td style="text-align:left;width: 7em; "> .sqlite </td>
   <td style="text-align:left;width: 14em; "> Standalone  relational database, SpatiaLite is the spatial extension of SQLite. </td>
   <td style="text-align:left;"> Vector and raster </td>
   <td style="text-align:left;width: 7em; "> Open </td>
  </tr>
  <tr>
   <td style="text-align:left;"> ESRI FileGDB </td>
   <td style="text-align:left;width: 7em; "> .gdb </td>
   <td style="text-align:left;width: 14em; "> Spatial and nonspatial objects created by ArcGIS. Allows: multiple feature classes; topology. Limited support from GDAL. </td>
   <td style="text-align:left;"> Vector and raster </td>
   <td style="text-align:left;width: 7em; "> Proprietary </td>
  </tr>
  <tr>
   <td style="text-align:left;"> GeoPackage </td>
   <td style="text-align:left;width: 7em; "> .gpkg </td>
   <td style="text-align:left;width: 14em; "> Lightweight database container based on SQLite allowing an easy and platform-independent exchange of geodata </td>
   <td style="text-align:left;"> Vector and raster </td>
   <td style="text-align:left;width: 7em; "> Open </td>
  </tr>
</tbody>
</table>
\index{Shapefile}

An important development ensuring the standardization and open-sourcing of file formats was the founding of the Open Geospatial Consortium ([OGC](http://www.opengeospatial.org/)) in 1994.
Beyond defining the simple features data model (see Section \@ref(intro-sf)), the OGC also coordinates the development of open standards, for example as used in file formats such as KML\index{KML} and GeoPackage\index{GeoPackage}.
Open file formats of the kind endorsed by the OGC have several advantages over proprietary formats: the standards are published, ensure transparency and open up the possibility for users to further develop and adjust the file formats to their specific needs.

ESRI Shapefile\index{Shapefile} is the most popular vector data exchange format.
However, it is not an open format (though its specification is open).
It was developed in the early 1990s and has a number of limitations.
First of all, it is a multi-file format, which consists of at least three files.
It only supports 255 columns, column names are restricted to ten characters and the file size limit is 2 GB.
Furthermore, ESRI Shapefile\index{Shapefile} does not support all possible geometry types, for example, it is unable to distinguish between a polygon and a multipolygon.^[To learn more about ESRI Shapefile limitations and possible alternative file formats, visit http://switchfromshapefile.org/.]
Despite these limitations, a viable alternative had been missing for a long time. 
In the meantime, [GeoPackage](https://www.geopackage.org/)\index{GeoPackage} emerged, and seems to be a more than suitable replacement candidate for ESRI Shapefile.
Geopackage is a format for exchanging geospatial information and an OGC standard. 
The GeoPackage standard describes the rules on how to store geospatial information in a tiny SQLite container.
Hence, GeoPackage is a lightweight spatial database container, which allows the storage of vector and raster data but also of non-spatial data and extensions.
Aside from GeoPackage, there are other geospatial data exchange formats worth checking out (Table \@ref(tab:formats)).

## Data input (I) {#data-input}

Executing commands such as `sf::st_read()` (the main function we use for loading vector data) or `raster::raster()` (the main function used for loading raster data) silently sets off a chain of events that reads data from files.
Moreover, there are many R packages containing a wide range of geographic data or providing simple access to different data sources.
All of them load the data into R or, more precisely, assign objects to your workspace, stored in RAM accessible from the [`.GlobalEnv`](http://adv-r.had.co.nz/Environments.html) of the R session.

### Vector data

\index{vector!data input}
Spatial vector data comes in a wide variety of file formats, most of which can be read-in via the **sf** function `st_read()`.
Behind the scenes this calls GDAL\index{GDAL}.
To find out which data formats **sf** supports, run `st_drivers()`. 
Here, we show only the first five drivers (see Table \@ref(tab:drivers)):


```r
sf_drivers = st_drivers()
head(sf_drivers, n = 5)
```

<table>
<caption>(\#tab:drivers)Sample of available drivers for reading/writing vector data (it could vary between different GDAL versions).</caption>
 <thead>
  <tr>
   <th style="text-align:left;">   </th>
   <th style="text-align:left;"> name </th>
   <th style="text-align:left;"> long_name </th>
   <th style="text-align:left;"> write </th>
   <th style="text-align:left;"> copy </th>
   <th style="text-align:left;"> is_raster </th>
   <th style="text-align:left;"> is_vector </th>
   <th style="text-align:left;"> vsi </th>
  </tr>
 </thead>
<tbody>
  <tr>
   <td style="text-align:left;"> ESRI Shapefile </td>
   <td style="text-align:left;width: 7em; "> ESRI Shapefile </td>
   <td style="text-align:left;"> ESRI Shapefile </td>
   <td style="text-align:left;"> TRUE </td>
   <td style="text-align:left;"> FALSE </td>
   <td style="text-align:left;"> FALSE </td>
   <td style="text-align:left;"> TRUE </td>
   <td style="text-align:left;"> TRUE </td>
  </tr>
  <tr>
   <td style="text-align:left;"> GPX </td>
   <td style="text-align:left;width: 7em; "> GPX </td>
   <td style="text-align:left;"> GPX </td>
   <td style="text-align:left;"> TRUE </td>
   <td style="text-align:left;"> FALSE </td>
   <td style="text-align:left;"> FALSE </td>
   <td style="text-align:left;"> TRUE </td>
   <td style="text-align:left;"> TRUE </td>
  </tr>
  <tr>
   <td style="text-align:left;"> KML </td>
   <td style="text-align:left;width: 7em; "> KML </td>
   <td style="text-align:left;"> Keyhole Markup Language (KML) </td>
   <td style="text-align:left;"> TRUE </td>
   <td style="text-align:left;"> FALSE </td>
   <td style="text-align:left;"> FALSE </td>
   <td style="text-align:left;"> TRUE </td>
   <td style="text-align:left;"> TRUE </td>
  </tr>
  <tr>
   <td style="text-align:left;"> GeoJSON </td>
   <td style="text-align:left;width: 7em; "> GeoJSON </td>
   <td style="text-align:left;"> GeoJSON </td>
   <td style="text-align:left;"> TRUE </td>
   <td style="text-align:left;"> FALSE </td>
   <td style="text-align:left;"> FALSE </td>
   <td style="text-align:left;"> TRUE </td>
   <td style="text-align:left;"> TRUE </td>
  </tr>
  <tr>
   <td style="text-align:left;"> GPKG </td>
   <td style="text-align:left;width: 7em; "> GPKG </td>
   <td style="text-align:left;"> GeoPackage </td>
   <td style="text-align:left;"> TRUE </td>
   <td style="text-align:left;"> TRUE </td>
   <td style="text-align:left;"> TRUE </td>
   <td style="text-align:left;"> TRUE </td>
   <td style="text-align:left;"> TRUE </td>
  </tr>
</tbody>
</table>

<!-- One of the major advantages of **sf** is that it is fast. -->
<!-- reference to the vignette -->
The first argument of `st_read()` is `dsn`, which should be a text string or an object containing a single text string.
The content of a text string could vary between different drivers.
In most cases, as with the ESRI Shapefile\index{Shapefile} (`.shp`) or the `GeoPackage`\index{GeoPackage} format (`.gpkg`), the `dsn` would be a file name.
`st_read()` guesses the driver based on the file extension, as illustrated for a `.gpkg` file below:


```r
vector_filepath = system.file("shapes/world.gpkg", package = "spData")
world = st_read(vector_filepath)
#> Reading layer `world' from data source `.../world.gpkg' using driver `GPKG'
#> Simple feature collection with 177 features and 10 fields
#> geometry type:  MULTIPOLYGON
#> dimension:      XY
#> bbox:           xmin: -180 ymin: -90 xmax: 180 ymax: 83.64513
#> epsg (SRID):    4326
#> proj4string:    +proj=longlat +datum=WGS84 +no_defs
```


For some drivers, `dsn` could be provided as a folder name, access credentials for a database, or a GeoJSON string representation (see the examples of the `st_read()` help page for more details).

Some vector driver formats can store multiple data layers.
By default, `st_read()` automatically reads the first layer of the file specified in `dsn`; however, using the `layer` argument you can specify any other layer.

Naturally, some options are specific to certain drivers.^[
A list of supported vector formats and options can be found at http://gdal.org/ogr_formats.html.
]
For example, think of coordinates stored in a spreadsheet format (`.csv`).
To read in such files as spatial objects, we naturally have to specify the names of the columns (`X` and `Y` in our example below) representing the coordinates.
We can do this with the help of the `options` parameter.
To find out about possible options, please refer to the 'Open Options' section of the corresponding GDAL\index{GDAL} driver description.
For the comma-separated value (csv) format, visit http://www.gdal.org/drv_csv.html.


```r
cycle_hire_txt = system.file("misc/cycle_hire_xy.csv", package = "spData")
cycle_hire_xy = st_read(cycle_hire_txt, options = c("X_POSSIBLE_NAMES=X",
                                                    "Y_POSSIBLE_NAMES=Y"))
```

Instead of columns describing xy-coordinates, a single column can also contain the geometry information.
Well-known text (WKT)\index{well-known text}, well-known binary (WKB)\index{well-known binary}, and the GeoJSON formats are examples of this.
For instance, the `world_wkt.csv` file has a column named `WKT` representing polygons of the world's countries.
We will again use the `options` parameter to indicate this.
Here, we will use `read_sf()`, the tidyverse-flavoured version of `st_read()`: strings are parsed as characters instead of factors and the resulting data frame is a [tibble](https://r4ds.had.co.nz/tibbles.html). The driver name is also not printed to the console.


```r
world_txt = system.file("misc/world_wkt.csv", package = "spData")
world_wkt = read_sf(world_txt, options = "GEOM_POSSIBLE_NAMES=WKT")
# the same as
world_wkt = st_read(world_txt, options = "GEOM_POSSIBLE_NAMES=WKT", 
                    quiet = TRUE, stringsAsFactors = FALSE, as_tibble = TRUE)
```

\BeginKnitrBlock{rmdnote}<div class="rmdnote">Not all of the supported vector file formats store information about their coordinate reference system.
In these situations, it is possible to add the missing information using the `st_set_crs()` function.
Please refer also to Section \@ref(crs-intro) for more information.</div>\EndKnitrBlock{rmdnote}

As a final example, we will show how `st_read()` also reads KML files.
A KML file stores geographic information in XML format - a data format for the creation of web pages and the transfer of data in an application-independent way [@nolan_xml_2014].
Here, we access a KML file from the web.
This file contains more than one layer.
`st_layers()` lists all available layers.
We choose the first layer `Placemarks` and say so with the help of the `layer` parameter in `read_sf()`.


```r
u = "https://developers.google.com/kml/documentation/KML_Samples.kml"
download.file(u, "KML_Samples.kml")
st_layers("KML_Samples.kml")
#> Driver: LIBKML 
#> Available layers:
#>               layer_name geometry_type features fields
#> 1             Placemarks                      3     11
#> 2      Styles and Markup                      1     11
#> 3       Highlighted Icon                      1     11
#> 4        Ground Overlays                      1     11
#> 5        Screen Overlays                      0     11
#> 6                  Paths                      6     11
#> 7               Polygons                      0     11
#> 8          Google Campus                      4     11
#> 9       Extruded Polygon                      1     11
#> 10 Absolute and Relative                      4     11
kml = read_sf("KML_Samples.kml", layer = "Placemarks")
```

All the examples presented in this section so far have used the **sf** package for geographic data import.
It is fast and flexible but it may be worth looking at other packages for specific file formats.
An example is the **geojsonsf** package.
A [benchmark](https://github.com/ATFutures/geobench) suggests it is around 10 times faster than the **sf** package for reading `.geojson`.

### Raster data

\index{raster!data input}
Similar to vector data, raster data comes in many file formats with some of them supporting even multilayer files.
**raster**'s `raster()` command reads in a single layer.


```r
raster_filepath = system.file("raster/srtm.tif", package = "spDataLarge")
single_layer = raster(raster_filepath)
```

In case you want to read in a single band from a multilayer file, use the `band` parameter to indicate a specific layer.


```r
multilayer_filepath = system.file("raster/landsat.tif", package = "spDataLarge")
band3 = raster(multilayer_filepath, band = 3)
```

If you want to read in all bands, use `brick()` or `stack()`.


```r
multilayer_brick = brick(multilayer_filepath)
multilayer_stack = stack(multilayer_filepath)
```

Please refer to Section \@ref(raster-classes) for information on the difference between raster stacks and bricks.

<!-- ### Databases -->
<!-- postgis input example -->

## Data output (O) {#data-output}

Writing geographic data allows you to convert from one format to another and to save newly created objects.
Depending on the data type (vector or raster), object class (e.g., `multipoint` or `RasterLayer`), and type and amount of stored information (e.g., object size, range of values), it is important to know how to store spatial files in the most efficient way.
The next two sections will demonstrate how to do this.

### Vector data

\index{vector!data output}


The counterpart of `st_read()` is `st_write()`.
It allows you to write **sf** objects to a wide range of geographic vector file formats, including the most common such as `.geojson`, `.shp` and `.gpkg`.
Based on the file name, `st_write()` decides automatically which driver to use. 
The speed of the writing process depends also on the driver.


```r
st_write(obj = world, dsn = "world.gpkg")
#> Writing layer `world' to data source `world.gpkg' using driver `GPKG'
#> Writing 177 features with 10 fields and geometry type Multi Polygon.
```

**Note**: if you try to write to the same data source again, the function will fail:


```r
st_write(obj = world, dsn = "world.gpkg")
#> Layer world in dataset world.gpkg already exists:
#> use either append=TRUE to append to layer or append=FALSE to overwrite layer
#> Error in CPL_write_ogr(obj, dsn, layer, driver, as.character(dataset_options), : Dataset already exists.
```

The error message tells us why the function failed:
`world.gpkg` already contains a layer called `world`. 
To fix this problem, use the `append` argument.
When this argument is set to `TRUE`, the old layer is kept and the `world` object is added as a second new layer:


```r
st_write(obj = world, dsn = "world.gpkg", append = TRUE)
```

Alternatively with `append = FALSE`, existing layers are deleted before the function attempts to write our object to a file:


```r
st_write(obj = world, dsn = "world.gpkg", append = FALSE)
```

You can achieve the same with `write_sf()` since it is equivalent to (technically an *alias* for) `st_write()`, except that its defaults for `append` is `FALSE` and `quiet` is `TRUE`.


```r
write_sf(obj = world, dsn = "world.gpkg")
```

The `layer_options` argument could be also used for many different purposes.
One of them is to write spatial data to a text file.
This can be done by specifying `GEOMETRY` inside of `layer_options`. 
It could be either `AS_XY` for simple point datasets (it creates two new columns for coordinates) or `AS_WKT` for more complex spatial data (one new column is created which contains the well-known text representation of spatial objects).


```r
st_write(cycle_hire_xy, "cycle_hire_xy.csv", layer_options = "GEOMETRY=AS_XY")
st_write(world_wkt, "world_wkt.csv", layer_options = "GEOMETRY=AS_WKT")
```

### Raster data

\index{raster!data output}
The `writeRaster()` function saves `Raster*` objects to files on disk. 
The function expects input regarding output data type and file format, but also accepts GDAL options specific to a selected file format (see `?writeRaster` for more details).

\index{raster!data types}
The **raster** package offers nine data types when saving a raster: LOG1S, INT1S, INT1U, INT2S, INT2U, INT4S, INT4U, FLT4S, and FLT8S.^[
Using INT4U is not recommended as R does not support 32-bit unsigned integers.
]
The data type determines the bit representation of the raster object written to disk (Table \@ref(tab:datatypes)).
Which data type to use depends on the range of the values of your raster object.
The more values a data type can represent, the larger the file will get on disk.
Commonly, one would use LOG1S for bitmap (binary) rasters.
Unsigned integers (INT1U, INT2U, INT4U) are suitable for categorical data, while float numbers (FLT4S and FLT8S) usually represent continuous data.
`writeRaster()` uses FLT4S as the default.
While this works in most cases, the size of the output file will be unnecessarily large if you save binary or categorical data.
Therefore, we would recommend to use the data type that needs the least storage space, but is still able to represent all values (check the range of values with the `summary()` function).

<table>
<caption>(\#tab:datatypes)Data types supported by the raster package.</caption>
 <thead>
  <tr>
   <th style="text-align:left;"> Data type </th>
   <th style="text-align:left;"> Minimum value </th>
   <th style="text-align:left;"> Maximum value </th>
  </tr>
 </thead>
<tbody>
  <tr>
   <td style="text-align:left;"> LOG1S </td>
   <td style="text-align:left;"> FALSE (0) </td>
   <td style="text-align:left;"> TRUE (1) </td>
  </tr>
  <tr>
   <td style="text-align:left;"> INT1S </td>
   <td style="text-align:left;"> -127 </td>
   <td style="text-align:left;"> 127 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> INT1U </td>
   <td style="text-align:left;"> 0 </td>
   <td style="text-align:left;"> 255 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> INT2S </td>
   <td style="text-align:left;"> -32,767 </td>
   <td style="text-align:left;"> 32,767 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> INT2U </td>
   <td style="text-align:left;"> 0 </td>
   <td style="text-align:left;"> 65,534 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> INT4S </td>
   <td style="text-align:left;"> -2,147,483,647 </td>
   <td style="text-align:left;"> 2,147,483,647 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> INT4U </td>
   <td style="text-align:left;"> 0 </td>
   <td style="text-align:left;"> 4,294,967,296 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> FLT4S </td>
   <td style="text-align:left;"> -3.4e+38 </td>
   <td style="text-align:left;"> 3.4e+38 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> FLT8S </td>
   <td style="text-align:left;"> -1.7e+308 </td>
   <td style="text-align:left;"> 1.7e+308 </td>
  </tr>
</tbody>
</table>

By default, `writeRaster()` saves outputs in its native format as `.grd` files, when a file extension is invalid or missing.
Other file formats can be specified by changing the extension of the output file name.
Naming a file `*.tif` will create a GeoTIFF file, as demonstrated below:


```r
writeRaster(single_layer, filename = "my_raster.tif", datatype = "INT2U")
```

Some raster file formats have additional options, that can be set by providing [GDAL parameters](http://www.gdal.org/formats_list.html) to the `options` argument of `writeRaster()`.
GeoTIFF files, for example, can be compressed using `COMPRESS`:
<!-- GeoTIFF files, for example, can be compressed using the `COMPRESS` option^[Find out about GeoTIFF options under http://www.gdal.org/frmt_gtiff.html.]: -->



```r
writeRaster(x = single_layer,
            filename = "my_raster.tif",
            datatype = "INT2U",
            options = c("COMPRESS=DEFLATE"),
            overwrite = TRUE)
```

Note that `writeFormats()` returns a list with all supported file formats on your computer.

<!-- ### Databases -->
<!-- postgis output example -->

## Visual outputs

\index{map making!outputs}
R supports many different static and interactive graphics formats.
The most general method to save a static plot is to open a graphic device, create a plot, and close it, for example:


```r
png(filename = "lifeExp.png", width = 500, height = 350)
plot(world["lifeExp"])
dev.off()
```

Other available graphic devices include `pdf()`, `bmp()`, `jpeg()`, and `tiff()`. 
You can specify several properties of the output plot, including width, height and resolution.

Additionally, several graphic packages provide their own functions to save a graphical output.
For example, the **tmap** package has the `tmap_save()` function.
You can save a `tmap` object to different graphic formats by specifying the object name and a file path to a new graphic file.


```r
library(tmap)
tmap_obj = tm_shape(world) + tm_polygons(col = "lifeExp")
tmap_save(tm = tmap_obj, filename = "lifeExp_tmap.png")
```

<!-- Note about that the `plot` function do not create an object -->
<!-- ```{r} -->
<!-- a = plot(world["lifeExp"]) -->
<!-- ``` -->

On the other hand, you can save interactive maps created in the `mapview` package as an HTML file or image using the `mapshot()` function:

<!-- example doesn't work, problem with colors I guess -->

```r
library(mapview)
mapview_obj = mapview(world, zcol = "lifeExp", legend = TRUE)
mapshot(mapview_obj, file = "my_interactive_map.html")
```

## Exercises

1. List and describe three types of vector, raster, and geodatabase formats.

1. Name at least two differences between `read_sf()` and the more well-known function `st_read()`.

1. Read the `cycle_hire_xy.csv` file from the **spData** package as a spatial object (Hint: it is located in the `misc\` folder).
What is a geometry type of the loaded object? 

1. Download the borders of Germany using **rnaturalearth**, and create a new object called `germany_borders`.
Write this new object to a file of the GeoPackage format.

1. Download the global monthly minimum temperature with a spatial resolution of five minutes using the **raster** package.
Extract the June values, and save them to a file named `tmin_june.tif` file (hint: use `raster::subset()`).

1. Create a static map of Germany's borders, and save it to a PNG file.

1. Create an interactive map using data from the `cycle_hire_xy.csv` file. 
Export this map to a file called `cycle_hire.html`.
