本文以libstdc++的源码作为参考。

## sort

sort()是不稳定的，它综合了快排、堆排和插入排序三种算法，并做了一定优化。相比于c库的qsort，我更推荐使用stl的sort，因为它能够将用户自定义的比较函数内联展开，以得到更好的性能。

影响sort()的使用哪个排序算法的因素主要有两个，分别是数组的长度N和递归深度depth，其中，N的下限是16， depth的上限是`log2（N）*2`.

sort算法的流程如下：

- 首先，按照快排的方式递归
  - 若当前数组范围长度N小于16，直接返回。
  - 若当前递归深度depth超过了上限，则将当前数组范围内的所有元素进行堆排序。
  - 否则，将当前范围内的元素进行一次快排的partition，将当前范围分为两个新的子范围，递归向下。
- 现在，数组大部分是有序的了，对数组进行一次插入排序。

对应以上流程，现在开始分析源码。为了方便阅读，以下源码省略了模板参数等乱七八糟的东西，变量名也删除了下划线前缀。

首先是入口，sort函数调用__sort进行排序，该函数代码如下。

```c++
void __sort(RandomAccessIterator first, RandomAccessIterator last, Compare comp)
{
    if (first != last)
    {
        //快排的递归向下
    	std::introsort_loop(first, last, lg(last - first) * 2, comp);
        //对整体数组进行的插入排序
    	std::final_insertion_sort(first, last, comp);
    }
}
```

`introsort_loop`函数的代码如下。不同于我们自己写快排时，对于patition出来的两个子数组会有各有一次递归调用，`introsort_loop`函数使用了循环替代了其中一个递归调用。

```c++
void introsort_loop(RandomAccessIterator first, RandomAccessIterator last, Size depth_limit, Compare comp)
{
    //判断当前数组长度是否大于S_threshold（其值为16）
    while (last - first > S_threshold)
    {
        //判断递归下降的深度
        if (depth_limit == 0)
        {
            //如果递归层数太多，说明快排已经不太有效
            //此时直接使用堆排序
            std::partial_sort(first, last, last, comp);
            return;
        }
        --depth_limit;
        //进行一次patition，pivot从数组开头，结尾和中间三者中选次大值
        RandomAccessIterator cut = unguarded_partition_pivot(first, last, comp);
        //对后半部分数组递归进行排序
        introsort_loop(cut, last, depth_limit, comp);
        //将前半部分数组当做循环的下一轮数组进行排序
        //本质上使用循环代替递归
        last = cut;
    }
}
```

`final_insertion_sort`函数是略带一点优化的插入排序，其源码没有多大的分析价值。

## stable_sort

stable_sort()是stl提供的稳定的排序算法，它综合了归并排序和插入排序，是TIMsort的一个实现。由于归并排序缓冲区的问题，它的实现比sort()麻烦得多。

在描述stable_sort的具体流程前，需要先说明buffer大小足够时的merge和buffer大小不足时的merge的流程。

### 带充足buffer的merge

如果 buffer大小 大于等于 待merge数组的总大小，那么merge就很简单了。只需要不断将两个待merge数组中较小的数复制到buffer中即可。代码如下：

```c++
//参数解释：
//fisrt1，last1表示第一个待merge数组的开头结尾
//first2，last2同理
//result表示buffer开头
//comp是用于比较的仿函数
OutputIterator __move_merge(InputIterator first1, InputIterator last1,
                 InputIterator first2, InputIterator last2,
                 OutputIterator result, Compare comp)
{
    while (first1 != last1 && first2 != last2)
    {
        if (comp(first2, first1))
        {
            //不要被_GLIBCXX_MOVE的名字所欺骗
            //它表示尽可能对该元素采用move而非copy的方式进行复制。
            *result = _GLIBCXX_MOVE(*first2);
            ++first2;
        }
        else
        {
            *result = _GLIBCXX_MOVE(*first1);
            ++first1;
        }
        ++result;
    }
    //将剩余的未拷贝的元素拷贝到buffer中。
    return _GLIBCXX_MOVE3(first2, last2, _GLIBCXX_MOVE3(first1, last1, result));
}
```

### 不充足buffer的merge

不充足buffer的意思是，buffer的大小 小于等于 待merge数组的总大小。为什么会出现这种情况呢？当stl准备对两个数组进行merge时，会分配一个buffer。但是这个buffer大小是有上限的，从而导致buffer大小小于待merge数组总大小的情况。

根据buffer的大小，还要细分为两种情况进行讨论，在这之前，我们记`[first, middle)`作为第一个待merge的数组（数组1），它的长度为len1，`[middle, last)`作为第二个被merge的数组（数组2），它的长度为len2。

#### 使用较小buffer的merge

当`buffer.size() >= min(len1, len2)`时，如果`len1<=len2`，先将数组1的元素拷贝到buffer中，然后再从前往后，将buffer中的元素与[middle, last)范围内的数据merge到[first, last)范围内。merge的时间复杂度是O(len1+len2)。
![](.\stl排序算法源码分析\buffer较大时merge.png)

如果`len1>len2`，将数组2的元素拷贝到buffer中，然后从后往前，进行merge。

![](.\stl排序算法源码分析\buffer较大时merge2.png)

总的来说，只要buffer能够容纳其中一个数组，就能够以O(len1+len2)的复杂度进行merge。

#### 不使用buffer的merge

当`buffer.size() < min(len1, len2)`时，如果`len1>len2`，先将len1对半分割，得到数组1的中间部分的位置为first_cut。然后根据first_cut的值去数组2中找到第一个大于`*first_cut`的值对应的位置(lower_bound)，记为second_cut。如下图所示：

![](.\stl排序算法源码分析\无buffer的merge1.png)

现在，`[middle, second_cut)`范围内的元素一定小于等于`[first_cut, middle)`，将这两个范围内的元素进行旋转，得到new_middle，即`first_cut+(second_cut - middle)`的值。

![](.\stl排序算法源码分析\无buffer的merge2.png)

接下来再对[first, first_cut)和[first_cut, new_middle)范围内的数据递归地进行merge，同样，对于[new_middle, second_cut)和[second_cut, last)范围内的数据也要递归地进行merge。

如果`len1<len2`，思路是一样的，只是将len2对半分割，而不是len1，接下来去数组1中找到第一个大于等于*second_cut的值对应的位置，同样进行旋转，接下来的流程略。

### stable_sort源码分析

这里先给出stable_sort的整体流程：

- 首先，尝试分配与数组相同大小的buffer
- 如果buffer分配完全失败，即buffer大小为0
  - 如果数组的大小小于15，直接对数组进行插入排序
  - 否则，实行归并排序，将数组均分为两个子数组，对每个子数组递归向下进行归并
  - 当两个子数组都有序后，不使用buffer对两个子数组进行merge。
- 如果buffer分配出来了
  - 如果buffer大小大于

## list.sort