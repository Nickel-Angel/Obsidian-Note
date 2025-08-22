### SKI
`I x -> x`
`K x y -> x`
`S x y z -> (x z)(y z)`

Expressions like `K x` or `S x y` are partially applied functions
### Recursion
`(S I I)(S I I) -> (I(S I I))(I(S I I)) -> (S I I)(S I I)`

`x = S (K f)(S I I)`
`S I I x ->* x x -> S (K f)(S I I) x -> ((K f) x)((S I I) x) ->* f(x x)`
`x x ->* f(x x) ->* f(f(x x))`
#### Conditional (Boolean)
`T x y -> x`
`F x y -> y`
`T = K` `F = S K`
`S K x y -> (K y)(x y) -> y`

not B is `B F T`, reverse the operation of B
B1 or B2 is `B1 T B2`, B1 and B2 is `B1 B2 F`, this is because we can translate 'and' and 'or' into if statement.
If B then X else Y `B X Y`
swap:
`swap x y -> y x`
`swap = S (K (S I))(S (K K) I)`
this is so complex...so we can find a systematic way to write cominators
we want a combinator `A(E,x) x -> E`, and x is not in A(E, x) (we can also say abstract E with respect to x)
then we have (if all E below with respect to x):
`A(x,x) = I`, `A(E,x) = K E`, `A(E1 E2,x) = S A(E1,x) A(E2,x)`

now, we can write swap:
`swap x y = (swap x) y -> y x`
`swap x = A(y x, y) = S A(y, y) A(x, y) = S I (K x)`
`swap = A(S I (K x),x) = S A(S I,x) A(K x, x) = S (K (S I)) (S A(K,x) A(x, x))`
`= S (K (S I)) (S (K K) I)`  

we can define helper combinators c1 c2 to reduce the size of result combinators:
`c1 x y z -> x (y z)`, `c2 x y z = (x z) y`
then:
`A(E1 E2,x) = c1 E1 A(E2,x)` if x is not in E1;
`A(E1 E2,x) = c2 A(E1,x) E2` if x is not in E2;
`A(E1 E2,x) = S A(E1,x) A(E2,x)` otherwise;

now, we rewrite swap:
`swap x = A(y x, y) = c2 I x`
`swap = A(c2 I x, x) = c1 (c2 I) I`

pair:
`pair x y z -> z x y`
`pair x y = A(z x y,z) = c2 A(z x,z) y = c2 (c2 I x) y`
`pair x = A(c2 (c2 I x) y,y) = c1 (c2 (c2 I x)) I`
`pair = A(c1 (c2 (c2 I x)) I,x) = c1 c1 A((c2 (c2 I x)) I,x) = c1 c1 (c2 A(c2 (c2 I x),x) I) = c1 c1 (c2 (c1 c2 A(c2 I x, x)) I) = c1 c1 (c2 (c1 c2 (c1 (c2 I) I)) I)` 

natural numbers:
`n f x` is $f^n(x)$
`0 f x = x`, `0 = S K`
`succ n f x = f (n f x)`, `succ = S (S (K S) K)`

`one = succ 0`, the successor number of 0 is 1
`add x y = x succ y`, x plus y is the y-successor number of x
`mul x y = x (add y) 0`, add y x times

`m = S (c1 mul (c2 I first))(c2 I second)`, p.first multiply p.second
`i2 = c1 succ (c2 I second)`, increase p.second by 1
`fac' = S (c1 pair m) i2`, factorial tail call
`fac = (c2 (c2 I fac')(pair one one)) first`

`fac' p = pair (m p) (i2 p)`
`fac' = A(pair (m p) (i2 p),p) = c1 pair A((m p)(i2 p),p) = c1 pair (S A(m p,p)(i2 p,p)) = c1 pair (S (c1 m I)(c1 i2 I))`

`fac n = (n fac'(pair one one)) first`
`fac = A((n fac'(pair one one)) first,n) = c2 A(n fac'(pair one one)) first`
`= c2 (c2 I (fac'(pair one one))) first`
