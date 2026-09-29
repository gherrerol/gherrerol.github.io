---
title: "Assignment 1: Reactive Control"
permalink: /portfolio/robotics/pid/
layout: single
author_profile: true
---

In this first robotics exercise, the goal is to implement a solution to the Unibotics problem: Follow Line. The exercise consists of getting the robot (in this case, a Formula 1 car) to follow the red line painted on the road across various circuits, using only the image from a front-facing camera.

The challenge is not only to follow the line but to do so stably and quickly under varying conditions (straights, curves of different radii, and transitions between them). To achieve this, we adopted a closed-loop reactive control approach, where the control signal (the vehicle’s steering angle) is calculated in each frame based on the visual error between the line’s position and the center of the image.

Below, I will discuss in sections how the project was developed, the final implementation, and the problems encountered, starting with image preprocessing.


## 1. Preprocessing


The first thing we need to do so the robot can see the red line it needs to follow is to define a color space to isolate it from the rest of the objects in the image. To do this, I used an HSV range to define the red color values. I used the HSV color space because it is more robust against changes in lighting, and I combined two red color ranges since HSV does not have a continuous range—red appears in both the low range (H: 0–10) and the high range (H: 160–180). This ensures that the red hue is being captured.

| Range               | Result                                                   |
|:--------------------|:---------------------------------------------------------|
| Low 0–10            | Detection in partially dark areas                        |
| High 160–180        | Detection in partially lit areas                         |
| Combination of both | Complete and robust detection throughout the entire path |

The result is a binary image in which the white pixels correspond to the red line. The centroid is calculated on this mask using OpenCV's image moments, yielding the X-coordinate of the midpoint of the line in each analyzed area.

There weren’t many issues in this section; the mask was calculated by searching for the color red in HSV space, and it worked on the first try. Debugging the centroids also worked correctly.
