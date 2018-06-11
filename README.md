# Rust and WebAssembly Workshop

This is a Rust and WebAssembly workshop! It was originally held on
[May 2nd, 2018 at the PDXRust meetup](https://www.meetup.com/PDXRust/events/249474845/).

--------------------------------------------------------------------------------

## Table of Contents

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->


- [Setup](#setup)
  - [Rust Toolchain](#rust-toolchain)
  - [Node and `npm` Toolchain](#node-and-npm-toolchain)
- [Hello World](#hello-world)
  - [Running Tests](#running-tests)
- [Conway's Game of Life](#conways-game-of-life)
  - [Design](#design)
    - [Infinite Universe?](#infinite-universe)
    - [Representing Cells](#representing-cells)
  - [Defining `Cell` and `Universe`](#defining-cell-and-universe)
    - [Testing the `Universe`'s `Display` Implementation](#testing-the-universes-display-implementation)
  - [Rendering to the DOM with JavaScript](#rendering-to-the-dom-with-javascript)
  - [Computing the Next Generation](#computing-the-next-generation)
    - [Testing `Universe::tick`](#testing-universetick)
  - [Rendering Each New Generation Inside `requestAnimationFrame`](#rendering-each-new-generation-inside-requestanimationframe)
  - [Rendering to `<canvas>` Directly from Memory](#rendering-to-canvas-directly-from-memory)
  - [More Exercises](#more-exercises)
- [The End](#the-end)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->

--------------------------------------------------------------------------------

## Setup

### Rust Toolchain

If you don't have `rustup` installed yet, run:

    curl https://sh.rustup.rs -sSf | sh

Next, use `rustup` to install the latest nightly and the
`wasm32-unknown-unknown` target:

    rustup toolchain add nightly
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
    cd rust-wasm-workshop

Take a look at the contents of `index.js` and `src/lib.rs`. Then build the
project and run the development server!

    npm install
    npm run build-debug
    npm run serve

If you navigate to [http://localhost:8080](http://localhost:8080) in your
browser<sup>*</sup>, you should now see the hello-world alert dialog!

You can interrupt the server and get your shell prompt back just by hitting
control-C.

[bug]: https://github.com/webpack/webpack/issues/6475
[fix]: https://github.com/webpack/webpack/pull/7144

### Running Tests

Add this test to `src/lib.rs`:

```rust
#[test]
fn two_plus_two() {
    assert_eq!(2 + 2, 4);
}
```

Now run `cargo +nightly test`. You should see this:

```
   Compiling hello_world v0.1.0 (file:///home/fitzgen/rust-wasm-workshop)
    Finished dev [unoptimized + debuginfo] target(s) in 0.88 secs
     Running target/debug/deps/hello_world-d8a1e575b45635c6

running 1 test
test two_plus_two ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

It is important that you familiarize yourself with running such tests. We will
use unit tests to determine whether your implementation for a section is working
or not, and therefore whether it is OK to start the next section or not.

--------------------------------------------------------------------------------

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

### Defining `Cell` and `Universe`

Let's begin by removing the `alert` import, `greet` function, and `two_plus_two`
test from `src/lib.rs`, and replacing them with a type definition for cells:

```rust
#[repr(u8)]
#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub enum Cell {
    Dead = 0,
    Alive = 1,
}
```

Next, let's define the universe. The universe has a width and a height, and a
vector of cells of length `width * height`.

```rust
#[wasm_bindgen]
pub struct Universe {
    width: u32,
    height: u32,
    cells: Vec<Cell>,
}
```

Let's define a constructor that initializes the universe with an interesting
initial pattern of live and dead cells:

```rust
/// Public methods, exported to JavaScript.
#[wasm_bindgen]
impl Universe {
    pub fn new() -> Universe {
        let width = 64;
        let height = 64;

        let cells = (0..width * height)
            .map(|i| {
                if i % 2 == 0 || i % 7 == 0 {
                    Cell::Alive
                } else {
                    Cell::Dead
                }
            })
            .collect();

        Universe {
            width,
            height,
            cells,
        }
    }
}
```

The state of the universe is represented as a vector of cells. To make this
human readable, let's implement a basic text renderer. The idea is to write the
universe line by line as text, and for each cell that is alive, print the
unicode character `◼️` ("black medium square"). For dead cells, we'll print `◻️`
(a "white medium square"). After each row, a newline should be printed.

By implementing the [`Display`] trait from Rust's standard library, we can add a
way to format a structure in a user-facing manner. This will also automatically
give us a [`to_string`] method.

[`Display`]: https://doc.rust-lang.org/1.25.0/std/fmt/trait.Display.html
[`to_string`]: https://doc.rust-lang.org/1.25.0/std/string/trait.ToString.html

```rust
use std::fmt;

impl fmt::Display for Universe {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        unimplemented!("insert code here")
    }
}
```

Next, let's expose the human-readable text to JavaScript, by adding a `render`
method to the `impl Universe`:

```rust
/// Public methods, exported to JavaScript.
#[wasm_bindgen]
impl Universe {
    // ...

    pub fn render(&self) -> String {
        self.to_string()
    }
}
```

#### Testing the `Universe`'s `Display` Implementation

Add this test to `src/lib.rs`:

```rust
#[test]
fn universe_displays_correctly() {
    let universe = Universe {
        width: 4,
        height: 4,
        cells: vec![
            Cell::Dead,  Cell::Dead,  Cell::Dead,  Cell::Dead,
            Cell::Dead,  Cell::Dead,  Cell::Dead,  Cell::Alive,
            Cell::Dead,  Cell::Dead,  Cell::Alive, Cell::Alive,
            Cell::Dead,  Cell::Alive, Cell::Alive, Cell::Alive,
        ],
    };

    assert_eq!(
        universe.to_string(),
        "◻ ◻ ◻ ◻ \n\
         ◻ ◻ ◻ ◼ \n\
         ◻ ◻ ◼ ◼ \n\
         ◻ ◼ ◼ ◼ \n"
    );
}
```

Make sure that running `cargo +nightly test` reports this test passing before
continuing!

### Rendering to the DOM with JavaScript

First, let's add a `<pre>` element to our `index.html` to render the universe
into. Edit the `<body>` element to look like this:

```html
<body>
    <pre id="game-of-life-canvas"></pre>
    <script src='./bootstrap.js'></script>
</body>
```

Additionally, we want the `<pre>` centered in the middle of the Web page. We
can use CSS flex boxes to accomplish this task. Add the following `<style>` tag
inside `index.html`'s `<head>`:

```html
<style>
    body {
        width: 100%;
        height: 100%;
        margin: 0;
        padding: 0;
        display: flex;
        align-items: center;
        justify-content: center;
    }
</style>
```

At the top of `index.js`, let's fix our import to bring in the `Universe` rather
than the old `greet` function:

```js
import { Universe } from "./hello_world";
```

Also, let's get that `<pre>` element we just added and instantiate a new
universe:

```js
const pre = document.getElementById("game-of-life-canvas");
const universe = Universe.new();
```

Now, we can draw the initial universe into the `<pre>`!

```js
pre.textContent = universe.render();
```

If you re-run

    npm run build-debug
    npm run serve

then you should be able to refresh
[http://localhost:8080](http://localhost:8080) and see a text representation of
your universe!

If not, it is time to troubleshoot and debug. Raise your hand and ask a mentor
for help if you are stuck!

You can add logging to help debug like this:

```rust
#[wasm_bindgen]
extern {
    #[wasm_bindgen(js_namespace = console)]
    fn log(msg: &str);
}

// A macro to provide `println!(..)`-style syntax for `console.log` logging.
macro_rules! log {
    ($($t:tt)*) => (log(&format!($($t)*)))
}
```

The `macro_rules!` definition needs to appear before all uses of the `log!`
macro it defines, so you should place it towards the top of the file.

You can use it like this:

```rust
log!("the value of `x` is {}", x);
```

If you open your browser's developer tools console, you can view the logs.

--------------------------------------------------------------------------------

### Computing the Next Generation

Now that we can draw the universe, let's turn our attention to computing the
next generation inside `src/lib.rs`.

To access the cell at a given row and column, translate the row and column into
an index into the cells vector, as described earlier:

```rust
impl Universe {
    fn get_index(&self, row: u32, column: u32) -> usize {
        unimplemented!("insert code here")
    }
}
```

We could just add this method to to the `impl` block we started above, but we
did label that one “Public methods, exported to JavaScript”, and `get_index`
isn't part of the API we want to expose to JS. It's nice to keep the public API
separate, so we'll start a second `impl Universe` block and put `get_index`
there.

In order to calculate the next state of a cell, we need to get a count of how
many of its neighbors are alive. Let's write a `live_neighbor_count` method to
do just that!

```rust
impl Universe {
    // ...

    fn live_neighbor_count(&self, row: u32, column: u32) -> u8 {
        let mut count = 0;
        for delta_row in [self.height - 1, 0, 1].iter().cloned() {
            for delta_col in [self.width - 1, 0, 1].iter().cloned() {
                if delta_row == 0 && delta_col == 0 {
                    continue;
                }

                let neighbor_row = (row + delta_row) % self.height;
                let neighbor_col = (column + delta_col) % self.width;
                let idx = self.get_index(neighbor_row, neighbor_col);
                count += self.cells[idx] as u8;
            }
        }
        count
    }
}
```

The `live_neighbor_count` method uses deltas and modulo to avoid special casing
the edges of the universe with `if`s. When applying a delta of `-1`, we *add*
`self.height - 1` and let the modulo do its thing, rather than attempting to
subtract `1`. `row` and `column` can be `0`, and if we attempted to subtract `1`
from them, there would be an unsigned integer underflow.

Now we have everything we need to compute the next generation from the current
one!

Recall the rules for each cell's state transition:

1. Any live cell with fewer than two live neighbours dies, as if caused by
   underpopulation.

2. Any live cell with two or three live neighbours lives on to the next
   generation.

3. Any live cell with more than three live neighbours dies, as if by
   overpopulation.

4. Any dead cell with exactly three live neighbours becomes a live cell, as if
   by reproduction.

All other cells remain in the same state.

Translate these rules into Rust code by filling in the `tick` method's
definition:

```rust
/// Public methods, exported to JavaScript.
#[wasm_bindgen]
impl Universe {
    // ...

    pub fn tick(&mut self) {
        let mut next = self.cells.clone();

        for row in 0..self.height {
            for col in 0..self.width {
                unimplemented!("insert code here")
            }
        }

        self.cells = next;
    }
}
```

Since we want to call this from JavaScript, be sure to put this function in the
`impl Universe` block with the `#[wasm_bindgen]` attribute above it!

#### Testing `Universe::tick`

Add the following tests to `src/lib.rs`:

```rust
use Cell::*;

fn assert_tick(w: u32, h: u32, before: Vec<Cell>, after: Vec<Cell>) {
    assert_eq!(before.len(), after.len());
    assert_eq!(w as usize * h as usize, before.len());

    let mut universe = Universe {
        width: w,
        height: h,
        cells: before,
    };
    universe.tick();

    assert_eq!(
        &universe.cells[..],
        &after[..]
    );
}

#[test]
fn tick_rule_1() {
    assert_tick(
        5,
        5,
        vec![
            Dead, Dead, Dead,  Dead, Dead,
            Dead, Dead, Dead,  Dead, Dead,
            Dead, Dead, Alive, Dead, Dead,
            Dead, Dead, Dead,  Dead, Dead,
            Dead, Dead, Dead,  Dead, Dead,
        ],
        vec![
            Dead, Dead, Dead, Dead, Dead,
            Dead, Dead, Dead, Dead, Dead,
            Dead, Dead, Dead, Dead, Dead,
            Dead, Dead, Dead, Dead, Dead,
            Dead, Dead, Dead, Dead, Dead,
        ],
    );
}

#[test]
fn tick_rule_2() {
    assert_tick(
        5,
        5,
        vec![
            Dead, Dead,  Dead,  Dead, Dead,
            Dead, Dead,  Dead,  Dead, Dead,
            Dead, Alive, Alive, Dead, Dead,
            Dead, Alive, Alive, Dead, Dead,
            Dead, Dead,  Dead,  Dead, Dead,
        ],
        vec![
            Dead, Dead,  Dead,  Dead, Dead,
            Dead, Dead,  Dead,  Dead, Dead,
            Dead, Alive, Alive, Dead, Dead,
            Dead, Alive, Alive, Dead, Dead,
            Dead, Dead,  Dead,  Dead, Dead,
        ],
    );
}

#[test]
fn tick_rules_3_and_4() {
    assert_tick(
        5,
        5,
        vec![
            Dead, Dead,  Dead,  Dead,  Dead,
            Dead, Dead,  Alive, Dead,  Dead,
            Dead, Alive, Alive, Alive, Dead,
            Dead, Dead,  Alive, Dead,  Dead,
            Dead, Dead,  Dead,  Dead,  Dead,
        ],
        vec![
            Dead, Dead,  Dead,  Dead,  Dead,
            Dead, Alive, Alive, Alive, Dead,
            Dead, Alive, Dead,  Alive, Dead,
            Dead, Alive, Alive, Alive, Dead,
            Dead, Dead,  Dead,  Dead,  Dead,
        ],
    );
}

#[test]
fn tick_cells_on_edge() {
    assert_tick(
        5,
        5,
        vec![
            Dead,  Dead, Dead, Dead,  Dead,
            Dead,  Dead, Dead, Dead,  Dead,
            Alive, Dead, Dead, Alive, Alive,
            Dead,  Dead, Dead, Dead,  Dead,
            Dead,  Dead, Dead, Dead,  Dead,
        ],
        vec![
            Dead, Dead, Dead, Dead, Dead,
            Dead, Dead, Dead, Dead, Alive,
            Dead, Dead, Dead, Dead, Alive,
            Dead, Dead, Dead, Dead, Alive,
            Dead, Dead, Dead, Dead, Dead,
        ],
    );
}
```

Make sure that these tests pass when you run `cargo +nightly test` before
continuing!

### Rendering Each New Generation Inside `requestAnimationFrame`

Remove the old `pre.textContent = universe.render();` line of code from
`index.js`. We will replace it with a recursive `requestAnimationFrame` loop. On
each iteration, it computes the next generation of the universe by calling
`tick`, and then renders the result into the `<pre>`:

```js
const renderLoop = () => {
  universe.tick();
  pre.textContent = universe.render();
  requestAnimationFrame(renderLoop);
};
```

To kick off the rendering process, all we have to do is schedule the initial
animation frame callback:

```js
requestAnimationFrame(renderLoop);
```

Let's rebuild and restart the server:

    npm run build-debug
    npm run serve

Refreshing [http://localhost:8080](http://localhost:8080) once more, and you
should see the Game of Life animated before your eyes!

--------------------------------------------------------------------------------

### Rendering to `<canvas>` Directly from Memory

Generating (and allocating) a `String` in Rust and then having `wasm-bindgen`
convert it to a valid JavaScript string makes unnecessary copies of the
universe's cells. Instead of our current `render` method, we can return a
pointer to the start of the cells array. The JavaScript code knows the width and
height of the universe, and can read the bytes that make up the cells directly.
This design does not copy the universe's cells or tax the JavaScript garbage
collector with allocations, but we must directly read the cells' bytes from
WebAssembly's linear memory in JavaScript. Instead of rendering unicode text,
we'll switch to using the [Canvas API]. We will use this design in the rest of
the tutorial.

[Canvas API]: https://developer.mozilla.org/en-US/docs/Web/API/Canvas_API

First, let's replace the `<pre>` we added earlier with a `<canvas>` we will
render into (it too should be within the `<body>`, before the `<script>` that
loads our JavaScript):

```html
<body>
    <canvas id="game-of-life-canvas"></canvas>
    <script src='./bootstrap.js'></script>
</body>
```

Return to `src/lib.rs` and export (i.e. define them within the `#[wasm_bindgen]`
block) three new functions to JavaScript:

1. A `width` function that returns the width of the universe as a `u32`.

2. A `height` function that returns the height of the universe as a `u32`.

3. A `cells` function that returns a `*const Cell` pointer to the first cell in
   the universe's `cells` array.

Next, let's edit `index.js` and define some constants that JavaScript will use
when rendering the canvas:

```js
const CELL_SIZE = 5; // px
const GRID_COLOR = "#CCCCCC";
const DEAD_COLOR = "#FFFFFF";
const ALIVE_COLOR = "#000000";

// These must match `Cell::Alive` and `Cell::Dead` in `src/lib.rs`.
const DEAD = 0;
const ALIVE = 1;
```

Now, let's rewrite our current JS code (except for the import) to no longer
write to the `<pre>` but instead draw to the `<canvas>`:

```js
// Construct the universe, and get its width and height.
const universe = Universe.new();
const width = universe.width();
const height = universe.height();

// Give the canvas room for all of our cells and a 1px border
// around each of them.
const canvas = document.getElementById("game-of-life-canvas");
canvas.height = (CELL_SIZE + 1) * height + 1;
canvas.width = (CELL_SIZE + 1) * width + 1;

const ctx = canvas.getContext('2d');

const renderLoop = () => {
  universe.tick();

  drawGrid();
  drawCells();

  requestAnimationFrame(renderLoop);
};
```

To draw the grid between cells, we draw a set of equally-spaced horizontal
lines, and a set of equally-spaced vertical lines. These lines criss-cross to
form the grid.

```js
const drawGrid = () => {
  ctx.beginPath();
  ctx.strokeStyle = GRID_COLOR;

  // Vertical lines.
  for (let i = 0; i <= width; i++) {
    ctx.moveTo(i * (CELL_SIZE + 1) + 1, 0);
    ctx.lineTo(i * (CELL_SIZE + 1) + 1, (CELL_SIZE + 1) * height + 1);
  }

  // Horizontal lines.
  for (let j = 0; j <= height; j++) {
    ctx.moveTo(0,                           j * (CELL_SIZE + 1) + 1);
    ctx.lineTo((CELL_SIZE + 1) * width + 1, j * (CELL_SIZE + 1) + 1);
  }

  ctx.stroke();
};
```

To draw the cells, we get the cells pointer into the WebAssembly linear memory
from the universe, construct a `Uint8Array` overlaying the cells buffer, iterate
over each cell, and draw a white or black rectangle depending on whether the
cell is dead or alive, respectively. By working with pointers and overlays, we
avoid copying the cells across the boundary on every tick.

```js
// Import the WebAssembly memory at the top of the file.
// The 'hello_world_bg' module is generated by wasm-bindgen,
// hence the '_bg'.
import { memory } from "./hello_world_bg";

// ...

const getIndex = (row, column) => {
  return row * width + column;
};

const drawCells = () => {
  const cellsPtr = universe.cells();
  const cells = new Uint8Array(memory.buffer, cellsPtr, width * height);

  ctx.beginPath();

  for (let row = 0; row < height; row++) {
    for (let col = 0; col < width; col++) {
      const idx = getIndex(row, col);

      ctx.fillStyle = cells[idx] === DEAD
        ? DEAD_COLOR
        : ALIVE_COLOR;

      ctx.fillRect(
        col * (CELL_SIZE + 1) + 1,
        row * (CELL_SIZE + 1) + 1,
        CELL_SIZE,
        CELL_SIZE
      );
    }
  }

  ctx.stroke();
};
```

To schedule rendering, we'll use the same code as above to start the first
iteration of the rendering loop:

```js
requestAnimationFrame(renderLoop);
```

Rebuild and restart the development server:

```
npm run build-debug
npm run serve
```

If you refresh [http://localhost:8080/](http://localhost:8080/), you should be
greeted with an exciting display of life in `<canvas>`!

### More Exercises

* Instead of hard-coding the initial universe, generate a random one, where each
  cell has a fifty-fifty chance of being alive or dead.

  *Hint: use `wasm_bindgen` to import the `Math.random` JavaScript function:*

  ```rust
  #[wasm_bindgen]
  extern {
      #[wasm_bindgen(js_namespace = Math)]
      fn random() -> f64;
  }
  ```

* Representing each cell with a byte makes iterating over cells easy, but it
  comes at the cost of wasting memory. Each byte is eight bits, but we only
  require a single bit to represent whether each cell is alive or dead. Refactor
  the data representation so that each cell uses only a single bit of space.

## The End

Thanks for following along!

For more Rust and WebAssembly:

* [Follow @rustwasm on Twitter](https://twitter.com/rustwasm)

* Join us on IRC at `#rust-wasm` on `irc.mozilla.org`

* [Read the Rust and WebAssembly Book.](https://rust-lang-nursery.github.io/rust-wasm/)
  This workshop was derived from the Game of Life tutorial in this book. The
  book keeps going and discusses profiling for speed and size, as well as
  debugging in depth!

* Get involved in making Rust and WebAssembly a great combination by
  [joining the working group!](https://github.com/rust-lang-nursery/rust-wasm#get-involved)
