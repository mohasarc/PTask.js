# ptask.js

Helps running tasks based on priority without having to deal with a priority queue. additionally, provides a utility to pause, resume and abort task execution.

Using `ptask.js` you need to only create the tasks, assign them priority, and run them. `ptask.js` will take care of running them in order.

## Installing the package

### using npm

```js
npm i ptaskjs
```

### using yarn

```js
yarn add ptaskjs
```

## Usage

### Running tasks

You need to only create `PTask` objects; assign them a priority, a function to run, and a value to pass to the function; then call `PTask.run()` method. `PTask.run()` method returns a promise which will resolve to the output of the function passed to `onRun` option.

```js
import { PTask } from "ptaskjs";

// Prepare tasks
const task1 = new PTask<number, number>({
    priority: 1,
    onRun: async (a: number) => a, // The function to run
    args: 1, // The value to be passed to the function
});

const task2 = new PTask<number, number>({
    priority: 2,
    onRun: async (a: number) => a,
    args: 2,
});

const task3 = new PTask<number, number>({
    priority: 3,
    onRun: async (a: number) => a,
    args: 3,
});

const res = [];
const p2 = task2.run().then((val) => res.push(val));
const p1 = task1.run().then((val) => res.push(val));
const p3 = task3.run().then((val) => res.push(val));

Promise.all([p1, p2, p3]).then(() => {
    console.log(res); // [3, 2, 1]
});
```

### Pausing and resuming tasks

To be able to pause a task, the task needs to accept a second parameter. the second parameter will have methods to learn the current execution stage of the task, namely `paused` or `canceled`. The task also should have a mechanism to pause execution. An example would be
a task that is processing some array elements, it can check the current stage of execution before processing the next element.

```js
// The function to execute
const calculateSquares = async (nums: number[], execInfo: ExecInfo) => {
    const squares = [];
    const iter = async (i: number, num: number) => {
    if (i === nums.length) return;

    squares.push(num * num);
    await new Promise(r => setTimeout(r, 20));
    // Checking if the task was paused. If paused will stop execution
    // and return the results obtained so far.
    if (!execInfo.isPaused()) await iter(++i, nums[i]);
    };

    await iter(0, nums[0]);
    return squares;
};

const task = new PTask<number[], number[]>({
    args: [1, 2, 3, 4, 5, 6],
    priority: 1,
    onRun: calculateSquares,
    // This optional function will be called after the task was paused with the 
    // currently available arguments and the results obtained so far.
    // The return of this function will be used as the argument for the
    // calculateSquares function, when execution is resumed.
    onPause: (args, resSoFar) => {
        // If no results obtained so far, return the same arguments
        if (!resSoFar) return args;

        // Based on the length of the results array,
        // remove some items from the argumets
        return args.slice(resSoFar.length);
    },
    // When resuming the execution, calculateSquares will have a subset of
    // the original argument; hence will produce a subset of final result.
    // This function is used to merge the previously obtained results with the new subset
    resultsMerge: (resSoFar, newRes) => {
    if (!resSoFar) return newRes;
    return resSoFar.concat(newRes);
    }
});

task.run().then((res) => {
    console.log(res); // [1, 4, 9, 16, 25, 36]
});

setTimeout(async () => {
    await task.pause();
    task.resume();
}, 20);
```

### Canceling tasks

A task can be canceled only if it hasn't yet started execution using `PTask.cancel()`. When a task is canceled, `PTask.run()` will throw `'Task canceled'` error.

```js
const task = new PTask<void, void>({
    args: null,
    priority: 1,
    onRun: async () => await new Promise((resolve) => setTimeout(resolve, 500))
});

task.run().catch((err) => {
    console.log(err.message); // 'Task canceled'
});

task.cancel();
```

### Aborting running tasks

Using `PTask.cancel({abort: true})`, the task will be aborted even if it was running or paused. however, for running tasks, similar to pausing a task, a second parameter is needed to be able to learn the current execution stage of the task and cancel the function execution.

```js
let p1RunCount = 0;
let p2RunCount = 0;

const ptask1 = new PTask<void, void>({
    args: null,
    priority: 2,
    onRun: async (args, execInfo) => {
    const iter = async () => {
        await new Promise((resolve) => setTimeout(resolve, 100));
        // chacking if the task execution was canceled. If so return.
        if (!execInfo.isCanceled()) {
            p1RunCount++; // Increment the counter for the first task
            await iter();
        }
    }

    await iter();
    },
});

const ptask2 = new PTask<void, void>({
    args: null,
    priority: 1,
    onRun: async (args, execInfo) => {
    const iter = async () => {
        await new Promise((resolve) => setTimeout(resolve, 100));
        // chacking if the task execution was canceled. If so return.
        if (!execInfo.isCanceled()) {
            p2RunCount++; // Increment the counter for the second task
            await iter();
        }
    }

    await iter();
    },
});

// * Start executing both tasks
// The first will run first, because of higher priority value
// and the second will start after the abortion of the first one
ptask1.run().catch((err) => null);
ptask2.run().catch((err) => null);

// * Abort the first task after 1 second
setTimeout(() => {
    ptask1.cancel({abort: true});
}, 1000);

// * Check the iterations count of both tasks after 3 seconds
setTimeout(() => {
    console.log(p1RunCount < 10); // true
    console.log(p2RunCount > 10); // true

    ptask2.cancel({abort: true}); // abort the second task so it doesn't run for ∞
}, 3000);
```

`PTask.run()` will throw an error once task abortion is successfully complete. If the function passed to `onRun` doesn't implement a way to stop execution abrubtly, `PTask.run()` will not throw the abortion error untill the function's execution is complete; however, another task will be permited to start execution in the meantime.
