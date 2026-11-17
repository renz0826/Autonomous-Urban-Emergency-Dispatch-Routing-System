# Module 1: Dynamic Route Planning (A*) build guide

Issue: [#1 Implement A* Search to navigate the ambulance](https://github.com/renz0826/Autonomous-Urban-Emergency-Dispatch-Routing-System/issues/1)
Branch: `feat/module-1-dynamic-route-planning`

The implementation cell is yours to write. Add the pieces below in order into the Module 1 code slot. Each piece is small and explained.

Cell 2 already provides: a directed 50x50 grid (`nx.grid_2d_graph(..., create_using=nx.DiGraph)`), edge `weight` (1.0 clear, 3.0 to 8.0 congested, `inf` flooded), edge `is_blocked`, node `hazard_state` (`clear`, `flooded`, `debris`), and `get_manhattan_heuristic(node_a, node_b)`.

## Step 1: Import the priority queue and define the failure error

```python
import heapq  # keeps numbers sorted automatically

class NoRouteError(Exception):
    """Raised when flooding makes the destination unreachable."""
```

`heapq` keeps entries sorted automatically: push costs `log n`, pop of the smallest costs `log n`. A plain list cannot do "always expand the cheapest estimate" cheaply. The custom exception gives a clear failure message instead of `None`.

## Step 2: Sanity checks before searching

```python
def plan_dynamic_route(self, origin, destination):
    if origin not in self.graph or destination not in self.graph:
        raise ValueError("origin and destination must be grid cells")
    if self.graph.nodes[destination]["hazard_state"] == "flooded":
        raise NoRouteError(f"destination {destination} is flooded")
```

Cells are `(x, y)` tuples, which is what `self.hubs[k]` and `hotspots[i]["coords"]` return. A flooded destination can never be reached, so check it first and state it clearly.

## Step 3: The three bookkeeping structures

```python
    g_score = {origin: 0.0}   # cheapest minutes found so far to reach each cell
    came_from = {}            # cell -> the cell we arrived from
    counter = 0               # tie-breaker number
    heap = []
    h = self.get_manhattan_heuristic
```

`g_score` answers "what is the best time to stand at this intersection?". Start with the hub at 0 minutes. `came_from` answers "how did I get here?", filled only when a cell's time improves. `counter` exists because heap entries are tuples `(f, counter, cell, g)`; when two cells share the same `f`, Python compares the second field, and an integer counter never crashes where two `(x, y)` tuples would. `h` is the notebook's Manhattan guess, which is safe because the fastest street costs 1 minute per block.

## Step 4: Push the start

```python
    heapq.heappush(heap, (0.0 + h(origin, destination), counter, origin, 0.0))
    # entry = (estimate f, tie-break, cell, minutes spent g)
```

At the hub, `g = 0`, so `f` is the guess of remaining blocks. This single entry starts the search.

## Step 5: The loop: pop, ignore stale entries, stop at the goal

```python
    while heap:
        f, _, cell, g_pushed = heapq.heappop(heap)

        if g_pushed > g_score[cell]:
            continue                      # outdated entry: a better path arrived later

        if cell == destination:
            break                         # first goal-pop is provably optimal
```

`heappop` always returns the smallest `f`, the intersection whose total trip estimate is cheapest. A cell can be pushed several times as better paths appear, so old pops must be discarded by comparing the stored `g_at_push` against `g_score[cell]`. When the goal pops, its `g` is final, because the guess never overestimates remaining minutes.

## Step 6: Look at every street leaving the popped cell

```python
        for nxt in self.graph.successors(cell):
            edge = self.graph.edges[cell, nxt]

            if edge["is_blocked"] or edge["weight"] == float("inf"):
                continue                  # flooded street: closed, do not compute anything
            if self.graph.nodes[nxt]["hazard_state"] == "flooded":
                continue                  # flooded intersection: never enter it

            new_g = g_score[cell] + edge["weight"]   # 1.0 clear, 3.0 to 8.0 jam
            if new_g < g_score.get(nxt, float("inf")):
                came_from[nxt] = cell
                g_score[nxt] = new_g
                counter += 1
                heapq.heappush(heap, (new_g + h(nxt, destination), counter, nxt, new_g))
```

`successors` gives streets leaving the cell in the direction you can travel. The grid is directed, so east-bound and west-bound streets are separate records, and congestion can exist on only one direction. Always read `self.graph.edges[cell, nxt]` for the direction actually traveled. Blocked edges are skipped before arithmetic; adding `inf` to `g` would make every estimate containing it huge and still disturb comparisons. `new_g` is real minutes for the next cell, and only when it beats `g_score.get(nxt, inf)` do you record the predecessor and push the new estimate `new_g + guess`.

## Step 7: After the loop: route, breakdown, or failure

```python
    else:
        raise NoRouteError("no route: flood wall separates origin and destination")

    route = [destination]
    while route[-1] != origin:
        route.append(came_from[route[-1]])
    route.reverse()

    breakdown = []
    for a, b in zip(route[:-1], route[1:]):
        w = self.graph.edges[a, b]["weight"]
        kind = "congested" if self.graph.edges[a, b]["congestion_factor"] > 1.0 else "clear"
        breakdown.append({"from": a, "to": b, "minutes": w, "kind": kind})

    return route, g_score[destination], breakdown
```

The `else` here attaches to the `while` loop at the same indentation, and runs only when the loop finished without `break`. That means the frontier emptied with nothing reachable left, which is a true real-world answer, not a bug. Rebuilding walks `came_from` backwards from the goal until it hits the hub, then reverses to get hub to incident. The breakdown labels each street as clear (1 minute) or congested (3 to 8 minutes) for the demo cell and Module 4 to consume.

## Attaching the method to the environment class

```python
MetroCityEnvironment.plan_dynamic_route = plan_dynamic_route
```

This line must sit at module level, after the function definition, not inside `NoRouteError` and not inside the function body. Class bodies run at definition time; a function body runs only when called, which is exactly the thing that would not exist yet.

## Behavior worth checking while writing

Hub 2 blocks from the incident, direct street jammed plus 5 minutes: the direct route estimate becomes `1 + 5 = 6`, the detour estimate becomes `3 + 1 = 4`, so the detour pops first and wins. Flood the destination itself and Step 2 raises. Flood a middle cell and Step 6 skips it, so the route bends around the wall.

## Before committing this cell

Run `nbstripout --extra-keys="metadata.kernelspec metadata.language_info"` before committing, since the CI workflow `check-notebooks.yml` rejects committed notebooks carrying outputs or kernelspec metadata. Headless execution needs a writable matplotlib directory: `MPLCONFIGDIR=<writable-dir> ./.venv/bin/python ...`.
