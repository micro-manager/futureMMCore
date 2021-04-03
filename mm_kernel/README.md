# MMKernel

MMKernel is the successor to MMCore, which aims to provide improved memory and thread management and a lighter weight API

## Saving and memory management at low level (performance)
**Problem:** Many current limitations with the Core’s memory model, which in some cases limit performance

See [overview](https://github.com/micro-manager/futureMMCore/blob/main/mm_kernel/memory_management.md)


### Threading
[Threading Model](threading.md)  


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
