---
title: "Assignment 4: End-To-End Visual Control"
permalink: /portfolio/robotics/endtoend/
layout: single
author_profile: true
---

In this fourth and final robotics lab, we aim to implement a solution to the Unibotics problem: End-to-End Visual Control. The goal, just like in Lab 1, is to get the car to follow the line along the track, but using a different approach. Instead of manually designing a controller that reacts to the image, we will train a neural network to learn how to drive directly from the image it receives.

This approach is called “end-to-end”: the network receives the image from the camera and directly outputs the actual values for speed (v) and steering angle (w), which are sent directly to the car. To do this, we start with a dataset of 50,000 images with their corresponding v and w labels. The chosen network architecture is PilotNet, originally proposed by NVIDIA for autonomous driving.

Next, I’ll discuss in sections how the practical development and final implementation were carried out, along with the results obtained.



## 1. The Dataset

The dataset used is JdeRobot/e2e-visual-control-combined-dataset, consisting of 50,000 images organized into 10 folders and a label.csv file with three columns: image path, linear velocity, and angular velocity. Before training, it is helpful to understand what the dataset contains:


| Variable                | Minimum          | Maximum          | Mean            |
|:------------------------|:-----------------|:-----------------|:----------------|
| v (linear velocity)     | ~ 2.0            | ~ 7.0            | ~ 4.6           |
| w (angular velocity)    | ~ -2.5           | ~ 2.5            | ~ 0.0           |


The distribution of w is centered at 0, which makes sense: the expert pilot flies straight most of the time. This means that a network that learns to always predict w=0 would achieve an artificially low error; this must be taken into account when choosing the loss function.

The split used was 80% training and 20% validation, randomly stratified with a fixed seed for reproducibility.


## 2. Preprocessing

The images in the dataset have a resolution of 640×480 and include sky, grass, and asphalt. The information useful for driving is exclusively in the lower half, where the red line appears; therefore, including the upper half has no value for the network. The following image preprocessing steps were applied:

- **Cropping the lower half:** removes the sky and background, leaving only the road area.
- **HSV thresholding:** As in Exercise 1, two ranges in HSV space are applied to isolate the red line from the asphalt. The result is an image where only the pixels of the line (represented in white) have value; the rest is black.
- **Resizing to 200x66:** the standard input size for PilotNet.
- **Pixel normalization:** the values are scaled to the range (0,1), and the standard ImageNet normalization is applied (mean and standard deviation per channel).

The decision to apply the HSV mask before feeding the image into the network is based on the fact that, rather than letting the network learn on its own to ignore the background, providing it directly with the relevant information reduces the complexity of the problem and accelerates convergence.


## 3. Model and Training

PilotNet was used for the model architecture; it is a relatively small convolutional network designed specifically for this type of task. It consists of five convolutional layers followed by four fully connected layers, with ELU activations and 20% dropout in the first two fully connected layers for regularization.

The output is a two-value vector: (v, w), which is sent directly to the simulator without any post-processing.
One advantage of this architecture is that the size of the fully connected layer is automatically calculated based on the input size, so changing the image resolution does not require modifying the network code.

The network was trained using the following settings:

| Parameter               | Value                                   |
|:------------------------|:----------------------------------------|
| Optimizer             | Adam                                      |
| Initial learning rate   | 1e-4                                    |
| Batch size              | 128                                     |
| Epochs                  | 10                                      |
| Scheduler               | ReduceLROnPlateu(patience=3,factor=0.5) |

One of the most important decisions in training is the choice of loss function. I chose a weighted MSE as the final loss function, assigning a weight of 3 to w relative to v:

> Loss = mean(abs(v_pred − v_real) + 3 · abs(w_pred − w_real))

Greater weight is given to w because the angular velocity value is more critical for following the line. I set it to 3 experimentally, since with lower values the network prioritized v, and with higher values convergence slowed down. I chose MAE over MSE because in autonomous driving, a small constant error is preferable to an error that is nearly zero but has occasional spikes, and MAE handles both cases more equitably.

A major issue I encountered regarding training speed was that I was loading the dataset from an HDD. With 4 DataLoader workers, the disk reached 100% utilization, and the GPU remained underutilized (peaking at 36% compared to the expected 80–90%). The solution was to move the dataset to an NVMe SSD and set `pin_memory = True` in the DataLoaders (to speed up the transfer from CPU to GPU), which significantly reduced the time per epoch.


## 4. Results

After applying the same preprocessing to the inference script in Unibotics (so that it receives input just as it did during training) and applying the model’s output directly to v and w, we obtain the following results:

<div style="text-align:center">
  <iframe width="700" height="394"
  src=“https://www.youtube.com/embed/2dQDfQh3CVI”
  frameborder="0" allowfullscreen>
  </iframe>
</div>


As can be seen in the video, the car is able to complete two full laps without any issues, indicating that the network is functioning correctly. I wasn’t able to test the Montreal track because it wouldn’t load, and on the Montmeló track, there’s a moment where the car crashes when encountering two consecutive turns. I suspect this is because there’s a straight section between the turns where the car picks up too much speed and isn’t able to slow down in time or correct its trajectory for the next turn.


## 5. Conclusions

This exercise was one of the simplest, since the dataset was well-balanced and contained a large amount of data; furthermore, the problem statement itself provided the PilotNet model to use as a baseline. That doesn’t mean it isn’t also interesting, since the vision learning approach allows for much faster inference with quite decent results.

Surely, it could be trained for more epochs or data augmentation could be applied to refine those tricky curves that it struggles with at Montmelo, but considering that it is a simple model trained in just 10 epochs, the results are more than satisfactory.
