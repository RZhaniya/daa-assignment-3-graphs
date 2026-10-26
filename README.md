# Assignment 3: Graph Algorithms for a Campus Festival

[Assignment (PDF, English)](docs/Assignment_3_EN.pdf) · [Задание (PDF, русский)](docs/Assignment_3_RU.pdf)

[Русская версия](README_RU.md)

**Course:** Design and Analysis of Algorithms

**Practical work:** 50 points

**Language:** Java 17, Python 3.12+, C++17 or C# (.NET 8)

**Deadline:** as announced in Moodle

## 1. Objective

Implement graph algorithms for cable connections, preparation dependencies and delivery routes. Each request contains an independent graph.

Implement `dsu`, `kruskal`, `prim`, `scc`, `topo` and `dag` using the methods specified below. Standard collections, sorting, priority queues and the supplied JSON adapters are permitted; implement the graph algorithms yourself.

## 2. Program requirements

Choose one language. Keep the supplied UTF-8 JSON Lines interface: one request per input line and one result object per output line. Send only results to standard output. A process may receive several requests; reset algorithm state for each request.

Vertices are numbered from `0` to `n-1`. Edge IDs are original zero-based positions in `edges`; preserve them when sorting or building adjacency lists. Integer inputs and outputs, including SCC labels, must fit signed 64-bit integers. Avoid overflow in cost and distance calculations; Python integers are suitable. Unreachable distances use JSON `null`.

All limits are inclusive. Let `n` be the number of vertices, `m` the number of edges and `q` the number of union operations.

| Method | Vertices | Edges or operations | Edge weights |
|---|---|---|---|
| `dsu` | `0 <= n <= 100000` | `0 <= q <= 250000` | Not used |
| `kruskal`, `prim` | `0 <= n <= 100000` | `0 <= m <= 250000` | `-1000000000 <= w <= 1000000000` |
| `scc`, `topo` | `0 <= n <= 100000` | `0 <= m <= 250000` | Not used |
| `dag` | `1 <= n <= 100000` | `0 <= m <= 250000` | `-1000000000 <= w <= 1000000000` |

Every edge endpoint and operation argument is a valid vertex. For `n=0`, the edge or operation list is empty. DAG requests contain a valid `source`, with `0 <= source < n`. Inputs satisfy these requirements; malformed JSON and out-of-range inputs are not assessed. Field names and task names are case-sensitive.

Handle deep graphs without call-stack failure. Memory bounds include the returned result. Output field order is unrestricted; any valid non-unique answer is accepted.

The supplied adapter handles `{"task":"ping"}` with `{"status":"ready"}` for the setup check. Keep this behaviour.

## 3. Connectivity: disjoint-set union

Venues are connected by cables added one at a time. Implement `find` and `union` with path compression and union by size or rank. Initially every vertex forms a separate component.

**Input:** `task="dsu"`, `n`, and `operations`, an array of pairs `[u,v]` representing undirected connections.

**Output:** `states`, containing `[components,largest]` after each operation. Here `components` is the number of connected components and `largest` is the number of vertices in the largest component. Do not include an initial state before the first operation.

Repeated unions and self-unions are valid and must not change the partition. Initially `largest=1` when `n>0`; for `n=0`, `largest=0` and `states=[]`. If you use union by rank, maintain component sizes separately: rank is not size.

Example input:

```json
{"task":"dsu","n":4,"operations":[[0,1],[1,2],[0,2],[3,3]]}
```

Output:

```json
{"states":[[3,2],[2,3],[2,3],[2,3]]}
```

**Complexity:** `O(n + q α(n))` amortised time and `O(n+q)` memory, including the states. `α(n)` denotes the inverse Ackermann function; `O((n+q) α(n))` is also an acceptable upper bound.

## 4. Minimum-cost cable network: Kruskal and Prim

Find a minimum spanning forest of an undirected weighted graph. Connect every connected component at minimum total cost without cycles. Count isolated vertices as components.

Write two separate implementations:

- `kruskal`: sort edges by weight and use DSU. Accept an edge only when `find(u) != find(v)`.
- `prim`: use adjacency lists and a priority queue. When one component is complete, restart from an unvisited vertex until all vertices are processed.

**Input:** `task="kruskal"` or `task="prim"`, `n`, and `edges`, an array of `[u,v,weight]`.

**Output:** `cost`, `edge_ids` and `components`. Return the sum of the selected edge weights, their original input IDs and the number of connected components in the input graph.

Parallel edges, equal weights, negative weights and self-loops are allowed. Do not select self-loops or repeat an edge ID. For `c` connected components, a spanning forest contains exactly `n-c` edges. For `n=0`, return `cost=0`, `edge_ids=[]`, `components=0`. Different optimal forests and different orders of selected IDs are accepted. Kruskal and Prim must agree on optimum cost and component count; their selected IDs may differ.

Example input:

```json
{"task":"kruskal","n":4,"edges":[[0,1,4],[1,2,2],[0,2,3]]}
```

One accepted output:

```json
{"cost":5,"edge_ids":[1,2],"components":2}
```

Vertex `3` is isolated. Replacing the task name with `prim` asks the other implementation to solve the same graph.

**Complexity:** Kruskal takes `O(n + m log(m+1))` time. Prim takes `O(n + m log(m+1))` with a lazy heap, or `O((n+m) log(n+1))` with an indexed heap. Both use `O(n+m)` memory.

## 5. Circular dependencies: strongly connected components

An edge `u -> v` represents a directed dependency. Find the maximal groups of mutually reachable vertices using **Kosaraju's algorithm**: two DFS passes, a transposed graph and decreasing first-pass finish order for the second pass. You may start on either graph, provided the second pass uses its transpose.

**Input:** `task="scc"`, `n`, and directed `edges` of the form `[u,v]`.

**Output:** `component`, with one nonnegative integer label per vertex, and `condensation`, an array of `[from_label,to_label]` pairs. Equal labels must identify exactly one SCC. Labels need not be consecutive or follow a particular order.

Include every cross-component directed pair exactly once. Exclude internal edges and duplicate pairs. Isolated vertices form separate SCCs. Duplicate input edges and self-loops are allowed. For `n=0`, both output arrays are empty. Condensation edge order is unrestricted.

Example input:

```json
{"task":"scc","n":4,"edges":[[0,1],[1,0],[1,2],[0,2],[2,3]]}
```

One accepted output:

```json
{"component":[5,5,7,9],"condensation":[[5,7],[7,9]]}
```

**Complexity:** `O(n+m)` expected time with hash-based condensation deduplication and `O(n+m)` memory.

## 6. Preparation schedule: DFS topological ordering

Determine whether all preparation tasks can be ordered. An edge `u -> v` means that `u` must finish before `v` starts.

Implement DFS with three states: unvisited, active and finished. An edge to an active vertex indicates a directed cycle. An edge to a finished vertex alone does not indicate a cycle. Check every component and reverse DFS finish order when no cycle exists.

**Input:** `task="topo"`, `n`, and directed `edges` of the form `[u,v]`. Duplicate edges and self-loops are allowed.

**Output:** `acyclic`, a Boolean, and `order`. For an acyclic graph, return `acyclic=true` and every vertex exactly once, with each edge going forward in the order. For any directed cycle, return `acyclic=false` and `order=[]`. A self-loop is a cycle. For `n=0`, return `acyclic=true` and `order=[]`.

Example input:

```json
{"task":"topo","n":4,"edges":[[0,2],[1,2],[2,3]]}
```

One accepted output:

```json
{"acyclic":true,"order":[1,0,2,3]}
```

**Complexity:** `O(n+m)` time and memory. An explicit DFS stack must retain the progress of each suspended traversal so that neighbours are not repeatedly rescanned.

## 7. Delivery routes: DAG shortest paths

Find shortest paths from a given source in a directed acyclic graph. Negative edge weights are allowed. Check the whole graph for cycles, then relax outgoing edges from reachable vertices in topological order.

**Input:** `task="dag"`, `n`, directed `edges` of the form `[u,v,weight]`, and `source`.

**Output:** `acyclic`, `distance` and `parent_edge`.

- If any component contains a directed cycle, including one unreachable from the source, return `acyclic=false`, `distance=[]`, `parent_edge=[]`.
- Otherwise return `acyclic=true` and arrays of length `n`.
- The source has `distance=0` and `parent_edge=-1`.
- An unreachable vertex has `distance=null` and `parent_edge=-1`.
- Every other vertex has its shortest distance and the original ID of an incoming edge on a shortest source path. The edge must end at that vertex; following parent edges must reach the source.

Never relax from an unreachable vertex. Do not use `-1` as an unreachable distance: it may be a valid finite cost. `parent_edge` stores an edge ID, not a predecessor vertex. Parallel edges and tied shortest routes may produce different accepted parent arrays. A self-loop makes the graph cyclic.

Example input:

```json
{"task":"dag","n":5,"source":0,"edges":[[0,1,3],[0,2,8],[1,2,-7],[2,3,4]]}
```

One accepted output:

```json
{"acyclic":true,"distance":[0,3,-4,0,null],"parent_edge":[-1,0,2,3,-1]}
```

Add a helper that reconstructs a vertex sequence from the selected `parent_edge` IDs. A reachable route must run from source to target; the route to the source is `[source]`. Choose and document a no-route result for an unreachable target, for example an empty list or a null value. This choice concerns the helper only; JSON `distance` must still use `null` for unreachable vertices. Call the helper in your own tests for a reachable target, the source and an unreachable target. Also test tied or parallel edges. In the example, the route to vertex `3` is `[0,1,2,3]`, with cost `0`.

The helper is not a separate JSON task. **Complexity:** `O(n+m)` time and memory for DAG shortest paths; `O(L)` time and output space to reconstruct a route containing `L` vertices.

## 8. Tests and analysis

Under `tests/`, include inputs, independently justified expected answers, actual outputs and commands. Cover applicable boundary cases: empty/disconnected graphs, isolates, repeated/self unions, loops, parallel/tied edges, negative weights, totals beyond 32 bits, unreachable vertices and cycles outside source reachability. Test large inputs and deep chains within the published limits, and repeat mixed requests in one process to check state reset.

Check non-unique answers semantically: forest feasibility and optimum cost, SCC membership and exact condensation, topological precedence, and shortest distances with valid parent paths. Where applicable, show that your checking accepts an alternative valid answer and rejects an invalid witness.

Create `ANALYSIS.md` at the repository root and write it in English. For each method, explain the steps, invariant, correctness, boundary cases and time/memory complexity. Explain state reset and deep DFS. Include actual route-helper calls, results and selected edge IDs for reachable, source, unreachable and tied/parallel-edge cases; verify direction and total weight.

Benchmark Kruskal and Prim on the **same graphs at three sizes**. Verify correct costs and valid forests first. At each size, record `n`, `m`, at least three timings per method and their medians. Give the generator/seed or stored inputs, commands, runtime, hardware, timer and warm-up. Exclude compilation; state whether graph construction, parsing and input/output are included, using the same boundary for both methods. Explain observations and limitations separately from the asymptotic argument.

## 9. Repository and submission

Select **Use this template → Create a new repository**. In `submission.json`, set `language` to `Java`, `Python`, `Cpp` or `CSharp`. Java is selected by default:

```json
{"language":"Java"}
```

Complete the six methods and add your route helper in the selected source directory. The entry points and JSON adapters are supplied.

| Language | Algorithm file | Entry point |
|---|---|---|
| Java | `src/Java/Algorithms.java` | `src/Java/Main.java`, class `Main`, no package |
| Python | `src/Python/algorithms.py` | `src/Python/main.py` |
| C++ | `src/Cpp/algorithms.hpp` | `src/Cpp/main.cpp` |
| C# | `src/CSharp/Algorithms.cs` | `src/CSharp/Program.cs`, `App.csproj`, `net8.0`, output `App.dll` |

The stubs already prepare `n`, `source`, `operations` and `edges`: Java/C# use `int` and `int[][]`, C++ uses `int` and `vector<vector<int>>`, and Python uses integers and lists. The row `edges[i]` has original ID `i`. Input conversion and JSON input/output are supplied; you do not need to edit those support files. Use `long` (Java/C#), `long long` (C++) or Python integers for totals. For nullable distances, use `Long[]`, `long?[]`, JSON `J()` or Python `None`, respectively.

Return the fields from the task contract. Once your DSU code has computed `states`, package it as follows:

```java
Map<String, Object> result = new java.util.LinkedHashMap<>();
result.put("states", states);
return result;
```

```python
return {"states": states}
```

```cpp
// Use J::A for a JSON array and J::O for an object.
return J::O{{"states", states}};
```

```csharp
JsonObject result = new JsonObject();
result["states"] = System.Text.Json.JsonSerializer.SerializeToNode(states);
return result;
```

Keep the supplied interface. Additional source files may be placed directly in the same source directory. Use Java 17-compatible code. Include required dependencies as source files; the solution must build and run without downloading packages.

Push your work: **Actions → Public checks** runs automatically in your copy. For local checks, install Python 3.12+ and your selected runtime: JDK 17+, Python 3.12+, `g++` with C++17 support, or the .NET 8 SDK. Run from the repository root:

```sh
python tools/run.py --setup
python tools/run.py --check
python tools/run.py --input tests/my_cases.jsonl
```

Create your own JSON Lines file for the last command; it prints actual results without comparing expected answers. `--setup` checks launch; `--check` checks all 36 public cases in `tools/public.json` and repeats the batch twice in one process. Public tests include deep chains of 2,000 vertices; also test the full published limits. Compilation is limited to 60 seconds and an input batch to 20 seconds. On Windows, `py -3` can replace `python`. Use `--language Python` (or another supported value) to override the manifest for one run. Python runs the checker even when your solution is Java; Actions provides the tools for you.

In this template, Actions checks the tools and launch of all four languages; in your copy, it also checks your algorithms. TODO bodies are unfinished work; setup success alone does not mean the algorithms pass. Public algorithm checks must pass before submission.

Your submitted commit must contain the selected source code, `submission.json`, `tests/`, completed English `ANALYSIS.md` and your build/run/test commands in `ANALYSIS.md`. Submit the **exact commit URL** in Moodle Online text, then click **Submit assignment** and check the status. An Actions URL may be added as evidence. Give me access if your repository is private. Follow the Moodle deadline and approved extensions; when replacing work before the deadline, update the commit link.

## 10. Assessment

The practical rubric is worth **50 points**; the separate Moodle quiz is worth **50 points**. Together they form Assignment 3 out of 100, with the existing 20% contribution to the second attestation.

**Public checks report pass/fail only. They do not calculate the practical grade or transfer marks to Moodle.** Your practical grade follows the 18-group rubric below; the separate grading tool is not included in this repository.

| Method | Core | Edge cases | Witness | Scale | Total |
|---|---:|---:|---:|---:|---:|
| DSU | 4 | 2 | — | 2 | 8 |
| Kruskal | 4 | — | 2 | 1 | 7 |
| Prim | 4 | — | 2 | 1 | 7 |
| Kosaraju SCC | 5 | — | 3 | 2 | 10 |
| DFS topological ordering | 4 | 2 | — | 2 | 8 |
| DAG shortest paths | 5 | — | 3 | 2 | 10 |
| **Total** | **26** | **4** | **10** | **10** | **50** |

Core checks results and optimum values; Edge cases checks boundaries; Witness checks forest edges, condensation or parent paths; Scale checks large inputs. A dash means no group of that type. Each of the 18 groups earns its maximum multiplied by the fraction of checks passed, rounded half-up to 0.01. Sum the group scores, up to 50; accept alternative valid optima.

The specified algorithms, helper, analysis, own tests and benchmark are required learning work without separate manual points or discretionary deductions. Public tests do not replace the full input requirements or the separate grading process.

The [Assignment 3 defence quiz](https://lms.astanait.edu.kz/mod/quiz/view.php?id=93521) is in English: 20 questions in 20 minutes, with one attempt. There are 15 single-answer questions and 5 multiple-response questions (select all that apply). Each question is worth 1 raw mark. Single-answer questions earn 1 for the correct choice and 0 otherwise. In multiple-response questions with two correct options, each selected correct option earns 0.50 and each selected incorrect option deducts 0.25. With three correct options, the weights are +1/3 and −1/6. Add the selected weights; the minimum question score is 0. Blank answers earn 0. Selecting every option earns 0.25 or approximately 0.67, respectively. Full credit requires all correct options and no incorrect options. The quiz mark is `raw / 20 × 50`, up to 50 points. Course policies, including corrections and academic integrity, follow the syllabus.

## 11. Academic integrity

Follow the syllabus. In `ANALYSIS.md`, acknowledge external sources and AI tools, their contributions and your own verification; state if none were used. You are responsible for correctness, originality and quality. Report only tests and measurements you actually performed.
