# `tokio_with_wasm`

[![Crates.io](https://img.shields.io/crates/v/tokio_with_wasm.svg)](https://crates.io/crates/tokio_with_wasm)
[![Documentation](https://docs.rs/tokio_with_wasm/badge.svg)](https://docs.rs/tokio_with_wasm)
[![License](https://img.shields.io/crates/l/tokio_with_wasm.svg)](https://github.com/cunarist/tokio-with-wasm/blob/main/LICENSE)

![Recording](https://github.com/cunarist/tokio-with-wasm/assets/66480156/77fa5838-23c7-4e3b-b1ba-61146972c2aa)

> Tested with [Rinf](https://github.com/cunarist/rinf)

`tokio_with_wasm` is a Rust library that provides `tokio` specifically designed for web browsers. It aims to provide the exact same `tokio` features for web applications, leveraging JavaScript web API.

This library is made up of JavaScript glue code that mimics the behavior of real `tokio`. Because `tokio_with_wasm` doesn't have its own runtime and adapts to the JavaScript event loop, advanced features of `tokio` might not work.

When using `spawn_blocking()`, the number of web workers are automatically adjusted adapting to the number of parallel tasks. Refer to the docs for additional details.

This library assumes that you're compilng your Rust project with `wasm-pack` and `wasm-bindgen`, which currently uses `wasm32-unknown-unknown` Rust target. Note that this library currently only supports the `web` target of `wasm-bindgen`, not [others](https://rustwasm.github.io/wasm-bindgen/reference/deployment.html) such as `no-modules`.

## Features

- **Familiar API**: If you're familiar with `tokio`, you'll feel right at home with `tokio_with_wasm`. It provides similar functionality and follows the same patterns for spawning and managing asynchronous tasks.

- **Web Worker Integration**: `tokio_with_wasm` adapts to the JavaScript environment by utilizing web API under the hood. This means you can write Rust code that runs concurrently and efficiently in web applications.

- **Spawn Async and Blocking Tasks**: You can spawn both asynchronous and blocking tasks. Asynchronous tasks allow you to perform non-blocking operations, while blocking tasks are suitable for compute-heavy or synchronous tasks.

> Though various IO functionalities can be added in the future, they're not included yet.

## Usage

Add this library to your `Cargo.toml` alongside `tokio`:

```toml
[dependencies]
tokio = { version = "0.0.0", features = ["rt"] }
tokio_with_wasm = { version = "0.0.0", features = ["rt"] }
```

Here's a simple example of using `tokio_with_wasm` that works on both native platforms and web browsers:

```rust
use tokio::task::{spawn, spawn_blocking, yield_now, JoinSet};
use tokio::time::{interval, sleep};
use tokio_with_wasm::alias as tokio;

#[tokio::main(flavor = "current_thread")]
async fn main() {
    let async_join_handle = spawn(async {
        // Asynchronous code here.
        // This will run concurrently
        // in the same web worker(thread).
    });
    let blocking_join_handle = spawn_blocking(|| {
        // Blocking code here.
        // This will run parallelly
        // in the external pool of web workers.
    });
    let async_result = async_join_handle.await;
    let blocking_result = blocking_join_handle.await;
    for i in 1..=1000 {
        // Some repeating task here
        // that shouldn't block the JavaScript runtime.
        yield_now().await;
    }
}
```

The `use tokio_with_wasm::alias as tokio;` statement is functionally equivalent to the code below. This import is provided for convenience and to allow for shorter code.

```rust
#[cfg(all(
    target_arch = "wasm32",
    target_vendor = "unknown",
    target_os = "unknown"
))]
use tokio_with_wasm as tokio;

#[cfg(not(all(
    target_arch = "wasm32",
    target_vendor = "unknown",
    target_os = "unknown"
)))]
use tokio;
```

## Documentation

API documentation can be found on [docs.rs](https://docs.rs/tokio_with_wasm).

## Caution

Keep in mind that you should NEVER write panicking code.

On `wasm32-unknown-unknown`, there's currently [no way](https://rustwasm.github.io/wasm-bindgen/api/wasm_bindgen_futures/fn.future_to_promise.html#panics) to catch and unwind panics like on native platforms. Panics will eventually lead to leaked JavaScript `Promise`s.

Stick to the `Result` enum whenever possible.

## Building and Deploying

If you're using Web Workers (threads) by calling `spawn_blocking`, you need to set specific Rust compiler flags. Also, you must use the `nightly` toolchain and include certain Rust standard library components in the compilation.

- `target-feature` flags
  - `+atomics`
  - `+bulk-memory`
  - `+mutable-globals`
- `build-std` components
  - `std`
  - `panic_abort`

Here's a full example command:

```shell
export RUSTFLAGS="-C target-feature=+atomics,+bulk-memory,+mutable-globals"
export RUSTUP_TOOLCHAIN="nightly"
wasm-pack build <path> --target web -- -Z build-std=std,panic_abort
```

After building your webassembly module and preparing it for deployment, ensure that your web server is configured to include cross-origin-related HTTP headers in its responses. These headers enable clients using your website to gain access to `SharedArrayBuffer` web API, which is something similar to shared memory on the web.

- [`Cross-Origin-Opener-Policy`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Cross-Origin-Opener-Policy): `same-origin`
- [`Cross-Origin-Embedder-Policy`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Cross-Origin-Embedder-Policy): `require-corp`

Additionally, don't forget to specify the MIME type `application/wasm` for `.wasm` files within the server configurations to ensure optimal performance.

## Why This is Needed

The web has many restrictions due to its sandboxed environment which prevents the use of threads, time, file IO, network IO, and many other native functionalities. Consequently, certain features are missing from Rust's `std` due to these limitations. That's why `tokio` doesn't really work well on web browsers.

To address this issue, this crate offers `tokio` modules with the **same names** as the original native ones, providing workarounds for these constraints.

## Future Vision

Because a large portion of Rust's web ecosystem is based on `wasm32-unknown-unknown` right now, we had to make an alias crate of `tokio` to use its functionalities directly on the web.

Hopefully, when `wasm32-wasi` becomes the mainstream Rust target for the web, [`jco`](https://github.com/bytecodealliance/jco) might be an alternative to `wasm-bindgen` as it can provide full `std` functionalities with browser shims (polyfills). However, this will take time because the [`wasi-threads`](https://github.com/WebAssembly/wasi-threads) proposal still has a long way to go.

Until that time, there's `tokio_with_wasm`!

## Contribution Guide

Contributions are always welcome! If you have any suggestions, bug reports, or want to contribute to the development of `tokio_with_wasm`, please open an issue or submit a pull request.

There are situations where you cannot use native Rust code directly on the web. This is because `wasm32-unknown-unknown` Rust target used by `wasm-bindgen` doesn't have a full `std` module. Refer to the links below to understand how to interact with JavaScript with `wasm-bindgen`.

- https://rustwasm.github.io/wasm-bindgen/reference/attributes/on-js-imports/js_name.html
- https://rustwasm.github.io/wasm-bindgen/reference/attributes/on-js-imports/js_namespace.html

It is possible for rust code to be called in a **web worker**. Therefore, we cannot access the global `window` JavaScript object
just like when you work in the main thread of JavaScript. Refer to the link below to check which web APIs are available in a web worker.
You'll be surprised by various capabilities that modern JavaScript has.

- https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API/Functions_and_classes_available_to_workers

Please note that this library uses a quite hacky and naive approach to mimic native `tokio` functionalities. That's because this library is regarded as a temporary solution for the period before `wasm32-wasi`. Any kind of PR is possible, as long as it makes things just work on the web.
