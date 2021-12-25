# C++ Lockless MP/MC Queue
A simple Multi-Producer, Multi-Consumer Queue Type. Written in Cpp using
just atomics with memory barriers to keep everything ordered appropriately.

Inspiration is taken from the LMAX Disruptor. If you're unfamiliar with lockless
programming and/or the usage of atomics with memory barriers, then the Disruptor
is a great place to start learning.

## How it Works

#### Bounded Ring Buffer
The queue uses a bounded ring buffer to store the data. In order to avoid the use
of the modulo operator for calculating the index, the queue must have a size which is equal to a power of two.
This allows us to use a bitmask instead, which is more efficient.
#### Two Cursors
A "Producer" curosr, and a "Consumer" cursor are used to indicate to the next incoming
producer/consumer which index on the ring buffer they will be accessing. When a producer/consumer is attempting access,
they first increment their respective cursor, then access the buffer. By doing the incrementation first then they will be effectively
claiming that index, ensuring that only they are accessing that particular index on the buffer. This way each index in the buffer is guarded against multiple access attempts at the same time.
Since multiple accessors may try  to increment the cursor at the same time, they first
enter a busy spin and via a CAS operation they verify that they were
successful in incrementing the cursor, and not someone else.
#### TSequentialContainer Type
The queue relies on a Sequential Container which uses memory barriers to synchronize
access to an atomic variable of template type "T".
Both of the cursors that the buffer uses are of type FSequentialInteger, which derives from the 
TSequentialContainer class. This is the type that all access to the ring buffer is protected through.


## Usage
First create a new queue as follows:
```c++
#include "MPMCQueue.h"

TMPMCQueue<int> MyQueue;
// Or with a manually specified size (default is 1024),
// size is automatically rounded up to the nearest power of two!!!
TMPMCQueue<int, 2048> MyQueue;
```
Then to add a new element:
```c++
MyQueue.Enqueue(5);
```
To claim an element:
```c++
int MyInteger = 0;
MyQueue.Dequeue(MyInteger);
```
The claimed result will be stored in "MyInteger", in this case.

## Example Benchmark code
The following code will create THREAD_COUNT number of threads, which will
each either produce an item and add it to the queue, or consume an item from the queue
TIMES_TO_CYCLE number of times. The main thread waits for all threads to finish
before the program terminates. A simple atomic counter is used to track how many
threads have finished by having each threads increment the counter when they're
done, so we just wait until the counter == THREAD_COUNT, then the program terminates.
```c++
#include "MPMCQueue.h"

#include <thread>

/** Define how many times to Produce/Consume an element. */
#define TIMES_TO_CYCLE  10000000
#define THREAD_COUNT    32

static TMPMCQueue<int, 419444> MyQueue;
static FSequentialInteger ThreadsCompleteCount;

int main()
{
    // Create a Producer & Consumer per iteration
    for(int64 i = 0; i < (THREAD_COUNT / 2); ++i)
    {
        // Create a Producer
        std::thread([]()
        {
            const int NumberToAddLoads = 100;
            for(int64 j = 0; j < TIMES_TO_CYCLE; j++)
            {
                MyQueue.Enqueue(NumberToAddLoads);
            }
            
            ThreadsCompleteCount.Increment();
        }).detach();
        
        // Create a Consumer
        std::thread([]()
        {
            int NumberToHoldLoads = 0;
            for(int64 j = 0; j < TIMES_TO_CYCLE; ++j)
            {
               MyQueue.Dequeue(NumberToHoldLoads);
            }
                    
            ThreadsCompleteCount.Increment();
        }).detach();
    }
    
    /** Spin yield until the four threads are complete */
    while(ThreadsCompleteCount.GetRelaxed() < THREAD_COUNT)
    { }
    
    system("pause");
    return 0;
}
```
