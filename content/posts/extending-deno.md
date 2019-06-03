---
title: "Understanding Deno: A dive into the internals"
date: 2019-05-29T19:00:00+01:00
---
I've recently been poking around with [Deno](https://github.com/denoland/deno) - the "secure JavaScript/TypeScript runtime built with V8, Rust, and Tokio". The reason being is that this lies at the intersection of a couple of my main current interests; JavaScript and Rust. I've been writing Node professionally now for around 5 years and Rust very _unprofessionally_ for just over a year.

Deno was created by Ryan Dahl, the also-creator of Node.js. Ryan introduced Deno to the world in a talk titled [10 things I regret about node.js](https://www.youtube.com/watch?v=M3BM9TB-8yA). Deno is his vision for a security-conscious modern successor to Node.js that stays truer to JavaScript's webiness and actually works with rather than against V8's inherent sandbox. I definitely recommend you listen to the full talk if you haven't already.

I'm planning to contribute to the Deno project because I think it will be a good way to learn more about Rust on a real-world project with considerable complexity. To dip my toes in before I try to pick up a real issue, I decided that I'd try to add a couple of dummy "features" to Deno. It was quite a fun and enlightening exploration so I wanted to share it with the hope that it will also be useful for any other would-be contributors to Deno in the future.

### Architecture of Deno

Many parallels can be drawn between Deno's architecture an how a Unix operating system is organised. In Deno, you have an unprivileged user-space, where code is written in Typescript and executed in a JavaScript sandbox by V8. To interact with the system, the user-code must dispatch messages (like system calls) to the privileged side, which is written in Rust and linked with V8. This comparison isn't just my own observation but actually stated in the [Deno manual](https://deno.land/manual.html):

<table>
<thead>
<tr>
<th style="text-align:right;"><strong>Linux</strong></th>
<th style="text-align:left;"><strong>Deno</strong></th>
</tr>
</thead>
<tbody>
<tr>
<td style="text-align:right;">Processes</td>
<td style="text-align:left;">Web Workers</td>
</tr>
<tr>
<td style="text-align:right;">Syscalls</td>
<td style="text-align:left;">Ops</td>
</tr>
<tr>
<td style="text-align:right;">File descriptors (fd)</td>
<td style="text-align:left;"><a href="#resources">Resource ids (rid)</a></td>
</tr>
<tr>
<td style="text-align:right;">Scheduler</td>
<td style="text-align:left;">Tokio</td>
</tr>
<tr>
<td style="text-align:right;">Userland: libc++ / glib / boost</td>
<td style="text-align:left;">deno_std</td>
</tr>
<tr>
<td style="text-align:right;">/proc/\$\$/stat</td>
<td style="text-align:left;"><a href="#metrics">Deno.metrics()</a></td>
</tr>
<tr>
<td style="text-align:right;">man pages</td>
<td style="text-align:left;">deno types</td>
</tr>
</tbody>
</table>

This can be contrasted to the situation we have in Node.js where the interface between JS and native code is a lot less contained. Native modules can punch holes into JavaScript from anywhere inserting functions and possibly security issues to the global scope of your scripts.

## Getting and building Deno

[First, be sure to follow the instructions here to download and build Deno on your machine.](https://deno.land/manual.html#buildfromsource)


## Adding a TypeScript feature to the userspace API

Let's start off easy, first let's add to the Deno TypeScript namespace that's available to all programs via `global.Deno`.

Let's add a new file under `js/number.ts` and add the following contents:

```js
export async function myNumber(): Promise<number> {
  return Promise.resolve(42);
}
```

We'll need to expose this on Deno namespace by adding the following to `js/deno.ts`:

```js
export { myNumber } from  "./number";
```

Rebuild Deno by running `./tools/build.py`. You might be surprised that we need to rebuild the `deno` executable after this change but Deno actually compiles the TypeScript in the core library into a JS bundle then builds all this code into the Deno binary as a V8 snapshot, hence why we need to recompile and link the main executable. This speeds up the startup of the Deno process which is great for users but slows down development a little.

_aside_:I think there might be a way to avoid actually rebuilding the binary and have the core library loaded and runtime but I couldn't figure out how to do this, if you know please shout me @mt_harrison.

Let's write a Deno script now to test our new core function, I'm going to put my example scripts inside `examples`.

```
mkdir examples
touch examples/number.ts
```

Inside `examples/number.ts` we add:

```js
async function main() {

  console.log(`The magic number is ${await Deno.myNumber()}`);
};

main();
```

Deno doesn't support top-level `await` yet so we wrap our code in a `async function main() {}` function to get around that.

We can test our script out now by running `./target/debug/deno run examples/number.ts`. All being well you should see:

```
The magic number is 42
```

This is all well and good but it's not as exciting as getting our paws muddied by writing some Rust, right? Agreed? Then onwards...

## Some Deno Rust <-> TS basics

Pure JavaScript in V8 has limited functionality, aside from the language features of course. Anything else that interacts with the operating system, such as networking, timers and process control needs to be provided by the host environment. That host environment in the case of Node.js is written in C++ and for Deno it's written in Rust.

First let's touch upon how we communicate between TS and Rust. Remember when I mentioned that [Node.js pokes holes into V8](https://github.com/nodejs/node/blob/master/src/node_process_object.cc#L165), injecting functions for each of the features it provides. Deno takes a different approach, instead it only pokes a single hole into V8 and provides a single function `dispatch()` that can be called from unpriviledged code. `dispatch()` accepts a message which may be one of many types, along with additional arguments and a buffer. `dispatch()` returns a message (or a `Promise` over a message). By using this very controlled method of message passing, Deno exerts much more restraint over the sandbox.

But what is this message? And isn't it expensive to keep passing them around? Rust uses [Flatbuffers](https://google.github.io/flatbuffers/) to encode typed messages which are passing between Rust and JS as byte arrays. The message fields can be read in place very efficiently with no (de)serialization required.

[insert diagram here]

## Adding a synchronous Rust feature to the privileged side

To add a new feature to the Rust side we'll need to define a new Flatbuffers message. Well two actually, one for the request and one for the response. To do this we open up `cli/msg.fbs` and add our new messages:

```
...

table MyPid {}

table MyPidRes {
  pid: uint32;
}

root_type Base
```

Node that our request message type `MyPid` has no fields because there's no arguments we need to pass along. The response message `MyPidRes` has just one, the pid as a `uint32`.

This file will be compiled with `flatc` - the flatbuffers compiler, into both Rust and TypeScript code which provides an API for reading and writing the Flatbuffers. If you've ever use Protocol Buffers and protoc before, this is very similar. We don't need to invoke `flatc` ourselves though as this is taken care of by the build system (which is [GN](https://chromium.googlesource.com/chromium/src/tools/gn/+/48062805e19b4697c5fbd926dc649c78b6aaa138/README.md) by the way).

Now we need to write the code that will receive the `MyPid` message and return the `MyPidRes` message in Rust and vice-versa in TypeScript. Let's start with the TypeScript side. We'll add this in `js/myPid.ts`:

```js
import * as flatbuffers from "./flatbuffers";
import * as msg from "gen/cli/msg_generated";
import { sendSync } from "./dispatch";
import { assert } from "./util";


export function myPid(): number {
  const builder = flatbuffers.createBuilder();
  const inner = msg.MyPid.createMyPid(builder);

  const baseRes = sendSync(builder, msg.Any.MyPid, inner)!;
  assert(msg.Any.MyPidRes === baseRes.innerType());
  const res = new msg.MyPidRes();
  assert(baseRes.inner(res) !== null);

  return res.pid();
}
```

A lot of the TypeScript core code has similar boilerplate to it: we first create a flatbuffers builder and then create an instance of the message type we want to dispatch, in this case `MyPid`. Then we call `sendSync` which is a wrapper around `Deno.core.dispatch()`. We get a generic response message back, which we convert to a specialized type by calling `message.inner(T)`.

We can then get at the fields of the response by calling methods corresponding to their names.

All that's left now is to write the Rust side of things. We'll do this in `cli/ops.rs`. Deno refers to it's commands between Rust and TS, or its syscalls or Ops. To handle the `MyPid` message we need to write a new op.

First we'll add a clause to the statement that matches on the type of the incoming message:

```rust
  msg::Any::Write => Some(op_write),
  msg::Any::MyPid => Some(op_my_pid),       // we're adding this
  msg::Any::Resolve => Some(op_resolve),
```

We need to write the function `op_my_pid()` too:

```rust
fn op_my_pid(
  _state: &ThreadSafeState,
  base: &msg::Base<'_>,
  _data: Option<PinnedBuf>,
) -> Box<OpWithError> {
  let builder = &mut FlatBufferBuilder::new();
  let inner = msg::MyPidRes::create(
    builder,
    &msg::MyPidResArgs {
      pid: std::process::id(),
    },
  );

  ok_future(serialize_response(
    base.cmd_id(),
    builder,
    msg::BaseArgs {
      inner: Some(inner.as_union_value()),
      inner_type: msg::Any::MyPidRes,
      ..Default::default()
    },
  ))
}
```

There's quite a lot going on here so I'll explain things line by line...

## Adding an asynchronous Rust feature to the privileged side...

## Writing some tests...

coming soon...nobody is actually going to read this, right?
