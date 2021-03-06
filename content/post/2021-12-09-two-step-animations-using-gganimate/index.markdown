---
title: "Bake-at-home animations: render individual frames with {gganimate} and animate later"
author: Jan Savinc
date: '2021-12-10'
slug: [bake-at-home-animations]
categories: []
tags: []
description: 'Rendering individual frames in {gganimate} and animating them in a separate step later'
draft: no
topics: []
---



I recently ran into an issue creating animated plots using {gganimate} in the [National Safe Haven environment](https://www.isdscotland.org/products-and-services/edris/use-of-the-national-safe-haven/) I use for research at [SCADR](https://www.scadr.ac.uk/), where the resulting files were either broken or could not be displayed, and I figured it was safer to produce images that definitely worked and use them to build an animation on a different computer. I think of this as *bake-at-home animations* (as in those partly-baked baguettes you're meant to finish baking at home).


This workaround involves creating animations in two steps:

1. Render the {gganimate}-created animation frames as separate images
2. Build an animated `.gif` or `.mp4` file from the individual frames

(in some situations I think a video file like `.mp4` works better, because you can pause the animation and seek forwards or backwards to look at individual steps) 


# Packages used

* [{ggplot2}](https://ggplot2.tidyverse.org/) for making plots
* [{gganimate}](https://gganimate.com/) for creating animations from {ggplot2} plots
* [{gifski}](https://github.com/r-rust/gifski) for saving a `.gif` from a set of input `.png` files
* [{av}](https://github.com/ropensci/av) for saving an `.mp4` video (other formats are supported!) from a set of input `.png` files
* [{png}](http://www.rforge.net/png/) to load a `.png` file and read the dimensions (width & height)


All of the above were available on CRAN as of the time of writing.

# Step 0: Making an animation

I made up a quick plot based on examples from the [{gganimate}](https://gganimate.com/) package:


```r
library(ggplot2)
library(gganimate)

animated_plot <- 
  ggplot(mtcars, aes(x = wt, y = hp, colour = as.factor(cyl))) +
  geom_point() +
  transition_states(cyl, transition_length = 3, state_length = 1) +
  enter_fade() +
  exit_fade() +
  labs(title = "Cyl: {closest_state}")
```

# Step 1: rendering individual frames

For this step, we use the `animate()` function and specify the `file_renderer()` function to save the individual frames as separate images. 

Because we'll be building an animation from these frames later, we need to design for the length of the animation required and the number of frames; {gganimate} can do some neat stuff like rendering smooth transitions between plot elements, and the more frames you use, the smoother the animation will be (at the cost of bigger file size). You will want to experiment with the values a bit depending on your plot.

`animate()` has default arguments `fps` (10 frames per second) and `nframes` (100), and can also take a `duration` argument (10 seconds when using the defaults, nframes/fps). By specifying two of the three arguments above we can modify how many frames we end up with and  how long the animation will be. I suggest you just change `fps` and `duration`, and let `animate()` calculate `nframes` - this is because in the next step, to build a `.gif` file from frames, you'll need to specify how long each frame should be displayed for (1/`fps`), or to build an `.mp4` file, you'll need to specify the `fps`.

To change the size of the image, you can pass the `width`,`height`,`units`, and `res` (resolution) parameters to `animate()` like you would for `ggsave()`.

The default image format is `.png`, which should be fine in most cases!


```r
duration = 10  # set up duration & fps to suit your needs
fps = 10

animate(
  plot = animated_plot,
  # width = X,  # you can specify width, height, units and resolution like in ggsave()
  # height = Y,
  # units = "cm",
  # res = 300,
  fps = fps,
  duration = duration,
  renderer = file_renderer(
    dir = "directory_for_individual_frames",  # a directory where the frames will go
    prefix = "frames_for_animation",  # frames will be named "frames_for_animation0001.png" and so on
    overwrite = TRUE  # I like this to be able to repeatedly regenerate the animations as I test settings
  )
)
```

# Step 2: building animation from individual frames

For this step, I'm using the {gifski} package to make a `.gif` file, and {av} to make an `.mp4` video. Here we'll be reusing the `fps` and `duration` parameters from step 1 above.


```r
## load the paths to the individual frames

frames <- 
  list.files(
  "directory_for_individual_frames",  # whatever dir your individual frames are in
  pattern = "png", # I've  added the "png" filter on files in case you've put anything else in there
  full.names = TRUE  # this is required to provide absolute paths
  )
```

## `.gif`

Let's build the `.gif` first, which is slightly complex if you intend the `.gif` to preserve the dimensions of the individual frames - if not, you can just set the `width` & `height` parameters in `gifski()` to whatever you need.


```r
library(gifski)

## work out the dimensions (in pixels) of the png files

image_data <- png::readPNG(source = frames[1])  # read first frame
image_width <- dim(image_data)[1]
image_height <- dim(image_data)[2]
rm(image_data)  # remove from memory, no longer needed

gifski(
  png_files = frames
  # I've also added the "png" filter on files just in case
  gif_file = "animation.gif"),
  width = image_width,
  height = image_height,
  delay = 1/fps  # alternatively if you've specified nframes and duration this would be `duration/nframes`
)
```

Note that the use of `png::readPNG()` above isn't optimal, since it loads the entire image into memory, resulting in a rather large object of dimensions width X height X colour channels; this is overkill, but I couldn't find any packages that would just read the image metadata. On most modern computers and with reasonably sized plots this shouldn't be too much of an issue (I hope!).


## `.mp4`

Making an `.mp4` video is simpler in comparison:


```r
library(av)
av_encode_video(
  input = frames,
  framerate = fps,  # alternatively, nframes/duration
  output = "animation.mp4"
)
```

# Conclusion

You should now have a directory of individual frames, and a `.gif` and/or `.mp4` file. As is often the case, I ended up spending most of my time trying to work out how to preserve the image dimensions between the original files and the animations, a detail I only noticed when the resulting animations were a lot smaller than animations saved using `gganimate::save_anim()` directly. Another thing I've learned from this is that the syntax for `animate()` and `file_renderer()` isn't very straightforward and it helps to go through the documentation to see which arguments go where!
