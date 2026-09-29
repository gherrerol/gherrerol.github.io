---
title: "Assignment 2: 3D Reconstruction"
permalink: /portfolio/robotics/3d/
layout: single
author_profile: true
---

In this second robotics lab, the goal is to implement a solution to the Unibotics problem: 3D Reconstruction. The lab involves reconstructing a series of objects visible in the scene (Mario Bros. figures, a cereal box, letter blocks, and a duck) based on images captured by a pair of stereo cameras.

The challenge lies in obtaining a point cloud that represents the edges of the objects in the scene. To do this, we must apply the concepts learned in class regarding backprojection and point matching, while also taking epipolar geometry into account.

[![Scene](/images/robotics/escena.png)]

Below, I will discuss in sections how the project was developed, the final implementation, and the problems encountered, starting with image preprocessing.

## 1. Preprocessing

Before we even process a single pixel, we need to mathematically model how our robot’s cameras perceive the world. To do this, we start by saving the sensor’s physical constants provided by the simulator console (which gives us the intrinsic (K) and extrinsic (RT) matrices):
* **Focal Length (F = 240.0):** focal length in pixels; defines the camera’s perspective.
* **Optical Center (CX = 320.0, CY = 240.0):** Since our images are 640x480, this is the exact point where the central ray of light hits the camera sensor.
* **Baseline (B = 220.0 mm):** the physical distance separating the left camera from the right. The greater this separation, the better the robot’s depth perception will be for distant objects. The console shows that the distance from the central axis to each camera is 110 mm; therefore, we obtain the value of 220 mm.

Using this data, we construct the **Stereo Projection Matrices (P1 and P2)** because the triangulation function we’ll use later requires them to understand where the cameras are located in 3D space.
* **P1 Matrix (Left Camera):** We set this to the origin of the universe (0,0,0).
* **P2 Matrix (Right Camera):** This is identical to P1, but we specify that it is offset along the X-axis by the baseline. This offset is encoded in the last column by multiplying the focal length by the baseline.

The next step for the robot to match points between the left and right cameras is to extract useful features from the images. Processing every pixel in the image (640x480) would significantly slow down the system’s performance, so we’ll isolate only the edges.

To clean the image of sensor noise and light variations without destroying important information, I used a bilateral filter which, unlike a Gaussian blur, smooths flat areas while preserving strong contrasts.

On these filtered images, I applied the Canny algorithm to obtain a binary edge mask. This way, I ensure that the search for correspondences is limited solely to the silhouettes that define the scene’s structure, drastically reducing the computational cost without losing the interpretation of the figures.

[![Canny edges](/images/robotics/canny.png)]

## 2. Matching and Epipolar Geometry

The next step—and the core of this approach—is to take an edge pixel from the left camera and find its corresponding pixel in the right camera. Searching the entire right image would be impossible, which is why we use epipolar geometry.

At first, one might assume that the cameras are perfectly aligned and search for the pixel simply by moving along the same horizontal row. However, to create a robust and general system (one that works on real robots where the cameras may not be canonical), I implemented mathematical ray tracing.
The process is as follows:

1.  I take a point from the left camera and draw an imaginary ray into 3D space.
2.  I project that ray onto the right camera, drawing a straight line on the right image.
3.  I search for the corresponding pixel by iterating solely along that line.

To determine whether two pixels are the same, we do not compare them pixel by pixel; instead, I extract a 15x15-pixel “patch” or window around the left point and use *Template Matching* to search for that same texture pattern on the right line.

In this section, the main problem was false positives (the patch was mistakenly matched to the background). The solution was to add a **disparity limit**. Since the right camera is physically offset to the right, the object will always appear further to the left in its image. By limiting the search to a maximum of 150 pixels to the left, I was able to capture both the background and the blocks in the foreground, eliminating some of the noise.

It’s also worth noting that I’m using the approach described by my colleague **Jorge Lozoya** on his blog; I found it very useful because by constraining the projection line within a range of minimum and maximum depth values, we avoid rendering ghost points and eliminate noise in the triangulation and final rendering.

## 3. Triangulation and Reconstruction

When we find a matching patch that exceeds our threshold, we have a pair of 2D coordinates corresponding to the same physical object. Since we have the 2D coordinates of the same point from both cameras, the final step is to transform it into a 3D point (X, Y, Z), using matrix triangulation (via the `triangulatePoints` function), crossing the line of sight rays using the cameras’ projection matrices (which contain the focal length and baseline distance), and returning a vector in 4D homogeneous coordinates. By dividing the first three coordinates by the fourth, we finally obtain our three-dimensional Cartesian coordinates (X, Y, Z) in millimeters.

This was undoubtedly the most visually challenging part and where calibration errors became most evident:

* **Points clustered at the origin:** In the early stages when I was trying to implement triangulation, the points clustered at the origin of the robot's camera. This was because I initially extracted the homogeneous coordinates directly without dividing the resulting vector by the fourth coordinate (the scale), which caused the vector values to be close to zero and to be plotted at the origin. Furthermore, before implementing the disparity limit, my system could mistakenly match a pixel on the far right of the left image with one on the far left of the right image. This generated a huge disparity, and when divided by such a large number, the depth tended toward 0, as the system interpreted these false positives as being right up against the camera lens. This problem was completely eliminated by introducing the depth clipping constraint I mentioned in the previous section (Z_MIN = 2000), which discards any point mathematically less than 2 meters away and limits the maximum horizontal disparity search range.

[![Points clustered at origin](/images/robotics/apelotonado.png)](/images/robotics/apelotonado.png)

* **“Radioactive” colors:** In my initial tests, Mario appeared bright cyan instead of red. It turned out that the simulation images are captured in BGR format, and when I extracted the color and converted it to RGB without reversing the order, the red and blue channels were swapped.

* **The upside-down scene:** When viewing the point cloud in the web viewer, the scene appeared flipped and mirrored. This is because the axes were oriented differently than I had thought; a quick fix was to add a minus sign to the Y and X axes, and the scene was immediately oriented correctly.

[![Flipped scene](/images/robotics/girado.png)](/images/robotics/girado.png)

* **The buried scene:** Now, when viewing the point cloud in the web viewer, the scene appeared cut in half, buried beneath the floor grid. This happens because the coordinate origin (0,0,0) is the center of the robot’s camera, which is at a certain height above the ground. The solution was to apply a vertical offset (a translation along the Y-axis) to raise the entire point cloud so that it would visually rest on top of the grid.


## 4. Final Result

<div style="text-align:center">
  <iframe width="700" height="394"
  src="https://www.youtube.com/embed/W_Qcs3pPGiA"
  frameborder="0" allowfullscreen>
  </iframe>
</div>


## 5. Conclusions

This exercise turned out to be very comprehensive and instructive—a bit more complicated than the previous one but less experimental. I find the possibility of reconstructing any scene based on information obtained from two cameras without a depth sensor very interesting; however, it requires greater mathematical and technical knowledge, which explains the complexity and the problems I encountered.

There are surely many optimizations yet to be tested, but even so, I believe we have obtained a point cloud that is sufficiently dense and accurate for the purpose of this exercise.

