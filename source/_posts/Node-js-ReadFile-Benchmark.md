---
title: Node.js ReadFile Benchmark
date: 2016-08-03 23:47:36
tags:
- Node
- Async
- Benchmark
- Sync
- I/O
---

For a lecture I wanted to show why Node.js is awesome. There are many reasons why node is a great tool, but the main point for me is that it embraces async & non-blocking processing. To show that behaviour I wanted to use an old school example of reading files. For this purpose I created a small bash-script which creates 10.000 files each with a size of 10kb in a `./files` subdirectory:
```sh
for i in $(seq 100); do dd if=/dev/zero of=./files/${i} bs=1M count=10 status=none; done
```

Next we just need to get the list of files with `fs.readdir` and iterate over it and read each single file synchronously and asynchrounisly.
## Scripts
#### Synchronous:

```js
let fs = require("fs");

//Synchronous
fs.readdir( "./files", function( error, files ) {
    for ( var i = 0; i < files.length; i++ ) {
        fs.readFileSync( "./files/" + files[i])
    };
});
```

#### Asynchronous:
```js
let fs = require("fs");

//Asynchronous
fs.readdir( "./files", function( err, files) {
    for ( var i = 0; i < files.length; i++ ) {
        fs.readFile( "./files/" + files[i], function( error, data ) {
        });
    }
});
```

We don't print anything or do anything else as it would just the inaccuracy of our benchmark. A single `console.log()` for example could block the whole scripts as it is most of the times synchronous.

**<span style="color: red">Attention</span>**: It depends on the runtime if `console.log()` is synchronous or asynchronous depends on the runtime. In our case we use node in the TTY in Linux and thanks to the [official documentation](https://nodejs.org/api/console.html#console_asynchronous_vs_synchronous_consoles) we know that it is blocking:
>The console functions are usually asynchronous unless the destination is a file. Disks are fast and operating systems normally employ write-back caching; it should be a very rare occurrence indeed that a write blocks, but it is possible.

>Additionally, console functions are blocking when outputting to TTYs (terminals) on OS X as a workaround for the OS's very small, 1kb buffer size. This is to prevent interleaving between stdout and stderr.

##  Benchmark

Now we are just running the scripts on our files and measuring the time with the unix builtin command `time`:
```
$ time node fileReadSync.js

real    0m0.200s
user    0m0.128s
sys    0m0.068s
```
```
$ time node fileReadAsync.js

real    0m0.627s
user    0m0.168s
sys    0m0.544s
```

Huh? Seems strange don't it!? Against our bet the the blocking `fileReadSync` seems to be faster than asynchronous & non-blocking `fileRead`. But how could that be? First I tried other combinations with 10 files each 100mb or 100 files each 10mb  and a few others but no matter what `fileReadSync` always was as fast as or up to 2 times faster than `fileRead`. After a few tries I took a guess and said that it has to be of my **SSD**. So I installed an old HDD into my system and retested it and watch this:
```
$ time node fileReadSync.js

real    0m4.203s
user    0m0.084s
sys    0m1.704s
```
```
$ time node fileReadParallel.js

real    0m1.823s
user    0m0.184s
sys    0m2.712s
```

It just worked like expected! The asynchronous example is faster than the synchronous one! So by implication the SSD is so fast that the overhead of scheduling the callbacks is much higher than just waiting for the files!

## Summary

`fileReadSync` is very very very fast on a good SSD and you probably don't need to care with asynchronous loading at server startup and all the callback hassle. Just load them synchronously and use your data! Nevertheless you should **always** use asynchronous `readFile` when processing data on request to not block any other requests! If someone has another opinion on my outcomes or my last recommendation please comment!
