# Melatonin Perfetto

[![](https://github.com/sudara/melatonin_perfetto/workflows/CI/badge.svg)](https://github.com/sudara/melatonin_perfetto/actions)

Sounds like an ice cream flavor (albeit a sleepy one).

However, it's just a way to use the amazing [Perfetto](http://perfetto.dev) performance tracing in JUCE.

[Perfetto](https://perfetto.dev) lets you accurately measure different parts of your code and visualize it over time. It's the successor to [`chrome://tracing`](https://slack.engineering/chrome-tracing-for-fun-and-profit/). 

![image-1024x396](https://user-images.githubusercontent.com/472/180338251-ce3c5814-ff9c-4fbb-a8c0-9caefc2f34dc.png)

## Why would I use Perfetto instead of the profiler?

✨  
✨✨  
✨✨✨  
### [Read the blog post!](https://melatonin.dev/blog/using-perfetto-with-juce-for-dsp-and-ui-performance-tuning)
✨✨✨  
✨✨  
✨  

## Installing with CMake

We worked hard so you don't have to. 

Not only do we handle building Perfetto (which is fairly annoying on Windows), but we've made it very easy to add this module into your CMake projects.

You have several options. In all cases, the exported target you should link against is: `Melatonin::Perfetto`.

### Option #1: `FetchContent`

Example usage:
```cmake
include (FetchContent)

FetchContent_Declare (melatonin_perfetto
  GIT_REPOSITORY https://github.com/sudara/melatonin_perfetto.git
  GIT_TAG origin/main)

FetchContent_MakeAvailable (melatonin_perfetto)

target_link_libraries (yourTarget PRIVATE Melatonin::Perfetto)
```

### Option #2 submodules and `add_subdirectory`

If you are a git submodule aficionado, add this repository as a git submodule to your project:
```sh
git submodule add -b main https://github.com/sudara/melatonin_perfetto.git modules/melatonin_perfetto
```
and then simply call `add_subdirectory` in your CMakeLists.txt:
```cmake
add_subdirectory (modules/melatonin_perfetto)

target_link_libraries (yourTarget PRIVATE Melatonin::Perfetto)
```

### Option #3 Installing and using `find_package`

Install the module to your system by cloning the code and then running the following commands:
```sh
cmake -B Builds
cmake --build Builds
cmake --install Builds
```

The `--install` command will write to system directories, so it may require `sudo`.

Once this module is installed to your system, you can simply add to your CMake project:
```cmake
find_package (MelatoninPerfetto)

target_link_libraries (yourTarget PRIVATE Melatonin::Perfetto)
```

## Installing with Projucer

### Step 1: Download the Perfetto SDK

It can go anywhere.

```
git clone https://android.googlesource.com/platform/external/perfetto -b v31.0
```

### Step 2: Add the perfetto headers in File Explorer

This is necessary to actually compile the perfetto tracing sdk from source.

In the File Explorer, hit the `+`, `Add Existing Files` and make sure the following two are added:

```
sdk/perfetto.h
sdk/perfetto.cc
```

### Step 3: Add Header Search Paths

In the Exporter (for example Xcode macOS), tell the Projucer where to find `perfetto/sdk` folder.

For example, if you downloaded it as a sibling folder to the project, you would add the following to Header Search Paths:

```
../perfetto/sdk
```

### Step 4: Add read/write permissions where necessary (macOS)

This lets perfetto write out the trace files.

With `App Sandbox` enabled, you'll have to add the following:

```
`File Access: Read/Write: Download Folder (Read/Write)` 
```

<img src="https://user-images.githubusercontent.com/472/213724719-be39512e-cda2-43cb-a589-0c3478625228.jpg" width="400"/>

### Step 5: Enable/Disable at will

As you'll see in "How to use", you can toggle perfetto traces on/off by adding the following in the exporter's preprocessor definitions:

```
PERFETTO=1 //enabled
```

## How to use

### Step 1: Add a few pieces of guarded code to your plugin


Add a member of the plugin processor:
```cpp
#if PERFETTO
    std::unique_ptr<perfetto::TracingSession> tracingSession;
#endif
```

Put this in PluginProcessor's constructor:

```cpp
#if PERFETTO
    MelatoninPerfetto::get().beginSession();
#endif
```
and in the destructor:
```cpp
#if PERFETTO
    MelatoninPerfetto::get().endSession();
#endif
```
### Step 2: Pepper around some sweet sweet macros

Perfetto will *only* measure functions you specifically tell it to.

Pepper around `TRACE_DSP()` or `TRACE_COMPONENT()` macros at the start of the functions you want to measure.

For example, in your `PluginProcessor::processBlock`:

```cpp
void SineMachineAudioProcessor::processBlock (juce::AudioBuffer<float>& buffer, juce::MidiBuffer& midiMessages)
{
    TRACE_DSP();
    
    // your dsp code here
}
```

or a component's `paint` method:

```cpp
void paint (juce::Graphics& g) override
{
    TRACE_COMPONENT();
    
    // your paint code here
}
```

By default, perfetto is disabled (`PERFETTO=0`) and the macros will evaporate on compile, so feel free to leave them in your code, especially if you plan on profiling again in the future.

### Step 3: Enable profiling

When you are ready to profile, you'll need to set `PERFETTO=1` 

This module offers a CMake option, `PERFETTO`, that when `ON`, adds the `PERFETTO` symbol to the module's exported
compile definitions. This CMake option 
is an easy way for you to turn tracing on and off from the command line when building your project:

```sh
cmake -B build -D PERFETTO=ON
```
The value of `PERFETTO` will be saved in the CMake cache, so you don't need to re-specify this every time you re-run
CMake configure. 

To turn it off again, you can do:
```sh
cmake -B build -D PERFETTO=OFF
```

You can add this as a CMake option in your IDE build settings, for example in CLion:

<img width="254" alt="CLion - 2023-01-07 06@2x" src="https://user-images.githubusercontent.com/472/211118287-0a3cd6bb-84e3-430b-a7d4-34474e51e42d.png">


Aaaaand if you are lazy like Sudara sometimes is, you can just edit melatonin_perfetto and change the default to `PERFETTO=1`...

That'll actually include the google lib and will profile the code.

### Step 4: Run your app 

Reminder: do you want to profile Release? Probably! I love profiling Debug builds too, but if you are looking for real-world numbers, you'll want to use Release.

Start your app and perform the actions you want traced.

When you quit your app, a trace file will be dumped 

**(Note: don't just terminate it via your IDE, the file will be only dumped on a graceful quit)**.

### Step 5: Drag the trace into Perfetto

Find the trace file and drag it into https://ui.perfetto.dev

You can keep the macros peppered around in your app during normal dev/release. 

Just remember to set `PERFETTO` back to `0` or `OFF` so everything gets turned into a no-op. 

## Customizing

By default, there are two perfetto "categories" defined, `dsp` and `components`. 

The current function name is passed as the name. 

You can add custom parameters that will show up in perfetto's UI:

```cpp
TRACE_DSP("startSample", startSample, "numSamples", numSamples);
```

You can also use the built in `TRACE_EVENT` which takes a name if you don't want it to derive a name based on the function.

This is handy if you want to do things like have multiple traces in a function, for example in a loop.

```cpp
    TRACE_DSP(); // start the trace, use the function name
    
    if (someCondition)
    {
        TRACE_EVENT("dsp", "someCondition");
        // do something expensive,  traced separately
    }

    moreFuctionCode(); // included in the main trace
```


You can also wrap code in `TRACE_EVENT_BEGIN` and `TRACE_EVENT_END`:

```cpp
TRACE_EVENT_BEGIN ("dsp", "memset");
// CLEAR the temp buffer
temp.clear();
TRACE_EVENT_END ("dsp");

```

Go wild! 


## Assumptions / Caveats

* On Mac, the trace is dumped to your Downloads folder. On Windows, it's dumped to your Desktop (sorry not sorry).
* Traces are set to in memory, 80MB by default.

## Troubleshooting

### I don't see a trace file

Did you quit your app gracefully, such as with cmd-Q? If you instead just hit STOP on your IDE, you won't get a trace file.

### My traces appear empty

You probably went over the memory size that Perfetto is set to use by default (80MB). 

If you are doing intensive profiling with lots of functions being called many times, you'll probably want to increase this limit. You can do this by passing the number of `kb` you want to `beginSession`:

```
MelatoninPerfetto::get().beginSession(300000); # 300MB
```

### I keep forgetting to turn perfetto off!

Get warned by your tests if someone left `PERFETTO=1` on with a test like this (this example is Catch2):

```c++
TEST_CASE ("Perfetto not accidentally left enabled", "[perfetto]")
{
#if defined(PERFETTO) && PERFETTO
    FAIL_CHECK ("PERFETTO IS ENABLED");
#else
    SUCCEED ("PERFETTO DISABLED");
#endif
}
```

If you use perfetto regularly, you can also do what I do and check for `PERFETTO` in your plugin editor and display something in the UI:

<img width="384" alt="AudioPluginHost - 2023-01-06 44@2x" src="https://user-images.githubusercontent.com/472/211118327-e984f359-4e2f-4aec-8b4d-991093b36e67.png">

## Running Melatonin::Perfetto's tests

`melatonin_perfetto` includes a test suite using CTest. To run the tests, clone the code and run these commands:
```sh
cmake -B Builds
cmake --build Builds --config Debug
cd Builds
ctest -C Debug
```

The tests attempt to build two minimal CMake projects that depend on the `melatonin_perfetto` module; one tests
finding an install tree using `find_package()` and one tests calling `add_subdirectory()`. These tests serve to
verify that this module's packaging and installation scripts are correct, and that it can be successfully imported
to other projects using the methods advertised above. Another test case verifies that attempting to configure a 
project that adds `melatonin_perfetto` before JUCE will fail with the proper error message.
