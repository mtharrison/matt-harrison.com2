---
title: "An introduction to hacking on Deno"
date: 2019-05-29T19:00:00+01:00
---

I've recently been playing around with [Deno](https://github.com/denoland/deno) - the "secure JavaScript/TypeScript runtime built with V8, Rust, and Tokio". The reason being is that this lies at the intersection of a couple of my main interests: JavaScript and Rust. I've been writing JS professionally now for around 5 years and Rust very _unprofessionally_ for just over a year.

Deno was created by Ryan Dahl, the also-creator of Node.js. Ryan introduced Deno to the JS world in a talk titled [10 things I regret about node.js](https://www.youtube.com/watch?v=M3BM9TB-8yA) at JSConf EU 2018. Deno is his vision for a security-conscious modern successor to Node.js that stays truer to JavaScript's web heritage and works with rather against the grain of V8's sandbox. I definitely recommend you listen to the full talk if you haven't already.

I'm hoping to contribute to the Deno project because I think it will be a good way to learn more about Rust on an actual project with real-world complexity, it also seems like a lot of fun. To dip my toes in before I try to pick up a real issue, I decided that I'd try to add a couple of dummy "features" to Deno. It was quite a fun and enlightening exploration so I wanted to share it with the hope that it will also be useful for any other would-be contributors to Deno in the future.

## Architecture of Deno

Many parallels can be drawn between Deno's architecture and how a Unix operating system is organised. In Deno, there is an unprivileged user space, where code is written in TypeScript then compiled and executed in a JavaScript sandbox by V8. To interact with the system, the user space code must dispatch messages (like OS system calls) to the privileged side, which is written in Rust and linked in with V8. This comparison isn't just my own observation but actually stated in the [Deno manual](https://deno.land/manual.html):

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

This can be contrasted to the situation we have in Node.js where the interface between JS and native code is a lot less, contained, shall we say. Native modules can punch holes into JavaScript from anywhere, inserting functions which have access to your system into the global scope of your scripts. Suddenly the code running in your sandbox isn't really all that sandboxed at all. The benefits of Deno's approach is that it's easier to keep track of what your programs are doing by using a single system call interface, including the resources they're using like files and sockets. It also makes it easy to apply capability-based permissions on executing scripts. Deno defaults scripts to having no access to things like network, host filesystem etc. You must give these permissions explicitly by passing flags e.g. `--allow-net` and `--allow-write`.

Security approach isn't the only difference between Node and Deno, [the manual](https://deno.land/manual.html) is a great place to learn more.

Deno provides the same asynchronous non-blocking I/O as default that Node.js does. Where Node.js relies on [libuv](https://github.com/libuv/libuv), Deno makes uses of the evolving async I/O stack in Rust composed of [Tokio](https://github.com/tokio-rs/tokio) and [Mio](https://github.com/tokio-rs/mio).

## Getting and building Deno

[First, be sure to follow the instructions here to download and build the Deno source code on your machine.](https://deno.land/manual.html#buildfromsource)


## Adding a TypeScript feature to the userspace API

Let's start off easy, first let's add to the global `Deno` namespace that's available to all programs via `global.Deno`.

Let's add a new file under `deno/js/number.ts` and add the following contents:

```js
export async function myNumber(): Promise<number> {
  return Promise.resolve(42);
}
```

We'll need to expose this on `Deno` namespace by adding the following to `deno/js/deno.ts`:

```js
export { myNumber } from  "./number";
```

Rebuild Deno by running `./tools/build.py`. You might be surprised that we need to rebuild the `deno` executable after this change because we only changed TypeScript. Deno actually pre-compiles the TypeScript in the core library into a JS bundle then builds all this code into the Deno binary as a [V8 snapshot](https://v8.dev/blog/custom-startup-snapshots), hence why we need to recompile and link the main executable. This speeds up the startup of the Deno process which is great for users but can slow down development a little.

_aside_:I think there might be a way to avoid actually rebuilding the binary and have the core library loaded and runtime via some build options but I couldn't figure out how to do this, if you know please shout me @mt_harrison.

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

Pure JavaScript executing in V8 has limited functionality, aside from the built-in language features of course. Anything else that interacts with the operating system, such as networking, timers and process control needs to be provided by the host environment. That host environment in the case of Node.js is written in C++ and for Deno it's written in Rust.

First let's touch upon how we communicate between TS and Rust. Remember when I mentioned that [Node.js pokes holes into V8](https://github.com/nodejs/node/blob/master/src/node_process_object.cc#L165), injecting functions for each of the features it provides. Deno takes a different approach, instead it only pokes a single hole into V8 and provides a single function `dispatch()` that can be called from unpriviledged code. `dispatch()` accepts a message which may be one of many types, along with additional arguments and a buffer. `dispatch()` returns a message (or a `Promise` over a message). By using this very controlled method of message passing, Deno exerts much more restraint over the sandbox.

But what is this message? And isn't it expensive to keep passing them around? Rust uses [Flatbuffers](https://google.github.io/flatbuffers/) to encode typed messages which are passed between Rust and JS as byte buffers. The message fields can then be read in place very efficiently with no (de)serialization required.

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

A lot of the TypeScript core code has similar boilerplate to it: we first create a flatbuffers builder and then create an instance of the message type we want to dispatch, in this case `MyPid`. Then we call `sendSync()` which is a wrapper around `Deno.core.dispatch()` which actually does the inter-language communication. We get a generic response message back, which we convert to a specialized type by calling `message.inner(T)`.

We can then get at the fields of the response by calling methods corresponding to their names.

All that's left now is to write the Rust side of things. We'll do this in `cli/ops.rs`. Deno refers to its bindings between Rust and TS, or its syscalls as Ops. To handle the `MyPid` message we need to write a new Op.

First we'll add a clause to the `match` statement that matches on the type of the incoming message:

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

```rust
1. fn op_my_pid(
2.   _state: &ThreadSafeState,
```

Every op receives this `state` parameter, a shared reference to a `ThreadSafeState`. This is a struct defined as:

```rust
pub struct ThreadSafeState(Arc<State>);
```

We can see it's a struct that wraps a `State` struct in an [`Arc`](https://doc.rust-lang.org/std/sync/struct.Arc.html) so it can be shared accross threads. `State` definition looks like:

```rust
// Isolate cannot be passed between threads but ThreadSafeState can.
// ThreadSafeState satisfies Send and Sync.
// So any state that needs to be accessed outside the main V8 thread should be
// inside ThreadSafeState.
#[cfg_attr(feature = "cargo-clippy", allow(stutter))]
pub struct State {
  pub dir: deno_dir::DenoDir,
  pub argv: Vec<String>,
  pub permissions: DenoPermissions,
  pub flags: flags::DenoFlags,
  ... // clipped
```

The comment is quite clear here. We can't pass the V8 Isolate between threads so we have this thread safe container for all the rest of the state that our program might need to access, such as the permissions, flags and arguments passed to the process.

Ok, let's look at the next part:

```rust
3.   base: &msg::Base<'_>,
4.   _data: Option<PinnedBuf>,
```

The `base` parameter is just a reference to the generic flatbuffers message we received, which we'll see how to turn it into a specialized instance later. `data` is an `Option<PinnedBuf>`. `data` is basically a writable buffer of bytes that has been borrowed from JavaScript. We'd use this if we wanted to efficiently transfer raw bytes to JavaScript, for instance `data` is used in `read/write` Ops. In this example we've prefixed `data` and `state` with `_` as we're not using them.

```rust
5.  ) -> Box<OpWithError> {
6.    let builder = &mut FlatBufferBuilder::new();
7.    let inner = msg::MyPidRes::create(
8.      builder,
9.     &msg::MyPidResArgs {
10.       pid: std::process::id(),
11.    },
12.  );
13.
14.  ok_future(serialize_response(
15.    base.cmd_id(),
16.    builder,
17.    msg::BaseArgs {
18.      inner: Some(inner.as_union_value()),
19.      inner_type: msg::Any::MyPidRes,
20.      ..Default::default()
21.    },
22.  ))
23.}
```

The op function returns a `Box<OpWithError>` which is defined as:

```rust
pub type OpWithError = dyn Future<Item = Buf, Error = DenoError> + Send;
```

So it can be anything that implements the `Future` trait where it's associated `Item` is a `Buf` and `Error` is a `DenoError`. `Buf` is just a slice of bytes which will contain a flatbuffer:

```rust
pub type Buf = Box<[u8]>;
```

Inside the op we write code quite similar to that which we wrote in TS. We build a `MyPidRes` message with a `pid` field which we get from the Rust stdlib function `std::process::id()`. Then we serialze the flatbuffer into a `Buf` and return an ok future. Note that this was all blocking.

Next we'll implement an op which will actually take some time, so needs to wire up into Tokio a bit more and return a `Promise` on the TypeScript side.

## Adding an asynchronous Rust feature to the privileged side

This time we're going to write a function `resolve()` in TypeScript that will resolve a hostname for us to an IP address using DNS. The signature will be:

```js
async function resolve(hostname: string): Promise<string>;
```

First step is to implement the TS side. This looks very similar to the synchronous example from before. The only differences are that this time:

- We use `syncAsyc()` instead of `sendSync()`
- We're returned a `Promise` over a message instead of the message itself

```js
import * as flatbuffers from "./flatbuffers";
import * as msg from "gen/cli/msg_generated";
import { sendAsync } from "./dispatch";
import { assert } from "./util";


export async function resolve(hostname: string): Promise<string> {
  const builder = flatbuffers.createBuilder();
  const host = builder.createString(hostname);
  const inner = msg.Resolve.createResolve(builder, host);

  const baseRes = await sendAsync(builder, msg.Any.Resolve, inner)!;
  assert(msg.Any.ResolveRes === baseRes.innerType());
  const res = new msg.ResolveRes();
  assert(baseRes.inner(res) !== null);

  return res.ip()!;
}
```

On the Rust side, things aren't too different either. Because there's no asyncronous support for DNS in Tokio, we've used `tokio_threadpool::blocking()`. This is a function that will take a blocking closure returning a `Future` and execute that closure in a thread pool returning a `Future` which resolves when the blocking closure returns. It's very useful for wrapping sync code.

```rust
use std::net::ToSocketAddrs;

fn op_resolve(
  _state: &ThreadSafeState,
  base: &msg::Base<'_>,
  _data: Option<PinnedBuf>,
) -> Box<OpWithError> {
  let inner = base.inner_as_resolve().unwrap();
  let cmd_id = base.cmd_id();
  let hostname = String::from(inner.hostname().unwrap());

  blocking(base.sync(), move || -> OpResult {

    let mut addrs_iter = hostname.to_socket_addrs().unwrap();
    let ip_addr = addrs_iter.find(|addr| addr.is_ipv4()).unwrap();
    let ip = format!("{}", ip_addr);

    let builder = &mut FlatBufferBuilder::new();
    let ip_fbs_string = builder.create_string(&ip);
    let inner = msg::ResolveRes::create(
      builder,
      &msg::ResolveResArgs { ip: Some(ip_fbs_string) },
    );

    Ok(serialize_response(
      cmd_id,
      builder,
      msg::BaseArgs {
        inner: Some(inner.as_union_value()),
        inner_type: msg::Any::ResolveRes,
        ..Default::default()
      },
    ))
  })
}
```


