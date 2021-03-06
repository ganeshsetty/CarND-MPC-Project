
# Objective

This project is about implementing a Model Predictive Controller (MPC) to drive the car around a track in simulator.
The video link shows the car behaviour with MPC implementation https://youtu.be/yPwjIMOOvl8. The video shows at start some kind of instability. This is caused due to recording ,but without recording this behaviour is not observed.It is almost smooth start.

# Model

A simple vehicle model called global kinematic model is adopted for designing MPC.This model is simplification of dynamic model by ignoring gravity, tire forces etc.

### State

[x,y,ψ,v] is the state of the vehicle 

* x,y - position of vehicle
* ψ   - orientation in radians
* v   - speed of vehicle

[cte,eψ] Errors also included in state vector.

* cte  - Cross Track Error(error between the center of the road and the vehicle's position)
* eψ   - error in orientation(desired orientation subtracted from the current orientation)


### Actuators

[δ,a] are the actuators/control inputs

* δ   -  steering angle [-1,1] normalized
* a   -  acceleration[+1,-1] corresponding to throttle, -ve value means braking, where +1 is full acceleration and -1 is full brake.


### Update equations

* x(t+1) = x(t) + v(t) * cos(ψ(t)) * dt
* y(t+1) = y(t) + v(t) * sin(ψ(t)) * dt
* ψ(t+1) = ψ(t) + v(t)/Lf * δ ∗ dt
* v(t+1) = v(t) + a * dt

where Lf  - physical characteristic of the vehicle( distance between front of vehicle and CoG(Center of Gravity))

# Timestep Length and Elapsed Duration (N & dt)
T  is prediction horizon and is product of N(time steps) and dt(elapses between actuations).N, dt, and T are hyperparameters are tuned while building model predictive controller. Initially i started with N = 10 and dt = 0.05, but the actuation commands have latency of 0.1s resulted in car driving off the road within few timesteps. Then played around by tuning dt by increasing it to 0.15 i.e prediction horizon increased, resulted in better behaviour atleast could run laps by incorporating  penalty weights for cte and steer angles for higher speeds (100mph) but observed max speed achieved is approx 65mph and drastically speed reduced during turns. But to improve the performance the latency handling part is implemented where the predicted state (ie after latency 0.1sec)is fed for mpc solve function.
Finally best performace is obtained with N = 10 and dt = 0.1sec at reference velocity of 100mph. Achieved a max speed of 90 mph.
Also while tuning the prediction horizon T it is kept as 1 to 1.5 seconds since more than this duration the environment will change enough that it won't make sense to predict any further into the future.

# Cost function and penalties

Cost should be function of how far the error is from 0 in case of cross track error and orientation error.
Additional cost considerations are velocity error,control inputs, value gap between sequential actuations

        cost = cte^2 + eψ^2 + (v-ref_v)^2 + δ^2 + a^2 + (δ(t+1) - δ(t))^2 + (a(t+1) -a(t))^2

All cost terms in the above equation can be penalized with weighing factors. For example, to reduce cte error and orientation error, i have choosen a high value of 4000,2000 penalty factors respectively and it helped at high speed during turns, the cte considerably reduced with compromise in speed (during turns average of 50mph it could achieve) but keeping car well around center of track. To improve smooth steering during turns a weighing factor of 500 used for reducing steering values gap of sequential actuations.


# Polynomial Fitting and MPC Preprocessing

The simulator sends to MPC solver , six waypoints(ptsx & ptsy) , the vehicle x,y position(px,py), orientation and speed (mph). Before fed to MPC solver,waypoints are preprocessed by converting them to car coordinate system and transformed ones are polyfitted well with 3rd order polynomial equation. The initial state vector fed to solver i.e, position and orientation as zeros since coordinate system is wrt car. The cross track error is determined with polyeval function at x =0 i.e constant term of fitted polynomial and orientation error is first derivative of polynomial at x =0. The resultant state vector [0,0,0,v,cte,eψ] are fed to mpc solver which tunes the actuators for optimal trajectory.

# Model Predictive Control with Latency

In a real car, an actuation command won't execute instantly - there will be a delay as the command propagates through the system. In main.cpp, before sending the actuators commands to simulator a latency of 100ms is introduced as below

        this_thread::sleep_for(chrono::milliseconds(100));
        
But this latency has to be taken care before we feed state vector to MPC solver, i.e state should reflect the values predicted after 100ms. In the code, it is handled by update equations. 

          //predict state in 100ms latency
          px = px + v*cos(psi)*latency;
          py = py + v*sin(psi)*latency;
          psi = psi + v*(-delta)/Lf*latency; //steering(delta) is -ve of value received  from simulator
          v = v + acceleration*latency;

Then as usual the waypoints are converted to this car coordinate system as explained above.

After handling the latency , the MPC tunes very well the actuators (steering angle and throttle) that chooses optimal MPC trajectory by minimizing the cost function within the actuators constraints.





# Setting up enviorenment for project
# CarND-Controls-MPC
Self-Driving Car Engineer Nanodegree Program

---

## Dependencies

* cmake >= 3.5
 * All OSes: [click here for installation instructions](https://cmake.org/install/)
* make >= 4.1
  * Linux: make is installed by default on most Linux distros
  * Mac: [install Xcode command line tools to get make](https://developer.apple.com/xcode/features/)
  * Windows: [Click here for installation instructions](http://gnuwin32.sourceforge.net/packages/make.htm)
* gcc/g++ >= 5.4
  * Linux: gcc / g++ is installed by default on most Linux distros
  * Mac: same deal as make - [install Xcode command line tools]((https://developer.apple.com/xcode/features/)
  * Windows: recommend using [MinGW](http://www.mingw.org/)
* [uWebSockets](https://github.com/uWebSockets/uWebSockets)
  * Run either `install-mac.sh` or `install-ubuntu.sh`.
  * If you install from source, checkout to commit `e94b6e1`, i.e.
    ```
    git clone https://github.com/uWebSockets/uWebSockets 
    cd uWebSockets
    git checkout e94b6e1
    ```
    Some function signatures have changed in v0.14.x. See [this PR](https://github.com/udacity/CarND-MPC-Project/pull/3) for more details.
* Fortran Compiler
  * Mac: `brew install gcc` (might not be required)
  * Linux: `sudo apt-get install gfortran`. Additionall you have also have to install gcc and g++, `sudo apt-get install gcc g++`. Look in [this Dockerfile](https://github.com/udacity/CarND-MPC-Quizzes/blob/master/Dockerfile) for more info.
* [Ipopt](https://projects.coin-or.org/Ipopt)
  * Mac: `brew install ipopt`
  * Linux
    * You will need a version of Ipopt 3.12.1 or higher. The version available through `apt-get` is 3.11.x. If you can get that version to work great but if not there's a script `install_ipopt.sh` that will install Ipopt. You just need to download the source from the Ipopt [releases page](https://www.coin-or.org/download/source/Ipopt/) or the [Github releases](https://github.com/coin-or/Ipopt/releases) page.
    * Then call `install_ipopt.sh` with the source directory as the first argument, ex: `bash install_ipopt.sh Ipopt-3.12.1`. 
  * Windows: TODO. If you can use the Linux subsystem and follow the Linux instructions.
* [CppAD](https://www.coin-or.org/CppAD/)
  * Mac: `brew install cppad`
  * Linux `sudo apt-get install cppad` or equivalent.
  * Windows: TODO. If you can use the Linux subsystem and follow the Linux instructions.
* [Eigen](http://eigen.tuxfamily.org/index.php?title=Main_Page). This is already part of the repo so you shouldn't have to worry about it.
* Simulator. You can download these from the [releases tab](https://github.com/udacity/self-driving-car-sim/releases).
* Not a dependency but read the [DATA.md](./DATA.md) for a description of the data sent back from the simulator.


## Basic Build Instructions


1. Clone this repo.
2. Make a build directory: `mkdir build && cd build`
3. Compile: `cmake .. && make`
4. Run it: `./mpc`.

## Tips

1. It's recommended to test the MPC on basic examples to see if your implementation behaves as desired. One possible example
is the vehicle starting offset of a straight line (reference). If the MPC implementation is correct, after some number of timesteps
(not too many) it should find and track the reference line.
2. The `lake_track_waypoints.csv` file has the waypoints of the lake track. You could use this to fit polynomials and points and see of how well your model tracks curve. NOTE: This file might be not completely in sync with the simulator so your solution should NOT depend on it.
3. For visualization this C++ [matplotlib wrapper](https://github.com/lava/matplotlib-cpp) could be helpful.

## Editor Settings

We've purposefully kept editor configuration files out of this repo in order to
keep it as simple and environment agnostic as possible. However, we recommend
using the following settings:

* indent using spaces
* set tab width to 2 spaces (keeps the matrices in source code aligned)

## Code Style

Please (do your best to) stick to [Google's C++ style guide](https://google.github.io/styleguide/cppguide.html).

## Project Instructions and Rubric

Note: regardless of the changes you make, your project must be buildable using
cmake and make!

More information is only accessible by people who are already enrolled in Term 2
of CarND. If you are enrolled, see [the project page](https://classroom.udacity.com/nanodegrees/nd013/parts/40f38239-66b6-46ec-ae68-03afd8a601c8/modules/f1820894-8322-4bb3-81aa-b26b3c6dcbaf/lessons/b1ff3be0-c904-438e-aad3-2b5379f0e0c3/concepts/1a2255a0-e23c-44cf-8d41-39b8a3c8264a)
for instructions and the project rubric.

## Hints!

* You don't have to follow this directory structure, but if you do, your work
  will span all of the .cpp files here. Keep an eye out for TODOs.

## Call for IDE Profiles Pull Requests

Help your fellow students!

We decided to create Makefiles with cmake to keep this project as platform
agnostic as possible. Similarly, we omitted IDE profiles in order to we ensure
that students don't feel pressured to use one IDE or another.

However! I'd love to help people get up and running with their IDEs of choice.
If you've created a profile for an IDE that you think other students would
appreciate, we'd love to have you add the requisite profile files and
instructions to ide_profiles/. For example if you wanted to add a VS Code
profile, you'd add:

* /ide_profiles/vscode/.vscode
* /ide_profiles/vscode/README.md

The README should explain what the profile does, how to take advantage of it,
and how to install it.

Frankly, I've never been involved in a project with multiple IDE profiles
before. I believe the best way to handle this would be to keep them out of the
repo root to avoid clutter. My expectation is that most profiles will include
instructions to copy files to a new location to get picked up by the IDE, but
that's just a guess.

One last note here: regardless of the IDE used, every submitted project must
still be compilable with cmake and make./
