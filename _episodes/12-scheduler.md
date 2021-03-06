---
title: "Using a cluster: Scheduling jobs"
teaching: 45
exercises: 30
questions:
- "What is a scheduler and why are they used?"
- "How do we submit a job?"
objectives:
- "Submit a job and have it complete successfully."
- "Understand how to make resource requests."
- "Submit an interactive job."
keypoints:
- "The scheduler handles how compute resources are shared between users."
- "Everything you do should be run through the scheduler."
- "A job is just a shell script."
- "If in doubt, request more resources than you will need."
---

An HPC system might have thousands of nodes and thousands of users.
How do we decide who gets what and when?
How do we ensure that a task is run with the resources it needs?
This job is handled by a special piece of software called the scheduler.
On an HPC system, the scheduler manages which jobs run where and when.

The scheduler used in this lesson is PBS Pro (PBS for short).
Although PBS is not used everywhere, 
running jobs is quite similar regardless of what software is being used.
The exact syntax might change, but the concepts remain the same.

## Running a batch job

The most basic use of the scheduler is to run a command non-interactively.
This is also referred to as batch job submission.
In this case, we need to make a script that incorporates some arguments for PBS such as resources needed and modules to load. An example has been included in the Zip on the setup page. This is just a bash script with some parameters set.

```
test.pbs
```
```
#!/bin/bash
#PBS -q workq
#PBS -A qris-gu
#Give the job a name ... in the past had to start alphabetical and be < 13 chars
#PBS -N test_script
#PBS -l walltime=00:03:00
#PBS -l select=1:ncpus=2:mem=2g

echo 'This script is running on:'
hostname
echo 'The date is :'
date
sleep 120
```

Please note, you may need to change qris-gu to your group. You can find out which group you are part of by running the `groups` command.

~~~
~> groups
qris-gu support4 Q0183RW

~~~


We will talk about the other parameters set above and their defaults later in this lesson.

As this is just a bash script, you can run it as any bash script on the head node. Please note- its very very important not to run any large or resource heavy script on the head node, as every user relies on this head node.

To test a small pbs script, run it as:
```
~> ./test.pbs
This script is running on:
awoonga1.local
The date is :
Wed Jan 10 11:27:38 AEST 2018
```
This script will take 2 minutes to finish due to the `sleep 120` command.

If you get a Permissions denied error, you will need to give your script executable permissions as per the last lesson.

```
~> chmod +x test.pbs
```

If you completed the previous challenge successfully, 
you probably realize that there is a distinction between 
running the job through the scheduler and just "running it".
To submit this job to the scheduler, we use the `qsub` command.

```
~> qsub test.pbs
25333.awongmgmr1
~>
```
The number that first appears is your Job ID. When the job is completed, you will get two files: an Output and an Error file (even if there is no errors). They will be named {JobName}.o{JobID} and {JobName}.e{JobID} respectively.

And that's all we need to do to submit a job. 
To check on our job's status, we use the command `qstat`.

```
~> qstat
Job id            Name             User              Time Use S Queue
----------------  ---------------- ----------------  -------- - -----
25399.awongmgmr1  test_script      amandamiotto      00:00:00 R Short  
```

{: .output}

We can see all the details of our job, most importantly that it is in the "R" or "RUNNING" state.
Sometimes our jobs might need to wait in a queue ("PENDING") or have an error.
The best way to check our job's status is with `qtat`.

## Customizing your job

### Parameters

Let's discuss the example PBS script.
```
test.pbs
```
```
#!/bin/bash
#PBS -q workq
#PBS -A qris-gu
#Give the job a name ... in the past had to start alphabetical and be < 13 chars
#PBS -N test_script
#PBS -l walltime=00:03:00
#PBS -l select=1:ncpus=2:mem=2g

echo 'This script is running on:'
hostname
echo 'The date is :'
date
sleep 120
```

Comments in UNIX (denoted by `#`) are typically ignored.
But there are exceptions.
For instance the special `#!` comment at the beginning of scripts
specifies what program should be used to run it (typically `/bin/bash`).
Schedulers like PBS also have a special comment used to denote special 
scheduler-specific options.
Though these comments differ from scheduler to scheduler, 
PBS's special comment is `#PBS`.
Anything following the `##PBS` comment is interpreted as an instruction to the scheduler.

In our example, we have set the following parameters:
 
| Flag | Name | Example Setting | Notes|
| --- | --- | --- | --- |
| -q | queue | workq | See next section for queue info |
| -A | account |qris-gu| Discuss with your admins or university re account |
| -N | jobname| test_script | Name of your script (no spaces, alphanumeric only) |
| -l | resource list| multiple settings| See next segment|

### Resource list
Resource list will contain a number of settings that informs the PBS scheduler what resources to allocate for your job and for how long (walltime).

#### Walltime
Walltime is represented by `walltime=00:01:01` in the format HH:MM:SS. This will be how long the job will run before timing out.  If your job exceeds this time the scheduler will terminate the job. It is recommended to find a usual runtime for the job and add some more (say 20%) to it. For example, if a job took approximately 10 hours, the walltime limit could be set to 12 hours, e.g. "-l walltime=12:00:00". By setting the walltime the scheduler can perform job scheduling more efficiently and also reduces occasions where errors can leave the job stalled but still taking up resource for the default much longer walltime limit (for queue walltime defaults run "qstat -q" command)

Resource requests are typically binding.
If you exceed them, your job will be killed.
Let's use walltime as an example.
We will request 30 seconds of walltime, 
and attempt to run a job for two minutes.

```
#!/bin/bash

#PBS -A qris-gu
#PBS -l walltime=00:00:30  ## <- altered to 30 seconds
#PBS -l select=1:ncpus=2:mem=2g

echo 'This script is running on:'
hostname
echo 'The date is :'
date
sleep 120

```

Submit the job and wait for it to finish. 
Once it is has finished, check the error log file. In the error file, there will be
```
=>> PBS: job killed: walltime 77 exceeded limit 30

```

Our job was killed for exceeding the amount of resources it requested.
Although this appears harsh, this is actually a feature.
Strict adherence to resource requests allows the scheduler to find the best possible place
for your jobs.
Even more importantly, 
it ensures that another user cannot use more resources than they've been given.
If another user messes up and accidentally attempts to use all of the CPUs or memory on a node, 
PBS will either restrain their job to the requested resources or kill the job outright.
Other jobs on the node will be unaffected.
This means that one user cannot mess up the experience of others,
the only jobs affected by a mistake in scheduling will be their own.

#### Compute Resources and Parameters
Compute parameters, represented by `select=1:ncpus=2:mem=2g` can be considered individually. The argument `select` specifies the number of nodes (or chunks of resource) required; `ncpus` indicates the number of CPUs per chunk required.


| select |  ncpus |  Description|
|---|---|---|
| 2|  16|  32 Processor job, using 2 nodes and 16 processors per node| 
| 4|  8|  32 Processor job, using 4 nodes and 8 processors per node| 
| 16|  1|  16 Processor job, using 16 nodes and 1 processor per node| 
| 8 | 16 | 128 Processor job, using 8 nodes and 16 processors per node|



Each of these parameters have a default setting they will revert to if not set however this means your script may act differently to what you expect.

You can find out more information about these parameters by viewing the manual page of the `qsub` function. This will also show you what the default settings are.

```
man qsub
```


## Queues

There are usually a number of available queues to use on your HPC. To see what queues are available, you can use the command `qstat -Q`. If you are not sure which to use, workq is a good start and is generally set as the default.

```
~> qstat -Q
Queue              Max   Tot Ena Str   Que   Run   Hld   Wat   Trn   Ext Type
---------------- ----- ----- --- --- ----- ----- ----- ----- ----- ----- ----
workq                0     0 yes yes     0     0     0     0     0     0 Rout
Short                0    18 yes yes     1    17     0     0     0     0 Exec
Single               0    36 yes yes     0    36     0     0     0     0 Exec
Interact             0     8 yes yes     0     7     0     0     0     0 Exec
Long                 0     2 yes yes     0     2     0     0     0     0 Exec
Special              0     0 yes yes     0     0     0     0     0     0 Exec
DeadEnd              0     0 yes  no     0     0     0     0     0     0 Exec
```
{: .bash}


> ## Submitting resource requests
>
> Submit a job that will use 2 cpus, 4 gigabytes of memory, and 5 minutes of walltime.
{: .challenge}


## Job environment variables

PBS sets multiple environment variables at submission time. The following PBS variables are commonly used in command files: 


| Variable Name |  Description |
|---|---|
| PBS_ARRAY_INDEX|  Array ID numbers for jobs submitted with the -J flag. For example a job submitted with #PBS -J 1-8 will run eight identical copies of the shell script. The value of the PBS_ARRAY_INDEX will be an integer between 1 and 8.|
| PBS_ENVIRONMENT|  Set to PBS_BATCH to indicate that the job is a batch job; otherwise, set to PBS_INTERACTIVE to indicate that the job is a PBS interactive job.|
| PBS_JOBID|  Full jobid assigned to this job. Often used to uniquely name output files for this job, for example: mpirun - np 16 ./a.out >output.${PBS_JOBID}|
| PBS_JOBNAME|  Name of the job. This can be set using the -N option in the PBS script (or from the command line). The default job name is the name of the PBS script.|
| PBS_NODEFILE|  Contains a list of the nodes assigned to the job. If multiple CPUs on a node have been assigned, the node will be listed in the file more than once. By default, mpirun assigns jobs to nodes in the order they are listed in this file |
| PBS_O_HOME|  The value of the HOME variable in the environment in which qsub was executed.|
| PBS_O_HOST|  The name of the host upon which the qsub command is running.|
| PBS_O_PATH|  Original PBS path. Used with pbsdsh.|
| PBS_O_QUEUE|  Queue job was submitted to.|
| PBS_O_WORKDIR|  PBS sets the environment variable PBS_O_WORKDIR to the directory from which the batch job was submitted PBS_QUEUE Queue job is running in (typically this is the same as PBS_O_QUEUE). |

## Canceling a job


Sometimes we'll make a mistake and need to cancel a job.
This can be done with the `qdel` command.
Let's submit a job and then cancel it using its job number.

```
> qsub test2.pbs
27790.awongmgmr1

> qstat
Job id            Name             User              Time Use S Queue
----------------  ---------------- ----------------  -------- - -----
27790.awongmgmr1  test2.pbs        amandamiotto      00:00:00 R Short           

```

Now cancel the job with it's job number. 
Absence of any job info indicates that the job has been successfully canceled.

```
> qdel 27790
> qstat
>
```



