---
jupytext:
  text_representation:
    extension: .md
    format_name: myst
    format_version: 0.13
    jupytext_version: 1.14.5
kernelspec:
  display_name: Python 3 (ipykernel)
  language: python
  name: python3
---


# Endogenous Grid Method


```{include} _admonition/gpu.md
```

## Overview

In this lecture we use the endogenous grid method (EGM) to solve a basic income
fluctuation (optimal savings) problem.

Background on the endogenous grid method can be found in [an earlier
QuantEcon lecture](https://python.quantecon.org/egm_policy_iter.html).

Here we focus on providing an efficient JAX implementation.

We will use the following libraries and imports.

```{code-cell} ipython3
:tags: [hide-output]

!pip install --upgrade quantecon interpolation
```

```{code-cell} ipython3
import quantecon as qe
import matplotlib.pyplot as plt
import numpy as np
import jax
import jax.numpy as jnp

from interpolation import interp
from numba import njit, float64
from numba.experimental import jitclass
```

Let's check the GPU we are running

```{code-cell} ipython3
!nvidia-smi
```

We use 64 bit floating point numbers for extra precision.

```{code-cell} ipython3
jax.config.update("jax_enable_x64", True)
```


## Setup 

We consider a household that chooses a state-contingent consumption plan $\{c_t\}_{t \geq 0}$ to maximize

$$
\mathbb{E} \, \sum_{t=0}^{\infty} \beta^t u(c_t)
$$

subject to

$$
    a_{t+1} \leq  R(a_t - c_t)  + Y_{t+1},
    \quad c_t \geq 0,
    \quad a_t \geq 0
    \quad t = 0, 1, \ldots
$$

Here $R = 1 + r$ where $r$ is the interest rate.

The income process $\{Y_t\}$ is a [Markov chain](https://python.quantecon.org/finite_markov.html) generated by stochastic matrix $P$.

The matrix $P$ and the grid of values taken by $Y_t$ are obtained by
discretizing the AR(1) process

$$
    Y_{t+1} = \rho Y_t + \nu \epsilon_{t+1}
$$

where $\{\epsilon_t\}$ is IID and standard normal.

Utility has the CRRA specification

$$
    u(c) = \frac{c^{1 - \gamma}} {1 - \gamma}
$$


The following function stores default parameter values for the income
fluctuation problem and creates suitable arrays.

```{code-cell} ipython3
def ifp(R=1.01,             # gross interest rate
        β=0.99,             # discount factor
        γ=1.5,              # CRRA preference parameter
        s_max=16,           # savings grid max
        s_size=200,         # savings grid size
        ρ=0.99,             # income persistence
        ν=0.02,             # income volatility
        y_size=25):         # income grid size
  
    # Create income Markov chain
    mc = qe.tauchen(y_size, ρ, ν)
    y_grid, P = jnp.exp(mc.state_values), mc.P
    # Shift to JAX arrays
    P, y_grid = jax.device_put((P, y_grid))
    
    s_grid = jnp.linspace(0, s_max, s_size)
    sizes = s_size, y_size
    s_grid, y_grid, P = jax.device_put((s_grid, y_grid, P))

    # require R β < 1 for convergence
    assert R * β < 1, "Stability condition violated."
        
    return (β, R, γ), sizes, (s_grid, y_grid, P)
```


## Solution method

Let $S = \mathbb R_+ \times \mathsf Y$ be the set of possible values for the
state $(a_t, Y_t)$.

We aim to compute an optimal consumption policy $\sigma^* \colon S \to \mathbb
R$, under which dynamics are given by

$$
    c_t = \sigma^*(a_t, Y_t)
    \quad \text{and} \quad
    a_{t+1} = R (a_t - c_t) + Y_{t+1}
$$


In this section we discuss how we intend to solve for this policy.


### Euler equation

The Euler equation for the optimization problem is

$$
    u' (c_t)
    = \max \left\{
        \beta R \,  \mathbb{E}_t  u'(c_{t+1})  \,,\;  u'(a_t)
    \right\}
$$

An explanation for this expression can be found [here](https://python.quantecon.org/ifp.html#value-function-and-euler-equation).

We rewrite the Euler equation in functional form

$$
    (u' \circ \sigma)  (a, y)
    = \max \left\{
    \beta R \, \mathbb E_y (u' \circ \sigma)
        [R (a - \sigma(a, y)) + \hat Y, \, \hat Y]
    \, , \;
         u'(a)
         \right\}
$$


where $(u' \circ \sigma)(a, y) := u'(\sigma(a, y))$ and $\sigma$ is a consumption
policy.

For given consumption policy $\sigma$, we define $(K \sigma) (a,y)$ as the unique $c \in [0, a]$ that solves

$$
u'(c)
= \max \left\{
           \beta R \, \mathbb E_y (u' \circ \sigma) \,
           [R (a - c) + \hat Y, \, \hat Y]
           \, , \;
           u'(a)
     \right\}
$$ (eq:kaper)

It [can be shown that](https://python.quantecon.org/ifp.html)

1. iterating with $K$ computes an optimal policy and
2. if $\sigma$ is increasing in its first argument, then so is $K\sigma$

Hence below we always assume that $\sigma$ is increasing in its first argument.

The EGM is a technique for computing the update $K\sigma$ given $\sigma$ along a grid of asset values.

Notice that, since $u'(a) \to \infty$ as $a \downarrow 0$, the second term in
the max in [](eq:kaper) dominates for sufficiently small $a$.

Also, again using [](eq:kaper), we have $c=a$ for all such $a$.

Hence, for sufficiently small $a$,

$$
   u'(a) \geq
   \beta R \, \mathbb E_y (u' \circ \sigma) \,
           [\hat Y, \, \hat Y]
$$

Equality holds at $\bar a(y)$ given by 

$$
   \bar a (y) =
   (u')^{-1}
   \left\{
       \beta R \, \mathbb E_y (u' \circ \sigma) \,
               [\hat Y, \, \hat Y]
   \right\}
$$

We can now write

$$
u'(c)
    = \begin{cases}
        \beta R \, \mathbb E_y (u' \circ \sigma) \,
               [R (a - c) + \hat Y, \, \hat Y]
               & \text{if } a > \bar a (y) \\
        u'(a)  & \text{if } a \leq \bar a (y)
    \end{cases}
$$

Equivalently, we can state that the $c$ satisfying $c = (K\sigma)(a, y)$ obeys

$$
c = \begin{cases}
        (u')^{-1}
        \left\{
            \beta R \, \mathbb E_y (u' \circ \sigma) \,
               [R (a - c) + \hat Y, \, \hat Y]
        \right\}
               & \text{if } a > \bar a (y) \\
            a  & \text{if } a \leq \bar a (y)
    \end{cases}
$$ (eq:oro)

We begin with an *exogenous* grid of saving values $0 = s_0 < \ldots < s_{N-1}$

Using the exogenous savings grid, and a fixed value of $y$, we create an *endogenous* asset grid
$a_0, \ldots, a_{N-1}$ and a consumption grid $c_0, \ldots, c_{N-1}$ as follows.

First we set $a_0 = c_0 = 0$, since zero consumption is an optimal (in fact the only) choice when $a=0$.

Then, for $i > 0$, we compute 

$$
    c_i
    = (u')^{-1}
    \left\{ 
        \beta R \, \mathbb E_y (u' \circ \sigma) \,
               [R s_i + \hat Y, \, \hat Y]
     \right\}
     \quad \text{for all } i
$$ (eq:kaperc)

and we set 

$$
    a_i = s_i + c_i 
$$ 

We claim that each pair $a_i, c_i$ obeys [](eq:oro).

Indeed, since $s_i > 0$, choosing $c_i$ according to [](eq:kaperc) gives

$$
    c_i
    = (u')^{-1}
    \left\{ 
        \beta R \, \mathbb E_y (u' \circ \sigma) \,
               [R s_i + \hat Y, \, \hat Y]
     \right\}
     \geq \bar a(y)
$$

where the inequality uses the fact that $\sigma$ is increasing in its first argument.

If we now take $a_i = s_i + c_i$ we get $a_i > \bar a(y)$, so the pair $(a_i, c_i)$ satisfies

$$
    c_i
    = (u')^{-1}
    \left\{ 
        \beta R \, \mathbb E_y (u' \circ \sigma) \,
               [R (a_i - c_i) + \hat Y, \, \hat Y]
     \right\}
     \quad \text{and} \quad a_i > \bar a(y)
$$


Hence [](eq:oro) holds.


We are now ready to iterate with $K$.

### JAX version 

First we define a vectorized operator $K$ based on the EGM.

Notice in the code below that 

* we avoid all loops and any mutation of arrays
* the function is pure (no globals, no mutation of inputs)

```{code-cell} ipython3
def K_egm(a_in, σ_in, constants, sizes, arrays):
    """
    The vectorized operator K using EGM.

    """
    
    # Unpack
    β, R, γ = constants
    s_size, y_size = sizes
    s_grid, y_grid, P = arrays
    
    def u_prime(c):
        return c**(-γ)

    def u_prime_inv(u_prime):
            return u_prime**(-1/γ)

    # Linearly interpolate σ(a, y)
    def σ(a, y):
        return jnp.interp(a, a_in[:, y], σ_in[:, y])
    σ_vec = jnp.vectorize(σ)

    # Broadcast and vectorize
    y_hat = jnp.reshape(y_grid, (1, 1, y_size))
    y_hat_idx = jnp.reshape(jnp.arange(y_size), (1, 1, y_size))
    s = jnp.reshape(s_grid, (s_size, 1, 1))
    P = jnp.reshape(P, (1, y_size, y_size))
    
    # Evaluate consumption choice
    a_next = R * s + y_hat
    σ_next = σ_vec(a_next, y_hat_idx)
    up = u_prime(σ_next)
    E = jnp.sum(up * P, axis=-1)
    c = u_prime_inv(β * R * E)

    # Set up a column vector with zero in the first row and ones elsewhere
    e_0 = jnp.ones(s_size) - jnp.identity(s_size)[:, 0]
    e_0 = jnp.reshape(e_0, (s_size, 1))

    # The policy is computed consumption with the first row set to zero
    σ_out = c * e_0

    # Compute a_out by a = s + c
    a_out = np.reshape(s_grid, (s_size, 1)) + σ_out
    
    return a_out, σ_out
```


Then we use `jax.jit` to compile $K$.

We use `static_argnums` to allow a recompile whenever `sizes` changes, since the compiler likes to specialize on shapes.

```{code-cell} ipython3
K_egm_jax = jax.jit(K_egm, static_argnums=(3,))
```


Next we define a successive approximator that repeatedly applies $K$.

```{code-cell} ipython3
def successive_approx_jax(model,        
            tol=1e-5,
            max_iter=100_000,
            verbose=True,
            print_skip=25):

    # Unpack
    constants, sizes, arrays = model
    
    β, R, γ = constants
    s_size, y_size = sizes
    s_grid, y_grid, P = arrays
    
    # Initial condition is to consume all in every state
    σ_init = jnp.repeat(s_grid, y_size)
    σ_init = jnp.reshape(σ_init, (s_size, y_size))
    a_init = jnp.copy(σ_init)
    a_vec, σ_vec = a_init, σ_init
    
    i = 0
    error = tol + 1

    while i < max_iter and error > tol:
        a_new, σ_new = K_egm_jax(a_vec, σ_vec, constants, sizes, arrays)    
        error = jnp.max(jnp.abs(σ_vec - σ_new))
        i += 1
        if verbose and i % print_skip == 0:
            print(f"Error at iteration {i} is {error}.")
        a_vec, σ_vec = jnp.copy(a_new), jnp.copy(σ_new)

    if error > tol:
        print("Failed to converge!")
    else:
        print(f"\nConverged in {i} iterations.")

    return a_new, σ_new
```


### Numba version 

Below we provide a second set of code, which solves the same model with Numba.

The purpose of this code is to cross-check our results from the JAX version, as
well as to do a runtime comparison.

Most readers will want to skip ahead to the next section, where we solve the
model and run the cross-check.

```{code-cell} ipython3
ifp_data = [
    ('R', float64),              
    ('β', float64),             
    ('γ', float64),            
    ('P', float64[:, :]),     
    ('y_grid', float64[:]),  
    ('s_grid', float64[:])    
]

# Use the JAX IFP data as our defaults for the Numba version
model = ifp()
constants, sizes, arrays = model
β, R, γ = constants
s_size, y_size = sizes
s_grid, y_grid, P = (np.array(a) for a in arrays)

@jitclass(ifp_data)
class IFP:

    def __init__(self,
                 R=R,
                 β=β,
                 γ=γ,
                 P=np.array(P),
                 y_grid=np.array(y_grid),
                 s_grid=s_grid):

        self.R, self.β, self.γ = R, β, γ
        self.P, self.y_grid = P, y_grid
        self.s_grid = s_grid

        # Recall that we need R β < 1 for convergence.
        assert self.R * self.β < 1, "Stability condition violated."

    def u_prime(self, c):
        return c**(-self.γ)
    
    def u_prime_inv(self, u_prime):
        return u_prime**(-1/self.γ)
```

```{code-cell} ipython3
@njit
def K_egm_nb(a_in, σ_in, ifp):
    """
    The operator K using Numba.

    """
    
    # Simplify names
    R, P, y_grid, β, γ  = ifp.R, ifp.P, ifp.y_grid, ifp.β, ifp.γ
    s_grid, u_prime = ifp.s_grid, ifp.u_prime
    u_prime_inv = ifp.u_prime_inv
    n = len(y_grid)
    
    # Linear interpolation of policy using endogenous grid
    def σ(a, z):
        return interp(a_in[:, z], σ_in[:, z], a)
    
    # Allocate memory for new consumption array
    σ_out = np.zeros_like(σ_in)
    a_out = np.zeros_like(σ_out)
    
    for i, s in enumerate(s_grid[1:]):
        i += 1
        for z in range(n):
            expect = 0.0
            for z_hat in range(n):
                expect += u_prime(σ(R * s + y_grid[z_hat], z_hat)) * \
                            P[z, z_hat]
            c = u_prime_inv(β * R * expect)
            σ_out[i, z] = c
            a_out[i, z] = s + c
    
    return a_out, σ_out
```

```{code-cell} ipython3
def successive_approx_numba(model,        # Class with model information
              tol=1e-5,
              max_iter=100_000,
              verbose=True,
              print_skip=25):

    # Unpack
    P, s_grid = model.P, model.s_grid
    n = len(P)
    
    σ_init = np.repeat(s_grid, y_size)
    σ_init = np.reshape(σ_init, (s_size, y_size))
    a_init = np.copy(σ_init)
    a_vec, σ_vec = a_init, σ_init
    
    # Set up loop
    i = 0
    error = tol + 1

    while i < max_iter and error > tol:
        a_new, σ_new = K_egm_nb(a_vec, σ_vec, model)
        error = np.max(np.abs(σ_vec - σ_new))
        i += 1
        if verbose and i % print_skip == 0:
            print(f"Error at iteration {i} is {error}.")
        a_vec, σ_vec = np.copy(a_new), np.copy(σ_new)

    if error > tol:
        print("Failed to converge!")
    else:
        print(f"\nConverged in {i} iterations.")

    return a_new, σ_new
```


## Solutions

Here we solve the IFP with JAX and Numba.

We will compare both the outputs and the execution time.

### Outputs

```{code-cell} ipython3
ifp_jax = ifp()
```

```{code-cell} ipython3
ifp_numba = IFP()
```


Here's a first run of the JAX code.

```{code-cell} ipython3
a_star_egm_jax, σ_star_egm_jax = successive_approx_jax(ifp_jax,
                                         print_skip=100)
```


Next let's solve the same IFP with Numba.

```{code-cell} ipython3
qe.tic()
a_star_egm_nb, σ_star_egm_nb = successive_approx_numba(ifp_numba,
                                         print_skip=100)
qe.toc()
```


Now let's check the outputs in a plot to make sure they are the same.

```{code-cell} ipython3
fig, ax = plt.subplots()

n = len(ifp_numba.P)
for z in (0, y_size-1):
    ax.plot(a_star_egm_nb[:, z], 
            σ_star_egm_nb[:, z], 
            '--', lw=2,
            label=f"Numba EGM: consumption when $z={z}$")
    ax.plot(a_star_egm_jax[:, z], 
            σ_star_egm_jax[:, z], 
            label=f"JAX EGM: consumption when $z={z}$")

ax.set_xlabel('asset')
plt.legend()
plt.show()
```


### Timing

Now let's compare execution time of the two methods

```{code-cell} ipython3
qe.tic()
a_star_egm_jax, σ_star_egm_jax = successive_approx_jax(ifp_jax,
                                         print_skip=1000)
jax_time = qe.toc()
```

```{code-cell} ipython3
qe.tic()
a_star_egm_nb, σ_star_egm_nb = successive_approx_numba(ifp_numba,
                                         print_skip=1000)
numba_time = qe.toc()
```

```{code-cell} ipython3
jax_time / numba_time
```


The JAX code is significantly faster, as expected.

This difference will increase when more features (and state variables) are added
to the model.

```{code-cell} ipython3

```

```{code-cell} ipython3

```
