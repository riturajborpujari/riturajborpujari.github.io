---
title         : "Competing with JSON.stringify - by building a custom one"
description   : >
                Building out a custom JSON serializer for fun, and testing
date          : 2024-08-14
template      : post.html
---

This came up during a discussion with my friend about Recursion. Why not build 
a Javascript `JSON.stringify` method as a recursive programming exercise? Seems like a great
idea. 

I quickly drafted out the first version. And it performed horribly! The 
**time required was about 4 times that of the standard** `JSON.stringify`.

## The first draft

```js
function json_stringify(obj) {
  if (typeof obj == "number" || typeof obj == "boolean") {
    return String(obj);
  }

  if (typeof obj == "string") {
    return `"${obj}"`;
  }

  if (Array.isArray(obj)) {
    return "[" + obj.map(json_stringify).join(",") + "]";
  }

  if (typeof obj === "object") {
    const properties_str = Object.entries(obj)
      .map(([key, val]) => {
        return `"${key}":${json_stringify(val)}`;
      })
      .join(",");
    return "{" + properties_str + "}";
  }
}
```

By running the following, we can see that our `json_stringify` works as
expected.

```js
const { assert } = require("console");
const test_obj = {
  name: "John Doe",
  age: 23,
  hobbies: ["football", "comet study"]
};

assert(json_stringify(test_obj) === JSON.stringify(test_obj))
```

To test more scenarios, and multiple runs to get an average idea of how our 
script runs, we made a simple testing script!

## A simple testing script

```js
function validity_test(fn1, fn2, test_values) {
  for (const test_value of test_values) {
    assert(fn1(test_value) == fn2(test_value));
  }
}

function time(fn, num_runs = 1, ...args) {
  const start_time = Date.now()

  for (let i = 0; i < num_runs; i++) {
    fn(...args);
  }

  const end_time = Date.now()
  return end_time - start_time
}


function performance_test(counts) {
  console.log("Starting performance test with", test_obj);

  for (const count of counts) {
    console.log("Testing", count, "times");
    
    const duration_std_json = time(JSON.stringify.bind(JSON), count, test_obj);
    console.log("\tStd lib JSON.stringify() took", duration_std_json, "ms");

    const duration_custom_json = time(json_stringify, count, test_obj);
    console.log("\tCustom json_stringify() took", duration_custom_json, "ms");
  }
}

const test_obj = {} // a deeply nested JS object, ommitted here for brevity 
const test_values = [
  12,
  "string test",
  [12, 34, 1],
  [12, true, 1, false],
  test_obj
];

validity_test(JSON.stringify, json_stringify, test_values);
performance_test([1000, 10_000, 100_000, 1000_000]);
```

Running this we get the timings like the following.
```shell
Testing 1000 times
	Std lib JSON.stringify() took 5 ms
	Custom json_stringify() took 20 ms
Testing 10000 times
	Std lib JSON.stringify() took 40 ms
	Custom json_stringify() took 129 ms
Testing 100000 times
	Std lib JSON.stringify() took 388 ms
	Custom json_stringify() took 1241 ms
Testing 1000000 times
	Std lib JSON.stringify() took 3823 ms
	Custom json_stringify() took 12275 ms
```

It might run differently on different systems but the ratio of the time taken 
by std `JSON.strngify` to that of our custom `json_stringify` should be about 
**1:3** - **1:4**

It could be different too in an interesting case. Read on to know more about 
that!

## Improving performance

The first thing that could be fixed is the use of `map` function. It creates
new array from the old one. In our case of objects, it is creating an array of
JSON stringified object properties out of the array containing object entries.

Similar thing is also happening with stringification of the array elements too.

We have to loop over the elements in an array, or the entries of an object! But
we can skip creating another array just to join the JSON stringified parts.

Here's the updated version (only the changed parts shown for brevity)

```js
function json_stringify(val) {
  if (typeof val === "number" || typeof val === "boolean") {
    return String(val);
  }

  if (typeof val === "string") {
    return `"${val}"`;
  }

  if (Array.isArray(val)) {
    let elements_str = "["

    let sep = ""
    for (const element of val) {
      elements_str += sep + json_stringify(element)
      sep = ","
    }
    elements_str += "]"

    return elements_str
  }

  if (typeof val === "object") {
    let properties_str = "{"

    let sep = ""
    for (const key in val) {
      properties_str += sep + `"${key}":${json_stringify(val[key])}`
      sep = ","
    }
    properties_str += "}"

    return properties_str;
  }
}
```

And here's the output of the test script now

```shell
Testing 1000 times
        Std lib JSON.stringify() took 5 ms
        Custom json_stringify() took 6 ms
Testing 10000 times
        Std lib JSON.stringify() took 40 ms
        Custom json_stringify() took 43 ms
Testing 100000 times
        Std lib JSON.stringify() took 393 ms
        Custom json_stringify() took 405 ms
Testing 1000000 times
        Std lib JSON.stringify() took 3888 ms
        Custom json_stringify() took 3966 ms
```

This looks a lot better now. Our custom `json_stringify` is taking only 3 ms
more than `JSON.stringify` to stringify a deep nested object 10,000 times.
Although this is not perfect, it is an acceptable delay.

## Squeezing out more??

The current delay could be due to all the string creation and concatenation 
that's happening. Every time we run `elements_str += sep + json_stringify(element)`
we are concatenating 3 strings.

Concatenating strings is costly because it requires
1. creating a new string buffer to fit the whole combined string
2. copy individual strings to the newly created buffer

By using a `Buffer` ourselves and writing the data directly there might give us
a performance improvement. Since we can create a large buffer (say 80 characters)
and then create new buffers to fit 80 characters more when it runs out.

We won't be avoiding the reallocation / copying of data altogether, but we will
reducing those operations.

Another possible delay is the recursive process itself! Specifically the
function call which takes up time. Consider our function call `json_stringify(val)`
which just has one parameter.

### Understanding Function calls
The steps would be
1. Push the return address to the stack
2. push the argument reference to the stack
3. In the called function
    1. Pop the parameter reference from the stack
    2. Pop the return address from the stack
    3. push the return value (the stringified part) onto the stack
4. In the calling function
    1. Pop off the value returned by the function from the stack

All these operations happen to ensure function calls happen and this adds CPU
costs.

If we create a non-recursive algorithm of `json_stringify` all these operations
listed above for function call (times the number of such calls) would be
reduced to none.

This can be a future attempt. 

## NodeJs version differences

One last thing to note here. Consider the following output of the test script

```shell
Testing 1000 times
        Std lib JSON.stringify() took 8 ms
        Custom json_stringify() took 8 ms
Testing 10000 times
        Std lib JSON.stringify() took 64 ms
        Custom json_stringify() took 51 ms
Testing 100000 times
        Std lib JSON.stringify() took 636 ms
        Custom json_stringify() took 467 ms
Testing 1000000 times
        Std lib JSON.stringify() took 6282 ms
        Custom json_stringify() took 4526 ms
```

Did our custom `json_stringify` just perform better than the NodeJs standard
`JSON.stringify`???

Well yes! But this is an older version of NodeJs (`v18.20.3`). Turns out, for
this version (and lower also perhaps) our custom made `json_stringify` works
faster than the standard library one!

All the tests for this article (except this last one) has been done with 
**Node v22.6.0**

The performance of JSON.stringify has increased from v18 to v22. This is so great

It is also important to note that, our script performed better in NodeJs v22.
So, it means, NodeJs has increased the overall performance of the runtime too.
Possibly an update has happened to the underlying **V8 engine** itself.

Well, this has been an enjoyable experience for me. And I hope it will be for
you too. And in the midst of all this enjoyment, we learnt a thing or two!

Keep building, keep testing!
