# Optimized Regression Discontinuity Application: the effect of national institutions over local development (WORK IN PROGRESS)

# Description
This is my final project for **Stanford's ECON 293: Machine Learning and Causal Inference** for the Spring 2021 quarter.

Here I revisit the findings of [Michalopoulos, S., & Papaioannou, E. (2014). National institutions and subnational development in Africa. The Quarterly journal of economics, 129(1), 151-213.](https://academic.oup.com/qje/article-abstract/129/1/151/1897929?redirectedFrom=fulltext). In the mentioned article, the authors explore the role of national institutions on local development levels for in Africa. They exploit the fact that the national boundaries of African countries drawn during their independence partitioned more than 200 ethnic groups across adjacent countries, so within the territory of an ethnic group we have individuals subjected to similar cultures, residing in homogeneous geographic areas, but that are exposed to different national institutions.

We can use a Geographical regression discontinuity design (as the authors do in Section IV.C of their paper) to see the effect of exposure to different levels of institutions over local development levels, measured with satellite images of light density at night, and using as a running variable the distance to the national border partitioning a ethnicity.

The authors use 3rd and 4th degree RD polynomials, define the bandwidth of their RD design in arbitrary ways, and there are some pixels were the distance to the border is not computed properly, so to overcome those flaws and check how robust their findings are, I plan to use the [optrdd package](https://github.com/swager/optrdd) from [Imbens, G., & Wager, S. (2019). Optimized regression discontinuity designs. Review of Economics and Statistics, 101(2), 264-278.](https://arxiv.org/pdf/1705.01677.pdf) to estimate an Optimized Regression Discontinuity model, a data-driven model that doesn't rely on polynomial or bandwidth definition. 






## Setting environment
```R
library(foreign)
library(sf)
library(dplyr)
library(giscoR)
library(rmapshaper)
```

# Data sources

I obtained the original pixel-level dataset from [Stelios Michalopoulos' website](https://drive.google.com/file/d/1UZzwCmT7RZ7JCSx-NXAfu_-n5i6xkjRr/view?usp=sharing). I obtained the shapefiles for the ethnic tribes from [Nathan Nunn's website](https://scholar.harvard.edu/files/nunn/files/murdock_shapefile.zip). The shapefile with the grid of pixels covering Africa was provided directly by Stelios Michalopoulos.


## Importing files

```R
#M & P (2014) original Stata replication file with pixel id for the grid
dataset = read.dta("pix_ready.dta")
#shapefile for the Murdock ethnic tribes
tribe_borders = st_read("tribes/borders_tribes.shp") %>% st_transform(3857)
# M & P (2014) grid of pixels for Africa
grid = st_read("grid/pixels_africa.shp")  %>% st_transform(3857)
grid = grid[,c("FID_idspli")]

#getting maps of Africa from GISCO (Geographic Information System of the COmmission) from Eurostat
africa <- gisco_get_countries(region = "Africa", resolution = 10) %>% st_transform(3857)

#getting GADM maps of Sudan and Egypt, so our Africa map contains the Halaib Triangle 
#(disputed zone between Sudan and Egypt)
#we simplify the maps to keep 0.15% of the points
sudan <- readRDS(gzcon(url("https://biogeo.ucdavis.edu/data/gadm3.6/Rsf/gadm36_SDN_0_sf.rds"))) %>% ms_simplify(keep=0.0015) %>% st_transform(3857)
egypt  <- readRDS(gzcon(url("https://biogeo.ucdavis.edu/data/gadm3.6/Rsf/gadm36_EGY_0_sf.rds")))%>% ms_simplify(keep=0.0015) %>% st_transform(3857)

#adding the maps of Sudan and Egypt to the map of Africa
africa[africa$ISO3_CODE=="SDN",6] <- sudan[,3]
africa[africa$ISO3_CODE=="EGY",6] <- egypt[,3]

# combining original dataset with grid
dataset=merge(dataset,as.data.frame(grid),by="FID_idspli")
dataset_sf=st_as_sf(dataset)
```
## Problems with the current data
In the current data, there are some problems with how the distance to the border is calculated. We can illustrate this with the Bideyat tribe, located in the border shared among Egypt, Sudan, Chad and Libya. Most of the pixels are in Sudan and Chad, so the authors considered only those 2 countries for their analysis.

Here we are going to plot the distance to the border for the pixels in Chad (left) and Sudan (right):
```R
par(xpd=TRUE)
#country and tribe borders
plot( st_intersection(africa[africa$ISO3_CODE %in%  c("TCD","EGY","SDN","LBY"),6],
                      tribe_borders[tribe_borders$NAME=="BIDEYAT",5]),
                      lwd=2, main="Distance to the border for pixels of the Bideyat tribe")
#Sudan and Chat terrain within the Bideyat territory
base_terrain=st_intersection(africa[africa$ISO3_CODE %in% c("TCD","SDN"),6],tribe_borders[tribe_borders$NAME=="BIDEYAT",5])
plot(base_terrain,col="azure3",add=T)
#outlining tribe border
plot(tribe_borders[tribe_borders$NAME=="BIDEYAT",5],border="blue",lwd=2,add=T)
#pixels with distance to the border
plot(st_intersection(dataset_sf[dataset_sf$name=="BIDEYAT",c("bdist")],base_terrain),add=T)
#Sudan and Chad borderline
common_border=st_intersection(st_buffer(africa[africa$ISO3_CODE=="SDN",6],500),
                              st_buffer(africa[africa$ISO3_CODE=="TCD",6],500))
plot(st_intersection(tribe_borders[tribe_borders$NAME=="BIDEYAT",5],common_border), border="red",lwd=4,add=T)
#adding legend
legend("bottomright",, legend=c("Bideyat border","National borders","SDN-TCD border", "SDN-TCD territory"),
       col=c("blue","black","red","azure3"),pch=15,inset = c(-0.05, 0) )
```
![](markdown_files/figure-html/unnamed-chunk-3-1.png)

