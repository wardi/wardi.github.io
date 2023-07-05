---
layout: page
permalink: /flood-fill-mazes/
title: Flood Fill Mazes
categories: [Python]
image: /images/maze4.png
description:
  We generate mazes using the scikit-image flood_fill function
excerpt_separator: <!--more-->
---

<img src="/images/maze-bucket.png" alt="paint bucket tool pouring a maze pattern">

We generate mazes using the scikit-image `flood_fill` function.

<!--more-->

<style>
.language-plaintext .highlight {background: black}
details > img {text-align: center; display: block; margin: 0 auto;}
</style>

We'll need jupyter and a few common Python libraries:

```bash
$ pip install jupyter matplotlib scikit-image
```

Launch jupyter notebooks and follow along:

* [Flood fill maze generation jypyter notebook](https://github.com/wardi/cpu/blob/main/maze/presentation.ipynb)

The examples below will use numpy for efficient array operations,
matplotlib for visualization, and scikit-image for image
manipulation. Imports are at the the top of the project, as usual.
If you get any errors running these imports make sure you have
the dependencies above installed:

```python
import numpy as np
import matplotlib.pyplot as plt
import matplotlib.patches as mpatches
import matplotlib as mpl
from skimage.segmentation import flood_fill
from skimage.io import imread, imsave
from skimage.color import rgb2gray
from skimage.transform import resize
from skimage.filters import threshold_local
```

## Square one

Our maze needs a border, a starting location, and an ending location.
Let's represent our maze with a numpy array where the border is marked with 1's,
the starting and ending locations are marked with 2's and everything else
is set to 0.

This function will generate such an array with any width and height. We follow
numpy's (y, x) convention for coordinates, so height comes first:

```python
def box(height, width):
    arr = np.zeros((height, width), dtype=np.uint8)
    # walls top, bottom, left, right = 1
    arr[0] = arr[-1] = arr[..., 0] = arr[..., -1] = 1
    # start and end locations = 2
    arr[0, 1] = arr[-1, -2] = 2
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

Let's visualize our array with matplotlib.
["inferno"](https://matplotlib.org/stable/tutorials/colors/colormaps.html#sequential)
is a nice a bright palette. We create a legend with color patches, and draw
arrays with this "show" function:

```python
palette = mpl.cm.inferno.resampled(3).colors
labels = ["0: unfilled", "1: wall", "2: passage"]

def show(arr):
    plt.figure(figsize=(9, 9))
    im = plt.imshow(palette[arr])
    # create a legend on the side
    patches = [mpatches.Patch(color=c, label=l) for c, l in zip(palette, labels)]
    plt.legend(handles=patches, bbox_to_anchor=(1.1, 1), loc=2, borderaxespad=0)
    plt.show()

show(b)
```

<img src="/images/maze-box1.png" alt="6x6 box with unfilled, wall, passage legend">

Very nice! We can see the starting and ending locations (passages) at
(y=0, x=1) and (y=5, x=4).

## Lay of the land

Instead of hard-coding them we can find the starting and ending locations
using numpy's `array.where` method:

```python
np.where(b == 2)
```

```
(array([0, 5]), array([1, 4]))
```

This gives us arrays with the y coordinates and the x coordinates of each location.
For our program we need the y and x values for just one of these locations:

```python
tuple(coord[0] for coord in np.where(b == 2))
```

```
(0, 1)
```

Next we collect the unfilled locations in the box (the middle part):

```python
np.where(b == 0)
```

```
(array([1, 1, 1, 1, 2, 2, 2, 2, 3, 3, 3, 3, 4, 4, 4, 4]),
 array([1, 2, 3, 4, 1, 2, 3, 4, 1, 2, 3, 4, 1, 2, 3, 4]))
```

Again we want the (y, x) coordinates not an array of y's and an array of x's
so let's `swapaxes`:

```python
np.swapaxes(np.where(b == 0), 0, 1)
```

```
array([[1, 1],
       [1, 2],
       [1, 3],
       [1, 4],
       [2, 1],
       [2, 2],
       [2, 3],
       [2, 4],
       [3, 1],
       [3, 2],
       [3, 3],
       [3, 4],
       [4, 1],
       [4, 2],
       [4, 3],
       [4, 4]])
```

These are the coordinates that our program will turn into either walls or passages.

We're going to need a bigger box for the maze, with many more coordinates to fill:

```python
a = box(30, 30)
show(a)
```

<img src="/images/maze-box2.png" alt="30x30 box with unfilled, wall, passage legend">

## The algorithm

Here's our first `flood_fill` maze generation algorithm "maze1":

1. take all the unfilled coordinates and shuffle their order
2. save the starting coordinates (anything marked as a passage)
3. set all the passages to unfilled so the whole array is 0's or 1's.
4. for each of the unfilled coordinates set it to a 1 (wall)
   then try flood filling the maze from the starting coordinates with 2's
   (passage)
   - if any part of the maze remains a 0 (unfilled) then this wall has divided
   the maze making some part of it unreachable, so set the wall back to unfilled.

```python
def maze1(arr):
    unfilled = np.swapaxes(np.where(arr == 0), 0, 1)
    np.random.shuffle(unfilled)

    start = tuple(coord[0] for coord in np.where(b == 2))
    arr = np.copy(arr)
    arr[arr == 2] = 0

    for lc in unfilled:
        lc = tuple(lc)
        arr[lc] = 1
        t = flood_fill(arr, start, 1)
        if np.any(t == 0):
            arr[lc] = 0

    arr[arr == 0] = 2
    return arr

show(maze1(a))
```

<img src="/images/maze1.png" alt="maze generated with maze1 algorithm">

So that's something. It is a maze, but I wasn't expecting the maze to allow moving diagonally.
Also it's very sparse (mostly walls) and easy to solve.

The first problem is easy to fix. `flood_fill` has a `connectivity` parameter that sets
the disance each cell can be from the next while allowing the fill to continue.

Here's "maze2" like "maze1" above but with `connectivity=1`:

```python
def maze2(arr):
    unfilled = np.swapaxes(np.where(arr == 0), 0, 1)
    np.random.shuffle(unfilled)

    start = tuple(coord[0] for coord in np.where(b == 2))
    arr = np.copy(arr)
    arr[arr == 2] = 0

    for lc in unfilled:
        lc = tuple(lc)
        arr[lc] = 1
        t = flood_fill(arr, start, 1, connectivity=1)  # no diagonals, please
        if np.any(t == 0):
            arr[lc] = 0

    arr[arr == 0] = 2
    return arr

show(maze2(a))
```

<img src="/images/maze2.png" alt="maze generated with maze2 algorithm">

Better, but the maze is still very easy to solve.

Our algorithm sets every unfilled cell to a wall as long as that wall
doesn't cause the maze to be divided. So our walls end up very
thick because any time a dead-end (a cell with three walls surrounding it)
is selected it will be turned into a wall as well.

Let's protect our dead-ends to generate more complex mazes with "maze3":

```python
# mask for cells above, below, left and right
neighbors = np.array([
    [0, 1, 0],
    [1, 0, 1],
    [0, 1, 0]], dtype=np.uint8)

def maze3(arr):
    unfilled = np.swapaxes(np.where(arr == 0), 0, 1)
    np.random.shuffle(unfilled)

    start = tuple(coord[0] for coord in np.where(b == 2))
    arr = np.copy(arr)
    arr[arr == 2] = 0

    for lc in unfilled:
        lc = tuple(lc)

        y, x = lc
        # protect dead-ends from becoming walls
        if np.sum(neighbors * arr[y-1:y+2, x-1:x+2]) > 2:
            continue

        arr[lc] = 1
        t = flood_fill(arr, start, 1, connectivity=1)
        if np.any(t == 0):
            arr[lc] = 0

    arr[arr == 0] = 2
    return arr

m = maze3(a)
show(m)
```

<img src="/images/maze3.png" alt="maze generated with maze3 algorithm">

That's more like it.

This maze has start and end positions on the edge so we can discover the
path from start to end by flood-filling the walls one side and tracing
the border between the two different colors:

```python
# spoiler
t = flood_fill(m, (0, 0), 0)
show(t)
```

<details><summary>Spoiler</summary>
<img src="/images/maze3-solved.png" alt="solution to maze generated with maze3 algorithm">
</details>

## A "normal" maze

To generate mazes that look more like standard grid mazes we can start with a
pattern of alternating passages and walls:

```python
c = box(31, 31)
c[::2, ::2] = 1  # walls on even (y, x) values
c[1::2, 1::2] = 2  # passages on odd (y, x) values
show(c)
```

<img src="/images/maze-grid-pattern.png" alt="alternating grid of passages and walls">

```python
sm = maze3(c)
show(sm)
```

<img src="/images/maze-grid.png" alt="generated standard grid maze">

The same algorithm now produces a common grid maze, but we don't have to stop there.

## Big brain solution

Let's take inspiration from nature and generate a maze from a brain coral image:

<img src="/images/brain-coral.jpg" alt="brain coral">

We load the image with `imread` then crop a small part and convert it to
grayscale using `rgb2gray`:

```python
brain = imread('brain-coral.jpg')
crop_brain = rgb2gray(brain[-500:,200:1000])
plt.figure(figsize=(15, 9))
im = plt.imshow(crop_brain, plt.cm.gray)
cb = plt.colorbar()
```

<img src="/images/brain-coral-gray.png" alt="brain coral cropped in grayscale">

Next reduce the resolution with `resize` and use `threshold_local` to convert
grayscale to 1s and 0s:

```python
small_brain = resize(crop_brain, (50, 80))
binary_brain = small_brain > threshold_local(small_brain, 15, 'mean')
plt.figure(figsize=(9, 9))
im = plt.imshow(binary_brain, plt.cm.gray)
```

<img src="/images/brain-coral-bw.png" alt="brain coral low resolution black and white">

Then wrap the image with a box and turn the 0s into passages and the 1s into
unfilled using modular arithmatic:

```python
d = box(52, 82)
insert_brain = (binary_brain + 2) % 3
d[1:-1, 1:-1] = insert_brain
show(d)
```

<img src="/images/brain-coral-pattern.png" alt="brain coral box pattern">

When we generate a maze with this pattern the walls can only be placed in the
unfilled areas so our maze will follow the shape of the brain coral:

```python
bm = maze2(d)  # maze2 is better: don't need lots of extra dead-ends
show(bm)
```

<img src="/images/brain-coral-maze.png" alt="brain coral generated maze">

Here's the solution:

```python
# spoiler
t = flood_fill(bm, (0,0), 0)
show(t)
```

<details><summary>Spoiler</summary>
<img src="/images/brain-coral-maze-solved.png" alt="solution to the brain coral maze">
</details>

## Why don't we have both?

Drawing patterns for the algorithm to fill lets us create all sorts of mazes.
If we save the grid and brain patterns with `imsave` we can use an image editor
to combine them any way we like:

```python
imsave('grid_pattern.png', palette[c])
imsave('brain_pattern.png', palette[d])

# use these to create combined_pattern.png with an image editing program
```

We can stitch patterns together and draw walls or passages through the image
to shape the generated maze solution to any style and difficulty.

```python
n = imread('combined_pattern.png')
# use red component to convert RGB to 0:unfilled, 1: wall, 2: passage
n = np.array((n[...,0] == 188) + (n[...,0] == 252) * 2, dtype=np.uint8)
show(n)
```

<img src="/images/maze-combined-pattern.png" alt="custom maze pattern">

```python
mn = maze3(n)
show(mn)
```

<img src="/images/maze-combined.png" alt="custom generated maze">

The solution follows our wandering path up and down through the different
maze styles:

```python
# spoiler
t = flood_fill(mn, (0,0), 0)
show(t)
```

<details><summary>Spoiler</summary>
<img src="/images/maze-combined-solved.png" alt="solution to the custom generated maze">
</details>
