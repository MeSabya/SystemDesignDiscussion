
A vector clock is a versioning mechanism to track the causal history of updates in distributed systems.

Each node maintains a vector (dictionary) of version numbers — one per node.

### ✅ Use Case: Shopping Cart Service
Imagine two concurrent updates:

- User adds Product A from a mobile device in Region A.
- At the same time, they remove Product A from their laptop in Region B.
- Both updates reach the cart database asynchronously.

🔧 Problem:
These updates conflict, but neither “wins” chronologically.

### ✅ Solution:
- Each write carries a vector clock.
- When the system detects divergent vector clocks:
- It surfaces conflicting versions.
- A resolution strategy (manual or automatic merge) is applied.
- E.g., last-write-wins, union of cart items, or user confirmation.

Two nodes/replicas (Node A and Node B) each maintain a shopping cart and use vector clocks to track updates.

🧱 Initial Setup
Each node has a vector clock initialized like this:

```css
Node A: {A: 0, B: 0}
Node B: {A: 0, B: 0}
```

The keys (A, B) are node IDs. Values are version counters.

🛒 Step-by-Step Example

### Step 1: User adds "Book" to cart on Node A
Node A increments its own counter:

```css
Vector clock at Node A: {A: 1, B: 0}
```
It sends:

```json
{
  event: "add Book",
  vector_clock: {A: 1, B: 0}
}
```
### Step 2: User adds "Pen" to cart on Node B concurrently
Node B increments its own counter:

```css
Vector clock at Node B: {A: 0, B: 1}
```
It sends:

```json
{
  event: "add Pen",
  vector_clock: {A: 0, B: 1}
}
```
### Step 3: Sync happens — each node receives the other's update
Now both nodes see:

```
Node A sees {A: 0, B: 1} and compares with its {A: 1, B: 0}

Node B sees {A: 1, B: 0} and compares with its {A: 0, B: 1}
```
### What Happens Next?
You must resolve the conflict:

- For shopping carts, a common strategy is merge:
- Final cart contains: ["Book", "Pen"]
- New vector clock: {A: 1, B: 1} (max of each component)



https://www.linkedin.com/pulse/vector-clocks-simple-way-keep-distributed-systems-sync-maheshwari/
