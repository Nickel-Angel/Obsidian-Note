type is a set values
`List(Int)` is the set of all lists of integers
`Int -> Int` is the set of functions mapping an integer to an integer

Every concrete value is an element of some type or types 
Every legal program has a type

#### Rules of Inference
If ... then ... : $\Rightarrow$
and : $\land$
$x$ has type $T$ : $x:T$

If $e_{1}$ and $e_{2}$ has type Int, then $e_{1} + e_{2}$ has type Int.
$(e_{1}: \mathrm{Int} \land e_{2}: \mathrm{Int}) \Rightarrow e_{1} + e_{2}: \mathrm{Int}$

$$\frac{\mathrm{Hypothesis}_{1} \cdots \mathrm{Hypothesis}_{n}}{\mathrm{Conclusion}}$$
Hypothesis and conclusion is like: $\vdash e : T$
$\vdash$ means “it is provable that...”
#### Environment
An environment gives types for free variables
An environment is a function from variables to types
Let $A$ be a environment:
$$
A \vdash e : T
$$
So add rules:
$$
\frac{\begin{array}{c}
A \vdash e_{1}:\mathrm{Int} \\
A \vdash e_{2}:\mathrm{Int}
\end{array}}{A \vdash e_{1} + e_{2}:\mathrm{Int}}
$$
Variable rules:
$$
\frac{A(x) = T}{A \vdash x:T}
$$
#### A Language of Typed Functions

Untyped lambda calculus: 
$e \rightarrow x \mid \lambda x.e \mid e e$
Simply typed lambda calculus: 
$e \rightarrow x \mid \lambda x: t.e \mid e e \mid i$
$t \rightarrow \alpha \mid t \rightarrow t \mid \mathrm{int}$

Var:
$$
\frac{}{A, x : t \vdash x : t}
$$
Int:
$$
\frac{}{A \vdash i : \mathrm{Int}}
$$
Abs: (how to show a function type)
$$
\frac{A, x : t \vdash e : t'}{A \vdash \lambda x : t.e : t \rightarrow t'}
$$
App: (how to show the type of return value of a function)
$$
\frac{
\begin{array}{c}
A \vdash e_{1} : t \rightarrow t' \\
A \vdash e_{2} : t
\end{array}}{A \vdash e_{1} e_{2} : t'}
$$
#### Type Inference
For function applications:
• The expression in function position must have a function type
• The function domain and the function argument must have the same type
Two steps to constructing a valid typing (or showing none exists) 
• Solve the equations
• Substitute the solution back into the type derivation to obtain a valid proof
When no new constraints can be added, the constraints are saturated.

Infinite Solutions: if $x = A \rightarrow B$，and $x$ occurs in $A$ or $B$.
$x = \mathrm{int} \rightarrow x$
$x = x \rightarrow \mathrm{int}$

Canonicalization
Given a saturated set of equations $S$ and a type $t$, the canonicalization algorithm $C(\varnothing, S,t)$ produces a canonical type for $t$ that does not depend on $S$
$$
\begin{align}
& C(X, S, \mathrm{int}) = \mathrm{int} \\
& C(X, S, t \rightarrow t') = C(X \cup \{ t \rightarrow t' \}, S, t) \rightarrow C(X \cup \{ t \rightarrow t' \}, S, t') \text{ if } t \rightarrow t' \not\in X \\
& C(X, S, \alpha) = C(X \cup \{ \alpha \}, S, t) \text{ if } \alpha \in S \text{ and } t \text{ is not a type variable}, \alpha \not\in X \\
& C(X, S, \alpha) = C(X \cup \{ \alpha \}, S, \beta) \text{ if } \alpha = \beta \in S \text{ and } \alpha < \beta, \alpha \not\in X \\
& C(X, S, \alpha) = \alpha \text{ otherwise}
\end{align}
$$
for all equation $A = B \in S$, check
$A' = C(\varnothing, S, A), B' = C(\varnothing, S, B)$ is defined.
$A' = B'$ is not between a function type and int.