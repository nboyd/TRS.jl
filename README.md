# TRS.jl: Solving the Trust Region Subproblem

This package solves the Trust-Region subproblem:
```
minimize    ½x'Px + q'x
subject to  ‖x‖ ≤ r
```
where `x` in the `n-`dimensional variable. This is a **matrix-free** method returning highly accurate solutions efficiently by solving a **single** eigenproblem. It accesses `P` *only* via matrix multiplications (i.e. via `mul!`), so it can take full advantage of `P`'s structure/sparsity.

Furthermore, the following extensions are supported:
* Ellipsoidal norms: `‖x‖ = sqrt(x'Cx)` for any positive definite `C`
* Linear equality constraints: `Ax = b`
* Degenerate cases (i.e. the so-called `hard case`)
* Finding the local-no-global minimizer
* Solving the related constant-norm (`‖x‖ = r`) problem for all of the cases described above.

If you are interested for support of linear inequality constraints `Ax ≤ b` check [this](https://no-link-yet.com) package.

The main reference for this package is
```
Adachi, S., Iwata, S., Nakatsukasa, Y., & Takeda, A.
Solving the trust-region subproblem by a generalized eigenvalue problem.
SIAM Journal on Optimization 27.1 (2017): 269-291.
```
Additionally, the cases of local-no-global minimizers and linear equality constraints are covered in
```
Rontsis N., Goulart P.J., & Nakatsukasa, Y.
Solving the extended trust-region subproblem via an active set method.
Preprint in Arxiv.
```

## Documentation
### Standard TRS
The global solution of the standard TRS
```
minimize    ½x'Px + q'x
subject to  ‖x‖ = r,
```
where ‖·‖ is the 2-norm, can be obtained with:
```
trs(P, q, r; kwargs...) -> x, info
```
**Arguments** (`T` is any real numerical type):
* `P`: The quadratic cost represented as any linear operator implementing `mul!`, `issymmetric` and `size`.
* `q::AbstractVector{T}`: the linear cost.
* `r::T`: the radius.

**Output**
* `x::Vector{T}`: The global solution to the TRS
* `info::TRSInfo{T}`: Info structure. See below for details.

**Keywords (optional)**
* `tol`, `maxiter`, `ncv` and `v0` that are passed to `eigs` used to solve the underlying eigenproblem. Refer to `Arpack.jl`'s [documentation](https://julialinearalgebra.github.io/Arpack.jl/stable/) for these arguments. Of particular important is `tol::T` which essentially controls the accuracy of the returned solutions.
* `tol_hard::T=1e-4`: Threshold for switching to the hard-case. Refer to [Adachi et al.](https://epubs.siam.org/doi/pdf/10.1137/16M1058200), Section 4.2 for an explanation.
* `compute_local::Bool=False`: Whether the local-no-global solution should be calculated. More details below.

### Ellipsoidal Norms
Results for ellipsoidal norms `‖x‖ := sqrt(x'Cx)` can be obtained with
```
trs(P, q, r, C; kwargs...) -> x, info
```
which is the same as `trs(P, q, r)` except for the input argument
* `C::AbstractMatrix{T}`: a positive definite, symmetric, matrix that defines the ellipsoidal norm `‖x‖ := sqrt(x'Cx)`.

**N.B.**: `trs(P, q, r, C)` solves a considerably different eigenproblem (a generalized eigenvalue problem as compared to the unsymmetric standard eigenproblem of `trs(P, q, r)`). The user might prefer to perform a change of variables `y = cholesky(C)\x` and use `trs(P, q, r)` in cases where:
* Repeated problems need to be solved with the same `C`; and/or
* A high accuracy solution is not required.

### Equality constraints
The problem
```
minimize    ½x'Px + q'x
subject to  ‖x‖ ≤ r
            Ax = b
```
can be solved as
```
trs(P, q, r, F, b; kwargs...) -> x, info
```
which is the same as `trs(P, q, r)` except for the input arguments `b::AbstractVector{T}` and `F` which can either be
* `F::Factorization{T}`: a factorization of the matrix `[I A'; A 0]`; or
* `F::function`: a mutating function `F(y, z)` which writes into `y` the projection of `z` into the nullspace of `A`.

Solution of problems involving both equality constraints and ellipsoidal norms can be solved with
```
trs(P, q, r, C, F, b; kwargs...) -> x, info
```

### Finding local-no-global minimizers.
Due to non-convexity, TRS can exhibit at most one local minimizer with objective value less than the one of the global. This can be obtained via:
```
trs(···; compute_local=true, kwargs...) -> x1, x2, info
```
Similarly to the cases above, `x1::Vector{T}` is the global solution. Regarding the second output `x2::Vector{T}`:

**In normal cases:** (i.e. not in the hard-case -> "almost always")  
`x2` is the local-no-global optimizer. If the local-no-global solution does not exist, a zero-length vector is returned.

**In the hard case:**  
`x2` corresponds to a second global minimizer. If no second global optimizer were found, a zero-length vector is returned.

The user can detect the hard case via the returned symbol in `info.status`.


### Solving constant-norm problems
Simply use `trs_boundary` instead of `trs`.

### The `TRSInfo` struct
The returned info structure contains the following 
* `hard_case::Bool` Flag indicating if the problem was detected to be in the hard-case.
* `niter::Int`:  Number of iterations of the eigensolver
* `nmul::Int`:   Number of multiplications with `P` requested by the eigensolver.
* `λ::Vector{T}` Lagrange Multiplier(s) of the solution(s).