# 线性代数 


## 矩阵分解


[矩阵分解](http://zh.wikipedia.org/zh-cn/%E7%9F%A9%E9%98%B5%E5%88%86%E8%A7%A3)是将一个矩阵分解为数个矩阵的乘积，是线性代数中的一个核心概念。

下面的表格总结了在 Julia 中实现的几种矩阵分解方式。具体的函数可以参考标准库文档的 [Linear Algebra](http://julia-cn.readthedocs.org/zh_CN/latest/stdlib/linalg/#stdlib-linalg)章节。


|Cholesky	|Cholesky 分解|
|:----|:----|
|CholeskyPivoted|	主元 Cholesky 分解|
|LU |	LU 分解|
|LUTridiagonal |	LU factorization for Tridiagonal matrices|
|UmfpackLU	|LU factorization for sparse matrices (computed by UMFPack)|
|QR	|QR factorization|
|QRCompactWY	|Compact WY form of the QR factorization|
|QRPivoted	|主元 QR 分解|
|Hessenberg	|Hessenberg 分解|
|Eigen|	特征分解|
|SVD	|奇异值分解|
|GeneralizedSVD|	广义奇异值分解|


## 特殊矩阵


线性代数中经常碰到带有对称性结构的特殊矩阵，这些矩阵经常和矩阵分解联系到一起。 Julia 内置了非常丰富的特殊矩阵类型，可以快速地对特殊矩阵进行特定的操作.

下面的表格总结了 Julia 中特殊的矩阵类型，其中也包含了 LAPACK 中的一些已经优化过的运算。

|Hermitian	|埃尔米特矩阵|
|:-----|:-----|
|Triangular	|上/下 三角矩阵|
|Tridiagonal	|三对角矩阵|
|SymTridiagonal	|对称三对角矩|
|Bidiagonal	|上/下 双对角矩阵|
|Diagonal	|对角矩阵|
|UniformScaling	|缩放矩阵|


## 基本运算


|矩阵类型	|+|	-|	*|	\ |	其它已优化的函数|
|:--|:--|:---|:---|:--|:-|
|Hermitian	| |	 |	|XY|	inv, sqrtm, expm|
|Triangular	 	| 	||XY|	XY|	inv, det|
|SymTridiagonal|	X|	X|	XZ|	XY|	eigmax/min|
|Tridiagonal	|X	|X	|XZ	|XY|	 |
|Bidiagonal|	X|	X|	XZ|	XY	 |
|Diagnoal	|X	|X	|XY|	XY|	inv, det, logdet, /|
|UniformScaling|	X|	X|	XYZ|	XYZ|	/|


图例：
|X|已对矩阵-矩阵运算优化|
|:--|:--|
|Y	|已对矩阵-向量运算优化|
|Z|	已对矩阵-标量运算优化|

## 矩阵分解



|矩阵类型	|LAPACK|	eig	|eigvals|	eigvecs	|svd	|svdvals|
|:--|:--|:---|:---|:--|:-|
|Hermitian	|HE|	 	ABC	 	 	 |
|Triangular	|TR	 	 	 	 	 |
|SymTridiagonal	|ST|	A	|ABC|	AD	 	 |
|Tridiagonal	|GT	 	 	 	 	 |
|Bidiagonal	|BD|	||| 	 	 	A	|A|
|Diagonal	|DI|	| 	A	 	 	 |

图例：

|A|	已对寻找特征值和/或特征向量优化	|例如 eigvals(M)|
|:----|:---|:--|
|B|	已对寻找 ilth 到 ihth 特征值优化	|eigvals(M, il, ih)|
|C|	已对寻找在 [vl, vh] 之间的特征值优化	|eigvals(M, vl, vh)|
|D|	已对寻找特征值 x=[x1, x2,...] 所对应的特征向量优化	|eigvecs(M, x)|

缩放运算
--------
A ``UniformScaling`` operator represents a scalar times the identity operator, ``λ*I``. The identity operator ``I`` is defined as a constant and is an instance of ``UniformScaling``. The size of these operators are generic and match the other matrix in the binary operations ``+``,``-``,``*`` and ``\``. For ``A+I`` and ``A-I`` this means that ``A`` must be square. Multiplication with the identity operator ``I`` is a noop (except for checking that the scaling factor is one) and therefore almost without overhead. 