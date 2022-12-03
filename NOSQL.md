# NOSQL基本概念
## BASE
Basically available, Soft state, Eventual Consistent
## CAP
Consistency, Availability, Partition tolerance
例子:
MongoDB: CP,主从结构，主节点挂掉后，剩余从节点重新选举出主节点
Cassandra: AP,全部都是对等节点
# 常见NOSQL

## Redis

### 常见数据结构
#### sds
```C
struct __attribute__ ((__packed__)) sdshdr5 { //最大长度32
    unsigned char flags; /* 3 lsb of type, and 5 msb of string length */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr8 {
    uint8_t len; /* used */
    uint8_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr16 {
    uint16_t len; /* used */
    uint16_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr32 {
    uint32_t len; /* used */
    uint32_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
struct __attribute__ ((__packed__)) sdshdr64 {
    uint64_t len; /* used */
    uint64_t alloc; /* excluding the header and null terminator */
    unsigned char flags; /* 3 lsb of type, 5 unused bits */
    char buf[];
};
```
flags前三位表示该sds的类型，后五位保留
#### hyperloglog

##### 算法思想:
对某一随机元素hash之后，其连续的0的个数的最大值n与随机的次数x有x = 2^n的关系
hyperloglog采用分桶+调和平均数来消除误差
##### redis实现:
```C
struct hllhdr {
    char magic[4];      /* "HYLL" */
    uint8_t encoding;   /* HLL_DENSE or HLL_SPARSE. */
    uint8_t notused[3]; /* Reserved for future use, must be zero. */
    uint8_t card[8];    /* Cached cardinality, little endian. */
    uint8_t registers[]; /* Data bytes. */
};
```
hyperloglog用来求集合大小的预估值，在redis中被抽象为多个6bit的计数器。
在实际代码中有两种表示方式
- sparse
  通过三种符号来表示
  1. ZERO 占用1byte 00xxxxxx代表 有0bxxxxxx个连续的计数器是空的
  2. XZERO 占用两个byte 01xxxxxxxxxxxxxx 代表0bxxxxxxxxxxxxxx个连续的计数器是空的
  3. VAL 占用1byte 1vvvvvxx 0bvvvvv表示计数器的值 0bxx代表连续的个数

- dense  
   +---------+---------+---------+---------//&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;//--+  
   |11000000|22221111|33333322|55444444&nbsp;....&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;|  
   +---------+---------+---------+---------//&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;//--+  
   每6位都是一个计数器，从低位到高位

当预估基数大于某值的时候redis会把表示从sparse切换到dense
#### bitmap
bitmap采用sds进行实现

#### listPack

LP_HDR_SIZE: 48 bit 32bit总长度 16bit元素个数

listPack的节点有如下的种类:
|   type    | first byte |           meaning           | length(byte) |
| :-------: | ---------- | :-------------------------: | :----------: |
| 7bit_uint | 0b0xxxxxxx |      7bit的无符号整数       |      2       |
| 6bit_str  | 0b10xxxxxx | 长度能通过6bit表示的字符串  |  无固定长度  |
| 13bit_int | 0b1100xxxx |      13bit的有符号整数      |      3       |
| 12bit_str | 0b1110xxxx | 长度能通过12bit表示的字符串 |  无固定长度  |
| 16bit_int | 0b11110001 |      16bit的有符号整数      |      4       |
| 24bit_int | 0b11110010 |      24bit的有符号整数      |      5       |
| 32bit_int | 0b11110011 |      32bit的有符号整数      |      6       |
| 64bit_int | 0b11110100 |      64bit的有符号整数      |      10      |
| 32bit_str | 0b11110000 | 长度能通过32bit表示的字符串 |  无固定长度  |
|    EOF    | 0b11111111 |            结尾             |      1       |

| data(上述不同种类) | backlen |

backlen是采用可变长编码的长度用于从后向前便利节点
每个byte的最高位是1代表前一byte依然是长度的一部分
例如 0b00001010 0b10001000 0b11010001 表示 0b101000010001010001 

listPack的在空间上连续 因此对于listpack的中间位置的添加 需要移动该位置后的全部内存 造成很大的资源消耗