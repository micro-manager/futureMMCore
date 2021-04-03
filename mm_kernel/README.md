# MMKernel
The major problems with `MMCore` as currently designed are:

**1. Memory management**:
  * Two pathways with image buffer and ciruclar buffer
  * Restrictions on image shape
  * Copy data when moving through binding to other languages

**2. Threading**: 
  * Threading model puts burden of writing high performance device adapters onto the developer

**3. Implicit model of microscope**:
  The core as it is currently constructed is implicit microscope architecture created by the “current” device roles. While this works quite nicely for many cases, many new or weird microscope architectures end up fitting into this in a rather clunky way. For example, the multi-camera device adapter, or similarly any device that has multiple XY or Z stages. 

To address these we propose to replace the current core `MMCore`, with a new object `MMKernel`, with the following design

### Memory management
[New Memory Management overview](memory_management.md)


### Threading
[New Threading Model overview](threading.md)  


### Metadata
- Consider normalizing metadata names, or at least maintain a list of preferred properties names to be shared with device adapter authors.
- Generic mechanism for persistent storage of calibration settings for devices
    - For example, persistent calbration settings for a liquid crystal or AOTF devices
- Delays: Re-evaluate the system used, at a minimum clean up documentation.


### Other low-level Modules outside MMKernel

#### Logging ([Discussion](https://github.com/micro-manager/futureMMCore/issues/12))

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
