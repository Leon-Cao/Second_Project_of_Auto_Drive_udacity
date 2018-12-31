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

[image1]: ./output_images/undistort_chessboard.jpg "Undistorted Chessboard"
[image2]: ./output_images/undistort_test_image.jpg "Undistorted Test Image"
[image3]: ./output_images/Combine_sobels_RS_Channel.jpg "Binary Images"
[image4]: ./output_images/bird-eyes-view.jpg "Birds-eye View"
[image5]: ./output_images/lane_line_and_birdeyes_view_road.jpg "Fit Visual"
[image6]: ./output_images/hightlight_road_images.jpg "Output"
[video1]: ./output_images/project_result.mp4 "Video"

## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points

### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---

### Writeup / README

#### 1. Provide a Writeup / README that includes all the rubric points and how you addressed each one.  You can submit your writeup as markdown or pdf.  [Here](https://github.com/udacity/CarND-Advanced-Lane-Lines/blob/master/writeup_template.md) is a template writeup for this project you can use as a guide and a starting point.  

You're reading it!

### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The code for this step is contained in the second and third code cell of the IPython notebook located in `./Advanced_Lane_Finding.ipynb`
I seperate the import part to first cell due to re-import will cause error in my implementation.

I created a class name Camera. The obj init function has 3 parameters. 1.Filename array, 2.Chessboard corner count of column, 3.Chess corner count of raw. And create obj of Camera named `camera_obj`.
Then use cv2.findChessboardCorners() to find corners for each gray image which converted to gray. If return is True, then append the objp and found corners to `objpoints` and `imgpoints`. After that use cv2.calibrateCamera(`objpoints`, `imgpoints`) to get camera's `mtx`, `dist`, `rvecs` and `tvecs` and record them to in `camera_obj` for future using.

Finally, I used `camera_obj.mtx` and `camera_obj.dist` to apply 'distortion correction' to the test image by `cv2.undistort()` function and obtained this result: 
![alt text][image1]

### Pipeline (single images)

#### 1. Undistortion for each test image or images of video. 

To demonstrate this step, I tried all test image in `./test_images/test*.jpg`. Let me describe how I apply the distortion correction to one of the test images like this one:
![alt text][image2]

#### 2. Color transforms, gradients to create a thresholded binary image in Cell#4 of `./Advanced_Lane_Finding.ipynb`, Class `BinaryImage` and its object named `binaryImage_obj`.

I also create a class named `BinaryImage` and its object named `binaryImage_obj`. I used below parameters for sobel transform and color channel transform to generate each binary image.

| Name            | Threshold | Kernel |
|:---------------:|:---------:|:------:| 
| absolute sobel  | 10, 70    | 11     |
| magnitude sobel | 20, 70    | 11     |
| direction sobel | 0.7, 1.3  | 15     |
| R channel       | 220, 255  | --     |
| S channel       | 180, 255  | --     |

I combined each type binary images to generate output binary image. The combination algorithm is 
 ```python
combine_binary[((sobelx_binary==1)&(sobely_binary==1)) |
              ((mag_sobel_binary==1)&(dir_sobel_binary==1)) |
              (s_binary_image == 1) | (r_binary_image == 1)]=1
```
I used a combination of color and gradient thresholds to generate a binary image by `binaryImage_obj.combine_sobels_RS_channels_f()`. 

Here's an example of my output for each transform methods. 
![alt text][image3]

#### 3. Perspective transform in Cell#5 of `./Advanced_Lane_Finding.ipynb`, Class `ViewTransform` and the local object named `viewTransform_obj`

The code for my perspective transform includes a function called `viewTransform_obj.cvt_image_interesting_range_2_birdview()`. Which is in the Cell#5 line 156 to 182 in `./Advanced_Lane_Finding.ipynb`.  The `viewTransform_obj.cvt_image_interesting_range_2_birdview()` function takes as inputs an image (`arg_image`), as well as a flag to draw interesting range (`drawInterestingRange`) on `arg_image`. 

The interesting range of my implemenation are created by two parameters: `self.center_point` and `self.miniY`. The `self.center_point` is the lanes cross point of straight road. I use this parameter because I assume the angles of`road`, `vehicle` and `camera` are always in same value. So, when vehicle on up-hill road，down-hill road or a `plane` road. The `self.center_point` should same. When the road‘s angle of 'water level' changing, the center point is not correct. So, I use `self.miniY` to idenetify a Y-axis value to reduce the effect. Due to the angle of road changed slowly which close to vehicle. `self.center_point` can be set by `self.set_center_point_f()` and `self.miniY` can be set by `self.set_minimal_y_f()`. The default value of `self.center_point` is `x = imshape[1]*0.5`, `y=imshape[0]*0.586`. Then there is a triangle of `left bottom`, `right bottom` and `self.center_point`. `miniY` will cut the triangle to a trapezium (`RED Lines` trapezium in below example image) which is the `interesting range`. If the `self.center_point` is correct, the four coners of trapezium should in same plane. I use it to do perspective transform. You can find the function `self.set_interesting_range_of_image_f` in line#73 to line#101 of Cell#5. 

The selection of `source points` and `destination points` of perspective transform should avoid transform failed. So, the value of x-axis of `source points` should moved 1/5 to middle of x-axis (`BLUE Lines` in below example image) based on the bottom line and the top line of trapezium. 

The value of x-axis of`destination points` is same as leftx_bottom and rightx_bottom of `source points`. 

I verified that my perspective transform was working as expected by drawing the `src` and `dst` points onto a test image and its warped counterpart to verify that the lines appear parallel in the warped image.

![alt text][image4]

#### 4. Detect lane-line pixels and find the lane boundary. Cell#6, Class `LaneLineCheck` and object named `laneLineCheck_obj`. 

Class `LaneLineCheck` inherit Class `ViewTransform`. Due to all lane-line detection need view transform.

The main functions are `self.fit_polynomial_f()` which can find the lane-lines and `self.draw_road_f()` can draw a green road on a empty bird-view image. `self.combine_birdview_to_eyeview_f()` convert bird-view green road image to eye-view image, then combine with undistorted original image. It is daw road on eyeview image. 

I used sliding windows algorithm to identify the lanes. I did a **little** change on sliding windows. In lession#9, the windows sliding from bottom to top. I implemented both bottom2top and top2bottom. The reason is that, in the test image and video, the right lanes are dash line. The max value of histogram may less than threshold. Calculate the max value of *top half* and *bottom half* are better than calculating one side.

Another optimization on caculate the max value of histogram is to check if the x_base is far away from last detected x_base. If yes, reuse last iteration x_base. Something more, maybe re-calculate the max value from x_base to x_midpoint or imshape[1] is better. 

Here is my example image.

![alt text][image5]

#### 5. Determine radius of curvature of the lane and the position of the vehicle with respect to center.

Class **MeasuringCurvature** and its object `measuringCurvature_obj` in Cell#7.

`self.measure_curvature_real_f()` caculate real radius of right lane and left lane and output an average radius.

`self.measure_real_diff_of_carCenterX_RoadCenterX_f()` caculate the difference vehicle center to road center. 

#### 6. Warp the detected lane boundaries back onto the original image.

I implemented this step in Cell#8, function find_road_f(). 

In this function, I use `laneLineCheck_obj.reuse_num` as the flag to reuse last correct detection for next n images. n can be set in Cell#8 Line29. Currently, I set it to 5. Means correct detection will be reuse 5 times. to reduce calculation duration. If change reuse number to 2 will get better result.

Here is an example of my result on a test image:

![alt text][image6]

---

### Pipeline (video)

#### 1. Project video process.

Here's a [link to my video result](./project_video.mp4)

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

1. For challenge video, the lane-line is not good. The reason is the line is more dark then project_video.mp4. So, the lane-line detection is wrong. Need find more better method to fix it.

2. Calculate the lane width. And use this lane width to optimize next iteration of `Color transforms and gradients` and `fit a polynomial".

3. Calculate the road width to optimize histogram peaks of lane.  

4. Learn more other method.  


