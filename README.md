# CS253 Final Project: myps
In this project you will write a simplified version of the ps command found on Linux/Unix based systems. The purpose of this command is to display the current processes on the system and some basic metadata including the process id number (PID) as well as the associated command (COMM/CMD). For your final project you will develop a simple program that loads information from the proc file system and displays it to the user with options provided to change the order that processes are displayed. For debugging/testing purposes an option will also be added to specify an alternate directory to load process data from.

## Learning objectives
    - Demonstrate knowledge of dynamic memory allocation
    - Demonstrate knowledge of Create/Destroy design pattern
    - Demonstrate knowledge of file stream processing
    - Demonstrate knowledge of file system navigation
    - Demonstrate knowledge of fundamental C language components: structs, arrays and pointers
    - Demonstrate good coding style by following provided Style Guide
    - Demonstrate good coding quality by producing code that has been well tested and is free of memory errors/warnings.
## Project Background
As we learned when we studied processes, the kernel is responsible for creating and managing processes within an operating system. On Linux, the kernel provides a window into its internal process structures in a virtual filesystem called /proc. This is mounted as a filesystem on Linux and can be navigated using the standard commandline tools. 

### Working with /proc file system
```
cd /proc
ls
```
In addition to details about device drivers, memory usage and various other metrics, the /proc directory contains numbered subdirectories that coorespond to the process id (PID) of currently running processes.  Within each directory is data for that particular process.  That this directory structure is highly volatile as the PID directories will appear and disappear as processes are created and terminated.

```
cd 1
ls
```
For this project we are particularly interested in the stat file located within each process. It contains a single line of space delimited values that provide a variety of metrics for the corresponding process.

```
cat stat
1 (systemd) S 0 1 1 0 -1 4194560 121396 20180319 149 12318 234 765 164412 91000 20 0 1 0 40 171683840 2025 18446744073709551615 1 1 0 0 0 0 671173123 4096 1260 0 0 0 17 1 0 0 32 0 0 0 0 0 0 0 0 0 0
```
### Man page: proc
The /proc man page has detailed information about the /proc file system which you can read about. For this project we will only be loading 6 data points about every process on a system.  

The man page on the /proc file system is huge so we have copied the section relevant to this lab here:
```
 /proc/[pid]/stat
              Status information about the process.  This is used by ps(1).
              It is defined in the kernel source file fs/proc/array.c.

              The fields, in order, with their proper scanf(3) format speci‐
              fiers, are listed below.  Whether or not certain of these
              fields display valid information is governed by a ptrace
              access mode PTRACE_MODE_READ_FSCREDS | PTRACE_MODE_NOAUDIT
              check (refer to ptrace(2)).  If the check denies access, then
              the field value is displayed as 0.  The affected fields are
              indicated with the marking [PT].

              (1) pid  %d
                        The process ID.

              (2) comm  %s
                        The filename of the executable, in parentheses.
                        This is visible whether or not the exe‐
                        cutable is swapped out.

              (3) state  %c
                        One of the following characters, indicating process
                        state:
                        R  Running
                        S  Sleeping in an interruptible wait
                        D  Waiting in uninterruptible disk sleep
                        Z  Zombie
                        T  Stopped (on a signal)
                        t  Tracing stop (Linux 2.6.33 onward)
                        W  Paging (only before Linux 2.6.0)
                        X  Dead (from Linux 2.6.0 onward)
                        x  Dead (Linux 2.6.33 to 3.13 only)
                        K  Wakekill (Linux 2.6.33 to 3.13 only)
                        W  Waking (Linux 2.6.33 to 3.13 only)
                        P  Parked (Linux 3.9 to 3.13 only)
...
              (14) utime  %lu
                        Amount of time that this process has been scheduled
                        in user mode, measured in clock ticks (divide by
                        sysconf(_SC_CLK_TCK)).  This includes guest time,
                        guest_time (time spent running a virtual CPU, see
                        below), so that applications that are not aware of
                        the guest time field do not lose that time from
                        their calculations.
              (15) stime  %lu
                        Amount of time that this process has been scheduled
                        in kernel mode, measured in clock ticks (divide by
                        sysconf(_SC_CLK_TCK)).
...
              (39) processor  %d  (since Linux 2.2.8)
                        CPU number last executed on.
```

## Project Guide (part 1)
The myps tool will collect the following information on each process from the /proc file system and store the data in a ProcEntry struct. 

    - pid - The pid of every process
    - comm - The filename of the executable
    - state - The state of the process (Running, Sleeping, etc)
    - utime - The amount of time that the process has been scheduled in user mode
    - stime - The amount of time that the process has been schedule in kernel mode
    - processor - The CPU number last executed on.

The only field that is extra is the char *path field. This field is use to store the file path to the stat file that you loaded. Normally this will be /proc/[pid]/stat unless the user uses the -d flag (described below) to load a different directory.

**TODO:** Carefully study the provided ProcEntry.h file including both the provided ProcEntry struct and the documentation for each support function.  Each ProcEntry will represent a single process on the system. Implement the specified support functions in ProcEntry.c.  Do not modify the provide ProcEntry.h file as the provided struct definition and function declarations will be used to test this portion of your project.  

**HINT:** Lab10 will be a great reference for this portion of the project.  It demonstrates the Create/Destroy design pattern as well as how to read data from files and load it into a struct.  It isn't and exact match, but a solid understanding of the concepts presented in Lab10 will be incredibly helpful here.

**TESTING:** Add test cases to mytests.c as you implement the functions declared in ProcEntry.h. As you write the tests, look for ways to exercise all the code in your functions.  It isn't practical to go for 100% code coverage,but 80 to 90% should be doable.  I will be running my own set of unit tests against your projects as part of the grading process so it would be a good idea to test the functions to ensure they handle expected and unexpected conditions as specified in the comments provided in ProcEntry.h

When testing, be certain to check the test cases with valrind. The provided makefile includes a **memtest-mytests** rule to assist with this testing.
```
make memtest-mytests 
valgrind --tool=memcheck --leak-check=yes --show-reachable=yes ./mytests
==72756== Memcheck, a memory error detector
==72756== Copyright (C) 2002-2017, and GNU GPL'd, by Julian Seward et al.
==72756== Using Valgrind-3.15.0 and LibVEX; rerun with -h for copyright info
==72756== Command: ./mytests
==72756== 
Create/Destroy Test passed
CreateFromFile/Destroy Test passed
            999     S     0     0    1 (gmain)                   test_proc/999/stat  
CreateFromFile/Print/Destroy Test passed
CreateFromFile NULL Test passed
CreateProcEntryFromFile: No such file or directory
CreateFromFile DoesNotExist Test passed
CreateFromFile InvalidFormat Test passed
==72756== 
==72756== HEAP SUMMARY:
==72756==     in use at exit: 0 bytes in 0 blocks
==72756==   total heap usage: 16 allocs, 16 frees, 15,478 bytes allocated
==72756== 
==72756== All heap blocks were freed -- no leaks are possible
==72756== 
==72756== For lists of detected and suppressed errors, rerun with: -s
==72756== ERROR SUMMARY: 0 errors from 0 contexts (suppressed: 0 from 0)
```

## Project Guide (part 2)
Once the functions specified


Referring back to the arrays and user defined functions labs we can construct a function to allocate an array of struct proc to store our data in:

```
struct proc* make_array(void)
{
     struct proc *rval = calloc(MAX_PROCESS,sizeof(struct proc));
     if(rval == NULL){
          fprintf(stderr,"calloc returned NULL! Out of memory!");
          abort();
     }
     return rval;
}
```

Don’t forget to define a function to free all the memory that you allocate!

```
void destroy_array(struct proc *procs)
{
     if(procs == NULL) return;

     for(int i =0;i<MAX_PROCESS;i++){
       //TODO: Free all memory!
     }
     free(procs);
}
```

## Loading information from /proc
Once we have our data structure defined and we have made functions to create and destroy them (constructor/destructor) we need to make a function to iterate over every directory in the /proc directory, open and load the stat file. This step will be a bit tricky because as you will find out there are other files in the /proc directory that you don’t want to load. You will need to write some code to filter that out.

We should use the gnu libc manual as a reference. The GNU documentation has examples on how to open and access files in a directory. We have reproduced the directory lister below for convenience.

Accessing directories
Directory Entry
Directory Lister

```
#include <stdio.h>
#include <sys/types.h>
#include <dirent.h>

int
main (void)
{
  DIR *dp;
  struct dirent *ep;

  dp = opendir ("./");
  if (dp != NULL)
    {
      while (ep = readdir (dp))
        puts (ep->d_name);
      (void) closedir (dp);
    }
  else
    perror ("Couldn't open the directory");

  return 0;
}
```

## Handling command line arguments
Your program must accept the following command line inputs with the specified behavior and use the getopt library.

```
./mylab -p When the user passes the -p switch your program will display the results in pid order to stdout and exit
./mylab -c When the user passes the c switch your program will display the results in comm order (lexicographic order) to stdout and exit
./mylab -z When the user passes the z switch your program will display the zombies to stdout and exit
./mylab -d dir When the user passes the d switch your program will start parsing the dir instead of /proc. For example the command ./mylab -d $HOME/test -p will treat the directory $HOME/test as the /proc directory and show the results in pid order.
./mylab When the user gives no command line arguments your program will default to the -p behavior.
./mylab -h When the user passes the h switch your program will display a help message to stdout and then exit.
```

If the user does not specify the -d flag you will default to the /proc directory!

Here is an incomplete example of using getopt in your program:

```
char *dir = "/proc";
int pid_o = 0;
int comm_o = 0;
int zombies_o = 0;
int c;

while ((c = getopt(argc, argv, "pczd:")) != -1)
    switch (c)
    {
    case 'p':
          pid_o = 1;
          break;
    case 'c':
          comm_o = 1;
          break;
    case 'z':
          zombies_o = 1;
          break;
    case 'd':
          dir = optarg;
          break;
    case '?':
          if (optopt == 'd')
              fprintf(stderr, "Option -%c requires an argument.\n", optopt);
          else if (isprint(optopt))
              fprintf(stderr, "Unknown option `-%c'.\n", optopt);
          else
              fprintf(stderr, "Unknown option character `\\x%x'.\n", optopt);
          return 1;
    default:
          abort();
    }

```

## Sorting
You will need to sort your array of struct proc’s depending on what command line flag is set. There is no need to reinvent the wheel! The C standard library has an implementation of quick sort that you must use!

Here is an incomplete example of a function that will sort pid order:

```
static int sort_pid(const void *a, const void *b)
{
     ProcEntry *f = *(ProcEntry **)a;
     ProcEntry *s = *(ProcEntry **)b;
     int rval = f->pid - s->pid;
     return rval;
}

qsort(procs, num_loaded, sizeof(ProcEntry *), sort_pid);

```

## Final output
Once you have loaded and sorted the process you should output the results. Failure to use the specified output will significantly impact your grade!

Your program must output the information using the function defined below. You should copy the function below verbatim and use it in your submission!

```
void PrintProcEntry(ProcEntry *entry)
{
     unsigned long int utime = entry->utime / sysconf(_SC_CLK_TCK);
     unsigned long int stime = entry->stime / sysconf(_SC_CLK_TCK);
     fprintf(stdout, "pid:%d comm:%s state:%c utime:%lu stime:%lu processor:%d path:%s\n",
             entry->pid,
             entry->comm,
             entry->state,
             utime,
             stime,
             entry->proc,
             entry->path);
}
```

## Test Data
We have compiled some test data for you to use.

Download test_proc_dir.tgz and use the command tar xzf test_proc_dir.tgz to unzip and untar all the files.

```
shane:Downloads$ tar xzf test_proc_dir.tgz
shane:Downloads$ ls
test_proc         test_proc_dir.tgz
```

### Example Output
Below are a few examples for what your output should look like. Both examples were generated from the test data provided. However only 10 entries are shown out of 500.

You will need to update the paths below to match where you downloaded and untar’d the test data!

#### -d switch
```
shane|(master *%=):final$ ./mylab -d `pwd`/test_proc/
pid:92 comm:kthrotld state:I utime:0 stime:0 processor:1 path:/Users/shane/repos/c-devel/labs/final/test_proc/92/stat
pid:95 comm:irq/26-pciehp state:S utime:0 stime:0 processor:1 path:/Users/shane/repos/c-devel/labs/final/test_proc/95/stat
pid:104 comm:irq/35-pciehp state:S utime:0 stime:0 processor:0 path:/Users/shane/repos/c-devel/labs/final/test_proc/104/stat
pid:132 comm:ipv6_addrconf state:I utime:0 stime:0 processor:1 path:/Users/shane/repos/c-devel/labs/final/test_proc/132/stat
pid:300 comm:kworker/1:1H-kblockd state:I utime:0 stime:0 processor:1 path:/Users/shane/repos/c-devel/labs/final/test_proc/300/stat
pid:763 comm:cleanup state:S utime:0 stime:0 processor:1 path:/Users/shane/repos/c-devel/labs/final/test_proc/763/stat
pid:1058 comm:gdbus state:S utime:0 stime:0 processor:1 path:/Users/shane/repos/c-devel/labs/final/test_proc/1058/stat
pid:1093 comm:gsd-wwan state:S utime:0 stime:0 processor:0 path:/Users/shane/repos/c-devel/labs/final/test_proc/1093/stat
pid:2872 comm:pool-org.gnome. state:S utime:5 stime:1 processor:1 path:/Users/shane/repos/c-devel/labs/final/test_proc/2872/stat
pid:2875 comm:snapd state:S utime:1 stime:0 processor:0 path:/Users/shane/repos/c-devel/labs/final/test_proc/2875/stat
```

#### -c switch with -d switch
```
shane|(master *%=):final$ ./mylab -d `pwd`/test_proc/ -c
pid:763 comm:cleanup state:S utime:0 stime:0 processor:1 path:/Users/shane/repos/c-devel/labs/final/test_proc/763/stat
pid:1058 comm:gdbus state:S utime:0 stime:0 processor:1 path:/Users/shane/repos/c-devel/labs/final/test_proc/1058/stat
pid:1093 comm:gsd-wwan state:S utime:0 stime:0 processor:0 path:/Users/shane/repos/c-devel/labs/final/test_proc/1093/stat
pid:132 comm:ipv6_addrconf state:I utime:0 stime:0 processor:1 path:/Users/shane/repos/c-devel/labs/final/test_proc/132/stat
pid:95 comm:irq/26-pciehp state:S utime:0 stime:0 processor:1 path:/Users/shane/repos/c-devel/labs/final/test_proc/95/stat
pid:104 comm:irq/35-pciehp state:S utime:0 stime:0 processor:0 path:/Users/shane/repos/c-devel/labs/final/test_proc/104/stat
pid:92 comm:kthrotld state:I utime:0 stime:0 processor:1 path:/Users/shane/repos/c-devel/labs/final/test_proc/92/stat
pid:300 comm:kworker/1:1H-kblockd state:I utime:0 stime:0 processor:1 path:/Users/shane/repos/c-devel/labs/final/test_proc/300/stat
pid:2872 comm:pool-org.gnome. state:S utime:5 stime:1 processor:1 path:/Users/shane/repos/c-devel/labs/final/test_proc/2872/stat
pid:2875 comm:snapd state:S utime:1 stime:0 processor:0 path:/Users/shane/repos/c-devel/labs/final/test_proc/2875/stat
```
## Hints
Run Valgrind often! Don’t wait until you have finished the program to check for leaks or errors
When parsing the comm field be aware that 1.(helloworld) 2.(hello world) 3.(hello) (world) are all valid values for comm. So make sure your parsing code is robust! There are LOTS of different permutations for comm your code must handle them all!
Leverage the test data we gave you or create your own!!
