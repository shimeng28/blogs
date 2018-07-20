---
title: C++泛型编程(一)
date: 2018-04-23 20:05:06
tags: C++
---
  最近一直在学习c++, c++ 中包含有面向过程编程， 面向对象编程，以及泛型编程。这篇文章主要记录一些刚学的泛型编程。
  
   我们知道c++ STL(standard template library)主要包含两部分组成，一部分是容器，如vector，list,  deque, set, map等；另一部分是泛型算法，如find(), sort()等。
而泛型编程就是在各类容器和数组之上实现的各类算法，这些算法并不强调具体的数据类型，其依据的就是函数模板(function template), 构造出与容器类型无关的算法，但是其主要原理还是将容器抽象出前后两个iterator标兵，遍历以及操作容器元素值。

如果给我们一个数组，实现一个函数，该数组是否包含某一个值。如果包含则返回该值，否则返回0. 我们的代码可以如下所示：
```c++
int find(const vector<int> &vec, int value)
{
  for(int i = 0; i < vec.size(); ++i)
  {
    if(vec[i] == vlaue)  return vec[i];
  }
  return 0;
}
```


 测试发现，符合我们的要求，那么当我们传入的vector，其元素类型不仅是int, 还要是double, float, string等时，我们应该如何做呢？很简单，使用函数模板即可。代码如下：
 
```c++
   template <typename  elemType>
   elemType find(const vector<elemType> &vec, elemType value) 
   {
     for(int i = 0; i < vec.size(); ++i)
     {
       if(vec[i] == value)
         return vec[i];
     }
     return 0;
   }
```

上述代码我们用function template的模式，使得该函数不在局限于元素类型为int型的vector数据结果了。那么更进一步，我们不仅要让该函数可以作用在vector上，还可以作用在array上。一种方法是重载该函数，一份用于vector, 一份用于array. 然而明显的，作为有着专业素养的码农，我们可以直接pass掉这种解决问题的方式。我们可以更加的抽象一下，基于vector和array的共性，此时最先出现在我们脑海中的便是指针。因为vector和array都是顺序容器，完全可以传入容器的基地址和结束地址，来操作容器。ok, 请看下面的代码：
```c++
  template<typename elemType>
  elemType find(const elemType *first, const elemType *last, const elemType &value)
  {
     if(!first || !last) return 0;
     for(; first != last; ++first)
     {
       if(*first == value) return *first;
     } 
     return 0;
  }
```

现在我们已经能适用与顺序容器了，上述代码中**++first**的潜在含义是自增sizeof(elemType), 并且该容器的存储结构必须是顺序的。但是当要求以这种形式取得list的下一个元素时，并不符合我们的逻辑。

因此我们的解决方案便是在所有的容器之上抽象出一层，即实现一个类，这个类会封装这些容器元素，并且重载其符号运算符(++, --, ==等)。具体如何实现，且听下回分解。