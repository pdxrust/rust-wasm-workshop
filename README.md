# Rust and WebAssembly Workshop

This is a Rust and WebAssembly workshop! It was originally held on
[May 2nd, 2018 at the PDXRust meetup](https://www.meetup.com/PDXRust/events/249474845/).

## Table of Contents

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->


- [Setup](#setup)
  - [Rust Toolchain](#rust-toolchain)
  - [Node and `npm` Toolchain](#node-and-npm-toolchain)
- [Hello World](#hello-world)
- [Conway's Game of Life](#conways-game-of-life)
  - [Design](#design)
    - [Infinite Universe?](#infinite-universe)
    - [Representing Cells](#representing-cells)

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
[fix]: https://github.com/webpack/webpack/pull/7144

## Conway's Game of Life

We will be implementing [Conway's Game of Life][gol]. If you're unfamiliar with
it, you can follow that link to Wikipedia, or wait for the group to go through
the Game of Life intro slides together.

[gol]: https://en.wikipedia.org/wiki/Conway%27s_Game_of_Life

### Design

Before we dive into writing code, we have some design decisions to consider.

#### Infinite Universe?

The Game of Life is technically played in an infinite universe, but we do not
have infinite memory and compute power. Instead, we will create a fixed-size,
periodic universe, where cells on the edges have neighbors that wrap around to
the other side of the universe.

#### Representing Cells

We can represent the universe as a flat array, and has a byte for each cell. We
will use `0` for dead cells and `1` for alive cells.

Here is what a 4 by 4 universe looks like:

![Screenshot of a 4 by 4 universe](./4-by-4-universe.png)

To find the array index of the cell at a given row and column in the universe,
we can use this formula:

```text
index(row, column, universe) = row * width(universe) + column
```
