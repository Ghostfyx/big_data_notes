# Java8 HashMap新特性

从源码入手对HashMap源码进行分析，围绕Java8的HashMap新特性：

- 引入红黑树数据结构
- 扩容计算方式改变
- 头插入改为尾插入

## 1. HashMap源码

### 1.1 静态常量

```java
/**
 * 默认初始大小，值为16，要求必须为2的幂
*/
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16

/**
* 最大容量，必须不大于2^30
 */
static final int MAXIMUM_CAPACITY = 1 << 30;

/**
* 默认加载因子，值为0.75
*/
static final float DEFAULT_LOAD_FACTOR = 0.75f;

/**
* HashMap的空数组
*/
static final Entry<?,?>[] EMPTY_TABLE = {};

/**
  * 可选的默认哈希阈值
*/
static final int ALTERNATIVE_HASHING_THRESHOLD_DEFAULT = Integer.MAX_VALUE;
```

### 1.2 构造函数

**无参构造函数**

```java
 public HashMap() {
        this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
 }
```

