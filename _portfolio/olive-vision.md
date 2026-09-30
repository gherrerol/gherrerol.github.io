---
title: "OliveVision"
excerpt: "System for the automatic quality assessment of olive shipments at a processing plant <br/><img src='/images/olives.png'>"
collection: portfolio
---


This project was developed together with **Tarek Elshami Ahmed** for the Master's in Computer Vision at URJC. OliveVision is a computer vision system for the **quality control of olive deliveries**: given a video of olives moving on a conveyor belt, it detects each olive, counts them, classifies them by ripeness and exports the results to a CSV file.

The problem starts at the mill. When a truck unloads a batch of olives, someone has to decide two things: whether the amount of olives matches the batch, and what the quality of that batch is. Today this is done by manual inspection, so the question we wanted to answer was whether vision could do it automatically.

![Problem](/images/olive_vision/problem.png)

Below, I will go through the pipeline step by step, the requirements we had to meet, the evaluation of the system, and the software design behind it.


## 1. Starting Point

The input is a video (MKV format) of olives of different colors going through a conveyor belt. It is a hard scenario for a detector: the olives overlap, they are all very similar in shape, the colors go from bright green to almost black, and there are branches and leaves mixed in with them.

<div style="text-align:center">
  <iframe width="700" height="394"
  src="https://www.youtube.com/embed/YOUTUBE_ID_OLIVEVISION"
  frameborder="0" allowfullscreen>
  </iframe>
</div>


## 2. The Pipeline

Every frame goes through the same five steps:

1. **Region of interest (ROI):** the frame is cropped to the area of the belt where the olives are, ignoring the borders.
2. **Cropped frame:** this is the image that the rest of the system works with.
3. **Detection:** a YOLO model detects every olive and returns a bounding box and a confidence value for each one.
4. **False positive filtering:** branches and other elements that are not olives are detected and discarded, so they do not end up in the count.
5. **Classification, counting and export:** each olive is classified by ripeness (green, purple or black), the olives are counted, and one row per frame is written to a CSV file.

![Pipeline](/images/olive_vision/pipeline.png)

The CSV has one row per processed frame, with the frame number, the total count, the count for each ripeness class and a timestamp:

```csv
numero_frame,conteo,madurez_verde,madurez_morada,madurez_negra,timestamp
1,126,51,29,46,21:31:27
2,128,43,28,57,21:31:27
3,126,54,14,58,21:31:28
```


## 3. Requirements

The project was developed against a set of functional (RF), non-functional (RNF) and interface (RI) requirements, each one with an acceptance criterion. All of them were met:

| Requirement                             | Acceptance criterion                                        | Status |
|:----------------------------------------|:------------------------------------------------------------|:------:|
| RF1 + RI1: MKV input                    | Cadence of 1.2 s/frame                                      | ✅     |
| RF2: Olive detection                    | IoU ≥ 0.7 on at least 1 olive                               | ✅     |
| RF3 + RNF1: Counting and accuracy       | Count error ≤ 10% (reference: a sample of 42 olives)        | ✅     |
| RF4: Ripeness classification            | Correct rate ≥ 90%                                          | ✅     |
| RF5: Element filtering                  | False positives on branches < 5% of the total detected      | ✅     |
| RF6 + RI3: CSV export                   | Correct header + 1 row generated per frame                  | ✅     |
| RI2: Invocation from C                  | Successful execution through `olivevision_invoke`           | ✅     |
| RNF2: Availability                      | No interruption while processing the test video             | ✅     |
| RNF3: Raspberry Pi                      | Full local execution with no dependency on an external network | ✅  |

Two of them are worth highlighting. The system has to run **locally on a Raspberry Pi**, with no dependency on an external network, and it also has to be **callable from C** through the `olivevision_invoke` function.


## 4. Detection Evaluation

To evaluate the detector I looked at the ROC curve and at the standard detection metrics.

![ROC curve and detection metrics](/images/olive_vision/roc_metrics.png)

| Metric    | Value |
|:----------|:-----:|
| Precision | 0.812 |
| Recall    | 0.641 |
| F1        | 0.716 |
| AUC (ROC) | 0.775 |

Precision is high, which means that most of what the system detects really is an olive. Recall is lower, so some olives are missed, most likely the ones that are partially hidden under other olives.


## 5. Ripeness Classification

The ripeness of each olive is classified into three classes: **green**, **purple** and **black**. The classifier analyzes the color of the region inside each detection.

![Confusion matrix](/images/olive_vision/confusion_matrix.png)

The confusion matrix (normalized by rows) shows that the classification is very reliable:

- **Green:** 100% correct.
- **Black:** 100% correct.
- **Purple:** 87.2% correct. The remaining 12.8% is classified as black.

The only confusion is between purple and black. A likely reason is that they are neighboring colors in the ripening process, so a dark purple olive can look very close to a black one.


## 6. Software Design

The system is documented with UML diagrams. It is made of small classes, each with a single responsibility, coordinated by a main class, `OliveVision`:

| Class              | Responsibility                                                       |
|:-------------------|:---------------------------------------------------------------------|
| `VideoReader`      | Loads the video (`cv2.VideoCapture`) and gives the frames one by one |
| `OliveDetector`    | Detects the olives with YOLO and filters the false positives         |
| `MaturityClassifier` | Classifies the ripeness of each detection by analyzing its color   |
| `OliveCounter`     | Counts the olives, in total and by ripeness                          |
| `ResultsExporter`  | Writes the results of each frame to the CSV file                     |
| `Deteccion`        | Data class that carries the bounding box, the confidence and the ripeness |

### Class diagram

![Class diagram](/images/olive_vision/class_diagram.png)

### Sequence diagram

For each frame of the video, `OliveVision` asks the reader for the frame, the detector finds the olives and filters the false positives, the classifier assigns the ripeness, the counter computes the totals and the exporter saves the row in the CSV.

![Sequence diagram](/images/olive_vision/sequence_diagram.png)


## 7. Development Metrics

The project was also measured as a piece of software:

| Module                 | Lines | Coverage |
|:-----------------------|:-----:|:--------:|
| `olive_detector.py`    | 187   | 85%      |
| `maturity_classifier.py` | 116 | 83%      |
| `OliveVision.py`       | 104   | 86%      |
| `results_exporter.py`  | 46    | 88%      |
| `video_reader.py`      | 47    | 82%      |
| `olive_counter.py`     | 32    | 89%      |

- **Total code:** 470 lines.
- **Global coverage:** 85%.
- **Documentation:** 15% of the code.
- **Unit tests:** 40 tests (about 400 lines of code).


## 8. Conclusions and Future Work

OliveVision replaces the manual inspection of a delivery with an automatic pipeline that counts the olives, classifies their ripeness and leaves a record of every frame in a CSV file. It meets all the requirements we set, and it does it running locally on a Raspberry Pi.

There are still several things that could be improved:

- **Separation on the belt:** if the olives came separated, there would be much less overlapping, which would improve the recall.
- **More labeled data:** to train the detector with more examples, especially of partially hidden olives.
- **Real camera:** the system has been tested with recorded video, so the next step is to try it with a camera installed on a real belt.
- **Olive tracking:** following each olive across frames, so that the same olive is not counted more than once.
