#### GISC 422 T1 2019
# Simple interpolation methods in *R*
<hr>

## THIS IS NOT THE 2020 EDITION OF THESE INSTRUCTIONS!
**Instructions need to be updated to use `sf` and `tmap` and friends.**

<hr>

You'll need to [download the data](week-7.zip?raw=true) and unzip them.

Load some libraries.

```{r}
library(rgdal) # for reading files
library(maptools) # for various conversions of formats
library(spatstat) # for making proximity polygons and doing IDW
library(tmap) # mapping library
library(raster) # handling raster data
```
Note that we need `rgdal` because it is the `Spatial*DataFrame` style of geospatial data in *R* that is more compatible with the libraries we need to use for these interpolations. It is likely that in a year or two these libraries will be easier to work with using the `sf` package.

## Some sample data
Load a dataset that we will use to explore the methods.

```{r}
heights <- readOGR("maungawhau.shp")
```

This is a grid of control points at 10 metre spacing for Maungawhau (Mt. Eden) in Auckland. It was originally digitised by Ross Ihaka at University of Auckland and is a dataset that comes with R in the form of a matrix of height values as the `volcano` dataset. You can get an idea of what you are looking at in a couple of ways.

One is a perspective view of the matrix data

```{r}
persp(volcano, theta=30, phi=35, expand=0.25, lwd=0.5, shade=0.5, col='green')
```

Experiment with some of the parameters of this plot, to see how they affect the view.

But we won't be using that version of the data, we'll be working with the shapefile points we just read in. We can confirm these look the same, by making a map.

```{r}
tm_shape(heights) +
  tm_dots(col='height', size=0.1, pal='-BrBG') +
  tm_legend(legend.outside=T)
```

Since this is the *result* we are looking for from any interpolation we run, we need to turn this into a set of *control point* data. This is a bit fiddly so I have made a function to make it easier.

```{r}
## Function to return a sample from a point layer
## By default will take only 100 control points
get_controls <- function(layer, n=100, jitter=5) {
  picks <- sample(1:nrow(layer), n)
  selected <- layer[picks,]
  selected@coords[,1] <- selected@coords[,1] + jitter * (runif(n) - 0.5)
  selected@coords[,2] <- selected@coords[,2] + jitter * (runif(n) - 0.5)
  return(selected)
}
```

Now use the function to make a set of control points.

```{r}
controls <- get_controls(heights) # this is a subset of the heights
cpp <- as(controls, 'ppp')
plot(cpp)
```

Note that every time you run the above cell, you will get a different random sample of the data. Also note that it would be very unusual, especially with elevation data for the control points to be a random sample. It is much more likely that surveyors will collect elevation data for key locations.

## Two simple deterministic interpolation methods
To figure out what follows I drew heavily on the Bivand et al. book, and also on [this useful source](https://mgimond.github.io/Spatial/interpolation-in-r.html).

### Proximity polygons
The idea behind proximity polygons is that we simply assign to every location the value associated with the nearest control point. We use the `dirichlet` function in `spatstat` to do this (Dirichlet was yet another reinventor of Thiessen/Voronoi/proximity polygons).

```{r}
pph  <-  as(dirichlet(cpp), "SpatialPolygons")
proj4string(pph) <- proj4string(controls)
plot(pph, lwd=0.5)
```

We need to add to this the data from the points, which we do using an overlay operation.

```{r}
pph.height <- sp::over(pph, controls, fn=mean)
pph.sp <- SpatialPolygonsDataFrame(pph, pph.height)
```

And now we can make a map of the result.

```{r}
tm_shape(pph.sp) +
  tm_polygons(col='height', pal='-BrBG') +
  tm_legend(legend.outside=T)
```

Fairly obviously, this method doesn't do very well with a small number of control points. Try going back and making a larger set of control points to see how many you need before the results are acceptable.

### Inverse distance weighted interpolation
Another option in `spatstat` is IDW. Now we have a `ppp` set of control points to work with, it's pretty easy to run.  But we'll get a new set of random control points to start.

```{r}
controls <- get_controls(heights, n=200)
cpp <- as(controls, 'ppp')
```

It's a simple matter to perform IDW with these.

```{r}
idw_h <- spatstat::idw(cpp, power=2)
```

That's it!

Now to plot it... we need to make this into a raster with which we can associate spatial reference information.

```{r}
idw_h <- raster(idw_h)
proj4string(idw_h) <- proj4string(heights)

tmap_mode("plot")

tm_shape(idw_h) +
  tm_raster(n=10, palette='-BrBG') +
  tm_shape(controls) + tm_dots(size=0.1) +
  tm_legend(legend.outside=T)
```

A better way to see the limitations of this interpolation is a perspective view, when what is going on should be very clear.

```{r}
persp(idw_h, theta=15, phi=25, expand=0.5)
```

### IDW interpolation retaining spatial properties
To do IDW in a less quick and dirty way we can use the `gstat` package. We'll see more of this next week, when we use its more advanced options.

```{r}
library(gstat)
```

You'll get a message warning you that the `idw()` function from `spatstat` has now been 'masked' by a similarly named function in `gstat`.

```{r}
# Make a sample set of control points
controls <- get_controls(heights, n=200)

# we make an empty spatial pixels dataset from
# the original dataset we read in
result <- as(heights, 'SpatialPixelsDataFrame')

# Interpolate by IDW function from gstat, with power 2
idw_h <- gstat::idw(height ~ 1, controls, newdata=result, idp=2.0)
persp(as.matrix(idw_h), expand=0.5, theta=-30, phi=25)

# For mapping make a raster object
r <- raster(idw_h)
crs(r) <- proj4string(heights)

# Plot
tm_shape(r) +
  tm_raster(n=10,palette = "-BrBG", title="IDW heights") +
  tm_legend(legend.outside=T)
```

### Splines
#### Using `MBA::mba.surf`
One library we can use for splines is `MBA`. (You will need to install this.)

```{r}
library(MBA)
```

Below is an example of how the `mba.surf` function can be used to perform spline interpolation. You should take a look at the help for this function to see if you can figure out what's going on.

```{r}
controls <- get_controls(heights, n=200)

xyz <- data.frame(x=controls@coords[,1],
                  y=controls@coords[,2],
                  z=controls@data$height)

spline_h <- mba.surf(xyz, no.X=61, no.Y=87, n=87/61, m=1,
                     extend=T, sp=T, b.box=as.vector(t(controls@bbox)))$xyz.est

r <- raster(spline_h)
crs(r) <- proj4string(heights)

p <- persp(r, theta=150, phi=35, expand=0.25)
points(trans3d(controls@coords[,1], controls@coords[,2], controls@data[,1], p), col='red', pch=20, cex=0.6)

tm_shape(r) +
  tm_raster(n=10,palette = "-BrBG", title="mba.surf spline heights") +
  tm_legend(legend.outside=T) +
  tm_shape(controls) + tm_dots(size=0.1)
```

#### Using `akima::interp`
Yet another tool that can perform spline interpolation is found in the `akima` package (again you may have to install this.)

```{r}
library(akima)
```

The function is called `interp`.  Again, it has a few settings.

```{r}
controls <- get_controls(heights, n=200)

xyz <- data.frame(x=controls@coords[,1],
                  y=controls@coords[,2],
                  z=controls@data$height)

spline_h <- interp(xyz$x, xyz$y, xyz$z, nx=61, ny=87, extrap=T, linear=F)

persp(spline_h, col=rgb(0.5, 1, 0.5, 0.1), theta=120, phi=35, expand=0.25)

r <- raster(spline_h)
crs(r) <- proj4string(heights)

tm_shape(r) +
  tm_raster(n=10, palette='-BrBG', title='akima interp spline heights') +
  tm_legend(legend.outside=T) +
  tm_shape(controls) + tm_dots(size=0.1)
```
