# ORCA
## Query optimization as a service

#### @addisonhuddy

---
# 20 min

- Little about query optimization
- Introduce ORCA
- ORCA Internals
- Pairing: adding a transformation
- ORCA Roadmap

---

# Why care about query optimization?

* Turns queries (SQL, MapReduce, ...) into an execution plan
* Data growth > Processing growth
* So many optimizers!

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


---

# Key Features

* Smarter partition elimination
* Subquery unnesting
* Common table expressions (CTE)
* Join ordering
* Sort order optimization
* Skew awareness

---

## Logical & || Physical Operators

---

## Pre-processing -> Exploration -> Implementation -> Optimization

> ALL possible logical plans are turned into its physical operators
- Dr. Venkatesh "Venky" Raghavan

<!-- The logical and physical operators describe how a query or update was executed. The physical operators describe the physical implementation algorithm used to process a statement, for example, scanning a clustered index. Each step in the execution of a query or update statement involves a physical operator. The logical operators describe the relational algebraic operation used to process a statement, for example, performing an aggregation. Not all steps used to process a query or update involve logical operations. -->

<!-- cascade framework -->

---

# Many Logical transformations

- Join Ordering Algorithm
- Expand NAry Join Min Card Algorithm
- Expand NAry Join Dynamic Programming
- Select to Filter
- Select to IndexGet
- Simplify Select With Subquery

## +77 more from what I could count on the plane

---

# 1,000,000,000

<!-- Expression pre-processing.
Exploration Phase where logical transformation rules are fired in the cascade framework of query optimization -->

---

<!-- Internals Overview -->
![fit](/Users/pivotal/Dropbox/presentations/fosdem2016/orcaInternals.png)

---

# Let's Pair

---
# Idea
## Split an aggregate into a pair of local and global aggregate.

```SQL
SELECT sum(c) FROM foo GROUP BY b

CREATE TABLE foo (a int, b int, c int) distributed by (a);
```

<!-- Table is distributed by column a, however the aggregation operation is on column b. One option is to gather all tuples to the master node and compute the aggregation. The drawback of this approach is that the tuples need to gathered on the master node and therefore does not take advantage of being a distributed database system.
An alternate design is to do local aggregation and then send the partial aggregation to the master. The final aggregation can then be done on the master. In the next section, we will walk through how to add such a logical to logical transformation rule inside Orca. -->

---

# `CXformSplitGbAgg`

```c++
// HEADER FILES
~/orca/libgpopt/include/gpopt/xforms

// SOURCE FILES
~/orca/libgpopt/src/xforms
```

---

# Pattern


```c++
GPOS_NEW(pmp)
CExpression
(
	pmp,
	GPOS_NEW(pmp) CLogicalGbAgg(pmp),
  // logical aggregate operator
	GPOS_NEW(pmp) CExpression(pmp, GPOS_NEW(pmp) CPatternLeaf(pmp)),
  // relational child
	GPOS_NEW(pmp) CExpression(pmp, GPOS_NEW(pmp) CPatternTree(pmp))  
  // scalar project list
	));
```

<!-- The constructor of each Xform must specify what kind of tree patterns that a transformation rule can be fired on.

The above pattern states that the Xform for splitting aggregation is applied when the engine sees an expression that has the following properties:
Logical aggregate operators
Has a relational child
Has a scalar child for the project list (aggregate function such as sum(b) in our running example) -->

---

# What?

```SQL
...
->  Redistribute Motion 2:2  (slice1; segments: 2)  (cost=0.00..431.00 rows=1 width=12)
			Hash Key: b
			Rows out:  Avg 1.5 rows x 2 workers at destination.  Max 2 rows (seg0) with 1.105 ms to end, start offset by 15 ms.
			->  Result  (cost=0.00..431.00 rows=1 width=12)
						Rows out:  Avg 1.5 rows x 2 workers.  Max 2 rows (seg1) with 0.273 ms to first row, 0.276 ms to end, start offset by 16 ms.
						->  GroupAggregate  (cost=0.00..431.00 rows=1 width=12)
									Group By: b
									Rows out:  Avg 1.5 rows x 2 workers.  Max 2 rows (seg1) with 0.272 ms to first row, 0.275 ms to end, start offset by 16 ms.
									->  Sort  (cost=0.00..431.00 rows=1 width=8)
												Sort Key: b
												Rows out:  Avg 1.5 rows x 2 workers.  Max 2 rows (seg1) with 0.260 ms to first row, 0.262 ms to end, start offset by 16 ms.
												Executor memory:  58K bytes avg, 58K bytes max (seg0).
												Work_mem used:  58K bytes avg, 58K bytes max (seg0). Workfile: (0 spilling, 0 reused)
												->  Table Scan on foo2  (cost=0.00..431.00 rows=1 width=8)
															Rows out:  Avg 1.5 rows x 2 workers.  Max 2 rows (seg1) with 0.074 ms to first row, 0.075 ms to end, start offset by 16 ms.
```

---

# Pre-condition Check

```c++
// Compatibility function for splitting aggregates
virtual
BOOL FCompatible(CXform::EXformId exfid){
	return (CXform::ExfSplitGbAgg != exfid);}
```

---

# The Actual Transformation

```c++
void Transform
	(
	CXformContext *pxfctxt,
	CXformResult *pxfres,
	CExpression *pexpr
	) const;
```

<!-- Each Xform must  implement the Transform API that takes as input the input logical/physical expression CExpression *pexpr and the transformation context CXformContext *pxfctxt to generate one or more output expressions CXformResult *pxfres. In our running example we take as input the global aggregate and generate as output another expression tree that has a local and global aggregate operations. -->

---

# Register Transformation

```c++
void CXformFactory::Instantiate()
{
….
Add(GPOS_NEW(m_pmp) CXformSplitGbAgg(m_pmp));
….
}
```

---

# What can't it do?

---

# Right
![filter](/Users/pivotal/Dropbox/presentations/fosdem2016/findingBalance.gif)
# Balance

---

## Improved performance for short running queries

---

# Not yet feature complete

- External parameters
- Cubes
- Multiple grouping sets
- Inverse distribution functions
- Ordered aggregates
- Indexed expressions

---

# PostgreSQL
![](/Users/pivotal/Dropbox/presentations/fosdem2016/postgres.png)

---

# Four pieces of low hanging fruit

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

---

# Slides

### Slides: github.com/addisonhuddy/presentations
