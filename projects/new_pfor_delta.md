## new-pfor-delta算法

> 背景： 在倒排索引中，不同文档包含了同一个分词时，那么这些文档的DOC ID（每一篇文档唯一赋值的标识），都会被加入到这个分词所对应的倒排链中，如文档1包含了分词{A, B, C}, 文档2包含了分词{C}, 文档3包含了分词{B,C}, 那么建立的倒排索引则是

```json
{
    "A":[1],
    "B":[1,3],
    "C":[1,2,3]
}
```

> 如果我们再进行差分相减，则该倒排链变为了：

```json
{
    "A":[1],
    "B":[1,2],
    "C":[1,1,1]
}
```

* 对倒排索引进行压缩以节省空间的一种算法， 算过程详见[链接1](https://www.cnblogs.com/bonelee/p/6882088.html)与[链接2](https://yq.aliyun.com/articles/563081)
* 算法思路： 我们假设一个数字数组中大部分的数字是很小的（因为在倒排索引中，文档的DOC ID的大小是按照其插入顺序递增的, 因此一个压缩单元内只需要存储其起始DOC ID，以及之后的差值）， 那么32bit的整数表示的实际值大部分情况高位都为0，相当于浪费了这部分存储空间。所以把小部分较大的值取出， 将剩下的值压缩在远小于32bit的二进制串中，能够把这些空闲的高位节省出来。

### 实现细节

* 设原数组为S， 编码后的字符串的二进制数组为T，framebit为x, f(a, b)为将数字a编码为二进制后，只保留低b位的二进制串， 则有

```code
T=Concat(f(S[i]&(2^x-1),x)) + OtherCompress(j+(S[j]>>x) | S[j]>=2^x)
```
> 将串S中小于2^x的数字取出(大于等于2^x的数字则取出低位)，然后把这些数字按照x bit进行压缩存储。 将那些大于等于2^x的数字与其在原数组中的位置一起(因为其位置下标必定递增，所以只需要存差值即可)，用别的压缩算法(例如S9算法)再次压缩。
* 令大于等于2^x的值为异常值，由于异常值除了存储高位信息外，还必须存储位置信息，如果两个异常值相隔很远的话，就必须要用一个较大的值来存储位置信息，为了尽可能保证OtherCompress的压缩效率，通常控制S压缩单元在一定的长度之内，如128个整数为一个压缩单元。 压缩单元过大还有一个问题就是会造成解压性能的下降。

### S9算法

* 令原数组为S，压缩后的数组为T。
* 对于T数组中一个32位的整数，用低28位来存储具体内容，高4位用来指明压缩方法。
  * 对于低28位的空间，划分成x块，每块y bit，如果S中连续x个值，每个值都小于2^y，那么可以把这x个数压缩在一个32位整数中存储
  * 由new-pfor-delta的实现可知，new-pfor-delta中选择的framebit不得低于a位(a为32-bits_of_number(max(S[i])), 即如果数组中最大的异常值为2^32-1,那么framebit不得低于4)，否则S9的28位数值空间可能无法存放。
  
### 如何选择framebit

* framebit的选取对于压缩效率来说很重要，如果framebit选取过大，则浪费了正常部分存储空间，如果framebit选取过小，则会增加异常值的数量。
* 具体实现的时候，我们可以设置异常值的比例不超过某一个阙值maxExceptionRatio，在这个前提下尽可能地选取压缩后空见更小的framebit。
  * 设原数组总长度为n，bits_of_number[i]为二进制最高位为i的数值的个数，sum_of_bits[i]为bits_number[i]的前缀和。当framebit为i时，异常值个数为`expcetionNum[i]=n-sum_of_bits[i]`。此时压缩后的空间总大小为n*framebit+len(OtherCompress(expcetionnum)), 对于异常值的压缩结果，这里只能给出估算值，开发者可以根据具体算法测试出平均压缩效果来，以便快速的估算。

### SIMD

> SIMD全称Single Instruction Multiple Data，单指令多数据流，通俗的说是指这样的指令集：收集多个操作数，将它们放入128位、256位甚至更多位的寄存器中（这个过程的访存操作也是并行的），同时（并行）对它们执行相同的操作。

* 由于索引的解压实际上是发生在查询过程中的，因此解压速度直接关系到查询性能。
> New PForDelta算法解压时分为正常部分和异常部分的解压。异常部分需要将异常数据的高位依据间隔补到正常部分解压出来的数组中，不能使用SIMD指令，而正常部分只是将一系列按相同的framebit位存储的数据顺序提取到数组中，完全可以使用SIMD指令，因此我们只针对正常部分进行优化，优化的指令集是128位的sse4指令集

* 在这里仅以framebit为4的情况进行举例说明，伪代码如下

```c++
// 注：实际工业运用中，考虑到cpu的流水线处理，通常会做不同程度的循环展开，这里为了节省代码行数，做了简化处理。
void pack(uint32_t* dest, const uint32_t* src, uint32_t n)
{
    for (uint32_t i = 0; i < n; i += 8) {
        for (uint32_t j = 0; j < 8; j ++) {
            dest[i >> 3] |= (src[i + j] & 15) << (j * 4);
        }
    }
}

void unpack(uint32_t* dest, const uint32_t* src, uint32_t n)
{
    for (uint32_t i = 0; i < n; i += 1) {
        for (uint32_t j = 0; j < 8; j ++) {
            dest[(i << 3) | j] = src[i] >> (j * 4) & 15;
        }
    }
}

```
* 在unpack这个函数中，假如我们把dest和src都换成以字节为单位的数组，那么则有以下伪代码：

```c++

void unpack(unsigned char* dest, const unsigned char* src, uint32_t n)
{
    for (uint32_t i = 0; i < n; i += 2) {
        // 这里假设char数组中，头部存储的uint32_t的低位。
        dest[i << 3] = src[i] & 15;
        dest[i << 3 | 4] = src[i] >> 4 & 15;
        dest[(i + 1) << 3] = src[i + 1] & 15;
        dest[(i + 1) << 3 | 4] = src[i + 1] >> 4 & 15;
    }
}
```

* 现在把`dest[i*8:i*8+16]`当作一个向量，利用CPU提供的SIMD指令来进行计算，代码如下。

```c++
void unpack_simd(uint32_t* dest, const unsigned char* src, uint32_t n)
{
    __m128i* vector_m128i = (__m128i*)src;
    for (uint32_t i = 0; i < n; i += 128) {
        __m128i data = _mm_loadu_si128(vector_m128i);
        // 为了加速CPU流水线计算，此处应展开循环，为省略代码故未展开
        for (uint32_t j = 0; j < 8; j ++) {
            smid_calculate(data, dest, j * 2);
        }
        vector_m128i ++;
    }
}

void smid_calculate(__m128i data, uint32_t*& result, unsigned int offset)
{
    __m128i shuffle = _mm_set_epi8(0xFF, 0xFF, 0xFF, offset+1, 0xFF, 0xFF, 0xFF, offset+1,
        0xFF, 0xFF, 0xFF, offset, 0xFF, 0xFF, 0xFF, offset);
    // 现构造序列A为{0, 0, 0, s[i+1],  0, 0, 0, s[i+1], 0, 0, 0, s[i], 0, 0, 0, s[i]};
    __m128i A = _mm_shuffle_epi8(data, shuffle);
    // 构造序列B为{0, 0, 0, 15<<4,  0, 0, 0, 15, 0, 0, 0, 15<<4, 0, 0, 0, 15};
    static __m128i B  = _mm_set_epi8 (0x00, 0x00, 0x00, 0xF0,
        0x00, 0x00, 0x00, 0x0F, 0x00, 0x00, 0x00, 0xF0, 0x00, 0x00, 0x00, 0x0F);
    // 构造乘法MASK序列C为{0, 0, 0, 1, 0, 0, 0, 16, 0, 0, 0, 1, 0, 0, 0, 16};
    static __m128i C  = _mm_set_epi32 (0x01, 0x10, 0x01, 0x10);
    
    // 可得A&B*C为{0, 0, 0, s[i+1]&(15<<4),  0, 0, 0, (s[i+1]&15)<<4, 0, 0, 0, s[i]&(15<<4), 0, 0, 0, (s[i]&15)<<4};
    __m128i result_mul = _mm_mullo_epi32(_mm_and_si128(A, B), C);
    
    // 整体右移4位,可得A&B*C>>4为{0, 0, 0, (s[i+1]>>4)&15,  0, 0, 0, s[i+1]&15, 0, 0, 0, (s[i]>>4)&15, 0, 0, 0, s[i]&15};
    __m128i result_s = _mm_srli_epi32(result_mul, 4);
    
    // 翻转上述得到的序列即为dest[i*8:i*8+16]
    _mm_storeu_si128((__m128i*)result, result_s);
    result = result + 4;
}

```

