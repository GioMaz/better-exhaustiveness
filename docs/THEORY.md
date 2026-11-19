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
\text{Statements} & s \Coloneqq & e \mid \textit{let } x = e \mid \textit{assert } e \\
\text{Expressions} & e \Coloneqq & n \mid C(e, \dots, e) \mid x \mid (e, \dots, e) \mid e + e \\
& \mid & \textit{case } e \; \{ C \mapsto e, \dots, C \mapsto e \} \\
& \mid & \{ s, \dots, s, e \}
\end{array}
$$

### Intraprocedural constructor analysis

We start by defining a constructor analysis that only takes into consideration finite types, i.e. types defined without using primitive types

- $\boxed{\textit{Val}} = \mathcal P(\textit{Ctor})$ is an abstract value, where $\textit{Ctor}$ is the set of constructors that appear as literals in the program
- $\boxed{\textit{State}} = \textit{Var} \to \textit{Val}$

For now we restrict the analysis to the intraprocedural case. Because Gleam does not allow for iteration, there are no loops in the CFG of the body of a function. Therefore, we obtain a valid state simply by performing a linear scan of the AST.

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
\textit{eval}(\sigma, n) & = \emptyset \\
\textit{eval}(\sigma, C) & = \{ C \} \\
\textit{eval}(\sigma, x) & = \sigma(x) \\
\textit{eval}(\sigma, (e_1, \dots, e_n)) & = \prod_{i \in \{ 1, \dots, n \}} \textit{eval}(\sigma, e_i) \\
\textit{eval}(\sigma, e + e) & = \emptyset \\
\textit{eval}(\sigma, \textit{case } x \; \{ C_i(e_{ij}) \mapsto e_i \}) & = \bigcup_{i \in \{ 1, \dots, n \}} \textit{eval}(\textit{assert}(\sigma, x == C_i(e_{ij})), e_i) \\
\textit{eval}(\sigma, \textit{case } e \; \{ C_i(e_{ij}) \mapsto e_i \}) & = \bigcup_{i \in \{ 1, \dots, n \}} \textit{eval}(\sigma, e_i) \\
\textit{eval}(\sigma, \{ s_1, \dots, s_n, e \}) & = \textit{eval}(\_, e) \circ t(\_, s_n) \circ \dots \circ t(\_, s_2)(t(\sigma, s_1)) \\
\end{aligned}
$$

$\boxed{\textit{assert} : \textit{State} \times e \to \textit{State}}$

$$
\begin{aligned}
\textit{assert}(\sigma, x == C(e_1, \dots, e_n)) & = \sigma[x \mapsto \{ C(C_1, \dots, C_n) \mid C_i \in \textit{eval}(\sigma, e_i)\}]
\end{aligned}
$$
