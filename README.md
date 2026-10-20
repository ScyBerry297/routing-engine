# Routing Engine: Contraction Hierarchies on Real Road Data

A from-scratch shortest-path routing engine written in Go. It loads real OpenStreetMap road data, answers "fastest route from A to B" queries with several algorithms, proves they agree, measures how much faster Contraction Hierarchies (CH) are than Dijkstra, and visualizes what each algorithm actually searched.

> **Status:** in development. Checked items in the [roadmap](#roadmap) are done. Benchmark numbers below are filled in only from real runs.

## Why this project

Dijkstra's algorithm finds the fastest route on a road network, but it explores a huge region around the start point before reaching the destination. Production routing engines avoid this by doing heavy work once in advance so each query is nearly instant.

This project implements that idea end to end and answers three questions with measurements instead of claims:

1. How much faster is CH than Dijkstra, bidirectional Dijkstra, and A* on a real road graph?
2. What does that speedup cost in preprocessing time and memory?
3. Can the result be trusted? (CH must return exactly the same distance as Dijkstra.)

## How it works

A road network is a directed graph: intersections are nodes, road segments are edges, and edge weights are travel times.

**Baselines**

- **Dijkstra:** expands outward from the source in all directions.
- **Bidirectional Dijkstra:** searches from source and target at once and stops when the searches meet.
- **A\*:** Dijkstra guided by a straight-line (haversine) travel-time lower bound.

**Contraction Hierarchies**

- *Preprocessing:* nodes are ranked by importance (heuristics: edge difference, number of contracted neighbours). Nodes are contracted from least to most important. When removing a node `v` would break the shortest path between two neighbours `u` and `w`, a shortcut edge `u -> w` is added, but only if a *witness search* finds no equally short path avoiding `v`. Each shortcut remembers its middle node.
- *Query:* a bidirectional search that only relaxes edges going **upward** in the ranking. The two searches meet at a high-ranked node, and shortcuts are unpacked via their middle nodes to recover the real path.

Shortcuts preserve shortest-path distances, so CH answers are exact.

## Features

- OSM PBF parsing with drivable-road filtering, one-way handling, and speed-based edge weights
- Compressed sparse row (CSR) graph storage with on-disk serialization
- Dijkstra, bidirectional Dijkstra, A\*, and Contraction Hierarchies
- Nearest-road snapping so a map click resolves to a graph node
- Concurrent batch queries using goroutines
- REST API with a debug mode that returns the ordered list of settled nodes
- Web UI (Vite + React + TypeScript + MapLibre GL JS) showing routes and each algorithm's search frontier side by side
- Differential testing, benchmark harness, and a CI benchmark gate

## Architecture

```
          OSM .pbf extract
                 |
        cmd/preprocess  (parse -> CSR graph -> contract -> write to disk)
                 |
         preprocessed graph file
                 |
          cmd/server (Go, loads graph once, serves queries)
           /      |       \
      snapping  algorithms  REST API
                              |
                    web/ (React + MapLibre)
```

The graph is loaded into memory once at server start. Preprocessing runs offline, so server startup stays fast.

## Repository layout

```
.
├── cmd/
│   ├── server/        # HTTP server entrypoint
│   └── preprocess/    # builds and serializes the graph + CH overlay
├── internal/
│   ├── osm/           # PBF parsing and road filtering
│   ├── graph/         # CSR graph and serialization
│   ├── algo/          # dijkstra, bidirectional, astar, ch
│   ├── spatial/       # nearest-node snapping (grid / k-d tree)
│   └── api/           # HTTP handlers
├── bench/             # benchmark harness and random query generator
├── web/               # Vite + React + TypeScript frontend
├── data/              # OSM extracts and built graphs (gitignored)
├── .github/workflows/ # CI
├── Dockerfile
└── README.md
```

## Getting started

### Prerequisites

- Go (recent stable release)
- Node.js 20+ and npm
- An OSM extract in `.pbf` format, for example from [Geofabrik](https://download.geofabrik.de/)
- Docker (optional)

### Build the graph

```bash
mkdir -p data
# place your extract at data/region.osm.pbf
go run ./cmd/preprocess -in data/region.osm.pbf -out data/region.graph
```

### Run the server

```bash
go run ./cmd/server -graph data/region.graph -addr :8080
```

### Run the frontend

```bash
cd web
npm install
npm run dev
```

### Run with Docker

```bash
docker build -t routing-engine .
docker run -p 8080:8080 -v "$(pwd)/data:/data" routing-engine -graph /data/region.graph
```

## API

> Draft interface; update this section if the implementation changes.

```
GET /route?from=<lat>,<lon>&to=<lat>,<lon>&algo=<dijkstra|bidir|astar|ch>&debug=<0|1>
```

Example response:

```json
{
  "algo": "ch",
  "distance_seconds": 1834.2,
  "path": [[15.4989, 73.8278], [15.4991, 73.8281]],
  "stats": {
    "nodes_settled": 312,
    "edges_relaxed": 1040,
    "query_micros": 85
  },
  "settled": []
}
```

`settled` is populated only when `debug=1` and lists nodes in the order they were settled, which drives the frontier animation in the UI.

## Testing

```bash
go test ./...
```

- **Differential tests:** CH, bidirectional, and A\* distances are compared against Dijkstra on thousands of random source/target pairs. Dijkstra is the oracle because CH bugs produce silent wrong answers.
- **Unit tests:** graph construction, serialization round-trips, snapping, and witness search edge cases.

## Benchmarks

Run:

```bash
go test ./bench -bench . -benchmem -count 10 | tee bench.txt
benchstat bench.txt
```

Results (fill in from real runs; record the machine, region, and query count):

| Algorithm | Avg nodes settled | Avg query time | Speedup vs Dijkstra |
|---|---|---|---|
| Dijkstra | TBD | TBD | 1x |
| Bidirectional Dijkstra | TBD | TBD | TBD |
| A\* | TBD | TBD | TBD |
| Contraction Hierarchies | TBD | TBD | TBD |

CH cost side:

| Metric | Value |
|---|---|
| Graph size (nodes / edges) | TBD |
| Shortcuts added | TBD |
| Preprocessing time | TBD |
| Memory (base graph / with CH) | TBD |

## Continuous integration

GitHub Actions runs on every push and pull request:

1. `go vet` and `go test ./...`
2. Differential correctness tests
3. A benchmark gate that fails the build if **nodes settled** or **edges relaxed** regress (deterministic metrics), plus a wall-time check with a generous tolerance, since CI runners have noisy timing

## Design decisions

- **Go for both engine and API.** A single process avoids a cgo or RPC boundary between languages. All speed comparisons are Go against Go, so the ratios are unaffected. Absolute times will be somewhat slower than a tuned C++ engine.
- **CSR graph layout.** Contiguous offset, target, and weight arrays are cache-friendly and match how production routing engines store graphs.
- **REST, not gRPC.** The only client is a browser, and there is no internal service boundary that gRPC would improve.
- **Vite + React instead of a server-rendering framework.** The UI is a single-page map app, so server rendering, SEO, and multi-page routing add nothing.
- **No GPU.** CH queries are tiny and branch-heavy, which suits CPUs, not GPUs.
- **Goroutines for batch queries.** Many independent queries parallelize cleanly and share a read-only graph.

## Limitations

- Static edge weights only. Live traffic would require a customizable CH variant.
- Turn restrictions and turn costs are not modelled (unless added later).
- Driving profile only.
- Node ordering uses a heuristic. Finding an optimal order is NP-hard.
- Performance is specific to road networks. CH relies on road graphs having a small set of important nodes (low highway dimension) and performs poorly on random graphs.

## Roadmap

- [ ] OSM parsing and CSR graph
- [ ] Dijkstra and benchmark harness
- [ ] Bidirectional Dijkstra and A\*
- [ ] Nearest-road snapping
- [ ] CH preprocessing (ordering, witness search, shortcuts)
- [ ] CH query and path unpacking
- [ ] Differential testing at scale
- [ ] Graph serialization
- [ ] Batch queries with goroutines
- [ ] REST API with debug mode
- [ ] Web UI with side-by-side frontier visualization
- [ ] CI with benchmark gate
- [ ] Docker image
- [ ] ALT (A\* with landmarks) as a stronger baseline

## References

- E. W. Dijkstra, "A note on two problems in connexion with graphs", 1959.
- R. Geisberger, P. Sanders, D. Schultes, D. Delling, "Contraction Hierarchies: Faster and Simpler Hierarchical Routing in Road Networks", 2008.
- A. V. Goldberg, C. Harrelson, "Computing the Shortest Path: A\* Search Meets Graph Theory", 2005.
- I. Abraham, A. Fiat, A. V. Goldberg, R. F. Werneck, "Highway Dimension, Shortest Paths, and Provably Efficient Algorithms", 2010.
- OpenStreetMap contributors, https://www.openstreetmap.org

## Data and licensing

Road data is © OpenStreetMap contributors, available under the [ODbL](https://www.openstreetmap.org/copyright). Add your own license for the code (MIT is a common choice).
