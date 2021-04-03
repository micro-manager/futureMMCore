# Changes to MMDevice API

Many of the device APIs were designed with a specific type of device in mind (e.g. the Camera device type is for physical cameras, not scanning systems; The Galvo API is particular to photobleaching systems). Furthermore, there have been many innovations in micrscopy hardware since the original design (e.g. event-based cameres, controllable LED array illumination). A major goal of this project is thus to rethink the device layer and update and add new devices as needed.

## Hardware triggering:
Should the current system be expanded? How so?
### Description of current system for hardware triggering

> Due to latencies in software command execution, and communication with
external devices, it is very difficult or impossible to achieve
microsecond time-scale synchronization between devices in software
alone. Such synchronization is possible when devices can respond (i.e.
change position or state, or start an action) to TTL signals provided
by, for instance, the camera (to signal sensor exposure), or an external
clock (such as a National Instruments DAQ board or an Arduino
microcontroller). Micro-Manager facilitates such work-flows using the
concept of "Sequences".
> 
> A sequence is a finite, discrete sequence of states or actions on a
particular device, each of which can optionally take arguments and data
and produce data. For example, a sequence on a camera will result in the
sequential exposure and readout of multiple images, and will place those
images into a buffer. A Z-stage will take a position as an argument for
each state in the sequence. A spatial light modulator will take a buffer
(usually an image) to describe the pattern it will display at each step
of the sequence
> 
> An important distinction is whether sequences will be triggered
internally or externally, a distinction that is equivalent to "leader"
vs. "follower" devices. For example, cameras can be asked to deliver a
sequence of images, which will most often be accomplished by putting the
camera into "internal trigger mode" so that it uses an internal clock
to guarantee exact exposure times and minimal dead-time in between
exposures. Most cameras in this mode will also produce TTL pulses at
defined times relative to the start of exposure, so that follower
devices can be synchronized to this clock. Alternatively, the camera can
be set in "external trigger mode", in which it waits for an external
trigger to start exposure of a new image.
> 
> The sequencing API contains functions checking whether a device is
sequenceable, loading states of the sequence into the device, and
starting the sequence. For instance, a (Z) stage can signal that it can
respond to TTL signals using its `int IsStageSequenceable(bool&
isSequenceable)` function. A sequence of positions is uploaded using
the `AddToStageSequence(double position)` and `SendStageSequence()`
functions. The Stage will start cycling through these positions, driven
by the external TTL signal after the `StartStageSequence()` is called.
Likewise, XY stages, DAs (analog output devices), and SLMs (spatial
light modulator or projectors) have similar sequencing interfaces.
Moreover, any property of any device can declare that it too can be
sequenced. The interface is similar (a list of property values is
created, sent to the device, and the sequence is started).
> 
> This sequencing interface enables multiple hardware configurations
(camera as "leader", or external device as "leader"), and the code
executing image acquisition can query the hardware and figure out how to
most optimally execute an acquisition sequence. However, it only works
for devices that respond more or less instantaneously. Providing a
mechanism for the device to provide feedback how long a transition will
take would be a useful addition to the interface. Currently, there is no
abstract notion of a device that generates the hardware timing pulses
(TTLs). Such an abstraction of a hardware clock could also be
beneficial.


* [What are the current things that people use labview for](https://github.com/micro-manager/futureMMCore/issues/22), and could this functionlaity be included?

Potential features
* Make the system aware of the time it takes for a device to respond to a TTL signal
* Generalize the concept of a device that drives acquisitions through TTL signals (and that uses knowledge about device delays and TTL feedback).


## Development usability -- Making it really easy for new people to write device adapter
**Problem:** People want to add new devices to MM but are confused/intimidated as to how to do this.

**Proposed solution:**
* Simplify/update the process + Really good documentation
* Update the c++ build system and repository organization, installing VS2010 and getting the various 3rd party dependencies is a time consuming task.
* Create simple to use tests that Device Adapter writers can use to test their code.  For instance, add tests that load and unload a device adapter, with and without an actual device attached, 
* Create tests for multiple device types that check their correct functioning.
* Create tests that measure performance of certain devices
* Investigate the possibility of converting python code to C code through cython to allow for people to create device adapters in Python
  * Or alternatively, provide, templates that can be used to generate most of the code for a simple device adapter without having to write from scratch  

# Specific devices

## Cameras: 
* unify the circular buffer and GetImageBuffer code paths, i.e. maybe every SnapImage should result in the image data automatically being deposited into the circular buffer. 
* Generalize or duplicate Camera API to include support for scanning based cameras or event based camera.Maybe an intermediate level of abstraction should be between MMDevice and Camera. A “DataAcquisitionDevice” which could include 2D cameras, 1D sensors (like you might find in a scanning system), and 3D cameras like this: https://www.imec-int.com/en/hyperspectral-imaging. I suppose an RGB camera falls into the 3D classification as well.
* Improve upon multi-component / multi-channel distinction

## Stages
* XYStages: add a “Move” function to start the stage to move to a specific direction at a specified speed.
* Generalize the idea of Affine transform to encompass coordinate transforms between any types of devices
* Deprecate the “Transpose XY” “MirrorX”, etc. properties of cameras and XY stages and consistently use the affine transform instead.


## Make it easy to use devices in different language by compiling bindings
* We think that this is possible by making MMDevice and new Core APIs C compatible. This still allows things to be written in c++.



