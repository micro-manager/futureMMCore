# MMKernel

MMKernel is the successor to MMCore, which aims to provide improved memory and thread management and a lighter weight API

## Saving and memory management at low level (performance)
**Problem:** Many current limitations with the Core’s memory model, which in some cases limit performance

See [discussion](https://github.com/micro-manager/futureMMCore/issues/17)

Some more unorganized ideas:
* 1 Buffer per each Camera or data input device
* Each buffer is sequence of images along with header containing essential metadata (e.g. width, height, depth/channel/component, pixeltype)
  * Or rather than width/height/depth, to support weird configurations (an array of RGB sensors?)
  * Maybe optional additional metadata, kept in a separate structure and able to be turned off to enable the fastest performance
  * Keep track of positions in buffer that have been used, allow reuse if so, throw error if overflowed
  * Write generically enough to allow for variable length images
* Make property cache usage more efficient. E.g. OnPropertiesChanged handler
* Metadata handling: every image in a sequence now has all metadata in the system attached, maybe only the metadata that changed should be included.
  * How much metadata is even needed at this level?



## Threading in devices
**Problem:** writing code for a device that performs well often requires implementing threading in the device adapter itself. For example, the device adapter has an internal thread that gets dispatched to when a function is called, so that the function doesn’t block, and then the device can be queried for when it is complete. There is also the issue of thread safety. Some devices can crash if accessed from multiple threads simultaneously.

This is problematic, because it makes writing a well performing device adapter difficult for a beginner and thus increases the barrier to entry for community contributions. 

It is likely possible to handle this at a higher level of abstraction (i.e. an acquisition engine that only accesses each device from one thread at a time). However, since thread safety is a desirable property even if not using the acquisition engine, it might make sense to do this in the kernel itself. 
But this still raises the possibility of the same thing happening within a given device (i.e. setX position blocks until completion, then setY position blocks until completion, when in fact they could be happening simultaneously -- does this ever really happen in practice??)

**Proposed solution:** All new device adapters can block for the duration of a call until a specific state is achieved, in order to make writing new devices adapters simple. In order to ensure backwards compatibility, and to allow for the most efficient behavior to be written at the level of device adapters, also have a function in the API that checks if a device is finished all commands (which, in the the case of a blocking device adapter will trivially always return 0).

Thread safety and performance  optimization should occur at the level of the kernel. A single instance of MMKernel should be safe to access from multiple threads. Rather than the current behavior of calling the device adapter code from the same thread that called on MMKernel the kernel should run a thread pool to execute device property handlers in parallel. MMKernel should make sure that each device can only have one property accessed at a time, that way device adapters can be written without worrying about thread safety.



### Metadata
consider normalizing metadata names, or at least maintain a list of preferred properties names to be shared with device adapter authors.

Generic mechanism for persistent storage of calibration settings for devices (NS: not sure what this means)

Delays: Re-evaluate the system used, at a minimum clean up documentation.




### Other low-level Modules outside MMKernel

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
