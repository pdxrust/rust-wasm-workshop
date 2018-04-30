# Rust and WebAssembly Workshop

This is a Rust and WebAssembly workshop! It was originally held on
[May 2<sup>nd</sup>, 2018 at the PDXRust meetup](https://www.meetup.com/PDXRust/events/249474845/).

## Table of Contents

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->


- [Setup](#setup)
- [Hello World](#hello-world)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

## Setup

### Rust Toolchain

If you don't have `rustup` installed yet, run:

    curl https://sh.rustup.rs -sSf | sh

Next, use `rustup` to install the latest nightly and the
`wasm32-unknown-unknown` target:

    rustup component add nightly
    rustup update nightly
    rustup target add wasm32-unknown-unknown --toolchain nightly

Finally, install the `wasm-bindgen` CLI tool:

    cargo +nightly install -f wasm-bindgen-cli

### Node and `npm` Toolchain

I recommend using https://github.com/creationix/nvm to manage side-by-side
installations of different node versions, but if you already have node >= 8
installed, you can skip this.

    curl -o- https://raw.githubusercontent.com/creationix/nvm/v0.33.11/install.sh | bash
    source ~/.bashrc
    nvm install 9
    nvm use 9

## Hello World

Clone the project template:

    git clone https://github.com/pdxrust/rust-wasm-workshop.git --branch hello-world

Take a look at the contents of `index.js` and `src/lib.rs`. Then build the
project and run the development server!

    npm install
    npm run build-debug
    npm run serve

If you navigate to [http://localhost:8080](http://localhost:8080) in your
browser<sup>*</sup>, you should now see the hello-world alert dialog!

> ### <sup>*</sup> Caveat for Chrome Users
>
> Due to a [bug in Webpack][bug], this example does not yet work in Chrome. Once
> that bug is [fixed in upstream Webpack][fix] then this example will work in
> Chrome (like it does currently in, for example, Firefox).

[bug]: https://github.com/webpack/webpack/issues/6475
[fix]: https://github.com/webpack/webpack/pull/6709
