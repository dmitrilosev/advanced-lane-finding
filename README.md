
## Writeup

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

[image1]: ./output_images/undistort_output.png "Undistorted"
[image2]: ./output_images/test1.png "Road Transformed"
[image3_1]: ./output_images/thresholder_gradx.png "Gradx"
[image3_2]: ./output_images/thresholder_grady.png "Grady"
[image3_3]: ./output_images/thresholder_magnitude.png "Magnitude"
[image3_4]: ./output_images/thresholder_combined.png "Combined"
[image3_5]: ./output_images/thresholder_color.png "Color"
[image3_6]: ./output_images/thresholder_stack.png "Stack"
[image3_7]: ./output_images/thresholder_finally_combined.png "Finally combined"
[image4]: ./output_images/warped_straight_lines.png "Warp Example"
[image5]: ./output_images/color_fit_lines.png "Fit Visual"
[image6]: ./output_images/example_output.png "Output"
[video1]: ./project_video.mp4 "Video"

---
### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The code for this step is contained in the first code cell of the [IPython notebook](./notebook.ipynb). 

I started by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.

I created **Camera** class with incapsulated **_findCorners** function and following interface functions:

* **calibrate** I used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.
* **undistort** I applied this distortion correction to the test image using the `cv2.undistort()` function
* **warp** Warp the image using OpenCV warpPerspective()
* **bird_view** Transforms an image to birds view

I obtained this result after camera calibrating and test image undistortion: 

![alt text][image1]

### Pipeline (single images)

#### 1. Provide an example of a distortion-corrected image.
To demonstrate this step, I will describe how I apply the distortion correction to one of the test images like this one:
![alt text][image2]

#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

I used a combination of color and gradient thresholds to generate a binary image (thresholding steps in **Thresholder** class, second code cell of the [IPython notebook](./notebook.ipynb)).  Here's an example of my output for this step.

Gradx
![alt text][image3_1]
Grady
![alt text][image3_2]
Magnitude
![alt text][image3_3]
Combined
![alt text][image3_4]
Color
![alt text][image3_5]
Stack
![alt text][image3_6]
Finally combined
![alt text][image3_7]

#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

The code for my perspective transform is a part of `Camera` class that includes two functions called `bird_view(self, image)`(cell 1, lines 69 through 84 in the [IPython notebook](./notebook.ipynb)) and `_warp(self, image, src, dst)`(cell 1, lines 56 through 64 in the [IPython notebook](./notebook.ipynb)). The `bird_view(self, image)` function takes as inputs an image (`image`), calculates source and destination coordinates, then calls `_warp(self, image, src, dst)` and returns perspectively transformed image. I chose the hardcode the source and destination points in the following manner:

```
h = image.shape[0]
w = image.shape[1]
hk = 3.165/4
wk = 3/4
# apply warp transform
src = np.float32([[int(w*(1-wk)), int(h*hk)], 
                  [int(w*wk), int(h*hk)],
                  [w, h],
                  [0, h]])
dst = np.float32([[0, int(h*hk)],
                  [w, int(h*hk)],
                  [w, h],
                  [0, h]])

```
This resulted in the following source and destination points:

Source
```
[[  320.   569.]
 [  960.   569.]
 [ 1280.   720.]
 [    0.   720.]]
```
Destination
```
[[    0.   569.]
 [ 1280.   569.]
 [ 1280.   720.]
 [    0.   720.]]
```

I verified that my perspective transform was working as expected by drawing the `src` and `dst` points onto a test image and its warped counterpart to verify that the lines appear parallel in the warped image.

![alt text][image4]

#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

I identified lane-line pixels in two ways dependent on conditions: 
* Scan the area around previous detected lines with a margin in case lines was detected successfully in the previous frame. The code for this part contains in `LaneAgent()` class, function `_search_lines(self, binary_warped)` (cell 12 from line 83 in the [IPython notebook](./notebook.ipynb))
* Scan the whole frame in case N fails of line detection. The code for this part contains in `LaneAgent()` class, function `_search_lines_with_previous_detection(self, binary_warped, left_fit, right_fit)` (cell 12 from line 162 in the [IPython notebook](./notebook.ipynb)).

After every successfull detection I fit my lane lines with a 2nd order polynomial.

![alt text][image5]

#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

The code for calculation the radius of curvature is a part of `Line()` class that includes computed `@property` called `curverad(self)` (cell 12, lines 55 through 71 in the [IPython notebook](./notebook.ipynb)).

I defined the following conversions from pixels to meters:

```
ym_per_pix = 30/720 # meters per pixel in y dimension
xm_per_pix = 3.7/700 # meters per pixel in x dimension
```
Then I calculated polinomial coefficients with respect to meter unit and calculated radius of curvature in meters.

The position of the vahicle is calculated in function `center_offset(self, image, left_fitx, right_fitx)` (cell 13, lines 70 through 81 in the [IPython notebook](./notebook.ipynb)) as a difference between center of the lane and center of the image.


#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

I implemented this step in lines 39 through 47, cell 13 of [IPython notebook](./notebook.ipynb).  Here is an example of my result on a test image:

![alt text][image6]

---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my video result](./lane_finding_project_video.mp4)

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

The most difficult part for me was detecting dashed and dotted line.

I think the algorithm is not robust and will likely to fail in the city with a lot of vehicles, turnes and road intersections.

For current problem I would like to use supervised learning technic with training dataset contained lane lines and not lane lines pictures. If the dataset will good enought (different weather and lighting conditions, different angles and etc) I can suppose the detection accuracy should be much better.

