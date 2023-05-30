# RMIT COSC2814 PAR'23 ROS Navigation & Search Challenge

## Overview

This report details the work achieved by Group 11 in the ROS Navigation & Search Challenge.

The main challenges of this assignment required navigation within an unknown space, classification and localisation of hazard signs, and the production of a map that reflects the environment.

It details the design choices we made, our code structure and package usage, our evaluation on its performance during the assessment and a reflection on the adaptions needed to construct a stronger solution.

Tracking Unknown Environment |  Map Production           | 1 of 12 Detectable Hazard Signs
:---------------------------:|:-------------------------:|:-------------------------:
![](https://github.com/archmod/PAR_Search_And_Rescue/blob/e5e52bbbf1b730ba69a00d8610ea2921bca00ef0/wiki_media/detect_hazard_midturn.gif)  |  ![](https://github.com/archmod/PAR_Search_And_Rescue/blob/be4fc55b184ba44455c2fe9994246011cc4203d5/wiki_media/run2_map_generation.png) | <img src="https://github.com/s3782095/par_search_and_rescue/blob/216a313fec01c24c019dbe3df04a6d403d795541/images/markers/hazard/placard-7-radioactive.png" width=400  >

## Sections

1. [Description of ROS Package](#Description-of-ROS-Package)
2. [ROS Nodes](#ROS-Nodes)
3. [Analysis and Evaluation of the Package](#Analysis-and-Evaluation-of-the-Software)

## Team Members

* Max Foord - s3888349@student.rmit.edu.au - s3888349
* Joshua Barry - s3718861@student.rmit.edu.au - s3718861
* Mitchell Gust - s3782095@student.rmit.edu.au - s3782095

## Description of ROS Package
### Overview
In this section we will go over in depth the architecture of our package and submission breaking down the packages used and why their inclusion was necessary for the project. 

Our ROS Package employs a minimal range of external/additional packages. This is due to the given time constraints and the fact that in the real world simpler tends to be better. A simple solution was the one strived for and a simple solution is what we got. 
ROS Packages Used

### MoveBase
MoveBase was used after seeing its success with other teams in the competition. We experimented using this package to explore the map through the use of ExploreLite however this method of exploration was found to be far less reliable and effective than compared to a standard wall follower.

### GMapping
GMapping was used to create and publish the map from our rosbot. It takes a range of sensory data primarily from the LIDAR to accurately publish the map that is used by MoveBase and our object placement nodes.

### FindObject2D
FindObject2D was chosen as our method to perform sign recognition. This is a semi lightweight package which worked efficiently for our purposes providing us with extensive configuration and tuning. Little was changed from the stock configuration as out of the box this package worked very efficiently. The only challenge that was faced with this package was attempting to negotiate the identification of 2 signs within the same frame. 

### Rospy
From ROSPy we used a number of the standard message libraries including but not limited to: Marker, MarkerArray, Twist and others.

## ROS Nodes
### Follower3

The follower node is our main navigation node for our project and handles all the movement of the bot from the flag to start the challenge until the final return home flag is issued. Initially, the navigation algorithm that we chose was a simple right hand wall follower, however given that there was the potential for there to be islands within the space we must explore, we settled on exploring a frontier exploration algorithm.

The structure of the proposed frontier exploration algorithm was to have a node that generated the frontier points to the `/generate/frontier` topic, and then run another node that would calculate the closest frontier point to the current robot position by Euclidean distance, and then using the move_base package, navigate to that point. Once at the point, a new point from the frontier will be chosen to continue the exploration.

The frontier generating algorithm used the occupancy grid received from the `/map` topic published to by gmapping. The single array grid is then converted to a 2-dimensional array. Then, for each grid space, if that grid space is unknown, the neighboring cells around it are checked to see if they have already been explored. In the case that these neighboring cells have been explored, they are added as a pose to the frontier and then once the full occupancy grid has been iterated through, the list of frontier poses are published to the frontier topic.

#### Issues Encountered

Unfortunately, the poses generated by the frontier were not working correctly when sent to move base. The x and y coordinates that were being generated were either very large or very small depending on the coordinate calculation done.

One explanation for this could be that the config params for the move base package may have not been appropriately configured for the coordinates given, or potentially the coordinates generated themselves were always in untravelable parts of the map that move base was unable to reach.

As such, we went back to the wall-following algorithm that we had originally chosen. Our wall follower is set up as a state machine that dictates the allowed movement choices throughout the run.

#### Wall Following Algorithm

Upon receiving the start symbol, the bot will check whether there already exists a wall to the right within the distance thresholds, and if it does, it will go straight to follow that wall. In the event that there is either no wall detected or the wall is outside the distance thresholds, the bot will make approximately a 90-degree turn to the right and drive until the wall is within the thresholds of the bot. It will then oscillate between turning left if the wall on the right is too close, driving straight if the wall is within the acceptable distance window, and turning right if the wall is outside the threshold. When a wall is detected to within the minimum threshold in front of the bot, it will turn left and continue the wall following oscillations. This will continue until the return home signal is received.

### Recognise_node

Recognise Node is responsible for recognising hazard signs and publishing them to the /hazards topic. Initially it was publishing these items as a MarkerArray but this was changed last minute to abide by the rules of the competition. This was why during the first attempt at the maze the robot failed to publish the hazards it recognised. 

Recognise node publishes the start marker upon detection to the ‘/start_marker’ topic. This topic applies logic to Follower3 which waits for the start_marker topic to publish a ‘1’ to denote that a start marker has been detected. 

Another critical function of this node is the identification of hazard markers. The way in which this was performed was through the use of the Find Object 2d package which implements OpenCV. The model was trained through the use of manually taken photos from the ROSBot 2 Pros camera. Angular photos were not used in the final product but were experimented with only to find the results inadequate without further time investment. An enum was created to allow for additional readability instead of just having the ID’s of each hazard marker. The ‘callback’ function is called upon a publishing of new messages to the ‘/objectsStamped’ topic which is being published to by FindObject2D. We only consider objects that were detected above a certain confidence threshold that was tuned to be a value of 0.5. After a successful identification we convert the position of the center of the detected image to a LIDAR position by converting the angle of the location of the image to a lidar position. This results in an accurate placement of the recognised image on the map. 

### Path_node
Path Node is responsible for publishing the starting position. This node was factored out to try and reduce the amount of responsibility on the Path Recorder Node. This node works by waiting for a transform of the robot's position and publishing it to the ‘/start_position’ topic. 

### Path_recorder_node
Path recorder nodes function is as the name entails, to record the robots path. It publishes a path type message to the ‘/path’ topic. Inside of the ‘record_path’ function we read the ROSBots transform buffer and grab the most recent transform of the robot's position. We use the lookup transform and convert the robots position to map position and then publish to the ‘/map’ topic.

### Return_home_node
Return Home Nodes primary function is to return the robot back to its starting position. The way in which it performs this task is through a simple implementation of MoveBase. The return home node takes a duration as its only constructor parameter, this was set to a value of 360 seconds to allow for more than ample time for the robot to navigate back to its original position, 90 seconds total time in order for this operation to take place. This node also also listens to Path_node awaiting for the marker that gets published to ‘start_position_pub’.

## Analysis and Evaluation of the Software

### Summary of Results

> Run 1
> -----
>
> Our Run 1 did not perform as expected and is not as important for our overall evaluation as there was a lot of elements of our package that were not able to be tested. However, the run definitely triggered modification to our package. Our run 1 failed as we mistakenly published information to the wrong topic for our hazard identification and path generation. This led us to understand how minor details can cause significant influence over robotics performance in a test environment.
>
> Though we could not test our path and hazard identification, run 1 allowed us to evaluate the ROSbot's movement, in particular it's ability to traverse around the end of a wall. 
>
> As shown in the generated map, our package did not allow traversal out of the first found area and required modification for the following demo. 
>
> <img src="https://github.com/archmod/PAR_Search_And_Rescue/blob/b084e45031a6d162f51215b0a2248465f65f0ace/wiki_media/run1_generated_map.png" width=15% height=15%>
>
> By having a wall follower, and a wall follower that is structured by explicit methods (ex. first_wall(), follow_wall(), back_to_wall(), turn_left(), etc.), it was easy for us to make adjustments to the angular movement, from `-0.3` to `-0.5`.
>
> Adjustment shown below: 
>
> ``` python
> def back_to_wall(self):
>   self.make_move(0.1, **-0.5**)
> ```
>
> ``` python
> def make_move(self, linear, angular):
>  msg = Twist()
>  msg.linear.x = linear
>  msg.angular.z = angular
>  self.pub.publish(msg)```
> ```
>
> This change caused harsher turns in Run 2, that led to complete map exploration.
>
> Also, the adjustments to ensure we were publishing to the correct test topic is shown below:
>
> <img src="https://github.com/archmod/PAR_Search_And_Rescue/blob/be4fc55b184ba44455c2fe9994246011cc4203d5/wiki_media/fix_topic_publishing.png" width= 60% height= 60%>
>
> Run 2 
> -----
>
> **Navigation**
> * Traversed Each Wall that was Not Part of an Island
> * Corrected Itself after Over and Underestimating Turns
> * Maintained a Safe Distance from the Wall
> * Led to a Complete Version of the Map
>
> **Image Detection**
> * Detected 3 out of the 4 Hazards that it came in contact with
> * Failed to Detect a Hazard that was Along a Wall the ROSbot Traversed
> * Able to Detect Hazards Where it Had Vision of the Front of the Hazard (During Turning or Forward Movement)
> 
> **Path Generation**
> * Generated a Complete Path that was Accurate to the ROSbot's positioning
> 
> **Map Generation**
> * Generated a Complete Map that was Accurate to the Test Environment of the Demo
> 
> **Hazard Marking**
> * Initially Identified the Hazards to be in the Correct Positioning
> * Due to Latency Between the `findobject2d` callback and the Lidar Reading, Incorrect Positioning of Hazards were made Post Initial Identification

### Positives
**Flexible Application - Wall Following**

The most valuable strength of our ROS package was its ability to navigate in an unknown environment without question. Our constructed state machine led the robot to follow its right hand regardless of the map variance. Though it may not produce a complete map in all environments, our navigation system will always ensure partial map completion without robot failure. Here we can see the benefit of simplicity and the use cases where wall following is preferable over complex frontier exploration. Our solution does not depend on live calculations on potentially corrupt or noisy maps, and is very unlikely to lead ROSbot to a detrimental state.

<img src="https://github.com/archmod/PAR_Search_And_Rescue/blob/e5e52bbbf1b730ba69a00d8610ea2921bca00ef0/wiki_media/follow_wall.gif" width=50% height=50%>

**Response to Imperfect Movement - Minimum Wall Distance Threshold**

Though we did not have time to tune the parameters of the robot’s movement to perfection - for example, it can be seen over and under estimating the degree at which it should turn - the navigation node is well suited to responding to unexpected movement.
It does this by ensuring its distance to a wall is its first consideration when publishing a `Twist()` command. That way, regardless of how unpredictable its movement is within the space, when it gets close to static obstacles, the minimum threshold will come into play and restrict movement - acting like a cost map.

<img src="https://github.com/archmod/PAR_Search_And_Rescue/blob/e5e52bbbf1b730ba69a00d8610ea2921bca00ef0/wiki_media/correct_incorrect_turns.gif" width=50% height=50%>

**Quick and Accurate Hazard Detection**

From launch, we can see how quickly our visual detection was able to accurately predict the first marker in the slight time it had whilst completing the turn.
We have assessed the risk of integrating colour extraction and performing greater computation on the hazard identification process. However, we found the time of computation needed to achieve greater confidence in identification will result in less hazards being found in the long run as it can not identify hazards whilst moving - requiring the robot instead to pause for long periods of time.
Following our tests, we believe that focusing on black and white image data is accurate enough for the contest environment.

<img src="https://github.com/archmod/PAR_Search_And_Rescue/blob/e5e52bbbf1b730ba69a00d8610ea2921bca00ef0/wiki_media/detect_hazard_midturn.gif" width=50% height=50%>

Also shown is the quick and successful detection of the start marker.

<img src="https://github.com/archmod/PAR_Search_And_Rescue/blob/6f6053a5ac1f266b77b5f128ea4ef0b2c98170de/wiki_media/run_on_start_marker_shortned.gif" width=50% height=50%>

**Return to Home**

Our package successfully stored the starting point location, and returned within millimetres - even factoring in orientation. Our implementation was successful, as the return to home command was registered as a callback of a raspy timer. This way regardless to the successful detection of hazards, the ROSbot would utilise the remaining 20% of time allocated to the demo to returning home. 

<img src="https://github.com/archmod/PAR_Search_And_Rescue/blob/6f6053a5ac1f266b77b5f128ea4ef0b2c98170de/wiki_media/return_to_home.gif" width=50% height=50%>

**Localisation and Path Record**

As shown on our generated map, our package is able to correctly record our ROSbot's position. By reading the transformation from map to base_link, we can convert the information to a PoseStamped() message as the robot traverses the environment, and append and publish our latest path to the /path topic.

<img src="https://github.com/archmod/PAR_Search_And_Rescue/blob/be4fc55b184ba44455c2fe9994246011cc4203d5/wiki_media/run2_map_generation.png" width=15% height=15%>


### Negatives

<img src="https://github.com/archmod/PAR_Search_And_Rescue/blob/e5e52bbbf1b730ba69a00d8610ea2921bca00ef0/wiki_media/run_maps_illustration.png" width=25% height=25%>

It may go without saying but by utilising a wall following algorithm our navigation system can **not** perform well when placed in maps that have <ins>islands</ins> or <ins>large open spaces</ins>. 

**Islands**

We open the conversation by looking at Map 1. 
As we know our goal is to detect hazards, and these hazards can only be found on walls. 
Knowing this and knowing Map 1 does not exhibit islands, our navigation system will allow us to attempt identification of every hazard.

However, when compared to Map 2, the inclusion of an <ins>island</ins> prevents our Navigation system from being able to traverse all walls - and therefore eliminates the potential to locate all hazards.  

In order to rectify islands being left untraversed, our implementation could record visited walls and utilise a move based approach to jump between islands whilst performing wall following.

**Large Open Space**

We can see by our robot’s generated visual representation of Map 2 that there was no loss in detail of the explored environment.
This was due to the map having minimal space between walls, however this may not always also be the case.

Map 3 describes a case whereby the space between walls is extensive  <ins>(large open space)</ins> and the laser scanner can not interpret the environment accurately whilst following the wall.

This unmapped area could contain goals and without adaptation to our approach, could result in an unsuccessful exploration. 
To counter this issue, we would need to implement frontier exploration along with a move based approach to alert the robot of unexplored areas and prompt its discovery.

**Oblique Angles - Image Detection**

There is an instance in the search and navigation contest where our package fails to identify a hazard which is along a traversed wall.

Assessing the footage and acknowledging that the camera has a 60 horizontal degree FOV, there is a potential to perform better without modifying the navigation node.

In our training set for hazard detection, we have taught the model to identify signage when viewing from front on. To improve our solution however we could add additional training data that models the hazard signs from various perspectives. This way the robot would be more likely to detect the sign as it will be trained to register the arrangement of data points expected when viewing the hazard from an angle.

<img src="https://github.com/archmod/PAR_Search_And_Rescue/blob/e5e52bbbf1b730ba69a00d8610ea2921bca00ef0/wiki_media/missing_hazard_along_wall.gif" width=50% height=50%>

<img src="https://github.com/archmod/PAR_Search_And_Rescue/blob/e5e52bbbf1b730ba69a00d8610ea2921bca00ef0/wiki_media/hazard_sign_on_angle.png" width=30% height=30%>

**Hazard Maker Locations**

Though we are able to initially tag identified hazards with a plausible position. Due to Latency Between the findobject2d callback and the Lidar Reading, incorrect positioning of hazards were made post initial identification.

<img src="https://github.com/archmod/PAR_Search_And_Rescue/blob/918dbce72130c38255ead3251616180a6555d9e2/wiki_media/hazard_position_movement.gif" width=15% height=15%>
