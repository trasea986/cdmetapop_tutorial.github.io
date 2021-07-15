#### _Tutorial Contributors: Travis Seaborn, Erin Landguth_
##### _Version 0.1, Last updated July 9th 2021_
Welcome to the tutorial on creating a patch variable file for a CDMetaPOP simulation. This short tutorial gives an example of how to extract environmental data and use it to parameterize mortality at an individual patch. We will be working with 40 random patches along a stream corridor.

We will be using R to:
a) load in GPS points for our study and an example input file
b) download and extract climate data at those points
c) use the example input file and the results of the climate data to build an input file
d) export as a csv file.

In some situations, it will be simpler to open one of the existing PatchVars files in the data directory in a spreadsheet program (like Excel) and simply directly manipulate your data. Another option would be to use R to extract information about the temperature and then manually add the columns outside of R after exporting the data frame. Regardless, it is important to read the manual to understand all of the options and how each column of this input file operates. An input file must match exactly for CDMetaPOP to run correctly.

First, we will load in our libraries and import our points. We are also going to rename our coordinate columns and add in the patch ID column.

```r
setwd("./data/")

#we use check.names to allow spaces to persist
points <- read.csv("./otherfiles/tut_patches.csv", check.names = FALSE)

example_df <- read.csv("./patchvars/PatchVars_tut.csv", check.names=FALSE)

#first just add in the patch number to our existing
nrow(points)
points$PatchID <- seq(1:41)
points$X <- points$x
points$Y <- points$y
points$x <- NULL
points$y <- NULL
```

Next, we will convert these points to a SpatialDataFrame and download data from [WorldClim](https://www.worldclim.org/). The extract function below will work for any environmental data raster you might be interested in using to determine the mortality of the individuals across a heterogenous landscape.

```r
points_spat <- points
coordinates(points_spat) <- ~X+Y

#set appropriate crs for our data. your GPS unit or data source by be using something different.
proj4string(points_spat) <- CRS("+proj=utm +zone=11 +datum=WGS84")

#for our purposes we are going to download from worldclim
worldclim <- getData("worldclim", var="bio", res=2.5)


#next reproject points to match worldclim data coordinate reference system
points_spat <- spTransform(points_spat,
                              crs(worldclim))

#make sure things line up, going to buffer around points and crop the raster for plotting purposess
#this does require rgeos, but you can skip to extract values step below, cropping not required
ext <- buffer(points_spat, 5000)

# define the circles as polygons to dissolve after converting. Note: rgeos package is required to dissolve.
ext_poly <- aggregate(ext)

#crop the raster stack by this shape
bio10_study_area <- crop(worldclim$bio10, ext_poly)
plot(bio10_study_area)
points(points_spat)
```
![env_data](https://user-images.githubusercontent.com/10428038/125866526-f735deb7-eeb2-4962-9f5e-53385f583f44.png)

Now that we have some environmental data, we will be extracting Bioclim 10 and 11, which represent Mean Temperature of Warmest Quarter and Mean Temperature of Coldest Quarter. We will use these for our two seasons. Obviously for your own system, this may not be appropriate and does introduce a lot of asumptions into the simulations. The Bioclim temperatures are in 1/10 degrees celsiu, so we are also going to change that to celsius.

```r
points_spat$"GrowthTemperatureOut" <- raster::extract(worldclim$bio10, points_spat)
points_spat$"GrowthTemperatureOut" <- points_spat$"GrowthTemperatureOut"/10
points_spat$"GrowthTemperatureOutStDev" <- 0
points_spat$"GrowthTemperatureBack"<- raster::extract(worldclim$bio11, points_spat)
points_spat$"GrowthTemperatureBack"<- points_spat$"GrowthTemperatureBack"/10
points_spat$"GrowthTemperatureBackStDev"<- 0

#now  we go back to data frame. we will need points_spat later in the connectivity tutorial.
points <- as.data.frame(points_spat)
```


Next we are going to add in the other columns. Again, in some ways it is easier to just do this directly in a spreadsheet program, like Excel. We are also here manually putting values. Depending on project you might want to use a function with a conditional to automatically populate the values in each row for each columns.

```r
#don't forget to always check the example data
colnames(example_df)

#some col names have spaces, so be wary with naming in R
#patch traits and which input files to pull in at each patch
points$SubpatchNO <- 1
points$K <- 500
points$"K StDev" <- 50
points$"N0" <- 250
points$"Natal Grounds" <- c(1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 0, 0)
points$"Migration Grounds" <- c(1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 0, 0)
points$"Genes Initialize" <- c("random") #or could point to tut file
points$"Class Vars" <- c("classvars/ClassVars_tut.csv")

#here we will use a conditional for mortality out based on the temperature. Extra death when things are hot.
points <- points %>% mutate("Mortality Out %" =
                     case_when(points$"GrowthTemperatureOut" <= 19 ~ "0", 
                               TRUE ~ "50"))

points$"Mortality Out StDev" <- 5
points$"Mortality Back" <- 0
points$"Mortality Back StDev" <- 0
points$"Mortality Eggs"<- 0
points$"Mortality Eggs StDev"<- 0

#next migration columns
points$"Migration" <- 1
points$"Set Migration" <- "Y"
points$"Straying" <- 1

#number of growing days
points$"GrowDaysOut" <- 256
points$"GrowDaysOutStDev" <- 10
points$"GrowDaysBack" <- 100
points$"GrowDaysBackStDev" <- 10

#capture probability
points$"Capture Probability Out" <- 0
points$"Capture Probability Back" <- 0

#not going to play with fitness columns, but you might want to use a conditional related to temperature and implement the addtional mortality here
points$"Fitness_AA"<- 0
points$"Fitness_Aa"<- 0
points$"Fitness_aa"<- 0
points$"Fitness_BB"<- 0
points$"Fitness_Bb"<- 0
points$"Fitness_bb"<- 0
points$"Fitness_AABB"<- 0
points$"Fitness_AaBB"<- 0
points$"Fitness_aaBB"<- 0
points$"Fitness_AABb"<- 0
points$"Fitness_AaBb"<- 0
points$"Fitness_Aabb"<- 0
points$"Fitness_aaBb"<- 0
points$"Fitness_AAbb"<- 0
points$"Fitness_aabb"<- 0
```

Lastly, we are going to reorder our columns to match the example dataset and then export the results
```r
names <- colnames(example_df)

#if having issues reording, check for matching
setdiff(colnames(example_df), colnames(points))

PatchVars_final <- points[,match(names, colnames(points))]

write.csv(PatchVars_final, "./patchvars/PatchVars_create_test.csv")
```
