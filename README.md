## Project: Search and Sample Return

---
***Resources:***
* [Setup Python and environments](https://github.com/ryan-keenan/RoboND-Python-Starterkit/blob/master/doc/configure_via_anaconda.md)
* Download the simulator [[Mac](https://s3-us-west-1.amazonaws.com/udacity-robotics/Rover+Unity+Sims/Mac_Roversim.zip)/[Linux](https://s3-us-west-1.amazonaws.com/udacity-robotics/Rover+Unity+Sims/Linux_Roversim.zip)/[Windows](https://s3-us-west-1.amazonaws.com/udacity-robotics/Rover+Unity+Sims/Windows_Roversim.zip)]
* [Udacity's project repository](https://github.com/udacity/RoboND-Rover-Project)

**The goals / steps of this project are the following:**  

**Training / Calibration**  

* Run simulator in "Training Mode" and take data out by recording
* Test out the functions in the Jupyter Notebook provided
* Add functions to detect obstacles and samples of interest (yellow/golden rocks)
* Fill in the `process_image()` function with the appropriate image processing steps (perspective transform, color threshold etc.) to get from raw images to rover vision to a map.  The `output_image` created in this step demonstrates that mapping pipeline works.
* Use `moviepy` to process the images in the saved dataset with the `process_image()` function to create a video. 
* the video (can be found under `output` folder or in the Jupyter Notebook)is produced as a part of this submission.

**Autonomous Navigation / Mapping**

* Fill in and modify the `perception_step()` function within the `perception.py` script with the appropriate image processing functions to analyze rover vision, create a map and update `Rover()` data (similar to what I have done with `process_image()` in the notebook). 
* Fill in and modify the `decision_step()` function within the `decision.py` script with conditional statements that take into consideration the outputs of the `perception_step()` in deciding how to issue throttle, brake and steering commands. 
* Iterate on perception and decision function until the rover does a reasonable job of navigating and mapping.
  * map environment > 40%
  * fidelity (accuracy of create a map) > 60%
  * locate at least one rock sample
* The rover has 3 modes: Navigating, Turning around, and Go to rock sample (see more in `decision.py`)
* This mode is run by calling a command line:
```sh
python drive_rover.py
```


[//]: # (Image References)

[image1]: ./calibration_images/example_rock1.jpg 
[image2]: ./calibration_images/rock_trans.png

## [Rubric](https://review.udacity.com/#!/rubrics/916/view) Points
### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---
### Writeup / README

#### 1. Provide a Writeup / README that includes all the rubric points and how you addressed each one.  You can submit your writeup as markdown or pdf.  

You're reading it! This file is my Writeup / README.

### Notebook Analysis
#### 1. Run the functions provided in the notebook on test images (first with the test data provided, next on data you have recorded). Add/modify functions to allow for color selection of obstacles and rock samples.

To identify obstacle, just simply take anything that is not navigable. In other words, the obstacle terrain is an binary invert of the navigable terrain

```python
threshed_obstacle = 1 - threshed_navigable # binary invert of navigable vision
```

I have also add a function `color_thresh_rock()` in *Color Thresholding* procedure. This function will identify yellow rock sample by using [Changing Colorspaces](http://docs.opencv.org/trunk/df/d9d/tutorial_py_colorspaces.html) technique. Color yellow in HSV is [30,255,255]. Therefore, I set the lowerbound and upperbound to identify yellow object, then extract only a vision for the sample.
```python

lower_yellow = np.array([20,100,100])

upper_yellow = np.array([40,255,255])
```

*Image of rock sample*

![rock_img][image1]


![rock_trans][image2]

#### 2. Populate the `process_image()` function with the appropriate analysis steps to map pixels identifying navigable terrain, obstacles and rock samples into a worldmap. Run `process_image()` on your test data using the `moviepy` functions provided to create video output of your result.

In order to demonstrate my analysis and, I include original image, perspective transformed image, rover vision, where navigable/obstacle/rocks are identified, and the wordmap I had created. 

Rover vision is simply a superimposed vision of obstacle in red channel, rock in green channel, and navigable in blue channel.
```python
vision = np.zeros_like(img)
vision[:,:,0] = threshed_obstacle*255 # red
vision[:,:,1] = threshed_rock*255 # appear to be yellowish (red + green)
vision[:,:,2] = threshed_navigable*255 # blue
```

Worldmap is created by convert the rover vision to rover-centric coordinates. Then the coordinates together with yaw angles, I can rotate, translate, and scale pixels and then add them into worldmap

```python
data.worldmap[ypix_world_obs, xpix_world_obs, 0] += 1 
data.worldmap[ypix_world_rock,xpix_world_rock, 1] += 1 
data.worldmap[ypix_world_nav, xpix_world_nav, 2] = 255 
```
* Video output: https://www.youtube.com/watch?v=AKVu897lHUY

### Autonomous Navigation and Mapping

#### 1. Fill in the `perception_step()` (at the bottom of the `perception.py` script) and `decision_step()` (in `decision.py`) functions in the autonomous mapping scripts and an explanation is provided in the writeup of how and why these functions were modified as they were.

The perception step pretty much follows `process_image()` function in the Jupyter Notebook. However, I found a technique to improve fidelity. Fidelity is often dropped when the rover build up the worldmap from the far-away vision. This is because perspective transform is distorted when the distance (respect to the rover) is far away. Only near vision will accurately contribute to world map. Therefore, I decided to crop the vision before further processing and adding it to the worldmap.

```python
threshed_navigable_crop = np.zeros_like(threshed_navigable)
threshed_obstacle_crop = np.zeros_like(threshed_obstacle)
x1 = np.int(threshed_navigable.shape[0]/2)      # index of start row  
x2 = np.int(threshed_navigable.shape[0])        # index of end row
y1 = np.int(threshed_navigable.shape[1]/3)      # index of start column
y2 = np.int(threshed_navigable.shape[1]*2/3)    # index of end column
# crop from start to end row/column
threshed_navigable_crop[x1:x2, y1:y2] = threshed_navigable[x1:x2, y1:y2]
threshed_obstacle_crop[x1:x2, y1:y2]= threshed_obstacle[x1:x2, y1:y2]
```

I also add a 'goto_rock' mode on a rover. This mode will take priority over a normal navigation if the rover detect any rock sample. It will navigate directly to the sample instead of nagigable terrain. So, the output of perception step has to be different depended on the current mode, telling the rover what it is looking for.

For decision step, I have modified it to be more complex. I have added features, for instance, detecting states of being stuck and unstuck itself. The rover can also roaming more smoothly by setting throttle only when it is not turning. It will also slowing down when going to pick up sample to ensure that it does not pass over the rock sample.

#### 2. Launching in autonomous mode your rover can navigate and map autonomously. Explain your results and how you might improve them in your writeup. 

**Note: running the simulator with different choices of resolution and graphics quality may produce different results, particularly on different machines! **

My simulator setting:
* Screen resolution: 1600 x 900 , windowed
* Graphic quality: Fantastic

FPS on `drive_rover.py`: ~40 frames per seconds

Result from a run on autonomous mode:
* Mapped: 76.5%
* Fidelity: 71.7%
* Sample collected: 4
* Time: 278 s
* Youtube video of the run: https://www.youtube.com/watch?v=9LPGp1_dDsU&feature=youtu.be

The features I might added if I were to pursue this project further are making it not repeat the route that already mapped, a better decision making of turning(left or right) at the intersection, and detecting the rock obstacles and avoid them.


It Fufuengsin
