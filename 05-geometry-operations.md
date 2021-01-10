# Geometry operations {#geometric-operations}

## Prerequisites {-}

- This chapter uses the same packages as Chapter \@ref(spatial-operations) but with the addition of **spDataLarge**, which was installed in Chapter \@ref(spatial-class):


```r
library(sf)
library(raster)
library(dplyr)
library(spData)
library(spDataLarge)
```

## Introduction

The previous three chapters have demonstrated how geographic datasets are structured in R (Chapter \@ref(spatial-class)) and how to manipulate them based on their non-geographic attributes (Chapter \@ref(attr)) and spatial properties (Chapter \@ref(spatial-operations)).
This chapter extends these skills.
After reading it --- and attempting the exercises at the end --- you should understand and have control over the geometry column in `sf` objects and the geographic location of pixels represented in rasters.

Section \@ref(geo-vec) covers transforming vector geometries with 'unary' and 'binary' operations.
<!-- TODO: add something on n-ary ops (RL) -->
Unary operations work on a single geometry in isolation.
This includes simplification (of lines and polygons), the creation of buffers and centroids, and shifting/scaling/rotating single geometries using 'affine transformations' (Sections \@ref(simplification) to \@ref(affine-transformations)).
Binary transformations modify one geometry based on the shape of another.
This includes clipping and geometry unions\index{vector!union}, covered in Sections \@ref(clipping) and \@ref(geometry-unions), respectively.
Type transformations (from a polygon to a line, for example) are demonstrated in Section \@ref(type-trans).

Section \@ref(geo-ras) covers geometric transformations on raster objects.
This involves changing the size and number of the underlying pixels, and assigning them new values.
It teaches how to change the resolution (also called raster aggregation and disaggregation), the extent and the origin of a raster.
These operations are especially useful if one would like to align raster datasets from diverse sources.
Aligned raster objects share a one-to-one correspondence between pixels, allowing them to be processed using map algebra operations, described in Section \@ref(map-algebra). The final Section \@ref(raster-vector) connects vector and raster objects. 
It shows how raster values can be 'masked' and 'extracted' by vector geometries.
Importantly it shows how to 'polygonize' rasters and 'rasterize' vector datasets, making the two data models more interchangeable.

## Geometric operations on vector data {#geo-vec}

This section is about operations that in some way change the geometry of vector (`sf`) objects.
It is more advanced than the spatial data operations presented in the previous chapter (in Section \@ref(spatial-vec)), because here we drill down into the geometry:
the functions discussed in this section work on objects of class `sfc` in addition to objects of class `sf`.

### Simplification

\index{vector!simplification} 
Simplification is a process for generalization of vector objects (lines and polygons) usually for use in smaller scale maps.
Another reason for simplifying objects is to reduce the amount of memory, disk space and network bandwidth they consume:
it may be wise to simplify complex geometries before publishing them as interactive maps. 
The **sf** package provides `st_simplify()`, which uses the GEOS implementation of the Douglas-Peucker algorithm to reduce the vertex count.
`st_simplify()` uses the `dTolerance` to control the level of generalization in map units [see @douglas_algorithms_1973 for details].
<!-- I have no idea what the next sentence means -->
<!-- As a result, all vertices in the simplified geometry will be within this value from the original ones. -->
Figure \@ref(fig:seine-simp) illustrates simplification of a `LINESTRING` geometry representing the river Seine and tributaries.
The simplified geometry was created by the following command:


```r
seine_simp = st_simplify(seine, dTolerance = 2000)  # 2000 m
```

<div class="figure" style="text-align: center">
<img src="05-geometry-operations_files/figure-html/seine-simp-1.png" alt="Comparison of the original and simplified geometry of the seine object." width="100%" />
<p class="caption">(\#fig:seine-simp)Comparison of the original and simplified geometry of the seine object.</p>
</div>

The resulting `seine_simp` object is a copy of the original `seine` but with fewer vertices.
This is apparent, with the result being visually simpler (Figure \@ref(fig:seine-simp), right) and consuming less memory than the original object, as verified below:


```r
object.size(seine)
#> 18096 bytes
object.size(seine_simp)
#> 9112 bytes
```

Simplification is also applicable for polygons.
This is illustrated using `us_states`, representing the contiguous United States.
As we show in Chapter \@ref(reproj-geo-data), GEOS assumes that the data is in a projected CRS and this could lead to unexpected results when using a geographic CRS.
Therefore, the first step is to project the data into some adequate projected CRS, such as US National Atlas Equal Area (epsg = 2163) (on the left in Figure \@ref(fig:us-simp)):


```r
us_states2163 = st_transform(us_states, 2163)
```

`st_simplify()` works equally well with projected polygons:


```r
us_states_simp1 = st_simplify(us_states2163, dTolerance = 100000)  # 100 km
```

A limitation with `st_simplify()` is that it simplifies objects on a per-geometry basis.
This means the 'topology' is lost, resulting in overlapping and 'holy' areal units illustrated in Figure \@ref(fig:us-simp) (middle panel).
`ms_simplify()` from **rmapshaper** provides an alternative that overcomes this issue.
By default it uses the Visvalingam algorithm, which overcomes some limitations of the Douglas-Peucker algorithm [@visvalingam_line_1993].
<!-- https://bost.ocks.org/mike/simplify/ -->
The following code chunk uses this function to simplify `us_states2163`.
The result has only 1% of the vertices of the input (set using the argument `keep`) but its number of objects remains intact because we set `keep_shapes = TRUE`:^[
Simplification of multipolygon objects can remove small internal polygons, even if the `keep_shapes` argument is set to TRUE. To prevent this, you need to set `explode = TRUE`. This option converts all mutlipolygons into separate polygons before its simplification.
]


```r
# proportion of points to retain (0-1; default 0.05)
us_states2163$AREA = as.numeric(us_states2163$AREA)
us_states_simp2 = rmapshaper::ms_simplify(us_states2163, keep = 0.01,
                                          keep_shapes = TRUE)
```

Finally, the visual comparison of the original dataset and the two simplified versions shows differences between the Douglas-Peucker (`st_simplify`) and Visvalingam (`ms_simplify`) algorithm outputs (Figure \@ref(fig:us-simp)):

<div class="figure" style="text-align: center">
<img src="05-geometry-operations_files/figure-html/us-simp-1.png" alt="Polygon simplification in action, comparing the original geometry of the contiguous United States with simplified versions, generated with functions from sf (center) and rmapshaper (right) packages." width="100%" />
<p class="caption">(\#fig:us-simp)Polygon simplification in action, comparing the original geometry of the contiguous United States with simplified versions, generated with functions from sf (center) and rmapshaper (right) packages.</p>
</div>

### Centroids

\index{vector!centroids} 
Centroid operations identify the center of geographic objects.
Like statistical measures of central tendency (including mean and median definitions of 'average'), there are many ways to define the geographic center of an object.
All of them create single point representations of more complex vector objects.

The most commonly used centroid operation is the *geographic centroid*.
This type of centroid operation (often referred to as 'the centroid') represents the center of mass in a spatial object (think of balancing a plate on your finger).
Geographic centroids have many uses, for example to create a simple point representation of complex geometries, or to estimate distances between polygons.
They can be calculated with the **sf** function `st_centroid()` as demonstrated in the code below, which generates the geographic centroids of regions in New Zealand and tributaries to the River Seine, illustrated with black points in Figure \@ref(fig:centr).


```r
nz_centroid = st_centroid(nz)
seine_centroid = st_centroid(seine)
```

Sometimes the geographic centroid falls outside the boundaries of their parent objects (think of a doughnut).
In such cases *point on surface* operations can be used to guarantee the point will be in the parent object (e.g., for labeling irregular multipolygon objects such as island states), as illustrated by the red points in Figure \@ref(fig:centr).
Notice that these red points always lie on their parent objects.
They were created with `st_point_on_surface()` as follows:^[
A description of how `st_point_on_surface()` works is provided at https://gis.stackexchange.com/q/76498.
]


```r
nz_pos = st_point_on_surface(nz)
seine_pos = st_point_on_surface(seine)
```

<div class="figure" style="text-align: center">
<img src="05-geometry-operations_files/figure-html/centr-1.png" alt="Centroids (black points) and 'points on surface' (red points) of New Zealand's regions (left) and the Seine (right) datasets." width="100%" />
<p class="caption">(\#fig:centr)Centroids (black points) and 'points on surface' (red points) of New Zealand's regions (left) and the Seine (right) datasets.</p>
</div>

Other types of centroids exist, including the *Chebyshev center* and the *visual center*.
We will not explore these here but it is possible to calculate them using R, as we'll see in Chapter \@ref(algorithms).

### Buffers

\index{vector!buffers} 
Buffers are polygons representing the area within a given distance of a geometric feature:
regardless of whether the input is a point, line or polygon, the output is a polygon.
Unlike simplification (which is often used for visualization and reducing file size) buffering tends to be used for geographic data analysis.
How many points are within a given distance of this line?
Which demographic groups are within travel distance of this new shop?
These kinds of questions can be answered and visualized by creating buffers around the geographic entities of interest.

Figure \@ref(fig:buffs) illustrates buffers of different sizes (5 and 50 km) surrounding the river Seine and tributaries.
These buffers were created with commands below, which show that the command `st_buffer()` requires at least two arguments: an input geometry and a distance, provided in the units of the CRS (in this case meters):


```r
seine_buff_5km = st_buffer(seine, dist = 5000)
seine_buff_50km = st_buffer(seine, dist = 50000)
```

<div class="figure" style="text-align: center">
<img src="05-geometry-operations_files/figure-html/buffs-1.png" alt="Buffers around the Seine dataset of 5 km (left) and 50 km (right). Note the colors, which reflect the fact that one buffer is created per geometry feature." width="75%" />
<p class="caption">(\#fig:buffs)Buffers around the Seine dataset of 5 km (left) and 50 km (right). Note the colors, which reflect the fact that one buffer is created per geometry feature.</p>
</div>

\BeginKnitrBlock{rmdnote}<div class="rmdnote">The third and final argument of `st_buffer()` is `nQuadSegs`, which means 'number of segments per quadrant' and is set by default to 30 (meaning circles created by buffers are composed of $4 \times 30 = 120$ lines).
This argument rarely needs to be set.
Unusual cases where it may be useful include when the memory consumed by the output of a buffer operation is a major concern (in which case it should be reduced) or when very high precision is needed (in which case it should be increased).</div>\EndKnitrBlock{rmdnote}



### Affine transformations

\index{vector!affine transformation} 
Affine transformation is any transformation that preserves lines and parallelism.
<!-- The midpoint of a line segment remains a midpoint and all points lying on a line initially still lie on a line after an affine transformation. -->
However, angles or length are not necessarily preserved.
Affine transformations include, among others, shifting (translation), scaling and rotation.
<!-- translation, scaling, homothety, similarity transformation, reflection, rotation, shear mapping -->
Additionally, it is possible to use any combination of these.
Affine transformations are an essential part of geocomputation.
For example, shifting is needed for labels placement, scaling is used in non-contiguous area cartograms (see Section \@ref(other-mapping-packages)), and many affine transformations are applied when reprojecting or improving the geometry that was created based on a distorted or wrongly projected map.
The **sf** package implements affine transformation for objects of classes `sfg` and `sfc`.


```r
nz_sfc = st_geometry(nz)
```

Shifting moves every point by the same distance in map units.
It could be done by adding a numerical vector to a vector object.
For example, the code below shifts all y-coordinates by 100,000 meters to the north, but leaves the x-coordinates untouched (left panel of Figure \@ref(fig:affine-trans)). 


```r
nz_shift = nz_sfc + c(0, 100000)
```

Scaling enlarges or shrinks objects by a factor.
It can be applied either globally or locally. <!-- my terms - jn-->
Global scaling increases or decreases all coordinates values in relation to the origin coordinates, while keeping all geometries topological relations intact.
It can be done by subtraction or multiplication of a`sfg` or `sfc` object.



Local scaling treats geometries independently and requires points around which geometries are going to be scaled, e.g., centroids.
In the example below, each geometry is shrunk by a factor of two around the centroids (middle panel in Figure \@ref(fig:affine-trans)).
To achieve that, each object is firstly shifted in a way that its center has coordinates of `0, 0` (`(nz_sfc - nz_centroid_sfc)`). 
Next, the sizes of the geometries are reduced by half (`* 0.5`).
Finally, each object's centroid is moved back to the input data coordinates (`+ nz_centroid_sfc`). 


```r
nz_centroid_sfc = st_centroid(nz_sfc)
nz_scale = (nz_sfc - nz_centroid_sfc) * 0.5 + nz_centroid_sfc
```

Rotation of two-dimensional coordinates requires a rotation matrix:

$$
R =
\begin{bmatrix}
\cos \theta & -\sin \theta \\  
\sin \theta & \cos \theta \\
\end{bmatrix}
$$

It rotates points in a clockwise direction.
The rotation matrix can be implemented in R as:
<!-- https://r-spatial.github.io/sf/articles/sf3.html#affine-transformations -->

```r
rotation = function(a){
  r = a * pi / 180 #degrees to radians
  matrix(c(cos(r), sin(r), -sin(r), cos(r)), nrow = 2, ncol = 2)
} 
```

The `rotation` function accepts one argument `a` - a rotation angle in degrees.
Rotation could be done around selected points, such as centroids (right panel of Figure \@ref(fig:affine-trans)).
See `vignette("sf3")` for more examples.


```r
nz_rotate = (nz_sfc - nz_centroid_sfc) * rotation(30) + nz_centroid_sfc
```

<div class="figure" style="text-align: center">
<img src="05-geometry-operations_files/figure-html/affine-trans-1.png" alt="Illustrations of affine transformations: shift, scale and rotate." width="100%" />
<p class="caption">(\#fig:affine-trans)Illustrations of affine transformations: shift, scale and rotate.</p>
</div>





Finally, the newly created geometries can replace the old ones with the `st_set_geometry()` function: 


```r
nz_scale_sf = st_set_geometry(nz, nz_scale)
```

### Clipping {#clipping}

\index{vector!clipping} 
\index{spatial!subsetting} 
Spatial clipping is a form of spatial subsetting that involves changes to the `geometry` columns of at least some of the affected features.

Clipping can only apply to features more complex than points: 
lines, polygons and their 'multi' equivalents.
To illustrate the concept we will start with a simple example:
two overlapping circles with a center point one unit away from each other and a radius of one (Figure \@ref(fig:points)).


```r
b = st_sfc(st_point(c(0, 1)), st_point(c(1, 1))) # create 2 points
b = st_buffer(b, dist = 1) # convert points to circles
plot(b)
text(x = c(-0.5, 1.5), y = 1, labels = c("x", "y")) # add text
```

<div class="figure" style="text-align: center">
<img src="05-geometry-operations_files/figure-html/points-1.png" alt="Overlapping circles." width="50%" />
<p class="caption">(\#fig:points)Overlapping circles.</p>
</div>

Imagine you want to select not one circle or the other, but the space covered by both `x` *and* `y`.
This can be done using the function `st_intersection()`\index{vector!intersection}, illustrated using objects named `x` and `y` which represent the left- and right-hand circles (Figure \@ref(fig:circle-intersection)).


```r
x = b[1]
y = b[2]
x_and_y = st_intersection(x, y)
plot(b)
plot(x_and_y, col = "lightgrey", add = TRUE) # color intersecting area
```

<div class="figure" style="text-align: center">
<img src="05-geometry-operations_files/figure-html/circle-intersection-1.png" alt="Overlapping circles with a gray color indicating intersection between them." width="50%" />
<p class="caption">(\#fig:circle-intersection)Overlapping circles with a gray color indicating intersection between them.</p>
</div>

The subsequent code chunk demonstrates how this works for all combinations of the 'Venn' diagram representing `x` and `y`, inspired by [Figure 5.1](http://r4ds.had.co.nz/transform.html#logical-operators) of the book *R for Data Science* [@grolemund_r_2016].
<!-- Todo: reference r4ds -->

<div class="figure" style="text-align: center">
<img src="05-geometry-operations_files/figure-html/venn-clip-1.png" alt="Spatial equivalents of logical operators." width="100%" />
<p class="caption">(\#fig:venn-clip)Spatial equivalents of logical operators.</p>
</div>

To illustrate the relationship between subsetting and clipping spatial data, we will subset points that cover the bounding box of the circles `x` and `y` in Figure \@ref(fig:venn-clip).
Some points will be inside just one circle, some will be inside both and some will be inside neither.
`st_sample()` is used below to generate a *simple random* distribution of points within the extent of circles `x` and `y`, resulting in output illustrated in Figure \@ref(fig:venn-subset).


```r
bb = st_bbox(st_union(x, y))
box = st_as_sfc(bb)
set.seed(2017)
p = st_sample(x = box, size = 10)
plot(box)
plot(x, add = TRUE)
plot(y, add = TRUE)
plot(p, add = TRUE)
text(x = c(-0.5, 1.5), y = 1, labels = c("x", "y"))
```

<div class="figure" style="text-align: center">
<img src="05-geometry-operations_files/figure-html/venn-subset-1.png" alt="Randomly distributed points within the bounding box enclosing circles x and y." width="50%" />
<p class="caption">(\#fig:venn-subset)Randomly distributed points within the bounding box enclosing circles x and y.</p>
</div>

The logical operator way would find the points inside both `x` and `y` using a spatial predicate such as `st_intersects()`, whereas the intersection\index{vector!intersection} method simply finds the points inside the intersecting region created above as `x_and_y`.
As demonstrated below the results are identical, but the method that uses the clipped polygon is more concise:


```r
sel_p_xy = st_intersects(p, x, sparse = FALSE)[, 1] &
  st_intersects(p, y, sparse = FALSE)[, 1]
p_xy1 = p[sel_p_xy]
p_xy2 = p[x_and_y]
identical(p_xy1, p_xy2)
#> [1] TRUE
```




### Geometry unions

\index{vector!union} 
\index{aggregation!spatial} 
As we saw in Section \@ref(vector-attribute-aggregation), spatial aggregation can silently dissolve the geometries of touching polygons in the same group.
This is demonstrated in the code chunk below in which 49 `us_states` are aggregated into 4 regions using base and **tidyverse**\index{tidyverse (package)} functions (see results in Figure \@ref(fig:us-regions)):


```r
regions = aggregate(x = us_states[, "total_pop_15"], by = list(us_states$REGION),
                    FUN = sum, na.rm = TRUE)
regions2 = us_states %>% group_by(REGION) %>%
  summarize(pop = sum(total_pop_15, na.rm = TRUE))
#> `summarise()` ungrouping output (override with `.groups` argument)
#> although coordinates are longitude/latitude, st_union assumes that they are planar
#> although coordinates are longitude/latitude, st_union assumes that they are planar
#> although coordinates are longitude/latitude, st_union assumes that they are planar
#> although coordinates are longitude/latitude, st_union assumes that they are planar
```



<div class="figure" style="text-align: center">
<img src="05-geometry-operations_files/figure-html/us-regions-1.png" alt="Spatial aggregation on contiguous polygons, illustrated by aggregating the population of US states into regions, with population represented by color. Note the operation automatically dissolves boundaries between states." width="100%" />
<p class="caption">(\#fig:us-regions)Spatial aggregation on contiguous polygons, illustrated by aggregating the population of US states into regions, with population represented by color. Note the operation automatically dissolves boundaries between states.</p>
</div>

What is going on in terms of the geometries?
Behind the scenes, both `aggregate()` and `summarize()` combine the geometries and dissolve the boundaries between them using `st_union()`.
This is demonstrated in the code chunk below which creates a united western US: 


```r
us_west = us_states[us_states$REGION == "West", ]
us_west_union = st_union(us_west)
#> although coordinates are longitude/latitude, st_union assumes that they are planar
```

The function can take two geometries and unite them, as demonstrated in the code chunk below which creates a united western block incorporating Texas (challenge: reproduce and plot the result):


```r
texas = us_states[us_states$NAME == "Texas", ]
texas_union = st_union(us_west_union, texas)
```



### Type transformations {#type-trans}

\index{vector!geometry casting} 
Geometry casting is a powerful operation that enables transformation of the geometry type.
It is implemented in the `st_cast` function from the **sf** package.
Importantly, `st_cast` behaves differently on single simple feature geometry (`sfg`) objects, simple feature geometry column (`sfc`) and simple features objects.

Let's create a multipoint to illustrate how geometry casting works on simple feature geometry (`sfg`) objects:


```r
multipoint = st_multipoint(matrix(c(1, 3, 5, 1, 3, 1), ncol = 2))
```

In this case, `st_cast` can be useful to transform the new object into linestring or polygon (Figure \@ref(fig:single-cast)):

<!-- a/ points -> lines -> polygons  -->

```r
linestring = st_cast(multipoint, "LINESTRING")
polyg = st_cast(multipoint, "POLYGON")
```

<div class="figure" style="text-align: center">
<img src="05-geometry-operations_files/figure-html/single-cast-1.png" alt="Examples of linestring and polygon casted from a multipoint geometry." width="100%" />
<p class="caption">(\#fig:single-cast)Examples of linestring and polygon casted from a multipoint geometry.</p>
</div>

Conversion from multipoint to linestring is a common operation that creates a line object from ordered point observations, such as GPS measurements or geotagged media.
This allows spatial operations such as the length of the path traveled.
Conversion from multipoint or linestring to polygon is often used to calculate an area, for example from the set of GPS measurements taken around a lake or from the corners of a building lot.

The transformation process can be also reversed using `st_cast`:


```r
multipoint_2 = st_cast(linestring, "MULTIPOINT")
multipoint_3 = st_cast(polyg, "MULTIPOINT")
all.equal(multipoint, multipoint_2, multipoint_3)
#> [1] TRUE
```

\BeginKnitrBlock{rmdnote}<div class="rmdnote">For single simple feature geometries (`sfg`), `st_cast` also provides geometry casting from non-multi-types to multi-types (e.g., `POINT` to `MULTIPOINT`) and from multi-types to non-multi-types.
However, only the first element of the old object would remain in the second group of cases.</div>\EndKnitrBlock{rmdnote}



Geometry casting of simple features geometry column (`sfc`) and simple features objects works the same as for single geometries in most of the cases. 
One important difference is the conversion between multi-types to non-multi-types.
As a result of this process, multi-objects are split into many non-multi-objects.

Table \@ref(tab:sfs-st-cast) shows possible geometry type transformations on simple feature objects.
Each input simple feature object with only one element (first column) is transformed directly into another geometry type.
Several of the transformations are not possible, for example, you cannot convert a single point into a multilinestring or a polygon (so the cells `[1, 4:5]` in the table are NA).
On the other hand, some of the transformations are splitting the single element input object into a multi-element object.
You can see that, for example, when you cast a multipoint consisting of five pairs of coordinates into a point.

<table>
<caption>(\#tab:sfs-st-cast)Geometry casting on simple feature geometries (see Section 2.1) with input type by row and output type by column</caption>
 <thead>
  <tr>
   <th style="text-align:left;">  </th>
   <th style="text-align:right;"> POI </th>
   <th style="text-align:right;"> MPOI </th>
   <th style="text-align:right;"> LIN </th>
   <th style="text-align:right;"> MLIN </th>
   <th style="text-align:right;"> POL </th>
   <th style="text-align:right;"> MPOL </th>
   <th style="text-align:right;"> GC </th>
  </tr>
 </thead>
<tbody>
  <tr>
   <td style="text-align:left;"> POI(1) </td>
   <td style="text-align:right;"> 1 </td>
   <td style="text-align:right;"> 1 </td>
   <td style="text-align:right;"> 1 </td>
   <td style="text-align:right;"> NA </td>
   <td style="text-align:right;"> NA </td>
   <td style="text-align:right;"> NA </td>
   <td style="text-align:right;"> NA </td>
  </tr>
  <tr>
   <td style="text-align:left;"> MPOI(1) </td>
   <td style="text-align:right;"> 4 </td>
   <td style="text-align:right;"> 1 </td>
   <td style="text-align:right;"> 1 </td>
   <td style="text-align:right;"> 1 </td>
   <td style="text-align:right;"> 1 </td>
   <td style="text-align:right;"> NA </td>
   <td style="text-align:right;"> NA </td>
  </tr>
  <tr>
   <td style="text-align:left;"> LIN(1) </td>
   <td style="text-align:right;"> 5 </td>
   <td style="text-align:right;"> 1 </td>
   <td style="text-align:right;"> 1 </td>
   <td style="text-align:right;"> 1 </td>
   <td style="text-align:right;"> 1 </td>
   <td style="text-align:right;"> NA </td>
   <td style="text-align:right;"> NA </td>
  </tr>
  <tr>
   <td style="text-align:left;"> MLIN(1) </td>
   <td style="text-align:right;"> 7 </td>
   <td style="text-align:right;"> 2 </td>
   <td style="text-align:right;"> 2 </td>
   <td style="text-align:right;"> 1 </td>
   <td style="text-align:right;"> NA </td>
   <td style="text-align:right;"> NA </td>
   <td style="text-align:right;"> NA </td>
  </tr>
  <tr>
   <td style="text-align:left;"> POL(1) </td>
   <td style="text-align:right;"> 5 </td>
   <td style="text-align:right;"> 1 </td>
   <td style="text-align:right;"> 1 </td>
   <td style="text-align:right;"> 1 </td>
   <td style="text-align:right;"> 1 </td>
   <td style="text-align:right;"> 1 </td>
   <td style="text-align:right;"> NA </td>
  </tr>
  <tr>
   <td style="text-align:left;"> MPOL(1) </td>
   <td style="text-align:right;"> 10 </td>
   <td style="text-align:right;"> 1 </td>
   <td style="text-align:right;"> NA </td>
   <td style="text-align:right;"> 1 </td>
   <td style="text-align:right;"> 2 </td>
   <td style="text-align:right;"> 1 </td>
   <td style="text-align:right;"> 1 </td>
  </tr>
  <tr>
   <td style="text-align:left;"> GC(1) </td>
   <td style="text-align:right;"> 9 </td>
   <td style="text-align:right;"> 1 </td>
   <td style="text-align:right;"> NA </td>
   <td style="text-align:right;"> NA </td>
   <td style="text-align:right;"> NA </td>
   <td style="text-align:right;"> NA </td>
   <td style="text-align:right;"> 1 </td>
  </tr>
</tbody>
<tfoot>
<tr>
<td style = 'padding: 0; border:0;' colspan='100%'><sup></sup> Note: Values like (1) represent the number of features; NA means the operation is not possible. Abbreviations: POI, LIN, POL and GC refer to POINT, LINESTRING, POLYGON and GEOMETRYCOLLECTION. The MULTI version of these geometry types is indicated by a preceding M, e.g., MPOI is the acronym for MULTIPOINT.</td>
</tr>
</tfoot>
</table>

Let's try to apply geometry type transformations on a new object, `multilinestring_sf`, as an example (on the left in Figure \@ref(fig:line-cast)):


```r
multilinestring_list = list(matrix(c(1, 4, 5, 3), ncol = 2), 
                            matrix(c(4, 4, 4, 1), ncol = 2),
                            matrix(c(2, 4, 2, 2), ncol = 2))
multilinestring = st_multilinestring((multilinestring_list))
multilinestring_sf = st_sf(geom = st_sfc(multilinestring))
multilinestring_sf
#> Simple feature collection with 1 feature and 0 fields
#> geometry type:  MULTILINESTRING
#> dimension:      XY
#> bbox:           xmin: 1 ymin: 1 xmax: 4 ymax: 5
#> CRS:            NA
#>                             geom
#> 1 MULTILINESTRING ((1 5, 4 3)...
```

You can imagine it as a road or river network. 
The new object has only one row that defines all the lines.
This restricts the number of operations that can be done, for example it prevents adding names to each line segment or calculating lengths of single lines.
The `st_cast` function can be used in this situation, as it separates one mutlilinestring into three linestrings:


```r
linestring_sf2 = st_cast(multilinestring_sf, "LINESTRING")
linestring_sf2
#> Simple feature collection with 3 features and 0 fields
#> geometry type:  LINESTRING
#> dimension:      XY
#> bbox:           xmin: 1 ymin: 1 xmax: 4 ymax: 5
#> CRS:            NA
#>                    geom
#> 1 LINESTRING (1 5, 4 3)
#> 2 LINESTRING (4 4, 4 1)
#> 3 LINESTRING (2 2, 4 2)
```

<div class="figure" style="text-align: center">
<img src="05-geometry-operations_files/figure-html/line-cast-1.png" alt="Examples of type casting between MULTILINESTRING (left) and LINESTRING (right)." width="100%" />
<p class="caption">(\#fig:line-cast)Examples of type casting between MULTILINESTRING (left) and LINESTRING (right).</p>
</div>

The newly created object allows for attributes creation (see more in Section \@ref(vec-attr-creation)) and length measurements:


```r
linestring_sf2$name = c("Riddle Rd", "Marshall Ave", "Foulke St")
linestring_sf2$length = st_length(linestring_sf2)
linestring_sf2
#> Simple feature collection with 3 features and 2 fields
#> geometry type:  LINESTRING
#> dimension:      XY
#> bbox:           xmin: 1 ymin: 1 xmax: 4 ymax: 5
#> CRS:            NA
#>                    geom         name length
#> 1 LINESTRING (1 5, 4 3)    Riddle Rd   3.61
#> 2 LINESTRING (4 4, 4 1) Marshall Ave   3.00
#> 3 LINESTRING (2 2, 4 2)    Foulke St   2.00
```

<!-- idea: example where a collection of points referring to different objects are combined to MULTIPOINT by object, then casted to LINESTRING geometries per object -->

## Geometric operations on raster data {#geo-ras}

\index{raster!manipulation} 
Geometric raster operations include the shift, flipping, mirroring, scaling, rotation or warping of images.
These operations are necessary for a variety of applications including georeferencing, used to allow images to be overlaid on an accurate map with a known CRS [@liu_essential_2009].
A variety of georeferencing techniques exist, including

- georectification based on known [ground control points](http://www.qgistutorials.com/en/docs/georeferencing_basics.html); 
- orthorectification, which also accounts for local topography; and
- image [registration](https://en.wikipedia.org/wiki/Image_registration) is used to combine images of the same thing but shot from different sensors the process of aligning one image with another (in terms of coordinate system, and resolution). 

R is unsuitable for the first two points since these often require manual intervention which is why they are usually done with the help of dedicated GIS software (see also Chapter \@ref(gis)).
On the other hand, aligning several images is possible in R and this section shows among others how to do so.
This often includes changing the extent, the resolution and the origin of an image.
A matching projection is of course also required but is already covered in Section \@ref(reprojecting-raster-geometries).
In any case, there are other reasons to perform a geometric operation on a single raster image.
For instance, in Chapter \@ref(location) we define metropolitan areas in Germany as 20 km^2^ pixels with more than 500,000 inhabitants. 
The original inhabitant raster, however, has a resolution of 1 km^2^ which is why we will decrease (aggregate) the resolution by a factor of 20 (see Section \@ref(define-metropolitan-areas)).
Another reason for aggregating a raster is simply to decrease run-time or save disk space.
Of course, this is only possible if the task at hand allows a coarser resolution.
Sometimes a coarser resolution is sufficient for the task at hand.

### Geometric intersections

\index{raster!intersection} 
In Section \@ref(spatial-raster-subsetting) we have shown how to extract values from a raster overlaid by other spatial objects.
To retrieve a spatial output, we can use almost the same subsetting syntax.
The only difference is that we have to make clear that we would like to keep the matrix structure by setting the `drop`-parameter to `FALSE`.
This will return a raster object containing the cells whose midpoints overlap with `clip`.


```r
data("elev", package = "spData")
clip = raster(xmn = 0.9, xmx = 1.8, ymn = -0.45, ymx = 0.45,
              res = 0.3, vals = rep(1, 9))
elev[clip, drop = FALSE]
#> class      : RasterLayer 
#> dimensions : 2, 1, 2  (nrow, ncol, ncell)
#> resolution : 0.5, 0.5  (x, y)
#> extent     : 1, 1.5, -0.5, 0.5  (xmin, xmax, ymin, ymax)
#> crs        : +proj=longlat +datum=WGS84 +ellps=WGS84 +towgs84=0,0,0 
#> source     : memory
#> names      : layer 
#> values     : 18, 24  (min, max)
```

For the same operation we can also use the `intersect()` and `crop()` command.

### Extent and origin

\index{raster!merging} 
When merging or performing map algebra on rasters, their resolution, projection, origin and/or extent have to match. Otherwise, how should we add the values of one raster with a resolution of 0.2 decimal degrees to a second raster with a resolution of 1 decimal degree?
The same problem arises when we would like to merge satellite imagery from different sensors with different projections and resolutions. 
We can deal with such mismatches by aligning the rasters.

In the simplest case, two images only differ with regard to their extent.
<!--The `projectRaster()` function reprojects one raster to a desired projection, say from UTM to WGS84.-->
Following code adds one row and two columns to each side of the raster while setting all new values to an elevation of 1000 meters (Figure \@ref(fig:extend-example)).


```r
data(elev, package = "spData")
elev_2 = extend(elev, c(1, 2), value = 1000)
plot(elev_2)
```

<div class="figure" style="text-align: center">
<img src="05-geometry-operations_files/figure-html/extend-example-1.png" alt="Original raster extended by one row on each side (top, bottom) and two columns on each side (right, left)." width="100%" />
<p class="caption">(\#fig:extend-example)Original raster extended by one row on each side (top, bottom) and two columns on each side (right, left).</p>
</div>

Performing an algebraic operation on two objects with differing extents in R, the **raster** package returns the result for the intersection\index{raster!intersection}, and says so in a warning.


```r
elev_3 = elev + elev_2
#> Warning in elev + elev_2: Raster objects have different extents. Result for
#> their intersection is returned
```

However, we can also align the extent of two rasters with `extend()`. 
Instead of telling the function how many rows or columns should be added (as done before), we allow it to figure it out by using another raster object.
Here, we extend the `elev` object to the extent of `elev_2`. 
The newly added rows and column receive the default value of the `value` parameter, i.e., `NA`.


```r
elev_4 = extend(elev, elev_2)
```

The origin of a raster is the cell corner closest to the coordinates (0, 0).
The `origin()` function returns the coordinates of the origin.
In the below example a cell corner exists with coordinates (0, 0), but that is not necessarily the case.


```r
origin(elev_4)
#> [1] 0 0
```

If two rasters have different origins, their cells do not overlap completely which would make map algebra impossible.
To change the origin , use `origin()`.^[
If the origins of two raster datasets are just marginally apart, it sometimes is sufficient to simply increase the `tolerance` argument  of `raster::rasterOptions()`.
]
Looking at Figure \@ref(fig:origin-example) reveals the effect of changing the origin.


```r
# change the origin
origin(elev_4) = c(0.25, 0.25)
plot(elev_4)
# and add the original raster
plot(elev, add = TRUE)
```

<div class="figure" style="text-align: center">
<img src="05-geometry-operations_files/figure-html/origin-example-1.png" alt="Rasters with identical values but different origins." width="100%" />
<p class="caption">(\#fig:origin-example)Rasters with identical values but different origins.</p>
</div>

Note that changing the resolution frequently (next section) also changes the origin.

### Aggregation and disaggregation

\index{raster!aggregation} 
\index{raster!disaggregation} 
\index{raster!resampling}
Raster datasets can also differ with regard to their resolution. 
To match resolutions, one can either decrease  (`aggregate()`) or increase (`disaggregate()`) the resolution of one raster.^[
Here we refer to spatial resolution.
In remote sensing the spectral (spectral bands), temporal (observations through time of the same area) and radiometric (color depth) resolution are also important.
Check out the `stackApply()` example in the documentation for getting an idea on how to do temporal raster aggregation.
]
As an example, we here change the spatial resolution of `dem` (found in the **RQGIS** package) by a factor of 5 (Figure \@ref(fig:aggregate-example)).
Additionally, the output cell value should correspond to the mean of the input cells (note that one could use other functions as well, such as `median()`, `sum()`, etc.):


```r
data("dem", package = "spDataLarge")
dem_agg = aggregate(dem, fact = 5, fun = mean)
```

<div class="figure" style="text-align: center">
<img src="05-geometry-operations_files/figure-html/aggregate-example-1.png" alt="Original raster (left). Aggregated raster (right)." width="100%" />
<p class="caption">(\#fig:aggregate-example)Original raster (left). Aggregated raster (right).</p>
</div>

By contrast, the`disaggregate()` function increases the resolution.
However, we have to specify a method on how to fill the new cells.
The `disaggregate()` function provides two methods.
The first (`method = ""`) simply gives all output cells the value of the input cell, and hence duplicates values which leads to a blocky output image.

The `bilinear` method, in turn, is an interpolation technique that uses the four nearest pixel centers of the input image (salmon colored points in Figure \@ref(fig:bilinear)) to compute an average weighted by distance (arrows in Figure \@ref(fig:bilinear) as the value of the output cell - square in the upper left corner in Figure \@ref(fig:bilinear)).


```r
dem_disagg = disaggregate(dem_agg, fact = 5, method = "bilinear")
identical(dem, dem_disagg)
#> [1] FALSE
```

<div class="figure" style="text-align: center">
<img src="05-geometry-operations_files/figure-html/bilinear-1.png" alt="Bilinear disaggregation in action." width="100%" />
<p class="caption">(\#fig:bilinear)Bilinear disaggregation in action.</p>
</div>

Comparing the values of `dem` and `dem_disagg` tells us that they are not identical (you can also use `compareRaster()` or `all.equal()`).
However, this was hardly to be expected, since disaggregating is a simple interpolation technique.
It is important to keep in mind that disaggregating results in a finer resolution; the corresponding values, however, are only as accurate as their lower resolution source.

The process of computing values for new pixel locations is also called resampling. 
In fact, the **raster** package provides a `resample()` function.
It lets you align several raster properties in one go, namely origin, extent and resolution.
By default, it uses the `bilinear`-interpolation.


```r
# add 2 rows and columns, i.e. change the extent
dem_agg = extend(dem_agg, 2)
dem_disagg_2 = resample(dem_agg, dem)
```

Finally, in order to align many (possibly hundreds or thousands of) images stored on disk, you could use the `gdalUtils::align_rasters()` function.
However, you may also use **raster** with very large datasets. 
This is because **raster**:

1. Lets you work with raster datasets that are too large to fit into the main memory (RAM) by only processing chunks of it.
2. Tries to facilitate parallel processing.
For more information, see the help pages of `beginCluster()` and `clusteR()`.
Additionally, check out the *Multi-core functions* section in `vignette("functions", package = "raster")`.

## Raster-vector interactions {#raster-vector}

\index{raster-vector!interactions} 
This section focuses on interactions between raster and vector geographic data models, introduced in Chapter \@ref(spatial-class).
It includes four main techniques:
raster cropping and masking using vector objects (Section \@ref(raster-cropping));
extracting raster values using different types of vector data (Section \@ref(raster-extraction));
and raster-vector conversion (Sections \@ref(rasterization) and \@ref(spatial-vectorization)).
<!-- operations are not symmetrical, for example: -->
<!-- - raster clipping - no vector counterpart -->
<!-- - raster extraction is connected to some methods used in vectorization and rasterization -->
<!-- - etc. -->
The above concepts are demonstrated using data used in previous chapters to understand their potential real-world applications.

### Raster cropping

\index{raster-vector!raster cropping} 
Many geographic data projects involve integrating data from many different sources, such as remote sensing images (rasters) and administrative boundaries (vectors).
Often the extent of input raster datasets is larger than the area of interest.
In this case raster **cropping** and **masking** are useful for unifying the spatial extent of input data.
Both operations reduce object memory use and associated computational resources for subsequent analysis steps, and may be a necessary preprocessing step before creating attractive maps involving raster data.

We will use two objects to illustrate raster cropping:

- A `raster` object `srtm` representing elevation (meters above sea level) in south-western Utah.
- A vector (`sf`) object `zion` representing Zion National Park.

Both target and cropping objects must have the same projection.
The following code chunk therefore not only loads the datasets, from the **spDataLarge** package installed in Chapter \@ref(spatial-class),
it also reprojects `zion` (see Section \@ref(reproj-geo-data) for more on reprojection):


```r
srtm = raster(system.file("raster/srtm.tif", package = "spDataLarge"))
zion = st_read(system.file("vector/zion.gpkg", package = "spDataLarge"))
zion = st_transform(zion, projection(srtm))
```

We will use `crop()` from the **raster** package to crop the `srtm` raster.
`crop()` reduces the rectangular extent of the object passed to its first argument based on the extent of the object passed to its second argument, as demonstrated in the command below (which generates Figure \@ref(fig:cropmask)(B) --- note the smaller extent of the raster background):


```r
srtm_cropped = crop(srtm, zion)
```

\index{raster-vector!raster masking} 
Related to `crop()` is the **raster** function `mask()`, which sets values outside of the bounds of the object passed to its second argument to `NA`.
The following command therefore masks every cell outside of the Zion National Park boundaries (Figure \@ref(fig:cropmask)(C)):


```r
srtm_masked = mask(srtm, zion)
```

Changing the settings of `mask()` yields different results.
Setting `updatevalue = 0`, for example, will set all pixels outside the national park to 0.
Setting `inverse = TRUE` will mask everything *inside* the bounds of the park (see `?mask` for details) (Figure \@ref(fig:cropmask)(D)).


```r
srtm_inv_masked = mask(srtm, zion, inverse = TRUE)
```

<div class="figure" style="text-align: center">
<img src="05-geometry-operations_files/figure-html/cropmask-1.png" alt="Illustration of raster cropping and raster masking." width="100%" />
<p class="caption">(\#fig:cropmask)Illustration of raster cropping and raster masking.</p>
</div>

### Raster extraction

\index{raster-vector!raster extraction} 
Raster extraction is the process of identifying and returning the values associated with a 'target' raster at specific locations, based on a (typically vector) geographic 'selector' object.
The results depend on the type of selector used (points, lines or polygons) and arguments passed to the `raster::extract()` function, which we use to demonstrate raster extraction.
The reverse of raster extraction --- assigning raster cell values based on vector objects --- is rasterization, described in Section \@ref(rasterization).

The simplest example is extracting the value of a raster cell at specific **points**.
For this purpose, we will use `zion_points`, which contain a sample of 30 locations within the Zion National Park (Figure \@ref(fig:pointextr)). 
<!-- They could represent places where soils properties were measured and we want to know what is the elevation of each point. -->
The following command extracts elevation values from `srtm` and assigns the resulting vector to a new column (`elevation`) in the `zion_points` dataset: 


```r
data("zion_points", package = "spDataLarge")
zion_points$elevation = raster::extract(srtm, zion_points)
```



The `buffer` argument can be used to specify a buffer radius (in meters) around each point.
The result of `raster::extract(srtm, zion_points, buffer = 1000)`, for example, is a list of vectors, each of which representing the values of cells inside the buffer associated with each point.
In practice, this example is a special case of extraction with a polygon selector, described below.

<div class="figure" style="text-align: center">
<img src="05-geometry-operations_files/figure-html/pointextr-1.png" alt="Locations of points used for raster extraction." width="60%" />
<p class="caption">(\#fig:pointextr)Locations of points used for raster extraction.</p>
</div>

Raster extraction also works with **line** selectors.
To demonstrate this, the code below creates `zion_transect`, a straight line going from northwest to southeast of the Zion National Park, illustrated in Figure \@ref(fig:lineextr)(A) (see Section \@ref(vector-data) for a recap on the vector data model):


```r
zion_transect = cbind(c(-113.2, -112.9), c(37.45, 37.2)) %>%
  st_linestring() %>% 
  st_sfc(crs = projection(srtm)) %>% 
  st_sf()
```



The utility of extracting heights from a linear selector is illustrated by imagining that you are planning a hike.
The method demonstrated below provides an 'elevation profile' of the route (the line does not need to be straight), useful for estimating how long it will take due to long climbs:


```r
transect = raster::extract(srtm, zion_transect, 
                           along = TRUE, cellnumbers = TRUE)
```

Note the use of `along = TRUE` and `cellnumbers = TRUE` arguments to return cell IDs *along* the path. 
The result is a list containing a matrix of cell IDs in the first column and elevation values in the second.
The number of list elements is equal to the number of lines or polygons from which we are extracting values.
The subsequent code chunk first converts this tricky matrix-in-a-list object into a simple data frame, returns the coordinates associated with each extracted cell, and finds the associated distances along the transect (see `?geosphere::distGeo()` for details):


```r
transect_df = purrr::map_dfr(transect, as_data_frame, .id = "ID")
#> Warning: `as_data_frame()` is deprecated as of tibble 2.0.0.
#> Please use `as_tibble()` instead.
#> The signature and semantics have changed, see `?as_tibble`.
#> This warning is displayed once every 8 hours.
#> Call `lifecycle::last_warnings()` to see where this warning was generated.
transect_coords = xyFromCell(srtm, transect_df$cell)
pair_dist = geosphere::distGeo(transect_coords)[-nrow(transect_coords)]
transect_df$dist = c(0, cumsum(pair_dist)) 
```

The resulting `transect_df` can be used to create elevation profiles, as illustrated in Figure \@ref(fig:lineextr)(B).

<div class="figure" style="text-align: center">
<img src="05-geometry-operations_files/figure-html/lineextr-1.png" alt="Location of a line used for raster extraction (left) and the elevation along this line (right)." width="100%" />
<p class="caption">(\#fig:lineextr)Location of a line used for raster extraction (left) and the elevation along this line (right).</p>
</div>

The final type of geographic vector object for raster extraction is **polygons**.
Like lines and buffers, polygons tend to return many raster values per polygon.
This is demonstrated in the command below, which results in a data frame with column names  `ID` (the row number of the polygon) and `srtm` (associated elevation values):




```r
zion_srtm_values = raster::extract(x = srtm, y = zion, df = TRUE) 
```

Such results can be used to generate summary statistics for raster values per polygon, for example to characterize a single region or to compare many regions.
The generation of summary statistics is demonstrated in the code below, which creates the object `zion_srtm_df` containing summary statistics for elevation values in Zion National Park (see Figure \@ref(fig:polyextr)(A)):


```r
group_by(zion_srtm_values, ID) %>% 
  summarize_at(vars(srtm), list(~min(.), ~mean(.), ~max(.)))
#> # A tibble: 1 x 4
#>      ID   min  mean   max
#>   <dbl> <dbl> <dbl> <dbl>
#> 1     1  1122 1818.  2661
```

The preceding code chunk used the **tidyverse**\index{tidyverse (package)} to provide summary statistics for cell values per polygon ID, as described in Chapter \@ref(attr).
The results provide useful summaries, for example that the maximum height in the park is around 2,661 meters (other summary statistics, such as standard deviation, can also be calculated in this way).
Because there is only one polygon in the example a data frame with a single row is returned; however, the method works when multiple selector polygons are used.

The same approach works for counting occurrences of categorical raster values within polygons.
This is illustrated with a land cover dataset (`nlcd`) from the **spDataLarge** package in Figure \@ref(fig:polyextr)(B), and demonstrated in the code below:


```r
zion_nlcd = raster::extract(nlcd, zion, df = TRUE, factors = TRUE) 
dplyr::select(zion_nlcd, ID, levels) %>% 
  tidyr::gather(key, value, -ID) %>%
  group_by(ID, key, value) %>%
  tally() %>% 
  tidyr::spread(value, n, fill = 0)
#> # A tibble: 1 x 9
#> # Groups:   ID, key [1]
#>      ID key    Barren Cultivated Developed Forest Herbaceous Shrubland Wetlands
#>   <dbl> <chr>   <dbl>      <dbl>     <dbl>  <dbl>      <dbl>     <dbl>    <dbl>
#> 1     1 levels  98285         62      4205 298299        235    203701      679
```

<div class="figure" style="text-align: center">
<img src="05-geometry-operations_files/figure-html/polyextr-1.png" alt="Area used for continuous (left) and categorical (right) raster extraction." width="100%" />
<p class="caption">(\#fig:polyextr)Area used for continuous (left) and categorical (right) raster extraction.</p>
</div>

So far, we have seen how `raster::extract()` is a flexible way of extracting raster cell values from a range of input geographic objects.
An issue with the function, however, is that it is relatively slow.
If this is a problem, it is useful to know about alternatives and work-arounds, three of which are presented below.

- **Parallelization**: this approach works when using many geographic vector selector objects by splitting them into groups and extracting cell values independently for each group (see `?raster::clusterR()` for details of this approach)
<!-- tabularaster (ref to the vignette - https://cran.r-project.org/web/packages/tabularaster/vignettes/tabularaster-usage.html)-->
- Use the **velox** package [@hunziker_velox:_2017], which provides a fast method for extracting raster data that fits in memory (see the package's [`extract`](https://hunzikp.github.io/velox/extract.html) vignette for details)
- Using **R-GIS bridges** (see Chapter \@ref(gis)): efficient calculation of raster statistics from polygons can be found in the SAGA function `saga:gridstatisticsforpolygons`, for example, which can be accessed via **RQGIS**
<!-- Methods similar to `raster::extract` can be found in GRASS GIS (e.g. v.rast.stats) -->
<!-- https://grass.osgeo.org/grass74/manuals/v.rast.stats.html - test -->
<!-- https://twitter.com/mdsumner/status/976978499402571776 -->
<!-- https://gist.github.com/mdsumner/d0b26238321a5d2c2c2ba663ff684183 -->

### Rasterization {#rasterization}

\index{raster-vector!rasterization} 
Rasterization is the conversion of vector objects into their representation in raster objects.
Usually, the output raster is used for quantitative analysis (e.g., analysis of terrain) or modeling.
As we saw in Chapter \@ref(spatial-class) the raster data model has some characteristics that make it conducive to certain methods.
Furthermore, the process of rasterization can help simplify datasets because the resulting values all have the same spatial resolution: rasterization can be seen as a special type of geographic data aggregation.

The **raster** package contains the function `rasterize()` for doing this work.
Its first two arguments are, `x`, vector object to be rasterized and, `y`, a 'template raster' object defining the extent, resolution and CRS of the output.
The geographic resolution of the input raster has a major impact on the results: if it is too low (cell size is too large), the result may miss the full geographic variability of the vector data; if it is too high, computational times may be excessive.
There are no simple rules to follow when deciding an appropriate geographic resolution, which is heavily dependent on the intended use of the results.
Often the target resolution is imposed on the user, for example when the output of rasterization needs to be aligned to the existing raster.

To demonstrate rasterization in action, we will use a template raster that has the same extent and CRS as the input vector data `cycle_hire_osm_projected` (a dataset on cycle hire points in London is illustrated in Figure \@ref(fig:vector-rasterization1)(A)) and spatial resolution of 1000 meters:


```r
cycle_hire_osm_projected = st_transform(cycle_hire_osm, 27700)
raster_template = raster(extent(cycle_hire_osm_projected), resolution = 1000,
                         crs = st_crs(cycle_hire_osm_projected)$proj4string)
#> Warning in showSRID(uprojargs, format = "PROJ", multiline = "NO", prefer_proj
#> = prefer_proj): Discarded datum Unknown based on Airy 1830 ellipsoid in CRS
#> definition
```

Rasterization is a very flexible operation: the results depend not only on the nature of the template raster, but also on the type of input vector (e.g., points, polygons) and a variety of arguments taken by the `rasterize()` function.

To illustrate this flexibility we will try three different approaches to rasterization.
First, we create a raster representing the presence or absence of cycle hire points (known as presence/absence rasters).
In this case `rasterize()` requires only one argument in addition to `x` and `y` (the aforementioned vector and raster objects): a value to be transferred to all non-empty cells specified by `field` (results illustrated Figure \@ref(fig:vector-rasterization1)(B)).


```r
ch_raster1 = rasterize(cycle_hire_osm_projected, raster_template, field = 1)
#> Warning in showSRID(uprojargs, format = "PROJ", multiline = "NO", prefer_proj
#> = prefer_proj): Discarded datum Unknown based on Airy 1830 ellipsoid in CRS
#> definition
#> Warning in showSRID(SRS_string, format = "PROJ", multiline = "NO", prefer_proj =
#> prefer_proj): Discarded datum OSGB 1936 in CRS definition
```

The `fun` argument specifies summary statistics used to convert multiple observations in close proximity into associate cells in the raster object.
By default `fun = "last"` is used but other options such as `fun = "count"` can be used,  in this case to count the number of cycle hire points in each grid cell (the results of this operation are illustrated in Figure \@ref(fig:vector-rasterization1)(C)).


```r
ch_raster2 = rasterize(cycle_hire_osm_projected, raster_template, 
                       field = 1, fun = "count")
#> Warning in showSRID(uprojargs, format = "PROJ", multiline = "NO", prefer_proj
#> = prefer_proj): Discarded datum Unknown based on Airy 1830 ellipsoid in CRS
#> definition
#> Warning in showSRID(SRS_string, format = "PROJ", multiline = "NO", prefer_proj =
#> prefer_proj): Discarded datum OSGB 1936 in CRS definition
```

The new output, `ch_raster2`, shows the number of cycle hire points in each grid cell.
The cycle hire locations have different numbers of bicycles described by the `capacity` variable, raising the question, what's the capacity in each grid cell?
To calculate that we must `sum` the field (`"capacity"`), resulting in output illustrated in Figure \@ref(fig:vector-rasterization1)(D), calculated with the following command (other summary functions such as `mean` could be used):


```r
ch_raster3 = rasterize(cycle_hire_osm_projected, raster_template, 
                       field = "capacity", fun = sum)
#> Warning in showSRID(uprojargs, format = "PROJ", multiline = "NO", prefer_proj
#> = prefer_proj): Discarded datum Unknown based on Airy 1830 ellipsoid in CRS
#> definition
#> Warning in showSRID(SRS_string, format = "PROJ", multiline = "NO", prefer_proj =
#> prefer_proj): Discarded datum OSGB 1936 in CRS definition
```

<div class="figure" style="text-align: center">
<img src="05-geometry-operations_files/figure-html/vector-rasterization1-1.png" alt="Examples of point rasterization." width="100%" />
<p class="caption">(\#fig:vector-rasterization1)Examples of point rasterization.</p>
</div>

Another dataset based on California's polygons and borders (created below) illustrates rasterization of lines.
After casting the polygon objects into a multilinestring, a template raster is created with a resolution of a 0.5 degree:


```r
california = dplyr::filter(us_states, NAME == "California")
california_borders = st_cast(california, "MULTILINESTRING")
raster_template2 = raster(extent(california), resolution = 0.5,
                         crs = st_crs(california)$proj4string)
#> Warning in showSRID(uprojargs, format = "PROJ", multiline = "NO", prefer_proj =
#> prefer_proj): Discarded datum Unknown based on GRS80 ellipsoid in CRS definition
```

Line rasterization is demonstrated in the code below.
In the resulting raster, all cells that are touched by a line get a value, as illustrated in Figure \@ref(fig:vector-rasterization2)(A).


```r
california_raster1 = rasterize(california_borders, raster_template2) 
#> Warning in showSRID(uprojargs, format = "PROJ", multiline = "NO", prefer_proj =
#> prefer_proj): Discarded datum Unknown based on GRS80 ellipsoid in CRS definition
```

Polygon rasterization, by contrast, selects only cells whose centroids are inside the selector polygon, as illustrated in Figure \@ref(fig:vector-rasterization2)(B).


```r
california_raster2 = rasterize(california, raster_template2) 
#> Warning in showSRID(uprojargs, format = "PROJ", multiline = "NO", prefer_proj =
#> prefer_proj): Discarded datum Unknown based on GRS80 ellipsoid in CRS definition
```

<!-- getCover? -->
<!-- the fraction of each grid cell that is covered by the polygons-->
<!-- ```{r, echo=FALSE, eval=FALSE} -->
<!-- california_raster3 = rasterize(california, raster_template2, getCover = TRUE) -->
<!-- r3po = tm_shape(california_raster3) + -->
<!--   tm_raster(legend.show = TRUE, title = "Values: ", style = "fixed", breaks = c(0, 1, 25, 50, 75, 100)) + -->
<!--   tm_shape(california) + -->
<!--   tm_borders() + -->
<!--   tm_layout(outer.margins = rep(0.01, 4), -->
<!--             inner.margins = rep(0, 4)) -->
<!-- ``` -->

<!-- It is also possible to use the `field` or `fun` arguments for lines and polygons rasterizations. -->

<div class="figure" style="text-align: center">
<img src="05-geometry-operations_files/figure-html/vector-rasterization2-1.png" alt="Examples of line and polygon rasterizations." width="100%" />
<p class="caption">(\#fig:vector-rasterization2)Examples of line and polygon rasterizations.</p>
</div>

As with `raster::extract()`,  `raster::rasterize()` works well for most cases but is not performance optimized. 
Fortunately, there are several alternatives, including the `fasterize::fasterize()` and `gdalUtils::gdal_rasterize()`. 
The former is much (100 times+) faster than `rasterize()`, but is currently limited to polygon rasterization.
The latter is part of GDAL\index{GDAL} and therefore requires a vector file (instead of an `sf` object) and rasterization parameters (instead of a `Raster*` template object) as inputs.^[
See more at http://gdal.org/gdal_rasterize.html.
]

### Spatial vectorization

\index{raster-vector!spatial vectorization} 
Spatial vectorization is the counterpart of rasterization (Section \@ref(rasterization)), but in the opposite direction.
It involves converting spatially continuous raster data into spatially discrete vector data such as points, lines or polygons.

\BeginKnitrBlock{rmdnote}<div class="rmdnote">Be careful with the wording!
In R, vectorization refers to the possibility of replacing `for`-loops and alike by doing things like `1:10 / 2` (see also @wickham_advanced_2014).</div>\EndKnitrBlock{rmdnote}

The simplest form of vectorization is to convert the centroids of raster cells into points.
`rasterToPoints()` does exactly this for all non-`NA` raster grid cells (Figure \@ref(fig:raster-vectorization1)).
Setting the `spatial` parameter to `TRUE` is needed to get a spatial object instead of a matrix.


```r
elev_point = rasterToPoints(elev, spatial = TRUE) %>% 
  st_as_sf()
```

<div class="figure" style="text-align: center">
<img src="05-geometry-operations_files/figure-html/raster-vectorization1-1.png" alt="Raster and point representation of the elev object." width="100%" />
<p class="caption">(\#fig:raster-vectorization1)Raster and point representation of the elev object.</p>
</div>

Another common type of spatial vectorization is the creation of contour lines representing lines of continuous height or temperatures (isotherms) for example.
We will use a real-world digital elevation model (DEM) because the artificial raster `elev` produces parallel lines (task: verify this and explain why this happens).
<!-- because when creating it we made the upper left corner the lowest and the lower right corner the highest value while increasing cell values by one from left to right. -->
Contour lines can be created with the **raster** function `rasterToContour()`, which is itself a wrapper around `contourLines()`, as demonstrated below (not shown):


```r
data(dem, package = "spDataLarge")
cl = rasterToContour(dem)
plot(dem, axes = FALSE)
plot(cl, add = TRUE)
```

Contours can also be added to existing plots with functions such as `contour()`, `rasterVis::contourplot()` or `tmap::tm_iso()`.
As illustrated in Figure \@ref(fig:contour-tmap), isolines can be labelled.


```r
# create hillshade
hs = hillShade(slope = terrain(dem, "slope"), aspect = terrain(dem, "aspect"))
plot(hs, col = gray(0:100 / 100), legend = FALSE)
# overlay with DEM
plot(dem, col = terrain.colors(25), alpha = 0.5, legend = FALSE, add = TRUE)
# add contour lines
contour(dem, col = "white", add = TRUE)
```

\index{hillshade}

<div class="figure" style="text-align: center">
<img src="figures/contour-tmap-1.png" alt="DEM hillshade of the southern flank of Mt. Mongón overlaid by contour lines." width="100%" />
<p class="caption">(\#fig:contour-tmap)DEM hillshade of the southern flank of Mt. Mongón overlaid by contour lines.</p>
</div>

The final type of vectorization involves conversion of rasters to polygons.
This can be done with `raster::rasterToPolygons()`, which converts each raster cell into a polygon consisting of five coordinates, all of which are stored in memory (explaining why rasters are often fast compared with vectors!).

This is illustrated below by converting the `grain` object into polygons and subsequently dissolving borders between polygons with the same attribute values (also see the `dissolve` argument in `rasterToPolygons()`).
Attributes in this case are stored in a column called `layer` (see Section \@ref(geometry-unions) and Figure \@ref(fig:raster-vectorization2)).
(Note: a convenient alternative for converting rasters into polygons is `spex::polygonize()` which by default returns an `sf` object.)


```r
grain_poly = rasterToPolygons(grain) %>% 
  st_as_sf()
grain_poly2 = grain_poly %>% 
  group_by(layer) %>%
  summarize()
#> `summarise()` ungrouping output (override with `.groups` argument)
#> although coordinates are longitude/latitude, st_union assumes that they are planar
#> although coordinates are longitude/latitude, st_union assumes that they are planar
#> although coordinates are longitude/latitude, st_union assumes that they are planar
```

<div class="figure" style="text-align: center">
<img src="05-geometry-operations_files/figure-html/raster-vectorization2-1.png" alt="Illustration of vectorization of raster (left) into polygon (center) and polygon aggregation (right)." width="100%" />
<p class="caption">(\#fig:raster-vectorization2)Illustration of vectorization of raster (left) into polygon (center) and polygon aggregation (right).</p>
</div>

<!-- ## distances? -->

## Exercises

Some of the exercises use a vector (`random_points`) and raster dataset (`ndvi`) from the **RQGIS** package.
They also use a polygonal 'convex hull' derived from the vector dataset (`ch`) to represent the area of interest:

```r
library(RQGIS)
data(random_points)
data(ndvi)
ch = st_combine(random_points) %>% 
  st_convex_hull()
```
1. Generate and plot simplified versions of the `nz` dataset.
Experiment with different values of `keep` (ranging from 0.5 to 0.00005) for `ms_simplify()` and `dTolerance` (from 100 to 100,000) `st_simplify()` .
    - At what value does the form of the result start to break down for each method, making New Zealand unrecognizable?
    - Advanced: What is different about the geometry type of the results from `st_simplify()` compared with the geometry type of `ms_simplify()`? What problems does this create and how can this be resolved?

1. In the first exercise in Chapter \@ref(spatial-operations) it was established that Canterbury region had 70 of the 101 highest points in New Zealand. 
Using `st_buffer()`, how many points in `nz_height` are within 100 km of Canterbury?

1. Find the geographic centroid of New Zealand. 
How far is it from the geographic centroid of Canterbury?

1. Most world maps have a north-up orientation.
A world map with a south-up orientation could be created by a reflection (one of the affine transformations not mentioned in Section \@ref(affine-transformations)) of the `world` object's geometry.
Write code to do so.
Hint: you need to use a two-element vector for this transformation.
    - Bonus: create an upside-down map of your country.

1. Subset the point in `p` that is contained within `x` *and* `y` (see Section \@ref(clipping) and Figure \@ref(fig:venn-clip)).
    - Using base subsetting operators.
    - Using an intermediary object created with `st_intersection()`\index{vector!intersection}.

1. Calculate the length of the boundary lines of US states in meters.
Which state has the longest border and which has the shortest?
Hint: The `st_length` function computes the length of a `LINESTRING` or `MULTILINESTRING` geometry.

1. Crop the `ndvi` raster using (1) the `random_points` dataset and (2) the `ch` dataset.
Are there any differences in the output maps?
Next, mask `ndvi` using these two datasets.
Can you see any difference now?
How can you explain that?

1. Firstly, extract values from `ndvi` at the points represented in `random_points`.
Next, extract average values of `ndvi` using a 90 buffer around each point from `random_points` and compare these two sets of values. 
When would extracting values by buffers be more suitable than by points alone?

1. Subset points higher than 3100 meters in New Zealand (the `nz_height` object) and create a template raster with a resolution of 3 km. 
Using these objects:
    - Count numbers of the highest points in each grid cell.
    - Find the maximum elevation in each grid cell.

1. Aggregate the raster counting high points in New Zealand (created in the previous exercise), reduce its geographic resolution by half (so cells are 6 by 6 km) and plot the result.
    - Resample the lower resolution raster back to a resolution of 3 km. How have the results changed?
    - Name two advantages and disadvantages of reducing raster resolution.

1. Polygonize the `grain` dataset and filter all squares representing clay.
    - Name two advantages and disadvantages of vector data over raster data.
    -  At which points would it be useful to convert rasters to vectors in your work?


