```cc[]
template <typename ResScalar, typename LhsScalar,
          typename RhsScalar, typename StorageIndex,
          typename OutputMapper, typename LhsMapper,
          typename RhsMapper>
struct TensorContractionKernel {
  ...
  template <typename Device>
  EIGEN_DEVICE_FUNC static void deallocate(
      Device& d, BlockMemHandle handle) { ... }
  ...
};
```

NOTES:

* The symptoms of the problem are that tests crash in the `deallocate` function.
* None of the template arguments were `long`, or `long long`, or any other
  primitive integer type.

@@@

```cc[]
#define REGISTER_TENSOR_CONTRACTION_KERNEL_NO_FALLBACK(    \
    RES_SCALAR, LHS_SCALAR, RHS_SCALAR)                    \
template <typename StorageIndex, typename OutputMapper,    \
          typename LhsMapper, typename RhsMapper>          \   
struct TensorContractionKernel<RES_SCALAR, LHS_SCALAR,     \
                               RHS_SCALAR, StorageIndex,   \
                               OutputMapper, LhsMapper,    \
                               RhsMapper> {                \
  ...                                                      \
  template <typename Device>                               \
  EIGEN_DEVICE_FUNC void deallocate(                       \
      Device& d, BlockMemHandle handle) { ... }            \
  ...                                                      \
};
```

@@@

@@@ 
[07daafc](https://github.com/tensorflow/tensorflow/commit/07daafc869841646d154a79a5246f6f1ebc2c3ab)

@@@@@

### What went well

* Testing, reproducibility
* Many engineers were able to help.

### What went poorly

* This bug has been undetected for ????? years

NOTES:

* This bug was introduced in April of 2019, and fixed in July 2020.


@@@



