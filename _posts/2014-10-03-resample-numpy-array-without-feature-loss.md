---
layout: post
title: Resample Numpy Array without Feature Loss
categories: [numpy, Python]
redirect_from: /article/2014/10/resample-numpy-array-without-feature-loss/
excerpt_separator: <!--more-->
---


For pyrf I needed to take data from a frequency plot, which could be any number of points, and present it as a spectrogram that fills the view size exactly. In the spectrogram I only care about the maximum values that appear in the range of frequencies represented by each pixel.

If I could just divide the number of source bins by an integer factor the solution would be simple:

```python
return np.amax(data.reshape((-1, factor)), axis=1)
```

But I have to be able to handle any number of source bins and output that to any number of pixels.

Fortunately numpy is awesome.

<!--more-->

I can use a numpy array to index another numpy array. In particular, I can get the first bin that applies to each of my output pixels with linspace:

```python
bins = len(data)
indexes = np.linspace(0, bins, pixels, endpoint=False, dtype=int)
return data[indexes]
```

Truncation to ints causes the indexes to be biased to the left. We can add 0.5 to fix that:

```python
bins = len(data)
indexes = np.linspace(0.5, bins + 0.5, pixels, endpoint=False, dtype=int)
return data[indexes]
```

Our indexes arrays elements now differ by bins // pixels or bins // pixels + 1. To cover all the bins except the + 1 we can create a matrix of index arrays and compute the maximums:

```python
bins = len(data)
indexes = np.linspace(0.5, bins + 0.5, pixels, endpoint=False, dtype=int)
index_matrix = [indexes + i for i in range(bins // pixels))]
# FIXME: skipping some of the source data!
return np.amax(data[np.vstack(index_matrix)], axis=0)
```

To catch the + 1 elements we can subtract one from the following starting indexes, since it doesn't matter if we include a number we already included once in a max calculation. This works for resampling arrays from any size to any size without losing any peaks:

```python
bins = len(data)
indexes = np.linspace(0.5, bins + 0.5, pixels, endpoint=False, dtype=int)
index_matrix = [indexes + i for i in range(bins // pixels))]
index_matrix.append(np.concatenate((indexes[1:], [bins])) - 1)
return np.amax(data[np.vstack(index_matrix)], axis=0)
```

But wait, sometimes the user resizes the window and we need to handle resampling a whole matrix instead of a single row.

Fortunately numpy is awesome again.

The ellipsis operator can be used to represent any number of dimensions in a matrix, so I don't have to loop over the rows, I can just write "..." and change the way I get the number of bins and now my code works for arrays and matrices:

```python
bins = data.shape[-1]
indexes = np.linspace(0.5, bins + 0.5, pixels, endpoint=False, dtype=int)
index_matrix = [indexes + i for i in range(bins // pixels))]
index_matrix.append(np.concatenate((indexes[1:], [bins])) - 1)
return np.amax(..., data[np.vstack(index_matrix)], axis=len(data.shape) - 1)
```
