

#嵌入式 Julia



我们已经知道 调用 [C 和 Fortran 代码](use-c-fortran.md)Julia 可以用简单有效的方式调用 C 函数。但是有很多情况下正好相反:需要从C 调用 Julia 函数。这可以把 Julia 代码整合到更大型的 C/C++ 项目中去， 而不需要重新把所有都用C/C++写一遍。 Julia提供了给C的API来实现这一点。正如大多数语言都有方法调用 C 函数一样， Julia的 API 也可以用于搭建和其他语言之间的桥梁。



高级嵌入
=====================


我们从一个简单的C程序入手，它初始化Julia并且调用一些Julia的代码::

```
  #include <julia.h>

  int main(int argc, char *argv[])
  {
      jl_init(NULL);
      JL_SET_STACK_BASE;

      jl_eval_string("print(sqrt(2.0))");

      return 0;
  }
```

编译这个程序你需要把 Julia的头文件包含在路径内并且链接函数库 ``libjulia``。 比方说 Julia安装在 ``$JULIA_DIR``， 就可以用gcc编译::


    gcc -o test -I$JULIA_DIR/include/julia -L$JULIA_DIR/usr/lib -ljulia test.c


或者可以看看 Julia 源码里 ``example/`` 下的 ``embedding.c``。


调用Julia函数之前要先初始化Julia， 可以用 ``jl_init`` 完成，这个函数的参数是Julia安装路径，类型是 ``const char*`` 。如果没有任何参数，Julia会自动寻找Julia的安装路径。 

The second statement initializes Julia's task scheduling system. This statement must appear in a function that will not return as long as calls into Julia will be made (``main`` works fine). Strictly speaking, this statement is optional, but operations that switch tasks will cause problems if it is omitted.

The third statement in the test program evaluates a Julia statement using a call to ``jl_eval_string``.


类型转换
========================

Real applications will not just need to execute expressions, but also return their values to the host program. ``jl_eval_string`` returns a ``jl_value_t*``, which is a pointer to a heap-allocated Julia object. Storing simple data types like ``Float64`` in this way is called ``boxing``, and extracting the stored primitive data is called ``unboxing``. Our improved sample program that calculates the square root of 2 in Julia and reads back the result in C looks as follows::

    jl_value_t *ret = jl_eval_string("sqrt(2.0)");

    if (jl_is_float64(ret)) {
        double ret_unboxed = jl_unbox_float64(ret);
        printf("sqrt(2.0) in C: %e \n", ret_unboxed);
    }

In order to check whether ``ret`` is of a specific Julia type, we can use the ``jl_is_...`` functions. By typing ``typeof(sqrt(2.0))`` into the Julia shell we can see that the return type is ``Float64`` (``double`` in C). To convert the boxed Julia value into a C double the ``jl_unbox_float64`` function is used in the above code snippet.

Corresponding ``jl_box_...`` functions are used to convert the other way::

    jl_value_t *a = jl_box_float64(3.0);
    jl_value_t *b = jl_box_float32(3.0f);
    jl_value_t *c = jl_box_int32(3);

As we will see next, boxing is required to call Julia functions with specific arguments.



调用 Julia 的函数
========================

While ``jl_eval_string`` allows C to obtain the result of a Julia expression, it does not allow passing arguments computed in C to Julia. For this you will need to invoke Julia functions directly, using ``jl_call``::

    jl_function_t *func = jl_get_function(jl_base_module, "sqrt");
    jl_value_t *argument = jl_box_float64(2.0);
    jl_value_t *ret = jl_call1(func, argument);

In the first step, a handle to the Julia function ``sqrt`` is retrieved by calling ``jl_get_function``. The first argument passed to ``jl_get_function`` is a pointer to the ``Base`` module in which ``sqrt`` is defined. Then, the double value is boxed using ``jl_box_float64``. Finally, in the last step, the function is called using ``jl_call1``. ``jl_call0``, ``jl_call2``, and ``jl_call3`` functions also exist, to conveniently handle different numbers of arguments. To pass more arguments, use ``jl_call``:

    jl_value_t *jl_call(jl_function_t *f, jl_value_t **args, int32_t nargs)

Its second argument ``args`` is an array of ``jl_value_t*`` arguments and ``nargs`` is the number of arguments.



内存管理
========================

As we have seen, Julia objects are represented in C as pointers. This raises the question of who is responsible for freeing these objects.

Typically, Julia objects are freed by a garbage collector (GC), but the GC does not automatically know that we are holding a reference to a Julia value from C. This means the GC can free objects out from under you, rendering pointers invalid.

The GC can only run when Julia objects are allocated. Calls like ``jl_box_float64`` perform allocation, and allocation might also happen at any point in running Julia code. However, it is generally safe to use pointers in between ``jl_...`` calls. But in order to make sure that values can survive ``jl_...`` calls, we have to tell Julia that we hold a reference to a Julia value. This can be done using the ``JL_GC_PUSH`` macros:

    jl_value_t *ret = jl_eval_string("sqrt(2.0)");
    JL_GC_PUSH1(&ret);
    // Do something with ret
    JL_GC_POP();

The ``JL_GC_POP`` call releases the references established by the previous ``JL_GC_PUSH``. Note that ``JL_GC_PUSH``  is working on the stack, so it must be exactly paired with a ``JL_GC_POP`` before the stack frame is destroyed.

Several Julia values can be pushed at once using the ``JL_GC_PUSH2`` , ``JL_GC_PUSH3`` , and ``JL_GC_PUSH4`` macros. To push an array of Julia values one can use the  ``JL_GC_PUSHARGS`` macro, which can be used as follows:

    jl_value_t **args;
    JL_GC_PUSHARGS(args, 2); // args can now hold 2 `jl_value_t*` objects
    args[0] = some_value;
    args[1] = some_other_value;
    // Do something with args (e.g. call jl_... functions)
    JL_GC_POP();



控制垃圾回收
---------------------------------------------------

There are some functions to control the GC. In normal use cases, these should not be necessary.

|``void jl_gc_collect()``  | Force a GC run|
| ------------- |:-------------| 
|``void jl_gc_disable()``  | Disable the GC|
|``void jl_gc_enable()`` |   Enable the GC|




处理数组
========================

Julia and C can share array data without copying. The next example will show how this works.

Julia arrays are represented in C by the datatype ``jl_array_t*``. Basically, ``jl_array_t`` is a struct that contains:

- Information about the datatype
- A pointer to the data block
- Information about the sizes of the array

To keep things simple, we start with a 1D array. Creating an array containing Float64 elements of length 10 is done by::

    jl_value_t* array_type = jl_apply_array_type(jl_float64_type, 1);
    jl_array_t* x          = jl_alloc_array_1d(array_type, 10);

Alternatively, if you have already allocated the array you can generate a thin wrapper around its data::

    double *existingArray = (double*)malloc(sizeof(double)*10);
    jl_array_t *x = jl_ptr_to_array_1d(array_type, existingArray, 10, 0);
    
The last argument is a boolean indicating whether Julia should take ownership of the data. If this argument is non-zero, the GC will call ``free`` on the data pointer when the array is no longer referenced.

In order to access the data of x, we can use ``jl_array_data``:

    double *xData = (double*)jl_array_data(x);
    
Now we can fill the array::

    for(size_t i=0; i<jl_array_len(x); i++)
        xData[i] = i;
      
Now let us call a Julia function that performs an in-place operation on ``x``::

    jl_function_t *func  = jl_get_function(jl_base_module, "reverse!");
    jl_call1(func, (jl_value_t*)x);

By printing the array, one can verify that the elements of ``x`` are now reversed.


访问返回的数组
---------------------------------

If a Julia function returns an array, the return value of ``jl_eval_string`` and ``jl_call`` can be cast to a ``jl_array_t*``::

    jl_function_t *func  = jl_get_function(jl_base_module, "reverse");
    jl_array_t *y = (jl_array_t*)jl_call1(func, (jl_value_t*)x);

Now the content of ``y`` can be accessed as before using ``jl_array_data``.
As always, be sure to keep a reference to the array while it is in use.



高维数组
---------------------------------

Julia's multidimensional arrays are stored in memory in column-major order. Here is some code that creates a 2D array and accesses its properties:

    // Create 2D array of float64 type
    jl_value_t *array_type = jl_apply_array_type(jl_float64_type, 2);
    jl_array_t *x  = jl_alloc_array_2d(array_type, 10, 5);

    // Get array pointer
    double *p = (double*)jl_array_data(x);
    // Get number of dimensions
    int ndims = jl_array_ndims(x);
    // Get the size of the i-th dim
    size_t size0 = jl_array_dim(x,0);
    size_t size1 = jl_array_dim(x,1);

    // Fill array with data
    for(size_t i=0; i<size1; i++)
        for(size_t j=0; j<size0; j++)
            p[j + size0*i] = i + j;

Notice that while Julia arrays use 1-based indexing, the C API uses 0-based indexing (for example in calling ``jl_array_dim``) in order to read as idiomatic C code.



异常
==========

Julia code can throw exceptions. For example, consider:

      jl_eval_string("this_function_does_not_exist()");

This call will appear to do nothing. However, it is possible to check whether an exception was thrown:

    if (jl_exception_occurred())
        printf("%s \n", jl_typeof_str(jl_exception_occurred()));

If you are using the Julia C API from a language that supports exceptions (e.g. Python, C#, C++), it makes sense to wrap each call into libjulia with a function that checks whether an exception was thrown, and then rethrows the exception in the host language.




抛出 Julia 异常
-------------------------

When writing Julia callable functions, it might be necessary to validate arguments and throw exceptions to indicate errors. A typical type check looks like:

    if (!jl_is_float64(val)) {
        jl_type_error(function_name, (jl_value_t*)jl_float64_type, val);
    }

General exceptions can be raised using the funtions:

    void jl_error(const char *str);
    void jl_errorf(const char *fmt, ...);

``jl_error`` takes a C string, and ``jl_errorf`` is called like ``printf``:

    jl_errorf("argument x = %d is too large", x);

where in this example ``x`` is assumed to be an integer.
