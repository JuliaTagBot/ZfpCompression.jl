# ZfpCompression.jl

[![Build Status](https://travis-ci.com/milankl/ZfpCompression.jl.svg?branch=master)](https://travis-ci.com/milankl/ZfpCompression.jl)
[![Build Status](https://ci.appveyor.com/api/projects/status/github/milankl/ZfpCompression.jl?svg=true)](https://ci.appveyor.com/project/milankl/ZfpCompression-jl)

Julia bindings for the data compression library [zfp](https://github.com/LLNL/zfp),
written by P Lindstrom ([@lindstro](https://github.com/lindstro)).
From the [zfp documentation](https://zfp.readthedocs.io/en/release0.5.5/):

*zfp is an open source library for compressed numerical arrays that support high
throughput read and write random access. To achieve high compression ratios, zfp
generally uses lossy but optionally error-bounded compression. Bit-for-bit lossless
compression is also possible through one of zfp’s compression modes.*

*zfp works best for 2-4D arrays that exhibit spatial correlation, such as
continuous fields from physics simulations, images, regularly sampled terrain
surfaces, etc. Although zfp also provides a 1D array class that can be used for
1D signals such as audio, or even unstructured floating-point streams, the
compression scheme has not been well optimized for this use case, and rate and
quality may not be competitive with floating-point compressors designed
specifically for 1D streams.*

See the documentation, or [zfp's website](https://computing.llnl.gov/projects/floating-point-compression)
for more information.

Requires Julia v1.3 or newer

## Example

![OzoneCompression](figures/zfp_precision3d_o3_85.png?raw=true "OzoneZfpCompression")  
*Compression of ozone (O₃) from the [CAMS data set](https://ads.atmosphere.copernicus.eu/about-cams) with zfp at various levels of precision.
Compression factors are relative to 64-bit floats regarding including the vertical dimension, shown is only one vertical level.*

## Usage
### Lossless compression

1 to 4-D arrays of eltype `Int32,Int64,Float32,Float64` can be compressed calling
the `zfp_compress` function.

```julia
julia> using ZfpCompression

julia> A = rand(Float32,100,50);

julia> Ac = zfp_compress(A)
16952-element Array{UInt8,1}:
 0xfd
 0xe1
 0x80
 0x8d
    ⋮
```
which initializes the zfp compression, preallocates the bitstream used for
the compressed array and performs the compression. The compressed array is returned
as `Array{UInt8,1}`. By default, the compressed array includes a header with the required
information about the type, size and shape of the uncompressed array as well
as lossy compression parameters (see below). This header can be deactivated with
```julia
julia> Ac = zfp_compress(A,write_header=false)
```

A compressed array (with header) can be decompressed as

```julia
julia> Ad = zfp_decompress(Ac)
```

Alternatively, the decompression of header-less compressed arrays can be performed
into an existing array (with same type, size and dimensions as the uncompressed array)

```julia
julia> Ad = similar(A)
julia> zfp_decompress!(Ad,Ac)
```

In this lossless example the compression is reversible
```julia
julia> A == Ad
true
```

### Lossy compression

Lossy compression is achieved by specifying additional keyword arguments
for `zfp_compress`, which are `tol::Real`, `precision::Int`, and `rate::Real`.
If none are specified (as in the example above) the compression is lossless
(i.e. reversible). Lossy compression parameters are

- [`tol` defines the maximum absolute error that is tolerated.](https://zfp.readthedocs.io/en/release0.5.5/modes.html#fixed-accuracy-mode)
- [`precision` controls the precision, bounding a weak relative error](https://zfp.readthedocs.io/en/release0.5.5/modes.html#fixed-precision-mode), see this [FAQ](https://zfp.readthedocs.io/en/develop/faq.html#q-relerr)
- [`rate` fixes the bits used per value.](https://zfp.readthedocs.io/en/release0.5.5/modes.html#fixed-rate-mode)

Only **one** of `tol, precision` or `rate` should be specified. For further details
see the [zfp documentation](https://zfp.readthedocs.io/en/release0.5.5/modes.html#compression-modes).

If we can tolerate a maximum absolute error of 1e-3, we may do
```julia
julia> Ac = zfp_compress(A,tol=1e-3)
9048-element Array{UInt8,1}:
 0xff
 0x2c
 0x01
 0x1a
 0xf3
 0xbc
 0xea
 0xbb
 0xc6
 0xd4
    ⋮
```
which clearly reduces the size of the compressed array. In this case the maximum
absolute error is limited to about 3e-4.
```julia
julia> A2 = zfp_decompress(Ac)
julia> maximum(abs.(A2 - A))
0.00030493736f0
```

For header-less compression, it is **essential** to provide the same compression
parameters also for `zfp_decompress!`. Otherwise the decompressed array is flawed. E.g.
```julia
julia> A2 = similar(A)
julia> zfp_decompress!(A2,Ac,tol=1e-3)
```

## OpenMP multi-threading

You can use compress in parallel using the `nthreads` argument of `zfp_compress` to trigger multi-threading via OpenMP.
No parallel decompression is currently (zfp v0.5.5) provided in the underlying C library.
On linux, `zfp_jll` is automatically built with OpenMP enabled,
[on macOS this is not supported by default](https://zfp.readthedocs.io/en/release0.5.5/execution.html#using-openmp).

```julia
julia> zfp_compress(temp,nthreads=8)
```

Compressing a 590MB array `A` with `precision=10` is benchmarked (`@btime`) as

Number of threads |      1|       2|        4|        8|       16|      32|
| --------------- | -----:| -----: | ------: | ------: | -------:|-------:|
Time              | 2.45s | 1.46s  | 0.73s   | 0.38s   | 0.25s   | 0.20s  |
Speed-up          |     1x|  1.7x  | 3.4x    | 6.4x    | 9.8x    | 12.3x  |


## Installation

ZfpCompression.jl is registered in the Julia Registry, so simply do
```julia
julia>] add ZfpCompression
```
and the C library is installed and built automatically.
