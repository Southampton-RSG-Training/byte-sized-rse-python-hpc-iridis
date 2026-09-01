---
title: "HPC with Python"
teaching: 0 # teaching time in minutes
exercises: 0 # exercise time in minutes
---

::: questions

- What is the difference between paralellism and concurrency?
- What are the options available for concurrently running tasks using the Python programming language?
- What are the strengths and weaknesses of each approach?
- What challenges does the Python language runtime itself present when attempting to run Python code concurrently?
- How can I effectively use Python in an HPC context?
- How can I use MPI to distribute Python tasks across multiple nodes in an HPC system?

:::

::: objectives

- Discuss the different approaches to concurrency in Python: asyncio, threading, and multi-process.
- Understand the Global Interpreter Lock (GIL) and its implications for threaded code in Python.
- Use the `concurrent.futures` library to write code that is easily adapted between different concurrency approaches.
- Run concurrent Python code locally and on a single HPC node using SLURM.
- Run concurrent Python code on multiple HPC nodes using SLURM, MPI and `mpi4py.futures`.

:::

## Concurrency and Parallelism

Concurrency and parallelism in programming are ways to get a piece of software to do multiple things at the same time. The two terms have subtly different meanings: parallelism is performing multiple tasks at the same time (for example, two processes working on different parts of a data set at the same time), while concurrency is the ability for a computation to pause tasks and resume them at a later time (for example, pausing a file download task while waiting for a server to respond and doing another computation instead).  You can have software which is concurrent but not parallel, and vice-versa.

When your goal is wanting to make your code go faster, both concurrency and parallelism can be important, but typically come into play when you have different problems with speed: concurrency is important when you are *waiting* for something to happen, and parallelism is important when you want to *distribute* the work across multiple processing units.

Both play a role in high-performance computing: for example when updating a deep neural network, you might *distribute* a batch across multiple GPUs to perform the forward computation, but you need to *wait* for all the results to be computed before you can do back propagation.

However, in many cases the computations that you run into are ones that do not have a lot of waiting or communication between tasks. Indeed, often it is simply the need to perform the same computation on a bunch of different inputs, with no linking between each computation. This sort of trivially parallelisable code is sometimes called "embarrassingly parallel."

::: callout

## Painting a room

Parallel computing means dividing a job into tasks that can run at the same time. Imagine painting four walls in a room. The problem is painting the room. The tasks are painting each wall. The tasks are independent, you don't need to finish one wall before starting another.

If there is only one painter, they must work on one wall at a time. With one painter, painting the walls is *sequential*, because they paint one wall at the time. But since each wall is independent, the painter can switch between painting them in any order. This is *concurrent* work, where they are making progress on multiple walls over time, but not simultaneously.

With two or more painters, walls can be painted at the same time. This is *parallel* work, because the painters are making painting the room by painting multiple walls at the same time.

In this analogy, the painters represent CPU cores. The number of cores limits how many tasks can run in parallel. Even if there are many tasks, only as many can progress simultaneously as there are cores. Managing many tasks with fewer cores is called concurrency.

:::

## Concurrency in Python

It's likely that you'll be writing a lot of your code in Python. Python has a number of different ways of accessing concurrency and parallelism with various advantages and disadvantages, not all of which are appropriate for HPC tasks, so it's important to have an understanding of each.

### Threading and the Global Interpreter Lock (GIL)

A thread is an operating-system level subprocess for computation which can be paused and restarted without needing direct intervention by the process which is running. It is a system for concurrency, but most operating systems will also allow different threads to run on different cores at the same time, so it is also a system which allows parallelism if the hardware can support it.  Since threads run inside a single process, they share the memory allocated to that process.  This means that threads within a program need to take care that they do not corrupt each other's data. There are a number of mechanisms that have been developed over the years to deal with this problem, and a common mechanism is a *lock*: a flag which can be set by a thread to prevent access to a resource by other threads.

Python provides access to operating-system threads. However a fundamental feature of Python for the last 30 years, ever since threading support was added, has been the so-called "Global Interpreter Lock" or "GIL".  This is a lock which prevents the internal state of the Python interpreter from being modified by two threads at the same time, and which the Python interpreter acquires before it executes a Python virtual machine instruction, and releases after the bytecode is finished executing.  This global lock simplifies the Python interpreter's code and experimentally has been shown to generally be faster than the alternative of a system of fine-grained locks on individual pieces of data for single-threaded code.

But it means that pure-Python threaded code is concurrent but *not* parallel. In practical terms what this means is that threading of pure Python code often doesn't speed things up, and may in fact slow things down: you can only execute one bytecode at a time across all the threads, and switching between threads has some performance cost.

*However* if Python calls out to C code, and that C code isn't going to interact with the Python interpreter, it can signal that by temporarily releasing the GIL at the C level, allowing other threads to do work until it re-acquires the lock.  This is commonly done in the core Python I/O libraries: file and network access typically releases the GIL while performing buffering or waiting for a server to respond.

Libraries like NumPy and SciPy which are built on top of HPC numerical packages like BLAS and FFTPACK can and do release the GIL before performing long-running computations on NumPy arrays which don't require any interaction with the Python interpreter. It is common and effective to use threads to work in parallel across pieces of a large NumPy array and get significant real-world speed-ups, depending on the number of CPU cores available.

The same is true of PyTorch and other GPU libraries: they will usually release the GIL before making CUDA calls.  This means that multiple CPU threads can be driving multiple interactions with the available GPU(s) in parallel.

There are some limits on naive threading approaches even when working in low-level languages like C: for example memory allocation can often be a bottleneck unless code is carefully written to minimize it, for example by pre-allocating memory for each thread to use.

### Multiprocessing

One way to overcome the GIL is to have multiple Python interpreters running, each in their own process, and use operating-system shared memory to communicate where needed.  The Python standard library `multiprocessing` module provides a relatively easy way to do this sort of computation. It provides an API very similar to the `threading` module that allows you to run tasks in multiple processes instead of multiple threads, so it is easy to convert code between the two.

However processes are very heavy-weight compared to threads, in particular there is a lot of overhead in starting a new process with a new Python interpreter. Additionally, unlike threads which can share state freely, in multiprocessing you need to carefully allocate which objects are shared between the processes, and they can only be certain fundamental data types (which fortunately includes views of array data). If you don't, you can end up with multiple copies of the same object, one in each process, which can be a problem when you are dealing with multi-gigabyte arrays or sets of neural network weights.

### Async Python

Python provides language support for cooperative concurrency via the `async` and `await` keywords and the `asyncio` standard library.  These provide a concurrency model where each task is passed control, does what it needs to do, and relinquishes control when it is done. Each time a task is given control, it resumes execution where it left off.  However only one task is ever running at one time, so async code is concurrent, but not parallel.

This simplifies the programming of these systems, because the fact that there is only one thing running at a time reduces the likelihood of race conditions and the need to acquire locks to gain control of a resource.  This is great for tasks which spend most of their time waiting for other things to do work, particularly I/O. Async is used heavily in Python networking software for this reason: most of the time each task is waiting for a remote client to respond.  Asyncio is very light-weight compared to threads so it is possible to have many more tasks active at a given time using asyncio compared to threading approaches.

However async-based programs are a poor choice for HPC, because most tasks are going to be working almost all the time: they are rarely waiting for other things to happen.  Using Python's async/await system for HPC effectively limits you to one core, and will likely be slower than if you simply performed the computation sequentially. Async code tends to really shine when doing things which require a lot of waiting, such as I/O or GUI code.

::: spoiler

### The Future

Two new features in recent and upcoming versions of Python promise some improvements to the way concurrency works in Python:

#### Multiple interpreters

In Python 3.14 the `concurrent.interpreters` module was introduced which allows multiple Python interpreters to exist within a single process. When running in separate threads, different interpreters can execute truly in parallel as each has its own GIL. However they are limited in the way that they share memory in a similar way to Python's multiprocessing module. Nevertheless it holds the promise of being a more efficient replacement for multiprocessing, since it avoids the overhead of multiple processes.  Unfortunately some C extension modules need to be adapted with the possibility that they might be used by multiple interpreters.  At the time of writing a number of important packages, such as NumPy, Pillow and PyTorch, have not been adapted.

#### Free threading

In Python 3.13 support for building a ["free threaded"](https://py-free-threading.github.io/) Python interpreter with no GIL was introduced with the intent that it will eventually become the default interpreter. The free-threaded interpreter is slightly slower when running sequential code, but significantly faster when running code that would have been slowed by the GIL. However C extension modules (like NumPy, PyTorch, and many others) need modification to their C code to be safe to use with the free threaded builds of Python. At the time of writing, many of these packages have *experimental* support for free threading, but because of the need for custom builds of the interpreter it is probably too early to be using free-threaded Python for production work. In particular free-threaded Python is not available on Iridis.

:::

::: callout

## Painting a room with Python

To understand the differences between the different approaches to concurrency in Python, we can take our analogy of painting a room, and stretch it a little:

#### Threaded Python

Threaded Python is a bit like having several painters, but only one brush. After each stroke of paint, the painter puts the brush down, and then any of the painters can pick it up. But only one painter is ever painting at a time.

C code which releases the GIL is like each painter having a paint roller in addition to the shared brush. Each painter can paint a lot of their wall quickly, and the painters can use their rollers to paint independently at the same time, but they still need the shared brush to do touch-up work.

#### Multiprocess Python

This is a bit like having several painters who can paint their walls independently, but there isn't much sharing: they all have their own paint tin and ladders and drop cloths and all the other equipment that they need to do the task, and they all need to fit it into the room where they are working.

#### Async Python

This is like having several painters, but only one brush. Instead of putting the brush down after each stroke, each painter paints as much as they want before putting the brush down. A painter could potentially pain their entire wall before letting go of the brush.

:::

## Practical Advice

On its own the standard version of Python (which written in C and so is sometimes called CPython) is a poor language for HPC: it is slow and doesn't support parallelism very well.  What makes Python such a popular language for HPC is that it is easy to extend with C, C++ or Fortran code allowing you to do your computation in those fast languages and in parallel if needed by releasing the GIL.

But more importantly there is an extensive, high-quality collection of extension libraries like Pillow (which wraps various C image format libraries), NumPy (which wraps linear algebra libraries like BLAS), SciPy (which wraps many C and Fortran computational libraries), and PyTorch (which wraps GPU code) that interoperate really well.

This means Python is an excellent "glue" language: effectively sticking together unrelated libraries from different domains which would be complex to do in C or Fortran.  For example, you can turn a Pillow `Image` into a NumPy `ndarray` and then into a PyTorch `Tensor` very efficiently. This means that carefully-written code can be extremely fast even when dealing with very large amounts of data.

If you find yourself writing complex numerical computations in pure Python, you should be looking to vectorize the computations using libraries like NumPy, Pandas, and PyTorch; or if the computations are not easily vectorised, use tools like Cython or Numba to convert working Python code to faster compiled code.

You should always pursue these sorts of optimizations before trying to run code on an HPC cluster.

### Threading and Multiprocessing

When your work is highly numerical code that releases the GIL (such as the linear algebra code in NumPy), threading is often effective at speeding things up. This is particularly the case when you are dealing with very large arrays in memory, as threaded code shares the data. Multiprocessing code does not, so you may end up with a copy of the data in each process which can cause problems.

The Python `threading` and `multiprocessing` libraries have a very similar interface, so it not hard to convert from one to the other. In particular, the `concurrent.futures` library we will discuss later provides a nice higher-level interface that makes it very easy to switch approaches.

### MPI

When work needs to be spread over multiple HPC nodes, Python has the MPI4Py library as a wrapper around the MPI protocol.  MPI4Py allows passing messages which contain any Python object which can be serialized using the standard library "pickle" protocol and is quite flexible as a result.  It has more efficient modes which use NumPy's array data structures to perform lower-level sharing of data

### PyTorch and Accelerate

When it comes to working with PyTorch for concurrent GPU work, `torch.distributed` is available with various backends depending on the hardware that you have.  However `torch.distributed` is moderately low-level: care needs tobe taken to indicate places where parallelised operations have to synchronise before proceeding on to the next step.

The `accelerate` library is a wrapper around `torch.distributed` which allows you to more easily adapt GPU code that works synchronously to be distributed across multiple threads, processes, or nodes as needed.

::: spoiler

### OpenMP

Another common way of parallelising HPC code is the OpenMP library. This doesn't make sense for pure Python code, but C extensions can make use of it and tools like Numba and Cython offer built-in support for it, which can potentially avoid the need for threading in Python.  Additionally NumPy is starting to introduce OpenMP support as a compile-time option for some operations (you will need to build NumPy from source to use this).

:::

## Example: Data augmentation

When doing deep learning with image data, a common approach to expanding the available training data is to perform data augmentation by randomly manipulating the images in ways that don't significantly change the subject, such as adjusting contrast and brightness, rotating and scaling, and other similar transforms.  This can be used both to make the model more robust against variations in size, position and image quality, but can also be used to even out the number of observations in classes of a classification data set.

PyTorch's `torchvision` library has a number of functions to perform these transforms, in particular there is `RandAugment` which takes an image and performs a random selection of transforms chosen from a predetermined set.

We're going to take an example image data set and apply this to generate an additional 10 images from each original image.  The basic code is very simple. We have a function that reads an image, generates some randomly augmented images from it, and then saves the results to a directory:

``` python
def augment_image(
    file_path,
    output_dir=DEFAULT_OUTPUT,
    n_augments=10,
    augmenter=DEFAULT_AUGMENTER,
):
    stem = file_path.stem
    file_dir = output_dir / stem
    file_dir.mkdir(exist_ok=True)

    with Image.open(file_path) as image:
        for i in range(n_augments):
            augmented = augmenter(image)
            augmented.save(file_dir / (stem + f"_{i:03d}.jpg"))
```

Then there is a main function which finds all the image files in the input directory and calls the `augment_image` function for all of them.

``` python
@click.command()
@click.argument(
    "input_dir",
    default=Path("flowers-102/jpg"),
    type=click.Path(dir_okay=True, file_okay=False, exists=True),
)
def main(input_dir):
    from tqdm import tqdm
    from time import perf_counter

    t = perf_counter()
    DEFAULT_OUTPUT.mkdir(exist_ok=True)
    paths = sorted(Path(input_dir).glob("*.jpg"))
    for path in tqdm(paths):
        augment_image(path)
    sys.stderr.write(f"Time taken: {perf_counter() - t:.2f}s\n")
```

This is a somewhat interesting problem since it's trivially parallelisable - every image can be dealt with independently of every other image, so if we had a big enough machine we could potentially process all the images simultaneously - but there is both a lot of computation and a lot of reading and writing to/from the filesystem.

::: spoiler

#### `click` and `tqdm`

This script uses a couple of third-party libraries that make the script more ergonomic to use.

The `click` library makes command-line argument processing much easier and more straightforward. In this case we provide a single command-line argument `input_dir` which is the input directory path and which gets passed into the `main` function. The `click` library handles all the parsing and validation of arguments (the path must be a directory which exists, and we get a `Path` object rather than a string).

The `tqdm` is a library that displays simple progress bars in command-line programs. While it has a lot of options and different ways to use it, the simplest way is to wrap the iterable of a `for` loop in a call to `tqdm` and it will handle everything else.

:::

::: prereq

### Local Install and Dataset Download

We need to create a Python environment with our dependencies.

``` bash
$ python3.14 -m venv venv
$ source venv/bin/activate
(venv)$ cd data_augmentation
(venv)$ pip install -U pip
(venv)$ pip install --requirements-from-script image_augmentation.py
```

We also need a local copy of the Oxford Flower Dataset. There is a script that will perform the installation:
``` bash
(venv)$ python download_images.py
```

This should create a `flowers-102/jpg` directory containing the images we will be working with. There are about 8000 images of flowers.

:::

::: discussion

We can now run the script on the flower dataset:
``` bash
(venv)$ python augment_images.py flowers-102/jpg
```

How long does it take to run on your machine?

:::

### Profiling Performance

On an MacBook Pro M5 it takes about 130 seconds to run the code as-is. But the CPU is far from fully utilized, so we can potentially do this faster.

The first step when parallelizing is to determine whether the CPU is the bottleneck or whether storage is.  We can do this in Python by running using the profiler:
``` bash
python -m cProfile -s cumulative augment_images.py flowers-102/jpg | more
```
This allows you to see which functions are using the most time
This can be inspected using the standard library `pstats` module, or more elegantly with tools like SnakeViz.

```
Time taken: 640.44s
         93241445 function calls (92849947 primitive calls) in 653.485 seconds

   Ordered by: cumulative time

   ncalls  tottime  percall  cumtime  percall filename:lineno(function)
    ...
    90129  325.345    0.004  325.345    0.004 {built-in method _io.open}
    ...
    81890    2.953    0.000  190.893    0.002 _auto_augment.py:425(forward)
    ...
    81890  101.019    0.001  101.019    0.001 {method 'encode_to_file' of 'ImagingEncoder' objects}
```

Inspection of the results tells us that about 1/2 of the time is being spent manipulating and encoding the images, and the remaining 1/2 of the time is spent on storage I/O.  Slower machines will likely show a split with more time spent on

::: spoiler

### Profiling Python Code

Profiling the performance of Python code is a deep enough topic to deserve its own course.

Python has two profilers as part of the standard Python library: `profile` and `cProfile`. These have the same interface and behave the same way, but `cProfile` is written as a C extension model and so is faster with less overhead. You should always use `cProfile` when available.

The Python profilers work by adding a hook that gets executed at the start of every function call, which allows them to gather timing information, but at the cost of adding significant runtime overhead: code runs 2-5 times slower when being profiled.

The third-party `line-profiler` (sometimes called "kernprof" after its original author) works similarly, but tracks individual lines executed within marked functions.

There are other third-party profilers which estimate time spent in each function by sampling: periodically checking the state of the Python interpreter to work out which function is executing at that moment, and then estimating time spent by the proportion of the checks which occur in a particular function. This is lighter weight, but potentially less accurate and can't track information about things like the number of times each function is called.

In Python 3.15, the standard library is getting a new sampling profiler `profiling.sampling`, and cProfile is being re-named as `profiling.tracing` but can still be used as `cProfile`.

:::

## Parallel Code with `concurrent.futures`

When you have simple parallelism where each call is independent, Python provides the `concurrent.futures` module.  This module is designed to make it easy to write "scatter-gather" and "map-reduce" style parallel code

This module provides an "executor" object that controls a pool of workers and allows you to submit functions and their arguments to be dispatched in parallel to a worker. When you submit the function, you are given a "future" object which gives you the state of the submitted work (waiting, executing, completed, and any output or errors).

Concurrent futures provides different executors for different versions of parallelism: threaded, multiprocess, and in recent versions of Python, multiple interpreter. The code for these is almost identical, allowing you to experiment with different approaches to see which works best for your particular code.

For example, the following code calls the `augment_image` function for each image using two worker threads (plus a main thread that)

``` python
from concurent.futures import ThreadPoolExecutor

n_workers = 2
paths = sorted(Path(input_dir).glob("*.jpg"))

with ThreadPoolExecutor(max_workers=n_workers) as executor:
    for result in executor.map(augment_image, paths, as_completed=True):
        # if the function returns anything we could do something with the
        # result; but augment_image doesn't return anything so we do nothing
        pass
```

In the threaded case, things are promising when we use a worker pool with 2 workers, but as we increase the size of the pool (up to a maximum of the number of cores in the CPU) the efficiency drops away fairly quickly and there is basically no improvement from going from 4 workers up to 10. Some of this is hardware-related—the particular computer these numbers were generated on has 4 performance cores—but it is also related to the Python GIL becoming the bottleneck.

| n_workers |  time  | efficiency |
| :-------- | ------:| ---------: |
|  1        | 130.00 |      1.00  |
|  2        |  68.94 |      0.94  |
|  3        |  52.71 |      0.82  |
|  4        |  43.46 |      0.75  |
|  5        |  42.10 |      0.62  |
|  6        |  41.57 |      0.52  |
|  7        |  38.89 |      0.48  |
|  8        |  39.22 |      0.41  |
|  9        |  38.77 |      0.37  |
| 10        |  40.30 |      0.32  |

::: challenge

### Running the Threaded Code

To do a fair comparison, we need to remove the output, since overwriting this many files is significantly slower than writing new files. This may take a few seconds:

``` bash
(venv)$ rm -rf output/
```

Run the threaded version of the script on your machine:

``` bash
(venv)$ python augment_images_threading.py flowers-102/jpg
```

How long does it take to run the code on your machine?

:::

If we make the change to replace the `ThreadPoolExecutor` with a `ProcessPoolExecutor` we can run again, and we see that we get a much better efficiency as we increase the number of workers:

``` python
from concurent.futures import ProcessPoolExecutor

n_workers = 2
paths = sorted(Path(input_dir).glob("*.jpg"))

with ProcessPoolExecutor(max_workers=n_workers) as executor:
    for result in executor.map(augment_image, paths, as_completed=True):
        # if the function returns anything we could do something with the
        # result; but augment_image doesn't return anything so we do nothing
        pass
```

| n_workers | time  | efficiency |
| :-------- | ----: | ---------: |
|  2        | 68.90 |      0.94  |
|  3        | 48.75 |      0.89  |
|  4        | 40.88 |      0.80  |
|  5        | 37.37 |      0.70  |
|  6        | 35.73 |      0.61  |
|  7        | 33.87 |      0.55  |
|  8        | 32.95 |      0.49  |
|  9        | 32.46 |      0.44  |
| 10        | 32.58 |      0.40  |


While we some decay in efficiency from additional workers, and further, we are still seeing improvements as we increase the number of workers up to the point where we have more workers than cores (including the main executor): running with 12 workers was slightly slower than running with 8, for example).

The result of this experiment is that running with even more cores on a larger machine would likely see continued improvement, but there will eventually be a bottleneck due to disk I/O bandwidth.

::: challenge

### Running the Multiprocess Code

Remove the output directory once again:

``` bash
(venv)$ rm -rf output/
```

Now run the multiprocess version of the script on your machine:

``` bash
(venv)$ python augment_images_threading.py flowers-102/jpg
```

How long does it take to run the code on your machine?

:::

::: spoiler

### Effort, Time and Flow

In this toy example we spend a fair bit of effort to reduce the run time from 4 minutes to about 40 seconds: we had to profile and run multiple experiments to see what would work. Obviously this would make no sense if this was the only time we needed to do this particular task.

However if you need to do this repeatedly with different datasets, or if this were an experiment on a smaller part of a larger dataset, or if you needed to generate more augmented data, this would likely save time in the long run.

When considering work on speeding up code you should keep in mind that there are qualitative differences based on how long things take.  In particular maintaining [*flow* state](https://en.wikipedia.org/wiki/Flow_(psychology)) can be very important in the context of software development.

For example a 2-times speed-up from 20 minutes to 10 is good, but it is still very interruptive if you are waiting on the results before you can continue your work: you will have lost "flow" because you have moved on to other tasks and will need to spend time mentally getting back into what you were working on which can take 20 minutes or more.  Effectively you have gone from a 40 minute interruption to work to a 30 minute interruption to work, an improvement, but not as dramatic as a 2 times speedup.

However, if the speed-up were from 10 minutes to 5 or less, you've now reached a point where you might go and get a cup of tea while you wait, but you can keep your mind on task with a little effort, so you've taken a 30 minute interruption down to only 5.

:::

## Adapting for Iridis

We've determined that this is a task that (a) distributes well with multiprocessing, and (b) doesn't involve GPU work, so Iridis 6 would be an appropriate cluster to run this on as a distributed workflow.  Because there is no communication need between tasks, it would gain both from running on a machine with more cores and for being distributed over more nodes.  However the second of these will require some re-thinking of the way the code distributes tasks.

::: prereq

### Installing on Iridis

You will need to log-in to Iridis using your credentials, something like (replacing `<username>` with your username, and depending on how you set up your `ssh` keys, it may have an additional `-i` option):
``` bash
$ ssh <username>@iridis6.soton.ac.uk
```

Once you've logged in, you'll need to download the git repo with:
``` bash
[login6003 ~]$ git clone https://github.com/Southampton-RSG-Training/byte-sized-rse-python-hpc-example.git
```

Since we're using python, we need to activate the python module:
``` bash
[login6003 ~]$ module load python
```
and then change directory to the source directory, and repeat the set-up for the python environment:
``` bash
[login6003 ~]$ cd augment_images
[login6003 augment_images]$ python3.14 -m venv venv
[login6003 augment_images]$ source venv/bin/activate
[login6003 augment_images] (venv)$ cd data_augmentation
[login6003 augment_images] (venv)$ pip install -U pip
[login6003 augment_images] (venv)$ pip install --requirements-from-script augment_images.py
```
and download the data set:
``` bash
[login6003 augment_images] (venv)$ python download_images.py
```

That puts us in a position to run the code using Iridis. In fact, for a small job like our example, running our efficient multi-process script on the login node, which should take less than a minute, would be acceptable.

However, we want to learn how to use the full power of Iridis.

:::

### Running with Multiple Cores

We don't need a lot of memory for our script, so the standard Iridis 6 nodes are a good fit. Each of these have 192 total cores, so if we run on one node we could potentially have 192 simultaneous workers.  In this case we can use our run_multiprocessing script unmodified, but call it from a batch script which looks like:

``` bash
#!/bin/sh

#SBATCH --job-name=python-multiprocessing
#SBATCH --partition=batch
#SBATCH --time=00:05:00     # Maximum run time
#SBATCH --nodes=1           # Run on just 1 node
#SBATCH --ntasks=1          # Run one instance of the script
#SBATCH --cpus-per-task=64  # Use 64 multiprocessing Python workers/cores

# Optional: print useful job info
echo "Running on host: $(hostname)"
echo "Job started at: $(date)"
echo "SLURM job ID: $SLURM_JOB_ID"
echo "Number of CPUs: $SLURM_CPUS_PER_TASK"

# Load required modules
module purge
module load python

# Move to job directory
cd $SLURM_SUBMIT_DIR

# activate the Python environment
source venv/bin/activate

# run the code (disable progress bar when running batch)
TQDM_DISABLE=1 python augment_images_multiprocessing.py

# Clean-up
deactivate
module purge
```

::: spoiler

### Disabling TQDM

By prepending `TQDM_DISABLE=1` to the Python command, we turn off the progress bar, which is not really appropriate for batch code.

:::

::: challenge

### Run the Code

Schedule the job to run on Iridis 6.

::: solution

On the Iridis 6 login node run:
``` bash
[login6003 augment_images]$ sbatch run_multiprocess.slurm
```
you can monitor job progress with `squeue -lu <username>` or looking at the `sbatch` file in your current directory.

When the job is done, the results in the SLURM output file should look like this:

```
===============================================================================
Job started on Thu Jul 23 12:39:31 BST 2026
Job ID          : 2865502
Job name        : python-multiprocessing
WorkDir         : /iridisfs/home/cjw1a26/byte-sized-rse-python-hpc-example
Command         : /iridisfs/home/cjw1a26/byte-sized-rse-python-hpc-example/run_m
ultiprocess.slurm
Partition       : batch
Num hosts       : 1
Num cores       : 64
Num of tasks    : 1
Hosts allocated : red6093
Job Output Follows ...
===============================================================================
Running on host: red6093
Job started at: Thu Jul 23 12:39:31 BST 2026
SLURM job ID: 2865502
Number of CPUs: 64
Available cores: 64
Time taken: 11.27s
==============================================================================
Running epilogue script on red6093.

Submit time  : 2026-07-23T12:39:30
Start time   : 2026-07-23T12:39:31
End time     : 2026-07-23T12:39:52
Elapsed time : 00:00:21 (Timelimit=00:05:00)

Job ID: 2865502
Cluster: iridis_vi
User/Group: cjw1a26/fp
State: COMPLETED (exit code 0)
Nodes: 1
Cores per node: 64
CPU Utilized: 00:00:13
CPU Efficiency: 0.97% of 00:22:24 core-walltime
Job Wall-clock time: 00:00:21
Memory Utilized: 11.14 MB
Memory Efficiency: 0.01% of 209.38 GB (3.27 GB/core)
```

This is significantly faster than the local speed: the time taken for the inner loop is only 11.27 seconds, although the total time taken from submission to completion is about 22 seconds, and the CPU efficiency is only about 1%, which means 64 cores is overkill.

:::

:::

## Multiple nodes with MPI4Py

There are a couple of ways that we can extend the work to multiple nodes.  MPI4Py supports an `MPIPoolExecutor` which is almost a drop-in replacement for the `ProcessPoolExecutor` (there is an `MPICommExecutor` which supports older MPI APIs but isn't a drop-in replacement).

The main difference between the MPI version of the script and other versions is that we have removed TQDM and other output relevant for the console, since the script will be run in batch mode. The core code looks almost the same as the `concurrent.futures` example.

``` python
from mpi4py.futures import MPIPoolExecutor

paths = sorted(Path(input_dir).glob("*.jpg"))

with MPIPoolExecutor() as executor:
    for result in executor.map(augment_image, paths):
        # if the function returns anything we could do something with
        # the result; but augment_image doesn't return anything so we
        # do nothing
        pass
```

The batch file is similar to the previous one:

``` bash
#!/bin/sh

#SBATCH --job-name=python-mpi
#SBATCH --partition=batch
#SBATCH --time=00:15:00     # Maximum run time
#SBATCH --nodes=2           # Run on just 2 node
#SBATCH --ntasks=64         # Run 64 workers across the nodes

# Optional: print useful job info
echo "Running on host: $(hostname)"
echo "Job started at: $(date)"
echo "SLURM job ID: $SLURM_JOB_ID"
echo "Number of tasks per node: $SLURM_NTASKS_PER_NODE"

# Load required modules
module purge
module load openmpi
module load python

# Move to job directory
cd $SLURM_SUBMIT_DIR

# activate the Python environment
source venv/bin/activate

# run the code
mpirun -mca coll_hcoll_enable 0 -np $SLURM_NTASKS python -m mpi4py.futures augment_images_mpi.py

# Clean-up
deactivate
module purge
```

The key differences are the number of nodes and tasks (which are ):

``` bash
#SBATCH --nodes=2           # Run on just 2 nodes.
#SBATCH --ntasks=64         # Run 64 processes across the nodes.
```

the fact that we need to load OpenMPI:
``` bash
module load openmpi
```
and the command to run the code:

``` bash
mpirun -mca coll_hcoll_enable 0 -n $SLURM_NTASKS python -m mpi4py.futures augment_images_mpi.py
```

The key points about the latter:

- `mpirun` (or equivalently `mpiexec`) runs a program that uses MPI. It is configured with the total number of processes being run. There are many other command-line options which can be set, but this is the main one for us.
- `-mca coll_hcoll_enable 0` turns off the `hcoll` collective communication which doesn't work with MPI on Iridis 6.
- `-n $SLURM_NTASKS` tells MPI how many total tasks are being run.
- instead of running Python code directly, we use the `mpi4py.futures` module which handles configuring the MPI infrastructure to run with the `mpi4py.futures` module, and then invokes our code.

::: challenge

### Run the Code

The first thing we need is to make sure that we have mpi4py and any other dependencies installed in our Python environment:

``` bash
[login6003 augment_images] (venv)$ pip install --requirements-from-script augment_images_mpi.py
```

As in previous exercises, to do a fair comparison we need to remove the output since overwriting this many files is significantly slower than writing new files. This may take a few seconds:

``` bash
(venv)$ rm -rf output/
```

Now schedule the job to run on Iridis 6.

::: solution

On the Iridis 6 login node run:
``` bash
[login6003 augment_images]$ sbatch run_mpi.slurm
```
you can monitor job progress with `squeue -lu <username>` or looking at the `sbatch` file in your current directory.

:::

:::

As it turns out, the overhead of distribution doesn't make up for the additional nodes in this case, and the work is starting to be I/O bound rather than CPU bound with the slower storage.  But the step from single node to multinode is very easy, and for more computationaly expensive code, you will see a speed-up.

::: callout

The `mpi4py.futures` module is a comparatively new addition to MPI4Py, and has a very different flavour to the way MPI is typically used. Typically the MPI job will start up a collection of different processes, and those processes will use environment information to decide what role they should perform, and sending data, recieving data and synchronization need to be handled at a low-level and with some care.

If you are writing code which has complex interactions between workers, you may need to learn how to program MPI at this level.

:::

::: instructor

### The "Happy Path" of `concurrent.futures`

Emphasise the ease with which the learners can go from working, local code to working single-node code, to working multi-node code with just a few simple changes to their code.

:::

::: keypoints

- There are different approaches to concurrency in Python:
  - the use of threading is a good choice when there is a lot of C extension code that releases the GIL
  - multiprocessing is good for both CPU and I/O bound code, but is more resource intensive
  - for I/O bound systems, such as servers and networking code, asyncio is a good choice
- Python's Global Interpreter Lock generally means that only one Python instruction can be executed at a time, but carefully written C extensions can release the GIL to permit concurrency.
- The `concurrent.futures` module in the standard library provides an interface for concurrency which is suiltable for common "scatter-gather" and "map-reduce" workflows and is consistent across different approaches to concurrency.
- The `mpi4py.futures` module, which is part of MPI4Py gives an easy method of taking `concurrent.futures` code which works locally and distributing it across multiple nodes of an HPC system.

:::
