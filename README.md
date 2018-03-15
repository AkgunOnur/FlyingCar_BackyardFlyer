# FCND - Backyard Flyer Project
In this project, a state machine will be set up using event-driven programming to have an autonomously flying drone. An Unity simulator will be used. 

The written code is similar to how a drone would be controlled from a ground station computer or an onboard flight computer.

[//]: # (Image References)

[image1]: ./examples/1.png ""
[image2]: ./examples/2.png ""

## Mission
The required task is to command the drone to fly a 15 meter box at a 3 meter altitude. You'll fly this path in two ways: first using manual control and then under autonomous control.

Manual control of the drone is done using the instructions found with the simulator.

Autonomous control will be done using an event-driven state machine. First, you will need to fill in the appropriate callbacks. Each callback will check against transition criteria dependent on the current state. If the transition criteria are met, it will transition to the next state and pass along any required commands to the drone.

![alt text][image1]

![alt text][image2]

## Step 1: Download the Simulator
If you haven't already, download the version of the simulator that's appropriate for your operating system [from this repository](https://github.com/udacity/FCND-Simulator-Releases/releases).

## Step 2: Set up your Python Environment
Set up your Python environment and get all the relevant packages installed using Anaconda following instructions in [this repository](https://github.com/udacity/FCND-Term1-Starter-Kit)

## Step 3: Clone this Repository
```sh
git clone https://github.com/MeRKeZ/FlyingCar_BackyardFlyer
```

## Step 4: Running the State Machine

After the commands below are prompted in Anaconda Environment, the code in first cell of Jupyter Notebook is executed

```sh
activate fcnd
jupyter notebook
```

## Drone API

To communicate with the simulator (and a real drone), you will be using the [UdaciDrone API](https://udacity.github.io/udacidrone/).  This API handles all the communication between Python and the drone simulator.  A key element of the API is the `Drone` superclass that contains the commands to be passed to the simulator and allows you to register callbacks/listeners on changes to the drone's attributes.  The goal of this project is to design a subclass from the Drone class implementing a state machine to autonomously fly a box.

### Drone Attributes

The `Drone` class contains the following attributes that you may find useful for this project:

 - `self.armed`: boolean for the drone's armed state
 - `self.guided`: boolean for the drone's guided state (if the script has control or not)
 - `self.local_position`: a vector of the current position in NED coordinates
 - `self.local_velocity`: a vector of the current velocity in NED coordinates

For a detailed list of all of the attributes of the `Drone` class [check out the UdaciDrone API documentation](https://udacity.github.io/udacidrone/).


### Registering Callbacks

As the simulator passes new information about the drone to the Python `Drone` class, the various attributes will be updated.  Callbacks are functions that can be registered to be called when a specific set of attributes are updated.  There are two steps needed to be able to create and register a callback:

1. Create the callback function:

Each callback function needs to be defined as a member function of the `BackyardFlyer` class provided in `backyard_flyer.ipynb` that takes only the `self` parameter.  There is a template below.

```python
class BackyardFlyer(Drone):
    ...

    def local_position_callback(self):
        """ this is triggered when self.local_position contains new data """
        pass
```

2. Register the callback:

In order to have a callback function called when the appropriate attributes are updated, each callback needs to be registered.  This registration takes place in you `BackyardFlyer`'s `__init__()` function as shown below:

```python
class BackyardFlyer(Drone):

    def __init__(self, connection):
        ...

        # TODO: Register all your callbacks here
        self.register_callback(MsgID.LOCAL_POSITION, self.local_position_callback)
```

Since callback functions are only called when certain drone attributes are changed, the first parameter to the callback registration indicates for which attribute changes you want the callback to occur.  For example, here are some message id's that you may find useful (for a more detailed list, see the UdaciDrone API documentation):

 - `MsgID.LOCAL_POSITION`: updates the `self.local_position` attribute
 - `MsgID.LOCAL_VELOCITY`: updates the `self.local_velocity` attribute
 - `MsgID.STATE`: updates the `self.guided` and `self.armed` attributes


### Outgoing Commands

The UdaciDrone API's `Drone` class also contains function to be able to send commands to the drone.  Here is a list of useful commands:

 - `connect()`: Starts receiving messages from the drone. Blocks the code until the first message is received
 - `start()`: Start receiving messages from the drone. If the connection is not threaded, this will block the code.
 - `arm()`: Arms the motors of the quad, the rotors will spin slowly. The drone cannot takeoff until armed first
 - `disarm()`: Disarms the motors of the quad. The quadcopter cannot be disarmed in the air
 - `take_control()`: Set the command mode of the quad to guided
 - `release_control()`: Set the command mode of the quad to manual
 - `cmd_position(north, east, down, heading)`: Command the drone to travel to the local position (north, east, down). Also commands the quad to maintain a specified heading
 - `takeoff(target_altitude)`: Takeoff from the current location to the specified global altitude
 - `land()`: Land in the current position
 - `stop()`: Terminate the connection with the drone and close the telemetry log

These can be called directly from other methods within the drone class:

```python
self.arm() # Seends an arm command to the drone
```

### Message Logging

The telemetry data is automatically logged in "Logs\TLog.txt" or "Logs\TLog-manual.txt" for logs created when running `backyard_flyer.ipynb`. Each row contains a comma seperated representation of each message. The first row is the incoming message type. The second row is the time. The rest of the rows contains all the message properties. The types of messages relevant to this project are:

* `MsgID.STATE`: time (ms), armed (bool), guided (bool)
* `MsgID.GLOBAL_POSITION`: time (ms), longitude (deg), latitude (deg), altitude (meter)
* `MsgID.GLOBAL_HOME`: time (ms), longitude (deg), latitude (deg), altitude (meter)
* `MsgID.LOCAL_POSITION`: time (ms), north (meter), east (meter), down (meter)
* `MsgID.LOCAL_VELOCITY`: time (ms), north (meter), east (meter), down (meter) 


#### Reading Telemetry Logs

Logs can be read using:

```python
t_log = Drone.read_telemetry_data(filename)
```

The data is stored as a dictionary of message types. For each message type, there is a list of numpy arrays. For example, to access the longitude and latitude from a `MsgID.GLOBAL_POSITION`:

```python
# Time is always the first entry in the list
time = t_log['MsgID.GLOBAL_POSITION'][0][:]
longitude = t_log['MsgID.GLOBAL_POSITION'][1][:]
latitude = t_log['MsgID.GLOBAL_POSITION'][2][:]
```

The data between different messages will not be time synced since they are recorded at different times.


### Autonomous Control State Machine

The six states predefined for the state machine:

* MANUAL: the drone is being controlled by the user
* ARMING: the drone is in guided mode and being armed
* TAKEOFF: the drone is taking off from the ground
* WAYPOINT: the drone is flying to a specified target position
* LANDING: the drone is landing on the ground
* DISARMING: the drone is disarming

While the drone is in each state, transition criteria needs to be checked with a registered callback. If the transition criteria are met, next state will be set and any commands will be passed along to the drone. For example:

```python
def state_callback(self):
	if self.state == States.DISARMING:
    	if !self.armed:
        	self.release_control()
        	self.in_mission = False
        	self.state = States.MANUAL
```

This is a callback on the state message. It only checks anything if it's in the DISARMING state. If it detects that the drone is successfully disarmed, it sets the mode back to manual and terminates the mission.       

### Reference Frames

Two different reference frames are used. Global positions are defined [longitude, latitude, altitude (pos up)]. Local reference frames are defined [North, East, Down (pos down)] and is relative to a nearby global home provided. Both reference frames are defined in a proper right-handed reference frame . The global reference frame is what is provided by the Drone's GPS, but degrees are difficult to work with on a small scale. Conversion to a local frame allows for easy calculation of m level distances. Two convenience function are provided to convert between the two frames. These functions are wrappers on `utm` library functions.

```python
# Convert a local position (north, east, down) relative to the home position to a global position (lon, lat, up)
def local_to_global(local_position, global_home):

# Convert a global position (lon, lat, up) to a local position (north, east, down) relative to the home position
def global_to_local(global_position, global_home):
```
