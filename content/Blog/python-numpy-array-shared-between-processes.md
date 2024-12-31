---
title: "Python Numpy Array Shared Between Processes"
date: 2024-01-05T10:03:05+01:00
publish: true
description: ""
tags: ["python", "numpy", "multiprocessing", "shared_memory"]
---

Python's [multiprocessing.shared_memory](https://docs.python.org/3/library/multiprocessing.shared_memory.html) module provides a way to share a block of memory between processes.
It's usefull in ml contexts, where an app will runs ML model in separate process and wants share the results
with its parent process. Frigate NVR does exacly that, see: [object_detection.py](https://github.com/blakeblackshear/frigate/blob/c5ccc0fb08cdcfd35d2dae534724103bfb7c82a8/frigate/object_detection.py#L186-L232).

```python
# Define properties of shared memory
SHARED_MEMORY_NAME = "shm-1"
SHARED_MEMORY_SHAPE = (5, 5)
NP_DATA_TYPE = np.float64

# Calculate shared numpy array size
dsize = np.dtype(NP_DATA_TYPE).itemsize * np.prod(SHARED_MEMORY_SHAPE).astype(int)

# Create allocate shared memory with a given name
shm = shared_memory.SharedMemory(
    create=True,
    size=dsize,
    name=SHARED_MEMORY_NAME,
)
```

To access it from a different process, create `SharedMemory` with `create=False` and the same `name`.

For example:

```python
class ShmModifier(Process):
    def __init__(self) -> None:
        # Required as per: https://docs.python.org/3/library/multiprocessing.html#multiprocessing.Process
        Process.__init__(self)

        # Get shared memory
        self.out_shm = shared_memory.SharedMemory(name=SHARED_MEMORY_NAME, create=False)

        # Map it to numpy array
        self.out_np = np.ndarray(
            SHARED_MEMORY_SHAPE, dtype=np.float32, buffer=self.out_shm.buf
        )

    def run(self) -> None:
        # Overrides a numpy array (runs in a different process)
        self.out_np[0, 0] = 1
        self.out_shm.close()
```

## Full working example

```python
import numpy as np
from multiprocessing import Process
import multiprocessing.shared_memory as shared_memory


SHARED_MEMORY_NAME = "shm-1"
SHARED_MEMORY_SHAPE = (5, 5)
NP_DATA_TYPE = np.float64


class ShmModifier(Process):
    def __init__(self) -> None:
        # Required as per: https://docs.python.org/3/library/multiprocessing.html#multiprocessing.Process
        Process.__init__(self)

        # Get shared memory
        self.out_shm = shared_memory.SharedMemory(name=SHARED_MEMORY_NAME, create=False)

        # Map it to numpy array
        self.out_np = np.ndarray(
            SHARED_MEMORY_SHAPE, dtype=np.float32, buffer=self.out_shm.buf
        )

    def run(self) -> None:
        # Overrides a numpy array (runs in a different process)
        self.out_np[0, 0] = 1
        self.out_shm.close()


if __name__ == "__main__":
    # Calculate shared numpy array size
    dsize = np.dtype(NP_DATA_TYPE).itemsize * np.prod(SHARED_MEMORY_SHAPE).astype(int)

    # Create allocate shared memory with a given name
    shm = shared_memory.SharedMemory(
        create=True,
        size=dsize,
        name=SHARED_MEMORY_NAME,
    )
    out_np = np.ndarray(SHARED_MEMORY_SHAPE, dtype=np.float32, buffer=shm.buf)

    # Fill arr with zeros
    out_np[:] = np.zeros_like(out_np)

    print("Array before running the process:")
    print(out_np)
    dp = ShmModifier()
    dp.start()
    dp.join()

    print("Array after running the process:")
    print(out_np)
    shm.unlink()
```

Running this script will result in:

```text
Array before running the process:
[[0. 0. 0. 0. 0.]
 [0. 0. 0. 0. 0.]
 [0. 0. 0. 0. 0.]
 [0. 0. 0. 0. 0.]
 [0. 0. 0. 0. 0.]]
Array after running the process:
[[1. 0. 0. 0. 0.]
 [0. 0. 0. 0. 0.]
 [0. 0. 0. 0. 0.]
 [0. 0. 0. 0. 0.]
 [0. 0. 0. 0. 0.]]
```

## Sources

- https://docs.python.org/3/library/multiprocessing.html
- https://docs.python.org/3/library/multiprocessing.shared_memory.html
- https://github.com/blakeblackshear/frigate/blob/c5ccc0fb08cdcfd35d2dae534724103bfb7c82a8/frigate/object_detection.py#L186-L232