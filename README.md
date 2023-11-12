# Conwey's Game of Life

  Before you start coding, make sure you have  **Rustc**, **Cargo**, **Wasm-pack**, and the latest **npm** installed (*npm install npm@latest -g*).

<video width="500" height="350" controls>
    <source src="https://cdn.cosmicjs.com/659d3120-809f-11ee-955f-5fd6e82e9841-part2.mov" type="video/mp4">
  </video>

## Part1 : Basic setup for hello-world
   This section will show you how to build and run your first Rust and WebAssembly program: a Web page that alerts "Hello, World!"
    
### • Clone the Project Template
```bash
$ cargo generate --git https://github.com/rustwasm/wasm-pack-template 
```
  > *note*: This should prompt you for the new project's name. We will use `wasm-game-of-life`.

```bash
$ cd  wasm-game-of-life
```
 
####  File structure
  ![FS1.png](https://imgix.cosmicjs.com/fb084980-8099-11ee-955f-5fd6e82e9841-FS1.png)
1. **`Cargo.toml`:**
   -  This file serves as the configuration file for the Rust project. It defines dependencies, project metadata, and build settings.

2. **`lib.rs`:**
   -  The main Rust source file containing the code for the WebAssembly (Wasm) Game of Life.

3. **`utils.rs`:**
   -  This Rust source file provides utility functions and helper methods used in the Wasm Game of Life project.

### • Build the Project
We use wasm-pack to orchestrate the following build steps:
```bash
$ warm-pack build
```
When the build has completed, we can find its artifacts in the pkg directory.

#### File structure
![FS2.png](https://imgix.cosmicjs.com/a56b84f0-809a-11ee-955f-5fd6e82e9841-FS2.png)
**`pkg/` Directory:**

1. **`wasm_game_of_life_bg.wasm`:**
   -  WebAssembly binary file containing the Game of Life logic.

2. **`wasm_game_of_life.d.ts`:**
   -  TypeScript declaration file for type information when using the WebAssembly module.

3. **`wasm_game_of_life.js`:**
   -  JavaScript file acting as a bridge for interaction with the WebAssembly module. 
   
4. **`package.json`:**
   - Configuration file for the JavaScript package. Includes metadata and dependencies.

### • Putting it into a Web Page
To take our wasm-game-of-life package and use it in a Web page, we use the create-wasm-app JavaScript project template.
```bash
$ npm init wasm-app www
```
#### File structure
![FS3.png](https://imgix.cosmicjs.com/f0ab2150-809a-11ee-955f-5fd6e82e9841-FS3.png)
**`wasm-game-of-life/www/` Directory:**

1. **`bootstrap.js`:**
   -  Initializes the application.It likely sets up configurations and kicks off the execution of the main JavaScript code.

2. **`index.html`:**
   -  Main HTML entry point for the web app.

3. **`index.js`:**
   -  Main JavaScript entry point, coordinating components.

4. **`package.json`:**
   -  npm package configuration.
   - 
5. **`webpack.config.js`:**
   -  webpack configuration for bundling and deployment.
   
 ####  Install decencies in *`wasm-game-of-life/www`* subdirectory:
```bash
$ cd www
$ npm install
```
### Using our Local wasm-game-of-life Package in `www`

Instead of using the `hello-wasm-pack` package from npm, we want to use our own local `wasm-game-of-life` package. This helps us develop the Game of Life program step by step.

In the `wasm-game-of-life/www/package.json` file, find "*devDependencies*" and add a "*dependencies*" section. Include a `"wasm-game-of-life": "file:../pkg"` entry there.
```json
📂 wasm-game-of-life/www/package.json
{
  // ...
  "dependencies": {                     // Add this three lines block!
    "wasm-game-of-life": "file:../pkg"
  },
  "devDependencies": {
    //...
  }
}
```
Next, modify `wasm-game-of-life/www/index.js` to import `wasm-game-of-life` instead of the `hello-wasm-pack` package:

```js
📂 wasm-game-of-life/www/index.js

import * as wasm from "wasm-game-of-life";  // change import here

wasm.greet();
```
now we need to install dependency in `www` subdirectory:
```bash
$ npm install
```
Our Web page is now ready to be served locally!

### • Running Locally
In the new terminal, run this command from within the `wasm-game-of-life/www` directory:
```bash
$ npm run start
```
Navigate to  *`http://localhost:8080/ `*:

<video width="500" height="350" controls>
    <source src="https://cdn.cosmicjs.com/eb60b0a0-809c-11ee-955f-5fd6e82e9841-part1.mov" type="video/mp4">
  </video>

## Part 2 : Implementation of Conway's Game of Life:
I'll keep this blog short and just touch on the rules of Conway's Game of Life. If you want more details, you can check out the Wikipedia link [insert link].

#### Rules of Game of Life
The universe of the Game of Life is an infinite two-dimensional orthogonal grid of square cells, each of which is in one of two possible states, alive or dead.At each step in time, the following transitions occur:

1. Any live cell with fewer than two live neighbours dies, as if caused by underpopulation.

2.  Any live cell with two or three live neighbours lives on to the next generation.

3.  Any live cell with more than three live neighbours dies, as if by overpopulation.

4. Any dead cell with exactly three live neighbours becomes a live cell, as if by reproduction.

### • Design and Logic Building (Infinite Universe) :
> **`My experience🤓`** : As I engage with this implementation, my DSA skills prove useful in constructing logic. If you prefer Java, I've also written this code in Java, available on my GitHub (`note`: I crafted this java version during a recent Meetup at *Retreat Dev Day in a TDD* session).

To optimize performance, we avoid unnecessary data transfers between WebAssembly and JavaScript in Conway's Game of Life. The universe is represented as a flat array in WebAssembly memory, with `0 for dead cells` and `1 for live cells`. To access a cell's index, we use a simple formula.

For JavaScript interaction, we initially use `std::fmt;` Display to create a Rust String, which is then copied to JavaScript for rendering. In the future, we might explore alternatives like returning a list of changed cells after each tick, reducing the need for JavaScript to iterate over the entire universe for rendering but introducing some complexity.

#### implementation: 
Let's begin by removing the *alert import and greet function* from `wasm-game-of-life/src/lib.rs`, and replacing them with a type definition for cells and implementation for Universe:
```js
📂 wasm-game-of-life/src/lib.rs

mod utils;

use std::fmt;
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
#[repr(u8)]
#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub enum Cell {
    Dead = 0,
    Alive = 1,
}


#[wasm_bindgen]
pub struct Universe {
    width: u32,
    height: u32,
    cells: Vec<Cell>,
}

impl Universe {
    fn get_index(&self, row: u32, column: u32) -> usize {
        (row * self.width + column) as usize
    }

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

/// Public methods, exported to JavaScript.
#[wasm_bindgen]
impl Universe {
    pub fn tick(&mut self) {
        let mut next = self.cells.clone();

        for row in 0..self.height {
            for col in 0..self.width {
                let idx = self.get_index(row, col);
                let cell = self.cells[idx];
                let live_neighbors = self.live_neighbor_count(row, col);

                let next_cell = match (cell, live_neighbors) {
                    // Rule 1: Any live cell with fewer than two live neighbours
                    // dies, as if caused by underpopulation.
                    (Cell::Alive, x) if x < 2 => Cell::Dead,
                    // Rule 2: Any live cell with two or three live neighbours
                    // lives on to the next generation.
                    (Cell::Alive, 2) | (Cell::Alive, 3) => Cell::Alive,
                    // Rule 3: Any live cell with more than three live
                    // neighbours dies, as if by overpopulation.
                    (Cell::Alive, x) if x > 3 => Cell::Dead,
                    // Rule 4: Any dead cell with exactly three live neighbours
                    // becomes a live cell, as if by reproduction.
                    (Cell::Dead, 3) => Cell::Alive,
                    // All other cells remain in the same state.
                    (otherwise, _) => otherwise,
                };

                next[idx] = next_cell;
            }
        }

        self.cells = next;
    }

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

    pub fn render(&self) -> String {
        self.to_string()
    }

}

impl fmt::Display for Universe {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        for line in self.cells.as_slice().chunks(self.width as usize) {
            for &cell in line {
                let symbol = if cell == Cell::Dead { '◻' } else { '◼' };
                write!(f, "{}", symbol)?;
            }
            write!(f, "\n")?;
        }

        Ok(())
    }
}

```



### • Rendering with javaScript
First, let's add a `<pre>` element to `wasm-game-of-life/www/index.html` to render the universe into, just above the `<script>` tag, and also css styling in `<style>`:
```html
📂 wasm-game-of-life/www/index.html

<!DOCTYPE html>
<html>
  <head>
    <meta charset="utf-8" />
    <title>Hello wasm-pack!</title>
    <style>
      body {
        position: absolute;
        top: 0;
        left: 0;
        width: 100%;
        height: 100%;
        display: flex;
        flex-direction: column;
        align-items: center;
        justify-content: center;
      }
    </style>
  </head>
  <body>
    <pre id="game-of-life-canvas"></pre>
    <noscript
      >This page contains webassembly and javascript content, please enable
      javascript in your browser.</noscript
    >
    <script src="./bootstrap.js"></script>
  </body>
</html>
```

At the top of `wasm-game-of-life/www/index.js`, let's fix our import to bring in the `Universe` rather than the old `greet` function:
```js
📂 wasm-game-of-life/www/index.js

import { Universe } from "wasm-game-of-life";

const pre = document.getElementById("game-of-life-canvas");
const universe = Universe.new();

const renderLoop = () => {
  pre.textContent = universe.render();
  universe.tick();
  requestAnimationFrame(renderLoop);
};

requestAnimationFrame(renderLoop);
```

### • Time to compile and run:
Rebuild the WebAssembly and bindings glue by running this command from within the root `wasm-game-of-life` directory:
```bash
$ wasm-pack build
```
Make sure your development server is still running. If it isn't, start it again from within the `wasm-game-of-life/www` directory:
```bash
$ npm run start
```
If you refresh `http://localhost:8080/`, you should be greeted with an exciting display of life!

