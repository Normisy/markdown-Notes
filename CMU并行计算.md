# Chapter 2. 并行计算的软件实现

## 2.1 ISPC并行语言

ISPC是英特尔开发的一门并行语言，可以在其Github主页[ISPC](http://ispc.github.com/)上进行下载，它提供了一种称为**SPMD单程序多数据**的抽象
SPMD意味着你有一段相同的代码，被一些逻辑上的“线程”执行或不执行，我们只是为它们提供代码，这些独立的“线程”会单独地在不同的数据片段上执行它，这就是一种常见的编写并行代码的方式

以使用泰勒展开计算多个sin函数值的程序为例，我们希望同时计算多个：
```c?
--------------sinx.ispc  //文件后缀为ispc
export void sinx(
	uniform int N,
	uniform int terms,
	uniform float* x,
	uniform float* result)
{
	for (uniform int i = 0; i < N; I += programCount) {
		int idx = i + programIndex;
		float value = x[idx];
		float numer = x[idx] * x[idx] * x[idx];
		uniform int denom = 6;
		uniform int sign = -1;

		for (uniform int j = 1; j <= terms; j++) {
			value += sign * numer / denom;
			numer *= x[idx] * x[idx];
			denom *= (2*j + 2) * (2*j + 3);
			sign *= -1;
		}
		result[idx] = value;
	}
}
```
看上去使用ISPC编写的代码和c代码差不多，它们被保存在`.ispc`为扩展名的文件中，这样的文件可以