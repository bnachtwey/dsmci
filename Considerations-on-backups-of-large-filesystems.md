# Parallelizing Backups of large filesystems using IBM TSM / SP
<!--
(C) 2016 original Text in German by Bjørn Nachtwey @ GWDG
(C) 2019 translated to englich Text by Bjørn Nachtwey @ GWDG
(C) 2024 Updates by Bjørn Nachtwey privately, hosted on [GitHub]()
-->
🚧 🚧 UNDER CONSTRUCTION :-) 🚧  🚧 

The you can summarize the root cause for long backup times of large filesystems often in one single statement:<br><br>
***Identifiying changed files takes too much time!***

The following text gives an analysis and some approaches to solve it. It was originally published in the [GWDG Nachrichten 11/2016](https://gwdg.de/about-us/gwdg-news/2016/GN_11-2016_www.pdf), a second edition followed in [GWDG Nachrichten 1-2/2019](https://www.gwdg.de/documents/20182/27257/GN_1-2-2019_www.pdf) as an english translation with some addons.

*This Text is now rolling a forward edition, updated whenever there's something to add :-)*
## Table of Content
* [Motivation](#Motivation)
* [Current Situation](#CurrentSituation)
* [Non-working approaches](#NonWorkingApproaches)
* [Acceleration with SP on-board tools](#SPOnboardTools)
   * [Simplified identification exclusively via the change date](#SimplifiedIdentification)
   * [Turning off ACLn and checksums](#TurningOffACL)
   * [Parallelization of backups for multiple file spaces](#MultipleFilespaces)
   * [Explicit backup of changed files only](#ChangedFilesOnly)
   * [JournalBasedBackup / FilepathDemon](#JBBFpD)
* [Hybrid approach with snapshots](#HybridSnapshots)
* [*Excursus:* File systems that support fast backup](#FSfastBackup)
   * [IBM Storage Scale (ISS, formerly GPFS)](#FSGPFS)
   * [NetApp SnapDiff](#FSSnapDiff)
   * [Quantum StorNext](#FSStorNext)
   * [General remarks](#FSGeneralRemarks)
* [Two (simple) ideas for all file systems](#Simple4All)
   * [Variant 1](#Simple4Allv1)
   * [Variant 2](#Simple4Allv2)
* [One approach for all file systems](#OneApproach4All)
* [Addon: functions to be added](#AAditionalFunctions)
   * [Profiling based reording of folderlist](#ProfilingReorder)
   * [Empty folders](#EmptyFolders)
   * [Continuous backups](#ContinuousBackup)
   * [Dynamic deep dive](#DynamicDeepDive)

## Motivation <a name="Motivation"></a>
As the topic ["Looking for suggestions to deal with large backups not completing in 24-hours"](https://www.mail-archive.com/adsm-l@vm.marist.edu/msg102161.html) was discussed on the *„ADSM-L“* mailing list recently I decided to update and translate a text published in the [GWDG News](https://gwdg.de/about-us/gwdg-news/) in November 2016 by November 2018. 
The growth of data mentioned in the first edition was still ongoing and therefore the file admins and backup operators have to face the challenge of coping with the backup of data within the specified time window and thus complying with the promised protection against manipulation and data loss. In the article some approaches to speed up the backup using SP/TSM are discussed. 
They will be explained briefly, but the scope is on the chances and limitations of each. The second part of the article develops an approach starting with the basic idea of parallelizing the backup towards different variants of a script including some reporting, error handling and statistics.

## Current situation <a name="CurrentSituation"></a>
The Amount of data is growing, different analysis show a value of 20% each year [1,2]. In addition to the challenges of storing this data sensibly and efficiently, one aspect often falls out of focus:

**How can this growing data be backed up?**

The nominal performance of the tape systems is growing faster than the growth of the data itself [1,2], but even this view is unfortunately incomplete, since the process of backing up, the actual backup, often represents the bottleneck. *“IBM Storage Protect (SP)”* (formerly known as „WDSF/VM“,“DFDSM“, „ADSM“, “Tivoli Storage Manager (TSM)”, "IBM Spectrum Protect (ISP)") 
has long pursued the approach of backing up only the changed files s since the last run only – instead of doing a full dump sometimes. As there are no planned full dumps, this approach is called *“incremental forever”*.

*The Advantage is obvious:*<br>
Especially for large file systems (say > 10 TB) the amount of data changed daily is relatively small. So that even with many versions of older data (the GWDG standard backup policy allows up to 350 versions in 90 days) the additionally needed space is small [2b]
compared to the space needed for the secured data. But, if a full backup is done periodically, the backup capacity must be several times greater than the secured data itself. For mixed approaches, e.g. the “Grandfather-Father-Son”-principle by means of

* monthly full backups (*Grandfather*)
* weekly differential hedges (*Father*)
* daily backups (*Son*)

the necessary backup capacity is larger than with *“incremental forever”* due to the full dumps. However, *“incremental forever”* does not solve an essential problem of any incremental backup namely the answer to the question of which data must be backed up at all. 
SP identifies the data to be backed up (*“backup candidates”*) by comparing all directories and files on the computer (to be backed up) with those from the last backup and remembering changed files. This process usually runs typically at a speed of 1 – 2 million objects (files and folders) per hour.

Searching through a 100 TB file system with around 100 million objects therefore lasts between 50 and 100 hours, **a daily backup of such a file system is not possible with the common approaches**.

In addition, there is the problem that within this long search time, a considerable amount of data will be changed or even deleted. As a result, ISP throws many error messages like<br>
`ANS4037E Object <NAME>' changed during processing. Object skipped`<br>
or <br>
`“ANS4005E Error processing '<NAME>': file not found`.

**How to solve this problem?**

## Non-working approaches <a name="NonWorkingApproaches"></a>
One possible solution might be the *“bird ostrich method”*:<br>
Just adapt the service description of the file servers by not guaranteeing daily backups but (initially) only every two days. As the amount of data grows, the backup frequency needs to be continuously adjusted. When reaching about 150 million objects the interval will only be a monthly backup :-(

At this point, the second *“zero solution”* should be considered:<br>
To give up the backup of corresponding file systems completely and not to lull the users into a (data) security that does not (no longer) exist.

Since searching for *“backup candidates”* is the problem with backups, one could consider biting the bullet and doing full backups, as tapes are relatively inexpensive compared to DISK storage. Unfortunately, in my experience, this is definitely no solution:<br>
For a full backup of 100 TB, theoretically, only about 24 hours are required with a 10GE connection. However, operational experience shows that only about 2 – 4 TB per day are effectively saved and about 25 – 50 days are required for each full backup. In other words, approximately the same time as for an *“incremental”* backup.
Using faster Ethernet Connections such as 25GE or even 100GE decrease the time, but in the end the backup still lasts days and is definitly not done daily or even weekly.

## Acceleration with SP on-board tools<a name="SPOnboardTools"></a>
IBM offers several on-board tools to speed up the backup process:

### Simplified identification exclusively via the change date<a name="SimplifiedIdentification"></a>
Usually the ISP client compares numerous meta data to select objects for the new backup. Besides the date of the last modification, these are also file size, checksum, access rights / ACLn. 
In the interactive call dsmc i and/or as Object in the client schedule, the check can be reduced to the comparison of the change date of the object with the date of the last backup by the option -INCRbydate and thus considerably accelerated.

However, the option also has some problems:<br>
Especially if no snapshots are used or if the backup fails, files that are modified or created while the backup is running will be skipped during the next run with [`-INCRbydate`](https://www.ibm.com/docs/en/storage-protect/8.1.24?topic=reference-incrbydate) if they have not been modified again. 
IBM therefore strongly recommends running a normal *“incremental”* regularly (see link mentioned before). Similar problems can occur if client and server have different system times.

**Another important point:**<br>
Deleted files are not recognized, they remain in the backup and files that come into the system with an old date, e.g. due to the installation of software, are not backed up!

In summary, the `-INCRbydate` option can only be used for the daily backups together with a normal backup at the weekend if the normal backup lasts slightly longer than 24 hours.

### Turning off ACLn and checksums<a name="TurningOffACL"></a>
Processing ACLn (and thus previously checking) and creating checksums slows down the identification process and can be influenced by several options. However, it should be carefully considered whether the relatively low speed gain sufficiently outweighs the loss of information.

* `SKIPACL Yes` completely disables ACL processing, but ACLs will probably not be saved either (option for Unix + MacOS only)
* `SKIPACLUPdatecheck Yes` also disables checksum calculation (option for Unix + MacOS and Windows)
* `SKIPNTPermissions Yes`  bypasses processing of Windows file system security information.
* `SKIPNTSecuritycrc Yes` prevents the calculation of a CRC checksum (option for Windows only)

### Parallelization of backups for multiple file spaces<a name="MultipleFilespaces"></a>
If the data to be backed up are on several partitions, the backup process can be distributed to parallel streams using the `RESSOURCEUTILIZATION` option (in contrast to IBM documentation, significantly more than 10 are possible, > 100 streams are reported in practice). 
This makes better use of the bandwidth and considerably reduces the search time through parallelization. Since this also generates additional sessions on the SP server side, the number of `MAXSESSIONS` may have to be increased, too. 
This approach works only when actually backing up multiple file spaces. As a workaround, of course, a single file space can be split into seemingly multiple file spaces with the VIRTUALMOUNTpoint option and then this approach works, of course. (See also Excursus on *virtual Mount points for Windows Clients*)

### Explicit backup of changed files only <a name=""></a>
If information is available, which files have changed since the last backup and which files have been deleted since then, ISP can only back up these files. Instead of an “incremental backup”, a “selective backup” with the explicit specification of these files is then possible:
```
dsmc sel –filelist=<File with Names of files changed>
```
or
```
dsmc expire –filelist=<File with Names of files deleted>
```
The basic principle of *“selective backup”* is also used in the following approach and in the “file systems that support fast backup”, but requires two explicit lists of files that have been modified or deleted.

### JournalBasedBackup / Filepath Demon <a name="JBBFpD"></a>
IBM has been offering the *JournalBasedBackup (JBB)* method since TSM 5.

The *JBB Demon* (or *Filepath Demon*) monitors the file system to be backed up and collects information on new, modified and deleted files. During backup, the TSM/ISP client uses this information in the same way as doing a selective backup. The effort for identifying the backup candidates is eliminated and the backup is reduced to the transfer of the new / changed data.

Tests done by the GWDG with a Linux fileserver with about 150 TB capacity distributed over 22 file spaces were not successful: The resource requirements for the JBB were extensive, but the time saving, especially due to regular re-indexing, was rather limited. In other constellations, the JBB may bring clear advantages.

There is also an important limitation:
Journal Based Backup only works with local file systems. CIFS / NFS and cluster file systems do not work.

*Hint:*<br>
Optimizations for data transmission can be found in the Performance Tuning-Guide (V7.1.6).

## Hybrid approach with snapshots <a name="HybridSnapshots"></a>
Numerous file systems and most filers offer the possibility to create snapshots. A hybrid approach can be implemented by combining snapshots and ISP backup:

Backups are done as often as possible, e.g. weekly, in between snapshots.

In addition to the considerable expansion of the backup time window, there is usually the positive side effect that the end users can access the snapshots directly and the admins are relieved of numerous restore requests. If the backup is also based on a snapshot, the problem of the opened files is also solved (error message ANE4987E Error processing '<NAME>': the object is in use by another process).

A prerequisite for this approach is, of course, that the file systems support snapshots – and in sufficient quantities.

## Excurs: File systems that support fast backup <a name="FSfastBackup"></a>
Some file systems / filers support a fast backup using ISP by identifying the necessary backup candidates and making them available to the ISP client. (This list is only a selection):

### IBM Spectrum Scale (ISS, formerly GPFS)<a name="FSGPFS"></a>
IBM's cluster file system naturally supports backup with ISP and even offers its own script mmbackup. This not only uses the information about the backup candidates, but can also parallelize the data transfer over several (ISP) nodes and GPFS servers.
However mmbackup does not simply run out-of-the-box: The initial creation of the configuration requires a little trial and error, but afterwards mmbackup runs both stable and performant.
In addition to ISP, IBM Spectrum Scale also offers close integration with HPSS as an HSM system, so that the problem can also be reduced by (partially) transferring the data to HPSS - whereby ISP/ISS can also back up very large data volumes in a comparatively short time.

### NetApp with SnapDiff<a name="FSSnapDiff"></a>
NetApp has also been supporting backup of its own NAS filers since TSM 5 in a variety of ways. In addition to NDMP, the SnapDiff function also accelerates the incremental backup. SnapDiff transfers the changes to files and directories between two snapshots to the ISP client. The integration goes so far that the ISP client can even trigger the required snapshots on the filer and after a successful backup can delete the previous one on its own.

Since the SnapDiff function compares only two snapshots, but does not take into account in any way whether the last backup was successful, the same problems arise as when using the -INCRbydate option: errors from the last backup are not compensated and a regular normal incremental backup is strongly recommended. mmbackup, in contrast, takes into account the backup status of all data and is fault-tolerant with regard to the problems mentioned above.

Basically, each cluster/scale-out file system should be able to provide a list of new, modified and deleted files, since this (meta) information is necessary for the consistency of the data (and especially the caches) on the cluster nodes. In practice, the problems are that this information is not easily accessible and there are no tools by manufacturers to access this data. Quantum has responded to customer demand and is currently examining how this information can be made available to the StorNext file system. DELL/EMC also offers ScaleOut NAS systems with the ISILON systems. In version 7 of the operating system, called OneFS, there is the possibility to log changed files, but the resource requirements are so high that there is a lasting impairment of the entire system. With OneFS 8 there should also be improvements here.

### Quantum StorNext<a name="FSStorNext"></a>
:construction: t.b.d. :construction:

### General Remarks<a name="FSGeneralRemarks"></a>
:construction: t.b.d. :construction:

All Filesystems mentioned above are somehow *distributed and scale out file systems*. They all share one basic property: Due to the distribution of the data and the distributed access using different access points (e.g. *Gateway Server / Nodes*) there must be an internal process that knows about all changes on the stored files, so that other nodes than the one holding the primary copy get notified on any changes and then update their cached copy. This knowledge of file changes allow to create file lists for new, changed and deleted files, that TSM/SP can then process as mentioned above. So any filesystem having such knowledge (e.g. also *CephFS*) should be able to provide *changed files lists*.

## Two (simple) ideas for all file systems <a name="Simple4All"></a>
For all users who do not have an IBM Spectrum Scale in operation (mmbackup is the best solution for this!) and neither full backups nor NDMP this raises the question of what to do now?

As previously mentioned, identifying backup candidates takes most of the time during ISP backup. This process examines the entire file tree of the file system to be backed up – sequentially in a single thread. The solution is to turn this one process into several parallel processes.

Users can usually be divided into groups (e.g. working groups or institutes). Especially in academic environments, this classification can also be found in file systems, since there is often a folder level with faculties or institutes for easier access control, and below this level are the user and workgroup directories.

### Variant 1 <a name="Simple4Allv1"></a>
Parallel backup is possible by setting up a separate node for each faculty or institute instead of a single ISP node for the entire file system and performing the backup “faculty by faculty” / “institute by institute”. Instead of a single process, several processes search the file system in parallel (file servers are able to process even several hundred parallel processes) and the search times should be significantly reduced. In practice, this approach reveals at least two problems::

Not all faculties / institutes have the same amount of data; usually there are one or two that use almost the entire capacity of the file system alone. Therefore, the backups of some run much faster than of others, for the “big users” the backup time is only slightly reduced in the worst case. Overall, the (time) gain is usually only marginal.
If further faculties (probably not so often) or institutes are added, the backup administrator must adapt his configuration in time, otherwise the new ones are left out.
Using Unix, the nodes can be separated relatively elegantly using “VIRTUALMOUNTs”, for Windows you either have to create exclude.dir rules for each node, which is both complex and error-prone, or work with a trick (see excursus “VIRTUALMOUNTPOINTS for Windows”).

### Variant 2 <a name="Simple4Allv1"></a>
Often, however, the users on the file systems are not organized in groups, but all directories lie flat next to each other on the entry level. Creating a separate ISP node for each user directory repeats the second problem mentioned above and is very time-consuming regarding the number of users.
It is therefore easier to distinguish the directories according to a pattern, for example after the first character(s): : ^[a,A], ^[b,B], … ,^[z,Z], ^[0-9] (ISP even provides regular expressions at this point!)
You get 27 or 729 ISP nodes, which automatically include all new directories. Unfortunately, the Regular Expressions (RegEx) formulations only capture the directories that exist, not the deleted ones. Remedy is possible if you additionally back up all directories of the start path without subdirectories.
Although this variant is often better than the first, it does not meet all expectations:

Solving the deleted directories problem is cumbersome.
The configuration becomes - especially if one distinguishes between the first two letters - very extensive.
Changes to the directory names distribute the data across several ISP nodes and the restore in particular becomes time-consuming.
In summary, there are certainly application scenarios for both approaches, but experience at GWDG shows that the effort is quite high and there are always a few power users who again need special treatment with these two approaches in order to achieve a usable benefit.

## One approach for all file systems <a name="OneApproach4All"></a>
### Idea and first steps using BASH
Already in the last decade the (at that time) *Generali Versicherungs-AG* was facing with the problem outlined at the beginning and *Rudolf Wüst* as backup admin extended the aforementioned approach by a decisive idea. 
From this, he developed a solution that successfully parallelized the *“search problem”* with up to 2000 threads. Mr. Wüst kindly shared his extension and the author took it up and developed it further within the scope of his work at the GWDG.
The goal of a practicable solution must be to capture all directories, store them in an ISP node and still parallelize the search. This can be done by executing a script instead of a simple backup call, which in turn starts several parallel threads to back up the directories. The core of the script consists of a loop of the following form (example 1).
```
For each (find all directories in given start path)
{
	run a backup for this specific directory
}
```
*Example 1: pseudo code*

Instead of *“one incremental backup”*, many partial incremental backups are performed for each directory.

The deleted directories are recorded with a subsequent backup of the start path without subdirectories –- the last specification is extremely important, otherwise, a normal *“incremental backup”* is made on the entire file system.

As source code for the BASH this looks like in example 2:
```bash
startpath=<path to start with>;
folderlist=<path to a file containing foldernames>;
find $startpath –xdev –mindepth 1 –maxdepth 1 –type d –print > $folderlist
for $i in $(cat $folderlist)
do
	dsmc –i $i –subdir=yes &
done
dsmc –i $startpath –subdir=no
rm $folderlist;
```
*Example 2: source code BASH*

During the first tests you will find out that the script in its present form will indeed start as many threads as existing directories. On the one hand, this forces the computer that performs the backup to its knees, and on the other hand, the “MaxSessions” setting of the ISP server is probably reached almost immediately and the server refuses further connections.
The remedy is a counter that simply waits when the allowed number of threads is reached. In the bash, the split backup threads have the “parent process ID” of the script itself, so these threads can be counted even if you run the script for several file systems simultaneously.

As BASH code the loop looks like example 3.

```bash
pid=$$;	# parents process id 
startpath=<path to start with>;
folderlist=<path to a file containing foldernames>;
maxthreads=<max. number of parallel threads>;
find $startpath –xdev –mindepth 1 –maxdepth 1 –type d –print > $folderlist

while [ -s $folderlist ]
do
	nthreads=$(ps axo ppid,cmd | grep $ppid | grep -v grep | wc -l)
	if [ $nthreads -le $maxthreads ]
	then
		# get new start path
		folder=$(head -n 1 < $folderlist);

		# backup actual folder
		dsmc i $folder/ -subdir=yes -quiet >> $ppid.log &

		# remove first line from folderlist
		sed -i '1 d' $folderlist
	else
		sleep 5; # wait to complete another thread
	fi;
done
dsmc –i $startpath –subdir=no

# wait for all running threads at the end
while [ $nthreads -gt 1 ]
do
	>&2 echo "Waiting for $nthreads threads to end"
	sleep 60;
	nthreads=$(ps axo ppid,cmd | grep $ppid | grep -v grep | wc -l)
done
rm $folderlist;
``` 
*Example 3: extended BASH code*

In the extended form, essential goals are now achieved, but one cannot be completely satisfied:

A return value is missing for reporting for the ISP server. In the simplest case, you can add a line return 0 at the end, then the schedule is always successful - regardless of whether errors occur or not. As already added in example 3, one should rather collect the output of the individual “partial incremental backups” and evaluate them at the end of the script, e.g. search for errors or summarize the “summaries”. Depending on the type (and number if necessary) of errors and the “Files failed”, the script can then give the appropriate return values. (This extension is already included in the published source code.)
The problem mentioned with the idea that individual directories use a considerable proportion of the file system size and thus significantly influence the runtime of the backup is not solved by the script. The inequality of the data set / number of objects will certainly be smaller, but will only shift.
The next step would be to create the directory list over several levels and thus increase the number of partial backups. As a result, inequality should be more evenly balanced out.
In addition to the return code of a “client schedule”, detailed error messages and an overview in the form of a summary can also be read out in the reporting of the ISP server; this is (currently) not possible with the specified script; it only provides a traffic-light status via the return code.

### An excursion to the PowerShell
PowerShell. Unix affinity combined with reservations about the Powershell and above all the double effort ended this project after some work without having created an executable version.

### PERL - one solution for all (?) worlds and further development of the simple approach
The closest solution was initially overlooked: a programming/scripting language for all operating systems, neither BASH/MinGW/WSL nor PowerShell / PowerShell Core on Linux, but PERL.

PERL offers numerous functions - also in the area of access to files and directories, which are encapsulated by the respective implementation in such a way that the actual command is independent of the operating system. File system paths can even be specified in both Unix and Windows nomenclature (i.e. with / or \ as directory separator) and thanks to the File::Spec→canonpath function they are converted to the correct format. To a large extend the source code does not need be individually adapted for the respective platform. Exceptions are the paths to the binaries, i.e. \opt\tivoli\client\ba\bin\dsmc or C:\Program Files\Tivoli\baclient\dsmc.exe and (currently) only partial readout of the directory tree using find (Linux) and Robocopy.exe (Windows).
Another reason for PERL is that it allows the use of threads in a simple way and also ensures that only a certain number of (sub) threads run at the same time and thus the start of further threads only takes place after completion of previous threads - and this independent of the operating system!

The steps outlined for the BASH are thus reduced to three essential steps in PERL:

create a new subthread with the fork() function
branch the source code into the paths main script and script for the subthread,
in the main script only the number of started threads is incremented, in the subthread the partial incremental backup takes place
check whether the desired number of threads has been reached and waiting for a thread to be terminated and then start a new one.
In detail, the source code is of course somewhat more complex and also takes into account, for example, if that starting a subthread was not successful.

#### Further development: Deeper dive into the directory tree and start parallel threads based on multiple directory levels
The tests with the parallelization approach directly below the base path showed exactly those effects that were already addressed during parallelization via institutes: Individual directories are (usually) larger than all other parallel-lying directories together, 
so that the speed gain is considerably lower than expected / or desired. A better balance can only be achieved via additional directories; these can be found by searching through further levels in addition to the first, highest directory level below the start path and then allowing the backup to be made via all these directories.
The first problem is that the directories are nested, i.e. a partial backup of a directory from a higher level also includes those subdirectories that are backed up in other parallel threads anyway. In this script, this problem was solved by saving all directories above the set *“dive depth”* with the option `-SUbdir=No`, 
i.e. only the contents of these directories including the names of the subdirectories, but not their contents. In a second step, the directories are backed up at the lowest level specified with their subdirectories (-SUbdir=Yes option) (Since backups without subdirectories are usually much faster, 
those directories with subdirectories are backed up first and those without are backed up second).

#### Evaluation of the individual runs
Not only for profiling (see below) but also to create a summary of the backup, each sub-thread writes its output to a separate file that contains its own ID in the name in addition to the process ID of the script. Thus, even if the script is aborted, the output can be clearly assigned.

Although the overall evaluation can only take place at the end of the backup, a sufficiently deep dive into the directory tree in practice quickly leads to several thousand to hundreds of thousands of small files and thus to considerable problems. Therefore, the sub threads write the content of the output file to a central log file after completion of their backup, which is evaluated at the end of the script. Within the context of this writing, the information whether subdirectories have been processed is also stored and the runtime is already converted into seconds for profiling and saved. The return value of the backup call is also added.

In the current implementation (December 2018), the final evaluation sums up In the current implementation (July 2018), the final evaluation sums up

* the number of objects inspected
* the number of objects saved
* the number of updated objects
* the number of deleted objects
* the number of orphaned objects
* the number of objects with errors
* the number of inspected bytes
* the number of bytes transferred
* the time for data transfer
* the elapsed time
* If necessary, a conversion to the common size (bytes, seconds) takes place.

In addition, the number of

* of warnings (/^AN[RS][0-9]{4}E/)
* the serious error (/^AN[RS][0-9]{4}S/)
* the error due to ANS1228E and ANS1820E
* the Server-Out-of-Space-error (ANS1292S) additionally
are counted in each case and as a total.

From the sum of the elapsed times and the runtime of the loop via the directories (WALL CLOCK TIME) the script calculates a parallel speedup, which shows how much faster the parallelization is compared to the sum of the individual times.

#### Performance optimization using profiling
The runtime of the parallel backup is essentially determined by the runtimes of the individual backup runs. Without a detailed measurement (but by comparing the time for the script call with commented out the backup call) it is assumed that the runtime of the PERL statements is negligible in comparison. The aim of the optimization is the “correct” order of the directories, so that

the large, long-running ones run as parallel as possible,
the large ones are started because a mismatch balance has a less dramatic effect on the shorter runtimes of the smaller directories.
Since the runtime of the backups cannot be estimated in advance, the optimization is based on the last backup (and does assume no dramatic changes, which could only be predicted by complex and therefore time-consuming analyses). As described above, the sub threads also write the runtime in seconds to the central log file at the end of the backup, so that a list of all directories with the respective runtimes is created when they are evaluated. This list is sorted by descending runtimes and written to a profile file.

The next time the script is called, it first creates a list of all directories to be backed up. In the next step (this part does not yet work for Windows and has therefore been swapped out again) the backup script compares this list with the entries from the profiling file.

Directories that are in the profiling file but no longer exist in the directory list are ignored. New entries in the directory list for which there is no runtime in the profiling file are assigned a long runtime (10^10 seconds ^= > 316 years) and are therefore ranked first. If there is no profiling file, the directories are processed in the order they appear in the list from the directory tree.

At the end of the evaluation, the profiling list is overwritten.

### Open Issues / Outlook
There are still some questions left, for example about transferring the summary to the server log.

For a good solution, error handling should be added to make the script fault-tolerant to certain situations.

It is also possible to split the work steps “Identify directories” and “partial incremental backup”, so that for very large file systems, the list of directories to be processed is filled up again as soon as the backup window has expired, but still runs one or a few threads - but probably increasing the immersion depth is the better approach.

One problem that cannot be solved is the fact that “partial incremental backups” do not change the “Last Backup” attributes of the nodes or file spaces and, of course, this is not done within the scope of the outlined script. You should refrain from writing to the DB2 of the ISP servers, as this affects IBM's warranty. IBM expressly prohibits direct access to the ISP-DB2 outside of corresponding instructions within the scope of support.

## In addition, how do you speed up the restore?
The previously mentioned approaches with ISP on-board means and the outlined approach for parallelization only works for backup. If many files are to be restored from the backup, this is very easy with the approaches with several nodes for a file space, since a separate restore must run for each node anyway and the processes run in parallel. For the parallel threads approach, an adjustment for the restore based on a file list is easily possible: Instead of a “folder list”, a file list is used for the restore.

However, it should be noted that in an environment with a tape library as a storage backend, the number of drives usually limits the performance of the restore. Furthermore, ISP usually organizes the restore (without the -disablenqr=yes option) so that the tape mounts are optimized. If a file list is processed in parallel by numerous parallel threads, the server cannot optimize the tape accesses. However, if a disk-based FILE or container pool is used, the parallel restore over numerous threads is faster. If the data is stored on two servers via server replication, the restore can also be distributed over both servers and thus additionally accelerated.

Unfortunately, experience shows that “full restores” also involve enormous effort when parallelizing and can only be accelerated unsatisfactorily.

## Availability / Access to source code / Alternatives
It can be assumed that neither Rudi Wüst only has invented the original idea nor I can claim to be the only one to have had and implemented the idea outlined. Rather, many TSM/SP users may have faced the same problem and found similar solutions.

A commercial implementation that follows a similar approach to parallelization can be found in the product [*„MAGS“*](https://www.general-storage.com/mags.html) of General Storage. In addition to binding support, *“MAGS”* offers regular further development and uses several NAS nodes for parallelization with ISILON Scale Out systems. 
A more detailed product analysis should not take place here. You must also determine the individual benefit.

The script mentioned in this article is freely available in my GITHUB Repository the *Apache 2.0 license*. The scripts may be used and modified without restrictions. I look forward to receiving your feedback and suggestions.

## Transferability to other backup solutions
The approaches presented address the problem of file identification and can therefore be applied to all other questions where a file list is to be created. If you replace the call of the SP-CLI with another CLI call, you can also find all files in parallel, filtered by all attributes supported by find using appropriate parameters. 
You can also add another loop that does arbitrary operations with all entries of a complete file list. This also allows you to optimize other backup solutions that can process a directory or file list.

## Acknowledgement
I'd like to thank Gerd Becker ([Cristie Data GmbH](https://www.cristie.de)), Wolfgang Hitzler ([IBM](https://www.ibm.com), retired) and Manuel Panea ([MPCDF](https://www.mpcdf.mpg.de/)) for proofreading the original article and making suggestions for changes and improvements.

Special thanks to [Mr. Rudolf Wüst](https://de.linkedin.com/in/rudolf-w%C3%BCst-2258a4122) for his generosity in sharing his ideas.

# Excursus
## Workaround for VIRTUALMOUNTPOINTS for Windows clients
For UNIX, Linux, and MacOS it is possible to configure individual directories as virtual drives in TSM/SP. This simplifies the configuration of the backup, since the virtual drive can be specified directly as backup source instead of specifying the actual drive and excluding all directories that are not to be backed up using exclude rules.

Unfortunately, there is no comparable function for Windows. This also eliminates the possibility of parallelizing the backup via different virtual drives.

However, if only individual directories are to be backed up, but not the remaining root directory in parallel, numerous exclusion rules must usually be created in the form of `exclude.dir´` statements in the `dsm.opt`. 
Of course, this way is highly error-prone, additionally all directories newly created in the root directory of a drive are not automatically excluded, but are included in the backup. The following workaround simplifies configuration and parallelization of the backup under Windows:

* create an advanced share for each directory you want to back up.
* by adding a `$` to the share name, the Windows SMB service also does not list it on the network map (*“hidden share”*).
* access to this share is only required by the local admins of the backup node
* the paths to be backed up can be accessed via the loopback device:
  * `DOMAIN \\127.0.0.1\<Share1>`
  * `DOMAIN \\127.0.0.1<Share2>`
	
**From the point of view of the TSM/SP client, the shares are independent network shares and can be backed up in parallel!**

## Threads in PERL <a name="ThreadsInPerl"></a>

PERL offers its own thread module and thus a much more elegant method than the complex solutions for the BASH or the Powershell:

Using the `fork()` function, the PERL interpreter creates a second thread that starts at just this point in the script. This thread processes all of the statements below in the same way as the original script. It therefore makes sense to use an IF statement to branch the different tasks. 
For this, the return value query of the `fork()` routine provides: if the value is not defined, no thread could be created, if the value is `true`, there is a new thread - and it is the parent routine in which this IF was executed. The value is also defined in the child thread, but `false`. The query could therefore look as follows:

```
my $cpid = fork();
if (! defined $cpid)
{	# forking failed!
	   exit 1; # abort the script
}
if ($cpid)
{	# parent process
	# … further commands
}
else
{	# child process
	# … further commands
}
```

Collecting / waiting for the started threads is also much easier: The `wait()` function waits for a child thread to end, so they can all be waited for with a simple loop:

```
while (wait() != -1 ) ;
```
---
# Addon: functions to be added <a name="AdditionalFunctions"></a>
## Profiling based reording of folderlist <a name="ProfilingReorder"></a>
 🚧 
### basic idea
1. with each run of the backup tool it collects the wallclock time for each folder
2. for the next run the folders are ordered by the runtime starting with the longest values
3. new folder never backed up before get an incredible high value, e.g. 1 trillion seconds (10E12 seconds are equal to about 32 years), and are processed therefore at the beginning

## Empty folders<a name="EmptyFolders"></a>
* As the parallel treewalk only identifies folders at rest, the parallel tool won't identify *"vanished folder"* that were directly processed (subfolders are tidied up by the dsmc client).
* Therefore the profiling information can also be used to identify such vanished folders and remove them from the backup with an explicit `dsmc expire -filelist=<file with paths of vanished folders>
* such run can either included in each run or separately scheduled using the profiling information

## continuous backups<a name="ContinuousBackup"></a>
* Even with all approaches mentioned above, some folders may limit the overall performance as they run so long, only few or even no more are available for parallel backup.
* at some time the parallel backup runs sequentially again as it proccesses only one last folder
* putting the folder list in a file and removing all folder already processed you can just fill up this file with a new folder tree walk adding those folders already processed after some time
* unfortunately this impacts the calculation of the "total wall clock time" and touches the integrity of the backup as some folders are processed more often than others (on the other hand many are processed once as long as the longest thread takes)

## dynamic deep dive<a name="DynamicDeepDive"></a>
* the actual implementation defines a level of depth for all folders
* as a workaround you can manually add subfolders as an additional startingpath and therefore do an non-even deep dive into the folder structure
* unfortunately this is not error prof by now. The code may not recognis subfolders if they are not identified in the foldertreewalk. Worst case they are processed doubled.
* A better approch would be to let the code dynamically decide where to deep dive for the next run. Definig "high water marks" for *dive in* and "low water marks" for *dive out* based on the runtimes derived from the profiling may improve the over-all performance?


---
## Footnotes
- [1](http://www.storageconference.us/2014/Presentations/Shimizu.pdf)
- [2](http://www.insic.org/news/2015%20roadmap/15pdfs/2015%20Technical%20Roadmap.pdf)
- [2b] After evaluation of the GWDG ISP servers in 2018: between 15% and 84% in addition to the active data, whereby the 84% is an outlier, the average value is 39%. 
