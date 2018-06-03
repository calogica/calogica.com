---
title:  "A Warehouse Allocation Example using Julia"
date:   2018-05-01 9:00AM
categories: [julia, optimization]
excerpt: "We solve a textbook optimization example involving planning multiple warehouse locations using Julia"
mathjax: true
---

## Location of Warehouses
This is based on _Example 5.1 (Location of Warehouses)_ from [Applied Linear Programming](https://www.wiley.com/en-us/Applied+Integer+Programming%3A+Modeling+and+Solution-p-9780470373064){:target="_blank"}

A firm has 5 distribution centers and we want to determine which subset of these should serve as a site for a warehouse. The goal is the build a minimum number of warehouses that can cover all distribution centers so that every warehouse is within 10 miles of each distribution center.

Per the problem statement, $$m$$ is the number of distribution centers.

We are given a table of distances between distribution centers, $$D$$:

```julia
m = 5
max_miles = 10

D = [0 10 15 20 18;
     10 0 20 15 10;
     15 20 0 8 17;
     20 15 8 0 5;
     18 10 17 5 0
    ]
```




    5×5 Array{Int64,2}:
      0  10  15  20  18
     10   0  20  15  10
     15  20   0   8  17
     20  15   8   0   5
     18  10  17   5   0



For example, it is *18* miles between distribution centers *1* (column 1) and *5* (row 5).

To convert this to a binary coverage vector $$A$$, we convert each distance into a binary variable indicating whether the distribution centers are 10 or fewer miles from one another:


```julia
A = [Int(D[i, j] <= max_miles) for i=1:m, j=1:m]
```




    5×5 Array{Int64,2}:
     1  1  0  0  0
     1  1  0  0  1
     0  0  1  1  0
     0  0  1  1  1
     0  1  0  1  1



Now we can model this problem using the [JuMP](https://www.juliaopt.org/) package and the (open source) `Cbc` solver:

(First we import the relevant packages)
```julia
using JuMP, Cbc
```


```julia
model = Model(solver=CbcSolver())

# decision variable (binary): whether to build warehouse near distribution center i
@variable(model, y[1:m], Bin)

# Objective: minimize number of warehouses
@objective(model, Min, sum(y))

# Constraint: has to cover all warehouses
# (.>= is the element-wise dot comparison operator)
@constraint(model, A*y .>= 1)

model
```




$$ \begin{alignat*}{1}\min\quad & y_{1} + y_{2} + y_{3} + y_{4} + y_{5}\\
\text{Subject to} \quad & y_{1} + y_{2} \geq 1\\
 & y_{1} + y_{2} + y_{5} \geq 1\\
 & y_{3} + y_{4} \geq 1\\
 & y_{3} + y_{4} + y_{5} \geq 1\\
 & y_{2} + y_{4} + y_{5} \geq 1\\
 & y_{i} \in \{0,1\} \quad\forall i \in \{1,2,3,4,5\}\\
\end{alignat*}
 $$



We have an additional constraint that at least 1 warehouse should be within 10 miles of distribution center 1, but our activity matrix $$A$$ already covers that, so technically we do not need this explicit constraint.


```julia
@constraint(model, y[1] + y[2] >= 1)
```




$$ y_{1} + y_{2} \geq 1 $$




```julia
# Solve problem using MIP solver
status = solve(model)
```




    :Optimal




```julia
println("Total # of warehouses: ", getobjectivevalue(model))

println("Build warehouses at distribution center(s):")

[i for i=1:m if getvalue(y[i]) == 1 ]
```

    Total # of warehouses: 2.0
    Build warehouses at distribution center(s):

    2-element Array{Int64,1}:
     2
     3

We should build warehouses at distribution centers 2 and 3.
