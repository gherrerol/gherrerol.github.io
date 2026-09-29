---
title: "Robotics Projects"
excerpt: "PID line following, stereo 3D reconstruction, marker-based localization, and end-to-end visual control. <br/><img src='/images/robotics.png'>"
collection: portfolio
---

Assignments from Computer Vision Master at URJC, a university robotics subject built around ROS2 and the Unibotics simulation platform (Gazebo). Each one tackles a different piece of the classic robotics stack:

<div style="display:flex; flex-wrap:wrap; align-items:center; gap:1.5em; margin-bottom:2em;">
  <a href="/portfolio/robotics/pid/" style="flex:0 0 240px;">
    <img src="/images/robotics/coche.png" alt="Reactive Control" style="width:100%; border-radius:6px;">
  </a>
  <div style="flex:1 1 260px;">
    <h3 style="margin-top:0;"><a href="/portfolio/robotics/pid/">Assignment 1: Reactive Control</a></h3>
    <p>PD controller for line tracking in an autonomous F1 car, isolating the track in HSV space.</p>
  </div>
</div>

<div style="display:flex; flex-wrap:wrap; align-items:center; gap:1.5em; margin-bottom:2em;">
  <a href="/portfolio/robotics/3d/" style="flex:0 0 240px;">
    <img src="/images/robotics/escena.png" alt="3D Reconstruction" style="width:100%; border-radius:6px;">
  </a>
  <div style="flex:1 1 260px;">
    <h3 style="margin-top:0;"><a href="/portfolio/robotics/3d/">Assignment 2: 3D Reconstruction</a></h3>
    <p>A stereo vision pipeline that triangulates a dense point cloud from the left and right cameras.</p>
  </div>
</div>

<div style="display:flex; flex-wrap:wrap; align-items:center; gap:1.5em; margin-bottom:2em;">
  <a href="/portfolio/robotics/autolocalization/" style="flex:0 0 240px;">
    <img src="/images/robotics.png" alt="Autolocalization" style="width:100%; border-radius:6px;">
  </a>
  <div style="flex:1 1 260px;">
    <h3 style="margin-top:0;"><a href="/portfolio/robotics/autolocalization/">Assignment 3: Autolocalization</a></h3>
    <p>Global positioning using AprilTags via PnP, combined with odometry.</p>
  </div>
</div>

<div style="display:flex; flex-wrap:wrap; align-items:center; gap:1.5em; margin-bottom:2em;">
  <a href="/portfolio/robotics/endtoend/" style="flex:0 0 240px;">
    <img src="/images/robotics/coche.png" alt="End-to-End Visual Control" style="width:100%; border-radius:6px;">
  </a>
  <div style="flex:1 1 260px;">
    <h3 style="margin-top:0;"><a href="/portfolio/robotics/endtoend/">Assignment 4: End-to-End Visual Control</a></h3>
    <p>The same line-tracking task as assignment 1, learned using a CNN trained on labeled images.</p>
  </div>
</div>
