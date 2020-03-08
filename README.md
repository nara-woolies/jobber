# Jobber

Jobber is a job queue service implemented using GO and Redis. 

#### Job Type
Jobber has the concept of a _Job Type_. Each _job type_ gets its own worker queue. One can have a number of worker instances running to execute job depending upon the load on the worker queue. A future enhancment would be be auto-scale the number of workers depending upon the jobs that are queued in the worker queue.

#### Recovering a job
Jobber supports recovering a job. If a worker fails during the execution of a job, the job can be recovered and executed by another worker. 
Recovery is supported using the concept of _Visiblity timeout_

#### Visiblity timeout
When a worker picks up a job to execute it has fixed amount of time in which to complete it. Within this time window the job is not _visible_ to other workers. This time period is called as _Visiblity timeout_
If the worker fails to do so, then a recoverer process recovers the job and pushes it back onto the worker queue.

# Jobber API
The API supports operation to queue a job:
_Request_
```
POST: https://<host>/jobber/queue
Body:
{
    "type": "job-type"
    "payload": { 
        /* free-form json payload */
    }
}
```
_Response_
```
201 OK
{
    "job-id": "uuid"
}
```

# Design details
```
Diagram here
```

## Sequence
1. The API accepts a new job and queues it a queue named `job_queue_job-type`. 
2. An instance of the worker is ready to take a new job, it tries to acquire lock on the job (job-id). The lock is set to auto expire after `visibility_timeout` time period.
3. If successful, it then does RPOPLPUSH, poping the job from the worker queue and pushing it onto `in_process_queue`
4. Worker then works to finish the job and then deletes it from `in_process_queue`
5. If in case worker is unable to finish the job and delete it from `in_process_queue`, then the recoverer process pushes it back onto the worker queue

## How does the Recoverer recovers jobs?
1. Recoverer regularly scans the `in_process_queue` for jobs that do not have an active lock 
2. If it finds a job present in the `in_process_queue` but without an active lock, it then pops that job and pushes it back onto the worker queue

## Steps followed by the worker
The worker follows the below steps:
	1. Read the head of the queue
	2. Try and lock the job
	3. If it cant, return and go back to waiting for the next job
	4. If the job is locked, then the worker pops the job from the worker queue and pushes it to in_process_queue (It does this using RPOPLPUSH so that the push and pop operations are done in a single step)
	5. Meanwhile if the recoverer tries to recover a job, it finds that there is an active lock on the job and it returns
	6. Process the job
	7. Delete from in_process_queue
	8. Delete the lock

If the worker fails in processing the job, the job remains in in_process_queue and the locks expires. The recoverer can then recover the job. Currently the recoverer attempts to recover only the job at the head of the queue. It does not look further down the queue. This should not be a problem as along the visibility timeouts are small and it is not highly critical to recover the jobs relatively early. 

## Future enhancements
1. Making the recovere look through the in_proceses_queue to recover jobs