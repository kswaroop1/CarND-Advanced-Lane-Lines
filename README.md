
# Advanced Lane Finding Project

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

[image1]: ./examples/undistorted_unwarped.png "Undistorted Chessboard"
[image2]: ./examples/undistorted_signs_vehicles.png "Undistorted Road"
[image3]: ./examples/threshold_filtered.png "Threshold Filtered Example"
[image4]: ./examples/warped_straight_lines.png "Warp Example"
[image4a]: ./examples/warped_filtered.png "Warp And Filtered Example"
[image5]: ./examples/sliding_windows_lanes.png "Sliding Windows and Initial Lane Dectection"
[image6]: ./examples/lanes_drawn.png "Output"
[video1]: ./project_video.mp4 "Video"

---
### Writeup / README

#### 1. Provide a Writeup / README that includes all the rubric points and how you addressed each one.

Here I will consider the [Rubric](https://review.udacity.com/#!/rubrics/571/view) points individually and describe how I addressed each point in my implementation.  

### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The code for this step is contained in the first two code cells of the IPython notebook located in "./P4.ipynb".

##### Camera Calibration 

In the first code cell I have a function `calibrate_camera()`.  I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image. It is prepared with values such that there are `nx` corners on horizontal axis and `ny` corners in vertical axis of the chess board.

I iterate over over the the camera calibration images provided for the project, where each image is for 9x6 corners. 
`imgpoints` is be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.  I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  The camera matrix is stored in variable `mtx` and the distortion coefficients in variable `dist`.

The name of files that were eventually used for calibration are displyed in the P4 notebook.  Please note the image dimensions are store in global variables `X` and `Y`, used later in the notebook.

##### Demonstration of undistortion and unwarping the chesssboard

In the second code cell, I have a function `corners_unwarp()` that applies the distortion correction to the test images using the `cv2.undistort()` function.  Whenever 9x6 corners are detected, chessboard corners are drawn and then the four ourside corners are used to obtain a perspective transformation matrix using `cv2.getPerspectiveTransform()` function and applied on the undistored image using `cv2.warpPerspective()` function.  Since the outside corners are mapped to fixed points for all images, we expect to see same chessboard image size in all the test images, however the lines drawn to show the corners will be of varying width depending on the amount of "magnification" applied on the undistored image to get the final result as shown below.  Of course, where 9x6 corners can't be detencted due to obscuring of the the corner in the image or after undistortion, there is no unwarped image.

![alt text][image1]

### Pipeline (single images)

#### 1. Provide an example of a distortion-corrected image.

The camera matrix `mtx` and the distortion coefficients `dist` used with `cv2.undistort()` function on the Udacity supplied image "signs_vehicles_xygrad" very vividly demostrate the  distortion correction.  The signboard in the original image cane be clearly seen suffering from radial distortion.  This is undistorted to appear straight in the RHS image below.

![alt text][image2]

#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

The code for this step is contained in the fourth code cells of the IPython notebook located in "./P4.ipynb".

After much experimentation, I ended up using a combination of color saturation (S channel from HLS) and luninosity (L channel from HLS) with sobel on L channel only along X axis.  Both colour and luminosity channels have their own thresholds to generate a binary image.  


Here's an example of  output for this step for the test images supplied for the project.

![alt text][image3]


#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

I first computed the persptective transformation (and its inverse) matrix by carefully calibrating the the source (`src`) and destination (`dst`) points on the two stright lanes images supplied for the project.  This can be seen in the fifth code cell in the "./P4.ipynb" note book.

```
offset,x2_offset,warp_offset=175,45,350
y2, x0,x1,x2,x3=450, offset, X//2-x2_offset-1, X//2+x2_offset+3, X-offset+1

src=np.float32([[x0,Y],[x1,y2],[x2,y2],[x3,Y]])
dst=np.float32([[warp_offset,Y],[warp_offset,0],[X-warp_offset,0],[X-warp_offset,Y]])

```
This resulted in the following source and destination points:

| Source        | Destination   | 
|:-------------:|:-------------:| 
|  175, 720     | 350, 720      | 
|  594, 450     | 350, 0        |
|  688, 450     | 930, 0        |
| 1106, 720     | 930, 720      |

I verified that my perspective transform was working as expected by drawing the `src` and `dst` points onto a test image and its warped counterpart to verify that the lines appear parallel in the warped image, as shown below.

![alt text][image4]

###### Further testing of warping, to check the lanes are roughly parallel in bird's eye view
![alt text][image4a]

#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

8th code cell in "./P4.ipynb" notebook has code for helper class `Line` used to track and average over last N lane lines detected.  It has method `iterate()` that takes the vectors `x` and `y` of coordinates detected for a lane.
- It fits a 2nd order polynomial using `np.polyfit()` function over `x` and `y` in pixel coordinates
- It scales the `x` and `y` to woorld coordinates in meters using scaling supplied in the project instructions and fits another 2nd order polynomial to in meter units
- Finally the meter units quadratic is used to calculate the radius of curvature in meters
The code for these three steps is listed below:
```
self.current_fit = np.polyfit(y, x, 2)  # fit 2nd order polynomial in pixel corrdinates
fit_cr = np.polyfit(y*ym_per_pix, x*xm_per_pix, 2) # fit 2nd order polynomial in world coordinates (meters)
m_curve=((1 + (2*fit_cr[0]*Y*ym_per_pix + fit_cr[1])**2)**1.5) / np.absolute(2*fit_cr[0]) # curvature in meters
```
This is not the final value for curvature, see next question on how this value is used.

Code in 9th cell lists `slide_win_fit_poly()` function used to detect **initial** left and right lanes by using histogram on bottom left and bottom right quarter of filtered image.  The code is slightly modified from course text to work with the `Line` class.

The various sliding windows, detected pixels for left ane right lanes and plotted lane line for various test images is shown below.

![alt text][image5]

#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

The `Line.iterate()` method mentioned above, continues after detecting the curvature to test the new curvature and fit against current best estimate for curvature and fit and rejects the lane values detected if the curvature is too sharp or the the fit differs a lot.  If the curvature are fit are within acceptable limits, the set of detected points are added toa the `deque collection` of last N values and averaged.  The 2nd order polynomial is fitted again in pixel and world coordinates on the averaged values and finally another radius of curvature is obtained. The lane line coordinates are calculated again and the X coordinate corresponding to the bottom-most pixel line of image is used to detect the offset from the middle of the image and scaled to meters to get position of vehicle with respect to center.

```
# project over all height (Y coordinates) of image
fitx = self.current_fit[0]*self.ploty()**2 + self.current_fit[1]*self.ploty() + self.current_fit[2]
self.recent_xfitted.append(fitx) # append, dequeue only keeps last N values
self.bestx = np.average(self.recent_xfitted, axis=0) # average to get current best estimate

# best fit obtained on average over last N iterations (to be precise, avergae of last N detected lanes)
self.best_fit = np.polyfit(self.ploty(), self.bestx, 2) # 2nd order polynomial fitting
self.allx = self.best_fit[0]*self.ploty()**2 + self.best_fit[1]*self.ploty() + self.best_fit[2] # LANE LINE

# Curvature
fit_cr = np.polyfit(self.ploty()*ym_per_pix, self.allx*xm_per_pix, 2)
self.radius_of_curvature = ((1 + (2*fit_cr[0]*Y*ym_per_pix + fit_cr[1])**2)**1.5) / np.absolute(2*fit_cr[0])

# use the bottom most point if the lane line to detect offset from center
self.line_base_pos=xm_per_pix*(self.allx[-1]+self.allx[-1]-X)/2
```

Finally the average of the left and right lane curvature and offset from center is written on the output image.

#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

Code in 10th cell lists `search_lanes()` function used to detect the lanes lines for subsequent images in the video. The code is slightly modified from course text to work with the `Line` class, two instances `LeftLane` and `RightLane` are used to track the two lanes. It  arranges to call the initial lane detection using sliding window fit method to determine initial lane positions.  For subsequent images, it uses the lanes detected in previous frame as starting point.

After lane detection, it uses the lane lines' x coordinates for the two lanes (which are in pixels in the warped bird's eye view) to first fill a polynomial (again in bird's eye view) and then inverse transforms the lane polynomial and adds it to the undistorted image.

This lane detection on warped images and finally drawn back to undistorted image is illustrated below.

![alt text][image6]

---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my video result](./project_video_output.mp4)

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

Here I'll talk about the approach I took, what techniques I used, what worked and why, where the pipeline might fail and how I might improve it if I were going to pursue this project further.  

Initial difficulties with camera calibration, esp on figuring out the `objp` were overcome with google serach of the camera calibration, I cite the source in my code.  From there on, the warping, initial version of lane detection and drawing it back on the undistorted image was done quite quickly.

I then went about making the process a bit more robust, with `Line` class to average over the last N lines detected and finally on determining the threshold binary filter for images.  I took several snapshots from the project and challenge videos.

Getting good thresholds for filtering the image is key to solving this proble, aided bygood limits and "memory" for averaging over last N lanes detected.

###### Threshold and filtering

The gradient threshold (combination of X and Y axis) was not very useful.  I tried using Canny edge detection and Hough Lines detection, which slowed down the filtering steps massively and (at least at the point where I left it for current submission) did not give good results.  In my P1 submission, I had gradient calculation on the Hough lines detected and I used to eliminate the lines with low absolute gradients (somewhat flat/horizontal/parallel to x-axis) before using them. 

The challenge video is beset with the initial lane recognition problem itself that carries on to rest of the video. The first few frames have road works in the middle of the lane and that itself is detected as a lane line due to S channel filter.  A combination of S channel and magnitude threshold would help; so would some guidance on location of initial lane detected.

Here's a [link to challenge video result](./challenge_video_output.mp4)

I did not have suffient time to try out possible vairations of those techniques to improve results for the challenge and harder challenge videos.

###### Limits on curvature and difference from previous best lane detection

I set about ignoring lanes where curvature is less than 184 m following US highway design guidelines.  I see several 'frames' ignored due to high curvature in harder challenge video.  I also want to try out experimenting with exponential weighting with most recent detection given higher weights.  ALso, since the harder challenge has frames with both left and right turns, I want to experiment with fitting 3rd order polynomial.  Finally, I want to use the highest Y axis for lanes actually detected, rather than the fixed point, for warp and inverse transformation to draw the lanes.

Here's a [link to harder challenge video result](./harder_challenge_video_output.mp4)
