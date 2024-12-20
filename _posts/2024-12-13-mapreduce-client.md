---
layout: post
title:  "Building a MapReduce Job Submission Client"
date:   2024-12-13 11:02:48 -0700
categories: distributed systems
---
The other day I wrote a bit about the high level goals for the MapReduce project, and today I'm going to attempt to tackle the design of the client component.

## Intro 

### Introduce what we need from job submission client 
What is it?  Imagine a batch job submission interface.  You tell the computer to do some work, submit the job, and then go do other stuff while your job request is being processed.  Later, you come back to check if the result has been output.  In essence, these jobs are asynchronous and potentially long lasting.  The system might be shared with many other users who also want to submit jobs.  Based on this description, let's characterize the behavior of this system:

### Assumptions
The job submission client serves both a user interface and a web service that provides an API for both the UI and the master to consume.  In a production-grade architecture these roles could be divided into separate components but for our purposes it's sufficient to have these roles in one component.
 
### Functional Requirements
The primary role of the job client is to ingest a job submission from a user and maintain a queue of job submissions that are waiting to be processed by the MapReduce component.  Optionally, the ordering of the queue could be governed by a given policy, such as priority, and it's up to the job client to enforce this.  For simplicity we will assume that the queue has FIFO semantics.

Since the master is the orchestrator for the MapReduce jobs, it is the master's responsibility to consume the jobs in the queue.


### Non-Functional Requirements

#### High Availability
We want the job client system to be highly available to users.  We want the exposed job queue to be highly available to any consumer.

#### Fault Tolerance
The most important aspect here is that once a job is submitted by the user, the system ensures that the job is eventually run to completion.  If the job is lost before being processed, then we have a failure of system since it has broken its guarantee to the user.  

#### Other non-functional requirements
In thinking about the non-functional requirements, we observe that the usage patterns for this system are synonymous with a batch processing system.  The MapReduce jobs in general are long-running background processes for which the user submits the job and then comes back at a later time to check on its status.  Additionally, in the future we'd like to support the ability to chain together MapReduce operations, so that the output of one MapReduce becomes the input for the next job.  For these reasons, it's important that the submitted jobs aren't lost even if the application is down or restarted.  This calls for data persistance for the job submissions: a submitted job should remain in the queue until it has been processed by the master. 

### Implementation of Functional Requirements
Let's first decide on the communication patterns between the job submission and its clients, as this will guide our implementation decisions for all the functional requirmeents.  The communication between the user and the job submission component is pretty straight forward.  The user will submit a job by POSTing an html form in the web browser. So they will communicate in a request/response fashion.  A straightforward implementation for this would be for job submission component to implement a web server that provides some HTTP/s endpoints for the user to consume.



### implementation issues

## testing and demo functionality

## lessons learned - outtro


Now that we have the background of what we want the components to achieve, let's design the client component.

What are the responsibilities of the client component?

1. Expose a user interface to the end user that allows her to submit a map reduce job involving various job parameters (ie. Input files, number of workers, etc)

2. Manage the job submission queue, allowing the master component to receive the next submission that is ready to be processed.

3. The service should be highly availability and fault tolerant, and not be a single point of failure for the system

Let's address these issues and find a suitable design that will solve them.

Implementing the first responsibility is pretty straightforward.  We can have a browser render a simple HTML form containing all the parameters that need to be specified by the user.  Submitting a form results in a POST request to the client web server containing the parameter values.

For the second one, first I have to decide how the client manages submissions.  Well, presumably job submissions will be queued up, and the item at the head of the question will be determined by a policy that can be applied on the queue (perhaps FIFO, or priority, or something else).  Since we are going for simplicity, we'll just serve up the jobs in FIFO order for now. 

The second statement in 2) implies that we need to design the communication scheme between the master and client.
 
## Client and Master Communication
At a high-level this involves choosing whether we are going for a polling or push-based design.  I think it's appropriate to require the master ask the client for new jobs.  After all, it's going to be fully preoccupied with the job orchestration and other RPCs that it doesn't need another process pushing messages to it when it's mostly busy doing other stuff.  And in general job submissions aren't going to pile up enough that the client gets overwhelmed.  In the name of simplicity, we'll use HTTP REST API in the client as the interface, since we need a REST API for the job client to interact with the browser.  We can use that same web server for the health checks as well as we'll see.  

I want to expose the queue-like semantics of our job submission data structure, but as it's been implemented as a REST API, I come up with this design for the API:

POST /submission - invoked by the frontend form submission to add a new job submission
GET /submission - invoked by the master to obtain the next available job message if available
DELETE /submission/id - invoked by the master to mark the job as completed

The master calls GET to retrieve the next job, begins processing it, and once completed calls DELETE on the returned message id which effectively dequeues the message.


## High Availability and Fault Tolerance

If we design this the right way Kubernetes will give this to us easily.  The client looks like a glorified CRUD app.  So we can spin up many replicas, and they are all stateless, so it doesn't matter to which replica the client get routed.  But how do the job submissions get collated?  Let's say the ingress routes all the client submission url requests randomly, then a user could post a submission to one replica, and another user could post a submission to a different replica.  And at the time a service is elected master, it must be able to receive the current state of all the submissions from all the replicas.  Now, I’m going to make the assumption that we want persistence for the job submissions, since it would be cool to have a history of all the job submissions for a particular user.  Persisting the job submission also greatly facilitates the requirement of fault tolerance.  It minimizes the likelihood that a job submission gets lost.  Alternative implementations could just broadcast all received submissions to other replicas and try to keep this all in memory, but that's not what we're going to do.  Let's keep it simple and strive for reliability.   Now the choice we are faced with is:
1. does each replica maintain its own persistent location and then query all the replicas to get all the submissions, or
2. The replicas share a persistent location and they all read and write to this

I mean 1) is good for spreading the data persistence across many nodes, which can increase reliability.  This is analogous to trading off convenience of having all the data in one place and risking losing it all if there is a failure in the underlying persistence mechanism vs. distributed data, which results in just a fraction of data loss if a replica's persistence mechanism at the expense of having to stitch together data from all the replicas for each request. (Talk more about this, and refine the discussion) We are going with a shared persistence solution this reason (next paragraph). 

We are going to go with 2, since we are using a cloud-native platform, and shared persistent storage can be extremely resilient, minimizing the chance we are going to lose data.  We also don't care much about our sole storage mechanism as a bottleneck for reads and writes.  In general, the job execution of MapReduce must be way more scalable than the job submission aspect.  It might take a user hours to develop and submit a MapReduce job that may have tens or hundreds of tasks invoke hundreds or thousands of workers (in the MapReduce paper some jobs used X workers).
 
Now we know how we are going to handle the data, lets focus on the cloud implementation details.

## Implementing the data strategy in Kubernetes

I said that I'd be using a shared persistent data store to store the job submissions so let's implement that.  The target cloud platform for the project is Microsoft Azure - AKS, but I don't want the implementation to be tied to a particular vendor, so I need a mechanism to abstract the details.  Enter Kubernetes persistent volumes (PV). 

## Persistent Volumes

Eventually we will need an Azure specific PV, but for now let's just figure out a local solution.  As far as the storage technology is concerned, I just chose to use a simple text file with lines containing serialized structs representing job submission records.  This is great for writing new records, but not so great at mutating existing records, since we will have to potentially scan the entire file to find the record to change then rewrite the entire file with its changes.  Alternatively we can use an append log-type structure to record changes in the state of the job submissions.  Eventually we can use a full-fledged database if we need more functionality.  Either way, for right now our interface to the data as far is the system is concerned will be "file-like," so let's find a suitable local file sharing mechanism that our local k8s cluster supports.  The solution I chose for this is a local type of PV.  These are similar to hostPath PV's, except that they should work in an multi-node environment, since the scheduler knows to which node a file mount belongs in the "local" type and will only schedule pods referencing this type of PV to that node.

<insert code sample>
Here I've mounted a directory called "data" as a local PV and referenced it in the job submission pods.




## Deployment and Testing
Let's interact with our job client through a k8s service that routes the requests randomly.  We will write to a log when a particular API gets handled on the client.

First we POST a few job submissions through the service endpoint:  We see in the logs that it hits pod 1 and has written the file successfully.  Now let's call GET a few time.  After the 2nd try,  we see that Pod 2 has serviced our GET request and can see the data that Pod 1 has written.  

The job submission had an ID attached to it, so let's now delete the submission to indicated that we're done with it.  Now we call GET which returns the next submission in the queue.  Another call to DELETE, and the next GET returns empty.  Good job, we have processed all the available jobs! 

## Other Problems

I previously chose to incorporate the job submission client with the master, but there are problems with that:

	1. The job submission replicas are stateless, whereas the master replicas aren't.  That makes it hard to manage that in Kubernetes
	2. Readiness and Liveness probes.  What does it mean when a master is "ready"?  There's no way to indicate if it means both the master and client servers are ready , so it's an overloaded operation.  That makes it a design smell
	3. Separation of concerns - the master has a complex set of responsibilites. Job submission is not one of them
	  
 

	
