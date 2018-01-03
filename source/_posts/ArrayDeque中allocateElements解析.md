---
title: ArrayDeque中allocateElements解析
date: 2018-01-03 21:10:59
tags:
	- java
	- queue
---
jdk1.7源码如下
```java
private void allocateElements(int numElements) {
        int initialCapacity = MIN_INITIAL_CAPACITY;
        // Find the best power of two to hold elements.
        // Tests "<=" because arrays aren't kept full.
        if (numElements >= initialCapacity) {
            initialCapacity = numElements;
            initialCapacity |= (initialCapacity >>>  1);
            initialCapacity |= (initialCapacity >>>  2);
            initialCapacity |= (initialCapacity >>>  4);
            initialCapacity |= (initialCapacity >>>  8);
            initialCapacity |= (initialCapacity >>> 16);
            initialCapacity++;

            if (initialCapacity < 0)   
            // Too many elements, must back off
            initialCapacity >>>= 1;// Good luck allocating 2 ^ 30 elements
        }
        elements = (E[]) new Object[initialCapacity];
    }
```

方法目的在于，当numElements 小于 initialCapacity的时候，直接取默认值 initialCapacity，而大于默认值的时候取 n = 2^k (k 为 32 以内的整数) ，此时 n > numElements。
下面简单分析代码：
假设 a = 1xxxxxxx xxxxxxxx  (2进制表示 1表示该数的最高位, x代表0或者1)
a | a >>> 1    =  11xxxxxx xxxxxxxx  (最高两位为11)
a | a >>> 2    =  1111xxxx xxxxxxxx
a | a >>> 4    =  11111111 xxxxxxxx
a | a >>> 8    =  11111111 11111111
……..
最终的目的是将表示该数的最高位的1复制到接下来的1位、2位、4位直到16位,保证接下来a的低位全部是1，在加1一位导致高位左移，其余位数变成0，并且是2^k。
