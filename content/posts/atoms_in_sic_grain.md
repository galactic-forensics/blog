+++
title = "The number of atoms in a SiC stardust grain"
date = "2022-03-26"

[taxonomies]
tags = ["stardust", "python"]

[extra]
repo_view = false
+++



Let's calculate how many atoms are in a SiC stardust grain and use python to do so!

<!-- more -->

In this article, we will analyze how many atoms of a given trace element are in a presolar SiC grain of a given size. Furthermore, the Jupyter notebook in which this article was written is available via [`binder`](https://mybinder.org/), so you can adjust it at any point to change the calculations and adopt it to your needs. Click on the icon on top to run this notebook in [`binder`](https://mybinder.org/). Throughout this notebook we will be using SI units.

# Calculation

First, let us calculate the mass of a given SiC grain. We therefore need to define its radius $r$ and it's typical density $\\rho = 3160\\,$kg m$^{-3}$ (according to [Wikipedia](https://en.wikipedia.org/wiki/Silicon_carbide)). Via the volume $V$ for a spherical grain

\begin{equation}
        V = \frac{4}{3} \pi r^3,
\end{equation}

we can calculate the mass of the grain as:

\begin{equation}
        m = V\rho = \frac{4}{3} \pi r^3 \rho
\end{equation}

For a given radius, we can now write a `python` function to calculate the mass as following:


```python
import math

rho = 3160  # kg / m**3
def mass(r: float) -> float:
    """Calculate mass of spherical grain with density `rho`.
    
    :param r: Radius in meters
    
    :return: Mass in kg
    """
    return 4/3 * math.pi * r**3 * rho
```

The approximate molar mass of SiC is the molar mass of a silicon atom plus the molar mass of a carbon atom, therefore $M\_\\mathrm{SiC} = 40\\,$g mol$^{-1}$. Let us assume we have some trace element, e.g., iron, given at a concentration of 10 ppm. If the concentration is by weight $c\_{wt}$, we can directly calculate the mass of all the iron as:

\begin{equation}
    m_\mathrm{Fe} = c_{wt} \times m
\end{equation}

If the concentration is given by number of atoms $c\_n$, we first have to convert it first in order to calculate the mass of iron of the grain. For this, we need to know the molar mass of iron, which is approximately $M\_\\mathrm{Fe} = 56\\,$g mol$^{-1}$. The mass of iron can in this case be calculated as:

\begin{equation}
    m_\mathrm{Fe} = c_n \frac{M_\mathrm{Fe}}{M_\mathrm{SiC}} \times m
\end{equation}

Let us now calculate the mass of the species of interest with `python`:


```python
molar_mass_sic = 40 * 1e-3  # kg / mol

# calculate mass of a species of interest
def mass_species(r: float, mol_mass: float, conc: float, conc_as_weight: bool) -> float:
    """Calculate mass of the given species in a SiC grain.
    
    :param r: Radius of the SiC grain in m.
    :param mol_mass: Molar mass of the species of interest in kg / mol.
    :param conc: Concentration of the species of interest.
    :param conc_as_weight: Is the concentration per weight (`True`) or by number (`False`).
    
    :return: Mass of species in kg
    """
    cwt = conc if conc_as_weight else conc * mol_mass / molar_mass_sic
    mass_species = cwt * mass(r)
    return mass_species
```

Finally, we know that per mol of material, Avogadro's number $N\_A = 6.0221415 \\times 10^{23}$ of atoms are present. For iron, we could therefore calculate the number of atoms as:

\begin{equation}
    n_\mathrm{Fe} = N_A \frac{m_\mathrm{Fe}}{M_\mathrm{Fe}}
\end{equation}

Here, $M\_\\mathrm{Fe}$ is again the molar mass of iron. We can now create our final function in `python`:


```python
n_a = 6.0221415e23

def number_of_atoms(r: float, mol_mass: float, conc: float, conc_as_weight: bool) -> float:
    """Calculate the number of atoms for a given species in a presolar SiC grain.
    
    :param r: Radius of the SiC grain.
    :param mol_mass: Molar mass of the species of interest in kg / mol.
    :param conc: Concentration of the species of interest.
    :param conc_as_weight: Is the concentration per weight (`True`) or by number (`False`).
    
    :return: Mass of species in kg
    """
    m_species = mass_species(r, mol_mass, conc, conc_as_weight)
    n_species = n_a * m_species / mol_mass
    return n_species
```

# Examples

We can now run some examples using the everything we just created.

## Example 1

Assume we have 10 ppm by weight of iron in a grain with 1 µm radius. We can then calculate the number of atoms as:


```python
r = 1e-6
mol_mass = 56e-3
conc = 10e-6
conc_as_weight = True

n_fe = number_of_atoms(r, mol_mass, conc, conc_as_weight)
print(f"Number of iron atoms per grain: {n_fe:.3e}")
```

    Number of iron atoms per grain: 1.423e+06


## Example 2

We can perform the same calculation, assuming that the concentration is per number. Since the molar mass of iron is heavier than the molar mass of SiC, we would expect the total number of atoms to be larger than in example 1. Let's see if that is true:


```python
n_fe = number_of_atoms(r, mol_mass, conc, False)
print(f"Number of iron atoms per grain: {n_fe:.3e}")
```

    Number of iron atoms per grain: 1.993e+06


This simple Jupyter Notebook allows  you to determine how many atoms of a species you would expect in a SiC stardust grain. If you then want to measure the atoms, you'll know what you're up against. Good luck!


[![Binder](https://mybinder.org/badge_logo.svg)](https://mybinder.org/v2/gh/galactic-forensics/mindbytes/HEAD?labpath=atoms_in_sic_grains%2Fatoms_in_sic_grain.ipynb)