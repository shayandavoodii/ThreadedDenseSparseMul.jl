# ThreadedDenseSparseMul.jl

> Threaded implementation of dense-sparse matrix multiplication, built on top of `Polyester.jl`.

## Usage
Just install and import this package, and launch Julia with some threads `e.g. julia --threads=auto`! Then e.g. any of these will be accelerated:
```julia
A = rand(1_000, 2_000); B = sprand(2_000, 30_000, 0.05); buf = similar(size(A,1), size(B,2))  # prealloc
res = A*B
buf .= A * B
buf .+= 2 .* A * B
```

## Rationale
I want to do $C \leftarrow D*S$ fast, where $D$ and $S$ are dense and sparse matrices, respectively. However:
- The SparseArrays.jl package doesn't support threaded multiplication.
- The IntelMKL.jl package doesn't seem to support dense $\cdot$ sparsecsc multiplication, although one can get similar performance using that package and transposing appropriately. It also comes with possible licensing issues and is vendor-specifig.
- The ThreadedSparseCSR.jl package also just supports sparsecsr $\cdot$ dense.
- The ThreadedSparseArrays.jl package also just supports ThreadedSparseMatrixCSC $\cdot$ dense, and also doesn't install for me currently.

I haven't found an implementation for that, so made one myself. In fact, the package `Polyester.jl` makes this super easy, the entire code is basically
```julia
import SparseArrays: SparseMatrixCSC, mul!; import SparseArrays
import Polyester: @batch

function SparseArrays.mul!(C::AbstractMatrix, A::AbstractMatrix, B::SparseMatrixCSC, α::Number, β::Number)
    @batch for j in axes(B, 2)
        C[:, j] .*= β
        C[:, j] .+= A * (α.*B[:, j])
    end
    return C
end
```

Julia will automatically use this 5-parameter definition to generate `mul!(C, A, B)` and calls like `C .+= A*B` and so forth.

Notice that this approach doesn't make sense for matrix-vector multiplication (the loop would just have one element), so that case is not considered in this package.


### Note on column-major and CSC vs CSR
I haven't found much literature on the choice of CSC vs CSR storage specifically of the context of dense $\cdot$ sparse multiplication with column-major dense storage.
However, as we can see in the code snippet above, the CSC format seems to be reasonably sensible for column-major dense storage.
To compute any given column in $C_{(:,j)}$ of $C$ we are essentially computing a weighted sum of columns in $A$, i.e. $C_{(:,j)} = \sum_k \lambda_k \cdot A_{(:,k)}$ which should be very cache efficient and SSE-able.

## Benchmarking
For matrices $(N\times K)$ and $(K\times M)$ we fix $N=1'000$ and $K=2'000$ and vary N.
Here's are the benchmark results, comparing against SparseArrays.jl, which ships with Julia but is single-threaded:

![scaling benchmark](/benchmark/scaling.png)

For all N we see a speed up over `_spmul!` from the SparseArrays package of up to ~3x for N in [300, 30_000], and ~2x otherwise.
