+++
title = "The day the Earth stood still"
date = "2023-09-06"

[taxonomies]
tags=["physics", "python", "rust"]

[extra]
repo_view = false
+++

Assume the following thought experiment:
The Earth circling the Sun at a cozy distance of 1 AU (around 150 million km)
suddenly stands still and doesn't move anymore. 
The only force left on the Earth is now the Sun's gravitational pull.
How long will it be until we crash into the Sun?

<!--more--> 

## An analytical solution

The Sun with mass \$m\_s\$ and the Earth with mass \$m\_e\$ are located at distance \$r\$ from each other.
In this scenario (an leaving all vector arrows away since everything takes place in one dimension here),
the gravitational force between the two bodies can be written as

\begin{equation}
    F = G \frac{m_s m_e}{r^{2}}.
\end{equation}

The force and acceleration the Earth is feeling is

\begin{equation}
    F = m_e a
\end{equation}

and setting both equations equal we get

\begin{equation}
    a = \ddot{r} = G \frac{m_s}{r^2}.
\end{equation}

Note that the mass of the Earth dropped out of the equation.
We now have a differential equation that we could solve in order to determine the free fall time. 
The solution when solving for the time as a function of position can be seen,
e.g., on [Wikipedia](https://en.wikipedia.org/wiki/Free_fall#Inverse-square_law_gravitational_field).


## Kepler's third law

A common way to solve this problem in astronomy is 
by using 
[Kepler's third law](https://en.wikipedia.org/wiki/Kepler%27s_laws_of_planetary_motion)
of planetary motions, 
which states that the square of a planet's orbital period
is proportional to the cube of the length of the semi-major axis of the planet's orbit.
Let us assume that the Earth's orbit is a circle 
(which is close to reality).
The semi-major axis is therefore simply twice the previously defined distance \$r\$.
Furthermore, the orbital period of the Earth is 1 year (or 365.24 days).

The free fall time \$t\$ can be calculated by imagining that the Earth travels around the Sun
in a degenerate elliptic orbit,
such that it travels radially inward towards it and then on the same line back out again.
Using Kepler's third law allows us to write the following equation:

\begin{equation}
    \frac{(2t)^2}{r^3} = \frac{p_e^2}{(2r)^3}
\end{equation}

Here, \$p\_e\$ is the orbital period of the Earth,
i.e., 1 year.
Solving this equation for \$t\$ results in

\begin{equation}
    t = \sqrt{\frac{p_e^2}{32}} = 64.52\\,\\mathrm{d}.
\end{equation}


## A numerical solution

If we do not know about Kepler's third law and do not want to solve the differential equation,
we can still solve this problem numerically.
The difficulty we need to keep in mind is that the acceleration of Earth is not constant
but depends on the distance \$r\$.
We can split the distance between the Sun and the Earth into small segments \$dr\$.
Within such a segment, we can assume that the acceleration is constant.
This allows us to apply the equation for distance traveled under constant acceleration:

\begin{equation}
    dr = \frac{a}{2} dt^2 + v_0 dt
\end{equation}

Here, \$v\_0\$ is the initial velocity of the Earth at the beginning of the segment under consideration
and is zero for the first step (see the title of this post).
The time it takes to traverse segment \$dr\$ is \$dt\$.
Rewriting above equation and solving for the time \$dt\$ results in

\begin{align}
    \frac{a}{2} dt^2 + v_0 dt - dr &= 0 \\\\
    dt_{1,2} &= \frac{-v_0 \pm \sqrt{v_0^2 + 2 a dr}}{a}.
\end{align}

Clearly, only the positive solution is of interest here
since it is the only one that results in a positive time \$t\$.

To calculate the time it takes for the Earth to crash into the Sun
we can now simply sum over all segments \$dr_i\$,
calculate the time \$dt_i\$ it takes to cross segment \$i\$,
and then calculate the total time \$t\$ as:

\begin{equation}
    t = \sum dt_i.
\end{equation}

### An implementation in Python

The following Python code implements this algorithm:

```python
import time

import numpy as np

# Constants
GRAV = 6.67408e-11 # m^3 kg^-1 s^-2
M_SUN = 1.98847e30 # kg
R_START = 1.495979e11 # m

# Step size
dr = 1e3  # m

def total_time(dr):
    # Initial conditions
    t_total = 0  # s: total time
    v_curr = 0  # m/s: initial velocity
    r_curr = R_START  # m: initial distance

    # Calculate time until Earth crashes into the Sun
    while r_curr > 0:
        # Calculate acceleration
        a_curr = GRAV * M_SUN / r_curr**2

        # Calculate time it takes to travel dr
        t_curr = (-v_curr + np.sqrt(v_curr**2 + 2 * a_curr * dr)) / a_curr

        # Update total time
        t_total += t_curr

        # Update velocity
        v_curr += a_curr * t_curr

        # Update distance
        r_curr -= dr
    
    return t_total

# Print result

tic = time.time()
t_total = total_time(dr)
toc = time.time()
print(f"Time until Earth crashes into the Sun: {t_total / 3600 / 24:.2f} days")
print(f"Computation time: {toc - tic:.2f} seconds")
```

Here we wrote the actual calculation into a little function,
which makes it easier to time it.
Running this routine on my computer results in the following output:

```
Time until Earth crashes into the Sun: 64.57 days
Computation time: 136.57 seconds
```

While we get the correct answer, 
the computation time is over two minutes.
Let's see if we can speed this up a bit.


### A faster implementation in Python using [`numba`](https://numba.pydata.org/)

The 
[`numba`](https://numba.pydata.org/)
package allows us to take the function `total_time(dr)`
and compile it to machine code before running it.
Here, just-in-time compilation is used,
which means that the compilation happens at runtime when the function is used for the first time.

The only thing we need to change is to import the package
and add a decorator to the function:

```python
...

from numba import njit

...

@njit
def total_time(dr):
    ...
```

Three dots in above example indicate that the rest of the code is unchanged.
Running the code again results in the following output:

```
Time until Earth crashes into the Sun: 64.57 days
Computation time: 1.84 seconds
```

The computation time is now only 1.84 seconds,
which is a speedup of a factor of 74!

We can ask the question how long it actually took to compile the function.
To find out, we can simply run the calculation twice and only measure the time for the second run:
The output from this is:

```
Time until Earth crashes into the Sun: 64.57 days
Computation time: 1.57 seconds
```

Just-in-time compilation thus took around 250 ms.

### A comparison with Rust

Just as a comparison, here is the output when implementing the same algorithm in Rust
and building the code with the `--release` flag:


```
Time until Earth crashes into the Sun: 64.56 days
Computation time: 1.58s.
```

The computation time is basically identical to the 
[`numba`](https://numba.pydata.org/)
implementation in Python.

This clearly shows the power of 
[`numba`](https://numba.pydata.org/)
and just-in-time compilation.

### Source code

The final python and Rust source code can be found 
[here on GitHub](https://gist.github.com/trappitsch/35d04ef43a8a53a2bd9ea513032da61a).


### Resources

- [Website for `numba`](https://numba.pydata.org/)
- [Documentation for `numba`](https://numba.readthedocs.io/en/stable/user/index.html)