---
title: "Supercharge your frontend with Rust 🦀 and Wasm 🚀"
date: 2018-07-10T23:29:08+01:00
---
#### A quick guide to creating, packaging and using your first WebAssembly module with Rust, wasm-pack and friends...

I'm guessing if you're here you've already heard about [WebAssembly](https://webassembly.org/) and you just want to get started building something without reading pages of specification, complex explanations or stewing your brains in binary.

If you don't know what WebAssembly (AKA Wasm) is yet, it's a new low-level language that can be executed by modern browsers, that traditionally only ran JavaScript. Much like x86 assembly, most programmers aren't expected to write WebAssembly by hand. Instead we can use a higher-level language and compile it down to WebAssembly. A popular choice is [Rust](https://www.rust-lang.org/en-US/), in part this is because the community is putting [so much work](https://github.com/rustwasm/team) into creating great tooling to make this a pleasurable experience for developers.

To help you get your feet wet with Rust and Wasm I'm going to show you a super-simple example. You don't need to already know any Rust or have experience programming in it, but you will need need to [follow the instructions here to install the toolchain](https://rustwasm.github.io/book/setup.html).

We're going to build a factorial calculator that allows us to choose the input in the browser and will call into our Wasm module written in Rust to calculate the factorial and write it back to the browser, as shown below:

![factorial](https://mattharrison.s3.amazonaws.com/blog/factorial.gif)

Follow along with the code below or check out my code on [Github](https://github.com/mtharrison/wasmpack-example-factorial).

## Creating a WebAssembly module in Rust

First things first, we need to write the code that will comprise our WebAssembly module. To keep things simple, it's just going to be a single Rust function that computes the [factorial](https://en.wikipedia.org/wiki/Factorial) for a given number.

### Writing a simple Rust library

Assuming you have Rust installed, find a directory to work in and type:

```
cargo new --lib factorial
```

This will create a new directory to contain your Rust project:

```
factorial
├── Cargo.toml
└── src
    └── lib.rs
```

First we write our `factorial()` function. `factorial()` takes as an argument an unsigned 32-bit integer and returns an unsigned 32-bit integer.

#### `factorial/src/lib.rs`

```rust
pub fn factorial(n: u32) -> u32 {
    if n == 1 {
        return 1;
    }
    n * factorial(n - 1)
}

#[cfg(test)]
mod tests {
    use super::factorial;

    #[test]
    fn factorial_test() {
        assert_eq!(factorial(1), 1);
        assert_eq!(factorial(2), 2);
        assert_eq!(factorial(3), 6);
        assert_eq!(factorial(4), 24);
        assert_eq!(factorial(10), 3628800);
    }
}
```

I've also written some simple test cases for the `factorial()` function. Rust allows you to write your tests in the same file as implementation like this. the `#[cfg(test)]` annotation tells the Rust compiler to exclude this code from non-test builds.

We can go ahead and run the tests with `cargo test`:

```
running 1 test
test tests::factorial_test ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

Looking good! Our `factorial()` function appears to work as advertised. We _could_ stop here and compile this Rust library down to Wasm and use it directly. However, there are some tools that make our life easier.

### Glueing things together with Wasm-bindgen

Let's edit `Cargo.toml` which is a configuration file used by the Rust compiler:

#### `factorial/Cargo.toml`

```
[package]
name = "factorial"
version = "0.1.0"
authors = ["Matt Harrison <hi@matt-harrison.com>"]

[lib]
crate-type = ["cdylib"]

[dependencies]
wasm-bindgen = "0.2"
```

We've added the `crate-type = ["cdylib"]` option under `[lib]` which tells the Rust compiler that we're producing a dynamic library intended to be used from another language.

We've also added `wasm-bindgen = "0.2"` under `[dependencies]`. Wasm-bindgen is a project that helps people to glue together Rust/Wasm and JavaScript in a seamless way. We won't dig too much into this in this post but I plan to do some follow-ups showing exactly what the potential of wasm-bindgen is.

We'll need to add some more things to our `lib.rs` file now too:

#### `factorial/src/lib.rs`

```
// The annotation above loads some experimental features needed by wasm-bindgen

#![feature(proc_macro, wasm_import_module, wasm_custom_section)]

// This tells Rust that we're using the wasm_bindgen crate

extern crate wasm_bindgen;

// Include some code from the wasm_bindgen crate

use wasm_bindgen::prelude::*;

// Export this function to JavaScript bindings

#[wasm_bindgen]
pub fn factorial(n: u32) -> u32 {
    if n == 1 {
        return 1;
    }
    n * factorial(n - 1)
}

#[cfg(test)]
mod tests {
    use super::factorial;

    #[test]
    fn factorial_test() {
        assert_eq!(factorial(1), 1);
        assert_eq!(factorial(2), 2);
        assert_eq!(factorial(3), 6);
        assert_eq!(factorial(4), 24);
        assert_eq!(factorial(10), 3628800);
    }
}

```

### Leveraging npm with wasm-pack

Typically, whenever we want to package up some functionality written in JavaScript and reuse it on the frontend, we usually publish a package on npm and have it as a dependency in our frontend project. Wouldn't it be cool if we could so the same with a Wasm module too and just import it like any regular JS module? Well, you can do exactly that thanks to wasm-pack!

To get started, simply install wasm-pack and type `wasm-pack init .` inside the `factorial` directory. This might take a couple of minutes but by the end, if you look in the `pkg` folder, what you have is a package ready to publish on npm that contains a `package.json`, a compiled WebAssembly module and a JavaScript file that contains some glue to call into WebAssembly in an efficient manner.

We're done for now on the Rust and Wasm side. Let's jump into the frontend world!

## Integrating WebAssembly on the frontend with Webpack

I'm going to create a directory to hold the frontend, I'll call mine `frontend`. First thing I'll do is to run `npm init --yes` to create a `package.json`. I'm going to install `webpack` and `webpack-serve`. Webpack will take care of loading our WebAssembly module and bundling our scripts into a single js file that the browser can load. `webpack-serve` is a simple development server to serve up our files.

### Making our wasm module a dependency

We also want the wasm module that we wrote before to be a dependency of our frontend so we can call into the exported functions. I'm not going to go ahead and publish the package to npm but instead I'll make use of the file dependency feature in npm to reference the directory containing my pkg.

I'll also add a script that I can run later with `npm run serve` in order to use the webpack development server to serve the website files.

#### `frontend/package.json`
```
...
"dependencies": {
	"webpack": "^4.6.0",
	"webpack-serve": "^1.0.4",
	"factorial": "file:../factorial/pkg"
},
"scripts": {
	"serve": "webpack-serve --config ./webpack.config.js --content public"
}
...
```

Go ahead and run `npm install` to install those dependencies.

### Creating a simple UI

Next up let's build a very simple HTML page that will display our factorial app. It's going to consist of a basic HTML 5 boilerplate, a single `<div>` element with a `number` type `<input>` element that has a min of 1 and a max of 12. The reason for the max of 12 is so we don't overflow the 32-bit integers that we're working with in Rust. There's also an element to hold the output, and that's it.

#### `frontend/public/index.html`

```
<!DOCTYPE html>
<html lang="en">
<head>
	<meta charset="UTF-8">
	<meta name="viewport" content="width=device-width, initial-scale=1.0">
	<meta http-equiv="X-UA-Compatible" content="ie=edge">
	<title>Wasmpack Example - Factorial</title>
	<link rel="stylesheet" href="style.css">

</head>
<body>
	<div id="fac">
		<p>factorial(<input type="number" id="input" min="1" max="12" value="1">)</p>
		<p>=</p>
		<p><span id="output">1</span></p>
	</div>

	<script src="bundle.js"></script>
</body>
</html>
```

We'll want to add a bit of basic styling in a css file to make the page look nicer. I'm just going to center things and use a nice monospaced font, you know to make it look more technical ;)

#### `frontend/public/style.css`

```
body {
	padding: 0;
	margin: 0;
}

#fac {
	font-size: 60px;
	font-weight: bold;
	font-family: monospace;
	text-align: center;
	padding-top: 50px;
}

#input {
	height: 50px;
	width: 85px;
	border: 0;
	font-size: 60px;
	font-weight: bold;
	font-family: monospace;
	outline: 0;
}

#input::selection {
	background: none;
}

p {
	padding: 0px;
	margin: 0px;
}

input[type=number]::-webkit-inner-spin-button,
input[type=number]::-webkit-outer-spin-button {
   opacity: 1;
}
```

### Hooking up to WebAssembly

Finally we can get to the meat of our frontend. In the entrypoint JS file `main.js` we'll first want to import the WebAsssembly module from our `node_modules`. There's a slight gotcha here. Typically you can use the following syntax to import modules in esmodules style:

```
import * as factorial from 'factorial';
```

However this doesn't work right now with WebAssembly modules because they must be loaded and compiled asyncronously, instead we can use `import()` function which returns us a promise. I'll wrap the whole file in an `async` IIFE so I can use `await` syntax.

We're simply going to grab the input value from the `<input>` element on the page, parse it as an integer, then call into the Wasm module passing that input as an argument. We'll then take the return value and copy it into the output element.

We'll do that both on initial page load and anytime a new input is made.

#### `frontend/lib/main.js`

```
(async function () {

	const module = await import('factorial');

	const input = document.getElementById('input');
	const output = document.getElementById('output');

	const calculate = () => {

		const number = parseInt(input.value);
		const result = module.factorial(number);
		output.innerText = `${result}`;
	};

	// Calculate on load

	calculate();

	// Calculate on input

	input.addEventListener('input', calculate);

})();

```
### Configuring Webpack

We need to write a `webpack.config.js` file to tell Webpack how to bundle our code up. Our config is very simple, it says to take `lib/main.js` as the entry point and bundle that file and all its import, then to output a file at `public/bundle.js`. This is a file that we can then load into the browser using a `<script>` tag. Webpack will add the code required to load the WebAssembly module too, so we don't need to write that manually, pretty cool!

#### `frontend/webpack.config.js`

```
const path = require('path');

module.exports = {
	entry: './lib/main.js',
	output: {
		path: path.resolve(__dirname, 'public'),
		filename: 'bundle.js',
	},
	mode: 'production'
};
```

### Test drive

If you run `npm run serve` inside of your `frontend` folder and open up `http://localhost:8080` in the browser you should see everything working! If not, please post a comment below and I'll try to help you out :)


## Where next?

This article has only scratched the absolute surface of what's possible with Rust+Wasm. I'm going to be digging more into myself over the coming months and I'll keep blogging with more interesting examples.

In the meantime checkout some of these resources on Rust + Wasm:

- [The Rust Wasm book](https://rustwasm.github.io/book/)
- [Wasm Bindgen on Github](https://github.com/rustwasm/wasm-bindgen)
- [Wasm Pack on Github](https://github.com/rustwasm/wasm-pack)
- [Official WebAssembly site](https://webassembly.org/)
