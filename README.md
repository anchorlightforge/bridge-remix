# NVIDIA RTX Remix Bridge

[![Build Status](https://github.com/NVIDIAGameWorks/bridge-remix/actions/workflows/build.yml/badge.svg)](https://github.com/NVIDIAGameWorks/bridge-remix/actions/workflows/build.yml)

The NVIDIA RTX Remix project allows bringing high quality pathtraced rendering, lighting, shadows etc. into classic games. This repo contains the NVIDIA RTX Remix Bridge client and server components required for enabling a 32-bit game to interact with the 64-bit Remix Runtime dll.

> **NOTE:** To experience the full feature set of NVIDIA RTX Remix, binaries compiled from this repo need to be combined with the binaries from the `dxvk-remix` [repo on GitHub](https://github.com/NVIDIAGameWorks/dxvk-remix/). For additional details and explanation see below!

# Prerequisites

All that is required to build Remix is a computer running an up-to-date version of Microsoft Windows 10 or 11 as well as [Microsoft Visual Studio](https://visualstudio.microsoft.com/).

The free Commmunity Edition of Visual Studio will work fine.

> **NOTE:** Make sure to install the `MSVC v142 - VS 2019 C++ x64/x86 build tools` from the _Individual Components_ list in the Visual Studio Installer interface. NVIDIA RTX Remix has only been tested with the Visual Studio 2019 build tools, and newer compiler versions may or may not work and/or lead to unexpected results/issues.

# How to build

To generate the initial Visual Studio solutions and build, run `build_bridge_all.bat`, which will generate four separate solutions for the debug/release configurations for both 32 and 64 bit builds.

You can also open and rebuild the projects/solutions individually from the different `_comp<config>_<platform>` directories.

> **NOTE:** To get a full Remix Bridge build you will need to open and compile both the x86 as well as the x64 solution, because the bridge client component is built by the x86 solution, and the bridge server component is built by the x64 solution.

The build output from the x86 solution goes into a folder called `_output` which is located (and will be created if it doesn't exist) in the repo root.

The build output from the x64 solution goes into a folder called `.trex` which is located inside the `_output` directory in the repo root.

> **NOTE:** Technically the Remix Bridge can be used on a game by itself without the Remix Runtime components from the `dxvk-remix` repo, but when used that way it will fall back onto the x64 system DirectX9 runtime that is installed in Windows and not be capable of performing any raytracing or asset replacements.

# How to run

## Drop and go

All you need to do to run Remix Bridge is copy the build output into a game directory so that the Remix Bridge `d3d9.dll` gets picked up on game launch, which will then load all other components.

> **NOTE:** Depending on the game the correct location for the Remix Runtime files may vary, it could be in the game's root directory or a subfolder called `bin` or something like that. It may take some trial and error to figure out the correct location for a specific game!

## Using the Remix Bridge Launcher

An alternative approach to loading the Remix Bridge `d3d9.dll` into a game would be to use an executable called `NvRemixLauncher32.exe`, which is generated by the Visual Studio x86 solution. Some games will not pick up the Remix Bridge `d3d9.dll` and require injection at runtime instead. The launcher executable needs to be located next to the other Remix Bridge files and can then be used from the command line like this:

```
> NvRemixLauncher32.exe <game executable> <other launch parameters>
```

Run the launcher without any parameters to see the list of available options that can be used.

## Visual Studio debugging

The intended way to use these components is to use the 32-bit `d3d9.dll` output compiled by the client `d3d9` project from the x86 solution and the 64-bit server dll compiled by the server `NvRemixBridge` project in the x64 solution. If the file exists inside the `.trex` directory, the server bridge component will load the 64-bit Remix Runtime (`dxvk-remix`) `d3d9.dll` to pass the rendering commands and results to the Vulkan pathtracing renderer and back, but by default it loads the regular system DirectX9 dll that is installed in the default Windows system directory.

> **NOTE:** The x86 solutions contain both the `client` and `server` projects, but not the copy job for the `server` project, because the `server` output comes from the x64 solution, which only contains the `server` project and copy job for it. The `server` project was left in the x86 solution only to make cross-process debugging easier so that Visual Studio picks up the source files correctly when doing child process debugging.

Files are copied to `_output` after building. During development it is helpful to create a symbolic link in Windows that reroutes the `_output` directory to a different path, for example directly into a game directory. The link can be created on the command line with

```
> mklink /j _output <path to game directory>
```

If you then enter the game executable in the `Command` field and the game path as the `Working Directory` property in the `d3d9` project settings on the `Debugging` property page, then it makes it easier to launch the game and debug Remix Bridge directly from Visual Studio.

> **NOTE:** It is recommended to download the VS extension to help debugging/attaching to child processes (such as the server NvRemixBridge component)
>
> For Visual Studio 2015, 2017, 2019: https://marketplace.visualstudio.com/items?itemName=vsdbgplat.MicrosoftChildProcessDebuggingPowerTool
>
> For Visual Studio 2022: https://marketplace.visualstudio.com/items?itemName=vsdbgplat.MicrosoftChildProcessDebuggingPowerTool2022

# Using RTX Remix Bridge with Retail Games

Remix Bridge should work out of the box with many retail games as well, but depending on the game extra configuration steps may be required to make sure the game runs with the correct DirectX settings. For example `Portal` requires being launched with `-dxlevel 70` to work properly.

Also, as mentioned above if you want to take advantage of the pathtracing capabilities of RTX Remix then you need to combine the Remix Bridge binaries with the Remix Runtime dll from the `dxvk-remix` [repo on GitHub](https://github.com/NVIDIAGameWorks/dxvk-remix/).

The easiest way to get a complete binary package is by downloading one of the releases from the `RTX Remix` [repo on GitHub]((https://github.com/NVIDIAGameWorks/rtx-remix/)).

# Remix Bridge Architecture & Implementation Details

- The client side `d3d9` project handles intercepting all the DirectX9 API calls from the 32-bit game/app, and the server side `NvRemixBridge.exe` receives those calls and executes them in 64-bit process and memory space, passing them on to either the x64 system DirectX9 runtime or another rendering shim (if present).
- The server renders directly into the client window. Any client side d3d9 runtime calls are implemented without calling into the system runtime, so only the server side bridge component creates an actual d3d9 device for the game process, which helps with game compatibility especially when running in exclusive fullscreen mode.
- The bridge server uses the window handle for the window opened by the client and presents directly into it, which saves us having to do any shared surface copying etc.
- The main communication channel between the client and the server is done via circular queues which are combined to form an `IpcChannel`.
  - One `IpcChannel` is used for all the client commands sent to the server, and another channel sends the responses back to the client. Likewise within each `IpcChannel` one queue handles the commands being invoked, and the other queue has all the data for parameters and arguments needed by the command.
  - The queues are global static variables and the queue sizes are defined in `util_common.h` as well.
  - The server and client processes each have their own queue instances, but both point at the same shared memory so they read/write data from the same memory to avoid any extra copying.
  - Read and write access follows a simplified producer/consumer pattern and is regulated either via atomic counters (`AtomicCircularQueue`) or via semaphores that get triggered when the client/server are done writing/reading (`BlockingCircularQueue`), and when the queue is empty the server will block and wait for a new command to come in. Should a queue get filled up by the writer then it will block until the reader consumed items from the queue.
  - Most of the time the client will be the writer and the server will be the reader for the queues, except during application startup where the client and server exchange a handshake, which the server acknowledges by sending a response back to the client, and when commands are executed that require a response from the server, for example when surface data is being copied into the data queue so that it can be read out on the client side.
  - New commands/data are added to the queue via `push()` and read by doing `pull()` calls. There are some `PUSH` and `PULL` macros defined to make it easier to quickly send a command or read data of specific data types.
  - The data queue supports sending data of arbitrary size by passing it a size and the pointer to the data being pushed. The underlying base type for the data queue is `uint32_t`, which means data is being sent in 4-byte chunks. Sending `nullptr` (i.e. no data but strongly typed) is also supported.
- There are some helper functions to support development and troubleshooting:
  - `LogMissingFunctionCall()` is used to mark D3D9 API functions that have not been fully implemented on the client and server yet. During execution when running in `Debug` configuration or with logging set to `Debug` level the client will print calls to missing functions in the debug output window in Visual Studio as well as write them to the log file, making it easy to spot functions that still need to be implemented for a specific game or use case.
  - `LogFunctionCall()` does nothing by default but when combined with the `LOG_ALL_CALLS` define can be used to log calls to all functions regardless of implementation status. This is helpful to understand the flow of calls that may have led to a certain crash/bug or other situation that needs to be debugged.
- Some comments on code organization:
  - On the client side the implementation code is split over separate files matching the names of the D3D9 API interfaces, but all the headers are consolidated into a single file `d3d9_lss.h`.
  - On the server side almost all code is currently in `main.cpp` and may get refactored into separate files for each interface similar to the client at some point. It's a long file with a very long `switch` statement, but since a lot of the interface methods are very similar this also makes it easier to navigate back and forth between related functions without having to jump between multiple files. It probably makes most sense to reorganize once we have reached a certain amount of completeness and stability, so refactoring doesn't lead to a lot of merge conflicts with other changes being done in parallel.
