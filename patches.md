#### _Tutorial Contributors: Travis Seaborn, Erin Landguth_
##### _Version 0.1, Last updated July 15th 2021_
Welcome to the tutorial on creating a patch variable file for a CDMetaPOP simulation. This short tutorial gives an example of how to extract environmental data and use it to parameterize mortality at an individual patch. We will be working with 40 random patches along a stream corridor.

##### Creating the patch file
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
<p align="center">
  <img src="https://user-images.githubusercontent.com/10428038/125866526-f735deb7-eeb2-4962-9f5e-53385f583f44.png">
</p>

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

#### Patches Column Details
Below is the description for each of the columns from the manual. See manual for full details.
xyfilename	‘PatchVars.csv’ – see example supplied for 7 patches:
‘PatchVars_climate.csv’ – an example on how to use cdclimate module changing parameters.
‘PatchVars_GMSelection.csv’ – an example for how to use the growth and maturation selection module.
‘PatchVars_IntroducePopulation.csv’ – an example for how to introduce additional individuals at given time units.
‘PatchVars_MultiClassVars.csv’ – an example for how to use multiple ClassVars files to simulate more than one species or groups of individuals. 
	The n-(x,y) patch locations with 28 columns/fields of information for each patch.  This is a comma delimited file. Some fields are optional and will not be considered – leave default values. 

Pparameters can vary through time either systematically or through temporal variation. Parameters with ‘_Std’ can be used for temporal variation and to create a covariance matrix with selected parameters (e.g., correlation_matrix: Patch_r.csv). See CDClimate section for details on how to implement systematic temporal variation. 

(PatchID)- a unique numerical identifier for each patch. Begin label 1 through n in consecutive order. 

(X)-x-coordinate location of patch.

(Y)-y-coordinate location of patch.

(SubPatchID) – This is a unique text/string identifier for each patch used to identify individuals labeled to a particular region. This field is used as a ‘tag’ ID for each individual and reported also in output files. 

(K)-the carrying capacity for each patch. A value of 0 can be included here and individuals will not disperse (migration out or stray) to these patches.

(K_Std)-the standard deviation on K for each patch which will vary patch K values through time. If a constant K per patch is wished, then enter 0 here.

(N0)-corresponding patches will be initialized with this number of individuals. cdclimate module can be initiated and multiple N0 values can be specified, e.g., for cdclimategen = 0|5|10, N0 = 100|10|10. This has a unique meaning, in that, it represents introduced individuals into that patch. A ClassVars or GenesInit file can be used to initialize these individuals. See PatchVars_IntroducePopulation.csv as an example for how to implement this option. Multiple species per patch can be initiated here by using ‘;’ delimiter. 
(Natal Grounds)-designates natal ground locations. ‘1’ – individuals can occupy this patch for e.g., spawning. ‘0’ – individuals cannot occupy this patch when they are ‘back’. N0 cannot be > 0 (initialization value starting individuals at natal grounds), if natal ground patch is 0. 

(Migration Grounds)-designated migration ground locations. ‘1’ – individuals can occupy this patch during DoEmigration() process (e.g., overwintering). ‘0’ individuals cannot occupy this patch when they are ‘out’. 

(Genes Initialize)- The choice for how to initialize the genotype for each individual. Different initialization can be used for each patch.
Use ‘random’ or ‘random_var’ to get a random assignment of alleles (number of alleles and loci specified in PopVars.csv). Patch will be at maximum genetic diversity with this option. ‘random_var’ will give a variable allele / locus option specified in the PopVars file field alleles. ‘random’ will assume the the number of alleles per locus are all the same. 
Enter an allele frequency file name: Then the alleles are drawn from the allele frequency distribution in file. Examples of format in comma delimited form are given in the data/genes/ folder location. This file is a column of allele frequencies and make sure the length of the column matches starting loci * starting alleles given in the PopVars.csv file. Different allele frequency files can be given for different patches, making sure they are the same size (loci * alleles).
In addition, you can choose to initialize each patch with multiple allele frequency files. This is achieved by using a ‘;’ to separate each file. E.g., allelefrequecyA.csv;allelefrequencyB.csv;allelefrequencyC.csv would randomly choose one of these three files to use for gene initialization for an individual. Note that this is a different deliminator than the ‘|’ which is linked to temporal changing files. 
(ClassVars)-the ClassVars input file that governs this patch location. See more on ClassVars specific parameters in next section 3.2. Note that different ClassVars files can be given at each patch, as well as through time with introduced populations. 
You can choose to initialize each patch with multiple ClassVars files that are linked to the multiple allele frequency files given in previous field. This is achieved by using a ‘;’ to separate each file. E.g., allelefrequecyA.csv;allelefrequencyB.csv;allelefrequencyC.csv could be associated with ClassVars files in the same patch, e.g., ClassVarsA.csv;ClassVarsB.csv;ClassVarsC.csv. An individual in a patch would get a random initial allele frequency assignment and then corresponding ClassVars assignment. Note that this species/space deliminator (‘;’) is a different deliminator than the ‘|’ which is linked to temporal changing files. 

(Mortality Out %) – This is the constant mortality percentage [0-100] applied to each patch in the rearing/overwintering/foraging stage. 
Enter ‘N’ here and this field is not considered. Note this is compounded with age and size level mortalities with given constMortans.
In the special case in which e.g., eradication as well as suppression operating at the class level is desired, then enter ‘E’. This will override any class level mortality values (described below).
(Mortality Out StDev) – This is the standard deviation around the ‘Mortality Out %’ [0-100] applied to each patch. Enter 0 here and this field will be deterministic, using the ‘Mortality Out %’ value at each time step. 

(Mortality Back %) – This is the constant mortality percentage [0-100] applied to each patch after immigration back to original/natal population. 
Enter ‘N’ here and this field is not considered. Note this is compounded with age and size level mortalities with given constMortans.
In the special case in which e.g., eradication as well as suppression operating at the class level is desired, then enter ‘E’. This will override any class level mortality values (described below).
(Mortality Back StDev) – This is the standard deviation around the ‘Mortality Back %’ [0-100] applied to each patch. Enter 0 here and this field will be deterministic, using the ‘Mortality Back %’ value at each time step. 

(Mortality Eggs %) – This is the egg or Age 0 mortality percentage [0-100]. Note that there is also a population wide level consideration on egg mortality (see below) in PopVars file. If used together, events will be considered mutually exclusive (additive).   
(Mortality Eggs StDev) – This is the standard deviation around the ‘Mortality Eggs %’ [0-100] applied to each patch. Enter 0 here and this field will be deterministic, using the ‘Mortality Eggs %’ value at each time step. 

(Migration) – The emigration probability [0-1] applied before moving to rearing/overwinter grounds. If just age/size class migration is wanted, then set these values to 1. 

(Set Migration) – If an individual becomes a migrant, it can stay a migrant with probability 1.0. 
To turn this on, specify ‘Y’. 
To turn this off, specify ‘N’.

(Straying) – The straying probability [0-1] applied before moving back to original/natal grounds. Set these values to 1 and age class can govern through Classvars. 

(Growth Temperature Out) – Temperature values that influence body size growth of individuals at patch location after DoEmigration() or migration out of natal grounds (e.g., during fall/winter for a spring breeder). 
Extract temperature values spatially to patch locations, or
Enter ‘N’ for individual patches or all, which can turn off temperature dependent patch specific growths at this time of year.
(Growth Temperature Out StDev) – This is the standard deviation around each temperature value ‘out’ and applied to each patch. Enter 0 here and this field will be deterministic, using the ‘Growth Temperature Out’ value at each time step. 

(Growing Days Out) – Patch site growing days entered here corresponding to the growth time ‘out’ of natal grounds in foraging patches for growth option ‘temperature’. If ‘N’ is used in the ‘Growth Temperature Out’, then this value is ignored. 
(Growth Days Out StDev) – This is the standard deviation around each grow days ‘out’ and applied to each patch. Enter 0 here and this field will be deterministic, using the ‘Growth Days Out’ value at each time step. 

(Temperature Back Growth) – Temperature values that influence body size growth of individuals at patch location after DoImmmigration() or migration back to natal grounds (e.g., during summer for a spring breeder). 
Extract temperature values spatially to patch locations, or
Enter ‘N’ for individual patches or all, which can turn off temperature dependent patch specific growths at this time of year.
(Growth Temperature Back StDev) – This is the standard deviation around each temperature value ‘back’ and applied to each patch. Enter 0 here and this field will be deterministic, using the ‘Growth Temperature Back’ value at each time step. 

(Growing Days Back) – Patch site growing days entered here corresponding to the growth time ‘back’ at natal grounds. If ‘N’ is used in the ‘Growth Temperature Back’, then this value is ignored.
(Grow Days Back StDev) – This is the standard deviation around each grow day value ‘back’ and applied to each patch. Enter 0 here and this field will be deterministic, using the ‘Grow Days Back’ value at each time step. 

(Capture Probability Back) – the probability of detection, capture, or sampling for each patch when individuals are back at natal grounds (e.g., for spring spawning fish during the summer). This occurs right before the individuals are preparing to migrate out of their natal grounds. 
Enter a probability [0-1] or 
Enter ‘N’ to not consider this option.
Enter 1 and class level capture probability will operate (ClassVars).

(Capture Probability Out) – the probability of detection, capture, or sampling for each patch when individuals are away from their natal grounds (e.g., for spring spawning fish during the winter). This occurs right before the individuals are preparing to migrate back to their natal grounds.
Enter a probability [0-1] or 
Enter ‘N’ to not consider this option.
Enter 1 and class level capture probability will operate (ClassVars).

(Fitness_AA, Fitness Aa, Fitness_aa) – When CDEVOLVE answer is 1 (in PopVars.csv file), then this is the offspring viability selection values for AA, Aa, and aa, respectively (i.e., one-locus selection model). These are differential mortality values tied to each genotype. You can link the selection via mortality to spatial environmental-genotype processes. 
When CDEVOLVE answer is ‘M’ or ‘MG’ (in PopVars.csv file), then these 3 genotypes are used to enter 2 - 6 maturation response parameters in the order and s slope:intercept. The 2-6 values must be entered and separated by a ‘:’. If 2 values are entered, then all sex classes will follow the same logistic maturation curve. If 4 are entered, then female_slope:female_intercept~male_slope:male_intercept. If 6 values are entered, then YY males will be considered for a different maturation curve, female_slope:female_intercept~male_slope:male_intercept~ YYmale_slope:YYmale_intercept. CDCLIMATE can still be considered, e.g., 0.06:23~0.08:21|0.08:24~0.10:22 using the ‘|’ to separate each parameter group to be considered. 
When CDEVOLVE answer is ‘stray’ (in PopVars.csv file), then these genotypes are used to enter stray rates, e.g., AA could be 0.05, Aa could be 0.01 and aa could be 0.00 indicating individuals with the genotype AA having a higher stray tendency than individuals with Aa or aa.

(Fitness_BB, Fitness Bb, Fitness_bb) – When CDEVOLVE answer is ‘G’ or ‘MG’ (in PopVars.csv file), then these 3 genotypes are used to enter 5 growth rate response parameters in the order 
temperature:353:0.57:13:0.33:-0.196. The 6 values must be entered and separated by a ‘;’. CDCLIMATE can still be considered, e.g., temperature:353:0.57:13:0.33:-0.196| temperature:353:0.57:13:0.33:-0.196 using the ‘|’ to separate each parameter group to be considered. 

(Fitness_AABB, Fitness AaBB, Fitness_aaBB, Fitness AABb, Fitness AaBb, Fitness aaBb, Fitness Aabb, Fitness Aabb, Fitness aabb) – When CDEVOLVE answer is 2, then this is the offspring viability selection values for each of the 9 genotypes (i.e., two-locus selection model). These are differential mortality values tied to each genotype. You can link the selection via mortality to spatial environmental-genotype processes. For example, to consider one genotype under directional selection, e.g., AABB, enter spatial mortality values here [0-100] and all other genotypes having 0% mortality. 
