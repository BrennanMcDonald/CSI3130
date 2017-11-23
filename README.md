# Term Project | Symmetric Hash Join1
## 1 Objective
You are to implement a new symmetric hash join query operator to replace the current hash join implementation.
Adding the new operator will require modications to both the optimizer component and the
executor component of PostgreSQL .
## 2 Hash Join
In this section we introduce the regular hash join operator which is currently implemented in PostgreSQL
8.1.4. Hash join can be used when the query includes join predicates of the form T1:attr1 = T2:attr2, where
T1 and T2 are two relations to be joined and attr1 and attr2 are join attributes with the same data type.
One of the relations is designated as the inner relation while the other is designated as the outer relation.
We will assume that T1 is the inner relation and T2 is the outer relation.
The hash join has two consecutive phases: the building phase and the probing phase.
Building Phase: In this phase, the inner relation (T1) is scanned. The hash value of each tuple is computed
by applying a hash function f to the join attribute (attr1). The tuple is then inserted into a hash
table. No outputs are produced during this phase.
The probing phase: When the inner relation has been completely processed and the hash table has been
constructed, the outer relation T2 is scanned and its tuples are matched to those from the inner relation.
This matching is done on two steps. First, the outer tuple's join attribute, attr2, is hashed using the
same hash function f. The resulting hash value is used to probe the inner hash table. Next, if the
appropriate bucket of the inner hash table is not empty, the attr1 value of each inner tuple in the
bucket is compared to attr2 of the outer tuple. If the actual values match, then the corresponding
tuples are returned as one tuple in the join result. This comparison of the actual attribute values of the
two tuples is necessary because of the possibility of hash collisions (multiple attribute values having
the same hash value).
One property of this algorithm is that no join outputs are produced until all of the tuples of the inner
relation are completely processed and inserted into the hash table. This represents a bottleneck in the query
execution pipeline. Symmetric hash join, which you will implement in this assignment, avoids this problem.
One of the drawbacks of the hashing algorithm that was just described is that there must be sucient
memory available to hold the hash table for the inner relation. This can be a problem for large relations.
For this reason, PostgreSQL actually implements a more complex algorithm, known as Hybrid Hash Join,
which avoids this problem. The hybrid hash join algorithm divides the tuples of the two relations into
multiple partitions, or batches. For example, each partition can be associated with a specic range of hash
values. Only one partition of the inner relation's tuples is kept in main memory at any time. The remaining
batches reside on disk. When an outer tuple is found to belong in a partition other than the current one
(i.e. the memory-resident partition), it is written to a temporary le and its processing is postponed until
the corresponding partition from the inner relation is retrieved. Partitions are processed in sequence until
all partitions have been processed.
3 Symmetric Hash Join
Here we describe the symmetric hash join [1] that you are required to implement. The symmetric hash join
operator maintains two hash tables, one for each relation. Each hash table uses a dierent hash function. It
supports the traditional demand-pull pipeline interface. The symmetric hash join works as follows:
1Do not redistribute this project which is partof an archive that Ihab Ilyas gave to me for my personal use in my course.
 Read a tuple from the inner relation and insert it into the inner relation's hash table, using the inner
relation's hash function. Then, use the new tuple to probe the outer relation's hash table for matches.
To probe, use the outer relation's hash function.
 When probing with the inner tuple nds no more matches, read a tuple from the outer relation. Insert
it into the outer relation's hash table using the outer relation's hash function. Then, use the outer
tuple to probe the inner relation's hash table for matches, using the inner table's hash function.
These two steps are repeated until there are no more tuples to be read from either of the two input
relations. That is, the algorithm alternates between getting an inner tuple and getting an outer tuple until
one of the two input relations is exhausted, at which point the algorithm reads (and hashes and probes with)
the remaining tuples from the other relation.
Note that the symmetric hash join operator must be implemented to adhere to the demand-pull pipeline
operator interface. That is, on the rst pull call, the algorithm should run until it can produce one join
output tuple. On the next pull call, the algorithm should pick up where it left o, and should run until it
can produce the second join output tuple, and so on. This incremental behaviour requires that some state
be saved between successive calls to the operator. Essentially, this state records where the algorithm left o
after the previous demand-pull call so that it can pick up from there on the next call. Clearly, this state
should include an indication of whether the operator is currently working on an inner or an outer tuple,
the current tuple itself, the current \probe position" within the inner or outer hash table, and so on. The
symmetric hash join operator records this state into a structure called a state node.
As an example of join execution, consider a join with join predicate T1:attr1 = T2:attr2. The join operator
will incrementally load a hash table H1 for T1 by hashing attr1 using hash function f1, and another hash
table H2 for T2 by hashing attr2 using hash function f2. The symmetric hash join operator starts by getting
a tuple from T1, hashing its attr1 eld using f1, and inserting it into H1. Then, it probes H2 by applying f2
to attr1 of the current T1 tuple, returning any matched tuple pairs that it nds. Next, it gets a tuple from
T2, hashes it by applying f2 to attr2, and inserts it into H2. Then, it probes H1 by applying f1 to attr2 of
the current T2 tuple, and returns any matches. This continues until all tuples from T1 and T2 have been
consumed.
## 4 PostgreSQL Implementation of Hash Join
In this section, we present an introduction to two components of PostgreSQL that you will need to modify
in this assignment, namely the optimizer and the executor. Then, we describe the hash join algorithm that
is already implemented in PostgreSQL .
## 4.1 Optimizer
The optimizer uses the output of the query parser to generate an execution plan for the executor. During
the optimization process, PostgreSQL builds Path trees representing the dierent ways of executing a query.
It selects the cheapest Path that generates the desired outputs and converts it into a Plan to pass to the
executor. A Path (or Plan) is represented as a set of nodes, arranged in a tree structure with a top-level
node, and various sub-nodes as children. There is a one-to-one correspondence between the nodes in the
Path and Plan trees. Path nodes omit information that is not needed during planning, while Plan nodes
discard planning information that is not needed by executor.
The optimizer builds a RelOptInfo structure for each base relation used in the query. RelOptInfo records
information necessary for planning, such as the estimated number of tuples and their order. Base relations
(baserel) are either primitive tables, or subqueries that are planned via a separate recursive invocation of
the planner. A RelOptInfo is also built for each join relation (joinrel) that is considered during planning.
A joinrel is simply a combination of baserel's. There is only one join RelOptInfo for any given set of
baserels. For example, the join fA ./ B ./ Cg is represented by the same RelOptInfo whether it is built
by rst joining A and B and then adding C, or by rst joining B and C and then adding A. These dierent
means of building the joinrel are represented as dierent Paths. For each RelOptInfo we build a list of
Paths that represent plausible ways to implement the scan or join of that relation. Once we have considered
all of the plausible Paths for a relation, we select the cheapest one according to the planner's cost estimates.
The nal plan is derived from the cheapest Path for the RelOptInfo that includes all the base relations of
the query. A Path for a join relation is a tree structure, with the top Path node representing the join method.
It has left and right subpaths that represent the scan or join methods used for the two input relations.
The join tree is constructed using a dynamic programming algorithm. In the rst pass we consider
ways to create joinrels representing exactly two relations. The second pass considers ways to make
joinrels that represent exactly three relations. The next pass considers joins of four relations, and so
on. The last pass considers how to make the nal join relation that includes all of the relations involved in
the query. For more details about the construction of query Path and optimizer data structures, refer to
src/backend/optimizer/README.
## 4.2 Executor
The executor processes a tree of Plan nodes. The plan tree is essentially a demand-pull pipeline of tuple
processing operations. Each node, when called, will produce the next tuple in its output sequence, or NULL
if no more tuples are available. If the node is not a primitive relation-scanning node, it will have child node(s)
that it calls recursively to obtain input tuples. The plan tree delivered by the planner contains a tree of
Plan nodes (struct types derived from struct Plan). Each Plan node may have expression trees associated
with it to represent, e.g., qualication conditions.
During executor startup, PostgreSQL builds a parallel tree of identical structure containing executor
state nodes. Every plan and expression node type has a corresponding executor state node type. Each
node in the state tree has a pointer to its corresponding node in the plan tree, in addition to executor state
data that is needed to implement that node type. This arrangement allows the plan tree to be completely
read-only as far as the executor is concerned; all data that is modied during execution is in the state
tree. Read-only plan trees simplify plan caching and plan reuse. Altogether there are four classes of nodes
used in these trees: Plan nodes, their corresponding PlanState nodes, Expr nodes, and their corresponding
ExprState nodes.
There are two types of Plan node execution: single tuple retrieval and multi-tuple retrieval. These
are implemented using the functions ExecInitNode and MultiExecProcNode, respectively. In single tuple
retrieval, ExecInitNode is invoked each time a new tuple is needed. In multi-tuple retrieval, the function
MultiExecProcNode is invoked only once to obtain all of the tuples, which are returned in a form of a hash
table or a bitmap. For more details about executor structures, refer to src/backend/executor/README.
## 4.3 PostgreSQL Hash Join Operator
In PostgreSQL , hash join is implemented in the le nodeHashjoin.c and creation of a hash table is implemented
in the le nodeHash.c. A hash join node in the query plan has two subplans that represents the
outer and the inner relations to be joined. The inner subplan must be of type HashNode.
As was described in Section 2, PostgreSQL implements hybrid hash join so that it can deal with large
relations. Recall that hybrid hash join processes tuples in batches based on their hash values. To make the
implementation of the symmetric hash join algorithm simpler, you may ignore this additional complexity.
Specically, your symmetric hash join implementation may assume that both hash tables will t memory
without resorting to paritioning into multiple batches. However, you will need to nd out how to disable the
use of multiple batches in the current implementation of hashing.
