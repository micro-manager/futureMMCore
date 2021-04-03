# The current state of MMCore

- Images are copied from cameras internal buffer into the `MMCore` circular buffer
- Higher level language wrappers (i.e. `MMCoreJ` and `MMCorePy`) then get access to this data by copying it into the respective buffers of their languages, using on many possible functions (`core.getTaggedImage`, `core.getImage`, `core.popNextImage`, etc.)
- In the case of Pycro-Manager, data are copied once again when passing through the ZeroMQ bridge from Java to Python

The cost of the copies for `MMCoreJ` and `MMCorePy` isn't much, but still might be limiting for the highest performance applications. This is evidenced by the fact that (with the right hardware), we can readily achieve writing speeds over 1 GB/s from the Java layer using `AcqEngJ`. The cost of the ZeroMQ transfer to Pycro-Manager is substantial, as it is limited to something on the order of 100 MB/s. It is unclear whether this an inherent consequence of transferring across processes (unlike `MMCoreJ` and `MMCorePy`, which stay in the same process), or whether it is a result of ZeroMQ itself. There may be other implementations of ZeroMQ that could substantially improve speed.

`MMKernel` should be designed so that its implementation is as fast as possible for use cases like streaming data from a camera to a file, or streaming to RAM for real time display like [this](https://pycro-manager.readthedocs.io/en/latest/application_notebooks/PSF_viewer.html). What is the best way to do this?

1. **Do everything at the level of C**. This may make sense for streaming to a file, but not so much for streaming to RAM for visualization/analysis
2. **Pass only pointers to addresses in memory for `MMCoreJ` and `MMCorePy`**. This would require that calling code explicitly handles memory management (i.e. calling the destructor). Thus, the `getImage()` function should be called only once or needs to implement some kind of reference counting. This may allow for fast performance using the existing setup with pycro-manager--that is, pointers to memory addresses could pass quickly through java layer, across ZMQ to python, and be used to instantiate memory of numpy arrays. However, doing this type of shared process memory is [complicated, seems to be OS-specific](https://valelab4.ucsf.edu/svn/3rdpartypublic/boost/doc/html/interprocess/sharedmemorybetweenprocesses.html), and may introduce its own overhead from stuff like locking mechanisms that may make this not even worth it.
3. **Create multiple instances of the kernel for each process**. To circumvent having to deal with inter-process communication, another possibility is having multiple instances of the Kernel, and only have the process that will eventually be using the the memory copy it out of the kernel. For example, if you start an acquisition through MM or PM, a kernel wrapping the camera(s) gets created on Java/Python side respectively so that image never gets copied between processes. This would require new data writing code on the python side, since our current fastest performance file writer [NDTiffStorage](https://github.com/micro-manager/NDTiffStorage) is a Java library, but making would be fairly easy.
 
**Remaining questions**:
- How much overhead is induced by the copy from the C layer to `MMCoreJ` and `MMCorePy`? Is it possible to get around this by wrapping native memory directly in Java/Python?
- Is creating code to write data in pure c++ really necessary? Or could a wrapped native memory + python/java code achieve the same thing?

