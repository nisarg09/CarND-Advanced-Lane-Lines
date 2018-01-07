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

## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points

### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---

### Writeup / README

#### 1. Provide a Writeup / README that includes all the rubric points and how you addressed each one.  You can submit your writeup as markdown or pdf.  [Here](https://github.com/udacity/CarND-Advanced-Lane-Lines/blob/master/writeup_template.md) is a template writeup for this project you can use as a guide and a starting point.  

You're reading it!

### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The code for this step is contained in the first code cell of the IPython notebook "project.ipynb". 

I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  I applied this distortion correction to the test image using the `cv2.undistort()` function and obtained this result: 

![screenshot](https://github.com/nisarg09/CarND-Advanced-Lane-Lines/blob/master/output_images/chessboard.png)

### Pipeline (single images)

#### 1. Provide an example of a distortion-corrected image.

To demonstrate this step, I will describe how I apply the distortion correction to one of the test images like this one:
![screenshot](https://github.com/nisarg09/CarND-Advanced-Lane-Lines/blob/master/output_images/undistort_road.png)
The effect of undistort is suble but can be perceived from the difference in shape of the hood of the car.

#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

I used a combination of color and gradient thresholds to generate a binary image.  Here's an example of my output for this step.  (note: this is not actually from one of the test images)

![screenshot](https://github.com/nisarg09/CarND-Advanced-Lane-Lines/blob/master/output_images/sobel_mag_dir.png)

The below image shows various channels of differen color spaces for the same image.
![screenshot](https://github.com/nisarg09/CarND-Advanced-Lane-Lines/blob/master/output_images/channel.png)

I chose to use L channel of HLS color space to islate white lines and B channel of LAB to isolate yellow lines. I have not used gradient thresholds in the pipeline. I chose to normalize the maximum values of the HLS L channel and the LAB B channel (presumably occupied by lane lines) to 255, since the values the lane lines span in these channels can vary depending on lighting conditions. Below are examples of thresholds in the HLS L channel and the LAB B channel:
![screenshot](https://github.com/nisarg09/CarND-Advanced-Lane-Lines/blob/master/output_images/HLS_L.png)
![screenshot](https://github.com/nisarg09/CarND-Advanced-Lane-Lines/blob/master/output_images/LAB_B.png)

Below is the result of binary thresholding pipeline to various sample images.
![screenshot](https://github.com/nisarg09/CarND-Advanced-Lane-Lines/blob/master/output_images/p1.png)

#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

The code for my perspective transform in the file `project.ipynb`. The unwarp() function takes input image aswll as src and dst points.  I chose the hardcode the source and destination points in the following manner:

```python
src = np.float32([(575,464),(707,464),(258,682),(1049,682)])
dst = np.float32([(450,0),(w-450,0),(450,h),(w-450,h)])
```
I verified that my perspective transform was working as expected by drawing the `src` and `dst` points onto a test image and its warped counterpart to verify that the lines appear parallel in the warped image.

The image below demonstrates the results of the perspective transform:

![screenshot](https://github.com/nisarg09/CarND-Advanced-Lane-Lines/blob/master/output_images/nwarped.png)

#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?
The functions sliding_window_polyfit and polyfit_using_perv_fit, which identify lane lines and fit a second order polynomial to both right and left lane lines and are clearly labeled in code. 
The first of these computes histogram of the bottom half of the image and finds the bottom most x-position of the left and right lane lines. Originally these locations were identified from the local maxima of the left and right halves of the histogram, but in my final implementation I changed these to quarters of the histogram just left and right of the midpoint. This helped to reject lines from adjacent lanes. The function then identifies ten windows from which to identify lane pixels, each one centered on the midpoint of the pixels from the window below. This effectively "follows" the lane lines up to the top of the binary image, and speeds processing by only searching for activated pixels over a small portion of the image. Pixels belonging to each lane line are identified and the Numpy polyfit() method fits a second order polynomial to each set of pixels. The image below demonstrates how this process works:

![screenshot](https://github.com/nisarg09/CarND-Advanced-Lane-Lines/blob/master/output_images/sliding_window.png)

The image below depicts the histogram generated by sliding_window_polyfit; the resulting base points for the left and right lanes - the two peaks nearest the center - are clearly visible:

![screenshot](https://github.com/nisarg09/CarND-Advanced-Lane-Lines/blob/master/output_images/histogram.png)

The polyfit_using_prev_fit function performs basically the same task, but alleviates much difficulty of the search process by leveraging a previous fit (from a previous video frame, for example) and only searching for lane pixels within a certain range of that fit.
![screenshot](https://github.com/nisarg09/CarND-Advanced-Lane-Lines/blob/master/output_images/polyift.png)


#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.
The radius of the curve is calculated as below:
	```
	curve_radius = ((1 + (2*fit[0]*y_0*y_meters_per_pixel + fit[1])**2)**1.5) / np.absolute(2*fit[0])
	```

In the equation above, fit[0] is the first coefficient of the second order polynomial fit and fit[1] is the second (y) coefficient. y_0 is the y position within the image upon which the curvature calculation is based. y_master_per_pixel is the factor used for converting from pixels to meters.

The position of the vehicle with respect to the center of the lane is calculated with following equation.
	```	
	lane_center_position = (r_fit_x_int + l_fit_x_int) /2
	center_dist = (car_position - lane_center_position) * x_meters_per_pix 
	```
r_fit_x_int and l_fit_x_int are the x-intercepts of the right and left fits, respectively. This requires evaluating the fit at the maximum y value (719, in this case - the bottom of the image) because the minimum y value is actually at the top (otherwise, the constant coefficient of each fit would have sufficed). The car position is the difference between these intercept points and the image midpoint (assuming that the camera is mounted at the center of the vehicle).

#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

I implemented this step in my code in `project.ipynb` in the function `draw_lane()`.  Here is an example of my result on a test image:

![screenshot](https://github.com/nisarg09/CarND-Advanced-Lane-Lines/blob/master/output_images/lane_lines.png)

Below is an example of the results of the draw_data function, which writes text identifying the curvature radius and vehicle position data onto the original image:
![screenshot](https://github.com/nisarg09/CarND-Advanced-Lane-Lines/blob/master/output_images/curved_lane.png)


---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my video result](./project_video_output.mp4)

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

Here I'll talk about the approach I took, what techniques I used, what worked and why, where the pipeline might fail and how I might improve it if I were going to pursue this project further. The problems I encountered were almost exclusively due to lighting conditions, shadwos, discoloration, etc. It wasn't difficult to dian n threshold parameters to get the pipeline to perform well on the original project video, even on lighter-gray bridge sections that comprised the most difficult sections of the video. The lane lines don't necessarily occupy the same pixel value (speaking of the L channel of the HLS color space) range on this video that they occupy on the first video, so the normalization/scaling technique helped here quite a bit, although it also tended to create problems (large noisy areas activated in the binary image) when the white lines didn't contrast with the rest of the image enough. My approach also invalidates fits if the left and right base points aren't a certain distance apart (within some tolerance) under the assumption that the lane width will remain relatively constant.
I've considered a few possible approaches for making my algorithm more robust. These include more dynamic thresholding (perhaps considering separate threshold parameters for different horizontal slices of the image, or dynamically selecting threshold parameters based on the resulting number of activated pixels), designating a confidence level for fits and rejecting new fits that deviate beyond a certain amount (this is already implemented in a relatively unsophisticated way) or rejecting the right fit (for example) if the confidence in the left fit is high and right fit deviates too much (enforcing roughly parallel fits). I hope to revisit some of these strategies in the future.

[link to output result](./project_video_output.mp4)

[link to challange result](./challenge_video_output.mp4)

