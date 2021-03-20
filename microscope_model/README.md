# `MicroscopeModel`

## Overview

Currently, MMCore defines an implicit model of microscope, and this model is built around the structure of a turnkey commercial system for cell biology circa 2005. It makes assumptions like a single XY Stage, a single Z stage, a single camera, etc.

In many respects, Micro-Manager has been remarkably successful at adapting this model to fit different systems. For example, the multi-camera device adapter enables the use of two or more physical cameras as a single logical camera. However, this introduces its own difficulties, and there are limits to how far this can be pushed.

The goal of the new `MicroscopeModel` is to generalize this idea of a microscope model to different types of microscopes. This has the potential to greatly simplify the writing of higher level code and extend is usefulness to various different types of microscopes.


## Background
MMCore currently defines the following properties for device "roles":

* AutoFocus: Which C++ autofocus device to use. Only has one option
* AutoShutter: True/False for whether to use autoshuttering
* Camera: The default camera device.
* ChannelGroup: The default `config group` 
* Focus: The default Z Stage
* Galvo: A default galvonometer
* SLM: A default Spatial Light Modulator
* Shutter: A default shutter
* XYStage: A default XY stage.
* ImageProcessor: Unknown, no options

 * Initialize: Not sure, indicates that the core is initialized? This doesn’t seem like it needs to be a property
 * TimeoutMs: The timeout to use for various operations.

These "roles" are then called by generic functions in the API. For example, `core.snapImage()` applies to the default camera. The advantage of such an approach is that it allows higher level code to be written generically, without explicit knowledge of the microscope. [This function](https://github.com/micro-manager/AcqEngJ/blob/c2ef88e98b2baf4117d3422fca3a6a37204e1c6d/src/main/java/org/micromanager/acqj/internal/acqengj/Engine.java#L410) in the acquisition engine makes a bunch of calls on devices in these default "roles", greatly simplifying even higher level code for acquiring data (like multi-dimensional acquisitons, or the `Acquistion` class in Pycro-Manager).

However, hard-coding default roles in such a way restricts means that the acquisition engine needs special code to be written for microscopes that don't fit this paradigm (e.g. [Acquisition hooks](https://pycro-manager.readthedocs.io/en/latest/acq_hooks.html) in Pycro-Manager).

A better solution would be to supply a configuration file that defines the Model for a given microscope and how its various pieces of hardware fit together.


## Examples of microscopes that don’t fit the implicit paradigm of current MMCore
* Multiple cameras (with different sizes)
* Scanning systems
* (Solicit more examples of specific systems from community) (TODO: make an issue to track this)


## An explicit `MicroscopeModel`

The new version of this will consist of:

* A list of devices
* Config groups, preset values, initialization and shutdown settings
* Coordinate transforms between stage devices 
* Temporal relationships and triggering relationships between devices? (TODO: add discussion issue)


### A flexible system for describing what a microscope "does"

Essentially this would be a brief script, calling device properties and core functions given a set of arguments supplied at runtime

* Create 1 or more MicroscopeModel objects
* Call `MicroscopeModel.execute(event)`
  * `event` is a map of key, value pairs corresponding to a particular set of actions on a microscope
   * Maybe organized in such a way that conveys hardware sequencing? 
   * These actions consist of Config groups (i.e. groups of properties), individual properties, and functions 

Generalize the idea of (Device, Property, Value) thruples to include functions in the essential API
  * E.g. `(“DemoCamera”, “.setExposure”, 300)`
* Stored in config file as: 
  * `DemoCamera, .setExposure, [exposure]`
* Here `[exposure]` is a placeholder for a variable with the name exposure to be provided in the event
* Can also explicitly hard code values in config file, to be reused every time
  * `DemoShutter, .setShutterOpen, True`
* And maybe also default arguments that can be overridden
  * `DemoCamera, .setExposure, [exposure]=100`
* This system would be extremely flexible, but also allow acquisitions to be expressed in a very concise way at truntime. Would make for concise programming through the API, and allow GUI control of more complicated stuff via the acquisition engine.

Example model which corresponds to the roles currently hard coded in core/AcqEng

```
__Device__DemoCamera, .setExposure, [exposure]
__Device__DemoXYStage, .setPosition, [xy_position] 
__Device__DemoZStage, .setPosition, [z_position]
__Keyword__ConfigGroup, Channel, [preset]  
__Device__DemoShutter, setOpen, True
__Device__DemoCamera, .snapImage
__Device__DemoShutter, setOpen, False
__Device__DemoCamera2, .readImage
```

Maybe could also add conditionals in here, like:

```
if [camera1]
   __Device__DemoCamera1, .snapImage
else 
   __Device__DemoCamera2, .snapImage
```


This model could then be fed an event like:

```
{
   'exposure': 100,
   'xy_position': (12.3, 432.1),
   'z_position': 7456.6,
   'preset': 'DAPI'
}

```



### What is currently in Micro-Manager config files, and how best to port them over

Store this in the config file? (What things are currently stored in config files?)

```
# Devices

# Pre-init settings for devices

# Pre-init settings for COM ports

# Initialize
Property,Core,Initialize,1

# Delays

# Roles

# Camera-synchronized devices

# Labels

# Configuration presets
# Group: Camera
# Preset: MedRes

# Group: System
# Preset: Startup

# PixelSize settings
# Resolution preset: Res20x
```
