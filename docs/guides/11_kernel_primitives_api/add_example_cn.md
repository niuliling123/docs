# API 应用实例 - Add
## Add
+ 案例功能说明：完成相同shape的两数相加，输入为InT类型，输出为OutT类型，根据Functor完成对应的计算.

### Functor 定义

```
AddFunctor:

template <typename InT, typename OutT>
struct AddFunctor {
  HOSTDEVICE OutT operator()(const InT &a, const InT &b) const { return statice<OutT>(a + b); }
};

```
### Kernel 实现说明

VecSize 表示每个线程连续读取VecSize个元素，根据剩余元素num与每个线程最大处理的元素个数VecSize x blockDim.x的关系，将数据处理分为2部分，第一部分，当VecSize * blockDim.x > num 表示当前数据处理需要进行边界处理，因此将IsBoundary设置为 true，避免访存越界，注意此处使用Init函数对寄存器arg0，arg1进行初始化，避免当arg0或者arg1作为分母时出现为0的情况。此处根据Functor完成两数求和操作，当需要进行两数相乘，可以直接修改对应的Functor即可，可以直接复用kernel 代码，提升开发效率。

### Kernel 代码

```
#include "kernel_primitives/kernel_primitives.h"
template<int VecSize, typename InT, typename OutT, typename Functor, bool IsBoundary>
__device__ void elementwiseImpl(InT *in0, InT * in1, OutT * out, Functor func, int num) {
  InT arg0[VecSize];
  InT arg1[VecSize];
  OutT result[VecSize];
  Init<InT, VecSize>(arg0, static_cast<OutT>(1.0f));
  Init<InT, VecSize>(arg1, static_cast<OutT>(1.0f));
  ReadData<InT, VecSize, 1, 1, IsBoundary>(arg0, in0, num);
  ReadData<InT, VecSize, 1, 1, IsBoundary>(arg1, in1, num);
  ElementwiseBinary<InT, OutT, VecSize, 1, 1, Functor>(result, arg0, arg1, func);
  WriteData<OutT, VecSize, 1, 1, IsBoundary>(out, result, num);
}

template<int VecSize, typename InT, typename OutT, typename Functor>
__global__ void elementwise(InT *in0, InT *in1, OutT *out, int size, Functor func) {
  int data_offset = VecSize * blockIdx.x * blockDim.x; // data offset of this block
  int stride = gridDim.x * blockDim.x * VecSize;
  for (int offset = data_offset; offset < size; offset += stride) {
    if (offset + blockDim.x * VecSize < size) {
      elementwiseImpl<VecSize, InT, OutT, Functor, false>(in0 + offset, in1 + offset, out + offset, func, size - offset);
    } else {
      elementwiseImpl<VecSize, InT, OutT, Functor, true>(in0 + offset, in1 + offset, out + offset, func, size - offset);
    }
  }
}

```