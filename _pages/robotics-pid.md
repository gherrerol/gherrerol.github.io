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


## 2. PD Control


The next step was to assign a standard speed to the robot so it would start moving. When given a constant speed and reaching a curve, it would crash; this is where the robot’s turning control comes into play, using a PD controller. The control formula for the PD controller is:

> **W = −( Kp · e + Kd · Δe/Δt )**

where:
- **e** = cx − c\_image → lateral error in pixels
- **Δe** = e(t) − e(t−1) → change in error between frames
- **Kp = 0.003** → proportional gain
- **Kd = 0.08** → derivative gain


As mentioned, the error is given by the difference between cx (which represents the centroid of the red line)—which we calculate using OpenCV’s moment functions—and the center of the image (which is the reference point to which we want to align the red line). As for the proportional term, it produces a correction proportional to the line’s current displacement, while the derivative term acts on the rate of change of the error: if the error increases rapidly (the car is veering off course more and more), it applies an additional correction; if the error decreases (the car is centering itself), it slows down the correction to prevent overshooting the center.

In an initial attempt, using only the proportional component, the robot was already able to follow the line (at a moderate speed) throughout the entire course without excessive oscillation. The final time to complete one lap with this configuration was 190 seconds, far from the target time of 90 seconds.

The next step was to increase the speed, which caused the vehicle to skid practically on the first turn. The solution was to add the derivative term to correct the trajectory in sections with greater error (that is, on the turns). This resulted in a reduction of nearly half the time, bringing the lap time down to 100 seconds.

In this section, the main challenge was adjusting Kp and Kd, which required experimenting with different configurations until finding the one that yielded the best and most consistent results (these values being those mentioned in the PD control formula).

- **Low Kp, low Kd:** The car reacts late to turns. It loses its line in tight turns due to insufficient correction.
- **High Kp, low Kd:** The car oscillates around the line even on straights. The corrections are amplified with each frame, creating a zigzag motion.
- **Low Kp, Kd higher than Kp**: The car follows the line with virtually no oscillations and smoothly returns to the center after turns. This was the desired behavior.


This could have been the end point, but I wanted to reduce the time, so the next step would be to replace the constant speed with an adaptive one.


## 3. Adjusting Speed (Multi-Zone Detection and Lookahead)


Since the car was traveling at a constant speed, abrupt changes in track section posed the risk of a collision, as the car would be unable to decelerate or brake in time to navigate the section of the track. To solve this, I decided to implement a speed that would adapt as the car approached a curve or straightaway.

With a single global center of mass, the robot didn’t have enough information to anticipate what was ahead; it only reacted once the turn was already directly beneath the car, causing delayed decelerations and abrupt corrections. To be able to anticipate, I divided the image into three horizontal bands to analyze the track at different distances:

<div style="text-align:center">
  <img src="assets/images/franjas.jpeg" width="700" alt="Descripción">
</div>

<table style="width:100%; border-collapse:collapse; font-family:monospace; font-size:0.9em;">
  <tr style="background:
#1a1a1a;">
    <td style="border:1px solid #444; padding:8px; color:#888; width:15%;">0%</td>
    <td style="border:1px solid #444; padding:8px; color:#888;">(cielo / fondo) — sin información útil</td>
  </tr>
  <tr style="background:
#0d2200;">
    <td style="border:1px solid #444; padding:8px; color:
#ff4444; width:15%;">50 – 68%</td>
    <td style="border:1px solid #444; padding:8px; color:
#ff4444;">● <strong>cx_far</strong> — lookahead: lo que viene</td>
  </tr>
  <tr style="background:
#0d1a00;">
    <td style="border:1px solid #444; padding:8px; color:
#ffff00; width:15%;">68 – 85%</td>
    <td style="border:1px solid #444; padding:8px; color:
#ffff00;">● <strong>cx_mid</strong> — zona intermedia</td>
  </tr>
  <tr style="background:
#001a00;">
    <td style="border:1px solid #444; padding:8px; color:
#00ff00; width:15%;">85 – 100%</td>
    <td style="border:1px solid #444; padding:8px; color:
#00ff00;">● <strong>cx_near</strong> — zona cercana: control inmediato</td>
  </tr>
</table>

By calculating the horizontal difference between the centroids of the far and near zones, I obtain the anticipated curvature. When both centroids coincide, the stretch of road is straight and the car can accelerate; when they diverge, it means there is a curve ahead, so the car begins to brake.

Therefore, the adaptive speed will depend on the following factors:
- **Current error:** reflects how much the car has already deviated
- **Anticipated curvature:** what lies ahead based on the far-field strip


Both are combined by taking the maximum of the two, with additional weight given to the curvature so that the system brakes before reaching the curve, not once it is already in it.

```py
shape_factor = max(error_norm, curvature_norm × 1.3)
v = max_velocity − (max_velocity − min_velocity) × shape_factor
```


This is undoubtedly the most problematic part and the one where I’ve done the most testing in practice. Everything has been experimental, based on trial and error: adjusting the speed ranges for the simple circuit, adjusting Kp and Kd accordingly, and paying special attention to the range bands, which allow me to anticipate (even if only slightly, since there isn’t enough lead time) in order to capture the line’s variation along the circuit and adapt to it.

It’s worth noting that all these values (Kp, Kd, and the speed values) may need to be adjusted depending on the circuit (the range bands do not seem to vary), as we can see in the results below.


## 4. Final Lap and Circuit Testing


### Simple Circuit
<div style="text-align:center">
  <iframe width="700" height="394"
  src="https://www.youtube.com/embed/w43bQVF1FJw"
  frameborder="0" allowfullscreen>
  </iframe>
</div>


The main test on the simple circuit. As you can see, the car stays on the line at all times throughout the lap, staying close even in the turns and with virtually no sudden oscillations. The lap time is finally reduced to 60 seconds—a time that can certainly be improved upon but is quite good compared to the initial 200 seconds. The RTF remains at 99% almost the entire time, indicating that the time is practically identical to that of the simulator.


### Montmelo Circuit
<div style="text-align:center">
  <iframe width="700" height="394"
  src="https://www.youtube.com/embed/4DEuwPCm53Q"
  frameborder="0" allowfullscreen>
  </iframe>
</div>


The first circuit I tested after the satisfactory performance achieved on the Simple Circuit. On this circuit, the car follows the line as expected, without oscillations and adjusting its speed well. It’s worth noting (as is the case with the circuits we’ll look at next) that the RTF drops significantly in some sections of the circuit, falling as low as 6%, which slows down the system’s processing and I believe is what causes the crash at the 7th turn. It may also be due to what I mentioned in the previous section about adjusting the constants and sacrificing speed for consistency and stability.


### Monaco Circuit
<div style="text-align:center">
  <iframe width="700" height="394"
  src="https://www.youtube.com/embed/w8XTNiviupE"
  frameborder="0" allowfullscreen>
  </iframe>
</div>


At first, I wasn't going to test this circuit because I seem to recall that we were advised not to test it in the classroom; even so, I wanted to see how my system would perform in this case. Overall, it works well. The track is long with a couple of sharp turns; the RTF stays between 40% and 60%, dropping as low as 10% in some cases. This may be why the car loses control in the first sharp turn (I’ll need to try adjusting the parameters again).


### Nurburgring y Montreal Classic Circuit
<div style="text-align:center">
  <iframe width="700" height="394"
  src="https://www.youtube.com/embed/55VyaNZQHRU"
  frameborder="0" allowfullscreen>
  </iframe>
</div>


These two have definitely given me the most headaches. For starters, the Montreal Circuit won’t even load for me—I had to test the Classic track directly (though there shouldn’t be any difference between them). On both circuits, as soon as the race starts, the car skids and crashes into the wall. I’ve tried adjusting the Kp, the Kd, the speed ranges, and setting a constant low speed—nothing seems to work. Not only that, but it doesn’t even seem to detect the racing line, which is strange because it works just like it does on all the other circuits. I haven’t managed to find a solution yet, but I’ve been checking out my fellow players’ blogs and see that they’ve had similar issues with these tracks, so I’m not quite sure if it’s just my problem or what might be going wrong. Also, as a little curious side note, I’ve tested the track on two different computers and… the track line is a different color! Could it be something to do with my settings on one of my machines?


<div style="display:flex; gap:10px; justify-content:center">
  <img src="assets/images/torre.jpeg" width="340" alt="Imagen 1">
  <img src="assets/images/portatil.jpeg" width="340" alt="Imagen 2">
</div>


## 5. Conclusions


In summary, this exercise has been very instructive, entertaining, and, in some ways, complex. What initially seemed to have an easy solution—simply setting Kp and reaching 200 seconds—turned out to be a challenge in getting the time down to around one minute without sacrificing smooth, oscillation-free tracking of the line (and that was just on the simple circuit).

There are still many things to test, such as implementing the integral component of the PID control, adjusting the parameters for each circuit, or even devising a better way to anticipate changes in the track layout in order to adapt the speed. It has been a comprehensive exercise that demonstrates the capabilities and possibilities of reactive control.
