---
layout: post
title: Digitize graphs with R - Import and edit graphs from PDF files
---

-   [What is this about?](#what-is-this-about)
-   [From PDF to PostScript to "Picture" object](#from-pdf-to-postscript-to-picture-object)
    -   [Import PostScript file in R with `grImport`](#import-postscript-file-in-r-with-grimport)
    -   [Read in the PostScript file and create a "Picture" object](#read-in-the-postscript-file-and-create-a-picture-object)
-   [Manipulate a "Picture" object](#manipulate-a-picture-object)
    -   [Visualize and clean (subset) the "Picture" object](#visualize-and-clean-subset-the-picture-object)
    -   [Extract PostScript coordinates from the cleaned "Picture" object](#extract-postscript-coordinates-from-the-cleaned-picture-object)
    -   [Convert PostScript coordinates into meaningful coordinates](#convert-postscript-coordinates-into-meaningful-coordinates)
    -   [Convert between coordinates systems](#convert-between-coordinates-systems)
    -   [Save coordinates as data frame](#save-coordinates-as-data-frame)
    -   [Assign the colors also](#assign-the-colors-also)
-   [Plotting examples](#plotting-examples)

What is this about?
===================

We need to reproduce a graph from a book or article. For example, importing the graph on the left and being able to plot something like the graph on the right.

<img src="Import_PostScript_grImport_files/figure-markdown_github/unnamed-chunk-1-1.png" alt="Graph to import in R (left) and a possible outcome (right)." width="45%" /><img src="Import_PostScript_grImport_files/figure-markdown_github/unnamed-chunk-1-2.png" alt="Graph to import in R (left) and a possible outcome (right)." width="45%" />
<p class="caption">
Graph to import in R (left) and a possible outcome (right).
</p>

For my simple [`plotbiomes`](https://github.com/valentinitnelav/plotbiomes) R package, I imported the `PostScript` version of the above graph on the left using [`grImport`](https://cran.r-project.org/web/packages/grImport/index.html) package. Then I created a data frame of coordinates that can be used for plotting with `ggplot2` package functionality (obtaining the graph on the right).

From PDF to PostScript to "Picture" object
==========================================

The original graph is Figure 5.5 in *Ricklefs, R. E. (2008), The economy of nature. W. H. Freeman and Company.* (Chapter 5, Biological Communities, The biome concept).

A PDF page containing the Whittaker biomes graph was imported in [Inkscape](https://inkscape.org/en/). Text and extra graphic layers where removed and the remaining layers were exported as PostScipt file format (`File > Save As > Save as type: > PostScript (*.ps)`). Note that a multi-page PDF document can be split into component pages with [PDFTK Builder](https://www.pdflabs.com/tools/pdftk-the-pdf-toolkit/) application.

Import PostScript file in R with `grImport`
-------------------------------------------

The main steps of importing PostsScipt in R are described in the [Importing vector graphics](https://cran.r-project.org/web/packages/grImport/vignettes/import.pdf) vignette: *Murrell, P. (2009). Importing vector graphics: The grImport package for R. Journal of Statistical Software, 30(4), 1-37.*

**In addition to installing package `grImport` with the usual `install.packages("grImport")`, [Ghostscript](https://www.ghostscript.com/download/) needs to be installed as well.**

Read in the PostScript file and create a "Picture" object
---------------------------------------------------------

To avoid the ghostscript error 'status 127' (at least on Windows OS), the path to ghostscript `gswin64c.exe` executable file was given as suggested [here](https://stackoverflow.com/questions/35256501/windows-rgrimport-ghostscript-error-status-127#42393056). The PostScript file `graph_ps.ps` produced with Inkscape can be downloaded from [here](https://raw.githubusercontent.com/valentinitnelav/valentinitnelav.github.io/master/_posts/2018-01-06-Import-PostScript-grImport/img/graph_ps.ps).

``` r
# Set a working directory with `setwd()`.
require(grImport)
# Avoid the ghostscript error 'status 127'.
Sys.setenv(R_GSCMD = normalizePath("C:/Program Files/gs/gs9.22/bin/gswin64c.exe"))
# Converts a PostScript file into an RGML file.
# Change file paths if needed. Fallow the link mentioned above for downloading `graph_ps.ps`.
# Note that PostScriptTrace() will creat a `capturegraph_ps.ps` file in your working directory.
grImport::PostScriptTrace(file = "img/graph_ps.ps",
                          outfilename = "img/graph_ps.xml",
                          setflat = 1)
# setflat = 1 assures a visual smooth effect (feel free to experiment)

# Reads in the RGML file and creates a "Picture" object
my_rgml <- readPicture(rgmlFile = "img/graph_ps.xml")
```

Manipulate a "Picture" object
=============================

Visualize and clean (subset) the "Picture" object
-------------------------------------------------

``` r
# Draws the picture object
plot.new(); grid.picture(my_rgml)
# Draws each path composing the picture. This helps selecting desired paths.
plot.new(); picturePaths(my_rgml)
```

<img src="Import_PostScript_grImport_files/figure-markdown_github/unnamed-chunk-3-1.png" alt="The picture object (left) and its component paths (right). The picture was flipped in Inkscape." width="45%" /><img src="Import_PostScript_grImport_files/figure-markdown_github/unnamed-chunk-3-2.png" alt="The picture object (left) and its component paths (right). The picture was flipped in Inkscape." width="45%" />
<p class="caption">
The picture object (left) and its component paths (right). The picture was flipped in Inkscape.
</p>

By using `picturePaths` one can identify the desired biome lines and the filled polygons. From polygons one can get the fill colors for further use. It requires some trial and error until identifying the indices of each element.

``` r
# Selects desired paths from the picture object. 
# These are the path corresponding to the filled polygons.
my_fills <- my_rgml[c(3,9,14,16,18,21,23,25,27)]
# Gets the colors
colors <- vector(mode = "character", length = length(my_fills@paths))
for (i in 1:length(my_fills@paths)){
  colors[i] <- my_fills@paths[[i]]@rgb
}

# These are the path corresponding to the biome lines (polygon boundaries)
my_rgml_clean <- my_rgml[c(4,10,15,17,19,22,24,26,28)]
# Assigns colors to the lines
for (i in 1:length(my_rgml_clean@paths)){
  my_rgml_clean@paths[[i]]@rgb <- colors[i]
}

plot.new(); grid.picture(my_fills)
plot.new(); grid.picture(my_rgml_clean)
```

<img src="Import_PostScript_grImport_files/figure-markdown_github/unnamed-chunk-4-1.png" alt="Polygons (left) and their borders (right)" width="45%" /><img src="Import_PostScript_grImport_files/figure-markdown_github/unnamed-chunk-4-2.png" alt="Polygons (left) and their borders (right)" width="45%" />
<p class="caption">
Polygons (left) and their borders (right)
</p>

Extract PostScript coordinates from the cleaned "Picture" object
----------------------------------------------------------------

``` r
x_lst <- vector(mode = "list", length = length(my_rgml_clean@paths))
y_lst <- vector(mode = "list", length = length(my_rgml_clean@paths))
for (i in 1:length(my_rgml_clean@paths)){
  x_lst[[i]] <- my_rgml_clean@paths[[i]]@x
  names(x_lst[[i]]) <- rep(i, length(x_lst[[i]]))
  y_lst[[i]] <- my_rgml_clean@paths[[i]]@y
  names(y_lst[[i]]) <- rep(i, length(x_lst[[i]]))
}
x <- unlist(x_lst)
y <- unlist(y_lst)

par(mar = c(4, 4, 1, 1))
plot(x,y)
```

<img src="Import_PostScript_grImport_files/figure-markdown_github/unnamed-chunk-5-1.png" alt="Plot using the original PostScript coordinates" width="45%" />
<p class="caption">
Plot using the original PostScript coordinates
</p>

Convert PostScript coordinates into meaningful coordinates
----------------------------------------------------------

This varies from graph to graph, but in this case luckily there is a grid and axis on the graph that can be used to transform from PostScript coordinates to meaningful coordinates (temperature and precipitations in this example).

``` r
my_grid <- my_rgml[1]
my_axis <- my_rgml[30]

x_grid <- my_grid@paths[[1]]@x
y_grid <- my_grid@paths[[1]]@y
x_grid_unq <- unique(sort(x_grid))
y_grid_unq <- unique(sort(y_grid))

# Plot original grid and axis
plot.new()
grid.picture(my_grid)
grid.picture(my_axis)

# Plot in PostScript coordinates
par(mar = c(4, 4, 1, 1))
plot(x_grid, y_grid, pch = 16, col = "red")
points(x, y, cex = 0.1)
abline(v = x_grid_unq[2], col = "red") # X min ~ -10
abline(v = max(my_axis@summary@xscale), col = "red") # X max ~ 30
abline(h = min(my_axis@summary@yscale), col = "red") # Y min ~ 0
abline(h = y_grid_unq[6], col = "red") # Y max ~ 400
```

<img src="Import_PostScript_grImport_files/figure-markdown_github/unnamed-chunk-6-1.png" alt="Original grid and axis (left) and their PostScript coordinates (right)" width="45%" /><img src="Import_PostScript_grImport_files/figure-markdown_github/unnamed-chunk-6-2.png" alt="Original grid and axis (left) and their PostScript coordinates (right)" width="45%" />
<p class="caption">
Original grid and axis (left) and their PostScript coordinates (right)
</p>

Convert between coordinates systems
-----------------------------------

Need to identify the corresponding min & max values on OX and OY axis on both coordinates systems. The conversion equations are those exemplified [here](https://gamedev.stackexchange.com/questions/32555/how-do-i-convert-between-two-different-2d-coordinate-systems).

``` r
# Notations:
# scs - source coordinate system
# rcs - result coordinate system

x_min_scs <- x_grid_unq[2] # is the X of the left most vertical grid line
x_min_rcs <- -10 # corresponds to -10
x_max_scs <- max(my_axis@summary@xscale) # is the X max of OX axis line
x_max_rcs <- 30 # corresponds to 30

y_min_scs <- min(my_axis@summary@yscale) # is the Y min of OY axis line
y_min_rcs <- 0 # corresponds to 0
y_max_scs <- y_grid_unq[6] # is the Y of the upper most horizontal grid line
y_max_rcs <- 400

# Apply conversion
xx <- (x - x_min_scs)/(x_max_scs - x_min_scs) * (x_max_rcs - x_min_rcs) + x_min_rcs
yy <- (y - y_min_scs)/(y_max_scs - y_min_scs) * (y_max_rcs - y_min_rcs) + y_min_rcs

# There should not be negative values on OY.
# This happens because the original PDF has some artefacts,
# including also some overlapping polygons sections.
min(yy) 
```

    ## [1] -1.4412

``` r
yy[yy < 0] <- 0

# Plot in original coordinates
par(mar = c(4, 4, 1, 1))
plot(x_grid, y_grid, pch = 16, col = "red", 
     xlab = "X - original coordinates",
     ylab = "Y - original coordinates")
points(x, y, cex = 0.1)
abline(v = x_grid_unq[2], col = "red") # X min ~ -10
abline(v = max(my_axis@summary@xscale), col = "red") # X max ~ 30
abline(h = min(my_axis@summary@yscale), col = "red") # Y min ~ 0
abline(h = y_grid_unq[6], col = "red") # Y max ~ 400

# Plot in converted coordinates
plot(xx, yy, cex = 0.1,
     xlab = "X - converted coordinates",
     ylab = "Y - converted coordinates")
abline(v = -10, col = "red") # X min
abline(v = 30, col = "red") # X max
abline(h = 0, col = "red") # Y min
abline(h = 400, col = "red") # Y max
```

<img src="Import_PostScript_grImport_files/figure-markdown_github/unnamed-chunk-7-1.png" alt="Left - original PostScript coordinates; Right - converted coordinates (precipitation vs. temperature)" width="45%" /><img src="Import_PostScript_grImport_files/figure-markdown_github/unnamed-chunk-7-2.png" alt="Left - original PostScript coordinates; Right - converted coordinates (precipitation vs. temperature)" width="45%" />
<p class="caption">
Left - original PostScript coordinates; Right - converted coordinates (precipitation vs. temperature)
</p>

Save coordinates as data frame
------------------------------

``` r
Whittaker_biomes <- data.frame(temp_c = xx, 
                               precp_cm = yy, 
                               biome_id = as.numeric(names(xx)))

# Assigns the biome names
for (i in 1:nrow(Whittaker_biomes)){
  Whittaker_biomes$biome[i] <- switch(as.character(Whittaker_biomes$biome_id[i]),
                                      "1" = "Tropical seasonal forest/savanna",
                                      "2" = "Subtropical desert",
                                      "3" = "Temperate rain forest",
                                      "4" = "Tropical rain forest",
                                      "5" = "Woodland/shrubland",
                                      "6" = "Tundra",
                                      "7" = "Boreal forest",
                                      "8" = "Temperate grassland/desert",
                                      "9" = "Temperate seasonal forest")
}

head(Whittaker_biomes)
```

    ##     temp_c precp_cm biome_id                            biome
    ## 1 18.65201 78.32278        1 Tropical seasonal forest/savanna
    ## 2 18.67003 79.43000        1 Tropical seasonal forest/savanna
    ## 3 18.72035 82.76120        1 Tropical seasonal forest/savanna
    ## 4 18.77968 87.15266        1 Tropical seasonal forest/savanna
    ## 5 18.83094 91.46236        1 Tropical seasonal forest/savanna
    ## 6 18.87450 95.65887        1 Tropical seasonal forest/savanna

Assign the colors also
----------------------

The fill colors where already extracted from PostScript. In the help of `ggplot2::scale_fill_manual` it is recommended to use a named vector. Therefore, colors were stored in the named character vector `Ricklefs_colors` below:

``` r
colors # hexadecimal color codes extracted from PostScript
```

    ## [1] "#A09700" "#DCBB50" "#75A95E" "#317A22" "#D16E3F" "#C1E1DD" "#A5C790"
    ## [8] "#FCD57A" "#97B669"

``` r
Ricklefs_colors <- colors
names(Ricklefs_colors) <- unique(Whittaker_biomes$biome)
# Sets a desired order for the names. This will affect the order in legend.
biomes_order <- c("Tundra",
                  "Boreal forest",
                  "Temperate seasonal forest",
                  "Temperate rain forest",
                  "Tropical rain forest",
                  "Tropical seasonal forest/savanna",
                  "Subtropical desert",
                  "Temperate grassland/desert",
                  "Woodland/shrubland")
Ricklefs_colors <- Ricklefs_colors[order(factor(names(Ricklefs_colors), levels = biomes_order))]
Ricklefs_colors
```

    ##                           Tundra                    Boreal forest 
    ##                        "#C1E1DD"                        "#A5C790" 
    ##        Temperate seasonal forest            Temperate rain forest 
    ##                        "#97B669"                        "#75A95E" 
    ##             Tropical rain forest Tropical seasonal forest/savanna 
    ##                        "#317A22"                        "#A09700" 
    ##               Subtropical desert       Temperate grassland/desert 
    ##                        "#DCBB50"                        "#FCD57A" 
    ##               Woodland/shrubland 
    ##                        "#D16E3F"

Plotting examples
=================

Now, with the constructed data frame and the color vector one can go on plotting with `ggplot2`. Check examples [here](https://rawgit.com/valentinitnelav/plotbiomes/master/html/Whittaker_biomes_examples.html).