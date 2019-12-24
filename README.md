## Writeup Template



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


[image1]: ./output_images/undist_diff_cal3.jpg "Undistorted"
[image2]: ./output_images/undist_diff_straight_lines1.jpg "Road Transformed"
[image3]: ./output_images/binary_straight_lines1.jpg "Binary Example"
[image4]: ./output_images/warp_straight_lines.jpg "Original Transformed"
[image5]: ./output_images/warped_binary_straight_lines1.jpg "Warp Example"
[image6]: ./output_images/lane_warped_binary_straight_lines1.jpg "Fit Visual"
[image7]: ./output_images/remark_straight_lines1.jpg "Output"
[video1]: ./project_video.mp4 "Video"

## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points

### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---

### Writeup / README

#### 1. Provide a Writeup / README that includes all the rubric points and how you addressed each one.  You can submit your writeup as markdown or pdf.  [Here](https://github.com/udacity/CarND-Advanced-Lane-Lines/blob/master/writeup_template.md) is a template writeup for this project you can use as a guide and a starting point.  



### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The code for this step is contained in the first code cell of the IPython notebook located in "AdvLaneFinding.ipynb".  

I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.  

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  I applied this distortion correction to the test image using the `cv2.undistort()` function and obtained this result: (left side is distorted image, and right side is undistorted image)



![alt text][image1]



### Pipeline (single images)

#### 1. Provide an example of a distortion-corrected image.

To demonstrate this step, I will describe how I apply the distortion correction to one of the test images like this one: (left side is distorted image, and right side is undistorted image)
![alt text][image2]

#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

I used a combination of color and gradient thresholds to generate a binary image. In particular, I used S channel and a combination of X-Y Sobel, Magitude, Directional thresholds. Because S channel can reliably find the near field lanes however the Sobel based filters can find lanes in the far distance. So I decided to combine both of them. Here's an example of my output for this step, this picture is the binary of straight_lines1.jpg. The code is in the second cell of the "AdvLaneFinding.ipynb" file.   

![alt text][image3]

#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

The code for my perspective transform includes a function called `warp_image()`, which is in the 3rd code cell of the .ipynb file.  The `warper()` function takes as inputs an image (`img`), as well as source (`src`) and destination (`dst`) points.  I chose the hardcode the source and destination points as following:


| Source        | Destination   | 
|:-------------:|:-------------:| 
| 577, 450      | 100, 0        | 
| 703, 450      | 1085, 0      |
| 1283, 720     | 1085, 720      |
| 12, 720       | 100, 720        |


I verified that my perspective transform was working as expected by drawing the `src` and `dst` points onto the two straight line test images to verify warped images. 
![alt text][image4]

And and one warped counterpart to verify that the lines appear parallel in the warped image. 
![alt text][image5]

#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

Then I implemented histogram, sliding window search and search near polyline (base on prior points) and fit my lane lines with a 2nd order polynomial kinda like this:

![alt text][image6]

#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

I did this in `measure_curvature()` function in the 5th code cell of the .ipynb file. Following the formula in course material to compute curvature for both lanes. For lane position offset, this was done by compute the difference between center of the vehicle (basically center of the view assuming camera is centered) and the center point of both lanes. After testing all the images, for curvature, the value is usually between 300m and 1000m during turns, and between 1000m and 10000m when driving straight; for vehicle offset, the value is below 0.5m. 


#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

I implemented this step in `inv_warp()` function in the 6th code cell of the .ipynb file. This is basically a reverse transform from bird's eye view to front view. Here is an example of my result on a test image:

![alt text][image7]

---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my video result](./project_video.mp4)

In the pipeline, the lane search is done by the following procedure:
* if there was a good fit in the last N_buffer frames (meaning `detected` is not zero), then search around prior fit lanes; if there was no good fit in the last N_buffer frames, then perform histogram and sliding window search
* with the polyfit for both lanes, perform sanity check on 4 points - minimal rad, difference between the two curverads (they should be within X3 differences when in curved lanes, and can be much larger difference when driving straight), lane width at y = 719 (bottom of the warped image) and y = 0 (top of the warped image) (usually between 3.Xm ~ 4.Xm)
* if pass sanity check, then update the two lanes, reset the counter `detected` to N_buffer; if not, then decrease the  counter `detected` by one, until it reaches zero
* the averaging is done by buffering the last N_buffer times when good fit of lanes were found. The polynomials were stored and averaged to produce `best_fit` and also the fitted x coordinates were stored and averaged to produce `best_fitx`.


---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

The pipeline works well with good lighting conditions. There are a few areas it could cause issues:
1. if the lane shoulder is narrow then the road boundary could be picked up as part of the left/right lanes
2. if the curvarad is about 300-400 meters, then the far end of the lane on the outside (e.g, right lane during right curve or left lane during left curve) only has a partial presence in the warpped image, in this condition, noise points (as of light change or road material/color change) could cause the cv2.polyfit() function to produce undesired lanes.

Looking at the thresold that I used, the S channel is still quite good, if I have more time, I would investigate and see if certain gradients can be applied to H and S channels and see if those can improve the result.


