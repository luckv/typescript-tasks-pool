
Sorry for my terrible english, if you see any errors.

# Do work concurrently in NodeJS

A simple class available for public usage (it's a MIT license) to handle concurrent work in NodeJS. All the code is the file `tasks-pool.ts`.

The work is handled in form of `Promise<void>` and is called '*task*'. The errors of the rejected promise task's are printed to the logger.

## Features
 - Handling of arbitrary number of parallel tasks. The only limit is the system resources in which the application is running
 - Waiting for some tasks to complete
 - Waiting until the remaining number of tasks in execution are below a certain value
 - Generate tasks and put them in the pool, keeping a maximum number of parallel tasks in execution

## Why?
Because I have noted at my work that executing a big number of parallel tasks (over 1000), with I\O work, in NodeJS without an upper bound of parallel tasks, take down the system performance at managing all that tasks. So I had to write a solution to handle asynchronous executions, that keeps a limit on the number of parallel tasks.

With handsight, probably the root cause could have been the limited number of treads responsible for handling I\O work. NodeJS was already doing a great work at managing all the async tasks I launched at him. Anyway, I'll keep this code here to prove, to anyone who visits my profile, my skills and my dedication of solving problems.

## Taking advantage of the thread model of NodeJS and the use of `EventEmitter`

### Single thread model quickly explained
When we call an `async` function within our code in a NodeJS process, we are not really executing it the asynchronous way many people would think: in another thread, doing parallel work. Instead we are putting that execution at the end of a queue of tasks that the thread will execute in a sequential order, that structure is called the thread loop. Doing parallel work that includes I/O operations use, instead, some parallel threads different from the thread loop, but only for the I\O work (network, file system) of task.
This model was successful amid developers because it permits to avoid concurrency problems.

If you want to know more about the single thread model of NodeJS, I found this reading: https://www.zartis.com/nodejs-threads-and-the-event-loop/

### Taking advantage of the thread model
What is the big advantage of using the single thread model for this case? No concurrency, all the async code of the class would have been in execution without worrying about concurrent modifications of shared variables. So I used a private property of the class to keep track of the number of pending tasks in the pool. The property is incremented when a new task is added and decremented when the task finished, using the `finally()` handler of the promise, that will have been executed asynchronously.

### Use of `EventEmitter`
After having my tasks runnning, I had another problem to solve, how to wait until some tasks were completed? The simplest solution that came to my mind was to write an `async` method with a `while()` loop that used `setTimeout` wrapped in a promise. At every loop the code would have checked if the value of `pendingTasks` has changed, if yes the method returned, if not the method did another loop. That would have been a very inefficient solution, because at every loop it used a little portion of the cpu time of the thread loop, depriving other code of that time. Moreover it's not instantenous, because between the reaching of the target number of tasks and the termination of the waiting method, would have been passed an entire cicle of the `while()` loop, with cpu time, and real time, waste

The solution was to use an instance of the `EventEmitter` class, a private property, Every time a task was added or terminated, an event `"completed"` was fired by the event emitter with the number of pending tasks after the operation. The methods that wanted to wait for some tasks, would have to subscribe to the event emitter, wrap the usage in a promise, and when the number of tasks would have reached the desired number, the promise would have been resolved. In this way was totally possible to wait for an exact number of remaining tasks, and to obtain the desidered parallelization. More than one caller can invoke the methods of this class without any concurrency problems, the events are delivered to all the callers.

## The library

### Pass a custom logger
The class constructor accepts an optional object to be used as custom logger instead of `console`. Because the class only use the methods `debug` and `error` of the `console` type, the constructor parameter has type `Pick<typeof console, 'debug' | 'error'>`.

### Build javascript
The `build` script generate javascript code to be used in a codebase without typescript. Code is not minified and comments are not removed.
```
$ npm run build
```
