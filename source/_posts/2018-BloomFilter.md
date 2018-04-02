---
title: Bloom Filter的推导和实现
date: 2017-04-13 00:26:51
tags: 算法
mathjax: true
---
## 布隆过滤器
写了个项目用到了布隆过滤器去处理几十万条的域名字符串，主要希望能有较快的查找速度

第一次使用哈希，记性不太好，记一下实现思路

### 算法思想：
1. 首先需要k个hash函数，每个函数可以把key散列成为1个整数
2. 初始化时，需要一个长度为m比特的数组，每个比特位初始化为0
3. 某个key加入集合时，用k个hash函数计算出k个散列值，并把数组中对应的比特位置为1
4. 判断某个key是否在集合时，用k个hash函数计算出k个散列值，并查询数组中对应的比特位，如果所有的比特位都是1，认为在集合中。

**优点**：不需要存储key，节省空间

**缺点**：
1. 存在假阳性误判（False Positive）
2. 无法删除

#### 复杂度分析
空间复杂度：Bloom Filter不会动态增长，运行过程中维护的始终只是m位的bitset，所以空间复杂度只有O(m)。

时间复杂度：Bloom Filter的插入与属于操作主要都是在计算k个hash，所以都是O(k)。

![image](http://blog-1252932770.cosgz.myqcloud.com/BloomFilter/BloomFilter.png)


### False positives 概率推导

假设 Hash 函数以等概率条件选择并设置 Bit Array 中的某一位，m 是该位数组的大小，k 是 Hash 函数的个数，那么位数组中某一特定的位在进行元素插入时的 Hash 操作中没有被置位的概率是：

```math
 $1-\frac{1}{m}$
```

那么在所有 k 次 Hash 操作后该位都没有被置 "1" 的概率是：
```
 (1-\frac{1}{m})^k
```

如果我们插入了 n 个元素，那么某一位仍然为 "0" 的概率是：
```math
 (1-\frac{1}{m})^{kn}
```

因而该位为 "1"的概率是：
```math
 1- (1-\frac{1}{m})^{kn}
```

现在检测某一元素是否在该集合中。标明某个元素是否在集合中所需的 k 个位置都按照如上的方法设置为 "1"，但是该方法可能会使算法错误的认为某一原本不在集合中的元素却被检测为在该集合中（False Positives），该概率由以下公式确定：
```math
 [1- (1-\frac{1}{m})^{kn}]^k \approx (1-{e}^{-kn/m})^k
```

### 最优哈希函数个数k
其实上述结果是在假定由每个 Hash 计算出需要设置的位（bit） 的位置是相互独立为前提计算出来的，不难看出，随着 m （位数组大小）的增加，假正例（False Positives）的概率会下降，同时随着插入元素个数 n 的增加，False Positives的概率又会上升，对于给定的m，n，如何选择Hash函数个数 k 由以下公式确定：

```math
 \frac{m}{n}\ln2\approx 0.7\frac{m}{n}
```
此时False Positives的概率为：
```math
 2^{-k}\approx 0.6185^\frac{m}{n}
```

### 最优位数组大小m
而对于给定的False Positives概率 p:
```math
 m=-\frac{n\ln p}{(\ln2)^2}
```

上式表明，位数组的大小最好与插入元素的个数成线性关系，对于给定的 m，n，k，假阳性概率最大为：

```math
(1-{e}^{-k(n+0.5)/(m-1)})^k
```

### Python实现摘录
参考了一些C++版本的，项目用到Python，Python的实现简洁了不少，也较快能实现

Python大法好，立个Flag，抽点时间系统学一下Python
```python
import cmath
from BitVector import BitVector

class BloomFilter(object):
    def __init__(self, error_rate, elementNum):
         #计算所需要的bit数
        self.bit_num = -1 * elementNum * cmath.log(error_rate) / (cmath.log(2.0))
        
        #四字节对齐
        self.bit_num = self.align_4byte(self.bit_num.real)
	#print 'bit_num = %d' % self.bit_num        

        #分配内存
        self.bit_array = BitVector(size=self.bit_num)
        
        #计算hash函数个数, only effected by error_rate
        self.hash_num = cmath.log(2) * self.bit_num / elementNum
        self.hash_num = self.hash_num.real
        
        #向上取整
        self.hash_num = int(self.hash_num) + 1
        #print 'hash_num = %d' % self.hash_num

        #产生hash函数种子
        self.hash_seeds = self.generate_hashseeds(self.hash_num)
	#for s in self.hash_seeds:
	#	print s

    /* 
    * hash_element函数，用于计算字符串哈希值，基于BKDR哈希实现
    * seed：BKDR hash函数素数种子，间隔大于50
    * element：恶意网址字符串
    */
    def hash_element(self, element, seed): # BKDR hash for string
        hash_val = 1  
        for ch in str(element):
            hash_val = hash_val * seed + ord(ch)
    
        hash_val = abs(hash_val)  #取绝对值
        hash_val = hash_val % self.bit_num  #取模，防越界
        return hash_val
        
    # insert_element函数，用于添加新的元素
    def insert_element(self, element):
    for seed in self.hash_seeds:
        hash_val = self.hash_element(element, seed)
        self.bit_array[hash_val] = 1   #设置相应的比特位
    
    #检查元素是否存在，存在返回true，否则返回false
    def has_element(self, element):
    for seed in self.hash_seeds:
        hash_val = self.hash_element(element, seed) #calculate hash value    
        if self.bit_array[hash_val] == 0:   #查看值
            return False
        return True

```



## 对比哈希表
空间占用小，哈希表一条数据需占用一个指定大小的字节空间

## 参考链接：

[1] Wiki： https://en.wikipedia.org/wiki/Bloom_filter

[2] 数学之美：http://www.google.com.hk/ggblog/googlechinablog/2007/07/bloom-filter_7469.html

[3] Bloom Filters - the math :http://pages.cs.wisc.edu/~cao/papers/summary-cache/node8.html