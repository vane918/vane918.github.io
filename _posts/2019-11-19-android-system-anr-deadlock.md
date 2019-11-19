---
layout: post
title:  "Deadlocks and ANRs(非原创)"
date:   2019-11-19 00:00:00
catalog:  true
tags:
    - android
    - ANR
    - 死锁
---



 In this article we're going to have a look at how to analyse a real-world Application Not Responding (ANR) trace, determine the cause (which turns out to be a deadlock in one of the libraries we're using) and eliminate it. We're also going to take the opportunity to provide a short primer on deadlocks and related computer science topics needed to understand this article. 

# A primer on Mutexes and Deadlocks

Let's start by having a look at what deadlocks and mutexes are. Those familiar with these subjects can skip forward to the "Interpreting ANR traces" section.

Please keep in mind this is not intended to be an exhaustive treatment of this subject (which is quite complex and interesting) but instead a short primer for those of not fresh out of university.

## What are mutexes

In computer science, a mutex (short for **mutual exclusion**) is a kind of object that allows only one thread at a time to access a shared resource (such as a file) during a critical section of the code. In some languages (most notably for us Java) it is called a `Lock` and we're going to use the terms interchangeably.

It typically has two methods: `acquire` and `release` (sometimes called `lock` and `unlock`). The operating system kernel guarantees that once a thread has successfully acquired the lock, no other thread can acquire it before the first thread has released it. Attempting to do so would block the second thread until the lock is available again. Hence if all the threads make sure to only use the shared resource between calls to `acquire` and `release` they are guaranteed to take turns using that resource.

## An analogy for mutexes

To better understand mutexes, it helps to imagine a physical world pattern that is similar. For example, in some agile teams it is customary to use a "speaking token" - such as a ball or whiteboard marker - to keep track of who is supposed to speak at any time. Only the person holding the token is allowed speak. When that person is done the token is put on the table and somebody else, not necessarily in order, picks it up and delivers his or her report.

Mutexes work in a similar fashion: threads are analogous to the team members, the shared resource is the attention of the colleagues while the mutex is represented by the "speaking token". Locking the mutex is similar to picking up the token while unlocking it is similar to putting the token back in the middle of the table.

## Mutexes in action

Let's consider two threads, `ThreadA` and `ThreadB` trying to write to a log file. Let's assume they are trying to write "Hello world from ThreadA" and "Hello world from ThreadA".

```java
try {  
    final FileOutputStream fos = new FileOutputStream(File.createTempFile("log", ".txt"));

    Runnable r = new Runnable() {
        @Override
        public void run() {
            try {
                fos.write(("Hello world from " + Thread.currentThread().getName()).getBytes());
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    };

    new Thread(r, "ThreadA").start();
    new Thread(r, "ThreadB").start();
} catch (IOException ex) {
    ex.printStackTrace();
}
// close the file before you exit the app
```

Even without synchronisation, most of the time this will produce the desired effect. However, once in a while, perhaps as rarely as once in a thousand runs, you will see output such as this:

```logcat
Hello worHello world from ThreadB  
ld from ThreadA  
```

What happened here is that ThreadB produced its output while ThreadA was producing its own and the file ends up being corrupted. The solution to this is to use a mutex (called a `Lock` in Java):

```java
try {  
    final FileOutputStream fos = new FileOutputStream(File.createTempFile("log", ".txt"));
    final Lock fileLock = new ReentrantLock();

    Runnable r = new Runnable() {
        @Override
        public void run() {
            try {
                fileLock.lock();
                // Thread is now guaranteed exclusive access
                fos.write(("Hello world from " + Thread.currentThread().getName()).getBytes());
                fileLock.unlock();
                // Thread no longer has exclusive access
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    };

    new Thread(r, "ThreadA").start();
    new Thread(r, "ThreadB").start();
} catch (IOException ex) {
    ex.printStackTrace();
}
```

With this addition, the situation described earlier is avoided. Before it can write to the file, the thread has to acquire the mutex (lock) and if both try to do so "at the same time" the Android Linux Kernel guarantees that only one returns from the method call while the other is blocked until the first one releases the mutex.

## Deadlocks

While mutexes are very powerful when used correctly, they do have the potential to cause a rare and elusive problem called a deadlock. This usually happens when two threads each try to acquire a mutex that the other one holds. This results in both of them being blocked permanently. On Android, if the threads are essential to running the application (for example if one of them is the main thread), it will most likely lead to an ANR (Application Not Responding) and the application crashing.

Deadlocks are notoriously nasty problems to have to deal with. The main reasons for this are the usually very low reproduction rate and the lack of logs detailing the problem. The latter problem is caused by the fact that there are no errors/exceptions thrown. The only symptom is the lack of output and the fact that the application is not responding to any messages (e.g. keypresses, intent broadcasts, etc.) which in turn causes the Android System to prompt the user with a dialog asking if he or she wishes to kill the app (the infamous ANR).

## An analogy for deadlocks

Our real-life analogy gets a bit stretched when trying to demonstrate deadlocks, but let's give it a go nevertheless. Let's assume that Bob and Alice are responsible for regulating access to two shared resources, say a log file and a database connection. Bob and Alice agree that whoever wants to use the database has to hold a particular red marker while whoever wants access to the log file has to hold a particular black marker.

This works very well for a while, until at one point the following scenario happens:

1. Bob grabs the red marker and starts writing to the database.
2. Alice grabs the black marker and starts writing some logs.
3. While still holding the red marker, Bob tries to get the black marker as well to write logs about the database activity.
4. Since the black marker is not available, Bob stops and waits for Alice to be done with it.
5. While still holding the black marker, Alice tries to get the red marker as well to check some database activity.
6. Since the red marker is not available, Alice stops and waits for Bob to be done with it.

As you can see, both Alice an Bob now hold one of the markers and waits for the other person to release theirs. No progress can be made and the activity permanently blocks. We have a deadlock.

## Resolving deadlocks

The solution to this situation, if detected in your code, is to change the order in which the locks are acquired. This strategy is sometimes called establishing a mutex hierarchy. Let's say Bob and Alice agree that in order to acquire both markers, you first have to grab the red marker and then the black marker (and *never* the other way around). The above scenario would then unfold like this:

1. Bob grabs the red marker and starts writing to the database.
2. Alice grabs the black marker and starts writing some logs.
3. While still holding the red marker, Bob tries to get the black marker as well to write logs about the database activity.
4. Since the black marker is not available, Bob stops and waits for Alice to be done with it.
5. Alice needs both markers, but due to the agreement above she releases the black marker first.
6. Alice now tries to acquire the red marker, but since it's not available, she stops and waits for Bob to be done with it.
7. Bob grabs the black marker (which is now available) and performs his work.
8. Bob releases the black marker
9. Bob releases the red marker.
10. Alice acquires the red marker which is now available.
11. Alice acquires the black marker.
12. Alice performs her work.
13. Alice releases the black marker.
14. Alice releases the red marker.

Hence, there is no potential for a deadlock with this rule in place.

# Dealing with ANRs

If you are unlucky enough to end up with an ANR in your application, the only things that you will have to work with are probably going to be stats and ANR traces. You will probably not be able to reproduce it and the feedback you are getting from the end users is going to be useless at best and misleading at worst.

First of all, please note that if you're using some 3rd party means of reporting crashes such as HockeyApp, your ANRs will not show up there. You will have to go to the Google Play Console, click **Crashes & ANRs** and make sure the toggle button is set to ANRs. Then select one of the issues that appears in the table and you are presented with stats and an ANR trace.

First look at the stats. These can tell you which android versions and phones are affected, as well as when the problem started (usually with one of the app releases).

Then you have to look at the ANR traces. These look quite intimidating at first, but with a bit of patience you can make sense of them.

## Interpreting ANR traces

An ANR trace is quite similar to a much more familiar stack trace that you get with your garden variety exception (such as a NullPointerException), with a few notable differences:

- It has a pretty big header at the top, spanning several lines, containing info about the process. Don't worry if you cannot understand much of it, very few android app developers do, and the solution to the ANR is rarely there.
- There is more than a single stacktrace in there - in fact there can be quite a few. The reason is that unlike the more familiar situation in which your code threw an exception in one of the app threads, in case of an ANR the app has ground to a halt and the best that Android can do is dump the state of **all** threads since there is no way of knowing which one is at fault.
- There is no exception name present since there is no exception thrown.
- There is a thread header at the top of each thread listing it's name and status:

```logcat
"ReferenceQueueDaemon" daemon prio=5 tid=3 Waiting
```

- A lot of the stacktrace entries are cryptic since they show native code:

```logcat
native: #00 pc 000000000048df54  /system/lib64/libart.so (_ZN3art15DumpNativeStackERNSt3__113basic_ostreamIcNS0_11char_traitsIcEEEEiPKcPNS_9ArtMethodEPv+236)  
```

In general, the structure is something like:

```logcat
----- pid 3775 at 2017-03-14 11:28:21 -----
Cmd line: ....  
Build fingerprint: ....  
ABI: 'x86'  
....

DALVIK THREADS (16):  
"thread name" ... Status
  | group="main" ...
  | held mutexes= ...
  at java.lang.Thread.sleep!(Native method)
  - sleeping on <0x0c075403> (a java.lang.Object)
  at java.lang.Thread.sleep(Thread.java:1031)
  - locked <0x0c075403> (a java.lang.Object)
  at java.lang.Thread.sleep(Thread.java:985)
  at ...
  at ...
...

"thread name2" ... Status
  | group= ...
  | held mutexes= ...
  at java.lang.Thread.sleep!(Native method)
  - sleeping on <0x0c075403> (a java.lang.Object)
  at java.lang.Thread.sleep(Thread.java:1031)
  - locked <0x0c075403> (a java.lang.Object)
  at java.lang.Thread.sleep(Thread.java:985)
  at ...
  at ...

...
```

Usually, the information you want is in the thread entry, so try to focus on those sections when chasing your ANR. Pay particular attention to your "main" (tid=1) thread which is the one that handles UI operations. An ANR is always caused by this thread being blocked or performing too much work.

## Detecting a deadlock

Each thread entry in the trace contains some hints that can help you detect a deadlock. First of all you can see the state of a thread: it usually is `Runnable`, `Sleeping`, `Waiting`, `Native` or `Blocked`. This tells you what state the thread was in when the ANR happened, and of particular interest here are the threads that are marked as `Blocked`.

Threads marked as blocked will generally tell you what mutex (lock) they are trying to acquire and the thread ID (tid) of the thread holding that lock. You can then scroll down to the entry corresponding to that thread in the list and if you find that thread is `Blocked` as well you can look at what mutex it is trying to acquire and which thread is holding that one. Often times this is the first thread, which means you have detected the deadlock.

Let's look at a real example (note: the trace is real, but it was doctored to remove all identifiable information):

```logcat
----- pid 25967 at 2017-03-30 05:32:57 -----  
Cmd line: com.example  
Build fingerprint:   
ABI: 'arm64'  
Build type: optimized  
Zygote loaded classes=4071 post zygote classes=1552  
Intern table: 57141 strong; 443 weak  
JNI: CheckJNI is off; globals=325 (plus 363 weak)  
Libraries: /data/app/com.google.android.gms-1/lib/arm64/libconscrypt_gmscore_jni.so /data/app/com.google.android.gms-1/lib/arm64/libgmscore.so /system/lib64/libandroid.so /system/lib64/libcompiler_rt.so /system/lib64/libfm_jni.so /system/lib64/libhwdeviceinfo.so /system/lib64/libhwtheme_jni.so /system/lib64/libjavacrypto.so /system/lib64/libjnigraphics.so /system/lib64/libmedia_jni.so /system/lib64/libwebviewchromium_loader.so libjavacore.so (12)  
Heap: 20% free, 22MB/28MB; 55958 objects  
Dumping cumulative Gc timings  
Start Dumping histograms for 1 iterations for partial concurrent mark sweep  
ProcessMarkStack:    Sum: 11.069ms 99% C.I. 0.105ms-10.838ms Avg: 3.689ms Max: 10.839ms  
MarkRootsCheckpoint:    Sum: 5.943ms 99% C.I. 2.735ms-3.208ms Avg: 2.971ms Max: 3.208ms  
UpdateAndMarkZygoteModUnionTable:    Sum: 4.417ms 99% C.I. 4.417ms-4.417ms Avg: 4.417ms Max: 4.417ms  
UpdateAndMarkImageModUnionTable:    Sum: 4.200ms 99% C.I. 4.200ms-4.200ms Avg: 4.200ms Max: 4.200ms  
MarkConcurrentRoots:    Sum: 3.501ms 99% C.I. 0.203ms-3.269ms Avg: 1.750ms Max: 3.298ms  
SweepMallocSpace:    Sum: 2.069ms 99% C.I. 0.040ms-2.029ms Avg: 1.034ms Max: 2.029ms  
ScanGrayAllocSpaceObjects:    Sum: 1.014ms 99% C.I. 4us-1010us Avg: 507us Max: 1010us  
ReMarkRoots:    Sum: 830us 99% C.I. 830us-830us Avg: 830us Max: 830us  
MarkAllocStackAsLive:    Sum: 527us 99% C.I. 527us-527us Avg: 527us Max: 527us  
(Paused)ScanGrayAllocSpaceObjects:    Sum: 497us 99% C.I. 0.500us-495.500us Avg: 248.500us Max: 497us
SweepLargeObjects:    Sum: 216us 99% C.I. 216us-216us Avg: 216us Max: 216us  
ImageModUnionClearCards:    Sum: 167us 99% C.I. 57us-110us Avg: 83.500us Max: 110us  
SweepSystemWeaks:    Sum: 93us 99% C.I. 93us-93us Avg: 93us Max: 93us  
AllocSpaceClearCards:    Sum: 84us 99% C.I. 1us-55us Avg: 21us Max: 55us  
FinishPhase:    Sum: 81us 99% C.I. 81us-81us Avg: 81us Max: 81us  
EnqueueFinalizerReferences:    Sum: 78us 99% C.I. 78us-78us Avg: 78us Max: 78us  
MarkNonThreadRoots:    Sum: 69us 99% C.I. 31us-38us Avg: 34.500us Max: 38us  
ScanGrayImageSpaceObjects:    Sum: 56us 99% C.I. 56us-56us Avg: 56us Max: 56us  
(Paused)ScanGrayImageSpaceObjects:    Sum: 53us 99% C.I. 53us-53us Avg: 53us Max: 53us
ZygoteModUnionClearCards:    Sum: 35us 99% C.I. 15us-20us Avg: 17.500us Max: 20us  
RevokeAllThreadLocalAllocationStacks:    Sum: 24us 99% C.I. 24us-24us Avg: 24us Max: 24us  
(Paused)PausePhase:    Sum: 20us 99% C.I. 20us-20us Avg: 20us Max: 20us
(Paused)ProcessMarkStack:    Sum: 18us 99% C.I. 18us-18us Avg: 18us Max: 18us
MarkingPhase:    Sum: 17us 99% C.I. 17us-17us Avg: 17us Max: 17us  
ScanGrayZygoteSpaceObjects:    Sum: 15us 99% C.I. 15us-15us Avg: 15us Max: 15us  
ProcessReferences:    Sum: 11us 99% C.I. 11us-11us Avg: 11us Max: 11us  
PreCleanCards:    Sum: 10us 99% C.I. 10us-10us Avg: 10us Max: 10us  
(Paused)ScanGrayZygoteSpaceObjects:    Sum: 9us 99% C.I. 9us-9us Avg: 9us Max: 9us
ProcessCards:    Sum: 8us 99% C.I. 4us-4us Avg: 4us Max: 4us  
MarkRoots:    Sum: 6us 99% C.I. 6us-6us Avg: 6us Max: 6us  
BindBitmaps:    Sum: 4us 99% C.I. 4us-4us Avg: 4us Max: 4us  
RecursiveMark:    Sum: 2us 99% C.I. 2us-2us Avg: 2us Max: 2us  
SweepZygoteSpace:    Sum: 1us 99% C.I. 1us-1us Avg: 1us Max: 1us  
FindDefaultSpaceBitmap:    Sum: 0 99% C.I. 0ns-0ns Avg: 0ns Max: 0ns  
Done Dumping histograms  
partial concurrent mark sweep paused:    Sum: 3.368ms 99% C.I. 3.368ms-3.368ms Avg: 3.368ms Max: 3.368ms  
partial concurrent mark sweep total time: 35.187ms mean time: 35.187ms  
partial concurrent mark sweep freed: 9904 objects with total size 1088KB  
partial concurrent mark sweep throughput: 282971/s / 30MB/s  
Start Dumping histograms for 1 iterations for sticky concurrent mark sweep  
FreeList:    Sum: 6.592ms 99% C.I. 9us-448.750us Avg: 160.780us Max: 457us  
SweepSystemWeaks:    Sum: 4.712ms 99% C.I. 4.712ms-4.712ms Avg: 4.712ms Max: 4.712ms  
ProcessMarkStack:    Sum: 3.085ms 99% C.I. 0.333us-2990us Avg: 771.250us Max: 3038us  
MarkConcurrentRoots:    Sum: 2.759ms 99% C.I. 0.017ms-2.723ms Avg: 1.379ms Max: 2.742ms  
MarkRootsCheckpoint:    Sum: 2.357ms 99% C.I. 0.699ms-1.658ms Avg: 1.178ms Max: 1.658ms  
SweepArray:    Sum: 2.075ms 99% C.I. 2.075ms-2.075ms Avg: 2.075ms Max: 2.075ms  
ScanGrayAllocSpaceObjects:    Sum: 1.745ms 99% C.I. 0.500us-934us Avg: 436.250us Max: 934us  
ScanGrayImageSpaceObjects:    Sum: 1.419ms 99% C.I. 33us-1386us Avg: 709.500us Max: 1386us  
ScanGrayZygoteSpaceObjects:    Sum: 362us 99% C.I. 6us-356us Avg: 181us Max: 356us  
ResetStack:    Sum: 204us 99% C.I. 204us-204us Avg: 204us Max: 204us  
MarkingPhase:    Sum: 200us 99% C.I. 200us-200us Avg: 200us Max: 200us  
ReMarkRoots:    Sum: 157us 99% C.I. 157us-157us Avg: 157us Max: 157us  
AllocSpaceClearCards:    Sum: 123us 99% C.I. 0.500us-67us Avg: 30.750us Max: 67us  
(Paused)ScanGrayAllocSpaceObjects:    Sum: 68us 99% C.I. 0.500us-68us Avg: 34us Max: 68us
MarkNonThreadRoots:    Sum: 59us 99% C.I. 24us-35us Avg: 29.500us Max: 35us  
ZygoteModUnionClearCards:    Sum: 46us 99% C.I. 11us-35us Avg: 23us Max: 35us  
FinishPhase:    Sum: 38us 99% C.I. 38us-38us Avg: 38us Max: 38us  
(Paused)ScanGrayImageSpaceObjects:    Sum: 31us 99% C.I. 31us-31us Avg: 31us Max: 31us
(Paused)PausePhase:    Sum: 30us 99% C.I. 30us-30us Avg: 30us Max: 30us
EnqueueFinalizerReferences:    Sum: 29us 99% C.I. 29us-29us Avg: 29us Max: 29us  
ReclaimPhase:    Sum: 17us 99% C.I. 17us-17us Avg: 17us Max: 17us  
InitializePhase:    Sum: 14us 99% C.I. 14us-14us Avg: 14us Max: 14us  
ProcessCards:    Sum: 11us 99% C.I. 4us-7us Avg: 5.500us Max: 7us  
PreCleanCards:    Sum: 7us 99% C.I. 7us-7us Avg: 7us Max: 7us  
(Paused)ScanGrayZygoteSpaceObjects:    Sum: 6us 99% C.I. 6us-6us Avg: 6us Max: 6us
ProcessReferences:    Sum: 5us 99% C.I. 5us-5us Avg: 5us Max: 5us  
MarkRoots:    Sum: 4us 99% C.I. 4us-4us Avg: 4us Max: 4us  
UnBindBitmaps:    Sum: 3us 99% C.I. 3us-3us Avg: 3us Max: 3us  
RecordFree:    Sum: 2us 99% C.I. 2us-2us Avg: 2us Max: 2us  
ForwardSoftReferences:    Sum: 1us 99% C.I. 1us-1us Avg: 1us Max: 1us  
(Paused)ProcessMarkStack:    Sum: 0 99% C.I. 0ns-0ns Avg: 0ns Max: 0ns
Done Dumping histograms  
sticky concurrent mark sweep paused:    Sum: 1.669ms 99% C.I. 1.669ms-1.669ms Avg: 1.669ms Max: 1.669ms  
sticky concurrent mark sweep total time: 26.304ms mean time: 26.304ms  
sticky concurrent mark sweep freed: 40059 objects with total size 2MB  
sticky concurrent mark sweep throughput: 1.54073e+06/s / 112MB/s  
Start Dumping histograms for 2 iterations for marksweep + semispace  
MarkRoots:    Sum: 76.622ms 99% C.I. 31.604ms-45.018ms Avg: 38.311ms Max: 45.018ms  
ProcessMarkStack:    Sum: 51.438ms 99% C.I. 0.008ms-30.576ms Avg: 12.859ms Max: 30.576ms  
ClearCardTable:    Sum: 12.596ms 99% C.I. 2.902ms-9.633ms Avg: 6.298ms Max: 9.694ms  
UpdateAndMarkImageModUnionTable:    Sum: 5.851ms 99% C.I. 1.762ms-4.077ms Avg: 2.925ms Max: 4.089ms  
MarkStackAsLive:    Sum: 2.255ms 99% C.I. 0.104ms-2.151ms Avg: 1.127ms Max: 2.151ms  
UpdateAndMarkZygoteModUnionTable:    Sum: 992us 99% C.I. 477us-515us Avg: 496us Max: 515us  
(Paused)EnqueueFinalizerReferences:    Sum: 819us 99% C.I. 51us-768us Avg: 409.500us Max: 768us
SweepSystemWeaks:    Sum: 570us 99% C.I. 263us-307us Avg: 285us Max: 307us  
RevokeAllThreadLocalBuffers:    Sum: 519us 99% C.I. 65us-193us Avg: 129.750us Max: 193us  
SweepLargeObjects:    Sum: 353us 99% C.I. 25us-328us Avg: 176.500us Max: 328us  
ImageModUnionClearCards:    Sum: 204us 99% C.I. 82us-122us Avg: 102us Max: 122us  
FinishPhase:    Sum: 174us 99% C.I. 58us-116us Avg: 87us Max: 116us  
SweepAllocSpace:    Sum: 105us 99% C.I. 35us-70us Avg: 52.500us Max: 70us  
(Paused)ProcessReferences:    Sum: 43us 99% C.I. 17us-26us Avg: 21.500us Max: 26us
SwapBitmaps:    Sum: 39us 99% C.I. 16us-23us Avg: 19.500us Max: 23us  
MarkReachableObjects:    Sum: 32us 99% C.I. 14us-18us Avg: 16us Max: 18us  
BindBitmaps:    Sum: 20us 99% C.I. 7us-13us Avg: 10us Max: 13us  
ProcessCards:    Sum: 16us 99% C.I. 5us-11us Avg: 8us Max: 11us  
Sweep:    Sum: 14us 99% C.I. 6us-8us Avg: 7us Max: 8us  
MarkingPhase:    Sum: 10us 99% C.I. 2us-8us Avg: 5us Max: 8us  
InitializePhase:    Sum: 9us 99% C.I. 2us-7us Avg: 4.500us Max: 7us  
ReclaimPhase:    Sum: 7us 99% C.I. 3us-4us Avg: 3.500us Max: 4us  
PreSweepingGcVerification:    Sum: 3us 99% C.I. 1us-2us Avg: 1.500us Max: 2us  
SweepZygoteSpace:    Sum: 2us 99% C.I. 1us-1us Avg: 1us Max: 1us  
PreGcVerificationPaused:    Sum: 1us 99% C.I. 250ns-1000ns Avg: 500ns Max: 1000ns  
PostGcVerificationPaused:    Sum: 0 99% C.I. 0ns-0ns Avg: 0ns Max: 0ns  
Done Dumping histograms  
marksweep + semispace paused:    Sum: 153.066ms 99% C.I. 71.381ms-81.685ms Avg: 76.533ms Max: 81.685ms  
marksweep + semispace total time: 152.772ms mean time: 76.386ms  
marksweep + semispace freed: 85295 objects with total size 4MB  
marksweep + semispace throughput: 561151/s / 32MB/s  
Total time spent in GC: 214.263ms  
Mean GC size throughput: 15MB/s  
Mean GC object throughput: 233017 objects/s  
Total number of allocations 105885  
Total bytes allocated 26MB  
Total bytes freed 3MB  
Free memory 5MB  
Free memory until GC 5MB  
Free memory until OOME 233MB  
Total memory 28MB  
Max memory 256MB  
Zygote space size 1540KB  
Total mutator paused time: 158.103ms  
Total time waiting for GC to complete: 5.416us  
Total GC count: 4  
Total GC time: 214.263ms  
Total blocking GC count: 0  
Total blocking GC time: 0  
Histogram of GC count per 10000 ms: 0:10,1:1,2:1  
Histogram of blocking GC count per 10000 ms: 0:12

suspend all histogram:    Sum: 3.470ms 99% C.I. 1us-1882.560us Avg: 266.923us Max: 1896us  
DALVIK THREADS (21):  
"Signal Catcher" daemon prio=5 tid=2 Runnable
  | group="system" sCount=0 dsCount=0 obj=0x12c0d0a0 self=0x55999df770
  | sysTid=25972 nice=0 cgrp=top_visible sched=0/0 handle=0x7f78de7450
  | state=R schedstat=( 6967792 0 10 ) utm=0 stm=0 core=3 HZ=100
  | stack=0x7f78ceb000-0x7f78ced000 stackSize=1013KB
  | held mutexes= "mutator lock"(shared held)
  native: #00 pc 000000000048df54  /system/lib64/libart.so (_ZN3art15DumpNativeStackERNSt3__113basic_ostreamIcNS0_11char_traitsIcEEEEiPKcPNS_9ArtMethodEPv+236)
  native: #01 pc 000000000045d404  /system/lib64/libart.so (_ZNK3art6Thread4DumpERNSt3__113basic_ostreamIcNS1_11char_traitsIcEEEE+220)
  native: #02 pc 0000000000469c70  /system/lib64/libart.so (_ZN3art14DumpCheckpoint3RunEPNS_6ThreadE+688)
  native: #03 pc 000000000046ab8c  /system/lib64/libart.so (_ZN3art10ThreadList13RunCheckpointEPNS_7ClosureE+276)
  native: #04 pc 000000000046b248  /system/lib64/libart.so (_ZN3art10ThreadList4DumpERNSt3__113basic_ostreamIcNS1_11char_traitsIcEEEE+188)
  native: #05 pc 000000000046bb2c  /system/lib64/libart.so (_ZN3art10ThreadList14DumpForSigQuitERNSt3__113basic_ostreamIcNS1_11char_traitsIcEEEE+492)
  native: #06 pc 0000000000435064  /system/lib64/libart.so (_ZN3art7Runtime14DumpForSigQuitERNSt3__113basic_ostreamIcNS1_11char_traitsIcEEEE+96)
  native: #07 pc 000000000044281c  /system/lib64/libart.so (_ZN3art13SignalCatcher13HandleSigQuitEv+1256)
  native: #08 pc 000000000044342c  /system/lib64/libart.so (_ZN3art13SignalCatcher3RunEPv+452)
  native: #09 pc 0000000000068464  /system/lib64/libc.so (_ZL15__pthread_startPv+52)
  native: #10 pc 000000000001d544  /system/lib64/libc.so (__start_thread+16)
  (no managed stack frames)

"main" prio=5 tid=1 Blocked
  | group="main" sCount=1 dsCount=0 obj=0x7477dea0 self=0x5597d71b70
  | sysTid=25967 nice=-1 cgrp=top_visible sched=0/0 handle=0x7f7dcb5000
  | state=S schedstat=( 688872496 20759648 497 ) utm=57 stm=11 core=0 HZ=100
  | stack=0x7fc4bc9000-0x7fc4bcb000 stackSize=8MB
  | held mutexes=
  at com.google.android.gms.tagmanager.zzp.zzav(unavailable:-1)
  - waiting to lock <0x08362446> (a com.google.android.gms.tagmanager.zzp) held by thread 15
  at com.google.android.gms.tagmanager.zzp.zza(unavailable:-1)
  at com.google.android.gms.tagmanager.zzp$zzd.zzOE(unavailable:-1)
  at com.google.android.gms.tagmanager.zzo.refresh(unavailable:-1)
  - locked <0x023ef607> (a com.google.android.gms.tagmanager.zzo)
  at com.example.MyApplication$1.onSuccess(MyApplication.java:176)
  at com.example.MyApplication$1.onSuccess(MyApplication.java:169)
  at com.example.callback.GenericCallBack.handleSuccess(GenericCallBack.java:39)
  at com.example.service.AnalyticsServiceImpl$1.onResult(AnalyticsServiceImpl.java:74)
  at com.example.service.AnalyticsServiceImpl$1.onResult(AnalyticsServiceImpl.java:65)
  at com.google.android.gms.internal.zzzx$zza.zzb(unavailable:-1)
  at com.google.android.gms.internal.zzzx$zza.handleMessage(unavailable:-1)
  at android.os.Handler.dispatchMessage(Handler.java:102)
  at android.os.Looper.loop(Looper.java:150)
  at android.app.ActivityThread.main(ActivityThread.java:5546)
  at java.lang.reflect.Method.invoke!(Native method)
  at com.android.internal.os.ZygoteInit$MethodAndArgsCaller.run(ZygoteInit.java:794)
  at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:684)

"ReferenceQueueDaemon" daemon prio=5 tid=3 Waiting
  | group="system" sCount=1 dsCount=0 obj=0x12c0d100 self=0x559992ff30
  | sysTid=25973 nice=0 cgrp=top_visible sched=0/0 handle=0x7f78cdf450
  | state=S schedstat=( 1998256 1180608 19 ) utm=0 stm=0 core=0 HZ=100
  | stack=0x7f78bdd000-0x7f78bdf000 stackSize=1037KB
  | held mutexes=
  at java.lang.Object.wait!(Native method)
  - waiting on <0x042f1334> (a java.lang.Class)
  at java.lang.Daemons$ReferenceQueueDaemon.run(Daemons.java:147)
  - locked <0x042f1334> (a java.lang.Class)
  at java.lang.Thread.run(Thread.java:833)

"FinalizerDaemon" daemon prio=5 tid=4 Waiting
  | group="system" sCount=1 dsCount=0 obj=0x12c0d160 self=0x5597d6e0d0
  | sysTid=25974 nice=0 cgrp=top_visible sched=0/0 handle=0x7f78bd3450
  | state=S schedstat=( 9917648 1193088 41 ) utm=0 stm=0 core=1 HZ=100
  | stack=0x7f78ad1000-0x7f78ad3000 stackSize=1037KB
  | held mutexes=
  at java.lang.Object.wait!(Native method)
  - waiting on <0x0e1d355d> (a java.lang.ref.ReferenceQueue)
  at java.lang.Object.wait(Object.java:423)
  at java.lang.ref.ReferenceQueue.remove(ReferenceQueue.java:101)
  - locked <0x0e1d355d> (a java.lang.ref.ReferenceQueue)
  at java.lang.ref.ReferenceQueue.remove(ReferenceQueue.java:72)
  at java.lang.Daemons$FinalizerDaemon.run(Daemons.java:185)
  at java.lang.Thread.run(Thread.java:833)

"HeapTaskDaemon" daemon prio=5 tid=5 Blocked
  | group="system" sCount=1 dsCount=0 obj=0x12c0d1c0 self=0x55995bd630
  | sysTid=25976 nice=0 cgrp=top_visible sched=0/0 handle=0x7f789c0450
  | state=S schedstat=( 209432704 2489136 42 ) utm=19 stm=1 core=4 HZ=100
  | stack=0x7f788be000-0x7f788c0000 stackSize=1037KB
  | held mutexes=
  kernel: (couldn't read /proc/self/task/25976/stack)
  native: #00 pc 000000000001a8d4  /system/lib64/libc.so (syscall+28)
  native: #01 pc 000000000013a02c  /system/lib64/libart.so (_ZN3art17ConditionVariable4WaitEPNS_6ThreadE+136)
  native: #02 pc 000000000026a1f4  /system/lib64/libart.so (_ZN3art2gc13TaskProcessor7GetTaskEPNS_6ThreadE+128)
  native: #03 pc 000000000026a85c  /system/lib64/libart.so (_ZN3art2gc13TaskProcessor11RunAllTasksEPNS_6ThreadE+120)
  native: #04 pc 000000000000054c  /data/dalvik-cache/arm64/system@framework@boot.oat (Java_dalvik_system_VMRuntime_runHeapTasks__+128)
  at dalvik.system.VMRuntime.runHeapTasks(Native method)
  - waiting to lock an unknown object
  at java.lang.Daemons$HeapTaskDaemon.run(Daemons.java:355)
  at java.lang.Thread.run(Thread.java:833)

"FinalizerWatchdogDaemon" daemon prio=5 tid=6 Waiting
  | group="system" sCount=1 dsCount=0 obj=0x12c0d220 self=0x559855a550
  | sysTid=25975 nice=0 cgrp=top_visible sched=0/0 handle=0x7f78acc450
  | state=S schedstat=( 1235520 0 10 ) utm=0 stm=0 core=2 HZ=100
  | stack=0x7f789ca000-0x7f789cc000 stackSize=1037KB
  | held mutexes=
  at java.lang.Object.wait!(Native method)
  - waiting on <0x0ba03ed2> (a java.lang.Daemons$FinalizerWatchdogDaemon)
  at java.lang.Daemons$FinalizerWatchdogDaemon.waitForObject(Daemons.java:255)
  - locked <0x0ba03ed2> (a java.lang.Daemons$FinalizerWatchdogDaemon)
  at java.lang.Daemons$FinalizerWatchdogDaemon.run(Daemons.java:227)
  at java.lang.Thread.run(Thread.java:833)

"Binder_1" prio=5 tid=7 Native
  | group="main" sCount=1 dsCount=0 obj=0x12c0d280 self=0x5599938310
  | sysTid=25977 nice=0 cgrp=top_visible sched=0/0 handle=0x7f63b1e450
  | state=S schedstat=( 6009328 2418208 23 ) utm=0 stm=0 core=3 HZ=100
  | stack=0x7f63a22000-0x7f63a24000 stackSize=1013KB
  | held mutexes=
  kernel: (couldn't read /proc/self/task/25977/stack)
  native: #00 pc 000000000006af08  /system/lib64/libc.so (__ioctl+4)
  native: #01 pc 0000000000074a00  /system/lib64/libc.so (ioctl+100)
  native: #02 pc 000000000002d494  /system/lib64/libbinder.so (_ZN7android14IPCThreadState14talkWithDriverEb+164)
  native: #03 pc 000000000002dce8  /system/lib64/libbinder.so (_ZN7android14IPCThreadState20getAndExecuteCommandEv+24)
  native: #04 pc 000000000002de04  /system/lib64/libbinder.so (_ZN7android14IPCThreadState14joinThreadPoolEb+76)
  native: #05 pc 0000000000036858  /system/lib64/libbinder.so (???)
  native: #06 pc 0000000000016394  /system/lib64/libutils.so (_ZN7android6Thread11_threadLoopEPv+208)
  native: #07 pc 0000000000092bf0  /system/lib64/libandroid_runtime.so (_ZN7android14AndroidRuntime15javaThreadShellEPv+96)
  native: #08 pc 0000000000015be4  /system/lib64/libutils.so (???)
  native: #09 pc 0000000000068464  /system/lib64/libc.so (_ZL15__pthread_startPv+52)
  native: #10 pc 000000000001d544  /system/lib64/libc.so (__start_thread+16)
  (no managed stack frames)

"Binder_2" prio=5 tid=8 Native
  | group="main" sCount=1 dsCount=0 obj=0x12c0d2e0 self=0x5599935b10
  | sysTid=25978 nice=0 cgrp=top_visible sched=0/0 handle=0x7f63a1f450
  | state=S schedstat=( 5317520 869440 23 ) utm=0 stm=0 core=1 HZ=100
  | stack=0x7f63923000-0x7f63925000 stackSize=1013KB
  | held mutexes=
  kernel: (couldn't read /proc/self/task/25978/stack)
  native: #00 pc 000000000006af08  /system/lib64/libc.so (__ioctl+4)
  native: #01 pc 0000000000074a00  /system/lib64/libc.so (ioctl+100)
  native: #02 pc 000000000002d494  /system/lib64/libbinder.so (_ZN7android14IPCThreadState14talkWithDriverEb+164)
  native: #03 pc 000000000002dce8  /system/lib64/libbinder.so (_ZN7android14IPCThreadState20getAndExecuteCommandEv+24)
  native: #04 pc 000000000002de04  /system/lib64/libbinder.so (_ZN7android14IPCThreadState14joinThreadPoolEb+76)
  native: #05 pc 0000000000036858  /system/lib64/libbinder.so (???)
  native: #06 pc 0000000000016394  /system/lib64/libutils.so (_ZN7android6Thread11_threadLoopEPv+208)
  native: #07 pc 0000000000092bf0  /system/lib64/libandroid_runtime.so (_ZN7android14AndroidRuntime15javaThreadShellEPv+96)
  native: #08 pc 0000000000015be4  /system/lib64/libutils.so (???)
  native: #09 pc 0000000000068464  /system/lib64/libc.so (_ZL15__pthread_startPv+52)
  native: #10 pc 000000000001d544  /system/lib64/libc.so (__start_thread+16)
  (no managed stack frames)

"pool-5-thread-1" prio=5 tid=11 Waiting
  | group="main" sCount=1 dsCount=0 obj=0x12c0d340 self=0x5598b73790
  | sysTid=26000 nice=0 cgrp=top_visible sched=0/0 handle=0x7f61cf6450
  | state=S schedstat=( 15990624 1283568 26 ) utm=1 stm=0 core=2 HZ=100
  | stack=0x7f61bf4000-0x7f61bf6000 stackSize=1037KB
  | held mutexes=
  at java.lang.Object.wait!(Native method)
  - waiting on <0x0ecaeba3> (a java.lang.Object)
  at java.lang.Thread.parkFor$(Thread.java:1235)
  - locked <0x0ecaeba3> (a java.lang.Object)
  at sun.misc.Unsafe.park(Unsafe.java:299)
  at java.util.concurrent.locks.LockSupport.park(LockSupport.java:158)
  at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.await(AbstractQueuedSynchronizer.java:2013)
  at java.util.concurrent.LinkedBlockingQueue.take(LinkedBlockingQueue.java:410)
  at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1036)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1098)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:588)
  at java.lang.Thread.run(Thread.java:833)

"pool-6-thread-1" prio=5 tid=12 Waiting
  | group="main" sCount=1 dsCount=0 obj=0x12c0d3a0 self=0x55999908b0
  | sysTid=26001 nice=0 cgrp=top_visible sched=0/0 handle=0x7f61bf1450
  | state=S schedstat=( 13138944 1240720 14 ) utm=1 stm=0 core=6 HZ=100
  | stack=0x7f61aef000-0x7f61af1000 stackSize=1037KB
  | held mutexes=
  at java.lang.Object.wait!(Native method)
  - waiting on <0x03442ea0> (a java.lang.Object)
  at java.lang.Thread.parkFor$(Thread.java:1235)
  - locked <0x03442ea0> (a java.lang.Object)
  at sun.misc.Unsafe.park(Unsafe.java:299)
  at java.util.concurrent.locks.LockSupport.park(LockSupport.java:158)
  at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.await(AbstractQueuedSynchronizer.java:2013)
  at java.util.concurrent.LinkedBlockingQueue.take(LinkedBlockingQueue.java:410)
  at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1036)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1098)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:588)
  at java.lang.Thread.run(Thread.java:833)

"Thread-1658" prio=5 tid=13 TimedWaiting
  | group="main" sCount=1 dsCount=0 obj=0x12c0d400 self=0x5598ca44c0
  | sysTid=26002 nice=10 cgrp=top_visible sched=0/0 handle=0x7f61aec450
  | state=S schedstat=( 18796336 3701568 40 ) utm=1 stm=0 core=2 HZ=100
  | stack=0x7f619ea000-0x7f619ec000 stackSize=1037KB
  | held mutexes=
  at java.lang.Object.wait!(Native method)
  - waiting on <0x09fcca59> (a java.lang.Object)
  at java.lang.Object.wait(Object.java:423)
  at com.google.android.gms.tagmanager.zza.zzOu(unavailable:-1)
  - locked <0x09fcca59> (a java.lang.Object)
  at com.google.android.gms.tagmanager.zza.zzb(unavailable:-1)
  at com.google.android.gms.tagmanager.zza$2.run(unavailable:-1)
  at java.lang.Thread.run(Thread.java:833)

"pool-7-thread-1" prio=5 tid=14 Waiting
  | group="main" sCount=1 dsCount=0 obj=0x12c0d460 self=0x55985c4f50
  | sysTid=26005 nice=0 cgrp=top_visible sched=0/0 handle=0x7f6174b450
  | state=S schedstat=( 12652016 149760 18 ) utm=1 stm=0 core=3 HZ=100
  | stack=0x7f61649000-0x7f6164b000 stackSize=1037KB
  | held mutexes=
  at java.lang.Object.wait!(Native method)
  - waiting on <0x001f761e> (a java.lang.Object)
  at java.lang.Thread.parkFor$(Thread.java:1235)
  - locked <0x001f761e> (a java.lang.Object)
  at sun.misc.Unsafe.park(Unsafe.java:299)
  at java.util.concurrent.locks.LockSupport.park(LockSupport.java:158)
  at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.await(AbstractQueuedSynchronizer.java:2013)
  at java.util.concurrent.LinkedBlockingQueue.take(LinkedBlockingQueue.java:410)
  at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1036)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1098)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:588)
  at java.lang.Thread.run(Thread.java:833)

"pool-8-thread-1" prio=5 tid=15 Blocked
  | group="main" sCount=1 dsCount=0 obj=0x12c0d4c0 self=0x5598cd6440
  | sysTid=26006 nice=0 cgrp=top_visible sched=0/0 handle=0x7f61646450
  | state=S schedstat=( 64174864 5461664 62 ) utm=6 stm=0 core=3 HZ=100
  | stack=0x7f61544000-0x7f61546000 stackSize=1037KB
  | held mutexes=
  at com.google.android.gms.tagmanager.zzo.zza(unavailable:-1)
  - waiting to lock <0x023ef607> (a com.google.android.gms.tagmanager.zzo) held by thread 1
  at com.google.android.gms.tagmanager.zzp.zza(unavailable:-1)
  - locked <0x08362446> (a com.google.android.gms.tagmanager.zzp)
  at com.google.android.gms.tagmanager.zzp.zza(unavailable:-1)
  at com.google.android.gms.tagmanager.zzp$zzc.zzb(unavailable:-1)
  - locked <0x08362446> (a com.google.android.gms.tagmanager.zzp)
  at com.google.android.gms.tagmanager.zzp$zzc.onSuccess(unavailable:-1)
  at com.google.android.gms.tagmanager.zzct.zzPD(unavailable:-1)
  at com.google.android.gms.tagmanager.zzct.run(unavailable:-1)
  at java.util.concurrent.Executors$RunnableAdapter.call(Executors.java:423)
  at java.util.concurrent.FutureTask.run(FutureTask.java:237)
  at java.util.concurrent.ScheduledThreadPoolExecutor$ScheduledFutureTask.access$201(ScheduledThreadPoolExecutor.java:154)
  at java.util.concurrent.ScheduledThreadPoolExecutor$ScheduledFutureTask.run(ScheduledThreadPoolExecutor.java:269)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1113)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:588)
  at java.lang.Thread.run(Thread.java:833)

"WonderPush" prio=5 tid=16 Native
  | group="main" sCount=1 dsCount=0 obj=0x12c0d520 self=0x5598c83520
  | sysTid=26007 nice=0 cgrp=top_visible sched=0/0 handle=0x7f61541450
  | state=S schedstat=( 86143824 20670208 187 ) utm=8 stm=0 core=0 HZ=100
  | stack=0x7f6143f000-0x7f61441000 stackSize=1037KB
  | held mutexes=
  kernel: (couldn't read /proc/self/task/26007/stack)
  native: #00 pc 000000000006b674  /system/lib64/libc.so (__epoll_pwait+8)
  native: #01 pc 000000000001dba4  /system/lib64/libc.so (epoll_pwait+32)
  native: #02 pc 000000000001b96c  /system/lib64/libutils.so (_ZN7android6Looper9pollInnerEi+144)
  native: #03 pc 000000000001bd4c  /system/lib64/libutils.so (_ZN7android6Looper8pollOnceEiPiS1_PPv+80)
  native: #04 pc 00000000000d7e2c  /system/lib64/libandroid_runtime.so (_ZN7android18NativeMessageQueue8pollOnceEP7_JNIEnvP8_jobjecti+48)
  native: #05 pc 000000000000082c  /data/dalvik-cache/arm64/system@framework@boot.oat (Java_android_os_MessageQueue_nativePollOnce__JI+144)
  at android.os.MessageQueue.nativePollOnce(Native method)
  at android.os.MessageQueue.next(MessageQueue.java:330)
  at android.os.Looper.loop(Looper.java:137)
  at com.wonderpush.sdk.WonderPush$1.run(WonderPush.java:67)
  at java.lang.Thread.run(Thread.java:833)

"Thread-1663" prio=5 tid=17 Sleeping
  | group="main" sCount=1 dsCount=0 obj=0x12c0d580 self=0x5598c95130
  | sysTid=26009 nice=0 cgrp=top_visible sched=0/0 handle=0x7f6143c450
  | state=S schedstat=( 2586896 703248 4 ) utm=0 stm=0 core=2 HZ=100
  | stack=0x7f6133a000-0x7f6133c000 stackSize=1037KB
  | held mutexes=
  at java.lang.Thread.sleep!(Native method)
  - sleeping on <0x0c969eff> (a java.lang.Object)
  at java.lang.Thread.sleep(Thread.java:1046)
  - locked <0x0c969eff> (a java.lang.Object)
  at java.lang.Thread.sleep(Thread.java:1000)
  at com.wonderpush.sdk.WonderPushRequestVault$1.run(WonderPushRequestVault.java:78)
  at java.lang.Thread.run(Thread.java:833)

"Okio Watchdog" daemon prio=5 tid=18 Waiting
  | group="main" sCount=1 dsCount=0 obj=0x12c0d5e0 self=0x55999f4a50
  | sysTid=26014 nice=0 cgrp=top_visible sched=0/0 handle=0x7f59452450
  | state=S schedstat=( 590928 0 4 ) utm=0 stm=0 core=3 HZ=100
  | stack=0x7f59350000-0x7f59352000 stackSize=1037KB
  | held mutexes=
  at java.lang.Object.wait!(Native method)
  - waiting on <0x08cd94cc> (a java.lang.Class)
  at com.android.okhttp.okio.AsyncTimeout.awaitTimeout(AsyncTimeout.java:311)
  - locked <0x08cd94cc> (a java.lang.Class)
  at com.android.okhttp.okio.AsyncTimeout.access$000(AsyncTimeout.java:40)
  at com.android.okhttp.okio.AsyncTimeout$Watchdog.run(AsyncTimeout.java:286)

"OkHttp ConnectionPool" daemon prio=5 tid=19 TimedWaiting
  | group="main" sCount=1 dsCount=0 obj=0x12c0d640 self=0x5599b2edb0
  | sysTid=26016 nice=0 cgrp=top_visible sched=0/0 handle=0x7f5934d450
  | state=S schedstat=( 455936 0 3 ) utm=0 stm=0 core=1 HZ=100
  | stack=0x7f5924b000-0x7f5924d000 stackSize=1037KB
  | held mutexes=
  at java.lang.Object.wait!(Native method)
  - waiting on <0x04a6b315> (a com.android.okhttp.ConnectionPool)
  at com.android.okhttp.ConnectionPool.performCleanup(ConnectionPool.java:305)
  - locked <0x04a6b315> (a com.android.okhttp.ConnectionPool)
  at com.android.okhttp.ConnectionPool.runCleanupUntilPoolIsEmpty(ConnectionPool.java:242)
  at com.android.okhttp.ConnectionPool.access$000(ConnectionPool.java:54)
  at com.android.okhttp.ConnectionPool$1.run(ConnectionPool.java:97)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1113)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:588)
  at java.lang.Thread.run(Thread.java:833)

"RenderThread" prio=5 tid=20 Native
  | group="main" sCount=1 dsCount=0 obj=0x12c0d6a0 self=0x5599ba3900
  | sysTid=26017 nice=-4 cgrp=top_visible sched=0/0 handle=0x7f59248450
  | state=S schedstat=( 2012608 2358720 12 ) utm=0 stm=0 core=3 HZ=100
  | stack=0x7f5914c000-0x7f5914e000 stackSize=1013KB
  | held mutexes=
  kernel: (couldn't read /proc/self/task/26017/stack)
  native: #00 pc 000000000006b674  /system/lib64/libc.so (__epoll_pwait+8)
  native: #01 pc 000000000001dba4  /system/lib64/libc.so (epoll_pwait+32)
  native: #02 pc 000000000001b96c  /system/lib64/libutils.so (_ZN7android6Looper9pollInnerEi+144)
  native: #03 pc 000000000001bd4c  /system/lib64/libutils.so (_ZN7android6Looper8pollOnceEiPiS1_PPv+80)
  native: #04 pc 000000000002c420  /system/lib64/libhwui.so (_ZN7android10uirenderer12renderthread12RenderThread10threadLoopEv+100)
  native: #05 pc 0000000000016394  /system/lib64/libutils.so (_ZN7android6Thread11_threadLoopEPv+208)
  native: #06 pc 0000000000092bf0  /system/lib64/libandroid_runtime.so (_ZN7android14AndroidRuntime15javaThreadShellEPv+96)
  native: #07 pc 0000000000015be4  /system/lib64/libutils.so (???)
  native: #08 pc 0000000000068464  /system/lib64/libc.so (_ZL15__pthread_startPv+52)
  native: #09 pc 000000000001d544  /system/lib64/libc.so (__start_thread+16)
  (no managed stack frames)

"Okio Watchdog" daemon prio=5 tid=21 Waiting
  | group="main" sCount=1 dsCount=0 obj=0x12c0d700 self=0x559852e380
  | sysTid=26018 nice=0 cgrp=top_visible sched=0/0 handle=0x7f5823f450
  | state=S schedstat=( 833040 194480 5 ) utm=0 stm=0 core=0 HZ=100
  | stack=0x7f5813d000-0x7f5813f000 stackSize=1037KB
  | held mutexes=
  at java.lang.Object.wait!(Native method)
  - waiting on <0x0cc8d62a> (a java.lang.Class)
  at okio.AsyncTimeout.awaitTimeout(AsyncTimeout.java:311)
  - locked <0x0cc8d62a> (a java.lang.Class)
  at okio.AsyncTimeout.access$000(AsyncTimeout.java:40)
  at okio.AsyncTimeout$Watchdog.run(AsyncTimeout.java:286)

"IntentService[WonderPushRegistrationIntentService]" prio=5 tid=22 Native
  | group="main" sCount=1 dsCount=0 obj=0x12c0d760 self=0x5599b3b010
  | sysTid=26019 nice=0 cgrp=top_visible sched=0/0 handle=0x7f5813a450
  | state=S schedstat=( 4619888 974688 10 ) utm=0 stm=0 core=4 HZ=100
  | stack=0x7f58038000-0x7f5803a000 stackSize=1037KB
  | held mutexes=
  kernel: (couldn't read /proc/self/task/26019/stack)
  native: #00 pc 000000000006b674  /system/lib64/libc.so (__epoll_pwait+8)
  native: #01 pc 000000000001dba4  /system/lib64/libc.so (epoll_pwait+32)
  native: #02 pc 000000000001b96c  /system/lib64/libutils.so (_ZN7android6Looper9pollInnerEi+144)
  native: #03 pc 000000000001bd4c  /system/lib64/libutils.so (_ZN7android6Looper8pollOnceEiPiS1_PPv+80)
  native: #04 pc 00000000000d7e2c  /system/lib64/libandroid_runtime.so (_ZN7android18NativeMessageQueue8pollOnceEP7_JNIEnvP8_jobjecti+48)
  native: #05 pc 000000000000082c  /data/dalvik-cache/arm64/system@framework@boot.oat (Java_android_os_MessageQueue_nativePollOnce__JI+144)
  at android.os.MessageQueue.nativePollOnce(Native method)
  at android.os.MessageQueue.next(MessageQueue.java:330)
  at android.os.Looper.loop(Looper.java:137)
  at android.os.HandlerThread.run(HandlerThread.java:61)

"OkHttp ConnectionPool" daemon prio=5 tid=23 TimedWaiting
  | group="main" sCount=1 dsCount=0 obj=0x12c0d7c0 self=0x5599bbd9e0
  | sysTid=26021 nice=0 cgrp=top_visible sched=0/0 handle=0x7f58035450
  | state=S schedstat=( 798928 0 2 ) utm=0 stm=0 core=3 HZ=100
  | stack=0x7f57f33000-0x7f57f35000 stackSize=1037KB
  | held mutexes=
  at java.lang.Object.wait!(Native method)
  - waiting on <0x01402c1b> (a com.squareup.okhttp.ConnectionPool)
  at com.squareup.okhttp.ConnectionPool.performCleanup(ConnectionPool.java:305)
  - locked <0x01402c1b> (a com.squareup.okhttp.ConnectionPool)
  at com.squareup.okhttp.ConnectionPool.runCleanupUntilPoolIsEmpty(ConnectionPool.java:242)
  at com.squareup.okhttp.ConnectionPool.access$000(ConnectionPool.java:54)
  at com.squareup.okhttp.ConnectionPool$1.run(ConnectionPool.java:97)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1113)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:588)
  at java.lang.Thread.run(Thread.java:833)

----- end 25967 -----
```

That does look intimidating, doesn't it? But let's not throw in the towel just yet, we Android developers are made of sterner stuff :)

Let's start by looking at the "main" thread:

```logcat
"main" prio=5 tid=1 Blocked
  | group="main" sCount=1 dsCount=0 obj=0x7477dea0 self=0x5597d71b70
  | sysTid=25967 nice=-1 cgrp=top_visible sched=0/0 handle=0x7f7dcb5000
  | state=S schedstat=( 688872496 20759648 497 ) utm=57 stm=11 core=0 HZ=100
  | stack=0x7fc4bc9000-0x7fc4bcb000 stackSize=8MB
  | held mutexes=
  at com.google.android.gms.tagmanager.zzp.zzav(unavailable:-1)
  - waiting to lock <0x08362446> (a com.google.android.gms.tagmanager.zzp) held by thread 15
  at com.google.android.gms.tagmanager.zzp.zza(unavailable:-1)
  at com.google.android.gms.tagmanager.zzp$zzd.zzOE(unavailable:-1)
  at com.google.android.gms.tagmanager.zzo.refresh(unavailable:-1)
  - locked <0x023ef607> (a com.google.android.gms.tagmanager.zzo)
  at com.example.MyApplication$1.onSuccess(MyApplication.java:176)
  at com.example.MyApplication$1.onSuccess(MyApplication.java:169)
  at com.example.callback.GenericCallBack.handleSuccess(GenericCallBack.java:39)
  at com.example.service.AnalyticsServiceImpl$1.onResult(AnalyticsServiceImpl.java:74)
  at com.example.service.AnalyticsServiceImpl$1.onResult(AnalyticsServiceImpl.java:65)
  at com.google.android.gms.internal.zzzx$zza.zzb(unavailable:-1)
  at com.google.android.gms.internal.zzzx$zza.handleMessage(unavailable:-1)
  at android.os.Handler.dispatchMessage(Handler.java:102)
  at android.os.Looper.loop(Looper.java:150)
  at android.app.ActivityThread.main(ActivityThread.java:5546)
  at java.lang.reflect.Method.invoke!(Native method)
  at com.android.internal.os.ZygoteInit$MethodAndArgsCaller.run(ZygoteInit.java:794)
  at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:684)
```

The first line tells us that the main thread is indeed blocked, and scrolling down a little we can see the immediate cause: A line in the Google Tagmanager library is trying to acquire a lock that is held by thread 15:

```logcat
  at com.google.android.gms.tagmanager.zzp.zzav(unavailable:-1)
  - waiting to lock <0x08362446> (a com.google.android.gms.tagmanager.zzp) held by thread 15
```

The exact line in question and the name of the lock are sadly unknown because the Google Tagmanager library is obfuscated (hence all the `zzp` and `zzav` names) but the line still points to the next culprit in the chain: the thread with ID 15.

Let's now search for the thread 15 in the trace:

```logcat
"pool-8-thread-1" prio=5 tid=15 Blocked
  | group="main" sCount=1 dsCount=0 obj=0x12c0d4c0 self=0x5598cd6440
  | sysTid=26006 nice=0 cgrp=top_visible sched=0/0 handle=0x7f61646450
  | state=S schedstat=( 64174864 5461664 62 ) utm=6 stm=0 core=3 HZ=100
  | stack=0x7f61544000-0x7f61546000 stackSize=1037KB
  | held mutexes=
  at com.google.android.gms.tagmanager.zzo.zza(unavailable:-1)
  - waiting to lock <0x023ef607> (a com.google.android.gms.tagmanager.zzo) held by thread 1
  at com.google.android.gms.tagmanager.zzp.zza(unavailable:-1)
  - locked <0x08362446> (a com.google.android.gms.tagmanager.zzp)
  at com.google.android.gms.tagmanager.zzp.zza(unavailable:-1)
  at com.google.android.gms.tagmanager.zzp$zzc.zzb(unavailable:-1)
  - locked <0x08362446> (a com.google.android.gms.tagmanager.zzp)
  at com.google.android.gms.tagmanager.zzp$zzc.onSuccess(unavailable:-1)
  at com.google.android.gms.tagmanager.zzct.zzPD(unavailable:-1)
  at com.google.android.gms.tagmanager.zzct.run(unavailable:-1)
  at java.util.concurrent.Executors$RunnableAdapter.call(Executors.java:423)
  at java.util.concurrent.FutureTask.run(FutureTask.java:237)
  at java.util.concurrent.ScheduledThreadPoolExecutor$ScheduledFutureTask.access$201(ScheduledThreadPoolExecutor.java:154)
  at java.util.concurrent.ScheduledThreadPoolExecutor$ScheduledFutureTask.run(ScheduledThreadPoolExecutor.java:269)
  at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1113)
  at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:588)
  at java.lang.Thread.run(Thread.java:833)
```

Again the first line tells us the thread is blocked, so let's look at what is causing that blockage:

```logcat
  at com.google.android.gms.tagmanager.zzo.zza(unavailable:-1)
  - waiting to lock <0x023ef607> (a com.google.android.gms.tagmanager.zzo) held by thread 1
```

So, we see that this thread is waiting for another mutex (lovingly named `com.google.android.gms.tagmanager.zzo`) that is held by thread 1 (the "main" thread). Hence, we see that thread 1 is waiting for thread 15 while thread 15 is waiting for thread 1. We've found the deadlock!

## Fixing the deadlock

Looking at the stack trace of thread 15 reveals that it does not pass through our code at all. By the look of the trace (note names such as `ThreadPoolExecutor`, `FutureTask.run()` and `onSuccess`) we could put forward an educated guess that this is part of an internal callback that is run when a backend call returns successfully, but sadly we cannot know for sure since the library is not open-source (Boo, Google!).

Looking at the main thread stack though we see that we do pass through our own code:

```logcat
  at com.google.android.gms.tagmanager.zzo.refresh(unavailable:-1)
  - locked <0x023ef607> (a com.google.android.gms.tagmanager.zzo)
  at com.example.MyApplication$1.onSuccess(MyApplication.java:176)
  at com.example.MyApplication$1.onSuccess(MyApplication.java:169)
  at com.example.callback.GenericCallBack.handleSuccess(GenericCallBack.java:39)
```

Let's have a quick look at the code in question:

```java
    private void loadGTMContainer() {
        mAnalyticsService.loadGTMContainer(new GenericCallBack<ContainerHolder>() {

            @Override
            public void onSuccess(ContainerHolder result) {
                if (result != null) {
                    LOGGER.debug("Google Tag Manager container loaded successfully");
                    //manually refresh the container, just in case
                    result.refresh();
                    //...
                }
            }

            @Override
            public void onError(Exception e) {
                //...
            }
        });
    }
```

The stack seems to indicate that the deadlock happens inside `result.refresh()` and indeed, the line (and comment above) seems suspicious. It looks like the refresh line is optional (and causing the problem inside GTM) so there is a good chance that removing it will also remove the deadlock. But there is no way to be sure without being able to reliably reproduce the bug.

Now is the time to do some research on the topic of "Google Tag Manager", "deadlock" and "refresh". Sooner or later you should find this [link](https://productforums.google.com/forum/#!topic/tag-manager/wlPpNKPXvu8) from the google's own forum. In that obscure forum post from 2014 you can find one of the developers of the GTM library acknowledging there is a bug when

> the code in your application may be calling refresh on the container while it's in the middle of saving the latest version of the container retrieved from the network. [...] One way to reduce the chance of deadlock is to avoid calling refresh during the initial load, and on time intervals when the automatic refresh is taking place. The getting started documents don't go into this detail, but the container is automatically refreshed on an interval, which is currently set for every 12 hours. [...] We'll work on a fix for the deadlock in a future version.

As it often happens, the fix in the future version never came even 3 years later, and I wouldn't hold my breath for that happening any time soon. But it does seem to be the case that the `refresh()` call is indeed useless and that it is likely the culprit.

Next steps would be then to remove that line, do a full regression testing pass on the GTM library functionality and then release the new version, keeping a close eye on the stats where we hope to see the number of ANR sharply declining.

What if you are not that lucky to find some forum thread about your exact situation? Well, the problem is much complicated then, but at least knowing the culprit you have a few avenues of attacking the issue. If searching the internet for a solution would not yield a result, I would try the following (more or less in this order):

- post a thread in the library's official forum with the evidence I have
- ask on StackOverflow (and put a bounty on the question)
- try to rewrite the code in question
- if everything else fails, replace the faulty library with an alternative (in this case for example Firebase).

# Conclusion

In this article I've tried to demystify ANRs and Deadlocks a little bit and show how you can gain a bit of traction on analysing them and finding their ultimate cause. We've looked at what they are and how they function and we've also followed an example of how to fix such an issue. Hopefully this will prove useful to you in the future.

转载于[Deadlocks and ANRs](http://zenandroid.io/deadlocks-and-anrs/)