---
title: "Raster Calculations in R"
teaching: 10
exercises: 0
questions:
- "How to subtract one raster from another and extract pixel values for defined locations."
objectives:
- "Be able to to perform a subtraction (difference) between two rasters using raster math."
- "Know how to perform a more efficient subtraction (difference) between two rasters using the raster `overlay()` function in R."
keypoints:
- ""
authors: [Leah A. Wasser, Megan A. Jones, Zack Brym, Kristina Riemer, Jason Williams, Jeff Hollister,  Mike Smorul, Joseph Stachelek]
contributors: [ ]
packagesLibraries: [raster, rgdal]
dateCreated:  2015-10-23
lastModified: 2017-09-19
categories:  [self-paced-tutorial]
tags: [R, raster, spatial-data-gis]
tutorialSeries: [raster-data-series]
mainTag: raster-data-series
description: "This tutorial covers how to subtract one raster from another using
efficient methods - the overlay function compared to basic subtraction. We also
cover how to extract pixel values from a set of locations - for example a buffer
region around plot locations at a field site. Finally, it explains the basic
principles of writing functions in R."
image:
  feature: NEONCarpentryHeader_2.png
  credit: A collaboration between the National Ecological Observatory Network (NEON) and Data Carpentry
  creditlink:
comments: true
---



> ## Things You’ll Need To Complete This Tutorial
>
> **R Skill Level:** Intermediate - you've got the basics of `R` down.
>
> You will need the most current version of `R` and, preferably, `RStudio` loaded
on your computer to complete this tutorial.
>
> ### Install R Packages
>
> * **raster:** `install.packages("raster")`
> * **rgdal:** `install.packages("rgdal")`
>
> * [More on Packages in R - Adapted from Software Carpentry.]({{site.baseurl}}/R/Packages-In-R/)
>
> #### Data to Download
>
> ### Additional Resources
> * <a href="http://cran.r-project.org/web/packages/raster/raster.pdf"
> target="_blank">Read more about the `raster` package in `R`.</a>
{: .prereq}

We often want to combine values of and perform calculations on rasters to create
a new output raster. This tutorial covers how to subtract one raster from
another using basic raster math and the `overlay()` function. It also covers how
to extract pixel values from a set of locations - for example a buffer region
around plot locations at a field site.

## Raster Calculations in R
We often want to perform calculations on two or more rasters to create a new
output raster. For example, if we are interested in mapping the heights of trees
across an entire field site, we might want to calculate the *difference* between
the Digital Surface Model (DSM, tops of trees) and the
Digital Terrain Model (DTM, ground level). The resulting dataset is referred to
as a Canopy Height Model (CHM) and represents the actual height of trees,
buildings, etc. with the influence of ground elevation removed.

<figure>
    <a href="{{ site.baseurl }}/images/dc-spatial-raster/lidarTree-height.png">
    <img src="{{ site.baseurl }}/images/dc-spatial-raster/lidarTree-height.png">
    </a>
    <figcaption> Source: National Ecological Observatory Network (NEON)
    </figcaption>
</figure>

* Check out more on LiDAR CHM, DTM and DSM in this NEON Data Skills overview tutorial:
<a href="http://neondataskills.org/self-paced-tutorial/2_LiDAR-Data-Concepts_Activity2/" target="_blank">
What is a CHM, DSM and DTM? About Gridded, Raster LiDAR Data</a>.

### Load the Data
We will need the `raster` package to import and perform raster calculations. We
will use the DTM (`NEON-DS-Airborne-Remote-Sensing/HARV/DTM/HARV_dtmCrop.tif`)
and DSM (`NEON-DS-Airborne-Remote-Sensing/HARV/DSM/HARV_dsmCrop.tif`) from the
NEON Harvard Forest Field site.


~~~
# load raster package
library(raster)
~~~
{: .language-r}



~~~
Loading required package: sp
~~~
{: .output}



~~~
library(rgdal)
~~~
{: .language-r}



~~~
rgdal: version: 1.2-8, (SVN revision 663)
 Geospatial Data Abstraction Library extensions to R successfully loaded
 Loaded GDAL runtime: GDAL 2.2.1, released 2017/06/23
 Path to GDAL shared files: /usr/share/gdal/2.2
 Loaded PROJ.4 runtime: Rel. 4.9.2, 08 September 2015, [PJ_VERSION: 492]
 Path to PROJ.4 shared files: (autodetected)
 Linking to sp version: 1.2-5 
~~~
{: .output}



~~~
# view info about the dtm & dsm raster data that we will work with.
GDALinfo("data/NEON-DS-Airborne-Remote-Sensing/HARV/DTM/HARV_dtmCrop.tif")
~~~
{: .language-r}



~~~
rows        1367 
columns     1697 
bands       1 
lower left origin.x        731453 
lower left origin.y        4712471 
res.x       1 
res.y       1 
ysign       -1 
oblique.x   0 
oblique.y   0 
driver      GTiff 
projection  +proj=utm +zone=18 +datum=WGS84 +units=m +no_defs 
file        data/NEON-DS-Airborne-Remote-Sensing/HARV/DTM/HARV_dtmCrop.tif 
apparent band summary:
   GDType hasNoDataValue NoDataValue blockSize1 blockSize2
1 Float64           TRUE       -9999          1       1697
apparent band statistics:
    Bmin   Bmax    Bmean      Bsd
1 304.56 389.82 344.8979 15.86147
Metadata:
AREA_OR_POINT=Area 
~~~
{: .output}



~~~
GDALinfo("data/NEON-DS-Airborne-Remote-Sensing/HARV/DSM/HARV_dsmCrop.tif")
~~~
{: .language-r}



~~~
rows        1367 
columns     1697 
bands       1 
lower left origin.x        731453 
lower left origin.y        4712471 
res.x       1 
res.y       1 
ysign       -1 
oblique.x   0 
oblique.y   0 
driver      GTiff 
projection  +proj=utm +zone=18 +datum=WGS84 +units=m +no_defs 
file        data/NEON-DS-Airborne-Remote-Sensing/HARV/DSM/HARV_dsmCrop.tif 
apparent band summary:
   GDType hasNoDataValue NoDataValue blockSize1 blockSize2
1 Float64           TRUE       -9999          1       1697
apparent band statistics:
    Bmin   Bmax    Bmean      Bsd
1 305.07 416.07 359.8531 17.83169
Metadata:
AREA_OR_POINT=Area 
~~~
{: .output}

As seen from the `geoTiff` tags, both rasters have:

* the same CRS,
* the same resolution
* defined minimum and maximum values.

Let's load the data.


~~~
# load the DTM & DSM rasters
DTM_HARV <- raster("data/NEON-DS-Airborne-Remote-Sensing/HARV/DTM/HARV_dtmCrop.tif")
DSM_HARV <- raster("data/NEON-DS-Airborne-Remote-Sensing/HARV/DSM/HARV_dsmCrop.tif")

# create a quick plot of each to see what we're dealing with
plot(DTM_HARV,
     main = "Digital Terrain Model \n NEON Harvard Forest Field Site")
~~~
{: .language-r}

<img src="../fig/rmd-load-plot-data-1.png" title="plot of chunk load-plot-data" alt="plot of chunk load-plot-data" style="display: block; margin: auto;" />

~~~
plot(DSM_HARV,
     main = "Digital Surface Model \n NEON Harvard Forest Field Site")
~~~
{: .language-r}

<img src="../fig/rmd-load-plot-data-2.png" title="plot of chunk load-plot-data" alt="plot of chunk load-plot-data" style="display: block; margin: auto;" />

## Two Ways to Perform Raster Calculations

We can calculate the difference between two rasters in two different ways:

* by directly subtracting the two rasters in `R` using raster math

or for more efficient processing - particularly if our rasters are large and/or
the calculations we are performing are complex:

* using the `overlay()` function.

## Raster Math & Canopy Height Models
We can perform raster calculations by simply subtracting (or adding,
multiplying, etc) two rasters. In the geospatial world, we call this
"raster math".

Let's subtract the DTM from the DSM to create a Canopy Height Model.


~~~
# Raster math example
CHM_HARV <- DSM_HARV - DTM_HARV

# plot the output CHM
plot(CHM_HARV,
     main = "Canopy Height Model - Raster Math Subtract\n NEON Harvard Forest Field Site",
     axes = FALSE)
~~~
{: .language-r}

<img src="../fig/rmd-raster-math-1.png" title="plot of chunk raster-math" alt="plot of chunk raster-math" style="display: block; margin: auto;" />

Let's have a look at the distribution of values in our newly created
Canopy Height Model (CHM).


~~~
# histogram of CHM_HARV
hist(CHM_HARV,
  col = "springgreen4",
  main = "Histogram of Canopy Height Model\nNEON Harvard Forest Field Site",
  ylab = "Number of Pixels",
  xlab = "Tree Height (m) ")
~~~
{: .language-r}

<img src="../fig/rmd-create-hist-1.png" title="plot of chunk create-hist" alt="plot of chunk create-hist" style="display: block; margin: auto;" />

Notice that the range of values for the output CHM is between 0 and 30 meters.
Does this make sense for trees in Harvard Forest?

> ## Challenge: Explore CHM Raster Values
> 
> It's often a good idea to explore the range of values in a raster dataset just like we might explore a dataset that we collected in the field.
> 
> 1. What is the min and maximum value for the Harvard Forest Canopy Height Model (`CHM_HARV`) that we just created?
> 2. What are two ways you can check this range of data in `CHM_HARV`?
> 3. What is the distribution of all the pixel values in the CHM?
> 4. Plot a histogram with 6 bins instead of the default and change the color of the histogram.
> 5. Plot the `CHM_HARV` raster using breaks that make sense for the data. Include an appropriate color palette for the data, plot title and no axes ticks / labels.
> 
> > ## Answers
> > 
> > 
> > ~~~
> > # 1)
> > minValue(CHM_HARV)
> > ~~~
> > {: .language-r}
> > 
> > 
> > 
> > ~~~
> > [1] 0
> > ~~~
> > {: .output}
> > 
> > 
> > 
> > ~~~
> > maxValue(CHM_HARV)
> > ~~~
> > {: .language-r}
> > 
> > 
> > 
> > ~~~
> > [1] 38.16998
> > ~~~
> > {: .output}
> > 
> > 
> > 
> > ~~~
> > # 2) Looks at histogram, minValue(NAME)/maxValue(NAME), NAME and look at values
> > # slot.
> > # 3
> > hist(CHM_HARV,
> >      col = "springgreen4",
> >      main = "Histogram of Canopy Height Model\nNEON Harvard Forest Field Site",
> >      maxpixels=ncell(CHM_HARV))
> > ~~~
> > {: .language-r}
> > 
> > <img src="../fig/rmd-challenge-code-CHM-HARV-1.png" title="plot of chunk challenge-code-CHM-HARV" alt="plot of chunk challenge-code-CHM-HARV" style="display: block; margin: auto;" />
> > 
> > ~~~
> > # 4
> > hist(CHM_HARV,
> >      col = "lightgreen",
> >      main = "Histogram of Canopy Height Model\nNEON Harvard Forest Field Site",
> >      maxpixels=ncell(CHM_HARV),
> >      breaks=6)
> > ~~~
> > {: .language-r}
> > 
> > <img src="../fig/rmd-challenge-code-CHM-HARV-2.png" title="plot of chunk challenge-code-CHM-HARV" alt="plot of chunk challenge-code-CHM-HARV" style="display: block; margin: auto;" />
> > 
> > ~~~
> > # 5
> > myCol=terrain.colors(4)
> > plot(CHM_HARV,
> >      breaks=c(0, 10, 20, 30),
> >      col=myCol,
> >      axes=F,
> >      main = "Canopy Height Model \nNEON Harvard Forest Field Site")
> > ~~~
> > {: .language-r}
> > 
> > <img src="../fig/rmd-challenge-code-CHM-HARV-3.png" title="plot of chunk challenge-code-CHM-HARV" alt="plot of chunk challenge-code-CHM-HARV" style="display: block; margin: auto;" />
> {: .solution}
{: .challenge}

## Efficient Raster Calculations: Overlay Function
Raster math, like we just did, is an appropriate approach to raster calculations
if:

1. The rasters we are using are small in size.
2. The calculations we are performing are simple.

However, raster math is a less efficient approach as computation becomes more
complex or as file sizes become large.
The `overlay()` function is more efficient when:

1. The rasters we are using are larger in size.
2. The rasters are stored as individual files.
3. The computations performed are complex.

The `overlay()` function takes two or more rasters and applies a function to
them using efficient processing methods. The syntax is

`outputRaster <- overlay(raster1, raster2, fun=functionName)`

> ## Data Tip
> If the rasters are stacked and stored
> as `RasterStack` or `RasterBrick` objects in `R`, then we should use `calc()`.
> `overlay()` will not work on stacked rasters.
{: .callout}

Let's perform the same subtraction calculation that we calculated above using
raster math, using the `overlay()` function.


~~~
CHM_ov_HARV<- overlay(DSM_HARV,
                      DTM_HARV,
                      fun=function(r1, r2){return(r1-r2)})

plot(CHM_ov_HARV,
  main = "Canopy Height Model - Overlay Subtract\n NEON Harvard Forest Field Site")
~~~
{: .language-r}

<img src="../fig/rmd-raster-overlay-1.png" title="plot of chunk raster-overlay" alt="plot of chunk raster-overlay" style="display: block; margin: auto;" />

How do the plots of the CHM created with manual raster math and the `overlay()`
function compare?

> ## Data Tip
> A custom function consists of a defined
> set of commands performed on a input object. Custom functions are particularly
> useful for tasks that need to be repeated over and over in the code. A
> simplified syntax for writing a custom function in R is:
> `functionName <- function(variable1, variable2){WhatYouWantDone, WhatToReturn}`
{: .callout}

## Export a GeoTIFF
Now that we've created a new raster, let's export the data as a `GeoTIFF` using
the `writeRaster()` function.

When we write this raster object to a `GeoTIFF` file we'll name it
`chm_HARV.tiff`. This name allows us to quickly remember both what the data
contains (CHM data) and for where (HARVard Forest). The `writeRaster()` function
by default writes the output file to your working directory unless you specify a
full file path.


~~~
# export CHM object to new GeotIFF
writeRaster(CHM_ov_HARV, "chm_HARV.tiff",
            format="GTiff",  # specify output format - GeoTIFF
            overwrite=TRUE, # CAUTION: if this is true, it will overwrite an
                            # existing file
            NAflag=-9999) # set no data value to -9999
~~~
{: .language-r}

### writeRaster Options
The function arguments that we used above include:

* **format:** specify that the format will be `GTiff` or `GeoTiff`.
* **overwrite:** If TRUE, `R` will overwrite any existing file  with the same
name in the specified directory. USE THIS SETTING WITH CAUTION!
* **NAflag:** set the `geotiff` tag for `NoDataValue` to -9999, the National
Ecological Observatory Network's (NEON) standard `NoDataValue`.


> ## Challenge: Explore the NEON San Joaquin Experimental Range Field Site
> 
> Data are often more interesting and powerful when we compare them across various
> locations. Let's compare some data collected over Harvard Forest to data
> collected in Southern California. The
> <a href="http://www.neonscience.org/science-design/field-sites/san-joaquin-experimental-range" target="_blank" >NEON San Joaquin Experimental Range (SJER) field site </a>
> located in Southern California has a very different ecosystem and climate than
> the
> <a href="http://www.neonscience.org/science-design/field-sites/harvard-forest" target="_blank" >NEON Harvard Forest Field Site</a>
in Massachusetts.
> 
> Import the SJER DSM and DTM raster files and create a Canopy Height Model.
> Then compare the two sites. Be sure to name your `R` objects and outputs
> carefully, as follows: objectType_SJER (e.g. `DSM_SJER`). This will help you
> keep track of data from different sites!
> 
> 1. Import the DSM and DTM from the SJER directory (if not aready imported
in the
> [Plot Raster Data in R]({{ site.baseurl }}/R/Plot-Rasters-In-R/)
tutorial.) Don't forget to examine the data for `CRS`, bad values, etc.
> 2. Create a `CHM` from the two raster layers and check to make sure the data
are what you expect.
> 3. Plot the `CHM` from SJER.
> 4. Export the SJER CHM as a `GeoTIFF`.
> 5. Compare the vegetation structure of the Harvard Forest and San Joaquin
> Experimental Range.
> 
> Hint: plotting SJER and HARV data side-by-side is an effective way to compare
both datasets!
> > ## Answers
> > 
> > ~~~
> > # 1.
> > # load the DTM
> > DTM_SJER <- raster("data/NEON-DS-Airborne-Remote-Sensing/SJER/DTM/SJER_dtmCrop.tif")
> > # load the DSM
> > DSM_SJER <- raster("data/NEON-DS-Airborne-Remote-Sensing/SJER/DSM/SJER_dsmCrop.tif")
> > 
> > # check CRS, units, etc
> > DTM_SJER
> > DSM_SJER
> > 
> > # check values
> > hist(DTM_SJER,
> >      maxpixels=ncell(DTM_SJER),
> >      main = "Digital Terrain Model - Histogram\n NEON SJER Field Site",
> >      col = "slategrey",
> >      ylab = "Number of Pixels",
> >      xlab = "Elevation (m)")
> > ~~~
> > {: .language-r}
> > 
> > <img src="../fig/rmd-challenge-code-SJER-CHM-1.png" title="plot of chunk challenge-code-SJER-CHM" alt="plot of chunk challenge-code-SJER-CHM" style="display: block; margin: auto;" />
> > 
> > ~~~
> > hist(DSM_SJER,
> >      maxpixels=ncell(DSM_SJER),
> >      main = "Digital Surface Model - Histogram\n NEON SJER Field Site",
> >      col = "slategray2",
> >      ylab = "Number of Pixels",
> >      xlab = "Elevation (m)")
> > ~~~
> > {: .language-r}
> > 
> > <img src="../fig/rmd-challenge-code-SJER-CHM-2.png" title="plot of chunk challenge-code-SJER-CHM" alt="plot of chunk challenge-code-SJER-CHM" style="display: block; margin: auto;" />
> > 
> > ~~~
> > # 2.
> > # use overlay to subtract the two rasters & create CHM
> > CHM_SJER <- overlay(DSM_SJER, DTM_SJER,
> >                     fun=function(r1, r2){return(r1-r2)})
> > 
> > hist(CHM_SJER,
> >      main = "Canopy Height Model - Histogram\n NEON SJER Field Site",
> >      col = "springgreen4",
> >      ylab = "Number of Pixels",
> >      xlab = "Elevation (m)")
> > ~~~
> > {: .language-r}
> > 
> > <img src="../fig/rmd-challenge-code-SJER-CHM-3.png" title="plot of chunk challenge-code-SJER-CHM" alt="plot of chunk challenge-code-SJER-CHM" style="display: block; margin: auto;" />
> > 
> > ~~~
> > # 3
> > # plot the output
> > plot(CHM_SJER,
> >      main = "Canopy Height Model - Overlay Subtract\n NEON SJER Field Site",
> >      axes = FALSE)
> > ~~~
> > {: .language-r}
> > 
> > <img src="../fig/rmd-challenge-code-SJER-CHM-4.png" title="plot of chunk challenge-code-SJER-CHM" alt="plot of chunk challenge-code-SJER-CHM" style="display: block; margin: auto;" />
> > 
> > ~~~
> > # 4
> > # Write to object to file
> > writeRaster(CHM_SJER, "chm_ov_SJER.tiff",
> >             format = "GTiff",
> >             overwrite = TRUE,
> >             NAflag = -9999)
> > 
> > # 4.Tree heights are much shorter in SJER.
> > # view histogram of HARV again.
> > par(mfcol = c(2, 1))
> > hist(CHM_HARV,
> >      main = "Canopy Height Model - Histogram\nNEON Harvard Forest Field Site",
> >      col = "springgreen4",
> >       ylab = "Number of Pixels",
> >      xlab = "Elevation (m)")
> > 
> > hist(CHM_SJER,
> >   main = "Canopy Height Model - Histogram\nNEON SJER Field Site",
> >   col = "slategrey",
> >   ylab = "Number of Pixels",
> >   xlab = "Elevation (m)")
> > ~~~
> > {: .language-r}
> > 
> > <img src="../fig/rmd-challenge-code-SJER-CHM-5.png" title="plot of chunk challenge-code-SJER-CHM" alt="plot of chunk challenge-code-SJER-CHM" style="display: block; margin: auto;" />
> {: .solution}
{: .challenge}

What do these two histograms tell us about the vegetation structure at Harvard
and SJER?
