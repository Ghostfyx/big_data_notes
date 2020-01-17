# 2.3 Spark核心概念简介

每个Spark应用都由一个驱动器程序（driver program）来发起集群上的各种并行操作，驱动器程序包含应用的main函数，如：

```java
// Java
public static void main(String[] args){}
```

```python
if __name == '__main__':
```

