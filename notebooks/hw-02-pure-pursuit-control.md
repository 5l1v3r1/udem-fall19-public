# Hardware Exercise 2: Pure Pursuit Control

We've made some changes to the repo, so we're going to do some local updates first. 


## Update your repo

Start by pushing all your local changes to be safe. 

If you are using a fork, then you are going to need to pull from the upstream remote. To do this first create the upstream:

    $ git remote add upstream https://github.com/duckietown-udem/udem-fall19-public.git

then pull from the upstream

    $ git pull upstream master --recurse-submodules
    


## Test to see if things are working - Run the lane following demo in the sim

from the `udem-fall19-public` directory run:

    $ docker-compose build
    $ docker-compose up
    
You should see now that three containers are starting

```
Starting udem-fall19_sim_1   ... done
Starting udem-fall19_novnc_1        ... done
Recreating udem-fall19_lanefollow_1 ... done
```

Note that thesimulator is now running in its own container. 

You can open the notebook just like before by copying the url that looks like:

    http://127.0.0.1:8888/?token=<aaaaannnnn123123123123....>

into your browser. 

You no longer need to build the `custom_ws` only the `catkin_ws`:

    $ catkin build --workspace catkin_ws
    $ source catkin_ws/devel/setup.bash
    
If this is your first time it will take ~5 mins or so but since we mount things the build artifacts are preserved.
You also need to manually start the "car interface" which runs the inverse kinematics.
You can do this with:

    $ ./launch_car_interface.sh

Now you can run any demo in the duckietown codebase. Let's test out one that you have experience with already:

    $ roslaunch duckietown_demos lane_following.launch

which is the launch file that is run when you executed `dts duckiebot demo --package_name duckietown_demos --demo_name lane_following` in the last hardware exercise (now you can see what the `package_name` and `demo_name` flags do...).

You should see the nodes launch. 

### Visualization and Debugging

We've recently changed the workflow for viewing the output to avoid having to use X. Now we are going to use `noVNC`. In your browser, enter the url `http://localhost:6901/vnc.html`. This should take you to a login page. The password is `vncpassword`. 

Once inside, you have pretty functional environment where you can launch windows inside the browser. Under "Applications" in the top left you can open a terminal. In the terminal you can run `rqt_image_view` just like before and choose an output to view. 

**NOTE* if you want to see some outputs (e.g., the `image_with_lines`) then you need to set the verbose flag to true by typing: 

    $ rosparam set /default/line_detector_node/verbose true


You might also run `rviz` in the terminal and play around with the debugging outputs that you can add in the viewer. To do so click the "Add" button in the bottom left and then the "By topic" tab. You might find the `/duckiebot_visualizer/segment_list_markers/` and the filterer version as well as the `/lane_pose_visualizer_node/lane_pose_markers` particularly interesting.

### Making the Robot Move in Simulator

At this point, the virtual robot is not moving. Why?

To remedy the situation you will need to use the virtual joystick just like you did with the real robot. Run on your laptop:

    $ dts duckiebot keyboard_control default --network <network_name>  --sim [--cli] --base_image duckietown/dt-core:daffy
    
where you can find the `network_name` by running `docker network ps` and look for the right one. It's probably something like `udem-fall19-public_duckietown-docker-net`. The `--cli` is optional but for example if you are running on Mac then the gui is not supported. 
Now you should be able to use the keyboard to make the duckiebot move and also start it doing autonomous lane following. 

We have now completely reproduced the lane following demo but in the simulator. 


## Create a repo for your packages the template repo

Now we are going to discuss how you might build a new node and incorporate it into the structure of an existing demo. You will be able to test the performance first in the simulator and then try it on the robot. 

Follow the instructions [here](https://github.com/duckietown/template-ros-core) to create a new repo for your packages. Clone this repo inside the folder `udem-fall19-public/catkin_ws/src`. You can now add packages inside the `packages` folder of that repo. 

Any packages that you put in there will be built when you run:

    $ catkin build --workspace catkin_ws

in the notebook terminal. 

If you have added a new launch file then you can launch it from the notebook terminal with:

    $ roslaunch <your_package_name> <your_launch_file_name>


A good way to build your own launch file might be to figure out what's actually happening in the `lane_following.launch` file in duckietown_demos in dt-core and use this as a template to create your own thing. I.e., turn off the "args" that correspond to the new nodes that you don't want to run and then add your own includes or node launching code after. 

## Trying your code on the robot


Once you are happy with the operation of your new package, you can try it on the robot. Go into your repo's base directory (the one you made from the template) and run:

    $ dts devel build --push

This will wrap your code into a docker image that inherits from the `dt-core` repo (so it has all of those packages if you are using them in your launchfile).

Then you can run it on the robot through the same API as the lane following demo:

    $ dts duckiebot demo --demo_name <your-launch-file-name> --package_name <your-package-name> --duckiebot_name <your-duckiebot-name> --image duckietown/<repo-name>:<branch-name>
    
where `<repo-name>` is the name of the repo that you created from the template and `<branch-name>` is the branch that you are working on in your repo. 

when you run it might hang for a bit at the line: ```INFO:dts:Running command roslaunch <your-package-name> <your-launch-file-name>.launch veh:=<your-duckiebot-name>``` since it is pulling the image. You can also manually pull the image before running to see the status. 



## Your Task for this Exercise


In this exercise we are going to replace the existing PD controller with a pure pursuit controller on the robot. 

One issue discussed in class with respect to implementation of the pure pursuit controller is that it requires a reference _trajectory_ rather than just a reference value with which we can compute the tracking error. 

In order to implement the pure pursuit controller, we need to be able to calculate \alpha. 

I propose that we can calculate an esitimate of \alpha directly from the line detections projected onto the ground plane. 

The output of the ground projection node provides ground plane endpoints (in the robot frame) of the lines that are detected, along with their color. 

The algorithm to follow should be roughly the following:

 1. Filter the line detections to find the white ones and yellow ones that are "close" to your lookahead distance L
 2. Use these line detections to calculate an estimate of \alpha (I suggest to average between the white and yellow ones)
 3. Encapsulate this in a node, wire things up,  and try it on the robot. 

We have provided a function that filters the lane detections and publishes only the inliers. This might be a better choice for this algorithm. It is published by the `lane_filter`. 



Note: I'm not really sure if this will actually work or not... let's see

