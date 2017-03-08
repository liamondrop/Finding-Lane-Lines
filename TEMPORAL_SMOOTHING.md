# **Temporal Smoothing to Remove Jitter in Detected Lane Lines**

### Averaging a time series to achieve smoother results

Project 1 of Udacity's Self-Driving Car Nanodegree is a computer vision problem challenging the student to construct an image processing pipeline that detects lane lines in a stream of images and overlays a solid line closely matching the lanes. I won't exhaustively detail the entire pipeline here, but check out [this repository](https://github.com/liamondrop/finding-lane-lines) if you'd like to see the full pipeline.

In broad strokes, the basic steps to achieve this are as follows:
  1. Convert each image to the HSV color space and mask out any colors not within a range of colors sampled from the yellow and white lane lines.
  1. Convert each image to grayscale and apply a Gaussian blur to reduce some of the image noise
  1. Use Canny edge detection to reduce the grayscale image to its edge outlines
  1. Simplify the image further by masking out any regions of the image that aren't likely to contain lane lines
  1. Apply a Hough Transform to the edges, which will return a collection of line segments wherever lines are detected
  1. Partition these line segments into left and right batches
  1. For each batch, fit a single straight line to the segments

You can see the results of my initial pipeline in this video:

::: INITIAL PIPELINE VIDEO :::

While this was a drastically improved result, there was still room for improvement:
  - The lines were still a bit jittery, and even dropped out of the video completely for a few frames when shadows occluded the road. If our theoretical self-driving car were using this data to keep itself centered within the lane, how well would it fare if the lines were jittering or even periodically disappearing?
  - Aesthetically, I didn't like the way the drawn lines ended at varying heights. This was due to the fact that the upper X value of each line was a constant number of pixels away from the center of the image, leaving the upper Y values to vary independently. If instead I calculated the intersection of the lines and ended them at that point, they should create a much more coherent shape.

To solve the first issue, I created a list to store the slope and y-intercept values for each line detected in the last N frames. The actual line drawn on the current frame is simply the average slope/intercept of all these lines. By continuously pushing the latest detected line onto this buffer and simultaneously dropping the oldest line, I can calculate a rolling mean of the lines over time, or what I call "temporal smoothing". Here's what that looks like in code:

```py
>>> import numpy
>>>
>>> # array of slope and y-intercept values for a series of lines
>>> buffer = [[.5, 100], [.45, 102], [.55, 95]]
>>> buffer_size = 3
>>>
>>> def add_line_to_buffer(line):
...     buffer.append(line)
...     return buffer[-buffer_size:]
...
>>> # append the current line and slice off the oldest (leftmost) line
>>> line = [.57, 104]
>>> add_line_to_buffer(line)
[[0.45, 102], [0.55, 95], [0.57, 104]]

>>> # numpy mean with axis=0 will give the average slope and
>>> # y-intercept values for all lines currently stored in the buffer
>>> # this will be what is drawn on the current frame
>>> numpy.mean(buffer, axis=0)
array([   0.5175,  100.25  ])

```

Finding the intersection of the lines is a simple geometry problem, made even simpler by using Numpy's linear equation solver. Remember that the equation of a line is `f(x) = ax + c`, where `a` is the slope and `c` is the y-intercept. By mapping two lines with differing slopes into [homogenous coordinates](), we can solve for x and y as follows.

```py
def get_line_intersection(line1, line2):
    slope1, intercept1 = line1
    slope2, intercept2 = line2

    # put the coordinates into homogeneous form
    a = [[slope1],
         [slope2]]
         
    b = [-intercept1, -intercept2]
    x, y = numpy.linalg.solve(a, b)
    return x, y

line1 = [0.5, 2]
line2 = [-0.75, 7]

get_line_intersection(line1, line2)
```
