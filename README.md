# futureMMCore
Place to discuss the future design of a universal microscope hardware interface

<img src="overview.png" width="600">


## Overarching design principles for new Micro-Manager Kernel/MicroscopeModel API
* Low level API should sit on top of minimal, essential code for hardware control
* Provide a  scripting interface for controlling microscope hardware 
* Usability from different languages
* Allow minimal paths towards acquiring data in order to work with scientific equipment with the highest data rates
* Maintain backwards compatibility with existing device adapters whenever possible
* Make it easier to create complex synchronization between devices


## Proposed division of functionality

* Kernel
  * Memory management
  * Thread safety and parallel device performance
  * All interaction with an external program (i.e. no calling hardware independent of kernel)
* MicroscopeModel
  * Stuff that is now in configuration
    * A list of devices
    * Config groups, preset values
    * Initialization and shutdown settings and API commands
  * Coordinate transforms between stage devices 
  * Temporal relationships and triggering relationships between devices
    * A minimal API called by the acquisition engine to execute devices in their defined order
    * (Maybe an arbitrary list of property values and device API commands)


## MicroscopeModel API
**Problem:** The core as it is currently constructed is implicit microscope architecture created by the “current” device roles. While this works quite nicely for many cases, many new or weird microscope architectures end up fitting into this in a rather clunky way. For example, the multi-camera device adapter, or similarly any device that has multiple XY or Z stages. 

**Proposed solution:** Add another explicit abstraction to allow further customization: A “MicroscopeModel” object with a particular API contract,and an easy way for users to implement custom versions of this object. For example, the calls like core.snapImage shouldn’t go directly to camera device adapters, but rather should pass through an implementation of this object, so that it can define what this actually means. In the case of multiple cameras, it would do essentially what the multi-camera device adapter currently does. Linking multiple devices in such an abstraction would also link together a bunch of features that have been added over the years, but have implementations linked to an assumed model of a microscope. For example, the affine transform functionality corresponds to an XY stage camera pair, but could really be implemented for any combination of devices with coordinate systems.

absorb all “utility” device adapters (multi-camera, multi-shutter) into customized microscope model

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

### What is the implicit microscope model of the current MMCore
The core currently defines the following properties:

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

### (Keep in MMKernel rather than moving to MicroscopeModel )
Initialize: Not sure, indicates that the core is initialized? This doesn’t seem like it needs to be a property

TimeoutMs: The timeout to use for various operations.


[This function](https://github.com/micro-manager/AcqEngJ/blob/c2ef88e98b2baf4117d3422fca3a6a37204e1c6d/src/main/java/org/micromanager/acqj/internal/acqengj/Engine.java#L410) in the acquisition is the one that makes the calls that go along with this model

### Examples of microscopes that don’t fit the implicit paradigm of current MMCore
* Multiple cameras (with different sizes)
* Scanning systems
* (Solicit more examples of specific systems from people on GitHub later)

### Generalized version (i.e. MicroscopeModel API)
* A list of devices
* Config groups, preset values, initialization and shutdown settings
* Coordinate transforms between stage devices 
* Temporal relationships and triggering relationships between devices?

### System for executing actions on MicroscopeModel
* Create 1 or more MicroscopeModel objects
* Call MicroscopeModel.execute(event)
* event is a map of key, value pairs corresponding to a particular set of actions on a microscope
  * Maybe organized in such a way that conveys hardware sequencing? 
* These actions consist of Config groups (i.e. groups of properties), individual properties, and functions 
* Generalize the idea of (Device, Property, Value) thruples to include functions in the essential API
  * E.g. (“DemoCamera”, “.setExposure”, 300)
* Stored in config file as: 
  * DemoCamera, .setExposure, [exposure]
* Here [exposure] is a placeholder for a variable with the name exposure to be provided in the event
* Can also explicitly hard code values in config file, to be reused every time
  * DemoShutter, .setShutterOpen, True
* And maybe also default arguments that can be overridden
  * DemoCamera, .setExposure, [exposure]=100
* This system would be extremely flexible, but also allow acquisitions to be expressed really concisely at runtime. Would make for concise programming through the API, and allow GUI control of more complicated stuff via the acquisition engine.

Example model which corresponds to the roles currently hard coded in core/AcqEng
```
DemoCamera, .setExposure, [exposure]
DemoXYStage, .setPosition, [xy_position] 
DemoZStage, .setPosition, [z_position]
$ConfigGroup, Channel, [preset]  
DemoShutter, setOpen, True
...
```



## Hardware sequencing/triggering
* Make the system aware of the time it takes for a device to respond to a TTL signal
* Generalize the concept of a device that drives acquisitions through TTL signals (and that uses knowledge about device delays and TTL feedback).



## Saving and memory management at low level (performance)
**Problem:** Many current limitations with the Core’s memory model

* 1 Buffer per each Camera or data input device
* Each buffer is sequence of images along with header containing essential metadata (e.g. width, height, depth/channel/component, pixeltype)
  * Or rather than width/height/depth, to support weird configurations (an array of RGB sensors?)
  * Maybe optional additional metadata, kept in a separate structure and able to be turned off to enable the fastest performance
  * Keep track of positions in buffer that have been used, allow reuse if so, throw error if overflowed
  * Write generically enough to allow for variable length images
  * Investigate the option to keep images in heap memory so that they can be passed across languages without copying before explicit deletion
* Study the option to keep images in its driver’s memory (i.e only keep a pointer to the data rather than copying the pixels into the circular buffer)
* Make property cache usage more efficient. E.g. OnPropertiesChanged handler
* Metadata handling: every image in a sequence now has all metadata in the system attached, maybe only the metadata that changed should be included.
  * How much metadata is even needed at this level?
* Allow for data writing output


## Threading in devices
**Problem:** writing code for a device that performs well often requires implementing threading in the device adapter itself. For example, the device adapter has an internal thread that gets dispatched to when a function is called, so that the function doesn’t block, and then the device can be queried for when it is complete. There is also the issue of thread safety. Some devices can crash if accessed from multiple threads simultaneously.

This is problematic, because it makes writing a well performing device adapter difficult for a beginner and thus increases the barrier to entry for community contributions. 

It is likely possible to handle this at a higher level of abstraction (i.e. an acquisition engine that only accesses each device from one thread at a time). However, since thread safety is a desirable property even if not using the acquisition engine, it might make sense to do this in the kernel itself. 
But this still raises the possibility of the same thing happening within a given device (i.e. setX position blocks until completion, then setY position blocks until completion, when in fact they could be happening simultaneously -- does this ever really happen in practice??)

**Proposed solution:** All new device adapters can block for the duration of a call until a specific state is achieved, in order to make writing new devices adapters simple. In order to ensure backwards compatibility, and to allow for the most efficient behavior to be written at the level of device adapters, also have a function in the API that checks if a device is finished all commands (which, in the the case of a blocking device adapter will trivially always return 0).

Thread safety and performance  optimization should occur at the level of the kernel. A single instance of MMKernel should be safe to access from multiple threads. Rather than the current behavior of calling the device adapter code from the same thread that called on MMKernel the kernel should run a thread pool to execute device property handlers in parallel. MMKernel should make sure that each device can only have one property accessed at a time, that way device adapters can be written without worrying about thread safety.



## Changes to MMDevice API

### Cameras: 
* unify the circular buffer and GetImageBuffer code paths, i.e. maybe every SnapImage should result in the image data automatically being deposited into the circular buffer. 
* Generalize or duplicate Camera API to include support for scanning based cameras or event based camera.Maybe an intermediate level of abstraction should be between MMDevice and Camera. A “DataAcquisitionDevice” which could include 2D cameras, 1D sensors (like you might find in a scanning system), and 3D cameras like this: https://www.imec-int.com/en/hyperspectral-imaging. I suppose an RGB camera falls into the 3D classification as well.
* Improve upon multi-component / multi-channel distinction

### Stages
* XYStages: add a “Move” function to start the stage to move to a specific direction at a specified speed.
* Generalize the idea of Affine transform to encompass coordinate transforms between any types of devices
* Deprecate the “Transpose XY” “MirrorX”, etc. properties of cameras and XY stages and consistently use the affine transform instead.


Make it easy to use devices in different language by compiling bindings
* We think that this is possible by making MMDevice and new Core APIs C, compatible. This still allows things to be written in c++


### Metadata
consider normalizing metadata names, or at least maintain a list of preferred properties names to be shared with device adapter authors.

Generic mechanism for persistent storage of calibration settings for devices (NS: not sure what this means)

Delays: Re-evaluate the system used, at a minimum clean up documentation.

## Usability

### Development usability -- Making it really easy for new people to write device adapter
**Problem:** People want to add new devices to MM but are confused/intimidated as to how to do this.

**Proposed solution:**
* Simplify/update the process + Really good documentation
* Update the c++ build system and repository organization, installing VS2010 and getting the various 3rd party dependencies is a time consuming task.
* Create simple to use tests that Device Adapter writers can use to test their code.  For instance, add tests that load and unload a device adapter, with and without an actual device attached, 
* Create tests for multiple device types that check their correct functioning.
* Create tests that measure performance of certain devices
* Investigate the possibility of converting python code to C code through cython to allow for people to create device adapters in Cython


### Other Modules outside MMKernel

#### Logging
Add option to provide logging output through a kernel callback, so that logging output can be redirected to other output than the Kernel log files.


#### Data processing and FileIO
In the current version of MMCore external software is interfaced with the core in order to control hardware and then data acquired from the hardware is passed back to the external software where it can be processed and saved. However, some applications require very high throughput data processing and storage which may not be achievable using this scheme, especially if the external software is not written in a high performance programming language. In order to enable high throughput data processing and storage regardless of the specifics how MMKernel is interfaced with external software a new module should be added to MMKernel which focuses on high performance data processing and file storage. A public interface to this module would allow external software to configure all details of how the data should be handled without requiring any real-time data transmission between MMKernel and the external software.

* Proposed Interface: This interface can probably be copied from the MMStudio data pipeline
```
FileFormat getSupportedFileFormats()
void setFileFormat(FileFormat fileFormat)
void setOutput(char* filePath)
Void setInput(Datastream* stream) // I suppose some sort of “Datastream”  concept would need to be added to devices that 
```
produce data (cameras and other sensors). I guess really any device property could also be considered a datastream.
    addProcessingStep(
