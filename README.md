# Swift-Jupyter

This is a Jupyter Kernel for Swift, intended to make it possible to use Jupyter
with the [Swift for TensorFlow](https://github.com/tensorflow/swift) project.

# Installation Instructions

## With TensorFlow toolchain

### Requirements

Operating system:

* Ubuntu 18.04 (64-bit); OR
* other operating systems may work, but you will have to build Swift from
  sources.

Dependencies:

* Python 3 (Ubuntu 18.04 package name: `python3`)
* Python 3 Virtualenv (Ubuntu 18.04 package name: `python3-venv`)

### Installation

swift-jupyter requires a Swift toolchain with LLDB Python3 support. Currently, the only prebuilt toolchains with LLDB Python3 support are the [Swift for TensorFlow Ubuntu 18.04 Nightly Builds](https://github.com/tensorflow/swift/blob/master/Installation.md#pre-built-packages). Alternatively, you can build a toolchain from sources (see the next section for instructions).

Extract the Swift toolchain somewhere.

Create a virtualenv, install the requirements in it, and register the kernel in
it:

```
python3 -m venv venv
. venv/bin/activate
pip install -r requirements.txt
python register.py --sys-prefix --swift-toolchain <path to extracted swift toolchain directory>
```

Finally, run Jupyter:

```
. venv/bin/activate
jupyter notebook
```

You should be able to create Swift notebooks. Installation is done!

### (optional) Building toolchain with LLDB Python3 support

Follow the
[Building Swift for TensorFlow](https://github.com/apple/swift/tree/tensorflow#building-swift-for-tensorflow)
instructions, with some modifications:

* Also install the Python 3 development headers. (For Ubuntu 18.04,
  `sudo apt-get install libpython3-dev`). The LLDB build will automatically
  find these and build with Python 3 support.
* Instead of running `utils/build-script`, run
  `SWIFT_PACKAGE=tensorflow_linux,no_test ./swift/utils/build-toolchain local.swift`
  or `SWIFT_PACKAGE=tensorflow_linux ./swift/utils/build-toolchain local.swift,gpu,no_test`
  (depending on whether you want to build tensorflow with GPU support).

This will create a tar file containing the full toolchain. You can now proceed
with the installation instructions from the previous section.

## Using the Docker Container

This repository also includes a dockerfile which can be used to run a Jupyter Notebook instance which includes this Swift kernel. To build the container, the following command may be used:

```bash
# from inside the directory of this repository
docker build -f docker/Dockerfile -t swift-jupyter .
```

The resulting container comes with the latest Swift for TensorFlow toolchain installed, along with Jupyter and the Swift kernel contained in this repository.

This container can now be run with the following command:

```bash
docker run -p 8888:8888 --cap-add SYS_PTRACE -v /my/host/notebooks:/notebooks swift-jupyter
```

The functions of these parameters are:

- `-p 8888:8888` exposes the port on which Jupyter is running to the host.

- `--cap-add SYS_PTRACE` adjusts the privileges with which this container is run, which is required for the Swift REPL.

- `-v <host path>:/notebooks` bind mounts a host directory as a volume where notebooks created in the container will be stored.  If this command is omitted, any notebooks created using the container will not be persisted when the container is stopped. 

# Usage Instructions

## Rich output

You can call Python libraries using [Swift's Python interop] to display rich
output in your Swift notebooks. (Eventually, we'd like to support Swift
libraries that produce rich output too!)

Prerequisites:

* You must use a Swift toolchain that has Python interop. As of July 2018,
  only the [Swift for TensorFlow] toolchain has Python interop.

* Install the `ipykernel` Python library, and any other Python libraries
  that you want output from (such as `matplotlib` or `pandas`) on your
  system Python. (Do not install them on the virtualenv from the Swift-Jupyter
  installation instructions. Swift's Python interop talks to your system
  Python.)

After taking care of the prerequisites, run
`%include "EnableIPythonDisplay.swift"` in your Swift notebook. Now you should
be able to display rich output! For example:

```swift
let np = Python.import("numpy")
let plt = Python.import("matplotlib.pyplot")
IPythonDisplay.shell.enable_matplotlib("inline")
```

```swift
let time = np.arange(0, 10, 0.01)
let amplitude = np.exp(-0.1 * time)
let position = amplitude * np.sin(3 * time)

plt.figure(figsize: [15, 10])

plt.plot(time, position)
plt.plot(time, amplitude)
plt.plot(time, -amplitude)

plt.xlabel("time (s)")
plt.ylabel("position (m)")
plt.title("Oscillations")

plt.show()
```

![Screenshot of running the above two snippets of code in Jupyter](./screenshots/display_matplotlib.png)

```swift
let display = Python.import("IPython.display")
let pd = Python.import("pandas")
```

```swift
display.display(pd.DataFrame.from_records([["col 1": 3, "col 2": 5], ["col 1": 8, "col 2": 2]]))
```

![Screenshot of running the above two snippets of code in Jupyter](./screenshots/display_pandas.png)

[Swift's Python interop]: https://github.com/tensorflow/swift/blob/master/docs/PythonInteroperability.md

## %include directives

`%include` directives let you include code from files. To use them, put a line
`%include "<filename>"` in your cell. The kernel will preprocess your cell and
replace the `%include` directive with the contents of the file before sending
your cell to the Swift interpreter.

`<filename>` must be relative to the directory containing `swift_kernel.py`.
We'll probably add more search paths later.
