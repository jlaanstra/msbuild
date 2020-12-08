# Managing tools with their own parallelism in MSBuild

MSBuild supports building projects in parallel using multiple processes. Most users opt into `NUM_PROCS` parallelism at the MSBuild layer.

In addition, tools sometimes support parallel execution. The Visual C++ compiler `cl.exe` supports `/MP[n]`, which parallelizes compilation at the translation-unit (file) level. If a number isn't specified, it defaults to `NUM_PROCS`.

When used in combination, `NUM_PROCS * NUM_PROCS` compiler processes can be launched, all of which would like to do file I/O and intense computation. This generally overwhelms the operating system's scheduler and causes thrashing and terrible build times.

As a result, the standard guidance is to use only one multiproc option: MSBuild's _or_ `cl.exe`'s. But that leaves the machine underloaded when things could be happening in parallel.

## Design

`IBuildEngine` will be extended to allow a task to indicate to MSBuild that it would like to consume more than one CPU core (`BlockingWaitForCore`). These will be advisory only—a task can still do as much work as it desires with as many threads and processes as it desires.

A cooperating task would limit its own parallelism to the number of CPU cores MSBuild can reserve for the requesting task.

All resources acquired by a task will be automatically returned when the task's `Execute()` method returns, and a task can optionally return a subset by calling `ReleaseCores`.

MSBuild will respect core reservations given to tasks for tasks that opt into resource management only. If a project/task is eligible for execution, MSBuild will not wait until a resource is freed before starting execution of the new task. As a result, the machine can be oversubscribed, but only by a finite amount: the resource pool's core count.

Task `Yield()`ing has no effect on the resources held by a task.

## Example

In a 16-process build of a solution with 30 projects, 16 worker nodes are launched and begin executing work. Most block on dependencies to projects `A`, `B`, `C`, `D`, and `E`, so they don't have tasks running holding resources.

Task `Work` is called in project `A` with 25 inputs. It would like to run as many as possible in parallel. It calls

TODO: what's the best calling pattern here? the thread thing I hacked up in the sample task seems bad.

```C#
int allowedParallelism = BuildEngine7.RequestCores(Inputs.Count); // Inputs.Count == 25
```

When `Work` returns, MSBuild automatically returns all resources reserved by the task to the pool. Before moving on with its processing, `Work2` calls `RequestCores` again, and this time receives a larger reservation.

## Implementation

The initial implementation of the system will use a Win32 [named semaphore](https://docs.microsoft.com/windows/win32/sync/semaphore-objects) to track resource use. This was the implementation of `MultiToolTask` in the VC++ toolchain and is a performant implementation of a counter that can be shared across processes.

On platforms where named semaphores are not supported (.NET Core MSBuild running on macOS, Linux, or other UNIXes), `RequestCores` will always return `0`. We will consider implementing full support using a cross-process semaphore (or an addition to the existing MSBuild communication protocol, if it isn't prohibitively costly to do the packet exchange and processing) on these platforms in the future.