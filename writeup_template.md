## Advanced Lane Finding Writeup

---

**Advanced Lane Finding Project**

The goals / steps of this project are the following:

* Compute the camera calibration matrix and distortion coefficients given a set of chessboard images.
* Apply a distortion correction to raw images.
* Use color transforms, gradients, etc., to create a thresholded binary image.
* Apply a perspective transform to rectify binary image ("birds-eye view").
* Detect lane pixels and fit to find the lane boundary.
* Determine the curvature of the lane and vehicle position with respect to center.
* Warp the detected lane boundaries back onto the original image.
* Output visual display of the lane boundaries and numerical estimation of lane curvature and vehicle position.

[//]: # (Image References)

[image1]: ./examples/chess_undistort.png "Undistorted"
[image1a]: ./examples/chess_warped.png "Chessboard Warped"
[image2]: ./test_images/road_undist.png "Road Transformed"
[image3]: ./examples/road_bin_warped.png "Binary Example"
[image4]: ./examples/road_warped.png "Warp Example"
[image5]: ./examples/color_fit_lines.jpg "Fit Visual"
[image6]: ./examples/static_img_pipeline_test.png "Output"
[video1]: ./project_output.mp4 "Video"

## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points

### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---

### Writeup / README

#### 1. Provide a Writeup / README that includes all the rubric points and how you addressed each one.  You can submit your writeup as markdown or pdf.  [Here](https://github.com/udacity/CarND-Advanced-Lane-Lines/blob/master/writeup_template.md) is a template writeup for this project you can use as a guide and a starting point.  

You're reading it!

### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The code for this step strats in the thrid code cell of the IPython notebook P2.ipynb. 

I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.  

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  I applied this distortion correction to the test image using the `cv2.undistort()` function and obtained this result: 

![alt text][image1]

I further applied perspective transform to display the board. To do this, I used 4 corners of the board as source and then picked 4 points on a plane with some offset from the image corners. Then I computed the perspective transform matrix using `cv2.getPerspectiveTransform(src, dst)`. After applying transformation with `cv2.warpPerspective` the resulting image looks like this:

![alt_text][image1a]

### Pipeline (single images)

#### 1. Provide an example of a distortion-corrected image.

To demonstrate this step, I will describe how I apply the distortion correction to one of the test images like this one:
![alt text][image2]

#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

I used a combination of Sobel derivarives and gradient thresholds to generate a binary image in cell #14 of the P2.ipynb notebook. The resulting image is shown below:

![alt text][image3]

#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

I'm calculating persective transform matrices in cell #11 using `cv2.getPerspectiveTransform`.
I chose the hardcode the source and destination points in the following manner:

```python
imshape = rd_und.shape
offset = 300 # offset for dst points
src = np.float32([(imshape[1]*0.45, imshape[0]*0.64),
                  (imshape[1]*0.55, imshape[0]*0.64),
                  (imshape[1]*0.85,imshape[0]),
                  (imshape[1]*0.15,imshape[0])])

dst = np.float32([[offset, 0],
                  [imshape[1]-offset, 0], 
                  [imshape[1]-offset, imshape[0]], 
                  [offset, imshape[0]]])
```

I am using the same function again to create M_reverse matrix which would recreate original perspective image from the warped.

I verified that my perspective transform was working as expected by drawing the `src` and `dst` points onto a test image and its warped counterpart to verify that the lines appear parallel in the warped image.

![alt text][image4]

#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

I created a function `find_lane_pixels` (cell #16) which takes a binary image and fits two polynomials for right and left lines of the lane. It employes window search and is used when line locations are unknown (i.e. for the very first image of the video, static image or when lane is lost in the video).
Window search creates a histogram for each window, finds the peak which corresponds to the highest number of white pixels and creates a `(x,y)` point. Further these coordinates are passed into `fit_polynomial` function which calculates 2nd degree polynomial coefficients of the best fitting curve using `np.polyfit()`.
For sequential frames I use another function called `search_around_poly` (cell #18) which receives a binary image as an input and polynomial coefficients. This function searches for non-zero pixels within certain margin (hyperparam, used 100 pix) from the given polynomial curve. Once all non-zero indices are found, they are being passed into `fit_poly` function which fit another polynomial.

![alt text][image5]

#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

Curvature radius is being calculated as part of `draw_lines` function:
```python
left_curverad = ((1 + (2*left_fit[0]*y_eval + left_fit[1])**2)**1.5) / np.absolute(2*left_fit[0])
right_curverad = ((1 + (2*right_fit[0]*y_eval + right_fit[1])**2)**1.5) / np.absolute(2*right_fit[0])
```
where left_fit and right_fit are polynomial coefficients and y_eval is the max value of Y axis (bottom of the image)

For the purpose of the video output I'm calculating average of the two values.

#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

I implemented the entire pipeline in function `process`, cell #26. It instantiates left and right lines from class `Line()`, takes image as an input. The result is shown below:

![alt text][image6]

---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my video result](./project_video.mp4)

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

The project video worked reasonably well, especially after smoothing the curve coefficients over last 10 frames of the video. However, the algorithm gets confused on the challenge video and the harder challenge as well (completely unreasonable).
I think I should further improve initial image processing to better catch white and yellow pixels. The things that caused problems in challenge videos are shadows and edges on the road which most likely were picked by the gradient.
The harder challenge video had a very twisty road so the algorithm had a hard time catching up with changes. Reducing frame smoothing number did not help.
