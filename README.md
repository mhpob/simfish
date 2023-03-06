
<!-- README.md is generated from README.Rmd. Please edit that file -->

# simfish

<!-- badges: start -->

[![Project Status: WIP – Initial development is in
progress.](http://www.repostatus.org/badges/latest/wip.svg)](http://www.repostatus.org/#wip)
<!-- badges: end -->

simulate fish tracks with/without acoustic detections in semi-realistic
environments

## Installation

You can install simfish from
[GitHub](https://github.com/ianjonsen/simfish) with:

``` r
# install.packages("remotes")
remotes::install_github("ianjonsen/simfish")
```

## Example

``` r
require(simfish, quietly = TRUE)
require(raster, quietly = TRUE)
require(tidyverse, quietly = TRUE)
require(sf, quietly = TRUE)
require(stars, quietly = TRUE)
```

## Create an environment to simulate fish movements

``` r
## create raster using rnaturalearth polygon data
land <- rnaturalearth::ne_countries(scale = 10, returnclass = "sf") %>%
    sf::st_transform(crs = 4326) %>%
    sf::st_crop(xmin=-68, ymin=43.5, xmax=-52, ymax=52) %>%
    sf::st_make_valid()

## rasterise
land <- as(land, "Spatial")

## rasterize at a high resolution for a pretty map
##  a lower resolution will run faster
land <- raster::raster(crs = crs(land), 
                   vals = 1, 
                   resolution = c(0.01, 0.01), 
                   ext = extent(c(-68, -52, 43.5, 52))) %>% 
  raster::rasterize(land, .)

## reproject to Mercator grid in km
## in principle, any projection will work as long as the units are in km
ext <- raster::projectExtent(land, crs = "+proj=merc +datum=WGS84 +units=km")
land <- raster::projectRaster(land, ext)
land[land > 1] <- 1
land2 <- land # make a copy

## set water = 1 & all land values to NA
land2[is.na(land2)] <- -1
land2[land2 == 1] <- NA
land2[land2 == -1] <- 1

## calculate gradient rasters - these are needed to keep fish off land
dist <- raster::distance(land2)
grad <- ctmcmove::rast.grad(dist)
grad <- raster::stack(grad$rast.grad.x, grad$rast.grad.y)

## write files for repeated use
writeRaster(grad, file = "data/grad.grd", overwrite = TRUE)
writeRaster(land, file = "data/land.grd", overwrite = TRUE)
```

## Set up the simulation

``` r
## first create a list with the required data
d <- list(land = raster("data/land.grd"), 
          grad = stack("data/grad.grd"), 
          prj = "+proj=merc +datum=WGS84 +units=km")

## then parameterize the simulation
my.par <- simfish::sim_par(
  N = 70*24,  # number of simulation time steps (60 days x 24 h)
  start = c(-7260, 5930), # start location for simulated fish
  start.dt = ISOdatetime(2023,03,01,12,00,00, tz="UTC"), # start date-time
  time.step = 60, # time step in minutes
  coa = c(-6250, 6000), # location of centre-of-attraction for biased correlated random walk (can be NA)
  nu = 0.1, # strength of attraction to CoA (range: 0 - infinity)
  fl = 0.15, # fish fork length in m
  rho = 0.95, # angular dispersion param for move directions (range 0 - 1)
  bl = 2, # move speed (body-lengths per s)
  beta = c(-2, -2) # potential function params; must be -ve to keep off land;
  # larger -ve numbers will cause the track to jump (unrealistically) big distances when encountering land,
  # possibly crossing narrow land masses
)
```

## Run simulation

``` r
## specify random seed to ensure same track is simulated each time this .Rmd runs
##  DO NOT do this for 'real' applications!
set.seed(100)

## simulate a single track
out <- sim_fish(id = 1, 
                data = d, 
                mpar = my.par,
                pb = FALSE) # turn off progress bar for tidy Rmd result
```

## Visualise simulated track

``` r
## convert raster to stars object

ggplot() +
  stars::geom_stars(data = stars::st_as_stars(d$land)) +
  geom_point(data = out$sim, 
             aes(x, y), 
             size = 0.1, 
             colour = "orange") +
  geom_point(data = with(out$params, data.frame(x = coa[1], y = coa[2])),
             aes(x, y),
             shape = 17,
             size = 2, 
             colour = "green") +
  theme_minimal() +
  theme(legend.position = "none") + 
  labs(x = element_blank(),
       y = element_blank()) +
  coord_sf(expand = FALSE)
```

<img src="man/figures/README-map results-1.png" width="100%" />
