# Usability

## Development usability -- Making it really easy for new people to write device adapter
**Problem:** People want to add new devices to MM but are confused/intimidated as to how to do this.

**Proposed solution:**
* Simplify/update the process + Really good documentation
* Update the c++ build system and repository organization, installing VS2010 and getting the various 3rd party dependencies is a time consuming task.
* Create simple to use tests that Device Adapter writers can use to test their code.  For instance, add tests that load and unload a device adapter, with and without an actual device attached, 
* Create tests for multiple device types that check their correct functioning.
* Create tests that measure performance of certain devices
* Investigate the possibility of converting python code to C code through cython to allow for people to create device adapters in Cython
