---
title: Python Performance
description: Python/Cython Performance
weight: 3
---
{{% pageinfo %}}
This is ugly code, quickly put together. Lots of room to improve
{{% /pageinfo %}}

## Abstract

We test on a 2d array the run time of 1) pure python 2) python with c type defs and 3) python that calls c code. The results find a 10x improvement just c type defs. And a futher 2x improvement by writing pure c code. 

Considering the complexity of writing C code, it makes sense to just start with python with c type defs and add pure c code only in a heavily bottle-necked area of code.

## Results

```bash
$ python main.py 
Timing for 'sum_data_pure_python'
Total execution time: 2.103925 seconds
Sum of data: 3232234374

Timing for 'sum_data_cython'
Total execution time: 0.065038 seconds
Sum of data: 3232234374

Timing for 'sum_data_pure_c'
Total execution time: 0.038198 seconds
Sum of data: 3232234374
```


## Goals
* Show off fancy pythoning (usage of decorators)
* Show off cython in 2 modes
  * Just declaring ctypes
  * Declaring ctypes and inline C code
* Monitor results of 3 different scenarios in order to find trade off

## Methodology

### Algorithm

The goal of the function is to add each element in a 2d array.

For this test we chose to a use a brute force algorithm.

```bash
sum = 0
for each row
  for each column
    sum = sum + array[row][column]
```

### Data creation

Some more investigation has to be done on the data set. 
Python normally uses a list. 
However we want need to convert to a 2d array for cython and pure C

```python
data_set_list = [[random.randint(1, 100) for _ in range(4000)] for _ in range(4000)]
data_set_array = np.array(data_list, dtype=np.int64)
```

### Timing decorator

It was tricky to use the same decorator with native Python (.py) and Cython (.pyx) files

  1. It requires a python (def) function to use the decorator @ symbol and call the cython function
  1. Then you must call the cython (cpdef) function as normally

```bash
import time

# Pass in function to time
def timing_decorator(func):

    # Pass in function to times's variables
    def wrapper(*args, **kwargs):

        # Start time
        print(f"Timing for '{func.__name__}'") 
        start_time = time.time()

        # Do the thing
        result = func(*args, **kwargs)

        # End time 
        end_time = time.time()
        print(f"Total execution time: {end_time - start_time:.6f} seconds")
        print(f"Sum of data: {result}")
        print()

        return result
    return wrapper
```

### Cython mixed code

```python
# cython: language_level=3

import random
from decorators import timing_decorator

cpdef long sum_data_set_cython_impl(long[:, :] data):
    cdef long total = 0
    cdef int i, j
    for i in range(data.shape[0]):
        for j in range(data.shape[1]):
            total += data[i, j]
    return total

# Python wrapper function to apply the decorator
@timing_decorator
def sum_data_cython(long[:, :] data):
    return sum_data_set_cython_impl(data)
```

### C code called from Python

```C
// sum_array.h

long sum_2d_array(long *array, int rows, int cols);

// sum_array.c

#include "sum_array.h"

long sum_2d_array(long *array, int rows, int cols) {
    long total = 0;
    for (int i = 0; i < rows; i++) {
        for (int j = 0; j < cols; j++) {
            total += array[i * cols + j];
        }
    }
    return total;
}

# sum_array.pyx

cdef extern from "sum_array.h":
    long sum_2d_array(long *array, int rows, int cols)

# Internal Cython function
cdef long sum_data_set_cython_impl(long[:,:] data):
    cdef int rows = data.shape[0]
    cdef int cols = data.shape[1]
    return sum_2d_array(&data[0,0], rows, cols)

# Python wrapper function to apply the decorator
from decorators import timing_decorator

@timing_decorator
def sum_data_pure_c(long[:,:] data):
    return sum_data_set_cython_impl(data)

```

Using the decorator like this results in some very clean code
```python
## Test Pure Python
import pure_python
result = pure_python.sum_data_pure_python(data_list)

## Test Mixed Python
import mixed_cython
result = mixed_cython.sum_data_cython(data_array)

## Test Pure C called from Python
import pure_c
result = pure_c.sum_data_pure_c(data_array)
```

Cython compiles Python into machine code that can be executed on the host.
- This usually comes in the form of static (.a) or shared (.so) libs

To build these libraries we can use the following python compilation

```python
setuptools import setup, Extension
from Cython.Build import cythonize

# Define the C extension
pure_c = Extension(
    name="pure_c",
    sources=["pure_c.pyx", "sum_array.c"],
)

decorators = Extension(
    name="decorators",
    sources=["decorators.py"],
)

mixed_cython = Extension(
    name="mixed_cython",
    sources=["mixed_cython.pyx"],
)

# Setup function
setup(
    ext_modules=cythonize([decorators, pure_c, mixed_cython])
)
```

## Github (coming soon)
