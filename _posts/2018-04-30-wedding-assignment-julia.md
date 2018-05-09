---
title:  "Wedding Table Assignment Problem using Julia"
date:   2018-04-30 9:00AM
categories: [julia, optimization]
excerpt: "A Julia port from Python of the Wedding Table Assignment example from [PuLP](https://github.com/coin-or/pulp/blob/master/examples/wedding.py)."
---

This is a port from Python of the Wedding Table Assignment example from [PuLP](https://github.com/coin-or/pulp/blob/master/examples/wedding.py){:target="_blank"}.


This is an example of a **set partitioning** problem. We have a _universe_ of guests and subsets of tables that have to _exactly_ cover our universe, i.e. the guests.


```julia
using JuMP, Cbc
using Combinatorics
```

We start with a little helper function to calculate "happiness" of a table, determined by how close the guests are (by their letter):
```julia
function happiness(table):
    """
    Find the happiness of the table
    - by calculating the maximum distance between the letters
    """
    return abs(Int(table[1][1]) - Int(table[end][1]))
end
```

We set up the number of table, table size, and an array of our guests:
```julia
max_tables = 5
max_table_size = 4
guests = ["A" "B" "C" "D" "E" "F" "G" "I" "J" "K" "L" "M" "N" "O" "P" "Q" "R"]
```




    1×17 Array{String,2}:
     "A"  "B"  "C"  "D"  "E"  "F"  "G"  "I"  …  "L"  "M"  "N"  "O"  "P"  "Q"  "R"



Then we generate the possible combinations of guests to tables up the maximum table size:
```julia
table_combos = [collect(combinations(guests, t)) for t in 1:max_table_size];
```


```julia
table_combos[end][end]    
```




    4-element Array{String,1}:
     "O"
     "P"
     "Q"
     "R"



And we flatten that out:
```julia
possible_tables = []
for i in 1:length(table_combos)
    append!(possible_tables, table_combos[i])
end
```

We end up with `3,213` possible tables. Time to run an optimizer to pick the best combos!
```julia
length(possible_tables)
```




`3213`



### Our Model:
```julia
m = Model(solver=CbcSolver())
num_possible_tables = length(possible_tables)
idx_possible_tables = 1:num_possible_tables

@variable(m, table_assignment[idx_possible_tables], Bin)

# Objective: maximize happiness = minimize happiness value
@objective(m, Min, sum([happiness(possible_tables[t]) * table_assignment[t]
    for t in idx_possible_tables]))

@constraint(m, sum([table_assignment[t]
    for t in idx_possible_tables]) <= max_tables)


# because this is a set partitioning problem, we enforce a strict equality constraint
# - i.e. every guest has to be seated
for guest in guests
    @constraint(m, sum([table_assignment[t] for t in idx_possible_tables
        if guest in possible_tables[t]]) == 1)
end

;
```
(This is a fairly large model, so we won't print it)

```julia
status = solve(m)

println("Objective value: ", getobjectivevalue(m))
```

    Objective value: 12.0


Collective happiness is 12, apparently. :)

```julia
table_assignment
```




$$ table_assignment_{i} \in \{0,1\} \quad\forall i \in \{1,2,\dots,3212,3213\} $$



We end up with the following table assignments, optimized by guest happiness:
```julia
[("Table: ", possible_tables[i], "Happiness: ", happiness(possible_tables[i]))
    for i=1:length(table_assignment) if getvalue(table_assignment[i]) == 1 ]
```




    5-element Array{Tuple{String,Array{String,1},String,Int64},1}:
     ("Table: ", String["Q", "R"], "Happiness: ", 1)          
     ("Table: ", String["E", "F", "G"], "Happiness: ", 2)     
     ("Table: ", String["A", "B", "C", "D"], "Happiness: ", 3)
     ("Table: ", String["I", "J", "K", "L"], "Happiness: ", 3)
     ("Table: ", String["M", "N", "O", "P"], "Happiness: ", 3)
