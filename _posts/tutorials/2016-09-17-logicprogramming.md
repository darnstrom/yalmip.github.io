---
title: "Logics and integer-programming representations"
category: inside
subcategory: 1
permalink: "/tutorial/logicprogramming"
tags: [Logic programming, Big-M, Integer programming]
excerpt: "Logic programming in YALMIP means programming with operators such as alldifferent, number of non-zeros, implications and similiar combinatorial objects."
---

YALMIP does a lot of modelling for you behind the scenes, but sometimes it is important to know how models are created and what the standard building blocks and tricks are. Creating integer programming representable models seem like magic to some, but there are really only a few standard tricks used leading to a family of models. 

1. [Logical models involving binary variables](#logics)
1. [Logical models involving implications with binary variables and constraints](#constraints)
1. [Multiplications of variables and functions](#products)
1. [Representations of functions](#functions)
1. [Other](#other)

Some simple rules and strategies when deriving models for complex logic and combinatorials models:
1. Try to represent things with disjoint events of the kind "exactly one of these things occur" with a binary variable associated to each event.
2. Try to arrive at "binary variable implies set of constraints"
3. Introduce intermediate auxilliary variables to keep things clean
4. Decompose the logic using intermediate variables to connect parts

In other words, a large sparse simple model is much better than a compact complex model. What we try to emphasize in the examples is that **most models can be derived using exactly the same strategy and a few very basic building blocks**. Many of the models can be simplified and reduced, but those steps are typically done just as efficiently inside the solver during pre-solve.

Throughout the examples here, **a** and **b** represent scalar binary variables, **z** represents a vector of binary variables, and **x** is just some generic variable (could be anything).

In the models, we make use of [big-M models](/tutorial/bigmandconvexhulls) with associated constant \\(M\\). We use a notation with the same value everywhere, but in practice you would spend effort on deriving as small constants as possible for every single constraint.

In the models here, we explicitly derive detailed models for implications, but in practice, we would of course implement the low-level implications using [implies](/command/implies).

## Logical models involving binary variables 
{: #logics }

### s = NOT a

With binary \\(a = 1\\) representing true and \\(a = 0\\) representing false, logical negation turns into 

$$
s = 1-a
$$


### s = a AND b

\\(s\\) has to be \\(1\\) if both \\(a\\) and \\(b\\) are 1. \\(s\\) has to be \\(0\\) if  either of \\(a\\) and \\(b\\) are 0.

$$
s \geq a + b -1,~s \leq a,~s\leq b
$$

The idea generalizes to an arbitrary number of binary variables \\(s = z_1 ~\&~ z_2~ \&~ \ldots ~\&~ z_n\\) with

$$
s \geq \sum_{i=1}^n z_i - (n-1), ~s\leq z
$$

### s = a OR b

\\(s\\) has to be \\(1\\) if  either of \\(a\\) and \\(b\\) are 1, and \\(0\\) if none of them are 1.

$$
s \geq a,~s\geq b, ~ s \leq a + b 
$$


Also this generalizes trivially to arbitrary length

$$
s \geq z, ~s\leq \sum_{i = 1}^n z_j
$$

### s = a XOR b

\\(s\\) has to be \\(1\\) if  exactly one of \\(a\\) and \\(b\\) are 1, and \\(0\\) otherwise

$$
s \geq a - b, ~s \geq b-a, ~ s \leq a + b, ~ a + b\leq 2-s
$$

The idea generalizes to an arbitrary number of binary variables  with

$$
s \geq z_i - \sum_{j\ne i} z_j,~ s\leq \sum_{i = 1}^n z_j \leq n + (1-n)s
$$


### If a then b

Logical implication of binary variables


$$
b \geq a
$$


## Logical models involving binary variables and constraints
{: #constraints }

### If a then  \\(f(x)\leq 0\\)

To ensure a constraint holds when a binary is true, we model implication using a [big-M strategy](/tutorial/bigmandconvexhulls).

$$
f(x) \leq M(1-a)
$$

This is the model construct which is used for almost all cases below. Make sure you understand why this works. In YALMIP, this is the core of the [implies](/command/implies) operator.

### If logical(z) then  \\(f(x)= 0\\)

Introduce a new binary variable \\(s\\) to represent the logical condition using the methods above, and then use the standard implication.

$$
 f(x) \leq M(1-s)
$$

Note that you do not have to model the logical condition exactly (both directions), all you need is \\(\mathop{logical}(z) \rightarrow s\\).

This is a general strategy throughout. If you enter complex expressions, introduce new variables for simple sub-parts and build the complete model by combining simple standard models.

### If a then  \\(f(x) < 0\\)

Strict inequalities are impossible in practice (unless \\(f(x)\\) is quantized such as only involving integer variables). Hence, you have to use a margin and hope that this in combination with solver tolerances leads to a solution which actually satisfies the constraints (solutions claimed feasible and optimal can easily be infeasible and are only guaranteed to satisfy solver tolerance and termination critera)

$$
f(x) \leq -\epsilon  + M(1-a)
$$

### If a then  \\(f(x)\leq 0\\) else  \\(f(x)\geq 0\\)

The standard implication is simply repeated for the two cases.

$$
f(x) \leq M(1-a),-f(x)\leq Ma
$$

To create more easily generalizable models and learn a common core strategy, it is adviced to think of this as two disjoint cases each associated with a set of constraints. 

$$
\begin{align}
z_1 &\rightarrow \{f(x)\leq 0, ~ a=1\}\\
z_2 &\rightarrow \{-f(x)\leq 0, ~ a=0\}\\
z_1 + z_2 &= 1
\end{align}
$$

Once again ending up as standard models for implications

$$
\begin{align}
f(x)&\leq M(1-z_1), -(1-z_1)\leq a-1 \leq (1-z_1)\\
-f(x) &\leq M(1-z_2), -(1-z_2) \leq a \leq (1-z_2)\\
z_1 + z_2 &= 1
\end{align}
$$

### If a then  \\(f(x)\leq 0\\) else  \\(g(x)\leq 0\\)

We could of course create  a model using \\(a\\) only, but using the same strategy the whole time trains us to realize that all model look the same in principle

$$
\begin{align}
z_1 &\rightarrow \{f(x)\leq 0, ~ a=1\}\\
z_2 &\rightarrow \{g(x)\leq 0, ~ a=0\}\\
z_1 + z_2 &= 1
\end{align}
$$

Standard models for implications

$$
\begin{align}
f(x) &\leq M(1-z_1), -(1-z_1)\leq a-1 \leq (1-z_1)\\
g(x) &\leq M(1-z_2), -(1-z_2) \leq a  \leq (1-z_2)\\
z_1 + z_2 &= 1
\end{align}
$$

### If a then  \\(f(x)= 0\\)

This is nothing but  two inequalities in the implication, hence

$$
\begin{align}
f(x)&\leq M(1-a)\\
-f(x) &\leq M(1-a)\\
\end{align}
$$

### If \\( f(x) \leq 0\\) then a

When \\(f(x)\\) becomes negative, the binary variable should be forced to be activated. 

$$
f(x)\geq -Ma
$$

Note that we use a non-strict inequality. If behaviour around 0 is important, a margin will have to be used as discussed before, ant is up to you and your goal to decide if the margin is positive or negative,i.e. towards whih side you prefer incorrect activations to happen.


### If \\( f(x) = 0\\) then a

First, this is extremely ill-posed in practice as solvers work with floating-point numbers so it might consider, e.g., \\(10^{-7}\\) to be 0. It is also often ill-posed from a practical point of view. If your model says **if waterlevel is 1** is it really absolutely crucial that it behaves differently compared to the case when the waterlevel is \\(1+10^{-11}\\), i.e. the thickness of one atom? If yes,  how do you plan to use that solution in practice?

The disjoint logical model is

$$
\begin{align}
f(x)&<0 \rightarrow a = 0\\
f(x)&=0 \rightarrow a = 1\\
f(x)& >0 \rightarrow a = 0
\end{align}
$$

This is interpreted as 

$$
\begin{align}
z_1 &\rightarrow \{f(x)<0, a=0\} \\
z_2 &\rightarrow \{f(x)=0, a=1\} \\
z_3 &\rightarrow \{f(x)>0,a=0\} \\
z_1+z_2+z_3 &= 1
\end{align}
$$

A big-M representation of the implications, using a margin \\(\epsilon\\) around 0 if wanted leads to

$$
\begin{align}
-(1-z_1) &\leq a \leq (1-z_1), f(x) \leq -\epsilon + M(1-z_1)\\
-(1-z_2) &\leq a-1 \leq (1-z_2), -M(1-z_2)-\epsilon \leq f(x) \leq \epsilon + M(1-z_2)\\
-(1-z_3) &\leq a \leq (1-z_3), f(x) \geq \epsilon -M(1-z_3)\\
z_1+z_2+z_3 &= 1
\end{align}
$$



### If \\( f(x) \leq 0\\) then a else not a

Now it starts paying of thinking in terms of auxilliary variables and disjoint cases. This is 

$$
\begin{align}
z_1 &\rightarrow \{f(x)\leq 0, ~ a=1\}\\
z_2 &\rightarrow \{f(x)\geq 0, ~ a=0\}\\
z_1 + z_2 &= 1
\end{align}
$$

A big-M model is thus

$$
\begin{align}
f(x) &\leq M(1-z_1), -(1-z_1)\leq a-1 \leq (1-z_1)\\
f(x) &\geq M(1-z_2), -(1-z_2) \leq a  \leq (1-z_2)\\
z_1 + z_2 &= 1
\end{align}
$$


### If \\( f(x) \leq 0\\) then  \\( g(x) \leq 0\\)

Glue the two conditions using an intermediate binary variable

$$
\begin{align}
f(x) & \leq 0 \rightarrow z_1\\
z_1  & \rightarrow g(x) \leq 0
\end{align}
$$

These are standard models and we thus have

$$
\begin{align}
f(x)&\geq -Mz_1\\
g(x)&\leq M(1-z_1)
\end{align}
$$

### If (\\( f(x) \leq 0\\) and \\( h(x) \leq 0\\) ) then  \\( g(x) \leq 0\\)

Not much variation and the pattern should be clear. Glue conditions using intermediate binary variables

$$
\begin{align}
f(x) & \leq 0 \rightarrow z_1\\
h(x) & \leq 0 \rightarrow z_2\\
z_1 & \& z_2 \rightarrow z_3\\
z_3 & \rightarrow g(x) \leq 0
\end{align}
$$

These are standard models and we thus have

$$
\begin{align}
f(x) & \geq -Mz_1\\
h(x) & \geq -Mz_2\\
g(x) & \leq M(1-z_3)\\
z_3  & \geq z_1+z_2 - 1
\end{align}
$$

## Multiplications of variables and functions
{: #products }

### y = ab, a and b binary

Binary multiplication is nothing but logical and, hence

$$
y \leq a,~ y\leq b,~ y\geq a+b-1
$$

Generalization to more terms follows the same generalization as logical and.

### y = ax, a binary x, x non-binary

Multiplication of binary and non-binary should be seen as a logical operation

$$
\begin{align}
a  &\rightarrow y = x\\
1-a &\rightarrow y = 0\\
\end{align}
$$

This is implemented using our standard model for implications

$$
\begin{align}
-M(1-a) &\leq y-x \leq M(1-a)\\
-Ma &\leq y \leq Ma
\end{align}
$$

### y = abx, a and b binary, x non-binary

When there are repeated binary variables, the procedure cab simply be repeated with intermediate variables. Start by introducing a new variable and  model for **c = ab**  and then use model for **y = cx**. 

The idea and model generalizes to arbitrary polynomials of binary variables, simply introduce intermediate variables to keep it simple.

### y = ax, a binary, x integer

An alternative for multiplying a binary with an integer variable is to make a binary expansion of the integer \\(x = z_1 + 2 z_2+ 4z_3 + \ldots +2^{n+1}z_n\\) with binary variables \\(z\\), model products with new variables \\(y_i = az_i\\) using standard model for binary times binary, and then use \\(y = y_1 + 2y_2 + 4y_3 \ldots\\).

### y = wx, w integer, x continuous 

Make binary expansion of **w** and then create models for binary times continuous.

### y = wx, w integer,  x integer

Make binary expansions of both **w** and **x** and then create models for binary products.

## Representations of functions
{: #functions }

### y = abs(x)

A logical representation of absolute value is 

$$
\begin{align}
x &\geq 0 \rightarrow y = x\\
x &\leq 0 \rightarrow y = -x
\end{align}
$$

A disjoint representation of this using binary variables

$$
\begin{align}
z_1 &\rightarrow \{x\geq 0,~y = x\}\\
z_2 &\rightarrow \{x\leq 0,~y = -x\}\\
z_1 + z_2 &= 1
\end{align}
$$

Once again standard implications so the result is a standard big-M model

$$
\begin{align}
-M(1-z_1) &\leq y - x\leq M(1-z_1), x\geq -M(1-z_1)\\
-M(1-z_2) &\leq y + x\leq M(1-z_2), x\leq M(1-z_2)\\
z_1 + z_2 &= 1
\end{align}
$$

Remember that absolute value is convex, so you only use a MILP representation if absoutely needed.

### y = max(x)

The maximum of a vector can also be thought of as a logical model 

$$
\begin{align}
x_1 &\geq x \rightarrow y = x_1\\
\vdots&\\
x_n &\geq x \rightarrow y = x_n
\end{align}
$$

Introduce a binary variable for every case and derive disjoint representation

$$
\begin{align}
z_1 &\rightarrow y = x_1, x_1 \geq x\\
\vdots\\
z_n &\rightarrow y = x_n, x_n \geq x\\
\sum_{i=1}^n z_i &= 1
\end{align}
$$

This is finalized with standard implication models

$$
\begin{align}
-M(1-z_1) &\leq y - x_1\leq M(1-z_1), ~x-x_1 \leq M(1-z_1)\\
\vdots\\
-M(1-z_n) &\leq y - x_1\leq M(1-z_n), ~x-x_n \leq M(1-z_n)\\
\sum_{i=1}^n z_1 &= 1
\end{align}
$$

Remember that max is convex, so you only use a MILP representation if absolutely needed.

### y = min(x)

Use \\( \min(x) = -\max(-x)\\).

### y = sort(x)

Let \\(P\\) be a binary permutation matrix (rows and columns sum to 1), and write \\(y\\) as permutation of \\(x\\), \\(y = Px\\). For \\(y\\) to be sorted (increasing), we must have \\(y_i \leq y_{i+1}\\). Multiplications \\(y = Px\\) are implemented using the models above or more directly as, e.g., \\(-M(1-P_{ij}) \leq y_i - x_j \leq M(1-P_{ij})\\).

### y = floor(x)

In theory introduce integer \\(y\\) and \\(x-1 < y \leq x\\). In practice strict inequality has to be approximated leading to \\(x-1+\epsilon \leq y \leq x\\)

### y = ceil(x)

In theory introduce integer \\(y\\) and \\(x \leq y < x+1\\). In practice strict inequality has to be approximated leading to \\(x \leq y \leq x+1-\epsilon\\)

### y = fix(x)

Round towards zero means we we want floor for positive arguements and ceil for negative arguments. We can write this as 

$$
\begin{align}
x & \leq 0 \rightarrow y = \mathop{ceil}(x)\\
x & \geq 0 \rightarrow s =  \mathop{floor}(x)\\
\end{align}
$$

Logical model

$$
\begin{align}
z_1 &\rightarrow \{x\leq 0,y = \mathop{ceil}(x)\} \\
z_2 &\rightarrow \{x\geq 0,y = \mathop{floor}(x)\}\\
z_1+z_2 &= 1
\end{align}
$$

Implement the implications with integer \\(y\\)  and a combined model for ceil and floor

$$
\begin{align}
x &\leq M(1-z_1)\\
x &\geq -M(1-z_2)\\
x - z_2 + z_2\epsilon &\leq y \leq x + z_1-z_1\epsilon\\
z_1+z_2 & = 1
\end{align}
$$


### y = rem(x,m)

\\( y = \mathop{rem}(x,m)\\) means \\( y = x - nm, n = \mathop{fix}(x/m)\\), meaning we have to implement \\(\mathop{fix}(x/m)\\) using the model above (we assume \\(m\\) is constant).

### y = mod(x,m)

\\( y = \mathop{mod}(x,m)\\) means \\( y = x - nm, n = \mathop{floor}(x/m)\\), meaning we have to implement \\(\mathop{floor}(x/m)\\) using the model above (we assume \\(m\\) is constant).

### y = sgn(x)

A logical model of \\(s = \mathop{sgn}(x)\\) is

$$
\begin{align}
x & < 0 \rightarrow s = -1\\
x & = 0 \rightarrow s = 0\\
x & > 0 \rightarrow s = 1\\
\end{align}
$$

This is interpreted as 

$$
\begin{align}
z_1 &\rightarrow \{x<0, s=-1\} \\
z_2 &\rightarrow \{x=0, s=0\} \\
z_3 &\rightarrow \{x>0,s=1\} \\
z_1+z_2+z_3 &= 1
\end{align}
$$

A big-M representation of the implications, using a margin \\(\epsilon\\) around 0 if wanted leads to

$$
\begin{align}
-M(1-z_1) &\leq s+1 \leq M(1-z_1), ~ x \leq -\epsilon + M(1-z_1)\\
-M(1-z_2) &\leq s \leq M(1-z_2), ~ -M(1-z_2)-\epsilon \leq x \leq \epsilon + M(1-z_2)\\
-M(1-z_3) &\leq s-1 \leq M(1-z_3), ~x \geq \epsilon -M(1-z_3)\\
z_1+z_2+z_3 &= 1
\end{align}
$$

### y = nnz(x)

Introduce additional binary vectors \\(v,u,z\\) and let \\(y = \sum_{i=1}^n v_i+u_i\\) and the logical model

$$
\begin{align}
v_i = 1 & \rightarrow  x_i \leq -\epsilon\\
z_i = 1 & \rightarrow  \epsilon \geq x_i \geq -\epsilon\\
u_i = 1 & \rightarrow x_i \geq \epsilon\\
u_i + z_i + v_i & = 1
\end{align}
$$

Once again standard implications

$$
\begin{align}
 \epsilon +x & \leq M(1-v_1)\\ 
 -\epsilon +x & \leq M(1-z_1),~-\epsilon -x & \leq M(1-z_1)\\
  \epsilon -x & \leq M(1-u_1)\\ 
u_i + z_i + v_i & = 1
\end{align}
$$

### y = f(x), x scalar integer

For an arbitrary function defined over a bounded integer set (here for simple notation assumed to be \\(1\\leq x \leq M\\), we simply see it as as the disjoint logic model

$$
\begin{align}
z_i = 1 & \rightarrow  \{x = i, y = f(i)\}\\
\sum_{i=1}^M z_i &= 1
\end{align}
$$

This is compactly written as 

$$
y = \sum_{i=1}^{M} z_if(i),~x = \sum_{i=1}^{M} i z_i, ~\sum_{i=1}^M z_i = 1
$$



### y = piecewise affine function

A typical piecewise affine model is represented as **if** \\(A_ix\leq b_i\\) then \\(y = c_i^Tx+d_i\\) where \\(i = 1,\ldots,N\\). From above, this is 

$$
\begin{align}
z_i = 1 & \rightarrow  \{A_i x \leq b_i, y = c^T_ix + d_i\}\\
\sum_{i=1}^N z_i &= 1
\end{align}
$$

Standard implication again. 

$$
\begin{align}
 A_ix - b_i &\leq M(1-z_i)\\ 
 -M(1-z_i) &\leq y-c_i^Tx-d_i \leq M(1-z_i)\\
 \sum_{i=1}^n z_i &= 1
\end{align}
$$

Note that [sos2](/command/sos2) constructions are more efficient though.

### y = piecewise quadratic function

Following the picewise affine model, it is trivial to extend to the quadratic case **if** \\(A_ix\leq b_i\\) then \\(y = \frac{1}{2}x^TQ_ix + c_i^Tx+d_i\\). However, one should be slightly more clever, as the standard approach would introduce equalities involving quadratic expressions, which is nonconvex. 

Define \\(N\\) vectors \\(p_i\\) with length equal to \\(x\\), and  define \\(y\\) as  \\( \sum_{i = 1}^N \frac{1}{2}p_i^TQ_ip_i + c_i^Tp_i+d_i z_i\\) where

$$
\begin{align}
z_i = 1 & \rightarrow  \{x = p_i, A_i x \leq b_i\}\\
\sum_{i=1}^n z_i &= 1
\end{align}
$$

Standard implication.

$$
\begin{align}
 A_ix - b_i &\leq M(1-z_i)\\ 
 -M(1-z_i) &\leq x-p_i \leq M(1-z_i)\\
 \sum_{i=1}^n z_i &= 1
\end{align}
$$

## Other modeling strategies
{: #other }

As you should have seen by know, the approach to model is almost always the same. Use the basic building blocks of implications, with auxilliary variables, and try to flatten the logic as much as possible

### Counters and loops

A very common mistake is to see code of the following pattern

````matlab
x = binvar(n,1)
y = 0
for i = 1:n
 if x(i) 
  y = y + z(i)
 end
end
````

or

````matlab
x = binvar(n,1)
y = 0
for i = 1:n
 if x(i) 
  Model = [Model, y == y + z(i)]
 end
end
````
Neither will work, as you cannot use if-operators, and the second cases simply says z=0.

To model the counter, or accumulated value, you introduce an auxiiliary variable for each case and model its logic.

$$
\begin{align}
x_i  &=1 \rightarrow q_i = z_i\\
x_i  &=0 \rightarrow q_i = 0\\
y = \sum q_i
\end{align}
$$

These are standard building-block implications, either manually modelled using big-M as above, or most easily implemented using the implies operator.
