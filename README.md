# Occlusion-Aware 3D Object Tracking and Robotic Manipulation

This is an RGB-D perception and robotic manipulation pipeline for detecting, tracking and picking objects from tabletop scenes.

The focus of this project is 3D object tracking when objects become partly hidden, move close to each other, or are viewed while the camera is moving. This project also works on how sensor noise, calibration errors and uncertain object positions affect tracking and grasp planning.

It contains RGB-D perception, 3D tracking, uncertainty estimation, grasp planning and robotic simulation. I have also included comparisons between tracking methods, tests under difficult conditions and evaluation on real RGB-D datasets.



## Key Results

| Evaluation                  |                 Result |
| --------------------------- | ---------------------: |
| Final tracker IDF1          |                  0.982 |
| Held-out simulated scenes   |                    120 |
| Pick-and-place success rate |                  88.6% |
| Successful placements       |  31 out of 35 attempts |
| Median placement error      |                 4.8 mm |
| mOSD segmentation precision |                  0.979 |
| ROS 2 tests                 | 36/36 per distribution |

These results come from separate experiments. The manipulation results are from simulation whereas the real RGB-D dataset tests focus on perception and geometry.

## 1. System Overview

The pipeline has five main parts:

1. **RGB-D perception:** Object detection and 3D position estimation from colour and depth images.
2. **3D object tracking:** Matching objects across frames and maintaining their identities during movement and occlusion.
3. **Uncertainty estimation:** Measuring the reliability of position estimates and using it during tracking and grasp planning.
4. **Grasp planning:** Generating possible grasps and checking geometry, clearance, collisions and reachability.
5. **Robotic manipulation:** Testing pick-and-place tasks with a Franka Panda robot in MuJoCo.

## 2. RGB-D Perception

The perception pipeline uses colour and depth images to separate objects from the tabletop and estimate their 3D positions.

Main components:

- Depth filtering and noise handling.
- Table plane estimation.
- Object segmentation using colour and depth.
- Separation of objects that are close to or touching each other.
- 3D position estimation using camera geometry.
- Handling of missing and noisy depth measurements.

### Segmentation under depth noise

The adaptive segmentation method was also compared with a fixed threshold method.

Under 5x depth noise:

| Metric                 | Adaptive method | Fixed-threshold method |
| ---------------------- | --------------: | ---------------------: |
| Recall                 |           0.960 |                  0.926 |
| False alarms per frame |           0.005 |                  0.742 |

The adaptive method gave higher recall and much fewer false alarms in this test.

### Segmentation under low light

I also tested the effect of reducing scene brightness. Depth only segmentation was compared with a method that used both depth and colour information.

The results that we got show that adding colour information helped maintain segmentation quality as brightness changed. False alarms were also measured to check whether objects were being detected where none were present.



## 3. 3D Multi-Object Tracking

The tracking pipeline uses:

- Kalman filtering for motion estimation.
- Hungarian assignment for matching detections to existing tracks.
- Image-space IoU and centre distance.
- Camera-space and world-space 3D distance.
- Appearance information for object matching.
- Re-identification after temporary occlusion.
- Track states for managing new, active and lost objects.

### Tracking methods

Five association methods were compared:

1. Image-space IoU.
2. Image-space centre distance.
3. Camera-space 3D distance.
4. World-space 3D distance.
5. Truncation-aware world-space distance.

The evaluation covered four scenarios:

- Nominal scenes.
- Occlusion.
- Camera motion.
- Crowded scenes.

The results show that image space matching can become less reliable during occlusion. World space information and truncation aware matching performed better in several difficult scenes.



### Tracker improvements

Different tracker versions were tested to understand the effect of individual changes.

T18 adds appearance information to the association cost of T14.

Results on 120 held out random simulated scenes:

| Metric                      | Earlier pipeline | Final T18 |
| --------------------------- | ---------------: | --------: |
| IDF1                        |            0.974 |     0.982 |
| Identity switches per scene |             1.18 |      1.09 |

The IDF1 improvement was statistically significant after Holm correction (p = 0.048). The change in identity switches was not statistically significant (p = 0.108).

Scripted difficult cases were also tested, including crossing objects, similar-looking objects, long occlusion, different speeds, sudden direction changes and irregular frame timing.

### Handling irregular frame timing

Tracking with a fixed assumed frame period was compared with tracking that uses actual frame timestamps.

When frames were dropped, using the actual timestamps helped maintain higher IDF1 and reduced identity switches. This is useful for camera streams where frames may not arrive at regular intervals.



## 4. Sensor Noise

The effect of sensor problems on object detection and tracking was tested under different conditions:

- Different levels of depth noise.
- Flying pixels.
- Speckle noise.
- Missing depth values.
- Changes in scene brightness.
- Camera movement.
- Crowded scenes.

Tracking performance was measured as depth noise increased. The results show that performance remained strong at lower and moderate noise levels, but started to decrease at higher noise levels.



## 5. Uncertainty Estimation and Calibration

A position estimated from an RGB-D camera is not always equally reliable. Depth noise and camera calibration errors can affect the estimated 3D position.

The pipeline includes uncertainty estimation to measure the reliability of position estimates.

The evaluation includes:

- Measurement covariance.
- Calibration uncertainty.
- Normalised Innovation Squared (NIS).
- Normalised Estimation Error Squared (NEES).
- Empirical covariance coverage.

### Calibration experiment

| Test condition                              | 95% covariance coverage |
| ------------------------------------------- | ----------------------: |
| Correct calibration                         |                   0.974 |
| Calibration changed, but not reported       |                   0.524 |
| Calibration changed and covariance supplied |                   0.986 |

When calibration was changed without informing the pipeline, the reported uncertainty no longer matched the actual errors. Supplying the calibration covariance brought the coverage close to the expected 95%.

Uncertainty is also considered during tracking and grasp planning, so that decisions can account for the quality of position estimates.



## 6. Grasp Planning

After estimating object positions, the pipeline generates possible grasps for a parallel-jaw gripper.

The grasp planner considers:

- Object geometry and position.
- Grasp position and orientation.
- Distance from object edges.
- Gripper clearance.
- Collision checks.
- Robot reachability.
- Uncertainty in the object position.

### Handling object movement and uncertain placement areas

The grasp planner was tested when an object moved after a grasp had already been planned.

For moved objects, the failure rate among kept plans changed from 0.492 with a fixed limit to 0.211 with the updated rule.

Placement availability was also tested. Before the updated rule, 55.8% of low object-occupancy spots were incorrectly marked as free. After the update, this fell to 0.3%.

## 7. Pick-and-Place Simulation

The perception and grasp planning pipeline was connected to a MuJoCo simulation using a Franka Panda robot.

The simulated workflow includes object detection, tracking, grasp selection, picking and placement.

### Manipulation results

The evaluation covered 46 held-out simulated trials.

| Metric                         | Result |
| ------------------------------ | -----: |
| Total trials                   |     46 |
| Trials declined by the planner |     11 |
| Placement attempts             |     35 |
| Successful placements          |     31 |
| Placement success rate         |  88.6% |
| Median placement error         | 4.8 mm |
| Trials with unwanted contact   |      2 |

The 88.6% success rate is based on 31 successful placements out of 35 attempts. The 95% Wilson confidence interval is 74.0–95.5%.

Declined trials are included in the evaluation because the planner should be able to reject a grasp when the conditions are not suitable.



## 8. Evaluation on Real RGB-D Datasets

Parts of the perception pipeline were also tested on real RGB-D data. These tests help check how the methods behave outside simulation.

### Object Segmentation Database (mOSD)

The segmentation pipeline was evaluated on 110 real scenes at 320 × 240 resolution. The test set contained 65 scenes.

| Scene type          | Clean recall |
| ------------------- | -----------: |
| Boxes               |        1.000 |
| Stacked boxes       |        0.857 |
| Occluded objects    |        1.000 |
| Cylindrical objects |        1.000 |
| Mixed objects       |        0.938 |
| Complex scenes      |        0.874 |

Different methods for separating objects that touch each other were also compared.

| Method    | Merged segments |
| --------- | --------------: |
| Baseline  |              74 |
| Variant A |              58 |
| Variant B |              18 |

For Variant B:

- Recall: 0.968.
- Clean recall: 0.922.
- Precision: 0.979.
- Merged segments: 18.
- Duplicate segments: 8.



### TUM RGB-D

The TUM RGB-D dataset was used to test camera geometry and related parts of the pipeline.

In the `fr1/xyz` sequence, the standard deviation of the estimated table height changed from 53.2 mm in the first stage to 6.4 mm in the last stage.

The timing differences between colour, depth and motion-capture data were also investigated. These streams were not always synchronised. Timestamp offsets and different camera geometry choices were tested.



## 9. Runtime Evaluation

Runtime was measured at 320 × 240 resolution using one CPU core.

| Scenario      | Perception p99 | Planning p99 |
| ------------- | -------------: | -----------: |
| Nominal       |        48.3 ms |       1.0 ms |
| Crowded       |        41.4 ms |     193.4 ms |
| Pick scenario |        51.6 ms |     246.0 ms |

At 15 FPS, each frame has about 66.7 ms available. Perception stayed below this limit in these tests. Planning was measured separately and took longer in crowded and pick scenarios.



## 10. ROS 2 Integration

A ROS 2 package was developed for the RGB-D tracking workflow.

The package includes:

- Synchronized RGB-D subscriptions.
- TF2 coordinate transforms.
- Configurable object tracking.
- 3D detection and tracking publishers.
- Integration with the robotics workflow.

The package passed 36 out of 36 tests on both ROS 2 Jazzy and Humble.

MoveIt 2 was also used for the motion-planning workflow.

## 11. Tools and Technologies

| Area            | Tools and methods                                        |
| --------------- | -------------------------------------------------------- |
| Programming     | Python, C++                                              |
| Computer vision | OpenCV, NumPy, SciPy                                     |
| 3D perception   | RGB-D processing, camera geometry, point clouds          |
| Tracking        | Kalman filter, Hungarian assignment, appearance matching |
| Uncertainty     | Covariance estimation, NIS, NEES                         |
| Robotics        | ROS 2, TF2, MoveIt 2                                     |
| Simulation      | MuJoCo, Franka Panda                                     |
| Evaluation      | Statistical testing, ablation studies, runtime profiling |

## 12. Limitations and Future Work

The next steps are:

- Testing the complete pipeline on a physical robot.
- Improving tracking during heavy occlusion and fast object movement.
- Testing uncertainty estimation with more real RGB-D cameras.
- Reducing planning time in crowded scenes.
- Improving separation of objects that touch each other.
- Testing grasp planning with more object shapes and sizes.
- Improving the connection between perception, planning and robot execution.

## 13. Reproducibility

### Installation

```bash
git clone <your-repository-url>
cd rgbd-object-tracking

python -m venv .venv
```

Activate the environment:

```bash
# Linux / macOS
source .venv/bin/activate

# Windows
.venv\Scripts\activate
```

Install the dependencies:

```bash
pip install -r requirements.txt
```

### Run the notebooks

```bash
jupyter notebook
```

Some experiments use external RGB-D datasets. Follow the dataset instructions and place the data in the paths specified in the experiment configuration.
