---
layout: post
title:  "F1 Fantasy optimization"
date:   2021-11-13 12:00:00 +0200
categories: optimization
---

Ever since watching the famous Netflix series *Drive to survive* I've been hooked on F1. If you haven't 
watched it, I highly recommend  it (The current F1-season will be the fourth season).


Next to the F1 season, there also exists something called F1 Fantasy, where you pick 5 drivers and a team 
and get points from how well your drivers and team does in the real races. More info regarding F1 Fantasy
can be found [here](https://fantasy.formula1.com/).


However, I thought it would be cool to formulate the choice of drivers and team as an optimization problem, 
and when you dive into it, you realize that it quite easily can be rewritten into a Mixed-Integer Linear Program.


# Mixed-Integer programming
Mixed-Integer programming (MIP) might sound scary at first, but it is just a standard Linear Programming problem
where some, or all decision variables are restricted to integer values. The standard MIP-problem is usually written as 

$$
\begin{align}
\text{min} \quad   &c^T \, z \\
\text{subject to} \quad  &A_{eq} \, z = b_{eq} \\
 &A \, z \leq b
\end{align} 
$$

where $z$ is a vector with the decision parameters and some, or all elements of $z$ are constrained to integer values. I would say that it's more important to understand what goes into 
standard Linear programming since MIP is an extension of that. In case you haven't heard about Stephen Boyd's lectures 
on convex optimization (that includes LP's) now's the time to find a [pure gold-mine](https://www.youtube.com/watch?v=McLq1hEq3UY&list=PL3940DD956CDF0622).


It's worth mentioning that MIP can solve a lot of cooler and more interesting optimization problems, such as scheduling-problems. This [paper](https://doi.org/10.1016/j.cirpj.2021.07.012)
uses nonlinear Mixed-Integer programming to find the optimal scheduling for industrial robots and their energy optimal cycle time.


# Implementation

To formulate and solve the MIP problem, I used the [Google OR-tools library](https://developers.google.com/optimization) that allows for both
modelling of the MIP and also includes a solver. I wrote my implementation in Python, but Google OR-tools exists for other languages as well.


My implementation can be found on my [github](https://github.com/eliasrhoden/f1_fantasy_optimization) in case you are interested in the details,
here I will explain the overall concept and how I ended up with a MIP-problem.


## F1 Fantasy as a MIP 
Assume that we have $N$ teams (and thereby $2N$ drivers), we then define two decision vectors $z^D \in \mathrm{I}^{2N}$ (One for each driver) and $z^T \in \mathrm{I}^N$ (one for each team). Each element of the $z$ vector are limited to the range $[0,\ 1]$ (and integer values) making them *boolean variables*.  Where they represent if a team/driver is picked or not.


As an example, $z_1^D$ is equal to $1$ if driver 1 should be picked into the lineup, and similar for the team vector.


In F1 fantasy you need to pick 5 drivers and a team, while being on a budget. We need to formulate this as a constraint for the optimization problem, it must pick 5 drivers 
and 1 team. No more, no less. This is quite easy to formulate based on how we defined our $z$ vectors.

The constraints are simply

$$
\sum_{i=1}^{2N} \, z_i^D = 5, \quad \sum_{i=1}^{N} \, z_i^T = 1.
$$


The next constraint is the budget, assuming we have a budget of $B$ million dollars, and that each driver and team have their corresponding costs $C^D_i$ and $C^T_i$. The budget constraint can then be 
formulated as 

$$
\sum_{i=1}^{2N} \, C^D_i \, z_i^D + \sum_{i=1}^{N} \, C^T_i\,z_i^T \leq B.
$$


Lastly we want to formulate the points scored by the lineup, what we actually want to maximize. It is very similar to how we formulated the budget constraint, assuming that each driver and team have scored $P^D_i$ and $P^T_i$ respectively.
Then the cost to maximize $J$ is 


$$
J = \sum_{i=1}^{2N} \, P^D_i \, z_i^D + \sum_{i=1}^{N} \, P^T_i\,z_i^T.
$$


And that's it! We now have formulated the problem of finding the optimal lineup within the budget, as a linear MIP. And since it is linear (the problem is convex) the optimal solution from the solver will be a **global optimum**.


Have in mind that this optimization does not account for special selections such as Turbo driver and Mega driver. This might be in scope for next year's F1 fantasy?


I also want to mention that this would not have been possible without the data I found on [kaggle](https://www.kaggle.com/prathamsharma123/formula-1-fantasy-2021).
It did needed some cleanup due to inconsistent  naming, and I also included a json-file of the [cleaned data](https://github.com/eliasrhoden/f1_fantasy_optimization/blob/master/clean_data.json) on my github-repo.

## The optimal lineup*
*Without regard to Mega and turbo-drivers.

As mentioned in my Github-readme, I optimized 3 cases. 
1. The same lineup throughout the entire season. 
2. Unlimited substitutions.
3. The same lineup for last 5 races.


### Optimal lineup for the whole season
```
Team: Red Bull
Drivers: 
Max Verstappen
Carlos Sainz
Lando Norris
Piere Gasly
George Russell
```

### Optimal lineup for each GP
```
--- Bahrain GP ---
Team: Red Bull
Drivers: 
Max Verstappen
Valtteri Bottas
Lando Norris
George Russell
Mick Schumacher

--- Imola GP ---
Team: McLaren
Drivers: 
Max Verstappen
Charles Leclerc
Carlos Sainz
Lando Norris
Kimi Raikkonen

--- Portugal GP ---
Team: Red Bull
Drivers: 
Max Verstappen
Sergio Perez
Lando Norris
Esteban Ocon
Mick Schumacher

--- Spain GP ---
Team: Red Bull
Drivers: 
Max Verstappen
Daniel Ricciardo
Charles Leclerc
Nicholas Latifi
George Russell

--- Monaco GP ---
Team: Red Bull
Drivers: 
Max Verstappen
Sebastian Vettel
Carlos Sainz
Lando Norris
Nikita Mazepin

--- Azerbaijan GP ---
Team: Red Bull
Drivers: 
Sergio Perez
Sebastian Vettel
Fernando Alonso
Lando Norris
Piere Gasly

--- French GP ---
Team: Red Bull
Drivers: 
Max Verstappen
Sergio Perez
Lando Norris
Antonio Giovinazzi
George Russell

--- Styrian GP ---
Team: Red Bull
Drivers: 
Max Verstappen
Valtteri Bottas
Kimi Raikkonen
Yuki Tsunoda
Mick Schumacher

--- Austrian GP ---
Team: Red Bull
Drivers: 
Max Verstappen
Carlos Sainz
Lando Norris
Piere Gasly
Mick Schumacher

--- British GP ---
Team: Ferrari
Drivers: 
Lewis Hamilton
Charles Leclerc
Lando Norris
Yuki Tsunoda
George Russell

--- Hungarian GP ---
Team: Alpine
Drivers: 
Lewis Hamilton
Fernando Alonso
Carlos Sainz
Piere Gasly
Esteban Ocon

--- Belgian GP ---
Team: Williams
Drivers: 
Lewis Hamilton
Max Verstappen
Daniel Ricciardo
Piere Gasly
George Russell

--- Dutch GP ---
Team: Red Bull
Drivers: 
Max Verstappen
Fernando Alonso
Lando Norris
Piere Gasly
Nicholas Latifi

--- Italian GP ---
Team: McLaren
Drivers: 
Valtteri Bottas
Daniel Ricciardo
Charles Leclerc
Lando Norris
George Russell

--- Russian GP ---
Team: Mercedes
Drivers: 
Daniel Ricciardo
Fernando Alonso
Carlos Sainz
Kimi Raikkonen
George Russell

--- Turkish GP ---
Team: Red Bull
Drivers: 
Valtteri Bottas
Sergio Perez
Charles Leclerc
Antonio Giovinazzi
George Russell

--- USA GP ---
Team: Red Bull
Drivers: 
Max Verstappen
Sergio Perez
Charles Leclerc
George Russell
Mick Schumacher
```

### Optimal lineup for the last 5 races
As of writing this, the last GP was in the US. 
```
Dutch GP
Italian GP
Russian GP
Turkish GP
USA GP

Team: Red Bull
Drivers: 
Valtteri Bottas
Daniel Ricciardo
Carlos Sainz
Lando Norris
Mick Schumacher
```
