# native-jvm-leaks

Anatomy of a memory leak - debugging native memory leaks in the JVM
===================================================================

What is an off-heap leak?
-------------------------
For the purposes of this discussion, I'll define it as a growth in resident memory size of your JVM over time, as seen by the RSS column in ps -aux or as RES in the top command. Don't panic over the virtual memory size. There are many things happening in a 64-bit system that can make this size appear huge. The kernel can allocate memory into various "arenas" to reduce contention in a highly threaded environment. This is nothing to be concerned about if your resident size is low. The memory is not actually in use.

Is your leak native?
--------------------
Probably not. Make sure you look for heap leaks first. It's the low-hanging fruit and there are many good resources for doing this with profilers such as "mat" from Eclipse, yourkit, etc. You can even find obvious leaks with nothing more than "jmap -histo:live pid" and watch the post-GC heap-size over time. If it's leaking, then compare the object instances to see which class is growing. Profilers like Yourkit are a little more difficult to wire into your jvm startup but do all the work for you are letting you snapshot object counts, analyzing the delta over time, etc.

Is your application allocating off-heap memory?
-----------------------------------------------
We aren't yet at the point of diving into off-heap memory allocations. It may still be possible to find your leak from the JVM heap. Within your profiler, you may see the numbers for off-heap allocation. This tracks the amount memory allocated by your application in the form of DirectByteBuffers which hold a reference to off-heap memory. If your heap is low but your off-heap is high, they you may be leaking away these byte buffers. Capture a heap dump with jmap and analyze it with your preferred profiler and see which on-heap objects are referencing the DirectByteBuffers and determine whether they are leaking. Also watch for MemoryObject instances which work similarly.

I did all that and I'm still screwed
------------------------------------
So your heap looks good, you can't find any off-heap references, the JVM claims there is no unreasonable off-heap allocations, and yet your resident size continues to grow. If your leak is very slow, you mean want to research discussions on how malloc arenas affect the JVM. I have seen reports that this can cause resident leaks in the JVM over time, though I have not seen this myself. Normally the malloc arenas allocate VIRTUAL space where memory can be allocated from various threads to reduce contention. There is an environment variable MALLOC_ARENA_MAX which defaults to 8 which means there may be up to 8 x <num-cores> separate arenas allocating in 64MB chunks. You do the math.

That said, it's probably more likely you have a real leak in native code. Much of the java code itself is native and so, if misused, may not be visible at all to the JVM. This is the problem we are trying to solve here.

JEMALLOC saves the day!
-----------------------
jemalloc is an alternative to the standard malloc in the C library. This alternate implementation was created with the focus of high performance in a highly threaded environment such as ours as well as an attempt to improve on memory fragmentation and groups at Facebook and Twitter have blogged about its use operationally. That is not my point of discussing it here though. It just happens to come with some fantastic profiling capabilities that can help us track down where all our memory is being consumed, so let's stop the chit-chat and get down to business.

Building
--------
The jemalloc library needs to built from source to enable profiling so download the source from this location and put it directly onto the machine you are troubleshooting:

Get it here: [Jemalloc on github](https://github.com/jemalloc/jemalloc/releases).

I took the file jemalloc-4.1.0.tar.bz2 for my testing. Be careful not to collide with existing jemalloc installations on your machine. Other software such as Cassandra may have already seen the light and installed it on the machine so be careful not to clobber what's there. Build the library like this:

```
./configure --enable-prof
make
sudo make install
```

The install step will write the .so files and utilities such as jeprof to /usr/local/... See the caveat above about not clobbing some existing installation. After this step, you will have jemalloc library as well as its profiling utility installed here:
```
/usr/local/lib/libjemalloc.so
/usr/local/bin/jeprof
```
Starting your JVM with jemalloc
-------------------------------
Normally one would link directly with the jemalloc.so file when compiling, but we don't have that luxury since we are running the jvm. Luckily, there is a trick for making the .so get loaded automatically when the process starts. To do this, modify your start script to have this environment variable set:
```
export LD_PRELOAD=/usr/local/lib/libjemalloc.so
```
This results in your executable automatically loading the shared library up front before anything else is loaded/defined. As a result, when the jvm gets loaded any references to malloc are already resolved to our .so.

Configuring the profiler
------------------------
We're not ready to fire it up yet! First we need to define what kind of output jemalloc will output in terms of profiling. Here is a sample of what I have used. Again, in a native application one can modify these settings via global variables in the library, however we are able to set them via an environment variable instead:
```
export MALLOC_CONF=prof:true,lg_prof_interval:31,lg_prof_sample:17,prof_prefix:/my/output/directory/jeprof
```
Some explanation of the numbers here. The 31 in this case is the log base 2 of the interval between allocations that we want jemalloc to report. That's right, log base 2, because profiling native memory allocation weren't already difficult enough. In this case 2^31 is about 2GB, so every 2GB of memory allocation, we'll get a .heap output file specified in the prof_prefix location. Make sure /my/output/directory exists and is writable by your process and you'll get a lot of files named /my/output/directory/jeprof*.heap These are the haystacks that jeprof will git through to find our needle.

Let 'er rip!
------------
Restart your JVM and confirm from your logging that you haven't destroyed things. In my own experience, I have seen no obvious slowdown even under heavy load in production. Observe /my/output/directory for a while to see if your .heap files start showing up. To test our your config, you could lower the lg_prof_interval just to confirm your setup before changing it to some reasonable number. In my setup, the .heap files were roughly 90K each so you'll want to monitor the disk usage there if you're going to be running for a long period of time.

Finding the needle
------------------
At any point, you can ask jeprof to analyze one of the .heap files to see where is presently holding all the memory. Here's how you do that. You'll need to specify the path to your java executable which jeprof will use to determine the symbles:
```
jeprof --show_bytes --gif \
/home/y/share/yjava_jdk/java/bin/java \
/my/output/directory/jeprof-blah-blah.heap > output.gif
```
Copy that back to some machine where you can view the gif. With any luck, the heavens will open, the sun will shine, and a little tear will form at the corner of your eye as you finally have a clue where those gigabytes are being stashed away. It may not be the EXACT location of the leak since this is native code, it's not java classes. I'm still looking into ways of finding out those links, but with any luck, the naming in the native code will give you a clue. In my case for example, I can see that the function was named: Java_sun_java2d_cmm_kcms_CMM_cmmGetTransform which clued me in to the Java2D graphics library we use. Here was my output:

And here we are zoomed in at the top of that branch:

More to come
------------
I'm working on how to tie those allocation addresses back to real locations in the JVM. Stay tuned.

