---
title: "Better Exhaustiveness"
author: "Filippo Nardon, Giovanni Maria Zanchetta"
date: \today

geometry: margin=0.75in
toc: true
header-includes:
  - \usepackage{mathtools}
  - \usepackage{stmaryrd}
---

$$
\gdef \pp#1{\llbracket #1 \rrbracket}
\gdef \empty{\emptyset}
$$

### Gleam syntax (minimal)

$$
\begin{array}{rcl}
s & \Coloneqq & e \mid \textit{let } x = e \mid \textit{assert } e \\
e & \Coloneqq & C(e, \dots, e) \mid x \\
& \mid & \textit{case } e \; \{ C(v, \dots, v) \mapsto e, \dots, C(v, \dots, v) \mapsto e \} \\
& \mid & \{ s, \dots, s, e \} \\
v & \Coloneqq & C(v, \dots, v) \\
C & \Coloneqq & \{ \text{constructors that appear as literals} \}
\end{array}
$$

We assume there are no free variables, a case ruled out by Gleam's semantic analysis.

### Intraprocedural constructor analysis

For now we restrict the analysis to the intraprocedural case. In this case there is no need to compute the fixpoint of the constraints because there are no loops in the CFG of the body of a function. This is because Gleam does not allow for iteration. An advantage of this is that we obtain a valid state simply by performing a linear scan of the AST. Moreover, we can afford to use an infinite-height lattice as a state:

$\boxed{\textit{State}} = \textit{Var} \to \mathcal P(v)$

$\boxed{t : \textit{State} \times s \to \textit{State}}$

$$
\begin{aligned}
t(\sigma, e) & = \sigma \\
t(\sigma, \textit{let } x = e) & = \sigma[x \mapsto \textit{eval}(\sigma, e)] \\
t(\sigma, \textit{assert } e) & = \textit{assert}(\sigma, e) \\
\end{aligned}
$$

$\boxed{\textit{eval} : \textit{State} \times e \to \textit{Val}}$

$$
\begin{aligned}
\textit{eval}(\sigma, C(e_1, \dots, e_n)) & = \{ C(v_1, \dots, v_n) \mid v_i \in \textit{eval}(\sigma, e_i) \} \\
\textit{eval}(\sigma, x) & = \sigma(x) \\
\textit{eval}(\sigma, \textit{case } x \; \{ C_i(v_{ij}) \mapsto e_i \}) & = \bigcup_{i \in \{ 1, \dots, n \}} \begin{cases} \textit{eval}(\textit{assert}(\sigma, x == C_i(v_{ij})), e_i) & \text{if } C_i(v_{ij}) \in \textit{eval}(\sigma, x) \\ \empty & \text{otherwise} \end{cases} \\
\textit{eval}(\sigma, \textit{case } e \; \{ C_i(v_{ij}) \mapsto e_i \}) & = \bigcup_{i \in \{ 1, \dots, n \}} \begin{cases} \textit{eval}(\sigma, e_i) & \text{if } C_i(v_{ij}) \in \textit{eval}(\sigma, e) \\ \empty & \text{otherwise} \end{cases} \\
\textit{eval}(\sigma, \{ s_1, \dots, s_n, e \}) & = \textit{eval}(\_, e) \circ t(\_, s_n) \circ \dots \circ t(\_, s_2)(t(\sigma, s_1)) \\
\end{aligned}
$$

$\boxed{\textit{assert} : \textit{State} \times e \to \textit{State}}$

$$
\begin{aligned}
\textit{assert}(\sigma, x == C(e_1, \dots, e_n)) & = \sigma[x \mapsto \{ C(v_1, \dots, v_n) \mid v_i \in \textit{eval}(\sigma, e_i)\}]
\end{aligned}
$$
