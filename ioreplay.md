

# Introduction #

**ioreplay** is a tool for replaying IO traces. It is capable of a very precise simulation of the original, traced application by issuing the same IO system calls in the same time. It is mainly useful as a standalone benchmark tool in situations, where running the original application is not possible (i.e. it requires a complicated system setup, licenses problems, input data problems etc.) or desirable.

The tool runs under Linux.

# Use case #

Imagine you have a complex application X that is quite IO intensive. It requires many libraries and it is rather hard to install, not to mention it processes data files that are not publicly available. You wonder whether using SSD disks instead of traditional hard drives could improve the performance of your application.

Your colleague (or a vendor) have proposed that he can run a simple benchmark to measure the performance difference on a new hardware he has.
Because of aforementioned reasons, you can't afford to run the real application. You don't want to use a synthetic benchmark such as _iozone_ or _bonnie++_ (while perfect tools) because they aren't really representative for your workload.

Instead, you can record all IO operations that your application does. Then, on a target test system, you replay them producing the same IO workload as the original application.

# Features #
  * Zero dependencies
  * Ability to replay complicated jobs that consists of **children processes** spawned by the traced parent process.
  * 20 system calls replicated, see [list](SystemCallList.md).
  * intelligent checking of environment prior to replaying the traces
  * possibility to define mapping of original file names to new one
  * possibility to define files that should be ignored (i.e. accesses to these files will not be replayed)
  * multiple replay time [modes](#Time_modes.md).
  * precise timing of calls delivery based on instruction count register
  * conversion of recorded traces to more suitable binary format
  * possibility to scale think-times (delays between individual calls)

# Usage #
> ## Record traces ##
    * trace the application of interest using this command
```
strace -q -a1 -s0 -f -tttT -oOUT_FILE -e trace=file,desc,process,socket APPLICATION ARGUMENTS
```
    * write down the time overhead the application had under strace to compare with a normal run
    * _optionally_ [convert](IOReplayConvert.md) the OUT\_FILE to a binary form using [ioreplay](ioreplay.md)
> ## Prepare environment ##
Once you have the recorded traces, you must prepare environment in such a way that it contains the files of at least the same size. In other words, all calls that previously succeeded/failed should succeed/fail again.

> ### Define map file ###
_Optional_. You can use **mapfile** that define mapping between original file name and a new file name. All calls that were performed on the original file name will now be performed on the new name. It applies for both, reads and writes.
Format of a mapfile is simple. It assumes that there aren't any spaces in file names (to be improved). Suppose you want to test Lustre filesystem and you obtained traces on GPFS:
```
#all lines starting with hash are ignored. Format is simple:
#original_name new_name
/gpfs/DATAFILE1.data /lustre/data/DATAFILE1.data
/gpfs/DATAFILE2.data /lustre/data/DATAFILE2.data
```

> ### Define ignore file ###
_Optional_. You can use **ignore file** that define list of files that will not be accessed at all. They will be simply ignored. It applies for both, reads and writes.
Suppose we only want to test replay accesses to data files, not to the libraries etc.
```
#all lines starting with hash are ignored. Format is simple:
#original_name
/lib64/liba.so
/lib64/libb.so
#and so on...
```

> ### Check the environment ###
`ioreplay` has a possibility to check whether replaying will succeed without any error or not. That is, whether all succeeded/failed calls will succeed/fail now. It can take into account the mapfile and the ignore file.

It reports every problem that can be:
  * a missing file. In this case a minimum file size in bytes is reported as well.
  * insufficient file size
  * existing file that should be deleted (i.e. previous open() call failed)

Every problem is reported only once, i.e. if a file is missing and some data are read from the file, it only reports failed `open()` call and not the failed `read()` calls, as they wouldn't appear if the file existed. In fact, the `ioreplay` constructs its own in-memory image of the file system based on the actual file system and the missing files/directories during the check process. This way, only roots of problems are reported.

Although quite intelligent, the check isn't perfect. It doesn't take into account any permission problems that could arise. In fact, there aren't enough information for reliable checking of such problems in the recorded trace file.

run it as:

```
ioreplay -C -f strace.output -m mapfile -i ignorefile
```
or, in case of the binary format
```
ioreplay -C -f strace.bin -F bin -m mapfile -i ignorefile
```

You can omit -m and -i options.

> ## Replay ##
Once you are satisfied with the check result, you can replay the traces:
```
ioreplay -r -f strace.output -m mapfile -i ignorefile -t diff
```
or
```
ioreplay -r -f strace.bin -F bin -m mapfile -i ignorefile -t diff
```

See time [modes](#Time_modes.md) for multiple options for time handling during replaying.
See man page for all options available.

> ## Verify scaling ##
In order to trust that the replaying is representative, you should verify that it scales similarly to the original application. To do that, you should measure time spent by original application and by replaying. If you have a chance to run both on a different hardware, do it, and measure the differences prior using it as a trusted benchmark.

# Installation #
## Dependencies ##
Install `strace` (if you don't have it already) for recording of traces.
## Installation ##
### From source ###
Grab the source tarball from Downloads section or from svn and then:

To install just the IOreplay and not the whole IOapps suite (see [this](IOAppsInstallation.md) for details), run following commands:
```
make replay
```
and then
```
make install_replay
```

This will install ioreplay binary and manual page to your system.
### From RPM package ###
TO BE DONE

This will install ioreplay binary and manual page to your system.

# Time Modes #
`ioreplay` provides three time modes in which IO calls can be replayed.
  * **diff** - default mode. Keeps delays between individual IO calls same as in the original run. This should be the most representative time mode.
  * **asap** - in this mode, the IO calls are made as fast as possible. It can be a option if you have a long running job and you don't mind sacrifing some precision. Beware that obtained results can differ A LOT when you run multiple instances of the same job.
  * **exact** - Try to make calls in same time relatively from start of the program.

In every case, sleeping is implemented as active waiting with periodical checking of instruction count register.

Lengths of the delays between calls (think times) can be scaled using -s switch.

# Architecture #
IOreplay is written completely in C.

# How precise can the replaying be? #
Well, it depends. It **can** behave almost as the original application but it can also be quite off. The only way how to find it out is to actually try it with your application. Main aspects affecting the precision:
  * Recording of traces. In some cases, the recording itself can skew timing by the factor of 100%. See [strace limitations](StraceLimitation.md). In that case, you can play with -s option to get quite reasonable behavior.
  * Calls delivery - given the multitasking nature of Linux systems, it is not possible to guarantee issuing of IO calls in the exactly same time as in the previous run.
  * Simultaneous IO calls. They are serialized, as the `ioreplay` is single-threaded. If your application does majority of its IO calls in the same time, `ioreplay` is probably not an option for you. Please let me know the name of the application in such a case.

Please contact the author if you can not achieve accuracy within 10% of the original application.

## Example ##
The scaling of `ioreplay` was tested using an [ATLAS ](http://atlas.web.cern.ch/Atlas/) analysis application. The job processed physicist data in order to find interesting events in them.
The application accessed 89 files, from which 9 were data files. Each data file had approximately 1GB, and around 250MB were read from each file, mostly sequentially but still with gaps. In total, the job read 2160MB from hard drive, spending totally 42 seconds in 7332 read requests.
The overhead of strace during recording was within 2%.
Used hardware: 2x Intel E5520@2.27GHz = eight cores in total, 16GB RAM, 300GB SAS 15k RPM hdd (Seagate ST3300656SS).
We tried to run up to twelve instances of the original job and `ioreplay` (each one accessing its own copy of data files) on the node, measuring the difference. The `ioreplay` was run on `diff` time [mode](#Time_mode.md).

![http://elvys.farm.particle.cz/scaling1.png](http://elvys.farm.particle.cz/scaling1.png)

As can be seen on the plot, the `ioreplay` performed very well (within 4%) until we overloaded the node with more than eight jobs (remember, the node had only 8 cores). After that, the original application start to run slower than `ioreplay`. However, this was well expected as the `ioreplay` was run in `diff` time [mode](#Time_mode.md), so it still kept the delays as if the node wasn't overloaded. On the other hand, the original application did the actual computing so it started to run slower as there were no free CPU cycles.

# Drawbacks #
  * Single threaded