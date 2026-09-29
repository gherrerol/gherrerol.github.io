---
title: "Assignment 3: Marker Visual Autolocalization"
permalink: /portfolio/robotics/autolocalization/
layout: single
author_profile: true
---

In this third robotics lab, the goal is to implement a solution to the Unibotics problem: Visual Loc Marker. The lab involves getting the robot to estimate its absolute position and orientation (pose: x, y, yaw) within a 2D home environment. To achieve this, the robot is equipped with a laser sensor, odometry, and a front-facing camera, which it uses to locate visual markers (AprilTags) scattered on the walls of the house.

The main challenge of this lab lies in sensor fusion: we must combine the continuous but noisy odometry estimate with the discrete but accurate information provided by the camera by applying Perspective-n-Point (PnP) algorithms to the detected markers.

Below, I will discuss in sections how the lab was developed, the final implementation, and the problems encountered.

## 1. Initial Detection

The first thing we need to do is verify that the robot is capable of seeing and recognizing its environment—that is, whether it can detect the markers. To do this, we convert the image to grayscale and use the detector from the `pyapriltags` library (configured for the tag36h11 family, as specified in the problem statement).
