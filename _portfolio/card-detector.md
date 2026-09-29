---
title: "Poker Card Detection and Classification"
excerpt: "Program capable of reliably detecting cards from a poker deck and classify them accordingly <br/><img src='/images/cards.png'>"
collection: portfolio
---

This project was developed together with **Jorge Lozoya Astudillo** for the Master's in Computer Vision at URJC. The goal is to detect poker cards in an image or a video stream and classify each one by its rank (number) and suit.

The initial idea was to build a blackjack assistant. As the project moved forward, it turned into something more interesting: a **comparison of several classical computer vision methods** (Template Matching, binarized template subtraction, SIFT and ORB) for detecting and classifying cards, looking at how accurate and how fast each of them is.

![Idea](/images/card_detector/idea.png)

Below, I will go through the building blocks shared by all the methods, then each approach in its own section, and finally the results and the conclusions.


## 1. Common Building Blocks

Every method starts from the same preprocessing, so that the comparison between them is as fair as possible:

- **Contour detection:** finds the card in the image.
- **Rectification (warp):** a perspective transform that leaves the card frontal, at a fixed size of 250 x 350 pixels. This gives invariance to perspective, scale and orientation (as well as possible).
- **Region of Interest (ROI):** the top-left corner of the card, where the rank and the suit are printed. It is split into two crops of 42 x 58 pixels, one for the **rank** and another for the **suit**.
- **Normalization:** all the templates have the same resolution, so they can be compared directly.
- **Ground truth:** the templates are named `number_suit.jpeg` (for example `7_trebol.jpeg`), so the file name is also the label.

![Warp and ROI](/images/card_detector/warp_roi.png)
![Warp and ROI 2](/images/card_detector/warp_roi_2.png)


## 2. Template Matching

The most classical approach: compare the ROI against a library of templates and keep the best match.

1. Detect the edges of the card, warp it and convert it to grayscale.
2. Extract the ROI (crop and split into rank and suit).
3. Load the templates and compute the correlation with `cv2.matchTemplate`. The template with the highest correlation gives the prediction.

![Template Matching pipeline](/images/card_detector/template_matching.png)


## 3. Binarized Template Subtraction

A variation of the previous idea, but instead of correlating grayscale images, the ROI and the templates are binarized and compared by subtracting them.

1. Detect the edges of the card and warp it.
2. Extract the ROI (crop and split).
3. Compare it with every template using the **SAD** (Sum of Absolute Differences). The template with the smallest difference is the prediction.

![Binarized subtraction pipeline](/images/card_detector/subtraction.png)


## 4. SIFT + RANSAC

Here the classification does not depend on comparing pixels directly, but on matching **keypoints**, which makes it more robust to small changes in scale, rotation and lighting.

1. Detect the edges and warp the card.
2. Extract the features of the card and of every template with **SIFT**.
3. Match the features and use **RANSAC** to keep only the geometrically consistent matches (the inliers).
4. The template with the most inliers is the prediction.

![SIFT + RANSAC pipeline](/images/card_detector/sift.png)


## 5. ORB + Binarization

The fastest keypoint-based option. It uses **ORB** features combined with a binarization step, and it gave the **best real-time result** of all the methods I tried. Its weak spot was the color red, which is the main thing that still needs work (see the conclusions).

<div style="text-align:center">
  <iframe width="700" height="394"
  src="https://www.youtube.com/embed/YOUTUBE_ID_ORB"
  frameborder="0" allowfullscreen>
  </iframe>
</div>


## 6. Uncontrolled Environment

All the previous methods rely on a template library and a reasonably controlled scene. To test something different, I also tried a version that works with no templates at all:

1. Thresholding based on a percentage of gray levels.
2. Contour detection and white pixel detection.
3. Extraction of the ROI of the card.
4. Counting the areas found inside the card.

<div style="text-align:center">
  <iframe width="700" height="394"
  src="https://www.youtube.com/embed/YOUTUBE_ID_ORB"
  frameborder="0" allowfullscreen>
  </iframe>
</div>


## 7. Results

### Detection examples

These are some of the predictions made on real photos of the deck. It gets many of them right (8 of diamonds, ace of diamonds, 9 of clubs, 10 of diamonds and 4 of clubs), but it also fails in some cases: the ace of clubs was read as a 3 of diamonds, the king of clubs as a queen of clubs, and the 2 of clubs as a 2 of diamonds. Note that most of the failures involve the suit, which is a very small part of the image.

![Results](/images/card_detector/results.png)

### Best result (SIFT)

<div style="text-align:center">
  <iframe width="700" height="394"
  src="https://www.youtube.com/embed/YOUTUBE_ID_SIFT"
  frameborder="0" allowfullscreen>
  </iframe>
</div>

### Comparison

The three methods were run on the same video and I measured the total processing time and the classification accuracy:

| Method                         | Processing time | Accuracy |
|:-------------------------------|:---------------:|:--------:|
| Binarized template subtraction | 23.83 s         | 36.4%    |
| SIFT + RANSAC                  | 372.14 s        | 59.1%    |
| Template Matching              | 21.04 s         | 45.5%    |

![Comparison charts](/images/card_detector/metrics.png)

SIFT is clearly the most accurate, but it is also about 17 times slower than the other two, which makes it unsuitable for real-time use. The two template-based methods run in a similar time, and Template Matching gets better accuracy than the binarized subtraction. ORB is not included in these charts, since it was evaluated mainly for its real-time behavior.


## 8. Conclusions and Future Work

The main conclusions of the project:

- **SIFT** is the most precise, but also the most costly.
- **ORB + binarization** is precise and much faster, but it struggles with the color red.
- There are many possibilities to explore in this problem. It looked simple and quick to implement... and it was not.

There are still several things to improve or test:

- Try other decks, with different designs and fonts.
- Improve the suit detection in ORB.
- Detect several cards at the same time.
- Improve the templates and reduce the noise of the binarization.
