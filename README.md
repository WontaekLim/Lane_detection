## Udacity Project4

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

[//]: # "Image References"

[image1]: ./writeup_image/distorted_image.jpg "Original"
[image2]: ./writeup_image/undistorted_image.jpg "Undistorted"
[image3]: ./test_images/test1/1_raw_img.jpg "Original"
[image4]: ./test_images/test1/2_undistortion.jpg "Undistorted"
[image5]: ./test_images/test1/3_filter_r.jpg "Filter of R channel"
[image6]: ./test_images/test1/4_filter_s.jpg "Filter of S channel"
[image7]: ./test_images/test1/5_combined.jpg "Fitered image"
[image8]: ./test_images/test1/6_perspective.jpg "Perspective transform"
[image9]: ./test_images/test1/7_lanefinding.jpg "Fit visual"
[image10]: ./test_images/test1/8_result.jpg "Result"
[cimage3]: ./chanllenge_images/challenge_13/1_raw_img.jpg "Original"
[cimage4]: ./chanllenge_images/challenge_13/2_undistortion.jpg "Undistorted"
[cimage5]: ./chanllenge_images/challenge_13/3_filter_r.jpg "Filter of R channel"
[cimage6]: ./chanllenge_images/challenge_13/4_filter_s.jpg "Filter of S channel"
[cimage7]: ./chanllenge_images/challenge_13/5_combined.jpg "Fitered image"
[cimage8]: ./chanllenge_images/challenge_13/6_perspective.jpg "Perspective transform"
[cimage9]: ./chanllenge_images/challenge_13/7_lanefinding.jpg "Fit visual"
[cimage10]: ./chanllenge_images/challenge_13/8_result.jpg "Result"

[video1]: ./test_video/result_project_video.mp4 "Video"


### Writeup / README

### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

 The camera matix and distortion coefficients are calculated by finding corners of chessboards. We can easily get these parameters using opencv functions: findChessboardCorners, calibrateCamera. Besides, these functions provides the parameters to convert the image into undistorted image (cv2.undistort). The example image are below.

|   Original image    |  Undistorted image  |
| :-----------------: | :-----------------: |
| ![alt text][image1] | ![alt text][image2] |

### Pipeline (single images)

#### 1. Provide an example of a distortion-corrected image.

Based on the calibration parameters calculated offline, the real-time image can be converted into undistorted image like following images.
|   Original image    |  Undistorted image  |
| :-----------------: | :-----------------: |
| ![alt text][image3] | ![alt text][image4] |

#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

To find lanes from image, we have to extract features to express each lane. In this project, we use two  sobel edge detectors which are based on image derivative. These detectors have different input image channel for robust lane edge detection. The first one is r-channel of RGB for white lane detection. The other one is s-channel of HLS for yellow lane. Two filtered images are merged into combined filter image using OR operation because we want to find white and yellow lanes, together. The example images are below.
| Filter of R channel | Filter of S channel |   Combined filter   |
| :-----------------: | :-----------------: | :-----------------: |
| ![alt text][image5] | ![alt text][image6] | ![alt text][image7] |

#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

`convBirdEyeView` function is to convert image into perspective image using `src`, `dst`. The `src` and `dst` are four points which refers to wrapping reference position of source and destination images, respectively. In this project, I selected these points in the following manner:

```python
offset_x = 100
offset_y = 100

yframe_up = 450
yframe_bom = 680
xframe_ls = 300
xframe_rs = 1120
xframe_ll = 600
xframe_rl = 710

perImgSize_w = 480
perImgSize_h = 720

    src = np.float32( [	[xframe_ll, yframe], 
    					[xframe_rl, yframe], 
    					[xframe_rs, yframe_bom], 
    					[xframe_ls, yframe_bom]] )
    					
    dst = np.float32( [	[offset_x,0], 
    					[perImgSize_w-offset_x,0], 
    					[perImgSize_w-offset_x,perImgSize_h], 
    					[offset_x, perImgSize_h]])
```

Besides, we converted the image size to express the lane more reasonable size on image. The parameters for perspective transformation in this project are blow.

| Source image | Destination image |
| :----------: | :---------------: |
|  (1280,720)  |     (480,720)     |

|   Source   | Destination |
| :--------: | :---------: |
| (600,450)  |   (100,0)   |
| (710,450)  |   (380,0)   |
| (1120,680) |  (380,720)  |
| (300,680)  |  (100,720)  |


This is the result of bird eye view: 
![alt text][image8]

#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

On the persepective image, we have to find the proper pixels to represent a left and right lane, repectively. To do this, we use slding window method. This method searches the lane pixel moving window position. To determine the initial window poisition, a peak point in a histogram of image is used as described in the udacity lecture. However, the method has a problem in a curved road. If the road is straight, we can find the rough poisition of each lane using histogram, easily. If the road is curved, the histogram peak has a chance to point a biased position because the pixels on the curved road far from ego. To overcome this problem, we use weighted histogram like following:

``` python
dist = list( range(int(img.shape[0]/2), img.shape[0]))    
histogram = np.dot( dist, img[img.shape[0]/2:,:])  
```
`dist` means the inverse distance from ego. By  weighted sum of `dist` and `pixel sum to y axis`, the initial position is more strong affection from closer pixels. The result of finding lane pixel is in a following image.

![alt text][image9]

#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

The curvature and vehicle position are calculated by fitting parameters(`fit_cur`). To convert pixel size to word (meters), we use `ym_per_pix` and `xm_per_pix`. these are based on U.S traffic regulation (dashed lane length and width of a lane). 

```python
# Calculate a curvature
    y_eval = 0
    
    ym_per_pix = 30/720
    xm_per_pix = 3.7/310
    
    # Fit new polynomials to x,y in world space    
    fit_cr_left = np.polyfit(ploty*ym_per_pix, (left_fitx[::-1])*xm_per_pix, 2)
    fit_cr_right = np.polyfit(ploty*ym_per_pix, (right_fitx[::-1])*xm_per_pix, 2)
    
    # Calculate the new radii of curvature
    curverad_left = ((1 + (2*fit_cr_left[0]*y_eval*ym_per_pix + fit_cr_left[1])**2)**1.5) / np.absolute(2*fit_cr_left[0])
    curverad_right = ((1 + (2*fit_cr_right[0]*y_eval*ym_per_pix + fit_cr_right[1])**2)**1.5) / np.absolute(2*fit_cr_right[0])
    
```

#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

The final image of finding left and right lanes is below.
![alt text][image10]

---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my video result]( ./test_video/result_project_video.mp4)

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

To reject shadow image from lane image is a difficult task. Besides, in curved road, the sliding window failed to search the initial position because the previous method used a average position of pixels in a half of image. So, we added the weighted histogram method. This method makes our algorithm more robust to find initial window.

And my pipeline failed the lane finding in some frames. The main reason of that is dirty road surface below the example image. Since a boundary between a barrier and road has a strong edge like a lane, my algorithm cannot distinguish the boundary and a yellow lane. 
![alt text][cimage3]

**Fail to find a left lane**
| Confused lane edges  | Fail to find a left lane |
| :------------------: | :----------------------: |
| ![alt text][cimage8] |   ![alt text][cimage9]   |

This problem is due to a poor algorithm to find lane pixels using simple  histogram. In the future, I have to implement the enhanced lane selecting lagorithm to grab the proper lane pixels.
