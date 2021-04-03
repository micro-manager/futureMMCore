# Changes to MMDevice API

Many of the device APIs were designed with a specific type of device in mind (e.g. the Camera device type is for physical cameras, not scanning systems; The Galvo API is particular to photobleaching systems). Furthermore, there have been many innovations in micrscopy hardware since the original design (e.g. event-based cameres, controllable LED array illumination). A major goal of this project is thus to rethink the device layer and update and add new devices as needed.


## Cameras: 
* unify the circular buffer and GetImageBuffer code paths, i.e. maybe every SnapImage should result in the image data automatically being deposited into the circular buffer. 
* Generalize or duplicate Camera API to include support for scanning based cameras or event based camera.Maybe an intermediate level of abstraction should be between MMDevice and Camera. A “DataAcquisitionDevice” which could include 2D cameras, 1D sensors (like you might find in a scanning system), and 3D cameras like this: https://www.imec-int.com/en/hyperspectral-imaging. I suppose an RGB camera falls into the 3D classification as well.
* Improve upon multi-component / multi-channel distinction

## Stages
* XYStages: add a “Move” function to start the stage to move to a specific direction at a specified speed.
* Generalize the idea of Affine transform to encompass coordinate transforms between any types of devices
* Deprecate the “Transpose XY” “MirrorX”, etc. properties of cameras and XY stages and consistently use the affine transform instead.


Make it easy to use devices in different language by compiling bindings
* We think that this is possible by making MMDevice and new Core APIs C, compatible. This still allows things to be written in c++



