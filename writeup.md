# Advanced Lane Finding
[//]: # (Image References)

[distortions]: https://www.back-prop.com/images/writeup_4_0.png
[radial_correction]: ./images_writeup/radial_correction.png
[tangential_correction]: ./images_writeup/tangential_correction.png
[xycorrected]: https://video.udacity-data.com/topher/2016/December/5840ae19_screen-shot-2016-12-01-at-3.10.19-pm/screen-shot-2016-12-01-at-3.10.19-pm.png
[before_sobel]: ./images_writeup/before_sobel.png
[after_sobel]: ./images_writeup/after_sobel.png
[gradients]: ./images_writeup/gradients.png
[birds-eye]: ./images_writeup/birds-eye.png
[polynomial]: ./images_writeup/polynomial.png
[object_points]: ./images_writeup/object_points.png
[corners]: ./images_writeup/corners.png
[board]: ./images_writeup/board.png
[original_undistorted]: ./images_writeup/original_undistorted.png
[undistorted_raw]: ./images_writeup/undistorted_raw.png
[gradients]: ./images_writeup/gradients.png
[sobel]: ./images_writeup/sobel.png
[sobel_x_y]: ./images_writeup/sobel_x_y.png
[gray]: ./images_writeup/gray.png
[thresh_image]: ./images_writeup/thresh_image.png
[source_points]: ./images_writeup/source_points.png
[source_points_image]: ./images_writeup/source_points_image.png
[warped_image]: ./images_writeup/warped_image.png
[histogram]: ./images_writeup/histogram.png
[sliding_window]: ./images_writeup/sliding_window.png
[radius_equation]: ./images_writeup/radius_equation.png
[hsv-hls]: ./images_writeup/hsv-hls.png
[hsl_thresh]: ./images_writeup/hsl_thresh.png
[hsl_thresh_warp]: ./images_writeup/hsl_thresh_warp.png
[actual_sliding_window]: ./images_writeup/actual_sliding_window.png
[advanced_lane_finding]: ./images_writeup/advanced_lane_finding.png
[shadows]: ./images_writeup/shadows.png
[full_pipeline]: ./images_writeup/full_pipeline.png
[review_img]: ./images_writeup/2.png

The goals/steps of this project are:

* Compute the camera calibration matrix and distortion coefficients given a set of chessboard images.
* Apply a distortion correction to raw images.
* Use color transforms, gradients, etc., to create a thresholded binary image.
* Apply a perspective transform to rectify binary image ("birds-eye view").
* Detect lane pixels and fit to find the lane boundary.
* Determine the curvature of the lane and vehicle position with respect to center.
* Warp the detected lane boundaries back onto the original image.
* Output visual display of the lane boundaries and numerical estimation of lane curvature and vehicle position.

---


## 1. Camera Calibration & Distortion Correction

According to the *"Udacity's Self Driving Car Nanodegree Course"* class notes of Class 5 (Distortion Correction) - Lesson 6 (Camera Calibation), *"image distortion occurs when a camera looks at 3D objects in the real world and transforms them into a 2D image - and this transformation isn???t perfect! Distortion actually changes what the shape and size of these 3D objects appear to be. So, the first step in analyzing camera images, is to undo this distortion so that one can get correct and useful information out of them"*.

Here, basically, we want to deal with two major kinds of distortion: the **radial distortion** and the **tangential distortion**. 

While radial distortion causes straight lines or objects to appear **more or less curved than they really are** (and, as we'll see later on when analyzing lane lines, this kind of distortion becomes larger the farther points are from the center of the image), tangential distortion is often caused when the camera lens and the image plane are not parallel, making an image look tilted (and, consequently, making lines or objects to appear **farther away or closer than they actually are**).

<mark>But, how do we calibrate a camera so we can deal with these kinds of distortion?</mark>

One method is to take pictures of known shapes, so we'll be able to easily detect and correct any distortion errors on it. A chessboard is typically chosen to be used as this "known shape", because its regular high contrast pattern makes it easy to detect these distortions automatically.

![distortions]

In both cases, in order to correct the distortion in an image, we can make use of the following correction formulas:

### For radial distortion correction
![radial_correction]

Where:

- **x** and **y** are the undistorted pixel locations, and are in the normalized image coordinates.

-  **k1**, **k2**, and **k3** are the radial distortion coefficients of the lens. In most cases, only two coefficients are enough for camera calibration (where k3 has a value close to or equal to zero and is negligible), but, for more extreme distortion (like in a picture taken from wide-angle lenses, for example), 3 coefficients shall be needed for correction.

- **r<sup>2</sup>** : x<sup>2</sup> + y<sup>2</sup> ??? **r** is the known distance between a point in an undistorted image and the center of the image distortion, which is often the center of that image, also known as the *distortion center*. 

![xycorrected]

### For tangential distortion correction
![tangential_correction]

Where:

- **x** and **y** are the undistorted pixel locations, and are in the normalized image coordinates.

- **p1** and **p2** are the tangential distortion coefficients of the lens.

- **r<sup>2</sup>** : x<sup>2</sup> + y<sup>2</sup>

<mark>The pipeline I used to do the Camera Calibration & Distortion Correction in this project was:

### 1.1 Defining the chessboard size

To correctly calibrate the camera, some chessboard pictures were provided to be used as a *pattern* to detect eventual distortions, as explained before.

Observing the chessboard image, we notice that:

**nx** = 9

**ny** = 6

Where:
 * **nx** is the number of corners in a row and **ny** is the number of corners in a column. Remembering that corners are only points where two black and two white squares intersect. We must count only the inside corners and not the outside corners.

```
#Number of corners in a row
nx = 9
#Number of corners in a column
ny = 6
```

We're going to map the coordinates of the corners in this 2D image in an empty array called **"Image points"** to the 3D coordinates of the real undistorted chessboard corners inside **"Object points"**.

The **Object points** will all be the same for all the calibration images, since they will be 3D coordinates (x,y,z) from the top left corner (0,0,0) to the bottom right corner (nx,ny,0) from the real chessboard. Remembering that z-coordinate will always be zero, since we're working with an image on a flat image plane!

![object_points]

### 1.2. Finding the Chessboard Corners (for an 9x6 borad)

To detect the corners:
```
# Glob helps us to read images with a consistent name (like calibration1,calibration2, calibration3...)
cal_images = glob.glob('camera_cal/calibration*.jpg')

# creating empty arrays to store/hold the object points and image points from all the images

objpoints = [] #3D points in real world space
imgpoints = [] #2D points in image plane

# Prepare object points, like (0,0,0), (1,0,0), (2,0,0) ..., (nx,ny,0)
objp = np.zeros((nx*ny,3), np.float32)
# Because the z-coordinate will always be zero, here we'll only generate the coordinates values of x and y
objp[:,:2] = np.mgrid[0:nx,0:ny].T.reshape(-1,2)

fig, axs = plt.subplots(5,4, figsize=(16, 11))
fig.subplots_adjust(hspace = .2, wspace=.001)
axs = axs.ravel()

#To iterate through all the images files, detecting corners and appending points to the object and image points arrays:
for i,fname in enumerate(cal_images):
    #read in each image
    img = mpimg.imread(fname)

    # In order to create the image points, we'll need to detect the corners of the real distorded chessboard image
    # Converting the image to grayscale
    gray = cv2.cvtColor(img, cv2.COLOR_BGR2GRAY)

    # Detecting the chessboard corners
    ret, corners = cv2.findChessboardCorners(gray, (nx,ny), None)

    # If corners are found, add object points and image points
    if ret==True:
        imgpoints.append(corners)
        objpoints.append(objp)
    
        #draw and display the corners in the real images
        corners_img = cv2.drawChessboardCorners(img, (nx,ny), corners,ret)
        axs[i].axis('off')
        axs[i].imshow(corners_img)
```

![corners]

It is possible to notice that there are some chessboard images that our script couldn't detect the image corners - so they didn't show up on the images printed above. This happened because these images didn't show the complete chessboard patters with 9 row corners and 6 column corners (because the photo was taken too close to it, probably), like the one on the example bellow:

![board]

### 1.3. Camera Calibration & Undistorting the test images
Here I basically applied the code developed to undistort the test images and the raw road images.
```
def cal_undistort(img, objpoints, imgpoints):
    #Camera calibration, given object points, image points, and the shape of the image:
    ret, mtx, dist, rvecs, tvecs = cv2.calibrateCamera(objpoints, imgpoints, img.shape[1::], None, None) 
    #Undistorting a test image:
    undist = cv2.undistort(img, mtx, dist, None, mtx)
    return undist
```


```
undistorted_image = cal_undistort(cal_image, objpoints, imgpoints)
```

![original_undistorted]

Looking at the undistorted image, it is important to state that, as the mentor Tejas J explained on this [Udacity forum question](https://knowledge.udacity.com/questions/432920), tangential distortion is different from perpesctive diference. *"Some of the calibration images are purposely taken at a different angle by the camera. Tangential distortion occurs when the lens and sensor are not parallel".*

```
#reading in the test image so we can try undistorting it
test_image = mpimg.imread('test_images/test2.jpg')
undistorted_test_image = cal_undistort(test_image, objpoints, imgpoints)
```

![undistorted_raw]

This image is specially good to demonstrate the undistortion, since it is possible to notice the radial distortion correction by looking at the car hood or the tangential distortion by looking at the yellow deer sign. 

---

## 2. Using color transforms, gradients, etc., to create a thresholded binary image
Taking advantage of the fact that the lines we are looking in this project tend to be close to vertical, we can use gradients in a smarter way in order to detect steep edges that are more likely to be lanes.

We know gradients as an extension of the **morphological operators** and, basically, an **image gradient** is a directional change in the intensity or color in an image. In computer vision, there are some algorithms that can actually track this intensity change, which are used in edge detection and further on object classification, object tracking, and so on.

![gradients]

In computer vision, there are some algorithms that can actually track this intensity change, which are used in edge detection and further on object classification, object tracking, and so on.

One very useful technique is to apply the [Sobel Operator](https://en.wikipedia.org/wiki/Sobel_operator), where, according to R.Fisher, this operator basically performs a 2-D spatial gradient measurement on an image and so emphasizes regions of high spatial frequency that correspond to edges. Typically it is used to find the approximate absolute gradient magnitude at each point in an input grayscale image. 

According to the **???Gradients & Color Space???** class notes, applying the Sobel operator to an image is a way of taking the derivative of the image in the *x* or *y* direction. The operators for *Sobel_x* and *Sobel_y*, respectively, look like this:

![sobel]

This operator is able to calculate the gradients in a specific direction, such as x-axis and y-axis, like the image below, where taking the gradient in the x direction emphasizes edges closer to vertical and taking the gradients in the y direction emphasizes edges closer to horizontal.

![before_sobel]

![after_sobel]

In the above images, it is possible to see that the gradients taken in both the **x** and the **y** directions detect the lane lines and pick up other edges. Taking the gradient in the **x** direction emphasizes edges closer to vertical. Alternatively, taking the gradient in the **y** direction emphasizes edges closer to horizontal. 

To transform the lane images in this project, the pipeline I followed was:

### 2.1.1. Because the sobel function requests the use of a single color channel image, first I converted the images to grayscale

```
gray = cv2.cvtColor(im, cv2.COLOR_RGB2GRAY)
 ```

![gray]

### 2.1.2. Then, I calculated the derivative in the x direction and in the y direction

```
sobelx = cv2.Sobel(gray, cv2.CV_64F, 1, 0)
sobely = cv2.Sobel(gray, cv2.CV_64F, 0, 1)
 ```

### 2.1.3. Because we want to compute the pixel values (of 1 or 0) based on the strength of the x gradient, I calculated the absolute value of the x derivative by using *np.absolute*

```
abs_sobelx = np.absolute(sobelx)
```

4. Although it's not essential to convert the absolute value image to 8-bit (range from 0 to 255), it is a good practice, because the 8-bit image can be useful in the event that you've written a function to apply a particular threshold, and you want it to work the same on input images of different scales, like jpg vs. png.

```
scaled_sobel = np.uint8(255*abs_sobelx/np.max(abs_sobelx))
```

5. Finally, I created a binary threshold to select pixels based on gradient strength. Thresholding is fundamentally a very simple method of segmenting an image into different parts, where thresholding will convert an image to consist of only two values, white or black. In the code, I called thresh_min as the lower bound thresh and threshold_max is the upper bound thresh.

```
thresh_min = 20
thresh_max = 100
sxbinary = np.zeros_like(scaled_sobel)
sxbinary[(scaled_sobel >= thresh_min) & (scaled_sobel <= thresh_max)] = 1
plt.imshow(sxbinary, cmap='gray')
```

![thresh_image]

When using a grayscale image, it is possible to see that, when ploting the thresholded lane lines image, we didn't get solid lines. Instead, we got some kind of "double lines" which can confuse our code later on when doing the next steps. Thats why I'll try to use an different color space to see if I'll get some better results.

### 2.2. Creating thresholded binary image on HSL Image
When analysing images, not only the **RGB** color space can necessary bethe best choice for all the projects. Two other very commonly used color spaces are the **HSV** (Hue, Saturation and Value) and **HLS** (Hue, Lightness and Saturation) color spaces. According to the Udacity's class notes, "Most of these different color spaces were either inspired by the human vision system, and/or developed for efficient use in television screen displays and computer graphics." Here we can see a better 3D representation of each cited color spaces: 

![hsv-hls]

In order to achieve a more robust result, here I'll experiment using the **HLS** color space for the lane detection job.

Here, I basically used the same concept described for the grayscale image pipeline, with some tweaks for a even better result:

```
def sobel_HSL(img, s_thresh=(100, 255), sx_thresh=(15, 255)):
    #Because the sobel function requests the use of a single color channel image, I'll now convert to HSL color space and use the L channel
    hls = cv2.cvtColor(img, cv2.COLOR_RGB2HLS).astype(np.float)
    l_channel = hls[:,:,1]
    s_channel = hls[:,:,2]
    h_channel = hls[:,:,0]
    
    #Calculating the derivative in the x direction and in the y direction
    sobelx = cv2.Sobel(l_channel, cv2.CV_64F, 1, 0)
    sobely = cv2.Sobel(gray, cv2.CV_64F, 0, 1)
    
    #Because we want to compute the pixel values (of 1 or 0) based on the strength of the x gradient, I calculated the absolute value of the x derivative by using np.absolute
    abs_sobelx = np.absolute(sobelx)
    
    #Converting the absolute value to 8-bit (range from 0 to 255)
    scaled_sobel = np.uint8(255*abs_sobelx/np.max(abs_sobelx))
    
    # Threshold x gradient
    sxbinary = np.zeros_like(scaled_sobel)
    sxbinary[(scaled_sobel >= sx_thresh[0]) & (scaled_sobel <= sx_thresh[1])] = 1
    
    #Creating a binary threshold to select pixels based on gradient strength
    s_binary = np.zeros_like(s_channel)
    s_binary[(s_channel >= s_thresh[0]) & (s_channel <= s_thresh[1])] = 1
    
    color_binary = np.dstack((np.zeros_like(sxbinary), sxbinary, s_binary)) * 255
    
    combined_binary = np.zeros_like(sxbinary)
    combined_binary[(s_binary == 1) | (sxbinary == 1)] = 1
    
    return combined_binary
```

![hsl_thresh]
![hsl_thresh_warp]

Got a decent result, specially when comparing to the grayscale thresholded image.

## 3. Perspective Transform

According to Cezanne Camacho - one of the Udacity's instructors, self-driving cars need to be told the correct steering angle to turn left or right, and we can calculate this angle it is known a few things about the speed and dynamics of the car, and <mark>how much the lane is curving</mark>.

After detecting the lane lines using masking and thresholding techniques, we perform a perspective transform to get a birs eye view of the lane, which lets us fit a 2nd degree polynomial to the lane lines so we can extract the curvature of the lanes based on this polynomial.

When working with perspective transform, we must understand that, in real real world coordinates x, y and z, *the greater the magnitude of an object  in z coordinate (or distance from the camera), the smaller it will appear in a 2D image*. So, for us to transform an image, we basically transform the apparent z coordinate of the object points in order to warp the image and effectively drag points towards or push them away from the camera to change the apparent perspective.

![birds-eye]

For a lane line that is close to vertical, it is possible to use the 2nd degree polynomial formula **f(y) = Ay^2 + By + C**, where:

* **A** gives the curvature of the lane line;
* **B** gives the heading or direction that the line is pointing;
* **C** gives the position of the line based on how far away it is from the very left of an image (y=0).

![polynomial]

In order to create and apply the perspective transform (and later measure the lane curvature and get a view that looks more like a map representation), we'll first select four points to define a linear transformation from one perspective to another. Then, we'll use OpenCV functions to calculate the transforms that maps the points in the original image to the warped image with the different perspective.

![source_points]

So, to take a better look and see if the selected source points are correctly manually selected, I displayed them on an undistorted test image using the code:

```
#source image points
img_size = np.float32([(undistorted_test_image.shape[1],undistorted_test_image.shape[0])])
print(img_size)

plt.imshow(undistorted_test_image)
plt.plot(550, 470, '.') #top left
plt.plot(740, 470, '.') #top right
plt.plot(128, 720, '.') #bottom left
plt.plot(1280, 720, '.') #bottom right
```

![source_points_image]

So, the function to properly warp the image woud be:

```
#Define perspective transform function
def warp(img):
    
    # Define calibration box in source (original) and destination (desired or warped) coordinates
    
    img_size = (img.shape[1],img.shape[0])
    
    # Four source coordinates
    src = np.float32(
        [[550, 470],
         [740, 470],
         [128, 720],
         [1280, 720]])
    
    # Four desired coordinates
    dst_size = (1280,720)
    dst = np.float32(
        [[0, 0],
         [1280, 0],
         [0, 720],
         [1280, 720]])

    # Compute the perspective transform, M
    # The function getPerspectiveTransform just takes in our four source points and our four destination points and it returns the mapping perspective matrix, which we'll call M
    M = cv2.getPerspectiveTransform(src, dst)

    # Could compute the inverse also by swapping the input parameters
    Minv = cv2.getPerspectiveTransform(dst, src)
    
    # And now, applying the transform M to the original image to get the warped image
    warped = cv2.warpPerspective(img, M, img_size, flags=cv2.INTER_LINEAR)
    
    return warped
```

![warped_image]

## 4. Detecting lane pixels and fitting them to find the lane boundary

Now that I have a thresholded warped image, here, we will decide which pixels are lane line pixels, and then determine the line shape and position. The method chosen to complete this task is the **Peaks in a Histogram Line Finding Method**.

1. Because lane lines are likely to be mostly vertical nearest to the car, I plotted the lower half histogram of the image with the following code:

```
bottom_half = img[img.shape[0]//2:,:]
histogram = np.sum(img[img.shape[0]//2:,:], axis=0)
plt.plot(histogram)
```

![histogram]

2. After applying the threshold to the images, we got as result a binary images with pixels values of either 0 or 1. As explained in the course notes, *"the two most prominent peaks in this histogram will be good indicators of the x-position of the base of the lane lines. We can use that as a starting point for where to search for the lines. From that point, we can use a sliding window, placed around the line centers, to find and follow the lines up to the top of the frame"*. 

![sliding_window]

Following the Udacity's *Sliding Window* Lesson, in order to iterate through nwindows to track curvature, I used the following pipeline:

1. Loop through each window in <mark>nwindows</mark>

2. Find the boundaries of our current window. This is based on a combination of the current window's starting point (<mark>leftx_current </mark>and <mark>rightx_current</mark>), as well as the margin you set in the hyperparameters.

3. Use <mark>cv2.rectangle</mark> to draw these window boundaries onto our visualization image <mark>out_img</mark>. This is required for the quiz, but you can skip this step in practice if you don't need to visualize where the windows are.

4. Now that we know the boundaries of our window, find out which activated pixels from <mark>nonzeroy</mark> and <mark>nonzerox</mark> above actually fall into the window.

5. Append these to our lists <mark>left_lane_inds</mark> and <mark>right_lane_inds</mark>.

6. If the number of pixels you found in Step 4 are greater than your hyperparameter minpix, re-center our window (i.e. <mark>leftx_current</mark> or <mark>rightx_current</mark>) based on the mean position of these pixels.

7. After finding the pixels belonging to each line through this method, it is necessary to fit a polynomial to the line.

![actual_sliding_window]

The code used on the project has turned out to be relatively efficient on static images, but, when working directly with videos, using the full algorithm on every frame is not that efficient. Once you don't need to do a *"blind search"* again because lane lines don't necessarily move a lot from frame to frame, we can **search in a margin around the previous lane line position**. *So, once you know where the lines are in one frame of video, you can do a highly targeted search for them in the next frame*.

**Note**: In the image above, specifically when observing the windows in the left lane, it is possible no notice that some windows seems to be open/incomplete, but this does not mean that it is wrong! According to the instructor Jay T (where he explained on this [knowledge question](https://knowledge.udacity.com/questions/239078)), *"Matplotlib draws px-wide lines, and due to the resizing that happened when ploting the curves, some lines didn't display properly"*.

## 5. Determine the curvature of the lane and vehicle position with respect to center

In the last step, because we could fit a curve for both left and right lane lines with a second degree polynomial function (y= Ax?? + Bx + C), now the task is for us to determine the curvature of the lane and discover the vehicle position with respect to center. Using as reference this [article by M. Bourne](https://www.intmath.com/applications-differentiation/8-radius-curvature.php), the Radius of Curvature could be achieved by applying this equation:


![radius_equation]


Following this equation's concept, it was possible to discover the actual lane curvature from our images! However, it is important to notice that firstly the curvature value would be based on pixel values. So, in order to get real world curvature values, it is necessary to define conversions in x and y from pixels space to meters.

```
left_curverad = ((1 + (2*left_fit[0]*y_eval + left_fit[1])**2)**1.5) / np.absolute(2*left_fit[0])
right_curverad = ((1 + (2*right_fit[0]*y_eval + right_fit[1])**2)**1.5) / np.absolute(2*right_fit[0])

```

So, the final curvature measurement code was:

```
def measure_curvature(img, leftx, rightx):
    ploty = np.linspace(0, img.shape[0]-1, img.shape[0])
    y_eval = np.max(ploty)
    ym_per_pix = 30.5/720 # meters per pixel in y dimension
    xm_per_pix = 3.7/720 # meters per pixel in x dimension

    # Fit new polynomials to x,y in world space
    left_fit_cr = np.polyfit(ploty*ym_per_pix, leftx*xm_per_pix, 2)
    right_fit_cr = np.polyfit(ploty*ym_per_pix, rightx*xm_per_pix, 2)
    # Calculate the new radii of curvature
    left_curverad = ((1 + (2*left_fit_cr[0]*y_eval*ym_per_pix + left_fit_cr[1])**2)**1.5) / np.absolute(2*left_fit_cr[0])
    right_curverad = ((1 + (2*right_fit_cr[0]*y_eval*ym_per_pix + right_fit_cr[1])**2)**1.5) / np.absolute(2*right_fit_cr[0])

    car_pos = img.shape[1]/2
    l_fit_x_int = left_fit_cr[0]*img.shape[0]**2 + left_fit_cr[1]*img.shape[0] + left_fit_cr[2]
    r_fit_x_int = right_fit_cr[0]*img.shape[0]**2 + right_fit_cr[1]*img.shape[0] + right_fit_cr[2]
    lane_center_position = (r_fit_x_int + l_fit_x_int) /2
    center = (car_pos - lane_center_position) * xm_per_pix / 10
    # Now our radius of curvature is in meters
    return (left_curverad, right_curverad, center)
```

## 6. Warp the detected lane boundaries back onto the original image and outputting visual display of the lane boundaries and numerical estimation of lane curvature and vehicle position.

Using the inverse warp function, I was then able to warp the detected lane boundaries back onto the original image with the following function:

```
def draw_lanes(img, left_fit, right_fit):
    ploty = np.linspace(0, img.shape[0]-1, img.shape[0])
    color_img = np.zeros_like(img)
    
    left = np.array([np.transpose(np.vstack([left_fit, ploty]))])
    right = np.array([np.flipud(np.transpose(np.vstack([right_fit, ploty])))])
    points = np.hstack((left, right))
    
    cv2.fillPoly(color_img, np.int_(points), (249,142,29))
    inverse_warped = inverse_warp(color_img)
    inverse_warped = cv2.addWeighted(img, 1, inverse_warped, 0.7, 0)
    return inverse_warped
```

Also, using the *"left_curverad, right_curverad and center"* arguments from the *measure_curvature()* function, I could display these values directly from the image!

```
out_img, curves, lanes, ploty = sliding_window(img_bin)
print(np.asarray(curves).shape)
left_curverad, right_curverad, center=measure_curvature(img_bin, curves[0],curves[1])
print(curverad)

img_ = draw_lanes(img, curves[0], curves[1])

font = cv2.FONT_HERSHEY_SIMPLEX # To chose other fonts, press TAB after the ???FONT_???, and it will show all the fonts supported by cv2
cv2.putText(img_, text='Left Lane Curvature Radius: {:.2f} m'.format(left_curverad), org=(50,50), fontFace=font, fontScale=1.8, color=(0,0,0), thickness=3, lineType=cv2.LINE_AA)  # org is the bottom left corner of our text box
cv2.putText(img_, text='Right Lane Curvature Radius: {:.2f} m'.format(right_curverad), org=(50,125), fontFace=font, fontScale=1.8, color=(0,0,0), thickness=3, lineType=cv2.LINE_AA)  # org is the bottom left corner of our text box
cv2.putText(img_, text='Vehicle Center Offset: {:.2f} m'.format(center), org=(50,200), fontFace=font, fontScale=1.8, color=(0,0,0), thickness=3, lineType=cv2.LINE_AA)  # org is the bottom left corner of our text box

plt.imshow(img_, cmap='hsv')
```

And the final result:

![advanced_lane_finding]

<mark> * The full project video can be found [here](https://github.com/LucasMirachi/CarND-Advanced-Lane-Lines/blob/main/output.mp4).</mark>

---
# Discussion
This project was definitely the most challenging project I have ever done in my -yet- short developer career so far. I took some months to properly finish this, and honestly thought about giving up several times on the way... But I finally achieved a reasonable result!

One improvement point I have in mind is to tweak a little bit more threshold parameters, together with trying different color spaces for the images, when utilizing these threshold methods. In this project in particular, I noticed that, specifically when working with images where the trees project shadows on the track, the thresholding gets a little confused.

![shadows]

Another idea I would like apply in a future (I have no clue if it is even possible to do directly from the notebook using any Python Library or if I'll need to use an external video editing software) is to put together, in a video, all the processes in a step-by-step layout, so one can visualize all the steps needed to go through in order to achieve the final lane detection result. Something like this, but in "real-time" video:

![full_pipeline]

---
### Project Reviewer notes

(Optional) Include a binary mini-map on the video

Another addition that would be helpful here is if you could display a mini-map in your video to visualize your fitted lines and sliding windows. This is how I did it in my project:

![review_img]

As seen here, the fitted lane does not perfectly cover the correct area; However, from the binary output, I was able to tell that it was caused by the pixels covered by the sliding windows. In this project, you may use this technique in addition to a better thresholding technique mentioned above to get your system to work with the challenge #1 video, but for the harder challenge video, a more viable technique to use is by using semantic segmentation.

It is possible to do that quite easily with opencv. All you need to do is combining the pixels at the appropriate places and resizing the images in each frame. Take a look at the minimap creation in the <mark>Annotate</mark> class from my [project](https://github.com/jaycode/Advanced-Lane-Lines/blob/f518f09b5948d5c6ba8cdad404a7f5df6f3dc2a7/image_pipeline.py#L570) as a reference.