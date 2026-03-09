<div align="center">

# 🎲 Rubik's Cube Solver

[![Live Demo](https://img.shields.io/badge/Try%20Live-rubikscube.fly.dev-blue?style=for-the-badge&logo=java)](https://rubikscube.fly.dev)
[![Java](https://img.shields.io/badge/Java-17+-ED8B00?style=for-the-badge&logo=openjdk&logoColor=white)](https://openjdk.org/)
[![Three.js](https://img.shields.io/badge/Three.js-3D%20Visualization-black?style=for-the-badge)](https://threejs.org)
[![Docker](https://img.shields.io/badge/Docker-Ready-2496ED?style=for-the-badge&logo=docker&logoColor=white)](https://docker.com)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg?style=for-the-badge)](LICENSE)

*Any scramble. 20 moves or fewer. Solved in real time.*

</div>

---

## The Puzzle That Has Humbled Generations

There are very few objects in the world that are simultaneously so simple in description and so incomprehensibly complex in practice. A Rubik's Cube is just six faces, each of nine colored tiles, and you have to get all nine tiles on each face to be the same color. That is the entire problem statement. A child can understand it in thirty seconds. And yet it has a state space of over 43 quintillion distinct configurations — 43,252,003,274,489,856,000 to be precise — and most people who pick one up, rotate a few faces, and try to solve it find themselves completely lost within a minute.

The puzzle became an obsession for millions of people after its commercial release in 1980. Speed-cubing competitions drew international attention. And then, almost quietly, it became one of the canonical problems in combinatorial mathematics: is there a universal solution that works for every possible scramble and does so in the minimum possible number of moves?

The answer — proven computationally in 2010 — is yes. No matter how scrambled a Rubik's Cube is, it can always be solved in 20 moves or fewer. That number became known as "God's Number." And the algorithm that gets there efficiently — not by brute force, but by intelligent search through a structured decomposition of the problem — is called Kociemba's Two-Phase Algorithm. It is the algorithm at the heart of this project.

---

## The Question Behind the Build

The motivation for building this was a mix of curiosity and challenge. Kociemba's algorithm is well-known by name in the cubing community, but understanding it deeply enough to implement it from scratch in Java is a different order of difficulty. The algorithm operates on abstract mathematical representations of cube states that most people never encounter in everyday programming. Its two-phase search structure is elegantly designed but non-obvious. And building a web-based interface around it — one where you can input any scramble and watch the solution animate step-by-step in both 2D and 3D — added a full-stack layer on top of the algorithmic core.

The project also had a practical deployment goal: build something that could run as a public web service, deployed on Fly.io or Railway or Render, accessible without installation, from any device with a browser. That constraint shaped every architecture decision.

---

## Understanding Kociemba's Two-Phase Algorithm

Before any code was written, the first task was understanding the algorithm at a mathematical level. Kociemba's algorithm is not a simple search — it is a structured decomposition of the cube's symmetry group into two nested subgroups, each of which can be searched much more efficiently than the full 43-quintillion-state space.

**Phase 1** operates on the full cube group G and searches for a short sequence of moves that brings the cube into a special subgroup G1. G1 is defined as the set of all cube states reachable using only the moves U, D, R2, L2, F2, and B2 — that is, the U and D faces can be turned in any direction, but the R, L, F, and B faces can only be turned by 180 degrees (half-turns). What makes this subgroup useful is that it has significantly fewer states than the full cube group — roughly 2 billion states versus 43 quintillion — and, crucially, there are efficient coordinate representations of these states that allow precomputed lookup tables to guide the search.

The Phase 1 search uses three coordinates to describe the cube state: the orientation of the 7 corner cubies (there are 2187 possible corner orientations), the orientation of the 11 edge cubies (there are 2048 possible edge orientations), and the position of the 4 UD-slice edges — the 4 edges that belong to the middle layer (there are 495 possible positions for these edges). Together, these three coordinates describe the cube's position relative to G1 without fully specifying its state, which makes Phase 1 a tractable search problem. The search proceeds using iterative-deepening A* with precomputed pruning tables that give lower bounds on how many additional Phase 1 moves are needed from any given state.

**Phase 2** takes the Phase 2 state — now in G1 — and searches for a solution using only the restricted move set of G1. Phase 2 has its own set of coordinates: the permutation of the 8 corner cubies (40320 possible permutations), the permutation of the 8 non-UD-slice edges (40320 possible permutations), and the permutation of the 4 UD-slice edges within the middle layer (24 possible arrangements). The Phase 2 state space is much smaller than the full cube — roughly 40,000 times fewer states — which makes it tractable to search to completion.

The overall algorithm runs Phase 1 to find a short move sequence that brings the cube into G1, then runs Phase 2 on the resulting state to complete the solution. The total solution length is the sum of the two phase lengths, and the algorithm iterates over different Phase 1 solutions of increasing length, searching for Phase 2 completions, until a solution within the desired total length limit is found. The precomputed pruning tables for both phases guide the search efficiently, allowing solutions to be found in milliseconds even though the theoretical state space is astronomical.

The key insight that makes this tractable in practice is that the pruning tables are the most computationally expensive part — they take time and memory to build once at startup, but once built they make every subsequent solve query extremely fast. This is the classic tradeoff between preprocessing cost and query cost, applied to combinatorial search.

---

## The Java Backend: Seven Classes, One HTTP Server

The backend is written in pure Java 17 using no external libraries except the JDK's own `com.sun.net.httpserver` package, which provides a lightweight HTTP server without requiring Tomcat, Jetty, or any other web framework. This was a deliberate choice: minimizing dependencies reduces the container image size, simplifies the deployment configuration, and eliminates dependency management complexity.

**`RubikWebServer.java`** is the entry point and HTTP routing layer. It creates an `HttpServer` on the configured port (read from the `PORT` environment variable for compatibility with cloud platforms, falling back to 8080 for local development) and registers three route handlers. The `/api/state` route accepts POST requests with a cube description and returns the 54-character facelet string representing the cube's state. The `/api/solve` route accepts POST requests with a cube description and returns the full solution — the facelet string, the solution move sequence as a string, the moves as a JSON array, and a trace array of facelet strings for every intermediate state in the solution sequence, which the frontend uses to animate the solve step by step. The `/` route serves static files from the `public/` directory, with path traversal protection enforced by normalizing the requested path and verifying it remains within the public directory before reading any bytes. The server implements automatic port increment on bind failures — if port 8080 is taken, it tries 8081, then 8082, and so on up to ten attempts, which makes local development more robust.

**`Solver.java`** is the command-line interface wrapper for the solving logic. It reads a 9×12 color grid from a text file, parses the grid into the 54-character facelet string format that the Kociemba engine expects, runs the solve, and writes the normalized solution to an output file. It maps arbitrary color characters from the input file to the standard face letters (U, R, F, D, L, B) by reading the center facelet of each face and building a dynamic color-to-face mapping. This means the solver works with any color convention — whether the input uses W for white or just W, whatever character appears at the center of each face is treated as that face's canonical color.

**`Search.java`** implements the core Kociemba Two-Phase Algorithm. This is the heart of the project — the class that actually computes solutions. It implements the full two-phase iterative-deepening search with precomputed pruning tables for both phases. The `solution(facelets, maxDepth, timeoutSeconds, useSeparator)` method takes a 54-character facelet string and search parameters, runs the two-phase search, and returns the solution as a move sequence string in standard notation (R, U, F, D, L, B, with ' for counter-clockwise and 2 for half-turn). The search terminates as soon as a solution within the depth limit is found, or after the timeout is exceeded, whichever comes first.

**`CoordCube.java`** represents the cube in coordinate space — the abstract numerical representation used by the Phase 1 and Phase 2 searches. It stores the three Phase 1 coordinates (corner orientation, edge orientation, UD-slice edge position) and the three Phase 2 coordinates (corner permutation, edge permutation, UD-slice edge permutation) as integer arrays. It also holds the precomputed move tables that map each coordinate to its value after applying each of the 18 possible moves, and the pruning tables that provide lower bound estimates for the search.

**`CubieCube.java`** represents the cube at the cubie level — the level of individual physical pieces. A 3x3 Rubik's Cube has 8 corner cubies and 12 edge cubies. Each corner cubie has a position (which of the 8 corner slots it occupies) and an orientation (which of the 3 ways it can be twisted). Each edge cubie has a position (which of the 12 edge slots it occupies) and an orientation (which of the 2 ways it can be flipped). `CubieCube` stores these four arrays and implements all 18 possible face moves as explicit cubie permutation operations. It also implements conversion to and from coordinate representations and the facelet representation.

**`FaceCube.java`** represents the cube at the facelet level — the level of individual colored squares, exactly as a human sees the cube. There are 54 facelets in total (6 faces × 9 squares). `FaceCube` holds a 54-element array of color values and implements conversion between facelet representation and cubie representation, which is necessary to bridge the human-readable input format and the abstract mathematical representation used by the search.

**`CubeInputUtil.java`** handles the parsing of the various input formats that the web frontend can submit — either a sequence of moves in standard notation (which are applied to a solved cube to produce the scrambled state) or a 9×12 color grid text representation. It normalizes both formats into the 54-character facelet string that the rest of the backend expects.

**`CubeTraceUtil.java`** generates the animation trace — the sequence of intermediate cube states as the solution is applied move by move. Given the initial facelet string and a list of moves, it applies each move in sequence and records the resulting facelet string after each step. This produces the trace array that the frontend uses to animate the solution — each frame of the animation corresponds to one step in the trace.

**`JsonUtil.java`** provides minimal JSON serialization utilities — string escaping and array serialization — without any external library dependency.

---

## The Frontend: Two Views, One Cube

The frontend is organized around two visualization modes for the same cube state, switchable at any time with a tab control.

**The 2D Net View** (implemented in `app.js`) renders the cube as an unfolded net — the classic diagram that shows all six faces laid out flat in a cross pattern. The net is drawn on an HTML canvas using 2D drawing APIs. Each facelet is a colored square with a thin border. As the solution animates, the facelet colors update on the canvas after each move, letting the user see exactly which tiles move where and how the colors converge toward the solved state. The 2D view is computationally lightweight and works well on all devices including low-end mobile hardware.

**The 3D Interactive View** (implemented in `cube3d.js`) uses Three.js, a WebGL-based 3D graphics library, to render a photorealistic 3D cube. The cube is built from 26 visible cubie meshes (the center core cubie is invisible) with colored materials matching the standard color scheme: white for the U face, red for R, green for F, yellow for D, orange for L, and blue for B. The scene includes directional lighting and ambient lighting to give the cube a solid, three-dimensional appearance. The user can drag to rotate the view freely, examining the cube from any angle. When the solution animates in 3D, the face moves are rendered as smooth 3D rotations of the relevant cubies, with each move taking a configurable number of milliseconds so the animation plays at a readable pace.

The solution animation works identically in both views. When the user clicks Solve, the frontend sends the current cube state to the `/api/solve` endpoint, receives the move sequence and the full trace array, and then plays through the trace frame by frame with a configurable delay between frames. A progress display shows the current move number and the total solution length. A copy button lets the user copy the solution sequence to the clipboard.

The frontend supports two input methods. The first is standard move notation — the user can type a scramble sequence like `R U R' U' R' F R2 U' R' U' R U R' F'` directly into the input field, and the frontend sends it to the backend which applies the moves to a solved cube and returns the resulting state. The second is drag-and-drop file import — the user can drag any `.txt` file containing a 9×12 color grid onto the application, which uploads the file to the backend for parsing. The 40 test cases in the `testcases/` folder use this format and can be dropped directly onto the app for instant testing.

---

## Deploying Without Friction: Multiple Platforms, One Codebase

A significant part of the engineering effort went into making the project deployable to multiple cloud platforms without code changes, using only configuration files.

**Fly.io** (`fly.toml`) was the primary deployment target and is where the live demo runs. Fly.io runs containerized applications on a global edge network with automatic HTTPS and free tier support. The `fly.toml` specifies the build context, the exposed port, health check configuration, and the HTTP service settings. The `Dockerfile` builds a minimal JDK image that compiles the Java source files, copies the public directory, and sets the entrypoint to run `RubikWebServer`. The multi-stage build keeps the final image small.

**Railway** (`railway.json`), **Render** (`render.yaml`), and **Docker** (`Dockerfile`) configurations are all included, meaning the project can be deployed to any of these platforms without modification by simply pointing at the repository. This was a deliberate choice to avoid platform lock-in and to make the project easy to run for anyone regardless of which cloud platform they prefer.

**Local development** is handled by `run.bat` (Windows) and `run.sh` (Mac/Linux), which compile the Java source files and launch the server with a single command. No IDE required, no build tool (Maven or Gradle) required — just a JDK installation and a terminal.

---

## The Test Cases: 40 Scrambles That Cover the Edge Cases

The `testcases/` folder contains 40 scramble files in the 9×12 color grid format. These were created to test the solver against a range of scramble types: shallow scrambles (easy to solve, useful for verifying the basic pipeline), deep scrambles (maximum distance from solved, stress-testing the search depth), scrambles that trigger specific Phase 1 behaviors, and near-solved states that should produce very short solutions.

Each test file uses the same format — a 9-row, 12-column grid where each row contains the color characters for the cube faces in their standard unfolded positions, with space-separated columns for each row of each face. The color legend is W (White, Up face), R (Red, Right), G (Green, Front), Y (Yellow, Down), O (Orange, Left), B (Blue, Back). The solver's `buildColorMap()` function reads the center facelet of each face to determine which character maps to which face letter, so the test files work even if the color conventions differ.

Running all 40 test cases through the solver and verifying that each solution actually produces a solved cube when applied to the scrambled state was a key validation step during development. A correct solution applied to its scramble should always produce the 54-character string `UUUUUUUUURRRRRRRRR...` — all faces in order. Any deviation indicates a bug in the move application logic or the facelet-to-cubie conversion.

---

## What the Project Discovers About Algorithmic Complexity

Building this project produced some observations that go beyond the immediate goal of solving a puzzle.

The first is about the power of precomputation. The Kociemba algorithm is fast at query time — it finds solutions in milliseconds — because the hard work of building pruning tables is done once at startup. Those tables encode millions of precomputed distances: for any Phase 1 or Phase 2 state, what is the minimum number of moves needed to reach the target? Without these tables, the search would be orders of magnitude slower. The lesson is that the right time complexity tradeoff is often to front-load computation that will be reused many times, rather than doing it on demand.

The second is about the relationship between representation and algorithm efficiency. The same physical cube can be described as a 9×12 color grid (human-readable but algorithmically useless for search), as a set of cubie positions and orientations (mathematically precise but large), or as a set of integer coordinates in a compressed representation (abstract but computationally tractable for search). The algorithm only works efficiently at the coordinate level. The layers of representation — facelet to cubie to coordinate — each serve a specific purpose in the pipeline, and building clean conversion logic between them was as important as implementing the search itself.

The third is about full-stack architecture for algorithmic tools. The Java backend implements a pure REST API with no opinions about the frontend. The frontend consumes that API and handles all visualization. This separation meant the 3D view could be rebuilt from scratch (replacing an earlier canvas-based approach with Three.js) without touching a single line of Java. The architecture pays dividends whenever the interface needs to change while the algorithm stays the same.

The fourth observation is about the depth of "solved problems." Rubik's Cube solving is a "solved problem" in the sense that the algorithm is known and implemented in multiple languages. But implementing it from scratch reveals just how much engineering is packed into that two-sentence algorithm description. The pruning tables, the coordinate transformations, the two-phase search orchestration, the input parsing, the animation trace generation — each piece required careful design and testing. "It's been done before" is not the same as "it's trivial to do."

---

## Running It Yourself

**Online:** Visit [rubikscube.fly.dev](https://rubikscube.fly.dev) (may take a few seconds to wake from cold start on the free tier).

**Locally (Windows):**

```bat
git clone https://github.com/Sahibjeetpalsingh/RubiksCube-Solver-Java.git
cd RubiksCube-Solver-Java
run.bat
```

**Locally (Mac/Linux):**

```bash
git clone https://github.com/Sahibjeetpalsingh/RubiksCube-Solver-Java.git
cd RubiksCube-Solver-Java
chmod +x run.sh
./run.sh
```

**Manual build:**

```bash
mkdir -p bin
javac -d bin src/*.java
java -cp bin RubikWebServer
```

**Docker:**

```bash
docker build -t rubiks-solver .
docker run -p 8080:8080 rubiks-solver
```

Once running, open [http://localhost:8080](http://localhost:8080). Type a scramble in standard move notation, or drag and drop one of the `.txt` files from the `testcases/` folder. Click Solve and watch the cube solve itself.

The project structure:

```
RubiksCube-Solver-Java/
├── src/
│   ├── RubikWebServer.java    # HTTP server: routing, static file serving, port management
│   ├── Solver.java            # CLI interface: file input parsing, solution output
│   ├── Search.java            # Kociemba Two-Phase Algorithm: the core solver
│   ├── CoordCube.java         # Coordinate cube: abstract state + precomputed move/prune tables
│   ├── CubieCube.java         # Cubie cube: physical piece positions/orientations + move ops
│   ├── FaceCube.java          # Facelet cube: 54-element color array + conversion to cubies
│   ├── CubeInputUtil.java     # Input parser: move notation and 9x12 grid to facelets
│   ├── CubeTraceUtil.java     # Animation trace: intermediate states for step-by-step playback
│   ├── Color.java             # Color enum for the six face colors
│   ├── Corner.java            # Corner cubie enum (URF, UFL, ULB, UBR, DFR, DLF, DBL, DRB)
│   ├── Edge.java              # Edge cubie enum (12 edges)
│   ├── Facelet.java           # Facelet enum (54 facelets)
│   └── JsonUtil.java          # Minimal JSON serialization without external libraries
├── public/
│   ├── index.html             # Main HTML: layout, controls, mode switching
│   ├── styles.css             # Responsive styles for all viewports
│   ├── app.js                 # 2D net view: canvas rendering, API calls, animation
│   └── cube3d.js              # 3D interactive view: Three.js scene, drag rotation, animation
├── testcases/
│   ├── scramble01.txt         # 40 scramble files in 9x12 color grid format
│   └── ...
├── Dockerfile                 # Multi-stage build: compile Java, copy public, run server
├── fly.toml                   # Fly.io deployment configuration
├── railway.json               # Railway deployment configuration
├── render.yaml                # Render deployment configuration
├── run.bat                    # Windows: one-command compile and run
├── run.sh                     # Mac/Linux: one-command compile and run
└── README.md                  # This document
```

---

## Authors

Built by **Sahibjeet Pal Singh** and **Bhuvesh Chauhan**.

Kociemba's Two-Phase Algorithm was originally developed by Herbert Kociemba. The 3D visualization uses Three.js. Deployment runs on Fly.io.

---

*Because 43 quintillion states deserve a better solution than trial and error.*
