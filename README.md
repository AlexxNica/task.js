# task.js [![Build Status](https://img.shields.io/travis/icodeforlove/task.js.svg?branch=master)](https://travis-ci.org/icodeforlove/task.js) [![Code Climate](https://img.shields.io/codeclimate/github/icodeforlove/task.js.svg)](https://codeclimate.com/github/icodeforlove/task.js) [![CDNJS](https://img.shields.io/cdnjs/v/task.js.svg)](https://cdnjs.com/libraries/task.js)
This module is intended to make working with blocking tasks a bit easier, and is meant to work in node as well as the browser.

**[LIVE DEMO](http://s.codepen.io/icodeforlove/debug/ZOjBBB)**

## install

```
# node
npm install task.js

# usage
import Task from 'task.js';
```


or

```
# browser
bower install task
```

or just grab the [cdnjs hosted version](https://cdnjs.com/libraries/task.js) directly

## important

Before using this module I want to expose the current performance issues

- Node.js

	- When using task.js in node with very large messages it can be very slow, this is due to the fact that node is copying all the message data.

- Clientside
	- When using task.js in a browser that doesn't support transferables (or you don't use them properly) then you will notice a slow down when passing massive array buffers

Rule of thumb in node keep your object size under 150kb, and in the clientside version you can go crazy and send 40MB array buffers if transferables are supported.

## callbacks and promises

task.js supports both styles

```javascript
var powTask = task.wrap(pow);

// callbacks
powTask(2, function (error, result) {

});

// promises
powTask(2).then(
	function (result) {},
	function (error) {}
);
```

## task.defaults (optional)

You can override the defaults like this

```javascript
// overriding defaults (optional)
var myCustomTask = task.defaults({
	debug: false, // extremely verbose, you should also set maxWorkers to 1
	logger: console.log, // (default: console.log, this is useful if you want to parse all log messages)
	warmStart: false, // (default: false, if set to true all workers will be initialized instantly)
	maxWorkers: 4, // (default: the system max, or 4 if it can't be resolved)
	idleTimeout: 10000, // (default: false)
	idleCheckInterval: 1000 // (default: null)
	globals: {}, // refer to globals information
	initialize: function (globals) {return globals;} // refer to globals information
});
```

behind the scenes it's spreading your dynamic work across your cores

## task.wrap

You can wrap a function if the method signatures match, and it doesn't rely on any external variables.

```javascript
// non async
function pow(number) {
	return Math.pow(number, 2);
}

var powTask = task.wrap(pow);
powTask(2).then(function (squaredNumber) {
	console.log(squaredNumber);
});
```

But keep in mind that your function cannot reference anything inside of your current scope because it is running inside of a worker.

## task.decorate

You can use task decorate for class methods.

```javascript
class MathTask () {
	@task.decorate
	pow(num, pow) {
		return Math.pow(num, pow);
	}
}
```

## task.run

Below is an example of using a transferable

```javascript
// create buffer backed array
var array = new Float32Array([1,3,3,7]);

task.run({
    arguments: [array.buffer],
    transferables: [array.buffer], // optional, and only supported client-side
    // if the browser supports transferables `array` will get zeroed out because 
    // we flagged the buffer arg as transferable.
    function: function (buffer) {
    	// do intensive operations on your buffer
        return buffer;
    }
}).then(function (buffer) {
    // convert buffer back into original type
    console.log(new Float32Array(buffer)); // [1,3,3,7]
});
```

## task.terminate

When you run terminate it will destroy all current workers in the pool, and throw an error on all outstanding work.

```javascript
task.terminate();
```

## globals in workers

You can initialize a task instance to have predefined data from your main thread, or generated within the worker.

```javascript
var data = {one: 1};

task.defaults({
	globals: {
		data: data,
		two: 2,
		three: 3
	}
})

task.wrap(function () {
	console.log(globals.data);
	console.log(globals.two);
	console.log(globals.three);
});
```

The above works great for small data, but with larger data this doesn't work. This is where you can use the initialize property in defaults.

```javascript
task.defaults({
	globals: {
		data: {one: 1}
	},
	initialize: function (globals) {
		globals.bigData = [];
		for (var i = 0; i < 100000; i++) {
			globals.bigData.push(0);
		}
		return globals;
	}
});
```

You can also use initialize to define common methods in the worker scope as well.

```javascript
task.defaults({
	globals: {
		data: {one: 1}
	},
	initialize: function (globals) {
		globals.myFunc = function () {
			...
		}

		return globals;
	}
});
```

Keep in mind that it is ok to have a slow initialize, no work will actually be processed until there is a fully initialized worker.

## task.setGlobals

You can set globals across all of your workers like this

```javascript
task.setGlobals({
	data: {one: 1}
});
```

The above will kill all current workers, create new ones, and then assign the new workers the outstanding tasks.

## compatibility

[![Selenium Test Status](https://saucelabs.com/browser-matrix/task-js.svg)](https://saucelabs.com/u/task-js)
