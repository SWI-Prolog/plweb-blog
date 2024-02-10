# Tabled reasoning about the dynamic database

Traditionally, in Prolog we represent the static world and rules in
Prolog predicates and we use _terms_ such as lists to represent answers
or guide a search process. A good example is solving [Einstein's
riddle](https://swish.swi-prolog.org/example/houses_puzzle.pl). Another
example is whether or not two nodes in a (possibly) cyclic graph are
connected using the code below.  Here the _Visited_ list ensures we
avoid non-termination due to cycles.

```
connected(X, Z) :-
    connected(X, Z, []).

connected(X, X, _).
connected(X, Z, Visited) :-
    link(X, Y),
    \+ memberchk(Y, Visited),
    connected(Y, Z, [Y|Visited]).
```

The above is a rather procedural way to express the connected closure of
a graph. A more natural way to express connectedness is using
[tabling](https://www.swi-prolog.org/pldoc/man?section=tabling), which
leads to the simple program below. This program is clean and
declarative. The price of the above is that it _materializes_ the
connected closure in a table.


```
:- table connected/2.

connected(X, X).
connected(X, Z) :-
    connected(X, Y),
    link(Y, Z).
```

Tabling turns Prolog into a _deductive database_. SWI-Prolog's tabling
has been developed in close cooperation with
[XSB](https://xsb.com/xsb-prolog). Tabling provides

  - Guaranteed termination of any Prolog program manipulating a
    finite set of Herbrand terms.  Notably, termination is not
    hindered by goals that call a _variant_ of itself.
  - Reduced impact of (sub-)goal ordering on performance due to
    _memoizing_.
  - _Well Founded Semantics_ to express _undefined_ thruth due
    to cycles through negation.  See e.g.,
    [Russels Paradox](https://swish.swi-prolog.org/p/russels%20paradox.swinb)


## Making the world dynamic

The advantage of classical Prolog backward reasoning is that derived
facts nor intermediate results are stored and changes to the _world_
(set of dynamic predicates) thus require no special attention. When
using tabling this changes as results are stored in tables. This problem
has been solved in XSB and SWI-Prolog using _incremental tabling_.
Establishing the initial tables using incremental tabling is no
different from normal tabling, but in addition to the tables an
__Incremetal Dependency Graph__ (IDG) is created. This graph keeps track
of which tables depend on each other and which tables depend on dynamic
predicates. Database changes (assert/1, retract/1, etc.) _invalidate_
dependent tables. Each table stores the _falsecount_ which represents
the number of invalid dependencies. Invalidat tables (with a positive
_falsecount_) are _re-evaluated_ when queried, i.e., using _lazy
re-evaluation_. If a table re-evaluates to the same content, i.e., the
database change has no impact on the outcome of the predicate, affected
tables have their _falsecount_ decremented and may thus become valid
without recomputing them.

Incremental tabling is activated with a modified directive as shown
below. Incremental tabling preserves all properties of normal tabling.
The overhead in terms of time and memory of maintaining the IDG is
typically affordable, but can become prohibitive in some scenarios.

    :- table connected/2 as incremental.
    :- dynamic link/2 as incremental.


### Monotonic tabling

Consider the connected/2 predicate above. Asserting a new link/2 fact
invalidates the connected/2 tables that may be affected by this new
fact. Querying connected/2 rebuilds the required tables.  This rather
silly as the connected/2 relation is _monotonic_, i.e., adding facts
for link/2 may only produce __new__ connected/2 relations and never
invalidates a derived connected/2 relation.

SWI-Prolog's _monotonic_ tabling deals with this. It does so by
annotating the edges of the IDG with
[continuations](https://www.swi-prolog.org/pldoc/man?section=delcont)
that relate the dynamic facts and intermediate tabled predicates with
the affected monotonic tables. Given this extended IDG we can propagate
new facts through the continuations and, if the continuation produces
one or more new answers, add these new answers to the affected table and
continue the propagation from there until fixed-point is reached.
Activating monotonic tabling is similar to incremental tabling:

    :- table connected/2 as monotonic.
    :- dynamic link/2 as monotonic.

There are two execution models for monotonic tabling: _eager_ (default)
and _lazy_. The eager model triggers the propagation process when axc
clause is added to a monotonic dynamic predicate. The _lazy_ model
invalidates the monotonic table in the same way as with incremental
tabling while recording the clauses with the dependencies. Re-evaluation
pushes these clauses through the continuations recorded with the
dependencies.

Currently, a _retract_ in a monotonic tabling setting invalidates the
affected tables in the same way as for incremental tabling and
subsequent re-evaluation is processed in the same way as for incremental
tabling, i.e., the table is rebuild.


## "What if" reasoning

Suppose we would like to do "what if" reasoning using connected/2, i.e.,
we would like to know the consequence of adding or removing one or more
link/2 facts.

