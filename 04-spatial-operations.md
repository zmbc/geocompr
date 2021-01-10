# Spatial data operations {#spatial-operations}

## Prerequisites {-}

- This chapter requires the same packages used in Chapter \@ref(attr): 


```r
library(sf)
library(raster)
library(dplyr)
library(spData)
```

## Introduction

Spatial operations are a vital part of geocomputation\index{geocomputation}.
This chapter shows how spatial objects can be modified in a multitude of ways based on their location and shape.
The content builds on the previous chapter because many spatial operations have a non-spatial (attribute) equivalent.
This is especially true for *vector* operations: Section \@ref(vector-attribute-manipulation) on vector attribute manipulation provides the basis for understanding its spatial counterpart, namely spatial subsetting (covered in Section \@ref(spatial-subsetting)).
Spatial joining (Section \@ref(spatial-joining)) and aggregation (Section \@ref(spatial-aggr)) also have non-spatial counterparts, covered in the previous chapter.

Spatial operations differ from non-spatial operations in some ways, however.
To illustrate the point, imagine you are researching road safety.
Spatial joins can be used to find road speed limits related with administrative zones, even when no zone ID is provided.
But this raises the question: should the road completely fall inside a zone for its values to be joined?
Or is simply crossing or being within a certain distance sufficient?
When posing such questions, it becomes apparent that spatial operations differ substantially from attribute operations on data frames:
the *type* of spatial relationship between objects must be considered.
These are covered in Section \@ref(topological-relations), on topological relations.

\index{spatial operations}
Another unique aspect of spatial objects is distance.
All spatial objects are related through space and distance calculations, covered in Section \@ref(distance-relations), can be used to explore the strength of this relationship.

Spatial operations also apply to raster objects.
Spatial subsetting of raster objects is covered in Section \@ref(spatial-raster-subsetting); merging several raster 'tiles' into a single object is covered in Section \@ref(merging-rasters).
For many applications, the most important spatial operation on raster objects is *map algebra*, as we will see in Sections \@ref(map-algebra) to \@ref(global-operations-and-distances).
Map algebra is also the prerequisite for distance calculations on rasters, a technique which is covered in Section \@ref(global-operations-and-distances).

\BeginKnitrBlock{rmdnote}<div class="rmdnote">It is important to note that spatial operations that use two spatial objects rely on both objects having the same coordinate reference system, a topic that was introduced in Section \@ref(crs-intro) and which will be covered in more depth in Chapter \@ref(reproj-geo-data).</div>\EndKnitrBlock{rmdnote}

## Spatial operations on vector data {#spatial-vec}

This section provides an overview of spatial operations on vector geographic data represented as simple features in the **sf** package before Section \@ref(spatial-ras), which presents spatial methods using the **raster** package.

### Spatial subsetting

Spatial subsetting is the process of selecting features of a spatial object based on whether or not they in some way *relate* in space to another object.
It is analogous to *attribute subsetting* (covered in Section \@ref(vector-attribute-subsetting)) and can be done with the base R square bracket (`[`) operator or with the `filter()` function from the **tidyverse**\index{tidyverse (package)}.
\index{vector!subsetting}
\index{spatial!subsetting}

An example of spatial subsetting is provided by the `nz` and `nz_height` datasets in **spData**.
These contain projected data on the 16 main regions and 101 highest points in New Zealand, respectively (Figure \@ref(fig:nz-subset)).
The following code chunk first creates an object representing Canterbury, then uses spatial subsetting to return all high points in the region:


```r
canterbury = nz %>% filter(Name == "Canterbury")
canterbury_height = nz_height[canterbury, ]
```

<div class="figure" style="text-align: center">
<img src="04-spatial-operations_files/figure-html/nz-subset-1.png" alt="Illustration of spatial subsetting with red triangles representing 101 high points in New Zealand, clustered near the central Canterbuy region (left). The points in Canterbury were created with the `[` subsetting operator (highlighted in gray, right)." width="100%" />
<p class="caption">(\#fig:nz-subset)Illustration of spatial subsetting with red triangles representing 101 high points in New Zealand, clustered near the central Canterbuy region (left). The points in Canterbury were created with the `[` subsetting operator (highlighted in gray, right).</p>
</div>

Like attribute subsetting `x[y, ]` subsets features of a *target* `x` using the contents of a *source* object `y`.
Instead of `y` being of class `logical` or `integer` --- a vector of `TRUE` and `FALSE` values or whole numbers --- for spatial subsetting it is another spatial (`sf`) object.

Various *topological relations* can be used for spatial subsetting.
These determine the type of spatial relationship that features in the target object must have with the subsetting object to be selected, including *touches*, *crosses* or *within* (see Section \@ref(topological-relations)). 
*Intersects* is the default spatial subsetting operator, a default that returns `TRUE` for many types of spatial relations, including *touches*, *crosses* and *is within*.
These alternative spatial operators can be specified with the `op =` argument, a third argument that can be passed to the `[` operator for `sf` objects.
This is demonstrated in the following command which returns the opposite of `st_intersects()`, points that do not intersect with Canterbury (see Section \@ref(topological-relations)):


```r
nz_height[canterbury, , op = st_disjoint]
```

\BeginKnitrBlock{rmdnote}<div class="rmdnote">Note the empty argument --- denoted with `, ,` --- in the preceding code chunk is included to highlight `op`, the third argument in `[` for `sf` objects.
One can use this to change the subsetting operation in many ways.
`nz_height[canterbury, 2, op = st_disjoint]`, for example, returns the same rows but only includes the second attribute column (see `` sf:::`[.sf` `` and the `?sf` for details).</div>\EndKnitrBlock{rmdnote}

For many applications, this is all you'll need to know about spatial subsetting for vector data.
In this case, you can safely skip to Section \@ref(topological-relations).

If you're interested in the details, including other ways of subsetting, read on.
Another way of doing spatial subsetting uses objects returned by *topological operators*.
This is demonstrated in the first command below:


```r
sel_sgbp = st_intersects(x = nz_height, y = canterbury)
class(sel_sgbp)
#> [1] "sgbp" "list"
sel_logical = lengths(sel_sgbp) > 0
canterbury_height2 = nz_height[sel_logical, ]
```

In the above code chunk, an object of class `sgbp` (a sparse geometry binary predicate, a list of length `x` in the spatial operation) is created and then converted into a logical vector `sel_logical` (containing only `TRUE` and `FALSE` values).
\index{binary predicate|seealso {topological relations}}
The function `lengths()` identifies which features in `nz_height` intersect with *any* objects in `y`.
In this case 1 is the greatest possible value but for more complex operations one could use the method to subset only features that intersect with, for example, 2 or more features from the source object.

\BeginKnitrBlock{rmdnote}<div class="rmdnote">Note: another way to return a logical output is by setting `sparse = FALSE` (meaning 'return a dense matrix not a sparse one') in operators such as `st_intersects()`. The command `st_intersects(x = nz_height, y = canterbury, sparse = FALSE)[, 1]`, for example, would return an output identical to `sel_logical`.
Note: the solution involving `sgbp` objects is more generalisable though, as it works for many-to-many operations and has lower memory requirements.</div>\EndKnitrBlock{rmdnote}

It should be noted that a logical  can also be used with `filter()` as follows (`sparse = FALSE` is explained in Section \@ref(topological-relations)):


```r
canterbury_height3 = nz_height %>%
  filter(st_intersects(x = ., y = canterbury, sparse = FALSE))
```

At this point, there are three versions of `canterbury_height`, one created with spatial subsetting directly and the other two via intermediary selection objects.
To explore these objects and spatial subsetting in more detail, see the supplementary vignettes on `subsetting` and [`tidverse-pitfalls`](https://geocompr.github.io/geocompkg/articles/).

### Topological relations

<!-- http://lin-ear-th-inking.blogspot.com/2007/06/subtleties-of-ogc-covers-spatial.html -->
<!-- https://edzer.github.io/sfr/articles/sf3.html -->
<!-- https://github.com/edzer/sfr/wiki/migrating#relevant-commands-exported-by-rgeos -->
<!-- Relations and inverse relations -->
<!-- http://desktop.arcgis.com/en/arcmap/latest/extensions/data-reviewer/types-of-spatial-relationships-that-can-be-validated.htm -->
<!-- Topological relations: + difference between datatypes -->
<!-- ?geos_binary_pred -->
<!-- Distance relations -->
<!-- Subset (1) points in polygons <-> (2) -->
Topological relations describe the spatial relationships between objects.
To understand them, it helps to have some simple test data to work with.
Figure \@ref(fig:relation-objects) contains a polygon (`a`), a line (`l`) and some points (`p`), which are created in the code below.
\index{topological relations}


```r
# create a polygon
a_poly = st_polygon(list(rbind(c(-1, -1), c(1, -1), c(1, 1), c(-1, -1))))
a = st_sfc(a_poly)
# create a line
l_line = st_linestring(x = matrix(c(-1, -1, -0.5, 1), ncol = 2))
l = st_sfc(l_line)
# create points
p_matrix = matrix(c(0.5, 1, -1, 0, 0, 1, 0.5, 1), ncol = 2)
p_multi = st_multipoint(x = p_matrix)
p = st_cast(st_sfc(p_multi), "POINT")
```

<div class="figure" style="text-align: center">
<img src="04-spatial-operations_files/figure-html/relation-objects-1.png" alt="Points (p 1 to 4), line and polygon objects arranged to illustrate topological relations." width="50%" />
<p class="caption">(\#fig:relation-objects)Points (p 1 to 4), line and polygon objects arranged to illustrate topological relations.</p>
</div>

A simple query is: which of the points in `p` intersect in some way with polygon `a`?
The question can be answered by inspection (points 1 and 2 are over or touch the triangle).
It can also be answered by using a *spatial predicate* such as *do the objects intersect*?
This is implemented in **sf** as follows:


```r
st_intersects(p, a)
#> Sparse geometry binary ..., where the predicate was `intersects'
#> 1: 1
#> 2: 1
#> 3: (empty)
#> 4: (empty)
```

The contents of the result should be as you expected:
the function returns a positive (`1`) result for the first two points, and a negative result (represented by an empty vector) for the last two.
What may be unexpected is that the result comes in the form of a list of vectors.
This *sparse matrix* output only registers a relation if one exists, reducing the memory requirements of topological operations on multi-feature objects.
As we saw in the previous section, a *dense matrix* consisting of `TRUE` or `FALSE` values for each combination of features can also be returned when `sparse = FALSE`:


```r
st_intersects(p, a, sparse = FALSE)
#>       [,1]
#> [1,]  TRUE
#> [2,]  TRUE
#> [3,] FALSE
#> [4,] FALSE
```

The output is a matrix in which each row represents a feature in the target object and each column represents a feature in the selecting object.
In this case, only the first two features in `p` intersect with `a` and there is only one feature in `a` so the result has only one column.
The result can be used for subsetting as we saw in Section \@ref(spatial-subsetting).

Note that `st_intersects()` returns `TRUE` for the second feature in the object `p` even though it just touches the polygon `a`: *intersects* is a 'catch-all' topological operation which identifies many types of spatial relation.

The opposite of `st_intersects()` is `st_disjoint()`, which returns only objects that do not spatially relate in any way to the selecting object (note `[, 1]` converts the result into a vector):


```r
st_disjoint(p, a, sparse = FALSE)[, 1]
#> [1] FALSE FALSE  TRUE  TRUE
```

`st_within()` returns `TRUE` only for objects that are completely within the selecting object.
This applies only to the first object, which is inside the triangular polygon, as illustrated below:


```r
st_within(p, a, sparse = FALSE)[, 1]
#> [1]  TRUE FALSE FALSE FALSE
```

Note that although the first point is *within* the triangle, it does not *touch* any part of its border.
For this reason `st_touches()` only returns `TRUE` for the second point:


```r
st_touches(p, a, sparse = FALSE)[, 1]
#> [1] FALSE  TRUE FALSE FALSE
```

What about features that do not touch, but *almost touch* the selection object?
These can be selected using `st_is_within_distance()`, which has an additional `dist` argument.
It can be used to set how close target objects need to be before they are selected.
Note that although point 4 is one unit of distance from the nearest node of `a` (at point 2 in Figure \@ref(fig:relation-objects)), it is still selected when the distance is set to 0.9.
This is illustrated in the code chunk below, the second line of which converts the lengthy list output into a `logical` object:


```r
sel = st_is_within_distance(p, a, dist = 0.9) # can only return a sparse matrix
lengths(sel) > 0
#> [1]  TRUE  TRUE FALSE  TRUE
```


\BeginKnitrBlock{rmdnote}<div class="rmdnote">Functions for calculating topological relations use spatial indices to largely speed up spatial query performance.
They achieve that using the Sort-Tile-Recursive (STR) algorithm.
The `st_join` function, mentioned in the next section, also uses the spatial indexing. 
You can learn more at https://www.r-spatial.org/r/2017/06/22/spatial-index.html.</div>\EndKnitrBlock{rmdnote}







<!-- Equals: -->
<!-- https://postgis.net/docs/ST_Equals.html -->

<!-- ```{r, eval=FALSE} -->
<!-- st_equals(a, b, sparse = FALSE) -->
<!-- ``` -->

<!-- Contains: -->
<!-- https://postgis.net/docs/ST_Contains.html -->
<!-- https://postgis.net/docs/ST_ContainsProperly.html -->

<!-- ```{r, eval=FALSE} -->
<!-- st_contains(a, b, sparse = FALSE) -->
<!-- st_contains_properly(a, b, sparse = FALSE) -->
<!-- ``` -->

<!-- Covers: -->
<!-- https://postgis.net/docs/ST_Covers.html -->
<!-- https://postgis.net/docs/ST_CoveredBy.html -->

<!-- ```{r, eval=FALSE} -->
<!-- st_covers(a, b, sparse = FALSE) -->
<!-- st_covered_by(a, b, sparse = FALSE) -->
<!-- ``` -->

<!-- Within: -->
<!-- https://postgis.net/docs/ST_Within.html -->

<!-- ```{r, eval=FALSE} -->
<!-- st_within(a, b, sparse = FALSE) -->
<!-- ``` -->

<!-- Overlaps: -->
<!-- https://postgis.net/docs/ST_Overlaps.html -->

<!-- ```{r, eval=FALSE} -->
<!-- st_overlaps(a, b, sparse = FALSE) -->
<!-- ``` -->

<!-- Intersects: -->
<!-- https://postgis.net/docs/ST_Intersects.html -->

<!-- ```{r, eval=FALSE} -->
<!-- st_intersects(a, b, sparse = FALSE) -->
<!-- ``` -->

<!-- Disjoint: -->
<!-- https://postgis.net/docs/ST_Disjoint.html -->

<!-- ```{r, eval=FALSE} -->
<!-- st_disjoint(a, b, sparse = FALSE) -->
<!-- ``` -->

<!-- Touches: -->
<!-- https://postgis.net/docs/ST_Touches.html -->

<!-- ```{r, eval=FALSE} -->
<!-- st_touches(a, b, sparse = FALSE) -->
<!-- ``` -->

<!-- Crosses: -->
<!-- https://postgis.net/docs/ST_Crosses.html -->

<!-- ```{r, eval=FALSE} -->
<!-- st_crosses(a, b, sparse = FALSE) -->
<!-- ``` -->

<!-- DE9-IM - https://en.wikipedia.org/wiki/DE-9IM -->
<!-- https://edzer.github.io/sfr/reference/st_relate.html -->

<!-- ```{r, eval=FALSE} -->
<!-- st_relate(a, b, sparse = FALSE) -->
<!-- ``` -->

<!-- examples (points/polygons) -->
<!-- examples (points/lines) -->
<!-- examples (lines/polygons) -->

<!-- TODO? create a series of polygons distributed evenly over the surface of the Earth and clip them. -->
<!-- set.seed(2018) -->
<!-- blob_points = st_sample(x = world, size = 2) -->
<!-- blobs = st_buffer(x = blob_points, dist = 1) -->
<!-- plot(blobs) -->

### Spatial joining 

Joining two non-spatial datasets relies on a shared 'key' variable, as described in Section \@ref(vector-attribute-joining).
Spatial data joining applies the same concept, but instead relies on shared areas of geographic space (it is also know as spatial overlay).
As with attribute data, joining adds a new column to the target object (the argument `x` in joining functions), from a source object (`y`).
\index{join!spatial}
\index{spatial!join}

The process can be illustrated by an example.
Imagine you have ten points randomly distributed across the Earth's surface.
Of the points that are on land, which countries are they in?
Random points to demonstrate spatial joining are created as follows:


```r
set.seed(2018) # set seed for reproducibility
(bb_world = st_bbox(world)) # the world's bounds
#>   xmin   ymin   xmax   ymax 
#> -180.0  -90.0  180.0   83.6
random_df = tibble(
  x = runif(n = 10, min = bb_world[1], max = bb_world[3]),
  y = runif(n = 10, min = bb_world[2], max = bb_world[4])
)
random_points = random_df %>% 
  st_as_sf(coords = c("x", "y")) %>% # set coordinates
  st_set_crs(4326) # set geographic CRS
```

<!-- This may seem a trivial question but if you consider being placed somewhere at random it would surely take some time to discover where you were and you'd probably have to ask someone. **comment - removed as it's too long-winded (RL)** -->
The scenario is illustrated in Figure \@ref(fig:spatial-join).
The `random_points` object (top left) has no attribute data, while the `world` (top right) does.
The spatial join operation is done by `st_join()`, which adds the `name_long` variable to the points, resulting in `random_joined` which is illustrated in Figure \@ref(fig:spatial-join) (bottom left --- see [`04-spatial-join.R`](https://github.com/Robinlovelace/geocompr/blob/master/code/04-spatial-join.R)).
Before creating the joined dataset, we use spatial subsetting to create `world_random`, which contains only countries that contain random points, to verify the number of country names returned in the joined dataset should be four (see the top right panel of Figure \@ref(fig:spatial-join)).


```r
world_random = world[random_points, ]
nrow(world_random)
#> [1] 4
random_joined = st_join(random_points, world["name_long"])
```

<div class="figure" style="text-align: center">
<img src="04-spatial-operations_files/figure-html/spatial-join-1.png" alt="Illustration of a spatial join. A new attribute variable is added to random points (top left) from source world object (top right) resulting in the data represented in the final panel." width="100%" />
<p class="caption">(\#fig:spatial-join)Illustration of a spatial join. A new attribute variable is added to random points (top left) from source world object (top right) resulting in the data represented in the final panel.</p>
</div>

By default, `st_join()` performs a left join (see Section \@ref(vector-attribute-joining)), but it can also do inner joins by setting the argument `left = FALSE`.
Like spatial subsetting, the default topological operator used by `st_join()` is `st_intersects()`.
This can be changed with the `join` argument (see `?st_join` for details).
In the example above, we have added features of a polygon layer to a point layer.
In other cases, we might want to join point attributes to a polygon layer.
There might be occasions where more than one point falls inside one polygon. 
In such a case `st_join()` duplicates the polygon feature: it creates a new row for each match.

### Non-overlapping joins

Sometimes two geographic datasets do not touch but still have a strong geographic relationship enabling joins.
The datasets `cycle_hire` and `cycle_hire_osm`, already attached in the **spData** package, provide a good example.
Plotting them shows that they are often closely related but they do not touch, as shown in Figure \@ref(fig:cycle-hire), a base version of which is created with the following code below:
\index{join!non-overlapping}


```r
plot(st_geometry(cycle_hire), col = "blue")
plot(st_geometry(cycle_hire_osm), add = TRUE, pch = 3, col = "red")
```

We can check if any points are the same `st_intersects()` as shown below:


```r
any(st_touches(cycle_hire, cycle_hire_osm, sparse = FALSE))
#> [1] FALSE
```



<div class="figure" style="text-align: center">
<!--html_preserve--><div id="htmlwidget-9580c84d0c32f0a22a8c" style="width:100%;height:415.296px;" class="leaflet html-widget"></div>
<script type="application/json" data-for="htmlwidget-9580c84d0c32f0a22a8c">{"x":{"options":{"crs":{"crsClass":"L.CRS.EPSG3857","code":null,"proj4def":null,"projectedBounds":null,"options":{}}},"calls":[{"method":"addCircles","args":[[51.52916347,51.49960695,51.52128377,51.53005939,51.49313,51.51811784,51.53430039,51.52834133,51.5073853,51.50597426,51.52395143,51.52168078,51.51991453,51.52994371,51.51772703,51.52635795,51.5216612,51.51477076,51.52505093,51.52773634,51.53007835,51.5222641,51.51943538,51.51908011,51.5288338,51.52728093,51.51382102,51.52351808,51.513735,51.52915444,51.52953709,51.52469624,51.5341235,51.50173726,51.49159394,51.4973875,51.5263778,51.52127071,51.52001715,51.53099181,51.52026,51.51073687,51.522511,51.507131,51.52334476,51.51248445,51.50706909,51.52867339,51.52671796,51.52295439,51.5099923,51.52174785,51.51707521,51.52058381,51.52334672,51.52452699,51.53176825,51.52644828,51.49738251,51.4907579,51.53089041,51.50946212,51.52522753,51.51795029,51.51882555,51.52059681,51.5262363,51.53136059,51.5154186,51.52352001,51.52572618,51.48591714,51.53219984,51.52559505,51.52341837,51.52486887,51.50069361,51.52025302,51.514274,51.50963938,51.51593725,51.50064702,51.48947903,51.51646835,51.51858757,51.5262503,51.53301907,51.49368637,51.49889832,51.53440868,51.49506109,51.5208417,51.53095071,51.49792478,51.52554222,51.51457763,51.49043573,51.51155322,51.51340693,51.50472376,51.51159481,51.51552971,51.51410514,51.52600832,51.49812559,51.51563144,51.53304322,51.5100172,51.51580998,51.49646288,51.52451738,51.51423368,51.51449962,51.49288067,51.49582705,51.52589324,51.51573534,51.51891348,51.52111369,51.52836014,51.49654462,51.50069491,51.51782144,51.51700801,51.49536226,51.5118973,51.50950627,51.53300545,51.52364804,51.50136494,51.504904,51.52326004,51.51196176,51.49087475,51.51227622,51.49488108,51.52096262,51.51530805,51.49372451,51.4968865,51.48894022,51.50074359,51.48836528,51.51494305,51.49211134,51.48478899,51.49705603,51.51213691,51.49217002,51.51183419,51.50379168,51.49586666,51.49443626,51.50039792,51.49085368,51.51461995,51.50663341,51.49234577,51.51760685,51.4931848,51.515607,51.517932,51.50185512,51.49395092,51.50040123,51.51474612,51.52784273,51.4916156,51.49121192,51.50486,51.512529,51.52174384,51.51791921,51.49616092,51.48985626,51.5129118,51.49941247,51.52202903,51.52071513,51.48805753,51.51733558,51.49247977,51.5165179,51.53166681,51.48997562,51.50311799,51.51251523,51.50581776,51.50462759,51.50724437,51.50368837,51.50556905,51.51048489,51.51492456,51.5225965,51.51236389,51.51821864,51.52659961,51.518144,51.518154,51.50135267,51.52505151,51.49358391,51.51681444,51.49464523,51.50658458,51.50274025,51.52683806,51.51906932,51.49094565,51.51615461,51.48971651,51.49016361,51.49066456,51.49481649,51.50275704,51.49148474,51.51476963,51.50935342,51.50844614,51.52891573,51.50742485,51.50654321,51.50669284,51.49396755,51.51501025,51.50777049,51.53450449,51.49571828,51.51838043,51.5084448,51.5233534,51.52443845,51.50545935,51.52200801,51.49096258,51.51611887,51.4853572,51.52285301,51.530052,51.50645179,51.49859784,51.489932,51.518908,51.5046364,51.53404294,51.53051587,51.52557531,51.51362054,51.52248185,51.50143293,51.50494561,51.5136846,51.5134891,51.49874469,51.51422502,51.52644342,51.51595344,51.50102668,51.49206037,51.49320445,51.50082346,51.4863434,51.53583617,51.50144456,51.50613324,51.49671237,51.52004497,51.50930161,51.50315739,51.5034938,51.51862243,51.490083,51.49580589,51.51906446,51.49914063,51.5272947,51.52367314,51.49236962,51.50923022,51.51678023,51.48483991,51.49888404,51.49815779,51.488226,51.50029631,51.49363156,51.49612799,51.50227992,51.49398524,51.50501351,51.51475963,51.466907,51.50295379,51.51211869,51.48677988,51.51816295,51.50990837,51.49907558,51.50963123,51.4908679,51.51918144,51.53088935,51.51734403,51.50088934,51.52536703,51.49768448,51.49679128,51.51004801,51.52702563,51.49357351,51.49785559,51.526293,51.523196,51.49652013,51.50908747,51.53266186,51.53114,51.53095,51.53589283,51.51641749,51.52085887,51.49334336,51.50860544,51.505044,51.51196803,51.504942,51.51108452,51.51175646,51.5333196,51.51444134,51.50810309,51.53692216,51.528246,51.48802358,51.50013942,51.51217033,51.51196,51.5017154373867,51.529423,51.486965,51.49459148,51.506767,51.486575,51.494412,51.520994,51.51643491,51.49782999,51.49675303,51.50391972,51.536264,51.5291212008901,51.519656,51.530344,51.5109192966489,51.5173721,51.50024195,51.520205,51.49775,51.515208,51.49980661,51.50402793,51.50194596,51.49188409,51.50535447,51.49559291,51.51431171,51.50686435,51.51953043,51.50935171,51.51310333,51.496481,51.51352755,51.49369988,51.51070161,51.51013066,51.521776,51.51066202,51.49942855,51.51733427,51.524826,51.492462,51.51809,51.51348,51.502319,51.52261762,51.517703,51.528187,51.52289229,51.51689296,51.50204238,51.49418566,51.512303,51.51222,51.51994326,51.493146,51.519968,51.49337264,51.488105,51.485821,51.504043,51.504044,51.49456127,51.490491,51.533379,51.493072,51.51397065,51.499917,51.48902,51.534474,51.496957,51.524564,51.488852,51.51824,51.488124,51.5338,51.483145,51.495656,51.510101,51.515256,51.52568,51.52388,51.528936,51.493381,51.50623,51.511088,51.515975,51.504719,51.505697,51.508447,51.487679,51.49447,51.538071,51.542138,51.504749,51.535179,51.516196,51.516,51.541603,51.534776,51.51417,51.53558,51.534464,51.523538,51.521564,51.513757,51.530535,51.533283,51.529452,51.496137,51.498125,51.499041,51.489096,51.49109,51.521889,51.527152,51.5128711,51.487129,51.509843,51.51328,51.528828,51.531127,51.525645,51.511811,51.506946,51.513074,51.508622,51.522507,51.524677,51.50196,51.528169,51.52512,51.527058,51.519265,51.522561,51.502635,51.520893,51.496454,51.520398,51.517475,51.528692,51.519362,51.517842,51.509303,51.511066,51.532091,51.5142228,51.516204,51.500088,51.5112,51.532513,51.528224,51.518811,51.526041,51.534137,51.525941,51.51549,51.511654,51.504714,51.503143,51.503802,51.507326,51.486892,51.508896,51.530326,51.50357,51.528222,51.531864,51.537349,51.51793,51.508981,51.531091,51.528302,51.506613,51.514115,51.509591,51.526153,51.539957,51.517428,51.509474,51.498386,51.49605,51.521564,51.503447,51.511542,51.535678,51.513548,51.502661,51.51601,51.493978,51.501391,51.511624,51.518369,51.51746,51.499286,51.509943,51.521905,51.509158,51.51616,51.53213,51.503083,51.506256,51.539099,51.485587,51.53356,51.497304,51.528869,51.527607,51.511246,51.48692917,51.497622,51.51244,51.490645,51.50964,51.52959,51.487196,51.53256,51.506093,51.5171,51.531066,51.513875,51.493267,51.472817,51.473471,51.494499,51.485743,51.481747,51.514767,51.472993,51.477839,51.538792,51.520331,51.504199,51.468814,51.491093,51.46512358,51.48256792,51.50646524,51.46925984,51.50403821,51.47817208,51.4768851,51.48102131,51.47107905,51.47625965,51.4737636,51.47518024,51.46086446,51.50748124,51.46193072,51.4729184,51.47761941,51.48438657,51.4687905,51.46881971,51.45995384,51.47073264,51.4795017,51.47053858,51.48959104,51.49434708,51.48606206,51.46706414,51.45787019,51.46663393,51.48357068,51.47286577,51.49824168,51.46916161,51.5190427,51.48373225,51.50173215,51.49886563,51.50035306,51.46904022,51.48180515,51.51687069,51.48089844,51.51148696,51.47084722,51.48267821,51.48294452,51.46841875,51.4996806,51.47729232,51.46437067,51.49087074,51.51632095,51.48498496,51.51323001,51.474376,51.46108367,51.49610093,51.5015946,51.49422354,51.47614939,51.46718562,51.475089,51.4646884,51.46517078,51.53546778,51.46348914,51.47787084,51.46866929,51.46231278,51.47453545,51.47768469,51.47303687,51.48810829,51.46506424,51.45475251,51.46067005,51.48814438,51.49760804,51.46095151,51.45922541,51.47047503,51.47946386,51.54211855,51.464786,51.53638435,51.48728535,51.53639219,51.53908372,51.5366541,51.47696496,51.47287627,51.52868155,51.51505991,51.45682071,51.45971528,51.50215353,51.49021762,51.46161068,51.46321128,51.47439218,51.48335692,51.51854104,51.54100708,51.47311696,51.51563007,51.51542791,51.53658514,51.53571683,51.53642464,51.48724429,51.53603947,51.52458353,51.46822047,51.45799126,51.47732253,51.47505096,51.47569809,51.46199911,51.47893931,51.49208492,51.47727637,51.50630441,51.50542628,51.46230566,51.46489445,51.47993289,51.47514228,51.48176572,51.51092871,51.5129814,51.51510818,51.46079243,51.46745485,51.47816972,51.4795738,51.48796408,51.53727795,51.53932857,51.4710956,51.51612862,51.45816465,51.49263658,51.5129006,51.48512191,51.47515398,51.48795853,51.51787005,51.52456169,51.52526975,51.48321729,51.50070305,51.52059714,51.46239255,51.46760141,51.45752945,51.45705988,51.46134382,51.473611,51.491026,51.509224,51.47250956,51.511891,51.470131,51.496664,51.460333,51.4619230679],[-0.109970527,-0.197574246,-0.084605692,-0.120973687,-0.156876,-0.144228881,-0.1680743,-0.170134484,-0.096440751,-0.092754157,-0.122502346,-0.130431727,-0.136039674,-0.123616824,-0.127854211,-0.125979294,-0.109006325,-0.12221963,-0.131161087,-0.135273468,-0.13884627,-0.114079481,-0.119123345,-0.124678402,-0.132250369,-0.11829517,-0.107927706,-0.143613641,-0.193487,-0.093421615,-0.083353323,-0.084439283,-0.129386874,-0.184980612,-0.192369256,-0.197245586,-0.078130921,-0.0755789,-0.083911168,-0.093903825,-0.157183945,-0.144165239,-0.162298,-0.06691,-0.183846408,-0.099141408,-0.145904427,-0.087459376,-0.104298194,-0.094934859,-0.143495266,-0.094475072,-0.086685542,-0.154701411,-0.120202614,-0.079248081,-0.114329032,-0.172190727,-0.089446947,-0.106323685,-0.089782579,-0.124749274,-0.13518856,-0.108657431,-0.108028472,-0.116688468,-0.134407652,-0.117069978,-0.098850915,-0.108340165,-0.088486188,-0.124469948,-0.105480698,-0.144083893,-0.124121774,-0.099489485,-0.102091246,-0.141327271,-0.111257,-0.131510949,-0.111778348,-0.078600401,-0.115156562,-0.079684557,-0.132053392,-0.123509611,-0.139174593,-0.111014912,-0.100440521,-0.109025404,-0.085814489,-0.097340162,-0.078505384,-0.183834706,-0.138231303,-0.158264483,-0.122806861,-0.0929401,-0.076793375,-0.192538767,-0.077121322,-0.190240716,-0.147301667,-0.096317627,-0.132102166,-0.132328837,-0.172528678,-0.157275636,-0.105270275,-0.183289032,-0.158963647,-0.073537654,-0.141423695,-0.114934001,-0.13547809,-0.090847761,-0.093080779,-0.156166631,-0.078869751,-0.104724625,-0.150905245,-0.094524319,-0.096496865,-0.09388536,-0.185296516,-0.137043852,-0.075459482,-0.136792671,-0.074754872,-0.191462381,-0.06797,-0.104708922,-0.097441687,-0.153319609,-0.157436972,-0.117974901,-0.085634242,-0.147203711,-0.198286569,-0.161203828,-0.111435796,-0.202759212,-0.129361842,-0.11614642,-0.138364847,-0.110683213,-0.168917077,-0.201554966,-0.101536865,-0.174292825,-0.11282408,-0.191933711,-0.092921165,-0.193068385,-0.196170309,-0.137841333,-0.131773845,-0.141334487,-0.121328408,-0.167894973,-0.183118788,-0.183716959,-0.159237081,-0.147624377,-0.195455928,-0.165164288,-0.108068155,-0.186753859,-0.173715911,-0.113001,-0.115163,-0.0811189,-0.188098863,-0.140947636,-0.141923621,-0.153645496,-0.152317537,-0.165842551,-0.14521173,-0.140741432,-0.175810943,-0.178433004,-0.164393768,-0.109914711,-0.132845681,-0.153520935,-0.133201961,-0.100186337,-0.091773776,-0.106237501,-0.098497684,-0.111606696,-0.082989638,-0.066078037,-0.161113413,-0.06954201,-0.100791005,-0.112432615,-0.06275,-0.062697,-0.153194766,-0.166304359,-0.165101392,-0.151926305,-0.158105512,-0.199004026,-0.149569201,-0.130504336,-0.088285377,-0.181190899,-0.082422399,-0.170194408,-0.19039362,-0.166485083,-0.13045856,-0.155349725,-0.090220911,-0.188129731,-0.196422,-0.131961389,-0.115480888,-0.134621209,-0.123179697,-0.103137426,-0.17873226,-0.112753217,-0.130699733,-0.106992706,-0.110889274,-0.073438925,-0.067176443,-0.175116099,-0.138019439,-0.10569204,-0.151359288,-0.139625122,-0.128585022,-0.142207481,-0.099994052,-0.168314,-0.170279555,-0.096191134,-0.162727,-0.079249,-0.116542278,-0.086379717,-0.106408455,-0.179592915,-0.116764211,-0.154907218,-0.178656971,-0.123247648,-0.135580879,-0.191351186,-0.103132904,-0.080660083,-0.109256828,-0.169249375,-0.180246101,-0.132224622,-0.144132875,-0.089740764,-0.122492418,-0.156285395,-0.110699309,-0.114686385,-0.20528437,-0.092176447,-0.084985356,-0.191496313,-0.07962099,-0.176645823,-0.162418,-0.127575233,-0.059642081,-0.112031483,-0.174653609,-0.128377673,-0.147478734,-0.151296092,-0.175488803,-0.138089062,-0.165471605,-0.209494128,-0.135635511,-0.092762704,-0.190603326,-0.106000855,-0.074189225,-0.136928582,-0.172729559,-0.148105415,-0.216573,-0.158456089,-0.16209757,-0.115853961,-0.135025698,-0.187842717,-0.085666316,-0.119047563,-0.116911864,-0.140485596,-0.176770502,-0.138072691,-0.083159352,-0.153463612,-0.141943703,-0.093913472,-0.138846453,-0.088542771,-0.139956043,-0.081608045,-0.073955,-0.083067,-0.101384068,-0.129697889,-0.099981142,-0.086016,-0.085603,-0.160854428,-0.179135079,-0.089887855,-0.194757949,-0.193764092,-0.115851,-0.120718759,-0.115533,-0.197524944,-0.119643424,-0.111781191,-0.087587447,-0.12602103,-0.150181444,-0.10102611,-0.166878535,-0.113936001,-0.150481272,-0.142783033,-0.1798541843891,-0.097122,-0.116625,-0.134234258,-0.123702,-0.117286,-0.173881,-0.139016,-0.124332175,-0.135440826,-0.138733562,-0.11342628,-0.133952,-0.171185284853,-0.132339,-0.100168,-0.1511263847351,-0.1642075,-0.15934065,-0.174593,-0.10988,-0.117863,-0.176415994,-0.11386435,-0.194392952,-0.125674815,-0.113656543,-0.179077626,-0.200838199,-0.150666888,-0.13577731,-0.14744969,-0.13121385,-0.192404,-0.130110822,-0.121394101,-0.121723604,-0.155757901,-0.068856,-0.142345694,-0.179702476,-0.103604248,-0.176268,-0.159919,-0.163609,-0.17977,-0.200742,-0.071653961,-0.154106,-0.075375,-0.171681991,-0.158249929,-0.184400221,-0.18267094,-0.159988,-0.160785,-0.170704337,-0.099828,-0.169774,-0.09968067,-0.110121,-0.149004,-0.105312,-0.104778,-0.15393398,-0.149186,-0.139159,-0.129925,-0.09294031,-0.174554,-0.17524,-0.122203,-0.173894,-0.116279,-0.105593,-0.11655,-0.120903,-0.118677,-0.113134,-0.114605,-0.211358,-0.058641,-0.055312,-0.065076,-0.055894,-0.007542,-0.02296,-0.057159,-0.053177,-0.063531,-0.070542,-0.055167,-0.021582,-0.014409,-0.144664,-0.145393,-0.057544,-0.03338,-0.029138,-0.038775,-0.138853,-0.071881,-0.052099,-0.08249,-0.076341,-0.030556,-0.022694,-0.020467,-0.025492,-0.028155,-0.027616,-0.019355,-0.011457,-0.020157,-0.009205,-0.018716,-0.04667,-0.058005,-0.0389866,-0.009001,-0.02377,-0.047784,-0.013258,-0.048017,-0.069543,-0.025626,-0.058681,-0.064094,-0.065006,-0.041378,-0.03562,-0.016251,-0.018703,-0.015578,-0.025296,-0.021345,-0.054883,-0.022702,-0.051394,-0.009506,-0.026768,-0.075855,-0.059091,-0.074431,-0.090075,-0.025996,-0.053558,-0.06142,-0.055656,-0.155525,-0.211316,-0.014438,-0.033085,-0.037471,-0.011662,-0.047218,-0.037366,-0.036017,-0.013475,-0.179668,-0.014293,-0.008428,-0.215808,-0.145827,-0.170983,-0.012413,-0.042744,-0.020068,-0.069743,-0.066035,-0.147154,-0.067937,-0.00699,-0.075901,-0.144466,-0.142844,-0.033828,-0.204666,-0.102208,-0.145246,-0.107987,-0.002275,-0.107913,-0.104193,-0.039264,-0.016233,-0.056667,-0.062546,-0.005659,-0.021596,-0.0985,-0.127554,-0.205991,-0.205921,-0.043371,-0.12335,-0.009152,-0.117619,-0.063386,-0.224103,-0.18697,-0.08299,-0.017676,-0.218337,-0.141728,-0.18119,-0.09315,-0.022793,-0.047548,-0.057133,-0.093051,-0.102996299,-0.125978,-0.19096,-0.014582,-0.08497,-0.0801,-0.179369,-0.16862,-0.2242237,-0.18377,-0.11934,-0.117774,-0.21985,-0.199783,-0.20782,-0.228188,-0.223616,-0.124642,-0.225787,-0.133972,-0.116493,-0.138535,-0.163667,-0.210941,-0.210279,-0.216493,-0.157788279,-0.172078187,-0.208486599,-0.141812513,-0.217400093,-0.144690541,-0.215895601,-0.209973497,-0.207842908,-0.193254007,-0.197010096,-0.167160736,-0.187427294,-0.205535908,-0.180791784,-0.132102704,-0.149551631,-0.20481514,-0.158230901,-0.184318843,-0.190184054,-0.126994068,-0.141770709,-0.163041605,-0.209378594,-0.215804559,-0.214428378,-0.193502076,-0.174691623,-0.169821175,-0.202038682,-0.148059277,-0.117495865,-0.174485792,-0.204764421,-0.223852256,-0.100292412,-0.137424571,-0.217515071,-0.19627483,-0.18027465,-0.213872396,-0.183853573,-0.218190203,-0.17070367,-0.117661574,-0.219346128,-0.199135704,-0.221791552,-0.16478637,-0.174619404,-0.206029743,-0.202608612,-0.167919869,-0.211593602,-0.155442787,-0.191722864,-0.208158259,-0.222293381,-0.236769936,-0.1232585,-0.152248582,-0.201968,-0.173656546,-0.18038939,-0.11619105,-0.182126248,-0.126874471,-0.146544642,-0.211468596,-0.170210533,-0.170329317,-0.214749808,-0.22660621,-0.163750945,-0.195197203,-0.198735357,-0.222456468,-0.21145598,-0.20066766,-0.180884959,-0.152130083,-0.195777222,-0.028941601,-0.215618902,-0.102757578,-0.217995921,-0.112721065,-0.070329419,-0.07023031,-0.174347066,-0.176267008,-0.065550321,-0.10534448,-0.202802098,-0.212145939,-0.083632928,-0.215087092,-0.21614583,-0.215550761,-0.163347594,-0.216305546,-0.034903714,-0.14326094,-0.137235175,-0.049067243,-0.02356501,-0.075885686,-0.060291813,-0.054162264,-0.205279052,-0.026262677,-0.058631453,-0.190346493,-0.184806157,-0.138748723,-0.150908371,-0.20587627,-0.206240805,-0.208485293,-0.229116862,-0.189210466,-0.087262995,-0.150817316,-0.175407201,-0.17302926,-0.19411695,-0.187278987,-0.185273723,-0.214594781,-0.219486603,-0.208565479,-0.212607684,-0.172293499,-0.18243547,-0.17903854,-0.161765173,-0.079201849,-0.074284675,-0.157850096,-0.120909408,-0.20600248,-0.234094148,-0.214762686,-0.174971902,-0.159169801,-0.187404506,-0.201005397,-0.165668686,-0.163795009,-0.211860644,-0.129698963,-0.032566533,-0.16829214,-0.20682737,-0.192165613,-0.200806304,-0.159322467,-0.191803,-0.209121,-0.216016,-0.122831913,-0.107349,-0.20464,-0.223868,-0.167029,-0.165297856693],10,null,null,{"interactive":true,"className":"","stroke":true,"color":"#03F","weight":5,"opacity":0.5,"fill":true,"fillColor":"#03F","fillOpacity":0.2},null,null,null,{"interactive":false,"permanent":false,"direction":"auto","opacity":1,"offset":[0,0],"textsize":"10px","textOnly":false,"className":"","sticky":true},null,null]},{"method":"addCircles","args":[[51.529125213623,51.5340156555176,51.5272903442383,51.5258293151855,51.5300140380859,51.5259437561035,51.5266494750977,51.5208587646484,51.5168228149414,51.5147361755371,51.4941635131836,51.500617980957,51.5178031921387,51.513298034668,51.5140533447266,51.5122489929199,51.4882698059082,51.5008125305176,51.5244827270508,51.500560760498,51.5033798217773,51.4950714111328,51.5074768066406,51.51416015625,51.5164070129395,51.5160675048828,51.5017127990723,51.502010345459,51.4952964782715,51.5189933776855,51.5199699401855,51.5119400024414,51.5217094421387,51.5156631469727,51.5124855041504,51.497802734375,51.5099983215332,51.4907989501953,51.4915580749512,51.5123100280762,51.4915046691895,51.5176963806152,51.5121421813965,51.4901008605957,51.5147857666016,51.4935836791992,51.4990310668945,51.4914512634277,51.5021781921387,51.5250129699707,51.5224571228027,51.522876739502,51.5253219604492,51.5219688415527,51.5164794921875,51.5145263671875,51.5168113708496,51.5358810424805,51.5181121826172,51.5233421325684,51.5256538391113,51.516731262207,51.5283546447754,51.5225028991699,51.530876159668,51.517276763916,51.5119209289551,51.5343589782715,51.5153121948242,51.5273170471191,51.5264739990234,51.5205345153809,51.5330848693848,51.5226020812988,51.5369148254395,51.5146942138672,51.5255584716797,51.5202369689941,51.5140647888184,51.5234184265137,51.5207405090332,51.5251617431641,51.5185699462891,51.5233764648438,51.5014801025391,51.5247001647949,51.4908981323242,51.5139122009277,51.5179023742676,51.5232048034668,51.5187034606934,51.5149116516113,51.5232696533203,51.5239067077637,51.5309143066406,51.5326194763184,51.5245056152344,51.5096054077148,51.5159797668457,51.5060081481934,51.5009002685547,51.5094451904297,51.5194892883301,51.5182151794434,51.5308990478516,51.521785736084,51.5235176086426,51.5269660949707,51.5086212158203,51.5047836303711,51.5065536499023,51.5093650817871,51.5111045837402,51.5031242370605,51.5027618408203,51.5063591003418,51.5027046203613,51.5030288696289,51.5228729248047,51.5071983337402,51.5067367553711,51.5333480834961,51.5329170227051,51.5173492431641,51.5254859924316,51.5202293395996,51.5236930847168,51.5276756286621,51.5300102233887,51.5288047790527,51.5262718200684,51.5092506408691,51.5185928344727,51.533073425293,51.5191764831543,51.5267524719238,51.5128898620605,51.5122489929199,51.5107154846191,51.5118522644043,51.509937286377,51.5099258422852,51.5119094848633,51.5057373046875,51.5053520202637,51.5158920288086,51.5115776062012,51.5170211791992,51.5191650390625,51.5209693908691,51.5216751098633,51.5319747924805,51.4943504333496,51.5294418334961,51.514461517334,51.5161094665527,51.5190467834473,51.5243988037109,51.5250663757324,51.5257682800293,51.5267105102539,51.5210418701172,51.4991226196289,51.4948768615723,51.4920654296875,51.4959564208984,51.4921226501465,51.5308647155762,51.5092468261719,51.5234756469727,51.5154342651367,51.5006256103516,51.4989547729492,51.5036697387695,51.4911231994629,51.5205955505371,51.4908218383789,51.4955253601074,51.5011711120605,51.4939422607422,51.5135307312012,51.4967422485352,51.4979972839355,51.5018653869629,51.4939193725586,51.4941368103027,51.4923629760742,51.507080078125,51.4970092773438,51.4936447143555,51.4965591430664,51.4994621276855,51.4988174438477,51.5012893676758,51.4946479797363,51.4931907653809,51.4924049377441,51.4978790283203,51.4901084899902,51.4899024963379,51.4911193847656,51.4897422790527,51.4880409240723,51.4907531738281,51.5014457702637,51.5119781494141,51.5103721618652,51.4921836853027,51.5150375366211,51.5065040588379,51.5149192810059,51.5050163269043,51.5117492675781,51.5143547058105,51.506591796875,51.5076751708984,51.5074043273926,51.5080032348633,51.5084495544434,51.5090751647949,51.5096893310547,51.5175819396973,51.513744354248,51.514705657959,51.5144729614258,51.5313301086426,51.5283851623535,51.5264587402344,51.5267333984375,51.5304489135742,51.5282669067383,51.528881072998,51.5291633605957,51.5278205871582,51.5318183898926,51.5345153808594,51.5316009521484,51.5344848632812,51.5183601379395,51.5100250244141,51.5299301147461,51.5262222290039,51.5046043395996,51.4964332580566,51.5157890319824,51.5137596130371,51.5159225463867,51.5094718933105,51.4948692321777,51.4958152770996,51.493968963623,51.4961357116699,51.5263519287109,51.4882659912109,51.493221282959,51.4923477172852,51.4936294555664,51.4899444580078,51.5057983398438,51.4853782653809,51.488109588623,51.4898338317871,51.4981575012207,51.5164451599121,51.5305595397949,51.4848556518555,51.5179138183594,51.5148849487305,51.5155792236328,51.5179176330566,51.5156059265137,51.5171279907227,51.5189018249512,51.5142593383789,51.5121574401855,51.5222663879395,51.4986839294434,51.4996109008789,51.500316619873,51.4973487854004,51.4966926574707,51.5003280639648,51.4962997436523,51.5053176879883,51.5006370544434,51.4980773925781,51.5183410644531,51.4937019348145,51.4933166503906,51.4958381652832,51.501407623291,51.518970489502,51.5046882629395,51.5213012695312,51.4974403381348,51.4907531738281,51.4868087768555,51.4862594604492,51.4937286376953,51.489444732666,51.492862701416,51.4848365783691,51.4904365539551,51.4957122802734,51.4889488220215,51.4860305786133,51.4908561706543,51.5000915527344,51.51220703125,51.5017890930176,51.5172843933105,51.5029449462891,51.5115776062012,51.5177955627441,51.5166702270508,51.4986000061035,51.4967498779297,51.5164489746094,51.5197067260742,51.4979667663574,51.5019798278809,51.5204887390137,51.4968948364258,51.5194702148438,51.5209770202637,51.513484954834,51.5129928588867,51.5295066833496,51.5109214782715,51.5181312561035,51.5093536376953,51.5107040405273,51.5068664550781,51.4971313476562,51.4997673034668,51.5023498535156,51.5360565185547,51.5134391784668,51.5176811218262,51.5180625915527,51.5136108398438,51.5100975036621,51.5106239318848,51.4994277954102,51.4917526245117,51.517276763916,51.5125617980957,51.5200080871582,51.5002670288086,51.5226516723633,51.5199127197266,51.5202560424805,51.519889831543,51.5150718688965,51.5041122436523,51.5224304199219,51.5236129760742,51.5246696472168,51.5218658447266,51.5059242248535,51.4934043884277,51.493106842041,51.5321311950684,51.5357322692871,51.5163230895996,51.5344505310059,51.5337982177734,51.4944076538086,51.5264663696289,51.531551361084,51.5303039550781,51.5311050415039,51.510139465332,51.5091667175293,51.5091285705566,51.5117263793945,51.5173530578613,51.5073699951172,51.5064506530762,51.5013732910156,51.5063514709473,51.5334243774414,51.519157409668,51.5282096862793,51.5061111450195,51.5050506591797,51.5048637390137,51.5039672851562,51.5037879943848,51.517147064209,51.5160942077637,51.5123672485352,51.5235061645508,51.5163383483887,51.5112266540527,51.5116233825684,51.5140647888184,51.5175437927246,51.5209159851074,51.525691986084,51.5225563049316,51.5340919494629,51.5289916992188,51.53515625,51.5325050354004,51.533317565918,51.5305099487305,51.5294303894043,51.5288352966309,51.5260200500488,51.4999237060547,51.528678894043,51.5112457275391,51.5142059326172,51.5110816955566,51.5192031860352,51.5088996887207,51.5390510559082,51.5092964172363,51.4875411987305,51.5162467956543,51.5311241149902,51.5309371948242,51.5070762634277,51.4960556030273,51.5289154052734,51.5288238525391,51.5289154052734,51.5288505554199,51.5288276672363,51.5288581848145,51.5287475585938,51.5287551879883,51.4969177246094,51.5017356872559,51.4592666625977,51.4619598388672,51.462329864502,51.4643669128418,51.4646339416504,51.4649353027344,51.5150985717773,51.4933090209961,51.4910316467285,51.5420379638672,51.5342826843262,51.5342788696289,51.5340614318848,51.5340614318848,51.4881706237793,51.5129814147949,51.5363426208496,51.4988250732422,51.5045928955078,51.5261192321777,51.5133056640625,51.5135383605957,51.5095901489258,51.5136680603027,51.5251502990723,51.5159912109375,51.4698219299316,51.4697875976562,51.4699058532715,51.4699401855469,51.488899230957,51.5295372009277,51.513744354248,51.5281829833984,51.5055847167969,51.5053405761719,51.5048713684082,51.5038681030273,51.5182075500488,51.5151634216309,51.5256118774414,51.5275001525879,51.527172088623,51.4920692443848,51.4730491638184,51.5115203857422,51.5215797424316,51.5194625854492,51.5380401611328,51.4958915710449,51.5040321350098,51.5040473937988,51.5041313171387,51.522575378418,51.5047378540039,51.5047645568848,51.5048065185547,51.5048294067383,51.5281829833984,51.4751319885254,51.5346603393555,51.528205871582,51.5468254089355,51.5462989807129,51.500301361084,51.5359039306641,51.5275382995605,51.540283203125,51.5359802246094,51.5118980407715,51.4939308166504,51.4814796447754,51.5410346984863,51.5416946411133,51.4858245849609,51.4675712585449,51.5387687683105,51.5164527893066,51.5039749145508,51.4762268066406,51.4794158935547,51.4799423217773,51.483585357666,51.4839019775391,51.5093116760254,51.5093193054199,51.5093231201172,51.5093383789062,51.5093421936035,51.5098342895508,51.509838104248,51.5099716186523,51.5099754333496,51.5408706665039,51.543083190918,51.5384292602539,51.5224761962891,51.4823341369629,51.5178565979004,51.5178756713867,51.5178985595703,51.5180282592773,51.4926681518555,51.5094337463379,51.5089530944824],[-0.0933877974748611,-0.129309207201004,-0.118235200643539,-0.0908360034227371,-0.121057197451591,-0.103827200829983,-0.112325102090836,-0.089885301887989,-0.158216997981071,-0.122161999344826,-0.182531297206879,-0.094515897333622,-0.0963746979832649,-0.0767054036259651,-0.0735850036144257,-0.0694098994135857,-0.135628700256348,-0.0898251011967659,-0.158880099654198,-0.0785683989524841,-0.0795795023441315,-0.0859436988830566,-0.096405602991581,-0.0806185975670815,-0.0796248018741608,-0.0821207985281944,-0.184889003634453,-0.184379398822784,-0.18527090549469,-0.1247294023633,-0.135864093899727,-0.120736703276634,-0.130427300930023,-0.13221150636673,-0.133161500096321,-0.0816432014107704,-0.157212495803833,-0.196125403046608,-0.186663806438446,-0.159797996282578,-0.192472696304321,-0.128009602427483,-0.162088096141815,-0.190465196967125,-0.16520619392395,-0.190665796399117,-0.0855932980775833,-0.0901288986206055,-0.0742235034704208,-0.166210398077965,-0.154845103621483,-0.171658992767334,-0.153420701622963,-0.151339396834373,-0.164416894316673,-0.15824930369854,-0.151897504925728,-0.160705104470253,-0.144139796495438,-0.183850601315498,-0.143984407186508,-0.175549402832985,-0.170091196894646,-0.162271693348885,-0.176805004477501,-0.175872594118118,-0.174384206533432,-0.168152794241905,-0.147055804729462,-0.174666598439217,-0.172155693173409,-0.154764696955681,-0.172641694545746,-0.161039903759956,-0.150157704949379,-0.148090302944183,-0.179504796862602,-0.157090201973915,-0.147284805774689,-0.175151601433754,-0.145077005028725,-0.135173499584198,-0.176669493317604,-0.124170199036598,-0.110720098018646,-0.0849407985806465,-0.139493301510811,-0.0928172022104263,-0.10842839628458,-0.104731000959873,-0.108025997877121,-0.0661884024739265,-0.120411798357964,-0.122491598129272,-0.0938763990998268,-0.0999554991722107,-0.0792410969734192,-0.0746577978134155,-0.16925460100174,-0.0927129983901978,-0.0834731012582779,-0.124383203685284,-0.119153201580048,-0.0626906976103783,-0.0785432010889053,-0.109234899282455,-0.108374498784542,-0.0886150002479553,-0.193726301193237,-0.192558497190475,-0.198990702629089,-0.196362406015396,-0.197485104203224,-0.153525605797768,-0.149403899908066,-0.170132204890251,-0.155268996953964,-0.191404402256012,-0.0999595001339912,-0.106189802289009,-0.103289797902107,-0.111789003014565,-0.136768698692322,-0.138106092810631,-0.138213202357292,-0.14128689467907,-0.128438904881477,-0.135379806160927,-0.138743206858635,-0.13221900165081,-0.134236499667168,-0.151116207242012,-0.131968900561333,-0.139157295227051,-0.140554800629616,-0.130548700690269,-0.153644695878029,-0.157555893063545,-0.1441929936409,-0.142809197306633,-0.138810202479362,-0.187907293438911,-0.137033000588417,-0.0997940972447395,-0.105653703212738,-0.0930432975292206,-0.0929120033979416,-0.0938486009836197,-0.0882396027445793,-0.0856646969914436,-0.094421498477459,-0.105481497943401,-0.0929258018732071,-0.0833719000220299,-0.0876550003886223,-0.128589496016502,-0.0597107000648975,-0.13815450668335,-0.131151705980301,-0.0884820967912674,-0.0781090036034584,-0.0787594988942146,-0.112052097916603,-0.11793690174818,-0.132180005311966,-0.135328501462936,-0.138317495584488,-0.0899377018213272,-0.0847280994057655,-0.143566697835922,-0.0988024994730949,-0.101928897202015,-0.100190803408623,-0.0984831005334854,-0.0971252992749214,-0.116697296500206,-0.181300804018974,-0.179187700152397,-0.180236399173737,-0.178733006119728,-0.191322594881058,-0.138812601566315,-0.143824398517609,-0.15922899544239,-0.14760759472847,-0.154608502984047,-0.147624105215073,-0.145930394530296,-0.168843403458595,-0.164939492940903,-0.150920301675797,-0.152325302362442,-0.165447399020195,-0.153244093060493,-0.15806670486927,-0.16802279651165,-0.178409799933434,-0.183833494782448,-0.162575304508209,-0.162599697709084,-0.173689395189285,-0.170212998986244,-0.166848197579384,-0.166260406374931,-0.17842809855938,-0.0975117981433868,-0.0829029977321625,-0.101552903652191,-0.11262109875679,-0.123274296522141,-0.116216897964478,-0.172720402479172,-0.119723200798035,-0.118467800319195,-0.131739303469658,-0.131008505821228,-0.134654104709625,-0.125960305333138,-0.131887093186378,-0.129745304584503,-0.131469696760178,-0.121338799595833,-0.135445103049278,-0.137537002563477,-0.141436398029327,-0.117013201117516,-0.101044498383999,-0.10912349820137,-0.104285500943661,-0.106341503560543,-0.104715898633003,-0.115418702363968,-0.109950698912144,-0.108051002025604,-0.114294797182083,-0.109049297869205,-0.109813898801804,-0.10696080327034,-0.0730184018611908,-0.143455997109413,-0.123548701405525,-0.123461499810219,-0.0917520001530647,-0.101619601249695,-0.105260796844959,-0.107882797718048,-0.111751697957516,-0.119044497609138,-0.130570992827415,-0.127540901303291,-0.136847898364067,-0.140881896018982,-0.125988394021988,-0.129254505038261,-0.14417290687561,-0.141303792595863,-0.140054807066917,-0.132818803191185,-0.136564701795578,-0.142174899578094,-0.140596807003021,-0.142041102051735,-0.131944105029106,-0.179172202944756,-0.167493104934692,-0.138083204627037,-0.187978401780128,-0.188125595450401,-0.183117806911469,-0.183628305792809,-0.190236896276474,-0.086660198867321,-0.156065493822098,-0.200990095734596,-0.201490193605423,-0.11407820135355,-0.103923097252846,-0.197503104805946,-0.193005800247192,-0.197288200259209,-0.205240100622177,-0.195428803563118,-0.105878598988056,-0.112193301320076,-0.202580004930496,-0.209433004260063,-0.100676998496056,-0.198381707072258,-0.194729894399643,-0.191833004355431,-0.191565096378326,-0.0788919031620026,-0.123250603675842,-0.0845493003726006,-0.0895150005817413,-0.106222197413445,-0.115849301218987,-0.122282296419144,-0.111050099134445,-0.115093097090721,-0.114959798753262,-0.110633701086044,-0.122760303318501,-0.111009202897549,-0.111419402062893,-0.124379597604275,-0.116879597306252,-0.113917402923107,-0.150479704141617,-0.179724097251892,-0.164408206939697,-0.158520206809044,-0.0769961029291153,-0.0900261998176575,-0.179517894983292,-0.0962169021368027,-0.0939660966396332,-0.124270096421242,-0.132357105612755,-0.135017797350883,-0.194334402680397,-0.0973121002316475,-0.161378994584084,-0.135711193084717,-0.138918504118919,-0.130049198865891,-0.131127893924713,-0.0970930978655815,-0.151217997074127,-0.134945601224899,-0.147493600845337,-0.142176300287247,-0.15065510571003,-0.19227659702301,-0.176309496164322,-0.200730696320534,-0.133782297372818,-0.17974080145359,-0.154045194387436,-0.163533195853233,-0.193469405174255,-0.155691504478455,-0.121644802391529,-0.17970509827137,-0.125699892640114,-0.103563599288464,-0.115013100206852,-0.09218680113554,-0.09272850304842,-0.0715885013341904,-0.169783800840378,-0.174565702676773,-0.17051850259304,-0.118133701384068,-0.217499002814293,-0.0417601987719536,-0.0749483034014702,-0.0356981009244919,-0.0466320998966694,-0.0706816986203194,-0.0998281985521317,-0.0998431965708733,-0.0613513998687267,-0.0627153962850571,-0.0984753966331482,-0.122087001800537,-0.118647001683712,-0.173518195748329,-0.0286301001906395,-0.0661886036396027,-0.0427928008139133,-0.0479843989014626,-0.211451902985573,-0.224229201674461,-0.219624802470207,-0.205994099378586,-0.123187303543091,-0.145741403102875,-0.143008604645729,-0.205932304263115,-0.218409404158592,-0.0932016000151634,-0.147731497883797,-0.0373585000634193,-0.114680297672749,-0.115952901542187,-0.115394696593285,-0.113868497312069,-0.113037802278996,-0.183437004685402,-0.187355905771255,-0.190958499908447,-0.0305780004709959,-0.0291713997721672,-0.0929962024092674,-0.068986102938652,-0.111169598996639,-0.107948698103428,-0.0513298995792866,-0.0552406013011932,-0.0548060983419418,-0.0373180992901325,-0.0558837987482548,-0.0334074012935162,-0.0329675003886223,-0.0281582996249199,-0.0254478007555008,-0.0276004001498222,-0.0477617010474205,-0.0358303003013134,-0.174587905406952,-0.0874010026454926,-0.0140482001006603,-0.0336204990744591,-0.0573639012873173,-0.0213794000446796,-0.0124099003151059,-0.141626298427582,-0.0258732996881008,-0.116860397160053,-0.155214801430702,-0.0859581008553505,-0.0855529978871346,-0.0668997019529343,-0.104211002588272,-0.0133453998714685,-0.013314500451088,-0.0133053995668888,-0.0133092002943158,-0.0133496997877955,-0.0133453998714685,-0.0133426999673247,-0.0133788995444775,-0.173877105116844,-0.100308798253536,-0.180845305323601,-0.180836006999016,-0.175301000475883,-0.174712300300598,-0.173866093158722,-0.172882899641991,-0.105443403124809,-0.219121798872948,-0.216540202498436,-0.0288302004337311,-0.0863690003752708,-0.0864045023918152,-0.0863825976848602,-0.0863469988107681,-0.222301796078682,-0.0640854984521866,-0.102562800049782,-0.137372195720673,-0.116579502820969,-0.0469631999731064,-0.0479707010090351,-0.116719298064709,-0.0846550986170769,-0.117624796926975,-0.015591099858284,-0.120800897479057,-0.140801593661308,-0.140767604112625,-0.140485495328903,-0.140521302819252,-0.105547197163105,-0.0800985991954803,-0.0204048994928598,-0.0754837989807129,-0.111793100833893,-0.113653101027012,-0.112904697656631,-0.113424003124237,-0.116502098739147,-0.0584450997412205,-0.0695533007383347,-0.0570680983364582,-0.0579186007380486,-0.229122996330261,-0.214725703001022,-0.0567092001438141,-0.0223765000700951,-0.0744313970208168,-0.144642606377602,-0.172876805067062,-0.217456996440887,-0.217363193631172,-0.217405200004578,-0.041015300899744,-0.0675463974475861,-0.0675292015075684,-0.0677891001105309,-0.0677718967199326,-0.0697977989912033,-0.159299001097679,-0.124966099858284,-0.0694345012307167,-0.0145474001765251,-0.0100236004218459,-0.159067794680595,-0.156053707003593,-0.134881302714348,-0.0216605998575687,-0.0266005005687475,-0.107156299054623,-0.127467095851898,-0.138067096471786,-0.143167093396187,-0.139073401689529,-0.148810297250748,-0.206682801246643,-0.138449296355247,-0.11837849766016,-0.0132076004520059,-0.193282201886177,-0.195741400122643,-0.194131806492805,-0.202047005295753,-0.197559401392937,-0.0259233005344868,-0.0258118994534016,-0.0257335007190704,-0.0257322005927563,-0.0258220005780458,-0.0237387008965015,-0.0237748995423317,-0.0236888006329536,-0.0237250998616219,-0.0107442997395992,-0.00798430014401674,-0.0118950000032783,-0.0417247004806995,-0.136271804571152,-0.0432214997708797,-0.0431764982640743,-0.04341059923172,-0.0432161018252373,-0.0923537015914917,-0.00241909991018474,-0.0069093001075089],10,null,null,{"interactive":true,"className":"","stroke":true,"color":"red","weight":5,"opacity":0.5,"fill":true,"fillColor":"red","fillOpacity":0.2},null,null,null,{"interactive":false,"permanent":false,"direction":"auto","opacity":1,"offset":[0,0],"textsize":"10px","textOnly":false,"className":"","sticky":true},null,null]}],"limits":{"lat":[51.45475251,51.5468254089355],"lng":[-0.236769936,-0.002275]}},"evals":[],"jsHooks":[]}</script><!--/html_preserve-->
<p class="caption">(\#fig:cycle-hire)The spatial distribution of cycle hire points in London based on official data (blue) and OpenStreetMap data (red).</p>
</div>

Imagine that we need to join the `capacity` variable in `cycle_hire_osm` onto the official 'target' data contained in `cycle_hire`.
This is when a non-overlapping join is needed.
The simplest method is to use the topological operator `st_is_within_distance()` shown in Section \@ref(topological-relations), using a threshold distance of 20 m.
Note that, before performing the relation, both objects are transformed into a projected CRS.
These projected objects are created below (note the affix `_P`, short for projected):


```r
cycle_hire_P = st_transform(cycle_hire, 27700)
cycle_hire_osm_P = st_transform(cycle_hire_osm, 27700)
sel = st_is_within_distance(cycle_hire_P, cycle_hire_osm_P, dist = 20)
summary(lengths(sel) > 0)
#>    Mode   FALSE    TRUE 
#> logical     304     438
```

This shows that there are 438 points in the target object `cycle_hire_P` within the threshold distance of `cycle_hire_osm_P`.
How to retrieve the *values* associated with the respective `cycle_hire_osm_P` points?
The solution is again with `st_join()`, but with an addition `dist` argument (set to 20 m below):


```r
z = st_join(cycle_hire_P, cycle_hire_osm_P,
            join = st_is_within_distance, dist = 20)
nrow(cycle_hire)
#> [1] 742
nrow(z)
#> [1] 762
```

Note that the number of rows in the joined result is greater than the target.
This is because some cycle hire stations in `cycle_hire_P` have multiple matches in `cycle_hire_osm_P`.
To aggregate the values for the overlapping points and return the mean, we can use the aggregation methods learned in Chapter \@ref(attr), resulting in an object with the same number of rows as the target:


```r
z = z %>% 
  group_by(id) %>% 
  summarize(capacity = mean(capacity))
#> `summarise()` ungrouping output (override with `.groups` argument)
nrow(z) == nrow(cycle_hire)
#> [1] TRUE
```

The capacity of nearby stations can be verified by comparing a plot of the capacity of the source `cycle_hire_osm` data with the results in this new object (plots not shown):


```r
plot(cycle_hire_osm["capacity"])
plot(z["capacity"])
```

<!-- Nearest neighbour analysis -->
<!-- e.g. two point's datasets (non-overlapping) -->
<!-- e.g. two point's datasets (overlapping) -->
<!-- ? topological problems of joining lines/polygons? -->
<!-- joining different types (e.g. points + polygons = geometry) -> save as GPKG? -->
<!-- `merge()`; `st_interpolate_aw()` -->

The result of this join has used a spatial operation to change the attribute data associated with simple features;  the geometry associated with each feature has remained unchanged.

### Spatial data aggregation {#spatial-aggr}

Like attribute data aggregation, covered in Section \@ref(vector-attribute-aggregation), spatial data aggregation can be a way of *condensing* data.
Aggregated data show some statistics\index{statistics} about a variable (typically average or total) in relation to some kind of *grouping variable*.
Section \@ref(vector-attribute-aggregation) demonstrated how `aggregate()` and `group_by() %>% summarize()` condense data based on attribute variables.
This section demonstrates how the same functions work using spatial grouping variables.
\index{aggregation!spatial}

Returning to the example of New Zealand, imagine you want to find out the average height of high points in each region.
This is a good example of spatial aggregation: it is the geometry of the source (`y` or `nz` in this case) that defines how values in the target object (`x` or `nz_height`) are grouped.
This is illustrated using the base `aggregate()` function below:


```r
nz_avheight = aggregate(x = nz_height, by = nz, FUN = mean)
```

The result of the previous command is an `sf` object with the same geometry as the (spatial) aggregating object (`nz`).^[
This can be verified with `identical(st_geometry(nz), st_geometry(nz_avheight))`.
]
The result of the previous operation is illustrated in Figure \@ref(fig:spatial-aggregation).
The same result can also be generated using the 'tidy' functions `group_by()` and `summarize()` (used in combination with `st_join()`):

<div class="figure" style="text-align: center">
<img src="04-spatial-operations_files/figure-html/spatial-aggregation-1.png" alt="Average height of the top 101 high points across the regions of New Zealand." width="50%" />
<p class="caption">(\#fig:spatial-aggregation)Average height of the top 101 high points across the regions of New Zealand.</p>
</div>


```r
nz_avheight2 = nz %>%
  st_join(nz_height) %>%
  group_by(Name) %>%
  summarize(elevation = mean(elevation, na.rm = TRUE))
#> `summarise()` ungrouping output (override with `.groups` argument)
```

The resulting `nz_avheight` objects have the same geometry as the aggregating object `nz` but with a new column representing the mean average height of points within each region of New Zealand (other summary functions such as `median()` and `sd()` can be used in place of `mean()`).
Note that regions containing no points have an associated `elevation` value of `NA`.
For aggregating operations which also create new geometries, see Section \@ref(geometry-unions).

Spatial congruence\index{spatial congruence} is an important concept related to spatial aggregation.
An *aggregating object* (which we will refer to as `y`) is *congruent* with the target object (`x`) if the two objects have shared borders.
Often this is the case for administrative boundary data, whereby larger units --- such as Middle Layer Super Output Areas ([MSOAs](https://www.ons.gov.uk/methodology/geography/ukgeographies/censusgeography)) in the UK or districts in many other European countries --- are composed of many smaller units.

*Incongruent* aggregating objects, by contrast, do not share common borders with the target [@qiu_development_2012].
This is problematic for spatial aggregation (and other spatial operations) illustrated in Figure \@ref(fig:areal-example).
Areal interpolation overcomes this issue by transferring values from one set of areal units to another.
Algorithms developed for this task include area weighted and 'pycnophylactic' areal interpolation methods [@tobler_smooth_1979].

<div class="figure" style="text-align: center">
<img src="04-spatial-operations_files/figure-html/areal-example-1.png" alt="Illustration of congruent (left) and incongruent (right) areal units with respect to larger aggregating zones (translucent blue borders)." width="100%" />
<p class="caption">(\#fig:areal-example)Illustration of congruent (left) and incongruent (right) areal units with respect to larger aggregating zones (translucent blue borders).</p>
</div>

The **spData** package contains a dataset named `incongruent` (colored polygons with black borders in the right panel of Figure \@ref(fig:areal-example)) and a dataset named `aggregating_zones` (the two polygons with the translucent blue border in the right panel of Figure \@ref(fig:areal-example)).
Let us assume that the `value` column of `incongruent` refers to the total regional income in million Euros.
How can we transfer the values of the underlying nine spatial polygons into the two polygons of `aggregating_zones`?

The simplest useful method for this is *area weighted* spatial interpolation.
In this case values from the `incongruent` object are allocated to the `aggregating_zones` in proportion to area; the larger the spatial intersection  between input and output features, the larger the corresponding value. 
For instance, if one intersection of `incongruent` and `aggregating_zones` is 1.5 km^2^ but the whole incongruent polygon in question has 2 km^2^ and a total income of 4 million Euros, then the target aggregating zone will obtain three quarters of the income, in this case 3 million Euros.
This is implemented in `st_interpolate_aw()`, as demonstrated in the code chunk below.


```r
agg_aw = st_interpolate_aw(incongruent[, "value"], aggregating_zones,
                           extensive = TRUE)
#> Warning in st_interpolate_aw.sf(incongruent[, "value"], aggregating_zones, :
#> st_interpolate_aw assumes attributes are constant or uniform over areas of x
# show the aggregated result
agg_aw$value
#> [1] 19.6 25.7
```

In our case it is meaningful to sum up the values of the intersections falling into the aggregating zones since total income is a so-called spatially extensive variable.
This would be different for spatially intensive variables, which are independent of the spatial units used, such as income per head or [percentages](http://ibis.geog.ubc.ca/courses/geob370/notes/intensive_extensive.htm).
In this case it is more meaningful to apply an average function when doing the aggregation instead of a sum function.
To do so, one would only have to set the `extensive` parameter to `FALSE`.

<!-- - `aggregate.sf()` - aggregate an sf object, possibly union-ing geometries -->
<!-- - disaggregation?? `st_cast()` - https://github.com/edzer/sfr/wiki/migrating -->
<!-- - `group_by()` + `summarise()` - potential errors -->
<!-- - ? generalization **rmapsharper** - https://github.com/ateucher/rmapshaper -->
<!-- `st_union` -->

### Distance relations 

While topological relations are binary --- a feature either intersects with another or does not --- distance relations are continuous.
The distance between two objects is calculated with the `st_distance()` function.
This is illustrated in the code chunk below, which finds the distance between the highest point in New Zealand and the geographic centroid of the Canterbury region, created in Section \@ref(spatial-subsetting):
\index{sf!distance relations}


```r
nz_heighest = nz_height %>% top_n(n = 1, wt = elevation)
canterbury_centroid = st_centroid(canterbury)
st_distance(nz_heighest, canterbury_centroid)
#> Units: [m]
#>        [,1]
#> [1,] 115540
```

There are two potentially surprising things about the result:

- It has `units`, telling us the distance is 100,000 meters, not 100,000 inches, or any other measure of distance
- It is returned as a matrix, even though the result only contains a single value

This second feature hints at another useful feature of `st_distance()`, its ability to return *distance matrices* between all combinations of features in objects `x` and `y`.
This is illustrated in the command below, which finds the distances between the first three features in `nz_height` and the Otago and Canterbury regions of New Zealand represented by the object `co`.


```r
co = filter(nz, grepl("Canter|Otag", Name))
st_distance(nz_height[1:3, ], co)
#> Units: [m]
#>        [,1]  [,2]
#> [1,] 123537 15498
#> [2,]  94283     0
#> [3,]  93019     0
```

Note that the distance between the second and third features in `nz_height` and the second feature in `co` is zero.
This demonstrates the fact that distances between points and polygons refer to the distance to *any part of the polygon*:
The second and third points in `nz_height` are *in* Otago, which can be verified by plotting them (result not shown):


```r
plot(st_geometry(co)[2])
plot(st_geometry(nz_height)[2:3], add = TRUE)
```

## Spatial operations on raster data {#spatial-ras}

This section builds on Section \@ref(manipulating-raster-objects), which highlights various basic methods for manipulating raster datasets, to demonstrate more advanced and explicitly spatial raster operations, and uses the objects `elev` and `grain` manually created in Section \@ref(manipulating-raster-objects).
For the reader's convenience, these datasets can be also found in the **spData** package.

### Spatial subsetting {#spatial-raster-subsetting}

The previous chapter (Section \@ref(manipulating-raster-objects)) demonstrated how to retrieve values associated with specific cell IDs or row and column combinations.
Raster objects can also be extracted by location (coordinates) and other spatial objects.
To use coordinates for subsetting, one can 'translate' the coordinates into a cell ID with the **raster** function `cellFromXY()`.
An alternative is to use `raster::extract()` (be careful, there is also a function called `extract()` in the **tidyverse**\index{tidyverse (package)}) to extract values.
Both methods are demonstrated below to find the value of the cell that covers a point located 0.1 units from the origin.
\index{raster!subsetting}
\index{spatial!subsetting}


```r
id = cellFromXY(elev, xy = c(0.1, 0.1))
elev[id]
# the same as
raster::extract(elev, data.frame(x = 0.1, y = 0.1))
```

It is convenient that both functions also accept objects of class `Spatial* Objects`.
Raster objects can also be subset with another raster object, as illustrated in Figure \@ref(fig:raster-subset) (left panel) and demonstrated in the code chunk below:


```r
clip = raster(xmn = 0.9, xmx = 1.8, ymn = -0.45, ymx = 0.45,
              res = 0.3, vals = rep(1, 9))
elev[clip]
#> [1] 18 24
# we can also use extract
# extract(elev, extent(clip))
```

Basically, this amounts to retrieving the values of the first raster (here: `elev`) falling within the extent of a second raster (here: `clip`).

<div class="figure" style="text-align: center">
<img src="figures/04_raster_subset.png" alt="Subsetting raster values with the help of another raster (left). Raster mask (middle). Output of masking a raster (right)." width="100%" />
<p class="caption">(\#fig:raster-subset)Subsetting raster values with the help of another raster (left). Raster mask (middle). Output of masking a raster (right).</p>
</div>

So far, the subsetting returned the values of specific cells, however, when doing spatial subsetting, one often also expects a spatial object as an output.
To do this, we can use again the `[` when we additionally set the `drop` parameter to `FALSE`.
To illustrate this, we retrieve the first two cells of `elev` as an individual raster object. 
As mentioned in Section \@ref(manipulating-raster-objects), the `[` operator accepts various inputs to subset rasters and returns a raster object when `drop = FALSE`.
The code chunk below subsets the `elev` raster by cell ID and row-column index with identical results: the first two cells on the top row (only the first 2 lines of the output is shown):


```r
elev[1:2, drop = FALSE]    # spatial subsetting with cell IDs
elev[1, 1:2, drop = FALSE] # spatial subsetting by row,column indices
#> class       : RasterLayer 
#> dimensions  : 1, 2, 2  (nrow, ncol, ncell)
#> ...
```




Another common use case of spatial subsetting is when a raster with `logical` (or `NA`) values is used to mask another raster with the same extent and resolution, as illustrated in Figure \@ref(fig:raster-subset), middle and right panel.
In this case, the `[`, `mask()` and `overlay()` functions can be used (results not shown):


```r
# create raster mask
rmask = elev 
values(rmask) = sample(c(NA, TRUE), 36, replace = TRUE)

# spatial subsetting
elev[rmask, drop = FALSE]           # with [ operator
mask(elev, rmask)                   # with mask()
overlay(elev, rmask, fun = "max")   # with overlay
```




In the code chunk above, we have created a mask object called `rmask` with values randomly assigned to `NA` and `TRUE`.
Next, we want to keep those values of `elev` which are `TRUE` in `rmask`.
In other words, we want to mask `elev` with `rmask`.
These operations are in fact Boolean local operations since we compare cell-wise two rasters.
The next subsection explores these and related operations in more detail.

### Map algebra

\index{map algebra}
Map algebra makes raster processing really fast.
This is because raster datasets only implicitly store coordinates.
To derive the coordinate of a specific cell, we have to calculate it using its matrix position and the raster resolution and origin.
For the processing, however, the geographic position of a cell is barely relevant as long as we make sure that the cell position is still the same after the processing (one-to-one locational correspondence).
Additionally, if two or more raster datasets share the same extent, projection and resolution, one could treat them as matrices for the processing.
This is exactly what map algebra is doing in R.
First, the **raster** package checks the headers of the rasters on which to perform any algebraic operation, and only if they are correspondent to each other, the processing goes on.^[
Map algebra operations are also possible with headerless rasters; in this case the user has to make sure that in fact there exists a one-to-one locational correspondence.
An example showing how to import a headerless raster into R is provided in a post at https://stat.ethz.ch/pipermail/r-sig-geo/2013-May/018278.html.
]
And secondly, map algebra retains the so-called one-to-one locational correspondence.
This is where it substantially differs from matrix algebra which changes positions when for example multiplying or dividing matrices.

Map algebra (or cartographic modeling) divides raster operations into four subclasses [@tomlin_geographic_1990], with each working on one or several grids simultaneously:

1. *Local* or per-cell operations.
2. *Focal* or neighborhood operations.
Most often the output cell value is the result of a 3 x 3 input cell block.
3. *Zonal* operations are similar to focal operations, but the surrounding pixel grid on which new values are computed can have irregular sizes and shapes.
<!-- sentence structure could be confusing in the sentence above -->
4. *Global* or per-raster operations; that means the output cell derives its value potentially from one or several entire rasters.

This typology classifies map algebra operations by the number/shape of cells used for each pixel processing step.
For the sake of completeness, we should mention that raster operations can also be classified by discipline such as terrain, hydrological analysis or image classification.
The following sections explain how each type of map algebra operations can be used, with reference to worked examples (also see `vignette("Raster")` for a technical description of map algebra).

### Local operations

\index{map algebra!local operations}
**Local** operations comprise all cell-by-cell operations in one or several layers.
A good example is the classification of intervals of numeric values into groups such as grouping a digital elevation model into low (class 1), middle (class 2) and high elevations (class 3).
Using the `reclassify()` command, we need first to construct a reclassification matrix, where the first column corresponds to the lower and the second column to the upper end of the class.
The third column represents the new value for the specified ranges in column one and two.
Here, we assign the raster values in the ranges 0--12, 12--24 and 24--36 are *reclassified* to take values 1, 2 and 3, respectively.


```r
rcl = matrix(c(0, 12, 1, 12, 24, 2, 24, 36, 3), ncol = 3, byrow = TRUE)
recl = reclassify(elev, rcl = rcl)
```

We will perform several reclassifactions in Chapter \@ref(location).

Raster algebra is another classical use case of local operations.
This includes adding, subtracting and squaring two rasters.
Raster algebra also allows logical operations such as finding all raster cells that are greater than a specific value (5 in our example below).
The **raster** package supports all these operations and more, as described in `vignette("Raster")` and demonstrated below (results not shown):


```r
elev + elev
elev^2
log(elev)
elev > 5
```

Instead of arithmetic operators, one can also use the `calc()` and `overlay()` functions.
These functions are more efficient, hence, they are preferable in the presence of large raster datasets. 
Additionally, they allow you to directly store an output file.

The calculation of the normalized difference vegetation index (NDVI) is a well-known local (pixel-by-pixel) raster operation.
It returns a raster with values between -1 and 1; positive values indicate the presence of living plants (mostly > 0.2).
NDVI is calculated from red and near-infrared (NIR) bands of remotely sensed imagery, typically from satellite systems such as Landsat or Sentinel.
Vegetation absorbs light heavily in the visible light spectrum, and especially in the red channel, while reflecting NIR light, explaining the NVDI formula:

$$
\begin{split}
NDVI&= \frac{\text{NIR} - \text{Red}}{\text{NIR} + \text{Red}}\\
\end{split}
$$

Predictive mapping is another interesting application of local raster operations.
The response variable corresponds to measured or observed points in space, for example, species richness, the presence of landslides, tree disease or crop yield.
Consequently, we can easily retrieve space- or airborne predictor variables from various rasters (elevation, pH, precipitation, temperature, landcover, soil class, etc.).
Subsequently, we model our response as a function of our predictors using `lm`, `glm`, `gam` or a machine-learning technique. 
Spatial predictions on raster objects can therefore be made by applying estimated coefficients to the predictor raster values, and summing the output raster values (see Chapter \@ref(eco)).

### Focal operations

\index{map algebra!focal operations}
While local functions operate on one cell, though possibly from multiple layers, **focal** operations take into account a central cell and its neighbors.
The neighborhood (also named kernel, filter or moving window) under consideration is typically of size 3-by-3 cells (that is the central cell and its eight surrounding neighbors), but can take on any other (not necessarily rectangular) shape as defined by the user.
A focal operation applies an aggregation function to all cells within the specified neighborhood, uses the corresponding output as the new value for the the central cell, and moves on to the next central cell (Figure \@ref(fig:focal-example)).
Other names for this operation are spatial filtering and convolution [@burrough_principles_2015].

In R, we can use the `focal()` function to perform spatial filtering. 
We define the shape of the moving window with a `matrix` whose values correspond to weights (see `w` parameter in the code chunk below).
Secondly, the `fun` parameter lets us specify the function we wish to apply to this neighborhood.
Here, we choose the minimum, but any other summary function, including `sum()`, `mean()`, or `var()` can be used.


```r
r_focal = focal(elev, w = matrix(1, nrow = 3, ncol = 3), fun = min)
```

<div class="figure" style="text-align: center">
<img src="figures/04_focal_example.png" alt="Input raster (left) and resulting output raster (right) due to a focal operation - finding the minimum value in 3-by-3 moving windows." width="100%" />
<p class="caption">(\#fig:focal-example)Input raster (left) and resulting output raster (right) due to a focal operation - finding the minimum value in 3-by-3 moving windows.</p>
</div>

We can quickly check if the output meets our expectations.
In our example, the minimum value has to be always the upper left corner of the moving window (remember we have created the input raster by row-wise incrementing the cell values by one starting at the upper left corner).
In this example, the weighting matrix consists only of 1s, meaning each cell has the same weight on the output, but this can be changed.

Focal functions or filters play a dominant role in image processing.
Low-pass or smoothing filters use the mean function to remove extremes.
In the case of categorical data, we can replace the mean with the mode, which is the most common value.
By contrast, high-pass filters accentuate features.
The line detection Laplace and Sobel filters might serve as an example here.
Check the `focal()` help page for how to use them in R (this will also be used in the excercises at the end of this chapter).

Terrain processing, the calculation of topographic characteristics such as slope, aspect and flow directions, relies on focal functions.
`terrain()` can be used to calculate these metrics, although some terrain algorithms, including the Zevenbergen and Thorne method to compute slope, are not implemented in this **raster** function.
Many other algorithms --- including curvatures, contributing areas and wetness indices --- are implemented in open source desktop geographic information system (GIS) software.
Chapter \@ref(gis) shows how to access such GIS functionality from within R.

### Zonal operations

\index{map algebra!zonal operations}
Just like focal operations, *zonal* operations apply an aggregation function to multiple raster cells.
However, a second raster, usually a categorical raster, defines the *zonal filters* (or 'zones') in the case of zonal operations as opposed to a predefined neighborhood window in the case of focal operations (see previous Section).
Consequently, the raster cells defining the zonal filter do not necessarily have to be neighbors.
Our grain size raster is a good example (right panel of Figure \@ref(fig:cont-raster)) because the different grain sizes are spread in an irregular fashion throughout the raster.
Finally, the result of a zonal operation is a summary table grouped by zone which is why this operation is also known as *zonal statistics* in the GIS world\index{GIS}. 
This is in contrast to focal operations which return a raster object (see previous Section).

For example, to find the mean elevation for each grain size class (Figure \@ref(fig:cont-raster)), we use the `zonal()` function.


```r
z = zonal(elev, grain, fun = "mean") %>%
  as.data.frame()
z
#>   zone mean
#> 1    1 17.8
#> 2    2 18.5
#> 3    3 19.2
```

This returns the statistics\index{statistics} for each category, here the mean altitude for each grain size class, and can be added to the attribute table of the ratified raster (see previous chapter).

### Global operations and distances

*Global* operations are a special case of zonal operations with the entire raster dataset representing a single zone.
The most common global operations are descriptive statistics\index{statistics} for the entire raster dataset such as the minimum or maximum (see Section \@ref(summarizing-raster-objects)).
Aside from that, global operations are also useful for the computation of distance and weight rasters.
In the first case, one can calculate the distance from each cell to a specific target cell.
For example, one might want to compute the distance to the nearest coast (see also `raster::distance()`).
We might also want to consider topography, that means, we are not only interested in the pure distance but would like also to avoid the crossing of mountain ranges when going to the coast.
To do so, we can weight the distance with elevation so that each additional altitudinal meter 'prolongs' the Euclidean distance.
Visibility and viewshed computations also belong to the family of global operations (in the exercises of Chapter \@ref(gis), you will compute a viewshed raster).

### Map algebra counterparts in vector processing

Many map algebra operations have a counterpart in vector processing [@liu_essential_2009].
Computing a distance raster (zonal operation) while only considering a maximum distance (logical focal operation) is the equivalent to a vector buffer operation (Section \@ref(clipping)).
Reclassifying raster data (either local or zonal function depending on the input) is equivalent to dissolving vector data (Section \@ref(spatial-joining)). 
Overlaying two rasters (local operation), where one contains `NULL` or `NA` values representing a mask, is similar to vector clipping (Section \@ref(clipping)).
Quite similar to spatial clipping is intersecting two layers (Section \@ref(spatial-subsetting)). 
The difference is that these two layers (vector or raster) simply share an overlapping area (see Figure \@ref(fig:venn-clip) for an example).
However, be careful with the wording.
Sometimes the same words have slightly different meanings for raster and vector data models.
Aggregating in the case of vector data refers to dissolving polygons, while it means increasing the resolution in the case of raster data.
In fact, one could see dissolving or aggregating polygons as decreasing the resolution. 
However, zonal operations might be the better raster equivalent compared to changing the cell resolution. 
Zonal operations can dissolve the cells of one raster in accordance with the zones (categories) of another raster using an aggregation function (see above).

### Merging rasters

\index{raster!merge}
Suppose we would like to compute the NDVI (see Section \@ref(local-operations)), and additionally want to compute terrain attributes from elevation data for observations within a study area.
Such computations rely on remotely sensed information.
The corresponding imagery is often divided into scenes covering a specific spatial extent.
Frequently, a study area covers more than one scene.
In these cases we would like to merge the scenes covered by our study area. 
In the easiest case, we can just merge these scenes, that is put them side by side.
This is possible with digital elevation data (SRTM, ASTER).
In the following code chunk we first download the SRTM elevation data for Austria and Switzerland (for the country codes, see the **raster** function `ccodes()`).
In a second step, we merge the two rasters into one.


```r
aut = getData("alt", country = "AUT", mask = TRUE)
ch = getData("alt", country = "CHE", mask = TRUE)
aut_ch = merge(aut, ch)
```

**Raster**'s `merge()` command combines two images, and in case they overlap, it uses the value of the first raster.
You can do exactly the same with `gdalUtils::mosaic_rasters()` which is faster, and therefore recommended if you have to merge a multitude of large rasters stored on disk.

The merging approach is of little use when the overlapping values do not correspond to each other.
This is frequently the case when you want to combine spectral imagery from scenes that were taken on different dates.
The `merge()` command will still work but you will see a clear border in the resulting image.
The `mosaic()` command lets you define a function for the overlapping area. 
For instance, we could compute the mean value. 
This might smooth the clear border in the merged result but it will most likely not make it disappear.
To do so, we need a more advanced approach. 
Remote sensing scientists frequently apply histogram matching or use regression techniques to align the values of the first image with those of the second image.
The packages **landsat** (`histmatch()`, `relnorm()`, `PIF()`), **satellite** (`calcHistMatch()`) and **RStoolbox** (`histMatch()`, `pifMatch()`) provide the corresponding functions.
For a more detailed introduction on how to use R for remote sensing, we refer the reader to @wegmann_remote_2016.

<!-- ## Spatial data creation -->

<!-- where should "area" example be? in this or the previous chapter? -->
<!-- Not here - I think this chapter should focus on geomtry data -->
<!-- `st_centroid()` -->
<!-- `st_buffer()` -->
<!-- http://r-spatial.org//r/2017/06/09/mapedit_0-2-0.html -->

<!-- Commented out - think this would be better in c3 (RL) -->
<!-- ```{r} -->
<!-- # add a new column -->
<!-- africa$area = set_units(st_area(africa), value = km^2) -->
<!-- africa$pop_density = africa$pop / africa$area -->

<!-- # OR -->
<!-- africa = africa %>% -->
<!--         mutate(area = set_units(st_area(.), value = km^2)) %>% -->
<!--         mutate(pop_density = pop / area) -->
<!-- ``` -->

<!-- Note that this has created a attributes for the area and population density variables: -->

<!-- ```{r} -->
<!-- attributes(africa$area) -->
<!-- attributes(africa$pop_density) -->
<!-- ``` -->

<!-- These can be set to `NULL` as follows: -->

<!-- ```{r} -->
<!-- attributes(africa$area) = NULL -->
<!-- attributes(africa$pop_density) = NULL -->
<!-- ``` -->

<!-- ## Spatial data transformation -->
<!-- changes classes; polygonize, etc-->

## Exercises

1. It was established in Section \@ref(spatial-vec) that Canterbury was the region of New Zealand containing most of the 100 highest points in the country.
How many of these high points does the Canterbury region contain?

1. Which region has the second highest number of `nz_height` points in, and how many does it have?

1. Generalizing the question to all regions: how many of New Zealand's 16 regions contain points which belong to the top 100 highest points in the country? Which regions?
    - Bonus: create a table listing these regions in order of the number of points and their name.

1. Use `data(dem, package = "spDataLarge")`, and reclassify the elevation in three classes: low, medium and high.
Secondly, attach the NDVI raster (`data(ndvi, package = "spDataLarge")`) and compute the mean NDVI and the mean elevation for each altitudinal class.
1. Apply a line detection filter to `raster(system.file("external/rlogo.grd", package = "raster"))`.
Plot the result.
Hint: Read `?raster::focal()`.
1. Calculate the NDVI of a Landsat image. 
Use the Landsat image provided by the **spDataLarge** package (`system.file("raster/landsat.tif", package="spDataLarge")`).
1. A StackOverflow [post](https://stackoverflow.com/questions/35555709/global-raster-of-geographic-distances) shows how to compute distances to the nearest coastline using `raster::distance()`.
Retrieve a digital elevation model of Spain, and compute a raster which represents distances to the coast across the country (hint: use `getData()`).
Second, use a simple approach to weight the distance raster with elevation (other weighting approaches are possible, include flow direction and steepness); every 100 altitudinal meters should increase the distance to the coast by 10 km.
Finally, compute the difference between the raster using the Euclidean distance and the raster weighted by elevation.
Note: it may be wise to increase the cell size of the input raster to reduce compute time during this operation.
