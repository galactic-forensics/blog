+++
title = "Seven in a row"
date = "2024-03-01"

[taxonomies]
tags = ["python", "science"]

[extra]
repo_view = false
+++


One of my favorite radio shows is [Radiolab](https://radiolab.org/), which always presents the listener with an interesting scientific question from across the board. In a recent rerun, Radiolab discussed [Stochasticity](https://radiolab.org/podcast/91684-stochasticity/transcript). Something struck me as odd...

<!-- more -->

I decided to do some math and figure out what was going on. And of course, what is simply math without thinking about how to also do this in python!

# Randomness

In the episode, they present an experiment of flipping a coin one hundred times and recording the outcome on a board. One group actually flips the coin and writes the actual results down, the other group makes up the results and writes down those results. Afterwards, the examiner walks in to evaluate the results and immediately points out which one is the true random sequence. The way she discovered it is by looking at the longest streaks. The true random group had a much longer streak - a seven in a row - than the group that made the results up.

The probability to get seven tails in a row can be calculated as

$$p = 0.5^{7} = 0.78\%,$$

which is very low. However, remember that they flipped one hundred times. This - according to the episode - will increase the chance of getting seven in row to about one in six.

# The problem

My issue with the episode was the following dialog (transcript taken directly from [here](https://radiolab.org/podcast/91684-stochasticity/transcript):

> JAD: And then Jay explained it to us: if you're just doing seven flips then yeah, getting seven in a row is really unlikely. But if you're doing multiple sets of seven ...
> 
> JAY KOEHLER: Fourteen of those sets of seven.
> 
> JAD: ... which we were, because we were doing a hundred. Then the probabilities start to add up. And we start small, like one percent. But then that one becomes two, which becomes four, which becomes eight until, when it's all said and done, the chances of getting seven tails in a row somewhere in a set of a hundred is—don't hold your breath.

This statement didn't make much sense to me, since any seven in a row in the series of 100 flips would work and not just one of the fourteen!

# Let's do the math

To calculate the probability of getting seven in a row out of one hundred flips, let's first define what this even means: In order to have exactly seven in a row, the flip before and after (if we are somewhere in between flip one and flip 100) must be of the opposite side. This means that to have seven tails, the flip before and after must be heads. Therefore, we need to get in fact nine in a row correct! The probability that the first or last seven flips are of the same suit requires the flip after or before that row to be different, thus for these edge cases we only need to get eight in a row correct. 

As so often with probability calculations, let us turn the problem around and ask the question of what is the probability that the two edge cases are not seven in a row. We can calculate the edge cases as following:

$$p_\mathrm{edge} = 1-0.5^8$$

For one hundred flips, to not get seven in a row at the beginning and the end, i.e., for the two edge cases, the probability then becomes:

$$p_\mathrm{edges} = (p_\mathrm{edge})^2 = (1-0.5^8)^2$$

For every case in between - remember we need nine in a row correct - the probability to not have seven in a row can be calculated as:

$$p_\mathrm{inside} = 1-0.5^9$$

In a series of 100, we have a total of 92 times seven flips in a row that could fit the bill, i.e., flips 2 through 8 (with 1 and 9 being of a different face), flips 3 through 9 (with 2 and 10 being of a different face), ... This gets us to a probability of not having seven in a row of the same face for the in-between cases as:

$$p_\mathrm{inside-cases} = (1-0.5^9)^{92}$$

Now that we have all cases covered, we can determine the total probability of not having seven in a row as

$$p_\mathrm{not} = p_\mathrm{edges} \cdot p_\mathrm{inside-cases} = (1-0.5^8)^2 \cdot (1-0.5^9)^{92}$$

Finally, we can turn this around again and determine the probability that we get at least one time seven in a row in one hundred flips as

$$p_\mathrm{result} = 1 - p_\mathrm{not} = 17.11\%.$$

This is pretty close to the answer on the show of one in six, which would be around 16.67%.

# Flipping coins in python

Let's do a simulation of this experiment in python. We are going to simulate `nmc` times `nflips` coin flips in a row and count how many times we get at least one seven in a row. Dividing the total count by `nmc` should then give us the same result as we have determined in the calculation above.


```python
from numba import njit
import numpy as np

@njit
def check_flips(all_flips: np.ndarray) -> np.ndarray:
    """Count how many times we get seven in a row.
    
    The routine is decorated with the `@njit` decorator, which we
    import from `numba`. This does a just-in-time compilation 
    of the function in order to speed things up.
    Feel free to remove the decorator and see the difference!

    The routine is provided an array of all flips with `nmc`
    rows and `nflips` columns.

    :param all_flips: An array of all flips, either 0 or 1.
        Dimension: `nmc` times `nflips`

    :return: An array of length `nmc` stating how many times
        seven in a row occured in this set of `nflips`.
    """
    results = np.zeros(all_flips.shape[0])

    for it, series in enumerate(all_flips):
        # check first 7
        if np.sum(series[:7]) == 7 and series[7] == 0:
            results[it] += 1

        # check last 7
        if np.sum(series[-7:]) == 7 and series [-8] == 0:
            results[it] += 1

        # check all in between
        for jt in range(1, nflips - 1 - 7):
            if np.sum(series[jt:jt+7]) == 7:  # row is 7
                # check that bounds are not 1
                if series[jt-1] == 0 and series[jt+7] == 0:
                    results[it] += 1

    return results
```


```python
# some starting values
nmc = 10_000_000
nflips = 100
```


```python
# let's generate all the flips using numpy's random.randint routine
all_flips = np.random.randint(0, 2, size=(nmc, nflips))
```


```python
# get the result from our routine that we defined above
seven_in_a_row = check_flips(all_flips)
```


```python
# Calculate the probability to get at least one seven in a row
prob = len(np.where(seven_in_a_row > 0)[0]) / nmc
print(f"Probability for at least one time seven in a row in {nflips} flips is {prob * 100:.2f}%.")
```

    Probability for at least one time seven in a row in 100 flips is 17.13%.



```python
# Calculate the probability to get exactly one time seven in a row
prob = len(np.where(seven_in_a_row == 1)[0]) / nmc
print(f"Probability for one time seven in a row in {nflips} flips is {prob * 100:.2f}%.")
```

    Probability for one time seven in a row in 100 flips is 15.77%.


# Comparison of the results

As you can see, we get the same result from the Monte Carlo calculation as we got from the analytical calculation. If you are interested in why we used `@njit` before the `check_flips` function, please have a look at the docstring in that function and check out [`numba`](https://numba.pydata.org/). Without this decorator, we would wait a lot longer to simulate 10 million times one hundred flips. Numba was also used in this blog post [here](https://galactic-forensics.space/resources/mind-bytes/suncrash/), where more details are given.

Also, feel free to play with this example! Some questions you might ask:

- How many streaks of at least seven in a row are in the set?
- How many streaks *insert you number here* are in the set?
- How well has the result converged after ten million tries?



[![Binder](https://mybinder.org/badge_logo.svg)](https://mybinder.org/v2/gh/galactic-forensics/mindbytes/HEAD?labpath=radiolab_stochasticity%2Fradiolab_stochasticity.ipynb)