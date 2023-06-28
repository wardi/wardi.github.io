---
layout: post
title: Flood Fill Maze Generation
categories: [Python]
image: /images/maze4.png
description:
  We generate mazes using the scikit-image flood_fill function
excerpt_separator: <!--more-->
---

<img src="/images/maze4.png" alt="Maze generated with flood_fill">

We generate mazes using the scikit-image `flood_fill` function.

<!--more-->

First we'll need a few common Python libraries:

```bash
$ pip install jupyter matplotlib scikit-image
```

* [Flood fill maze generation jypyter notebook](https://github.com/wardi/cpu/blob/main/maze/presentation.ipynb)

The examples below will use numpy for fast array operatios,
matplotlib for visualization, and scikit-image for image
manipulation. All the imports are gathered in one place here:

```python
import numpy as np
import matplotlib.pyplot as plt
import matplotlib.patches as mpatches
import matplotlib as mpl
from skimage.segmentation import flood_fill
from skimage.io import imread
from skimage.color import rgb2gray
from skimage.transform import resize
from skimage.filters import threshold_local
```

## Starting with a box

Our maze needs a border, a starting location, and an ending location.
Let's represent that with a numpy array where the border is marked with `1`s,
the starting and ending locations are marked with `2`s and everything else
is set to `0`.

Here's a function to generate this box with any width and height:

```python
def box(height, width):
    arr = np.zeros((height, width), dtype=np.uint8)
    # walls
    arr[0] = 1
    arr[-1] = 1
    arr[..., 0] = 1
    arr[..., -1] = 1
    arr[0, 1] = 2
    arr[-1, -2] = 2
    return arr

b = box(6, 6)
b
```

```
array([[1, 2, 1, 1, 1, 1],
       [1, 0, 0, 0, 0, 1],
       [1, 0, 0, 0, 0, 1],
       [1, 0, 0, 0, 0, 1],
       [1, 0, 0, 0, 0, 1],
       [1, 1, 1, 1, 2, 1]], dtype=uint8)
```

Matplotlib can display this array with colors and a legend for reference.
We choose a bright palette, create a legend with color patches, and draw
an array passed with our `show` function:

```python
palette = mpl.cm.inferno.resampled(3).colors
labels = ["0: unfilled", "1: wall", "2: passage"]

def show(arr):
    plt.figure(figsize=(9, 9))
    im = plt.imshow(palette[arr])
    patches = [mpatches.Patch(color=c, label=l) for c, l in zip(palette, labels)]
    plt.legend(handles=patches, bbox_to_anchor=(1.1, 1), loc=2, borderaxespad=0)
    plt.show()

show(b)
```

<img src="/images/maze_box1.png" alt="box with legend">

Let's find the starting and ending locations using numpy's `array.where` method:

```python
np.where(b == 2)
```

```
(array([0, 5]), array([1, 4]))
```

This gives us arrays with the y coordinates and the x coordinates of each location,
but we need the y and x values for just one of these locations.

`zip` generates tuples with one item from each argument and `next`
takes the first tuple:

```python
start = next(zip(*np.where(b == 2)))
start
```

```
(0, 1)
```

Next we collect the unfilled locations in the box:

```python
np.where(b == 0)
```

```
(array([1, 1, 1, 1, 2, 2, 2, 2, 3, 3, 3, 3, 4, 4, 4, 4]),
 array([1, 2, 3, 4, 1, 2, 3, 4, 1, 2, 3, 4, 1, 2, 3, 4]))
```
