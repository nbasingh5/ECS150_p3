# ECS 150 Project 4 Report

### Part 1: Semaphore API
This phase involved building out a semaphore API for our threads to share common resources.

##### Implementation
Each semaphore is assigned a struct with the following parameters within it.
```c
struct semaphore {
  queue_t blocked_queue;
  int count;
};
```
There is a count integer stored inside the semaphore with the number of resources available to the semaphore and a queue object that contains the TID's of the threads that are waiting for the resource to be available.

### Part 2: Thread Private Storage (TPS) API
This API allows our threads to create their own private storage where they can freely read and write memory, but is protected from other processes trying to access that memory.

##### Implementation
Each TPS is assigned a struct with the following parameters within  it.
```c
struct TPS{
	pthread_t TID;
	struct Page* page;
};
```
The TPS stores the TID of the thread associated with it as well as a pointer to the Page object assigned to it. The page object is a seperate struct that contains a pointer to the page of memory allocated to that Page object as well as a count of the TPS's currently pointing to it (this is done to implement copy-on-write). The declaration of the Page struct is as follows.
```c
struct Page{
	void* pageptr;
	int count;
};
```
A more in-depth look at the operation of the TPS API is as follows:
* Each thread with a TPS is assigned a TPS structure that is stored in a static global queue. This allows the library to deal with finding the associated TPS with each thread. When a new TPS is created for a thread, it is assigned the TID of the thread done by calling `pthread_self()` and a new Page object is created with the memory page assigned with it using `mmap()`. The protection of this memory address is set to *PROT_NONE* so that a segmentation fault is caused if this memory address is accessed.
* The TPS of a thread can also be cloned by another thread, which causes the page pointer of the cloned thread to be copied into the current thread and the count of the Page object is increased by 1. This signifies that another thread is pointing to this page memory address, so that a write to this address does not modify the data for both of them. This is done to implement copy-on-write protection so that the contents are copied over to a new Page object only when necessary. More details about this is included in the next paragraph while explaining how the write implementation works.
* Writing to a TPS invloves the use of `memcopy()` which copies the contents of  provided buffer at a certain offset in the memory page associated with the TPS. This function however also checks if the Page object associated with the TPS is greater than 1. If this is the case, it would imply that multiple TPS's reference this page and a write would affect all of them which would be less than ideal. So in this case, it means that we have to actually perform the copy-on-write where we create a new Page object (with a new associated memory page pointer) and copy everything from the old page into the newly created page. Once this is performed it proceeds to perform the write as described in the first line.
* Reading from a tps is much simpler than writing as we dont have to deal with copy-on-write operations. We simply use `memcopy()` to copy values from the offsetted page pointer to the buffer provided as an input to the function.

