---
title: "Animating two images transitioning horizontally with Python"
publish: true
date: 2021-11-01T00:00:00+02:00
---

Hi, I'll start with showing what I'm trying to achieve (I couldn't come up with a shorter title).

![[f7d3e6d83207e6266b22bc2ad88940b4_MD5.gif]]

We have two images, the second one slowly replaces the first (horizontally).

To do this we'll need to prepare two images with the same shape (I won't show how to scale them in python).

> This method can be used to make any number of images transition (with a little bit of work).

## Loading images

First things first. Let's load two images.

```python
# We'll need them later
import numpy as np
import matplotlib.pylab as plt

from PIL import Image

image1 = np.array(Image.open('image1.png').convert('RGB'))
image2 = np.array(Image.open('image2.png').convert('RGB'))
```

In my case, this is the first image:
![[9172ad33fc07aa19feb44eebba45cfa3_MD5.png]]

And the second:
![[f3e6ae1586b6db7b27a54809700d957d_MD5.png]]

Both images are of shape: (339, 339, 3)

## Generating frames

Firstly, let's flatten our images, we'll take advantage of that later.

```python
fimage1 = image1.reshape((-1,3))
fimage2 = image2.reshape((-1,3))
```

To create animation we'll need to know how each frame will look. In my case I wanted to define a frame by a given part (`perc`) of pixels of the second image. For example if `perc=0.1` then first 0.1 pixels are from second image, and 0.9 are from first. `Fade` is a function that will do that. Given two flattened images, the original shape of the image and perc, it will concatenate the two images and return the image to the original shape.

```python
def fade(shape, fimage1, fimage2, perc):
    i = int(fimage1.shape[0] * perc)
    return np.concatenate((fimage2[:i], fimage1[i:, :])).reshape(shape)
```

Where:

- shape: tuple, shape of the original image
- fimage1: array, image flattened to shape (-1, 3)
- fimage2: same as fimage2
- perc: float, part of image2 to show

Here are some images that I generated with it:

![[17423efe0f5a7d0386c06a3c301f20ad_MD5.png]]

## Creating animation

I won't go into details on how to create animations in Matplotlib, here's my code.

```python
fig = plt.figure(figsize=(12, 10.8)) # Depends on aspect of your images
ax = plt.axes()
pic = ax.imshow(np.zeros(image1.shape)) # Create empty image of the same shape as image to plot
frames = 150 # Number of frames to generate

def init():
    pic.set_array(np.zeros(image1.shape))
    return [pic]

# This funtion generates i-th frame.
def animate(i):
    pic.set_array(fade(image1.shape, fimage1, fimage2, i/frames))
    return [pic]

anim = animation.FuncAnimation(fig, animate, init_func=init,
                               frames=frames, blit=True)

anim.save('animaton.mp4', fps=60, extra_args=['-vcodec', 'libx264'])

plt.show()
```

The file `animation.mp4` should generate, depending on number of frames.
