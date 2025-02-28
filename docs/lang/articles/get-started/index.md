---
sidebar_position: 1
slug: /docs
---

# Getting Started

Welcome to the Taichi Language documentation!

## Installation

To get started with the Taichi Language, simply install it with `pip`:

```shell
python3 -m pip install taichi
```

:::note
Currently, Taichi only supports Python 3.6/3.7/3.8/3.9/3.10 (64-bit).
:::

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

There are a few of extra requirements depend on which operating system you are using:

<Tabs
  defaultValue="arch-linux"
  values={[
    {label: 'Arch Linux', value: 'arch-linux'},
    {label: 'Windows', value: 'windows'},
  ]}>

  <TabItem value="arch-linux">

  On Arch Linux, you need to install `ncurses5-compat-libs` package from the Arch User Repository: `yaourt -S ncurses5-compat-libs`

  </TabItem>
  <TabItem value="windows">

  On Windows, please install [Microsoft Visual C++
  Redistributable](https://aka.ms/vs/16/release/vc_redist.x64.exe) if you haven't done so.

  </TabItem>
</Tabs>

Please refer to the [Installation Troubleshooting](../misc/install.md) section if you run into any issues when installing Taichi.

## Hello, world!

We introduce the Taichi programming language through a very basic _fractal_ example.

Running the Taichi code below using either `python3 fractal.py` or `ti example fractal` _(you can find more information about the Taichi CLI in the [Command line utilities](../misc/cli_utilities.md) section)_ will give you an animation of [Julia set](https://en.wikipedia.org/wiki/Julia_set):

<center>

![image](https://raw.githubusercontent.com/taichi-dev/public_files/master/taichi/fractal.gif)

</center>

```python title=fractal.py
import taichi as ti

ti.init(arch=ti.gpu)

n = 320
pixels = ti.field(dtype=float, shape=(n * 2, n))

@ti.func
def complex_sqr(z):
    return ti.Vector([z[0]**2 - z[1]**2, z[1] * z[0] * 2])

@ti.kernel
def paint(t: float):
    for i, j in pixels:  # Parallelized over all pixels
        c = ti.Vector([-0.8, ti.cos(t) * 0.2])
        z = ti.Vector([i / n - 1, j / n - 0.5]) * 2
        iterations = 0
        while z.norm() < 20 and iterations < 50:
            z = complex_sqr(z) + c
            iterations += 1
        pixels[i, j] = 1 - iterations * 0.02

gui = ti.GUI("Julia Set", res=(n * 2, n))

for i in range(1000000):
    paint(i * 0.03)
    gui.set_image(pixels)
    gui.show()
```

Let's dive into this simple Taichi program.

### import taichi as ti

Taichi is a domain-specific language (DSL) embedded in Python.

To make Taichi as easy to use as a Python package, we have done heavy
engineering with this goal in mind - letting every Python programmer
write Taichi programs with minimal learning effort.

You can even use your favorite Python package management system, Python IDEs and other Python packages in conjunction with Taichi.

```python
# Run on GPU, automatically detect backend
ti.init(arch=ti.gpu)

# Run on GPU, with the NVIDIA CUDA backend
ti.init(arch=ti.cuda)
# Run on GPU, with the OpenGL backend
ti.init(arch=ti.opengl)
# Run on GPU, with the Apple Metal backend, if you are on macOS
ti.init(arch=ti.metal)

# Run on CPU (default)
ti.init(arch=ti.cpu)
```

:::info

Supported backends on different platforms:

| **platform** | **CPU** | **CUDA** | **OpenGL** | **Metal** | **C source** |
| :----------: | :-----: | :------: | :--------: | :-------: | :----------: |
|   Windows    |   OK    |    OK    |     OK     |    N/A    |     N/A      |
|    Linux     |   OK    |    OK    |     OK     |    N/A    |      OK      |
|    macOS     |   OK    |   N/A    |    N/A     |    OK     |     N/A      |

(OK: supported; N/A: not available)

With `arch=ti.gpu`, Taichi will first try to run with CUDA. If CUDA is
not supported on your machine, Taichi will fall back on Metal or OpenGL.
If no GPU backend (CUDA, Metal, or OpenGL) is supported, Taichi will
fall back on CPUs.
:::

:::note

When used with the CUDA backend on Windows or ARM devices (e.g., NVIDIA
Jetson), Taichi allocates 1 GB GPU memory for field storage by default.

You can override this behavior by initializing with
`ti.init(arch=ti.cuda, device_memory_GB=3.4)` to allocate `3.4` GB GPU
memory, or `ti.init(arch=ti.cuda, device_memory_fraction=0.3)` to
allocate `30%` of the total GPU memory.

On other platforms, Taichi will make use of its on-demand memory
allocator to allocate memory adaptively.
:::

### Fields

Taichi is a **data**-oriented programming language where dense or
spatially-sparse fields are the first-class citizens.

In the code above, `pixels = ti.field(dtype=float, shape=(n * 2, n))`
allocates a 2D dense field named `pixels` of size `(640, 320)` and
element data type `float`.

### Functions and kernels

Computation resides in Taichi **kernels** and Taichi **functions**.

Taichi **kernels** are defined with the decorator `@ti.kernel`. They can
be called from Python to perform computation. Kernel arguments must be
type-hinted (if any).

Taichi **functions** are defined with the decorator `@ti.func`. They can
**only** be called by Taichi kernels or other Taichi functions.

See [syntax](../basic/syntax.md) for more details about Taichi
kernels and functions.

The language used in Taichi kernels and functions looks exactly like
Python, yet the Taichi frontend compiler converts it into a language
that is **compiled, statically-typed, lexically-scoped, parallel and
differentiable**.

:::info

**Taichi-scopes v.s. Python-scopes**:

Everything decorated with `@ti.kernel` and `@ti.func` is in Taichi-scope
and hence will be compiled by the Taichi compiler.

Everything else is in Python-scope. They are simply Python native code.
:::

:::caution

Taichi kernels must be called from the Python-scope. Taichi functions
must be called from the Taichi-scope.
:::

:::tip

For those who come from the world of CUDA, `ti.func` corresponds to
`__device__` while `ti.kernel` corresponds to `__global__`.
:::

:::note

Nested kernels are **not supported**.

Nested functions are **supported**.

Recursive functions are **not supported for now**.
:::

### Parallel for-loops

For loops at the outermost scope in a Taichi kernel is **automatically
parallelized**. For loops can have two forms, i.e. _range-for
loops_ and _struct-for loops_.

**Range-for loops** are no different from Python for loops, except that
they will be parallelized when used at the outermost scope. Range-for
loops can be nested.

```python {3,7,14-15}
@ti.kernel
def fill():
    for i in range(10): # Parallelized
        x[i] += i

        s = 0
        for j in range(5): # Serialized in each parallel thread
            s += j

        y[i] = s

@ti.kernel
def fill_3d():
    # Parallelized for all 3 <= i < 8, 1 <= j < 6, 0 <= k < 9
    for i, j, k in ti.ndrange((3, 8), (1, 6), 9):
        x[i, j, k] = i + j + k
```

:::note

It is the loop **at the outermost scope** that gets parallelized, not
the outermost loop.

```python {3,9}
@ti.kernel
def foo():
    for i in range(10): # Parallelized :-)
        ...

@ti.kernel
def bar(k: ti.i32):
    if k > 42:
        for i in range(10): # Serial :-(
            ...
```

:::

**Struct-for loops** are particularly useful when iterating over
(sparse) field elements. In the `fractal.py` above, `for i, j in pixels` loops
over all the pixel coordinates, i.e.,
`(0, 0), (0, 1), (0, 2), ... , (0, 319), (1, 0), ..., (639, 319)`.

:::note

Struct-for is the key to [sparse computation](../advanced/sparse.md) in
Taichi, as it will only loop over active elements in a sparse field. In
dense fields, all elements are active.
:::

:::caution

Struct-for loops must live at the outer-most scope of kernels.

It is the loop **at the outermost scope** that gets parallelized, not
the outermost loop.

```python
x = [1, 2, 3]

@ti.kernel
def foo():
    for i in x: # Parallelized :-)
        ...

@ti.kernel
def bar(k: ti.i32):
    # The outermost scope is a `if` statement
    if k > 42:
        for i in x: # Not allowed. Struct-fors must live in the outermost scope.
            ...
```

:::

:::caution

`break` **is not supported in parallel loops**:

```python {5,9,16}
@ti.kernel
def foo():
  for i in x:
      ...
      break # Error!

  for i in range(10):
      ...
      break # Error!

@ti.kernel
def foo():
  for i in x:
      for j in range(10):
          ...
          break # OK!
```

:::

### GUI system

Taichi provides a cpu-based [GUI system](../gui/gui_system.md) for users to render
their results on the screen.

```python
gui = ti.GUI("Julia Set", res=(n * 2, n))

for i in range(1000000):
  paint(i * 0.03)
  gui.set_image(pixels)
  gui.show()
```

### Interacting with other Python packages

#### Python-scope data access

Everything outside Taichi-scopes (`ti.func` and `ti.kernel`) is simply
Python code. In Python-scopes, you can access Taichi field elements
using plain indexing syntax. For example, to access a single pixel of
the rendered image in Python-scope, you can simply use:

```python
import taichi as ti
pixels = ti.field(ti.f32, (1024, 512))

pixels[42, 11] = 0.7  # store data into pixels
print(pixels[42, 11]) # prints 0.7
```

### Sharing data with other packages

Taichi provides helper functions such as `from_numpy` and `to_numpy` to
transfer data between Taichi fields and NumPy arrays, so that you can
also use your favorite Python packages (e.g., `numpy`, `pytorch`,
`matplotlib`) together with Taichi as below:

```python
import taichi as ti
pixels = ti.field(ti.f32, (1024, 512))

import numpy as np
arr = np.random.rand(1024, 512)
pixels.from_numpy(arr)   # load numpy data into taichi fields

import matplotlib.pyplot as plt
arr = pixels.to_numpy()  # store taichi data into numpy arrays
plt.imshow(arr)
plt.show()

import matplotlib.cm as cm
cmap = cm.get_cmap('magma')
gui = ti.GUI('Color map')
while gui.running:
    render_pixels()
    arr = pixels.to_numpy()
    gui.set_image(cmap(arr))
    gui.show()
```

See [Interacting with external arrays](../basic/external.md#interacting-with-external-arrays) for more details.

## What's next?

Now we have gone through core features of the
Taichi programming language using the fractal example,
feel free to dive into the language concepts in
the next section, or jump to the advanced topics, such as the [Metaprogramming](../advanced/meta.md) or [Differentiable programming](../advanced/differentiable_programming.md). Remember that you can
use the search bar at the top right corner to search for topics or keywords
at any time!

If you are interested in joining the Taichi community, we strongly recommend you take some time to
familiarize yourself with our [contribution guide](../contribution/contributor_guide.md).

We hope you enjoy your adventure with Taichi!
