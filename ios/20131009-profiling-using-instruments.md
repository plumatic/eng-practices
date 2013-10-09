Profiling and Debugging
===============
(by [AshFurrow](http://github.com/AshFurrow))

# Instruments

Instruments is a tool that ships with Xcode that profiles your app for different characteristics. There are different profilers you can use from within Instruments – we'll cover the common ones later. For now, we need to make an important point.

Deferred Mode
----------------

Instruments profiles your app on the *device on which it is being profiled*. That means that if you profile an app running on a MacBook Pro, you're not going to get the same results as when you profile on an iPhone 4.

Additionally, Instruments collecting data on-device and transferring it to the computer carries an not-insignificant cost. It's possible to put Instruments into *deferred mode*, so that it will collect appropriate usage information on-device and wait until profiling has stopped to transfer it to the computer for analysis. A downside to using deferred mode is that you don't see real-time results (you have to wait until the profile has completed).

To enable deferred mode, open Instruments' preferences and check the top box "Always use deferred mode."

What's more, some profilers are available on one platform, but not the other. Play around – profiling is fun!

Traces
----------------

Data collected and interpreted by Instruments is stored in a file called a *trace*. These can be saved for later viewing. A trace can also contain more than run of a profiling session. To see all the runs in a trace, expand the left-arrow to the left of a profiler.

![](http://cloud.ashfurrow.com/image/2O352i3V2H3E/Screen%20Shot%202013-09-25%20at%204.30.21%20PM.png)

This is useful for verifying that a change to the code resulted in a desirable change to the run.

Time Profiler
----------------

The *Time Profiler* is one of the most useful profilers in Instruments. It is used to examine, on a per-thread basis, the activity occurring on the CPU.

![](http://f.cl.ly/items/0O1O3b0B08300x1X121o/Screen%20Shot%202013-09-25%20at%204.34.11%20PM.png)

This is graph of the CPU time spent executing code from your app. The bottom part of the screen displays a tree of every method invocation on a per-thread basis. The easiest way to find the heaviest (most expensive, time-wise) method stack trace on a thread is to open the *Extended Detail* pane.

![](http://f.cl.ly/items/2l2x1E1T2t0s0U3N3A1W/Screen%20Shot%202013-09-25%20at%204.36.42%20PM.png)

It's a wonder this isn't open by default. The Extended Detail pane shows the heaviest stack trace belonging to the currently selected leaf of the tree in the main view. System libraries are greyed out while your application code is in black.

By default, the Time Profiler displays information about the *entire duration* of the trace. To focus on a specific part of the trace, hold the ⎇ key and click-and-drag.

![](http://f.cl.ly/items/3m3B3S460N1y363s3623/Screen%20Shot%202013-09-25%20at%204.39.30%20PM.png)

As we can see here, the most expensive operation is the `LineHeightLabel`'s `sizeToFit` method invocation(s).

Core Animation Profiler
----------------

The Core Animation template contains two profilers: a *Core Animation Profiler* for measuring screen refresh rates and the Time Profiler from the last section. This tool is very useful when you notice dropped frame rates.

In general, it's desirable to maintain a screen refresh rate of 60 frames per second. That means that, in between screen refreshes, your app needs to spend fewer than 16 milliseconds executing.

There are also options on the left side of the window to color views that are drawn offscreen, etc.

*Note*: The Core Animation Profiler is only available on actual devices, *not* on the simulator.

Allocations Profiler
----------------

The *Allocations Profiler* is useful for measuring the total amount of memory used by an application. It contains two measurement tools: an allocations tool for measuring the total amount of memory in use by objects within an application, and a VM Tracker for measuring the total virtual memory in use by the system.

The VM Tracker is off by default and needs to be enabled (VM scans are expensive).

![](http://f.cl.ly/items/35003d0G0S13201R1i3n/Screen%20Shot%202013-09-25%20at%204.48.33%20PM.png)

A common and useful approach to using the Allocations Profiler is to perform some action, undo that action, and repeat. For example, tap a user name to push a new view controller onto the navigation stack, then tap the back button, and repeat. The memory use before the action is performed should be equivalent to the memory use after the action has been undone. This will help you detect memory leaks (often caused by reference cycles).

Another useful technique is to simulate a memory warning from the simulator to verify that your application responds accordingly by freeing up memory. Unfortunately, there is no (documented) way to simulate a memory warning on an actual device ([wink wink](http://stackoverflow.com/questions/12425720/a-way-to-send-low-memory-warning-to-app-on-iphone)).

Leaks Profiler
----------------

The *Leaks Profiler* helps you find memory leaks in your application code. It includes the Allocation Profiler (sans VM tracker).

![Look, I found a leak!](http://f.cl.ly/items/3B1q2e3Z0i3F3I0j3O1Q/Screen%20Shot%202013-09-25%20at%204.58.27%20PM.png)

Any leaks found are shown in the Leaks Profiler as red bars. Focus (⎇-click-and-drag) and select the Leaks profiler for more information.

Open the Extended Details pane and select the leaked object to see the stack trace of the where the leak was performed. In this example, it's a problem with `stringWithURIEncoded`.

![](http://f.cl.ly/items/1L0d282b1w1M402Q3A1k/Screen%20Shot%202013-09-25%20at%205.01.11%20PM.png)`

# Debugger

Xcode 5 ships with the LLDB debugger, which is automatically attached to a running application. You can add breakpoints to your project by clicking on the gutter to the right of your file.

![Breakpoint](http://f.cl.ly/items/090V0x2a280x093N3q0Q/Screen%20Shot%202013-09-25%20at%205.23.12%20PM.png)

You can see all of the breakpoints set in your current workspace with the *Breakpoints Navigator*.

![Breakpoints Navigator](http://f.cl.ly/items/2r2c1r0e2l2J0d3M0v3i/Screen%20Shot%202013-09-25%20at%205.24.28%20PM.png)

Active breakpoints are in dark blue and inactive breakpoints are ghosted out.

When your application hits a breakpoint, the app will pause and the debugger becomes active.

![](http://f.cl.ly/items/1k2K2A27360O1w2V2y0B/Screen%20Shot%202013-09-25%20at%205.25.37%20PM.png)

You can use the pane on the left to navigate objects, print their descriptions, etc. The console on the right side of the debugger pane, which can be shown and hidden with ⌘⇧Y, is the debugger. Here you can continue (`c`) execution until the next breakpoint, step to the next line (`n`), and print pointer descriptions (`p`).

If it can determine it, LLDB will cast structs to their type. So `p size` becomes `p (CGSize)size`. Sometimes, LLDB can't suss out what type to cast to, so you might have to do it yourself (i.e.: `p (CGSize)[someObj someMethodReturningSize]`).

You can also print out object descriptions with the `po` command: `po object` will call `object`'s `description` and print that out to the debugging console. This is useful for views because their description contains their frame.

The debugger can be used to invoke methods, like `p [obj method]`, or `p obj.property`.

The debugger doesn't have access to `enum`s or `#defines`. This makes this ... tricky sometimes. For example, if you've received data from an API and you want to turn it into an `NSString`, you'd typically use `[[NSString alloc] initWithData:data encoding:NSUTF8StringEncoding]`, but that encoding is an `enum` member, so you have to use `4`, instead.
