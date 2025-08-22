### Order of Evaluation
normal order: always traverse the left most spine of the expression tree
if any reduction order terminates, normal order will terminate

Confluence
A set of rewrite rules is confluent if for any expression $E_0$, if $E_0 \overset{*}{\rightarrow} E_1$ and $E_0 \overset{*}{\rightarrow} E_2$ , there exists $E_3$ such that $E_1 \overset{*}{\rightarrow} E_3$ and $E_2 \overset{*}{\rightarrow} E_3$.
one step diamond property
If $A \rightarrow B$ and $A \rightarrow C$, there exists $D$ such that $B \rightarrow D$ and $C \rightarrow D$.

we can see SKI calculus doesn't have one step diamond property, because `S` copies its third argument. But we can define `X >> Y`:
`X -> Y` via a rewrite at the root node of the expression tree;
`X = A B`, `Y = A' B'` and `A >> A'`, `B >> B'`

we can see `A ->* B` if and only if `A >>* B`.
`>>` allows multiple rewrites as long as they in indenpendent subtrees, so now `S` has  one step diamond property.

Combinator calculus has the advantage of having no varibles.
`S` can pass information to the function we need, this function is like varibles.