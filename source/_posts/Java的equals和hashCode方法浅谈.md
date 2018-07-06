---
title: Java的equals和hashCode方法浅谈
date: 2018-07-02 16:20:39
categories: 技术人生
tags: Java
---
`equals`和`hashCode`两个方法都是`Object`类里的方法，作为Java基础经常在面试中被问到的一个问题是：二者关系是什么样的？或者这么说，当`equals`方法返回`true`时，`hashCode`一样吗？当`hashCode`一样时，`equals`方法返回什么？先给出答案：`equals`返回`true`，`hashCode`一定一样，反过来，如果`hashCode`一样时，`equals`可以返回`false`，很多人都知道这一点，可是为什么是这样呢，有些人可能就不知道了

- `equals`和`==`有什么区别？
- `equals`和`hashCode`有什么关系？
- `equals`和`hashCode`如何编写？

先来看见`equals`方法的签名
，
```java 
public boolean equals(Object obj) {
    return (this == obj);
}
```

可以看到入参是`Object`，很多人没有注意到这一点，上来就写了一个具体的类。equals方法顾名思义就判断对象的相等性，默认实现就是==，那么说到二者的区别，个人理解，equals方法是一种用户定义的“逻辑等”，而==是一种“物理等”，用俗语解释就是，equals判断是否相同，==判断是否一样。

`equals`方法在编写的时候需要遵循以下原则：

- 自反性
- 对称性
- 传递性
- 一致性

下面展开说一下，

自反性的意思是，对于一个非`null`的对象x，`x.equals(x)`一定为`true`，这是显而易见的，无须赘述。

对称性，对于非`null`对象x、y，`x.equals(y) == true`，当且仅当`y.equals(x) == true`。

传递性，对于非`null`对象x、y、z，如果`x.equals(y) == true`且`y.equals(z) == true`，那么`x.equals(z) == true`

 一致性，对于非`null`对象x、y，多次调用`x.equals(y)`返回一致
