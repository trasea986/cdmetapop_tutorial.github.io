#### _Tutorial Contributors: Travis Seaborn, Erin Landguth_
##### _Version 0.1, Last updated July 15th 2021_
Welcome to the tutorial on creating a pair-wise distance matrix to serve as input for a CDMetaPOP simulation. We are going to cover a one distance option in R, the least-cost path. Now, CDMetaPOP can handle many connectivity matrices, including more complicated connectivity models (such as omnidirectional resistance layers from [Circuitscape](https://github.com/Circuitscape/Circuitscape.jl'). The goal of this tutorial is not to give a robust introduction to 

We are going to build a very simple input, where any location not near one of our sites will have a high value, and everywhere around our sites will have a low value.

First, we will bring in our patch file and create the buffer around our points
```r
library(raster)
library(gdistance)
library(geosphere)
library(rgdal)
library(tidyverse)

setwd("./data/")

points <- read.csv("./otherfiles/tut_patches.csv")

points_spat <- points
coordinates(points_spat) <- ~x+y

#set appropriate crs
proj4string(points_spat) <- CRS("+proj=utm +zone=11 +datum=WGS84")

#create raster/study extent. we are in meters because these are UTMS
ext <- buffer(points_spat, 2500)

# define the circles as polygons to dissolve after converting. Note: rgeos package is required to dissolve.
ext_poly <- aggregate(ext)
```


We will then next create a raster around these points, where the value of raster is 1 for the points and 1000 for areas outside of the buffer we created around the points.
```r
#create a raster around this polygon
r <- raster(extent(ext_poly)+10000)
#set raster resolution
res(r) <- 200
#make known locations low resistance...
r[ext_poly,] <- 1
#and everything else high resistance
r[is.na(r[])] <- 1000
plot(r)
points(points_spat)
```

Last, we will calculate the least-cost path across all points and then write the matrix to a csv. Note that the input file does not have patch number or ID as the first row or column.

```r
#first create the transition object
#create transition object
tr1 <- transition(1/r, transitionFunction=mean, directions=8)
tr1 <- geoCorrection(tr1, type="c") #type here should be changed if using random walks

#calculate least-cost paths
distance_matrix <- costDistance(tr1, as.matrix(points))

write.csv(as.matrix(distance_matrix), "./cdmats/tut_create_input.csv", row.names = FALSE, col.names = FALSE)
```

We can also do some simple visualization or our path
```r
#next, going to do some simple visualization
#we can look at each path
plot(shortestPath(tr1, as.matrix(points[1,]), as.matrix(points[25,]), output="SpatialLines"), add=TRUE)

#or we can view all paths from point 1. this will create an animation in your plot window
for (i in 1:41) {
  plot(shortestPath(tr1, as.matrix(points[1,]), as.matrix(points[i,]), output="SpatialLines"), add=TRUE)
}
```
![LC-demo](https://user-images.githubusercontent.com/10428038/125870478-da83db98-9d1f-4f6c-a17d-e5ecb25644d2.png)


