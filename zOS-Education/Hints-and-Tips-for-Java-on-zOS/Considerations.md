# Hints and Tips for Java on z/OS – Considerations

This page includes various considerations one needs to consider when running Java applications on z/OS.  Topics include:

1. [Java and LE runtime options](#topic1)
2. [JCL REGION size parameter](#topic2)
3. [64-bit Java](#topic3)
4. [Java Start Up Time](#topic4)
   - [Quickstart mode](#topic4a)
   - [Shared Classes Cache](#topic4b)


## <a name="topic1"></a> Topic 1 - Java and LE runtime options

The Java Virtual Machine (JVM) requires and runs under a z/OS Language Environment (LE) enclave.   As such, the Java executables (java, javac, javah, JZOS Batch Launcher, etc.) are built with a recommended set of LE runtime options.
You can use the LE runtime option `RPTOPTS(ON)` to produce a report that displays the options in effect for the Java executable.  More information about LE runtime options and tunings can be found in the [LE Programming Reference](https://www.ibm.com/support/knowledgecenter/SSLTBW_2.4.0/com.ibm.zos.v2r4.ceea300/abstract.htm?view=kc).

## <a name="topic2"></a> Topic 2 - JCL REGION size parameter

The JCL [REGION](https://www.ibm.com/support/knowledgecenter/en/SSLTBW_2.4.0/com.ibm.zos.v2r4.ieab600/xexreg.htm) size parameter needs to be large enough to allow for all of the heap and stack storage requirements for running your Java application, the Java virtual machine and any underlying z/OS Language Enviornment control blocks.   Failure to specify a large enough REGION size will likely result in a Java OutOfMemory error being thrown.  Information about the allocation failure leading to the OutOfMemory error can usually be seen at the beginning of any javacore text files produced.

## <a name="topic3"></a>Topic 3 - 64-bit Java

Use of 64-bit Java makes it possible to define larger Java heap sizes to avoid out of virtual memory constraints and improve application reliability. Large Java heaps also means that Garbage Collection will occur less frequently, which may improve your applications performance. 

#### 64-bit Java version information

The following Java command is invoked to display the java version information. The "s390x-64" in the IBM J9 VM string signifies that 64-bit Java is being run.

```
> java -version
java version "1.8.0_144"
Java(TM) SE Runtime Environment (build 8.0.5.0 - pmz6480sr5-20170905_01(SR5))
IBM J9 VM (build 2.9, JRE 1.8.0 z/OS s390x-64 Compressed References 20170901_363591 (JIT enabled, AOT enabled)
J9VM - d56eb84
JIT  - tr.open_20170901_140853_d56eb84
OMR  - b033a01)
JCL - 20170823_01 based on Oracle jdk8u144-b01
```

#### Memory above the bar

To run 64-bit Java, the memory limit or memlimit for memory above the bar must be sufficient. Before attempting to use 64-bit Java for the first time on z/OS, check that the memlimit for "memory above the 2 Gigabyte bar" is not zero by using the `ulimit -a` command as shown below. In this example, the memory above the bar is 1000 megabytes. Your configuration may be different.

```
> ulimit -a
core file 8192b
cpu time unlimited
data size unlimited
file size unlimited
stack size unlimited
file descriptors 1500
address space unlimited
memory above bar 1000m
```

Suppose we reduce the memory above the bar from 100 MB to 200 MB using the `ulimit -M` option. 

```
> ulimit -M 200
> 
> ulimit -a
core file 8192b
cpu time unlimited
data size unlimited
file size unlimited
stack size unlimited
file descriptors 1500
address space unlimited
memory above bar 200m
```

With this reduced memory limit of 200M, the 64-bit Java virtual machine is unable to allocate enough virtual memory to initialize its runtime components.  In this case, the JVM failed to initialize the JIT compiler.  Other failure symptoms including ABENDs may result due to insufficient memory.

```
> java HelloWorld
JVMJ9VM015W Initialization error for library j9jit29(11): cannot initialize JIT
Error: Could not create the Java Virtual Machine.
Error: A fatal exception has occurred. Program will exit.
```

If the first time use of 64-bit Java fails, the first thing to check is `ulimit -a` memory above the bar storage is sufficient to run the Java application.  Either increase the memory above the bar limit, or reduce the memory requirements of the Java application as detailed below.

#### Default memory segment sizes

The default JVM sizes for heap, stack and classes segments can be viewed with the `-verbose:sizes` options.  In this example, we see the initial Java heap size to be 8 MB (`-Xms8m`) and the maximum Java heap size to be 512MB (`-Xmx512M`).  These defaults can be overwritten by specifying different values for the individual options.  For example, to reduce maximum Java heap sizes to 128MB, specify `-Xmx128M`.

```
> java -verbose:sizes -version
  -Xmca32K        RAM class segment increment
  -Xmco128K       ROM class segment increment
  -Xmcrs200M      compressed references metadata initial size
  -Xmns2M         initial new space size
  -Xmnx128M       maximum new space size
  -Xms8M          initial memory size
  -Xmos6M         initial old space size
  -Xmox512M       maximum old space size
  -Xmx512M        memory maximum
  -Xmr16K         remembered set size
  -Xlp:objectheap:pagesize=1M,pageable	 large page size
                  available large page sizes:
                  4K pageable
                  1M pageable
                  1M nonpageable
                  2G nonpageable
  -Xlp:codecache:pagesize=1M,pageable	 large page size for JIT code cache
                  available large page sizes for JIT code cache:
                  4K pageable
                  1M pageable
  -Xmso384K       operating system thread stack size
  -Xiss2K         java thread stack initial size
  -Xssi16K        java thread stack increment
  -Xss1M          java thread stack maximum size
```




#### Large Pages

64-bit Java virtual machines can take advantage of 1MB and 2GB Large Pages for the Java heap.  Large pages improves performance by reducing the number of page table entries required.   For more information on Large Pages, please refer to [KnowledgeCenter](https://www.ibm.com/support/knowledgecenter/en/SSYKE2_8.0.0/openj9/xlp/index.html).

## <a name="topic4"></a> Topic 4 - Java Start Up Time

When running hundreds or thousands of small Java batch jobs, the Java start up elapsed time and CPU time can become an important performance measurement for many customers. The following Java options can reduce the Java startup times for applications that start a new JVM frequently:


### <a name="topic4a"></a> -Xquickstart - Quickstart mode

The [`-Xquickstart`](https://www.ibm.com/support/knowledgecenter/en/SSYKE2_8.0.0/openj9/xquickstart/index.html) JVM option causes the JIT compiler to run with a subset of optimizations, which can improve the performance of short-running applications.

### <a name="topic4b"></a> -Xshareclasses - Shared Classes Cache

The [`-Xshareclasses`](https://www.ibm.com/support/knowledgecenter/en/SSYKE2_8.0.0/com.ibm.java.zos.80.doc/diag/appendixes/cmdline/Xshareclasses.html) option uses a shared memory segment to allow the sharing of class data and ahead-of-time (AOT) compilations between JVMs to improve start up performance and reduce memory footprint.

Start up performance can be improved by placing classes that an application needs when initializing into a shared classes cache. The next time the application runs, it takes much less time to start because the classes are already available. When you enable class data sharing, Ahead-of-time (AOT) compilation is also enabled by default, which dynamically compiles certain methods into AOT code at runtime. By using these features in combination, startup performance can be improved even further because the cached AOT code can be used to quickly enable native code performance for subsequent runs of your application.

More information on share classes cache can be found on [KnowledgeCenter](https://www.ibm.com/support/knowledgecenter/en/SSYKE2_8.0.0/com.ibm.java.vm.80.doc/docs/shrc_intro.html).

#### BPXPRMxx parmlib settings

On z/OS, the shared class cache is backed by shared memory segments.  As such, certain `BPXPRMxx` parmlib settings may affect shared classes performance. Using the wrong settings can stop shared classes from working. These settings might also have performance implications.  

For example, if the `IPCSHMMPAGES` setting is too small to provide the necessary shared memory pages to satisfy the shared classes cache, the shared classes cache will not be successfully created.

Please refer to the following [KnowledgeCenter](https://www.ibm.com/support/knowledgecenter/en/SSYKE2_8.0.0/com.ibm.java.vm.80.doc/docs/shrc_bpxprmxx.html) page for more information on the relevant `BPXPRMxx` settings.


#### Shared classes cache name

Shared classes cache is instantiated with the `-Xshareclasses` option.  A unique name can be specified with the `name=` suboption, to create unique instances.  For example, the following will create a shared classes cache named `cache1`. Note that this example requires access to a HelloWorld Java program.

```
> java -Xshareclasses:name=cache1 HelloWorld
Hello World
```

#### Shared class cache size

The size of the shared classes cache can be adjusted with the `-Xscmx` option.  For example, the following command will create an 8MB shared class segment named `cache2`.

```
> java -Xscmx8m -Xshareclasses:name=cache2 HelloWorld
Hello World
```

After an application run populates the shared classes cache (SCC), you can add the `printStats` suboption to gain more insight into how much the SCC was used.  If the % used is 100%, you are not gaining the maximal benefit and should consider specifying a larger `-Xscmx` value.

In this example, we see that the 8MB SCC is 15% full, and has room for additional class and AOT data.

```
> java -Xshareclasses:name=cache2,printStats
Current statistics for cache "cache2":
base address = 0x26F00058
end address = 0x276FFFF8
allocation pointer = 0x2703F508
cache size = 8388520
free bytes = 7067020
ROMClass bytes = 1307824
Metadata bytes = 13676
Metadata % used = 1%
# ROMClasses = 306
# Classpaths = 2
# URLs = 0
# Tokens = 0
# Stale classes = 0
% Stale classes = 0%
Cache is 15% full
```

#### Listing Shared classes cache instances

The `listAllCaches` suboption will list all the shared classes cache instances available to your user ID.

The following example shows two different shared classes cache: `cache1` and `cache2`, along with their respective OS shared memory ID and time of last access by a JVM.

```
> java -Xshareclasses:listAllCaches
Listing all caches in cacheDir /tmp/javasharedresources/

Cache name      	level         cache-type      feature         OS shmid       OS semid       last detach time

Compatible shared caches
cache1          	Java8 64-bit  non-persistent  cr              8199           2297871        Thu Apr 23 20:53:06 2020
cache2          	Java8 64-bit  non-persistent  cr              8201           2297874        Thu Apr 23 20:53:21 2020
```

The `ipcs -bom` command will show the shared memory segments used.  The ID will match the shmid of the above `listAllCaches` suboption.  Note, we see that ID 8201, corresponding to the `cache2` SCC instance has 8 MB of pages under user TESTER.

```
> ipcs -bom

IPC status as of Thu Apr 23 20:54:51 2020
Shared Memory:
T         ID     KEY        MODE       OWNER    GROUP   NATTCH   SEGSZPG PGSZ       SEGSZ
m       8199 0x61c1fb03 --rw-------    TESTER    DEPTD60        0         1  1M      1048576
m       8201 0x61c2b303 --rw-------    TESTER    DEPTD60        0         8  1M      8388608
```

#### Removing a Shared classes cache

Shared memory for Java shared classes is not removed when a JVM terminates, as it must persist in order to be accessible for future JVM invocations.  The `destroy` suboption must be specified in order to remove and delete a given shared classes cache, along with its corresponding shared memory segments. 

Suppose we have the following Shared classes cache and corresponding shared memory.

```
> java -Xshareclasses:listAllCaches
Listing all caches in cacheDir /tmp/javasharedresources/

Cache name      	level         cache-type      feature         OS shmid       OS semid       last detach time

Compatible shared caches
cache1          	Java8 64-bit  non-persistent  cr              8199           2297871        Thu Apr 23 20:53:06 2020

> ipcs -bom

IPC status as of Thu Apr 23 20:54:51 2020
Shared Memory:
T         ID     KEY        MODE       OWNER    GROUP   NATTCH   SEGSZPG PGSZ       SEGSZ
m       8199 0x61c1fb03 --rw-------    TESTER    DEPTD60        0         1  1M      1048576
```

We can issue the following Java command to destroy `cache1`

```
> java -Xshareclasses:name=cache1,destroy
JVMSHRC010I Shared Cache "cache1" is destroyed
Could not create the Java virtual machine.
```

Re-issuing the `ipcs -bom` command, we see the corresponding shared memory segment has now been removed.

```
> ipcs -bom

IPC status as of Thu Apr 23 20:54:51 2020
Shared Memory:
T         ID     KEY        MODE       OWNER    GROUP   NATTCH   SEGSZPG PGSZ       SEGSZ
```

#### Shared Classes Group Access

If multiple users in the same operating system group are running the same application, use the `groupAccess` suboption.  This creates the cache allowing all users in the same primary group to access and share the same cache.  If multiple
operating system groups are running the same application, the `%g` modifier can be added to the cache name, causing each group running the application to get a separate and unique cache.

In the following example, consider the user ID: `TESTER`, who belongs `DEPTD60` group.

```
> id
uid=258(TESTER) gid=193(DEPTD60)
```

A shared classes cache was created prefixed by the user's group name, along with the `groupAccess` suboption.  This shared classes cache can then be accessed by JVMs created by other IDs in the same group.

```
> java -Xshareclasses:name=%g_cache,groupAccess HelloWorld
Hello World
```

Note, the resultant shared classes cache name `DEPTD60_cache` has the user's group `DEPTD60` expanded in place of the `%g` modifier.

```
> java -Xshareclasses:listAllCaches

Listing all caches in cacheDir /tmp/javasharedresources/

Cache name      	level         cache-type      feature         OS shmid       OS semid       last detach time

Compatible shared caches
DEPTD60_cache     	Java8 64-bit  non-persistent  cr              8200           2822155        Thu Apr 23 19:50:51 2020
```

#### Shared Classes Cache persistance

Shared memory pages are not persistent across an z/OS IPL.  As a result, Java applications starting immediately after an IPL will not benefit from an existing shared classes cache.  In Java 8 SR1, a new mechanism was added to persist the Shareclasses to a snapshot file, along with a corresponding option to restore a shared class cache from the snapshot.  This allows a shared classes cache to be reinstantiated post IPL.

More information about the `snapshotCache` and `restoreFromSnapshot` suboptions can be found on [KnowledgeCenter](https://www.ibm.com/support/knowledgecenter/en/SSYKE2_8.0.0/com.ibm.java.zos.80.doc/diag/appendixes/cmdline/Xshareclasses.html).


 




