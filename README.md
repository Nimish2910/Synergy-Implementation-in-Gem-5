# ECEN 5598 Final project :Synergy-Implementation-in-Gem-5
## Team members: Nimish Sabnis, Disha Gundecha, Eli Wall
Abstract—Modern memory systems must account
for both reliability and security, especially in high-
performance and cloud environments where data integrity
and confidentiality are critical. In this project, we first eval-
uated a baseline secure memory implementation in Gem5,
where integrity metadata such as Message Authentication
Codes (MACs) are stored separately and require additional
memory accesses for verification. Building on this, we im-
plemented and integrated Synergy a secure memory archi-
tecture that co-locates data and MACs in the same memory
block within the gem5 simulation framework. Synergy
reduces overhead by eliminating separate metadata fetches
and takes advantage of ECC encoded blocks for embedding
authentication. Our implementation modifies the secure
memory controller to support Synergy’s data layout by co
location of MAC. We simulate few benchmarks to compare
performance between the original and Synergy-enhanced
secure memory. The results show that Synergy improves
IPC and reduces MAC related access latency, validating its
effectiveness. We also propose our dynamic synergy model
and an extensive evaluation plan to evaluate synergy under
memory intensive workloads.

# How to implement the above Gem5 structure in your device:

## Setup

```
# clone the repo

git clone https://github.com/gem5/gem5.git --version v23.1.0.0
```
  
```
# initial build

# note: this step takes a while
scons build/<ISA>/gem5.opt -j<num procs>
```
  

## Verify that the build worked!

```
./build/<ISA>/gem5.opt ./configs/example/gem5_library/<isa>-hello.py
```
  

A note about the hello script – in version 23, gem5 started using the “gem5 resources” library to manage external resources such as binaries for syscall emulation mode and kernels/disk images for full system mode. By default, these are downloaded from the gem5 resources website (https://resources.gem5.org/) and stored in `~/.cache/gem5/`. To manage or modify these resources, you can specify your own resources using the call to `obtain_resouce`, specified at `src/python/gem5/resources/resource.py`. I have found this function is slightly less intuitive than it is made out to be… be sure to disable re-downloading on md5 mismatch if you modify any of the resources.

# Build Custom SimObject for Secure Memory Management


Note, there are files that represent all intermediate steps in the repository. First, let’s make a directory where we can build and manage the `SimObject`.

```
mkdir src/mem/secmem-tutorial

# Add SConscript file to allow compilation
touch src/mem/secmem-tutorial/SConscript

## file contents can be found in src/mem/secmem-tutorial/SConscript ##
# change the sources to the v0 versions of the source and header files
```

## Make Python file to declare m5 classes and connect gem5 backend to front-end
```
touch src/mem/secmem-tutorial/SecureMemoryTutorial.py

## file contents can be found in src/mem/secmem-tutorial/SecureMemoryTutorial.py ##
```

Note, if we later want to add runtime parameters to the construction of this object, we can add them to the declaration of the object in this class declaration. When we call the object in the front end, we can pass the parameters to the construction of the `SecureMemory` `SimObject` there.

## Create header and source for custom object

```
## file contents can be found in src/mem/secmem-tutorial/secure_memory_v0.{cc,hh} ##
```

A few notes about the source as currently constructed. We declare our own instantiations of the port class so that we can call the functions associated with the new class on receiving requests/responses to the object. For simplicity, we used the standard ports, but gem5 also has more comprehensive “Queued{Request, Response}Port” classes. We also declare our own stats field that we can use for counting whatever we want.

# Creating Secure Memory Component for Front-End

Once the object has been compiled, we need to build some way of accessing the compiled source in the front end. In the new gem5 library, this is done by adding components and compiling them into front end objects. In particular, these components are located at `src/python/gem5/components`. Let’s start by adding a secure memory component into the front-end accessible memory components.

  
## Add file for the secure memory component

```
## file contents can be found in src/python/gem5/components/memory/secure.py ##
```

This file replicates the `SimpleMemory` object declared in `src/python/gem5/components/memory/simple.py` and creates a callable instance of the `SecureSimpleMemory` (to remain consistent with the callable components in `single_channel.py`). The key component of this class is the creation of the `SecureMemory` object. Note, it is linked to the underlying memory controller through the memory controller port and that the `get_memory_ports` function returns the `cpu_side` port from the `SecureMemory` object. This will allow the cache hierarchy/memory bus to be attached to the `SecureMemory` object before the memory object.

  

Next, we need to be able to compile this new component into the simulator. We do this by adding the following line to `src/python/SConscript` (note, this file is a mess) → `PySource('gem5.components.memory', 'gem5/components/memory/secure.py')` and recompile the simulator.

## Linking custom object into simulation

Now Test that our newly created back-end object and front-end component can be linked into the simulator. Start by creating a new run script that uses the `SecureSimpleMemory` component that we declared in the prior step. We can do this by calling `from gem5.components.memory.secure import SecureSimpleMemory` and replacing the declaration of the memory device with `SecureSimpleMemory`.

  

See example of this in `configs/example/gem5_library/<isa>-hello-secmemtutorial.py`

  

# Adding Functionality 

Next, add the secure memory functionality! In particular, we'll implement the memory access pattern of SYNERGY: Rethinking Secure-Memory Design for Error-Correcting Memories 10.1109/HPCA.2018.00046

Start by adding functions to compute where the metadata addresses for each type are stored. This is done in the `startup()` function, which is declared as virtual in the base `SimObject` class and is called after the constructor for each `SimObject` is completed – this allows us to use the address range for the underlying memory device.
  

After this, we can add functions to compute which precise metadata are to be used for some specific data. We will use these functions to query the memory device for these metadata in particular. These functions are `getParentAddr(uint64_t)` and `getHmacAddr(uint64_t)` and they perform computation on some address. In general, the formula is to figure out the index in the child metadata type and fetches the equivalent index in the parent level (i.e., the parent node in the tree for any data/metadata) divided by the arity of the tree.

If we do this, we break some assumptions about the address range in an abstract memory device. Let’s be sure to turn those assertions off. See these changes in `src/mem/abstract_mem.{cc,hh}` Furthermore, let’s make sure that there is some space to access metadata when we try to fetch it. We do this by creating a buffer for all metadata accesses to load/store from (i.e., `security_metadata`). Because we are only measuring the timing of access, it sufficient for this to be a single block for the purpose of this tutorial.

Next, let’s modify the `handleRequest` and `handleResponse` in our `secure_memory.cc` source to implement the security protocol. When we get a request, we should query memory to fetch the relevant metadata. When we get responses, then we should verify the data and metadata and once the data is verified send it to memory or to the processor.

Finally, let's modify the `handleResponse` call to ensure that all metadata are verified. We authenticate against the root of the tree because it is stored on-chip. Thus, if we receive the response for the root back from the memory device we can assume that our simulated secure memory can authenticate it against the stored value and use that value to verify its children. When some tree node other than the root is returned from memory, we store it in a temporary buffer for `pending_untrusted_packets` for later authentication. When the HMAC fetch returns from memory, we use that value to mark the data as authenticated.

## Acknowledgment
We would like to sincerely thank our professor Tamara
Lehman and the Teaching Assistant Alex Mueller for all
their support. We also would like to acknowledge the Gem5 open-source communities for providing
the base simulation infrastructure and documentation,
which significantly accelerated our development process.
## References:
- G. Saileshwar, P. J. Nair, P. Ramrakhyani, W. Elsasser and M. K. Qureshi, "SYNERGY: Rethinking Secure-Memory Design for Error-Correcting Memories," 2018 IEEE International Symposium on High Performance Computer Architecture (HPCA), Vienna, Austria, 2018, pp. 454-465, doi: 10.1109/HPCA.2018.00046. keywords: {Encryption;Metadata;Memory management;Reliability engineering;Error correction codes;Memory-Security;Reliability;ECC-DIMM},
- https://github.com/samueltphd/SecureMemoryTutorial/tree/b6feae273a708d5dc9d6e8b8de2be8bcfb1ab1a4
- The gem5 Simulator. Nathan Binkert, Bradford Beckmann, Gabriel Black, Steven K. Reinhardt, Ali Saidi, Arkaprava Basu, Joel Hestness, Derek R. Hower, Tushar Krishna, Somayeh Sardashti, Rathijit Sen, Korey Sewell, Muhammad Shoaib, Nilay Vaish, Mark D. Hill, and David A. Wood. ACM SIGARCH Computer Architecture News, May 2011. [ doi: 10.1145/2024716.2024718 ]
  





