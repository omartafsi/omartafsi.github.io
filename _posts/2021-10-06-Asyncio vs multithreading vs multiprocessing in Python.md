---
published: true
---


## Asyncio vs multithreading vs multiprocessing in Python

Let's say you have the task of crawling a list of 500 URLs, one way to do it is to run a for loop where you call 1 URL at a time on every iteration using the following code:

```python

for url in url_list:
	requests.get(url)
```


As you may have guessed, this is not the most optimal way to do it, this is because our program will be blocked on every iteration waiting to receive the HTTP response of the requests.

What if during the time our program is blocked waiting for the HTTP response, we have it process something else, for instance, send another request. This can be done via **Asyncio** and **multithreading**.

Or, because the HTTP request operation is not complex, we code our program in a way that the requests are spread across different cores of our processor instead of having all the cores making one request at a time. This can be achieved with **multiprocessing**.

Before going into comparison, let's explicitly define some concepts that are necessary to understand the differences between Async, multithreading, and multiprocessing.


## Concurrency vs parallelism:

### Concurrency:  

In python, we can define concurrency as the ability to optimally execute multiple threads in a way that reduces blockage time, a thread here referring to a set of instructions running in order. It is based on the Global Interpreter Lock (GIL) mechanism insuring that only one thread is in the execution state at any point in time, This is because CPython (the reference implementation of python) is not thread-safe. This results in never achieving true simultaneous concurrency using multithreading in Python.

To make an analogy with our example, let's say we initialize three tasks [A, B, C] responsible for requesting URLs [a.com, b.com, c.com] Respectively. Concurrency simply means that when we start task A and get blocked waiting for the a.com server to reply, we switch the execution context to task B. And when B itself is blocked waiting for b.com response, we either switch to task C or go back to A.

### Parallelism: 

Parralellism is when we seperattly allocate cores, memory, and resources for each of our processes. This enables true simultaneous execution. Again to go back to our analogy, in the case of parallelism, we will have three process A, B and C where each will get to request its own URL on a separate core of the processor.

## how does multithreading work in Python?

Multithreading is a form of concurrency, it is based on the concept of [pre-emptive multitasking](https://en.wikipedia.org/wiki/Preemption_%28computing%29#Preemptive_multitasking) which is basically the operating system handling the execution of multiple [threads](https://en.wikipedia.org/wiki/Thread_(computing)) and deciding under the hood which one gets to be executed.

Here is an implementation of a bot that scraps a list of given URLs using multithreading, we're using the ThreadPoolExecutor class from the python concurrent library:

```python
from concurrent.futures import ThreadPoolExecutor, as_completed, ProcessPoolExecutor
import requests

def multithreading_scraper(urls):
    threads = []
    with ThreadPoolExecutor() as executor:
        for url in urls:
            threads.append(executor.submit(requests.get, url))
            try:
                for task in as_completed(threads):
                    try:
                        resp = task.result()
                        # ADD RESPONSE MANIPULATION CODE HERE
                    except ssl.SSLCertVerificationError:
                        print("SSL verification error, ignore...")
            except Exception as e:
                print(f'uncaught exception :{e}')
```

#### Advantages:
* works well for I/O Bound (threads where the main cause blockage is waiting for input/output)  
* Easy to use: the task management is handled by the OS.
* Scalability: in practice, the threads are distributed across multiple cores.

#### Drawbacks: 
* deadlocking: two threads waiting for each other to finish to be able to run
* race condition: two threads access the same resource at the same time.

## how does Asyncio work in Python?

Because our goal here is to understand the overall dynamic of the process, we will refrain from going into details and instead try to cover this method in an overly simplified way to understand the intuition behind it.

The entire concept is based on something called the event loop, this is an actual loop that keeps iterating and decides which task gets to be run next based on its state. To do this it keeps track of two lists based on task state, one for tasks that are in the **Waiting state**, meaning tasks that are waiting for some external resource which is in our case an HTTP response, and a list of **Ready state** tasks that are ready to be executed. The list is in FIFO mode so the first task in the ready list is the one that gets executed first. In practice, there are more than two states but we won't go into detail.

When the task has finished executing, it gets placed in one of the lists depending on his state. The event loop verifies the state of tasks awaiting and proceeds to add them to the ready list in case their awaiting has been completed. After this, it goes back to the initial task of choosing the next ready task to execute.

Now you might be asking, what makes it different from multithreading, the actual difference is that the event loop actually **awaits** for tasks to send a "finished" signal before it moves to the next **ready** task and never interrupts them. This finished signal is what we call a **promise**, fortunately with the async/await syntax we don't need to manage these signals.

This is an overly simplified explanation of the Asyncio concept, here is an implementation of a bot that scraps a list of given URLs this time, using the Async optimized python library *aiohttp*:

```python
import aiohttp
import asyncio
from asgiref import sync

def async_aiohttp_scraper(urls):

    async def get_all(urls):
        async with aiohttp.ClientSession() as session:
            async def fetch(url):
                try:
                    async with session.get(url) as response:
                    	resp = await response.text()
                        # ADD RESPONSE MANIPULATION CODE HERE
                        return resp

                except Exception as e:
                    print(f'uncaught exception :{e}')

            return await asyncio.gather(*[
                fetch(url) for url in urls
            ])

    # call get_all as a sync function to be used in a sync context
    return sync.async_to_sync(get_all)(urls)
```
#### Advantages:
* works well with I/O Bound tasks.
* no deadlocks or race conditions

#### Drawbacks: 
* can be tricky in implementing at first


## how does multiprocessing work?

In Python, Multiprocessing in the standard library that was designed to break down the barrier that the GIL mechanism causes and by running your code using parallelism across multiple CPUs. At a high level, it does this by creating a new instance of the Python interpreter to run on each CPU and then farming out part of your program to run on it.

As you can imagine, bringing up a separate Python interpreter is not as fast as starting a new thread in the current Python interpreter. It’s a heavyweight operation and comes with some restrictions and difficulties, but for the correct problem, it can make a huge difference.

here is an implementation of a bot that scraps a list of given URLs, this time, using multiprocessing. As it can be seen the implementation resembles the one featured in multithreading:

```python
from concurrent.futures import , as_completed, ProcessPoolExecutor

def multiprocessing_scraper(urls):
    tasks = []
    with ProcessPoolExecutor() as executor:
        for url in urls:
            tasks.append(executor.submit(requests.get, url))
            try:
                for task in as_completed(tasks):
                    try:
                        task.result()
                    except Exception as e:
                        print(f" Error while executing : {e}")
            except Exception as e:
                print(f'uncaught exception :{e}')
 ```               

#### Advantages: 
* Easy to implement
* the intuition is easy to grasp

#### Drawbacks: 
* IPC (Inter-Process Communication) can be more involved with more overhead required.
It has a larger memory footprint.



## Is it possible to combine Asyncio with multiprocessing?

Yes, it is, the idea is to implement all the Asyncio logic on multiple cores, here is an example where we are making requests on a list of URLs with a combination of parallelism and Asyncio:

```python

from concurrent.futures import  as_completed, ProcessPoolExecutor
import aiohttp
import asyncio
from time import time
from asgiref import sync



def masync_multiprocessing_scraper(urls, url_per_asynf):
    tasks = []
    with ProcessPoolExecutor() as executor:
        for i in range(0, len(urls), url_per_asynf):
            list_urls_to_scrap = urls[i: i + url_per_asynf]
            tasks.append(executor.submit(async_aiohttp_scraper, list_urls_to_scrap))
            try:
                for task in as_completed(tasks):
                    try:
                        task.result()
                    except Exception as e:
                        print(f" Error while executing : {e}")
            except Exception as e:
                print(f'uncaught exception :{e}')


def async_aiohttp_scraper(urls):

    async def get_all(urls):
        async with aiohttp.ClientSession() as session:
            async def fetch(url):
                try:
                    async with session.get(url) as response:
                        return await response.text()

                except Exception as e:
                    print(f'uncaught exception :{e}')

            return await asyncio.gather(*[
                fetch(url) for url in urls
            ])

    # call get_all as a sync function to be used in a sync context
    return sync.async_to_sync(get_all)(urls)
```   



## When should you use multiprocessing vs Asyncio or threading?

#### The rule of thumb is: 

* If the task spends time waiting for an input/output(**I/O bound**) in our case the server responds with **fast** I/O and a **limited** number of connections:  **Multithreading**
* If the task pends time waiting for an input/output(**I/O bound**), with **slow** I/O and **many** connections possible: **Asyncio**
* If the task is computationally complex requiring big CPU time (**CPU bound**): **multiprocessing**


## Speed testing multithreading vs async vs multiprocessing?


I have run the four scrapers featured above on a list of the most visited domains on the internet, I ran two tests, one where the list only contains the top 10 and another containing 500, the complete test code is in the following [python file](https://www.github.com):

For the 10 URLS list, the results are the following:

```python

Function 'multithreading_scraper' executed in 5.1004s
Function 'multiprocessing_scraper' executed in 5.4535s
Function 'async_aiohttp_scraper' executed in 1.6210s
Function 'async_multiprocessing_scraper' executed in 4.3092s
```
for the async multiprocessing we chose 1 as the number of URLs per core

For the 500 URLs list, the results are the following:
```python
Function 'multithreading_scraper' executed in 44.6328s
Function 'multiprocessing_scraper' executed in 443.7407s
Function 'async_aiohttp_scraper' executed in 10.3853s
Function 'async_multiprocessing_scraper' executed in 1607.4033s
```
for the async multiprocessing we chose 42 as the number of URLs per core.

As we can see, the Async method wins the race by being the most optimal in making the HTTP GET requests. An important point to clarify is that the speed also varies with regards to the library used.
