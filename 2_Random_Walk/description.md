# Project goal

Simulate many particles performing 2D random walks, visualize trajectories (static + animated), compute statistics (e.g., mean squared displacement), and optionally add boundaries/obstacles.

# Prerequisites

* Python, NumPy, Matplotlib.
* Familiarity with arrays, broadcasting, `np.cumsum`, `np.linalg.norm`.
* (Optional) `matplotlib.animation.FuncAnimation` for animations.

---

# High-level milestones

## **Beginner (core)**

- [x] Single-particle random walk (lattice & continuous).
- [x] Vectorize multiple independent walkers using NumPy (no Python loops).
- [ ] Plot trajectories (line plot) and final positions (scatter).

## **Intermediate (features & analysis)**
- [ ] Add boundary conditions: reflecting, periodic (wrap), absorbing.
- [ ] Add obstacles (e.g., circular) and simple collision response (reject or reflect).
- [ ] Animate the walk and add interactive controls (if in notebook).
- [ ] Compute and plot statistics: MSD(t), ensemble averages; estimate diffusion coefficient.

## **Optional advanced experiments**

- [ ] Biased walks, variable step sizes, Levy flights, inter-particle interactions, performance optimizations (numba/chunks).

---

# Step-by-step plan with tasks

## Step 1 — Single-particle prototypes (quick checks)

- [ ] Implement a simple lattice random walk (4-neighbor: up/down/left/right).
- [ ] Implement continuous-angle walk: choose angle θ ∼ Uniform(0,2π), step length `step_size`.

- Purpose: understand one-walker behaviour and plotting.

## Step 2 — Vectorized multi-particle simulation (core)

- [ ] Represent steps shape `(n_steps, n_particles, 2)`.
- [ ] For continuous angles: generate `theta = np.random.uniform(0, 2*np.pi, size=(n_steps, n_particles))`, then `dx = step_size * np.cos(theta)`, `dy = ...`.
- [ ] Positions = `np.cumsum(steps, axis=0)` → shape `(n_steps, n_particles, 2)`.

Benefits: uses broadcasting, no Python loop over particles.

## Step 3 — Visualization

- [ ] Trajectories: plot a small subset (10–50) of particles as line plots (x vs y over time).
- [ ] Final positions: scatter plot colored by particle index or distance.
- [ ] Use subplots to show MSD and final spread.

## Step 4 — Add boundaries and obstacles

- [ ] **Periodic (wrap)**: after computing new positions, `pos = (pos - min) % width + min` — fully vectorizable.
- [ ] **Reflecting**: simpler to handle step-by-step: compute tentative `pos_new = pos + step`; find where `pos_new` crosses bounds, compute reflected coordinate by reflecting the overshoot: `pos_new = 2*bound - pos_new` for those coords. This generally requires iterating over time steps (still vectorized across particles).
- [ ] **Absorbing**: mark particles that exit as inactive and stop updating them.
- [ ] **Obstacle (circular)**: detect `dist(center, pos_new) < radius`. Simple rule: reject the step (keep old position) — easiest. Later implement reflection off the circle normal for realism.

## Step 5 — Animation

- [ ] Use `matplotlib.animation.FuncAnimation`. For many particles, animate a subsample, or animate as scatter with 2D coordinates. Use blitting for speed.

## Step 6 — Analysis & metrics

- [ ] **MSD(t)**: `msd[t] = np.mean(np.sum(positions[t]**2, axis=1))` if origin start. For ensemble average across particles.
- [ ] For 2D simple random walk: `MSD(t) ~ 4 * D * t`. Fit `msd vs t` to estimate diffusion coefficient `D ≈ slope/4`.

## Step 7 — Tests, debugging & performance

- [ ] Start with small `n_particles` & `n_steps` to debug (e.g., 10 × 200).
- [ ] Verify shapes (`steps.shape`, `positions.shape`).
- [ ] Plot a few particle trajectories to detect bugs.
- [ ] For large simulations: use `float32`, chunking (simulate in blocks), or numba for step-by-step physics.

---

# Example implementations

### 1) Vectorized continuous-angle random walk (minimal, core)

```python
import numpy as np

def simulate_continuous_walk(n_particles=100, n_steps=1000, step_size=1.0, seed=None):
    rng = np.random.default_rng(seed)
    # angles: shape (n_steps, n_particles)
    theta = rng.uniform(0, 2*np.pi, size=(n_steps, n_particles))
    # steps shape (n_steps, n_particles, 2)
    steps = np.stack((np.cos(theta), np.sin(theta)), axis=-1) * step_size
    # positions: cumsum over time -> (n_steps, n_particles, 2)
    positions = np.cumsum(steps, axis=0)
    # prepend origin (optional) -> shape (n_steps+1, n_particles, 2)
    origins = np.zeros((1, n_particles, 2))
    positions = np.vstack((origins, positions))
    return positions  # positions[t, p] -> (x,y) at time t
```

### 2) Lattice random walk (vectorized)

```python
def simulate_lattice_walk(n_particles=100, n_steps=1000, seed=None):
    rng = np.random.default_rng(seed)
    # choices: 0=right,1=left,2=up,3=down
    choices = rng.integers(0, 4, size=(n_steps, n_particles))
    dx = np.where(choices==0, 1, np.where(choices==1, -1, 0))
    dy = np.where(choices==2, 1, np.where(choices==3, -1, 0))
    steps = np.stack((dx, dy), axis=-1)
    positions = np.cumsum(steps, axis=0)
    positions = np.vstack((np.zeros((1, n_particles, 2)), positions))
    return positions
```

### 3) Compute MSD and estimate D

```python
def compute_msd(positions):
    # positions shape (T, N, 2)
    r2 = np.sum(positions**2, axis=2)  # shape (T, N)
    msd = np.mean(r2, axis=1)          # shape (T,)
    return msd

# estimate D with linear fit on msd[t] ~ 4 D t (skip t=0)
from numpy.linalg import lstsq
def estimate_diffusion(msd, t_start=1):
    t = np.arange(len(msd))[t_start:]
    A = np.vstack([t, np.ones_like(t)]).T
    slope, intercept = lstsq(A, msd[t_start:], rcond=None)[0]
    D = slope / 4.0
    return D, slope, intercept
```

### 4) Simple periodic boundary (wrap)

```python
def apply_periodic(positions, x_min, x_max, y_min, y_max):
    width = x_max - x_min
    height = y_max - y_min
    positions[...,0] = ((positions[...,0] - x_min) % width) + x_min
    positions[...,1] = ((positions[...,1] - y_min) % height) + y_min
    return positions
```

### 5) Reflecting boundary (step-by-step outline)

* For reflecting, you typically need to update `pos` sequentially over time steps:

```python
pos = np.zeros((n_particles,2))
positions = []
for t in range(n_steps):
    step = generate_step_for_all_particles(...)  # shape (n_particles,2)
    tentative = pos + step
    # reflect x
    over_x = tentative[:,0] > x_max
    tentative[over_x,0] = 2*x_max - tentative[over_x,0]
    under_x = tentative[:,0] < x_min
    tentative[under_x,0] = 2*x_min - tentative[under_x,0]
    # similarly for y
    # update pos and store
    pos = tentative
    positions.append(pos.copy())
positions = np.stack(positions, axis=0)
```

This is vectorized across particles but loops in time (usually fine).

---

# Visualization / Animation (quick example)

```python
import matplotlib.pyplot as plt
from matplotlib.animation import FuncAnimation

# assume `positions` from simulate_continuous_walk (T, N, 2)
T, N, _ = positions.shape
fig, ax = plt.subplots(figsize=(6,6))
ax.set_aspect('equal')
lines = [ax.plot([], [], lw=1, alpha=0.7)[0] for _ in range(min(20, N))]  # plot subset

def init():
    ax.set_xlim(-100, 100)
    ax.set_ylim(-100, 100)
    for ln in lines:
        ln.set_data([], [])
    return lines

def update(frame):
    # show full trajectory up to frame for subset of particles
    for i, ln in enumerate(lines):
        p = positions[:frame+1, i, :]
        ln.set_data(p[:,0], p[:,1])
    return lines

ani = FuncAnimation(fig, update, frames=T, init_func=init, blit=True, interval=30)
plt.show()
# To save: ani.save('walk.mp4', dpi=150)  # requires ffmpeg
```

---

# Debugging & sanity checks

* Start with `n_particles=5, n_steps=200` and visually inspect trajectories.
* Check `positions.shape == (n_steps+1, n_particles, 2)`.
* Plot `msd` and confirm roughly linear growth for simple random walk.
* For lattice walk, mean displacement ≈ 0 (unbiased). For biased walk, expect linear drift.

---

# Performance tips & memory

* Memory ~ `n_steps * n_particles * 2 * 8 bytes` (float64). Use `float32` if needed.
* If `n_steps * n_particles` too big, save positions at intervals (every 10 steps) or aggregate online metrics (msd) without keeping full history.
* Use vectorized ops; only loop in time if necessary for boundaries/obstacles.

---

# Extension ideas / experiments

* Levy flights (power-law step lengths).
* Inter-particle forces (repulsion / Lennard-Jones style).
* Time-dependent step-size (temperature schedule).
* Fit MSD to detect anomalous diffusion (MSD ∝ t^α, find α).
* Add GUI controls (sliders for n_particles, step_size) in a notebook.

---

# Minimal full starter (put together)

(Use the `simulate_continuous_walk` above; then plot a few trajectories and compute MSD — copy/paste those blocks to run.)

---

If you want, I can:

* Give a complete runnable Jupyter notebook combining simulation, plots, and animation.
* Or produce a version that uses reflecting collisions with circular obstacles (with code to reflect across the normal).

Which of those would help you next?
