# API Examples - Reduce
## Reduce
+ Description:
Perform reduction operations on the highest dimension. For example, the input is x [N, H, W, C], the axis value is 0, and the reduction is out [1, H, W, C].

### ReduceOp Definition
```
template <typename T>
struct DivideFunctor {
  HOSTDEVICE explicit inline DivideFunctor(int n) : n_inv((T)(1.0 / n)) {}

  HOSTDEVICE inline T operator()(const T* x) const { return x[0] * n_inv; }

  HOSTDEVICE inline T operator()(const T& x) const { return x * n_inv; }

  private:
    T n_inv;
};

template <typename Tx, typename Ty = Tx>
struct CustomMean {
  using Transformer = kpds::DivideFunctor<Tx>;

  inline Ty initial() { return static_cast<Ty>(0.0f); }

  __device__ __forceinline__ Ty operator()(const Ty &a, const Ty &b) const {
     return b + a;
  }
};
```
### Kernel Description

Perform the reduction operation on the highest dimension, merge the dimensions that do not need to be reduced, and map blockIdx.x to the dimensions that do not need to be reduced to ensure maximum access and storage efficiency. There is no dependency on data between threads, only intra-thread protocol operations are required. When size < blockDim.x, IsBounary needs to be set to true, indicating that the memory access boundary judgment needs to be performed to avoid access to memory out of range.

### Code

```
template <typename Tx, typename Ty, typename MPType, typename ReduceOp, typename TransformOp, bool IsBoundary = false>
__device__ void HigherDimImp(const Tx* x, Ty* y, ReduceOp reducer,
                                     TransformOp transformer, MPType init,
                                     int reduce_num, int left_num,
                                     int block_size) {
  const int NY = 1;
  int idx = blockIdx.x * blockDim.x;
  int idy = blockIdx.y * block_size; // block_offset of rows
  Tx reduce_input[NY];
  MPType reduce_compute[NY];
  MPType result = init;
  int block_offset = idy * left_num + idx + blockIdx.z * reduce_num * left_num; // the offset of this block
  const Tx* input = x + block_offset;
  int store_offset = blockIdx.y * left_num + blockIdx.z * gridDim.y * left_num + idx;
  // how many columns left
  int size = left_num - idx;
  // how many rows have to be reduced
  int loop = reduce_num - idy;
  loop = loop > block_size ? block_size : loop;

  for (int loop_index = 0; loop_index < loop; loop_index += NY) {
    kps::ReadData<Tx, Tx, 1, NY, 1, IsBoundary>(&reduce_input[0], input + loop_index * left_num, size, NY, 1, left_num);
    kps::ElementwiseUnary<Tx, MPType, REDUCE_VEC_SIZE, 1, 1, TransformOp>(&reduce_compute[0], &reduce_input[0], transformer);
    kps::Reduce<MPType, NY, 1, 1, ReduceOp, kps::details::ReduceMode::kLocalMode>( &result, &reduce_compute[0], reducer, false);
  }

  Ty temp_data = static_cast<Ty>(result);
  kps::WriteData<Ty, 1, 1, 1, IsBoundary>(y + store_offset, &temp_data, size);
}

template <typename Tx, typename Ty, typename MPType, typename ReduceOp, typename TransformOp>
__global__ void ReduceHigherDimKernel(const Tx* x, Ty* y, ReduceOp reducer,
                                      TransformOp transformer, MPType init,
                                      int reduce_num, int left_num,
                                      int blocking_size) {
  // when reduce_dim.size() == 1 and reduce_dim[0] != x_dim.size() - 1, this function will be used
  // eg: x_dim = {nz, ny, nx}, nx != 1, axis can be 0 or 1
  //     if axis = 1 then grid.z = nz, grid.y = ny / block_size, grid.x = nx / 32
  //     else grid.z = 1, grid.y = ny / block_size, grid.x = nx /32

  int size = left_num - blockIdx.x * blockDim.x;
  if (size >= blockDim.x) {  // complete segment
    HigherDimImp<Tx, Ty, MPType, ReduceOp, TransformOp>(
        x, y, reducer, transformer, init, reduce_num, left_num, blocking_size);
  } else {
    HigherDimImp<Tx, Ty, MPType, ReduceOp, TransformOp, true>(
        x, y, reducer, transformer, init, reduce_num, left_num, blocking_size);
  }
}

```