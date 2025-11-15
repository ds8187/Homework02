# For Graphs Reference Attatched PDF

Mutex [30 Points] 


Short Answer Analysis for Prior Code’s Unintended Behaviors: 
An entry is lost when two or more threads call the insert method at the same time. As a consequence of kernel threads being preemptable, the last threads that inserts into the bucket_entry *table[i] will be kept. At a high-level what will occur is two or more threads will call insert with a key that maps to the same index, i, in the table in an interleaved fashion. For simplicity lets use 2 threads. Initially two threads will create their bucket_entry object and one of the threads will be preempted (reason can vary from clock interrupt, device interrupt, exceptions, page fault, or segmentation errors) and set to the sleeping state. Now, let's say that thread1 is running and starts pointing its new bucket_entry.next pointer towards the current “head” bucket_entry at table[i] and then replacing its new bucket_entry as the “head” of the list at table[i]. After this thread1 is now interrupted and replaced by thread2 which repeats the same steps as the first thread before it is interrupted. The key issue that occurred was the the new entry which thread1 added first was overwritten by thread2 as it replaced itself as the “head” of the linked list at table[i] and because thread2 filled up its next pointer before thread1 had finishing entering its bucket_entry into the table it was missed by thread2. This code is susceptible to race conditions when more than one thread is involved due to its lack of synchronization primitives.





Graph of Original Code Running Time vs Mutex-Based Running Time:


Running Time Analysis:
The addition of mutex impacts both the insertion and retrieval runtime in different ways. For insertion, adding a mutex increases the overall time for all threads between 1-10. Although we saw the runtime was fairly close between 1-2 threads and after 8 threads. This was a result of balancing the trade-off between the overhead cost of mutex vs the amount of threads competing. In the beginning 1-2 threads do not compete with each other as much because as one thread is doing the work the other is sleeping (even if on different CPU cores). The resulting pattern is just alternating threads which result in moderate lock/unlock calls made. The runtime difference changes as the number of threads increases because the additional threads call lock at the same time forcing them into the wait queue in the kernel via system call. This results in the program/process to behave sequentially since mutex unlock only sets one thread's state as RUNNABLE at a time. For example our environment (docker container with Ubuntu image) has 8-core which causes anything below 8 cores to grab the lock at the same time resulting in all threads states being updated. After exceeding 8 cores, or amount in your system, everything changes. In the mutex program everything behaves the same as soon as the additional threads get a chance to run (They are in RUNNABLE until one of the 8 cores frees up), it then calls lock and goes to the wait queue. In contrast, the unsynchronized program keeps adding new threads in the RUNNABLE queue increasing competition for the shared resource leading to more race condition events now via preemption. So, performance starts to become better for the mutex version because the initial gain of running in true parallel across all 8 cores is replaced with many context switches due to a filled up RUNNABLE queue with invalidated caches when restarted. The insertion performance can be summarized as: the added safety of synchronization comes with a performance trade-off. 

Now for the retrieval function, the original program’s retrieval runtime increases without synchronization because partial time is wasted incrementing the lost variable for the thousands of “keys” not found in table situations. In addition, a lot of time is wasted having to traverse the full linked list when the key was not added (explained more in the spinlock section). This starts to stabilize near the end because we exceed the available core in the CPU which adds the context switching overhead to both programs which end up being the largest bottleneck in performance. This is because adding processes in and out of states, sleeping/waking up threads, saving/loading registers, swapping stacks, and clearing TLB wastes a lot more time than traversing a longer list for missing keys. In addition, the linked lists start to get shorter near 8 threads because there could be 7 nodes lost from having 8 inserts happening at the same time with a stale state of the table at index i “head”. Then only the last of the 8 will be persisted, missing 7 levels that would otherwise be added to the linked list.  

Estimate of Timing Overhead and Explanation: 
To calculate the overhead from using mutexes, it is important to just focus on the time differences from the put phase. This is because insertion is the only function that utilizes mutexes for synchronization. Retrieval does not. The way that we calculated overhead was to take the average difference in time for each thread quantity using the formula: (TimeToInsert_mutex - TimeToInsert_original) / NumberOfThreads. Next Average all ten of those values (one per each thread quantity measured) 

Here is the time to insert for the original code and mutex based code:


Here is the Average over head for the mutex based code by thread (and total):


The overhead for using mutexes for a given thread is about .0026 seconds per thread.
Spinlock [30 Points] 

Short Answer of what you expect to happen when replacing mutexes with spinlocks:
The effect on the running time would differ depending on how many threads are used. For a small amount of threads, the spinlocks would actually be faster because they don’t involve putting the threads waiting on a resource to sleep. A spinlock requires less overhead than a mutex because there is less context switching, so as long as there is not a large quantity of threads using the CPU in a “busy wait”, the spinlocks will be faster. If too many threads are used then the mutexes will be faster because the “busy wait” from a lot of threads will take over the CPU. In this case the sleeping used by mutexes is more efficient.

Graph of Prior Running Time vs Mutex Running Time vs Spinlock Running Time


Running Time Analysis:
For the insertion time results, the performance matched our intuition. At low thread counts, spinlocks outperformed mutexes as they avoided context switching and the cost of busy waiting from the few threads does not overwhelm the CPU. The environment used to run this code (docker container with Ubuntu image) has 8 cores. Spinlocks are faster before all 8 cores are utilized because the threads can continuously check the lock without blocking, while mutexes need to exit user space (context-switching). Once thread counts exceed core counts, mutexes begin to close the gap, as sleeping becomes more efficient than spinning when the CPU is already fully utilized.  This would likely become more apparent as more threads are introduced.

For retrieval, the original, no lock, code is slower for several reasons. It is the only one that has to update the lost variable in the retrieval loop, which although is added work the others don’t need, realistically has little effect on performance. The primary slowdown most likely comes from race conditions and attempting to retrieve lost keys. Each bucket is implemented as a linked list and for an entry to be labeled lost, the entire list must be traversed. In other words, lost keys require the program to traverse all n nodes of the list. Traversing the linked list is an O(n) operation and doing so for thousands of entries accumulates significant overhead. Successful lookups are faster on average because the desired key may appear earlier in the list. This leads to an average lookup time of less than n nodes, compared to the average lookup time for lost nodes of exactly n nodes. Across the thousands of lost keys that may appear in the original code, this produces a small but measurable slowdown. The retrieval times between different strategies seems to converge once the 8 cores are used. This is most likely due to the increased overhead of constant context-switching from sleeping threads switching between user and kernel space. This slowdown effect from using many threads is present in all three strategies.





Estimate of Timing Overhead and Explanation:
Using the same method as was used to calculate mutex overhead, we calculated spinlock overhead to be .0015 seconds per thread.

Here is the time to insert for the original code and spinlock based code:

Here is the Average over head for the spinlock based code by thread (and total):


Mutex, Retrieve Parallelization [20 Points]
 
Why We Do Not Need a Lock For Retrieval:
We do not need to lock the table when multiple threads are competing to acquire the lock only because in this specific program we are only updating the lost variable when a key is not found in the table. So, if two threads retrieve the same value at the same time nothing is going to happen in terms of the state of the program. This does not mean that reading is something that does not need to be synchronized. It is only safe depending on the context of what the program does with what is read and in our program it is safe.

What We Changed to Allow Retrieval to be Parallelizable:
Nothing needed to change to allow multiple threads to access the same bucket at the same time without causing a race condition. This is inherently due to the specifics of what the program is doing. The program can have multiple threads accessing the same value at the same time, but only if there is a “key not found” then the lost variable will be incremented otherwise nothing will happen so the program state stays the same. The insertion and retrieval phases are two distinct parts of the program. It is safe to assume that once the program enters the retrieval loop, no buckets will be updated. Thus reading from those buckets at the same time will not cause a race condition. If synchronization were not in place for insertion, guaranteeing no lost keys, then we would need to add synchronization to the retrieval to prevent over-counting lost keys (more than one thread incrementing the lost value for the same key).


Mutex, Insert Parallelization [20 Points]

Describe a Situation in Which Insertions could Happen Safely:
Multiple insertions could happen safely provided they are being inserted into different buckets. To ensure that each bucket is being updated by one thread at any given moment, a lock can be placed on each individual bucket rather than the entire table. This would allow one thread to have atomic access to the bucket it needs to modify while not blocking other threads from acquiring their own atomic access to other buckets. This would safely allow multiple insertions to happen in parallel without synchronization issues.

What We Changed to Allow Retrieval to be Parallelizable:
What changed was that we increased the granularity of the lock to be at a per bucket level since multiple threads modifying different buckets at the same time does not cause a race condition. Concurrency is improved by replacing the single global mutex with individual mutexes for each hashtable bucket. Before this change, every instance of an insertion into the hashtable required acquiring the same individual lock. This forces all threads to act sequentially in the put phase. The updated solution applies a separate lock to each bucket. Threads that are inserting key-value pairs to separate buckets can now do so in parallel. This improves the performance of the code without risking synchronization errors as race conditions only occur when threads try to insert into the same bucket.

