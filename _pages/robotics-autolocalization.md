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

```py
detector = pyapriltags.Detector(searchpath=[‘apriltags’], families=‘tag36h11’)
```

We extract the four corners of the detected tag and draw a green box around it, along with a red dot in the center and its ID. By setting the robot to rotate at a constant speed, I was able to verify that the detection was working correctly.

<div style="text-align:center">
  <iframe width="700" height="394"
  src="https://www.youtube.com/embed/llEfwzAZ_0Q"
  frameborder="0" allowfullscreen>
  </iframe>
</div>

## 2. From 2D to 3D: PnP

Once we've verified that the robot detects the tag in pixels, we need to convert it to actual distances in meters. To do this, I calculated the camera’s intrinsic matrix based on its resolution and used the `cv2.solvePnP` function, passing it the four 2D corners of the image and the tag’s actual physical size (0.24 meters). OpenCV returns the exact rotation and translation of the camera relative to the tag.

In theory, the idea seemed simple: if I know where the tag is in the world (by reading it from the provided YAML file) and I know the distance and angle between the camera and the tag, multiplying the homogeneous transformation matrices (4x4) would give me the robot’s exact position in the world.

But at this point, I ran into a problem: the estimate (the red robot) would jump to the tag’s position.

<div style="text-align:center">
  <iframe width="700" height="394"
  src="https://www.youtube.com/embed/trWnT0SUXpE"
  frameborder="0" allowfullscreen>
  </iframe>
</div>

The problem was that I hadn’t properly aligned the axes:
- **OpenCV (camera):** considers the Z-axis to be forward, the X-axis to be right, and the Y-axis to be down
- **ROS (robot):** considers the X-axis to be forward, the Y-axis to be left, and the Z-axis to be up

To solve this, I created two rotation matrices: `T_robot_cam` to align the camera’s view with the robot’s base, and `T_tagROS_tagCV` to indicate that the tag was standing upright, attached to the wall. By multiplying the world matrix by these corrections and the inverse of the camera’s position, the red robot stopped jumping around the map and aligned with the rest.

## 3. Odometry and Sensor Fusion

Tag localization with PnP works correctly; now we need to apply odometry to estimate the robot’s rotation position when it loses sight of a tag. Previously, the robot would freeze until it found another marker and would point directly at it.

To ensure the robot knows where it is at all times, we use the robot’s odometry, but we can only use it to estimate increments as specified in the problem statement. So, to do this, we integrate odometry and vision as follows:
- **Odometry (when I can no longer see tags):** In each iteration, I calculate how far the robot has moved since the previous moment and add that small relative increment to my global position.
- **Vision (when I see tags):** We ignore the odometry and overwrite the estimate with the result from `solvePnP`.

<div style="text-align:center">
  <iframe width="700" height="394"
  src="https://www.youtube.com/embed/qAImHWa8kOg"
  frameborder="0" allowfullscreen>
  </iframe>
</div>

As you can see in the demonstration video, the simulation (the red robot) follows the real robot’s (the green one) turns quite accurately. You can see that when it loses sight of the tag, the robot continues turning in the correct direction until it detects the next one.

## 4. Navigation

Now that we have the robot properly located, the next step is for it to explore the house. At first, I tried navigating using only the camera data (if it sees a tag, it moves forward), but the robot kept bumping into furniture and was unable to recover by turning.

The final solution was to combine tag detection with the laser data provided in the problem statement, establishing an order of priority:
- **Priority 1. Obstacles:** We read the laser’s forward cone, and if there’s anything within 80 centimeters, we ignore the vision data, stop moving forward, and turn (we also ignore some simulator rays that are within the cone but aren’t interpreted as numbers or tend toward infinity).
- **Priority 2. Visual Tracking:** If there are no obstacles and we see a tag head-on, we move toward it, adjusting the turn so that the center of the image aligns with the tag (`HAL.setW(0.001 * error_x)`, where error_x is the difference between the centers).
- **Priority 3. Search:** If there are no obstacles but we do not see any tags, we turn right to explore.

As shown in the video below, the robot is able to navigate the room by correctly detecting the tags, moving toward them, and avoiding obstacles, achieving a fairly acceptable path.

## 5. Final Results

<div style="text-align:center">
  <iframe width="700" height="394"
  src="https://www.youtube.com/embed/Atu8Ncht5Aw"
  frameborder="0" allowfullscreen>
  </iframe>
</div>

## 6. Conclusions

This hands-on activity has proven to be very instructive for reinforcing the self-localization concepts covered in class. In my opinion, it has been the most entertaining one so far; watching the robot move and interact with its environment is very satisfying, and adjusting the path to avoid obstacles has also been quite rewarding.

There are surely many optimizations left to try, such as finding another way to avoid obstacles, configuring a path that explores the room more thoroughly, and giving the robot more freedom to make decisions when navigating. But I believe that the main mathematical and problem-solving challenges have been resolved as expected, resulting in a decent exploration of the environment.
