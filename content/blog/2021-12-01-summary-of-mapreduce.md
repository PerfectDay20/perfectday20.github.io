+++
title =  "Summary of MapReduce"
+++

Many computation tasks can be expressed in map tasks and reduce tasks.

The system takes care of parallelization, scheduling, failure handling, so we can focus on the computation logic and utilize distributed system easily.

The MapReduce system contains two components: master and worker. Master needs to schedule map and reduce tasks to workers, transfer task info between them, reschedule failed tasks, monitor job states. Workers receive tasks and do the actual computation, report states to master.

The cluster is both used for computing and data warehouse, so Master can schedule map tasks along side the data to conserve network bandwidth. Because network bandwidth is scarce resource.

Map tasks convert input to intermediate key-value pairs in memory buffer, then periodically write to disk. Each map task will produce R(the number of reduce tasks) intermediate files. (So totally M*R files will be created, Spark has an optimization on this)

Master passes the intermediate files info to reducer, reducer pulls intermediate data from mapper, sorts data by keys then iterates the values, reduce values to the final result.

The number of map and reduce tasks should be much larger than the number of workers for better dynamic load balancing and fast recovery(one worker's tasks can be shared by more workers if task number is larger).

Backup tasks for straggler tasks: schedule tasks when a MapReduce job is close to finish, using no more than a few percent of resource. (Different workers can behave dramatically different, so using backup tasks can cut off long-tail tasks that due to resource deficiency)

Other refinements:

- User provided partitioner
- data is in increasing order within partition
- combiner merges data within mapper
- option to skip bad input records, rely on Master
- local execution for debug
- status monitoring
- counter for debug and sanity check, rely on Master

> [R]stricting the programming model makes it easy to parallelize and distribute computations and to make such computations fault-tolerant.
