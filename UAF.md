# UAF

## Mathematical Formulation:

- $G = (V, E)$ represents the code property graph, where $V$ is the set of vertices (nodes) and $E$ is the set of edges.
- Each edge $e \in E$ has a type $e.type$ and a context $e.context$.
- For a set of edges $S \subseteq E$, we define the following edge type filters:
  - $Load(S) = \{e \in S | e.type = "LOAD\_MEMORY"\}$
  - $Store(S) = \{e \in S | e.type = "STORE\_MEMORY"\}$
  - $Alias(S) = \{e \in S | e.type \in \{"MAY\_ALIAS", "CONTAINS"\}\}$
  - $BaseAlloc(S) = \{e \in S | e.type = "MAY\_ALIAS" \land \neg\exists e' \in E : e'.type = "SUBREGION" \land e'.target = e.source\}$
  - $FreedMem(S) = \{e \in S | e.type = "POINTS\_TO" \land e.target \notin Null\}$
  - $FreedPtr(S) = \{e \in S | e.source = c.argument0 \text{ for some } c \in CallToFree(E)\}$
  - $CallToFree(S) = \{e \in S | e.type = "CALL\_TO\_FUNCTION" \land e.target \in FreeFunctions\}$
  - $DataflowSig(S) = \{e \in S | e.type = "DATAFLOW\_SIGNATURE"\}$
  - $Dataflow(S) = \{e \in S | e.type = "DATAFLOW"\}$
  - $Allocates(S) = \{e \in S | e.type = "ALLOCATES"\}$

Then, we can define the sets used in the query:

1. Stop set:

$Stop = Load \circ Alias \circ BaseAlloc \circ FreedMem \circ FreedPtr \circ CallToFree(E)$
   $\cup Store \circ Alias \circ BaseAlloc \circ FreedMem \circ FreedPtr \circ CallToFree(E)$
   $\cup DataflowSig \circ Dataflow \circ Alias \circ BaseAlloc \circ FreedMem \circ FreedPtr \circ CallToFree(E)$

2. Start set:

$Start = \pi_{info,uuid,context}(\sigma_{(info,start\_uuid,start\_context) \in \pi_{info,start\_uuid,start\_context}(Stop)}(CallToFree(E)))$

3. AvoidAllocs set:

$AvoidAllocs = \pi_{info,allocation}(Stop \bowtie_{stop\_uuid=stop\_uuid \land stop\_context=stop\_context} Stop)$

4. Avoid set:

$Avoid = \pi_{info,start\_uuid,start\_context,avoid\_uuid,avoid\_context}(AvoidAllocs \bowtie Allocates(E))$

5. UseAfterFree set:

$UseAfterFree = \{(free, use, free\_contexts[(free, use)], use\_contexts[(free, use)]) | (free, use) \in dom(free\_contexts)\}$

where $free\_contexts$ and $use\_contexts$ are computed by:

$(free\_contexts, use\_contexts) = ControlFlowAvoiding(G, Start, Avoid, Stop, stack\_bound, limit)$

6. ControlFlowAvoiding function:

$ControlFlowAvoiding(G, S, A, T, stack\_bound, limit) =$
    $\{(u, v, p, q) | u \in S \land v \in T \land p \in Paths(G, u, v, A, [], stack\_bound, limit) \land q = p.context\}$

7. Paths function:

$Paths(G, u, v, A, visited, stack\_bound, limit) =$
    $\{[u] | u = v\}$
    $\cup \{u \cdot p | u \notin A \land u \notin visited \land |visited| < stack\_bound \land limit > 0$
              $\land p \in Paths(G, w, v, A, u \cdot visited, stack\_bound, limit - 1)$
              $\land (u, w) \in E\}$

where $\cdot$ represents list concatenation.

This recursive definition of $Paths$ can be read as follows:
- If ( $u$ ) is the same as $v$, then the only path is $[u]$.
- Else, if ( $u$ ) is not in the avoid set $A$, has not been visited before, the stack depth limit has not been reached, and the total path limit has not been reached, then for each neighbor $w$ of $u$, we recursively find paths from $w$ to $v$ (updating the visited set, stack depth, and path limit), and prepend $u$ to each of these paths.

The notation $\{x | P(x)\}$ represents a set comprehension -> "the set of all $x$ such that $P(x)$ is true".

## Summary

use-after-free query can be expressed in terms of:
1. Set operations on the graph $G$, using edge type filters and relational algebra operators like composition ($\circ$), projection ($\pi$), selection ($\sigma$), and join ($\bowtie$).

2. A recursive graph traversal $Paths$ that finds paths between start and end nodes, avoiding certain nodes and respecting stack depth and total path limits.

3. A top-level set comprehension $UseAfterFree$ that collects the results of the graph traversal into tuples of $(free \text{ node}, use \text{ node}, free \text{ contexts}, use \text{ contexts})$.
