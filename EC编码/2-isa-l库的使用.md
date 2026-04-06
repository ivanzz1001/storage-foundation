# Erasure Code: isa-l库的使用

文章转载自[erasure code: isa-l库使用](https://blog.csdn.net/wsqyz/article/details/142322571), 并对其中的一些内容做了相应的修改。

参看：

- [isa-l库github](https://github.com/intel/isa-l.git)

- [纠删码(Erasure Code)的数学原理竟然这么简单](https://www.miaokee.com/2546675.html)

- [Erasure-Code(纠删码) 最佳实践](https://zhuanlan.zhihu.com/p/106096265)

- [纠删码(erasure code)介绍](https://zhuanlan.zhihu.com/p/554262696)


<br>

## 1. isa-l库的使用

最近在PureFlash存储项目上实现EC功能，用到了intel isa-l库的Erasure code。查看了`isa-l`库关于EC的使用用例，这里记录一下编码以及解码过程中函数的使用。

在EC的实现中，我们想要对一段数据进行编码，假设`chunk size`为128K， ec模式为`8+4`(k=8, p=4, m=12)，因此当凑足1M数据后，我们需要对1M数据进行编码，生成512K的校验数据。

![ec-isal](https://raw.githubusercontent.com/ivanzz1001/storage-foundation/master/EC%E7%BC%96%E7%A0%81/image/isal/erasure-encode-0001.jpg)

### 1.1 编码过程

编码过程用到了3个函数：

```C
gf_gen_cauchy1_matrix()

ec_init_tables_base()

ec_encode_data_base()
```

#### 1) gf_gen_cauchy1_matrix()

基于柯西矩阵来生成编码矩阵。编码矩阵用一个`m*k`字节长度的buff来存放，生成的编码矩阵如下：

![ec-isal](https://raw.githubusercontent.com/ivanzz1001/storage-foundation/master/EC%E7%BC%96%E7%A0%81/image/isal/erasure-encode-0002.jpg)

>ps: encode-matrix每个数据块的大小为1字节

#### 2) ec_init_tables_base()

本函数根据编码矩阵`encode-matrix`的校验块部分（上图中的浅粉色数据块）生成`g_tbls`:

![ec-isal](https://raw.githubusercontent.com/ivanzz1001/storage-foundation/master/EC%E7%BC%96%E7%A0%81/image/isal/erasure-encode-0003.jpg)

>ps: g_tbls每个数据块的大小为32字节


#### 3) ec_encode_data_base()

使用g_tbls对数据进行编码:

![ec-isal](https://raw.githubusercontent.com/ivanzz1001/storage-foundation/master/EC%E7%BC%96%E7%A0%81/image/isal/erasure-encode-0004.jpg)

这里可以简单看下行列式的乘法实现：

```C
  // Generate EC parity blocks from sources
  ec_encode_data(len, k, p, g_tbls, frag_ptrs, &frag_ptrs[k]);
  
  
  void ec_encode_data(int len, int srcs, int dests, unsigned char *v, unsigned char **src,
                 unsigned char **dest)
  {
          ec_encode_data_base(len, srcs, dests, v, src, dest);
  }
  
  void
  ec_encode_data_base(int len, int srcs, int dests, unsigned char *v, unsigned char **src,
                      unsigned char **dest)
  {
          int i, j, l;
          unsigned char s;
  
          for (l = 0; l < dests; l++) {
                  for (i = 0; i < len; i++) {
                          s = 0;
                          for (j = 0; j < srcs; j++)
                                  s ^= gf_mul(src[j][i], v[j * 32 + l * srcs * 32 + 1]);
  
                          dest[l][i] = s;
                  }
          }
}
```

![ec-isal](https://raw.githubusercontent.com/ivanzz1001/storage-foundation/master/EC%E7%BC%96%E7%A0%81/image/isal/erasure-encode-0005.jpg)

从上面可以看到校验块(128K)的第`i`个字节，就是用8个src块(ps: 每个块128KB)的第`i`个字节，分别与`g_tbls`的8个块(ps: 每个块32字节）中的**第一个字节**进行`gf_mul`运算所得。


### 1.2 解码过程

解码的过程需要生成解码矩阵，而此过程需要我们自己编码实现。好在示例中，已经给出了生成解码矩阵的代码，理解了其中的含义，将其移植到我们的代码中即可。

解码相关函数：

```C
gf_gen_decode_matrix()

ec_init_tables_base()

ec_encode_data_base()
```

#### 1) gf_gen_decode_matrix()

![ec-isal](https://raw.githubusercontent.com/ivanzz1001/storage-foundation/master/EC%E7%BC%96%E7%A0%81/image/isal/erasure-encode-0006.jpg)

在一个stripe的所有chunk中，已知哪些块存在数据错误，例如上图：`nsrcErrs = 2`, `nErrs = 3`

![ec-isal](https://raw.githubusercontent.com/ivanzz1001/storage-foundation/master/EC%E7%BC%96%E7%A0%81/image/isal/erasure-encode-0007.jpg)

从编码矩阵中，按照src_in_err选择可以用于解码的方阵，decode_index为: `[0， 2， 3， 4， 6， 7,  8， 9]`

![ec-isal](https://raw.githubusercontent.com/ivanzz1001/storage-foundation/master/EC%E7%BC%96%E7%A0%81/image/isal/erasure-encode-0008.jpg)

对方阵求逆`gf_invert_matrix`：

![ec-isal](https://raw.githubusercontent.com/ivanzz1001/storage-foundation/master/EC%E7%BC%96%E7%A0%81/image/isal/erasure-encode-0009.jpg)

gf_invert_matrix得到的invert_matrix并非最终的解码矩阵。如下代码，生成`nerrs * k`解码矩阵：

```C
/*
* Generate decode matrix from encode matrix and erasure list
*
*/

static int
gf_gen_decode_matrix_simple(u8 *encode_matrix, u8 *decode_matrix, u8 *invert_matrix,
                            u8 *temp_matrix, u8 *decode_index, u8 *frag_err_list, int nerrs, int k,
                            int m)
{   
    int i, j, p, r;
    int nsrcerrs = 0;
    u8 s, *b = temp_matrix;
    u8 frag_in_err[MMAX];

    memset(frag_in_err, 0, sizeof(frag_in_err));

    // Order the fragments in erasure for easier sorting
    for (i = 0; i < nerrs; i++) {
            if (frag_err_list[i] < k)
                    nsrcerrs++;
            frag_in_err[frag_err_list[i]] = 1;
    }

    // Construct b (matrix that encoded remaining frags) by removing erased rows
    for (i = 0, r = 0; i < k; i++, r++) {
            while (frag_in_err[r])
                    r++;
            for (j = 0; j < k; j++)
                    b[k * i + j] = encode_matrix[k * r + j];
            decode_index[i] = r;
    }

    // Invert matrix to get recovery matrix
    if (gf_invert_matrix(b, invert_matrix, k) < 0)
            return -1;

    // Get decode matrix with only wanted recovery rows
    for (i = 0; i < nerrs; i++) {
            if (frag_err_list[i] < k) // A src err
                    for (j = 0; j < k; j++)
                            decode_matrix[k * i + j] = invert_matrix[k * frag_err_list[i] + j];
    }

    // For non-src (parity) erasures need to multiply encode matrix * invert
    for (p = 0; p < nerrs; p++) {
            if (frag_err_list[p] >= k) { // A parity err
                    for (i = 0; i < k; i++) {
                            s = 0;
                            for (j = 0; j < k; j++)
                                    s ^= gf_mul(invert_matrix[j * k + i],
                                                encode_matrix[k * frag_err_list[p] + j]);
                            decode_matrix[k * p + i] = s;
                    }
            }
    }
    return 0;
}
```

#### 2) ec_init_tables_base()

同编码过程，使用decode_matrix生成`g_tbls`


#### 3) ec_encode_data_base()

同编码过程，使用g_tbls与正常的数据块运算，恢复损坏的数据块。

![ec-isal](https://raw.githubusercontent.com/ivanzz1001/storage-foundation/master/EC%E7%BC%96%E7%A0%81/image/isal/erasure-encode-0010.jpg)

<br>
