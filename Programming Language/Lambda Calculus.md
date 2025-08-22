Lambda calculus be like:
$$
e \rightarrow x \mid \lambda x.e \mid e e \mid (e)
$$
More precisely:
$$
\begin{aligned}
E & \rightarrow \lambda x.E \mid T \\
T & \rightarrow F F \\
F & \rightarrow x \mid (E)
\end{aligned}
$$
$e$ is a expression, $x$ is a variable, an abstraction $\lambda x.e$, or an application $e_1 e_2$ 
A function $\lambda x.e$ is a definition just like $f(x) \overset{\mathrm{def}}{=} e$, it has two features: anonymous, a value 
The body of lambda abstraction extends as far as right as possible.
$\lambda x.x \, \lambda y.y = \lambda x.(x \, \lambda y.y)$
$\lambda x.(\lambda y.\lambda z.y \, z) \, x \neq \lambda x.\lambda y.\lambda z.y \, z \, x$
Computation Rule:
beta reduction: $(\lambda x.e_1) e_2 \rightarrow e_1 [x := e_2]$
Parameter $x$ is replaced by the $e_2$ in the body of the function $e_1$.
So SKI be like:
$I: \lambda x.x$
$K: \lambda z.\lambda y.z$

$x [x := e] = e$
$y[x:=e] = y$
$(e_1 e_2)[x := e] = (e_1 [x:=e])(e_2 [x:=e])$
$(\lambda x.e_1) [x:=e] = \lambda x.e_1$
$(\lambda y.e_1) [x := e] = \lambda y.(e_1 [x := e])$ if $x \neq y$ and $y$ does not free appear in $e$.
**The last rule guarantee $(\lambda y.x)[x := y] \neq \lambda y.y$**

Free Variables
$$
\begin{align}
\operatorname{FV}(x) & = \{ x \} \\
\operatorname{FV}(e_{1} \, e_{2}) & = \operatorname{FV}(e_{1}) \cup \operatorname{FV}(e_{2}) \\
\operatorname{FV}(\lambda x.e) & = \operatorname{FV}(e) - \{ x \}
\end{align}
$$
We can rewrite the last rule as:
$(\lambda y.e_1) [x := e] = \lambda y.(e_1 [x := e]) \text{ if } x \ne y \text{ and } y \not\in \operatorname{FV}(e)$
Maybe we can rename bound variables to avoid name collisions:
$(\lambda y.e_{1})[x := e] = \lambda z.((e_{1}[y:=z])[x:=e]) \text{ if } x \ne y \text{ and } z \text{ is fresh name.}$
Notice that $\lambda y.e_{1}$ indicates that $y$ is in $e_{1}$. So we can do this: $e_{1}[y:=z]$
So:
$$
(\lambda y.x)[x:=y] = (\lambda z.x)[x:=y] = \lambda z.y
$$
Renaming of bound variables is called alpha conversion, and the third rule, eta conversion is:
$$
e = \lambda x.e \, x, x \not\in \operatorname{FV}(e) 
$$
#### Summary
Lambda calculus has three rules:
beta conversion: $(\lambda x.e_{1})e_{2} \rightarrow e_{1}[x:=e_{2}]$
alpha conversion: $\lambda x.e = \lambda z.e [x:=z] \text{ where } z \text{ is fresh.}$
eta conversion: $e = \lambda x.e \, x, x \not\in \operatorname{FV}(e)$
#### Recursion
$$
\begin{align}
& Y = \lambda f.(\lambda x.f(x \, x))(\lambda x.f(x \, x)) \\
& Y \, g \, a = \lambda f.(\lambda x.f(x \, x))(\lambda x.f(x \, x)) \, g \, a \rightarrow \\
& (\lambda x.g(x \, x))(\lambda x.g(x \, x)) \, a \rightarrow \\
& g((\lambda x.g(x \, x))(\lambda x.g(x \, x))) \, a \rightarrow \\
& g(g(\lambda x.g(x \, x))(\lambda x.g(x \, x))) \, a
\end{align}
$$
#### Booleans
$$
\begin{align}
T &= K = \lambda x.\lambda y.x \\
F &= \lambda x.\lambda y.y
\end{align}
$$
Note that $T, F$ have no free variables, so we can just reuse the SKI encoding of Boolean operations.
Let $B$ be a boolean, then:
$$
\begin{align}
\lnot B &= B \, F \, T \\
A \land B &= A \, B \, F \\
A \lor B &= A \, T \, B
\end{align}
$$
#### Pairs
$$
\begin{align}
& \operatorname{pair} \, x \, y \, z = z \, x \, y \\
& \operatorname{pair} = \lambda x.\lambda y. \lambda z.z \, x \,y \\
& \operatorname{first} \, x \, y = x \\
& \operatorname{first} = \lambda x.\lambda y.x \\
& \operatorname{second} \, x \, y = y \\
& \operatorname{second} = \lambda x.\lambda y.y \\ 
\end{align}
$$
#### Natural Numbers
$$
\begin{align}
n \, f \, x & = \overbrace{(f \circ f \circ \cdots \circ f)}^{n}(x) \\
0 \, f \, x & = x \\
0 & = \lambda f.\lambda x.x \\
\operatorname{succ} n \, f \, x & = f \, (n \, f \, x) \\
\operatorname{succ} & = \lambda n.\lambda f.\lambda x.f \, (n \, f \, x)
\end{align}
$$
#### Factorial
$$
\begin{align}
\operatorname{one} & = \operatorname{succ} 0 \\
\operatorname{add} & = \lambda m.\lambda n.m \operatorname{succ} n \\
\operatorname{mul} & = \lambda m.\lambda n. m \, (\operatorname{add} n) \, 0 \\
\operatorname{p} & = \lambda p. \operatorname{pair} \, (\operatorname{mul} \, (p \operatorname{first}) \, (p \operatorname{second})) \,(\operatorname{succ} \, (p \operatorname{second})) \\
\operatorname{fac} & = \lambda n.(n \, p \, (\operatorname{pair} \operatorname{one} \operatorname{one}) \operatorname{first})
\end{align}
$$
#### Algebraic Data Types
An algebraic data type is a data type that is a union of multiple cases.
`Type T =`
`constructor_1 Type_11 Type_12 ... Type_1n |`
`constructor_2 Type_21 Type_22 ... Type_2m | ...`
constructor is a function, and also is a tag.
each constructor is corresponding to a destructor.
here are some examples:
`Type Nat = succ Nat | 0`
`Type List = nil | cons Nat List`
`Type Tree = leaf Nat | branch Tree Tree`

TODO: ctor and dtor