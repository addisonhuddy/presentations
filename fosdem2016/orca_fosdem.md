# ORCA
## A modular approach to query optimizer

#### @addisonhuddy

---
# 20 min

- Little about query optimization
- Introduce ORCA
- ORCA Internals
- ORCA Roadmap
- Example: adding a transformation

---

# What it's all about

```sql
PREPARE query(int, int) AS SELECT sum(bar) FROM test
    WHERE id > $1 AND id < $2
    GROUP BY foo;

EXPLAIN ANALYZE EXECUTE query(100, 200);

                                                       QUERY PLAN
-------------------------------------------------------------------------------------------------------------------------
 HashAggregate  (cost=39.53..39.53 rows=1 width=8) (actual time=0.661..0.672 rows=7 loops=1)
   ->  Index Scan using test_pkey on test  (cost=0.00..32.97 rows=1311 width=8) (actual time=0.050..0.395 rows=99 loops=1)
         Index Cond: ((id > $1) AND (id < $2))
 Total runtime: 0.851 ms
(4 rows)
```
---

# Why care about query optimization?

* Turns queries (SQL, MapReduce, ...) into an execution plan
* Data growth > Processing growth
* So many optimizers!
* Combination of NP-hard optimization problems

---

# Why care about ORCA?

* Modular
* Extensible
* Plugable

<!-- Cyclomatic Complexity
Maximum: 102 (Orca 8.5)
Minimum: 6.4 (Orca 1.5) -->


# => R&D test-bed

---

![inline 350%](/Users/pivotal/Dropbox/presentations/fosdem2016/logo-hawq.png)
![inline 275%](/Users/pivotal/Dropbox/presentations/fosdem2016/logo-greenplum.png)

---
# TPC-DS 5X Faster[^1]

![original fit](/Users/pivotal/Dropbox/presentations/fosdem2016/gporcaPerf.png)

[^1]:© 2014 ACM, used with permission. Figure 3. TCP-DS performance testing results of Pivotal Greenplum with Pivotal Query Optimizer vs. Pivotal Greenplum with “planner” query optimizer.

---

# Now Open Source
![original filtered](/Users/pivotal/Dropbox/presentations/fosdem2016/github.png)

---

# What makes it so unique?

![](/Users/pivotal/Dropbox/presentations/fosdem2016/multiply.gif)

## Logical & || Physical Operators

---

## Pre-processing -> Exploration -> Implementation -> Optimization

> ALL possible logical plans into its physical operators
- Dr. Venkatesh "Venky" Raghavan

<!-- The logical and physical operators describe how a query or update was executed. The physical operators describe the physical implementation algorithm used to process a statement, for example, scanning a clustered index. Each step in the execution of a query or update statement involves a physical operator. The logical operators describe the relational algebraic operation used to process a statement, for example, performing an aggregation. Not all steps used to process a query or update involve logical operations. -->

---

# 1,000,000,000

---
# Many Logical transformations

- Join Ordering Algorithm
- Expand NAry Join
- Expand NAry Join Min Card Algorithm
- Expand NAry Join Dynamic Programming
- Select to Filter
- Select to IndexGet
- Simplify Select With Subquery

## +76 more from what I could count on the plane

---

# Key Features

* Smarter partition elimination
* Subquery unnesting
* Common table expressions (CTE)
* Join ordering
* Sort order optimization
* Skew awareness

---

<!-- Internals Overview -->
![fit](/Users/pivotal/Dropbox/presentations/fosdem2016/orcaInternals.png)

---

# Adding A Tranformation


---

# What can't it do?




---

# Four pieces of hanging fruit

1. Distinguish between Physical and Logical
1. Move expression evaluation inside ORCA
1. ORCA assumes indexes are all forward access
1. Constraint evaluation from a `NOT NULL`

---
<!-- END SLIDE -->

# Get Involved

### github.com/greenplum-db
### gpdb-dev@greenplum.org
### Pivotal Tracker: bit.ly/1m1WGDn
### White Paper: bit.ly/1ntrE8v
### @addisonhuddy
