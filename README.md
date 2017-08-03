# **Advanced Lane Finding Project**

My project includes the following files:
* pipeline.ipynb - Jupyter notebook to process the original video (project_video.mp4)
* video.mp4 - output video with lane lines marked
* README.md - summarizing the results

The goals / steps of this project are the following:

* Compute the camera calibration matrix and distortion coefficients given a set of chessboard images.
* Apply a distortion correction to raw images.
* Use color transforms, gradients, etc., to create a thresholded binary image.
* Apply a perspective transform to rectify binary image ("birds-eye view").
* Detect lane pixels and fit to find the lane boundary.
* Determine the curvature of the lane and vehicle position with respect to center.
* Warp the detected lane boundaries back onto the original image.
* Output visual display of the lane boundaries and numerical estimation of lane curvature and vehicle position.

### Camera Calibration

The calibration is done in cell #2 of the [notebook](./pipeline.ipynb). I used `cv2.findChessboardCorners` to find all the corners of the calibration [images](./camera_cal) and created lists of measured coordinates of chessboard corners in the image plane `imgpoints` and object points in a plane (z=0) in real world space `objpoints`.

Here is an example of the corner detection. 

<a href="./camera_cal/corners_found1.jpg" target="_blank"><img src="./camera_cal/corners_found1.jpg" alt="camera calibration" width="360" height="240" border="10" /></a>

### Distortion correction

With `imgpoints` and `objpoints`, the camera distortion can be corrected with `cv2.calibrateCamera()` (cell #3).

![alt text](./output_images/camera_correction.png "distortion correction")

### Thresholded binary image

I used a combination of color and gradient thresholds to generate a binary image (cell #4).  Here's an example of the combined threshold image for test_images/test5.jpg (left). The gradient threshold component (_blue_) and the color channel threshold component (_green_) are shown in the right.

![alt text](./output_images/threshold.png "thresholded binary image")

The code is similar to the one used in the lesson (section 30) that measures the gradient in x direction in the L channel (lightness) and sets a threshold in the S channel (saturation). I also include a cut in the hue space (H < 100) that seems to be useful removing the shadow on the road.

### Perspective transform

Here I assume the road is flat and I'll map a straight lane line image (test_images/straight_lines1.jpg) into vertical lines. I chose 4 points `src` that form a trapezoid in the undistored source image and 4 points `dst` that form a rectangle in the transformed image (cell #5). The mapping `M` is carried out by `cv2.getPerspectiveTransform(src,dst)`.

Here is an example of transforming the image into a bird's eye view perspective. `src` and `dst` points are drawn by red lines.

![alt text](./output_images/perspective.png "perspective transform")


Here is the warped thresholded binary image. The lane lines appear parallel. The binary image has applied region masking (cell #6) before changing into a bird's eye view perspective. Using a mask saves me a lot of troubles finding lane-line pixels next. It also excludes pixels that are not part of the road in the bottom of the image.

![alt text](./output_images/warped_binary.png "warped binary image")

### Identify lane-line pixels and fit their positions with a polynomial

I used the method taught in the lesson to find lane-line pixels (cell #8) and fit those pixels with a 2nd order polynomial. An example of the lane-line pixels (_red_ and _blue_) and the fit (_yellow_) are shown below.

![alt text](./output_images/lane_detection.png "lane detection")


### Calculate the radius of curvature of the lane and the position of the vehicle with respect to the center of the lane

The radius of curvature can be derived from the coefficients of the polynomial, shown in cell #10. To convert the pixel-based value to the physical value, I assume the lane is 3.7m in width and 40m in length (there are 720 pixels in length and the dashed lane line (10ft) is about 60 pixels). The car's position (middle of the image) relative to the center of the lane (estimated from the polynomial fits) is in cell #12.

### Warp the detected lane boundaries back onto the original image

The birds-eye view of the lane is transformed back to the normal view with `Minv` by `cv2.warpPerspective()` and overlaid on the original distortion corrected image in cell #13. An example of the result on test_images/test5.jpg is shown below.

![alt text](./output_images/lane_marked.png "lane detection")

### Pipeline to process the video

I combined all the steps above into a single function `process_image` (cell #16) and used `VideoFileClip` (cell #19) learned from [Project 1](https://github.com/enhsin/p1-laneLines) to process every frame in the video. A `Line()` class (cell #14) is used to store the polynomial fit of the previous frame and to keep track some other parameters. If the lane detection of the previous frame is successful, I'll select pixels that are within the margin of the lane line detected previously to determine the new fit (see section 33 of the lesson). I define a successful detection as the difference between the curvature of the left and the right lane is no greater than 90% (be order of magnitude similar). If the detection fails, it will use the lane line from the previous fit. 

The pipeline works well for most of the frame of the video, but fails to detect the left lane line around 21-23 seconds. Here is the image at 22.2 second. The lane line is quite faint. 

<a href="./test_t22.jpg" target="_blank"><img src="./test_t22.jpg" alt="faint line" width="400" height="240" border="10" /></a>

After lowering the lower threshold for the S channel from 170, the value used in the lesson, to 80 from lots of trial and error, the left lane line can be detected well. The resulting video is [here](./video.mp4). The radius of curvature of the left and right lane line and the vehicle offset are shown in the upper right.

### Discussion

This is my second submission. The curvature radius of my first submission seems to be low, so I redo the perspective transform and rerun the whole analysis. I find it quite tricky to define the source `src` and destination points `dst`. I’m not sure how deep (farther to the top of the image) the source points should go and whether I should view the road close-up or far way top down.  All these matter to the measurement of the curvature. Ideally, we should go as deep as possible to have more data points for the polynomial fit. On the other hand, the road may not be flat and data points on the top are more uncertain. I end up using test_images/straight_lines2.jpg (straight_lines1.jpg in my first submission) and select a region that makes my test case _look good_ (more parallel).

The reviewer also gave me a tip for color thresholding (thanks). It’s much faster than the gradient-base method and it’s less messy, too.

```python
HSV = cv2.cvtColor(your_image, cv2.COLOR_RGB2HSV)

# For yellow
yellow = cv2.inRange(HSV, (20, 100, 100), (50, 255, 255))

# For white
sensitivity_1 = 68
white = cv2.inRange(HSV, (0,0,255-sensitivity_1), (255,20,255))

sensitivity_2 = 60
HSL = cv2.cvtColor(your_image, cv2.COLOR_RGB2HLS)
white_2 = cv2.inRange(HSL, (0,255-sensitivity_2,0), (255,255,sensitivity_2))
white_3 = cv2.inRange(your_image, (200,200,200), (255,255,255))

bit_layer = your_bit_layer | yellow | white | white_2 | white_3
```
I tried to play around with it, but I didn’t find a good set of parameters for the faint lane line images (I probably didn’t try hard enough), so I still used my original color and gradient thresholds. 

