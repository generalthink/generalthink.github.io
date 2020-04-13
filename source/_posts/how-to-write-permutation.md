---
title: 万万没想到,我还是找到了适合我理解的全排列实现方式
date: 2020-04-09 13:35:48
tags: [algorithm]
---

以前刚入行当CRUD boy的时候，我常去刷leetcode,每次遇到排列问题以及它的变种我就抓瞎了，然后搜索别人的实现方式我发现我都无法理解，甚至我还背过这类题的解法，可惜没啥用，过几天又忘记了，想不到时隔多年，我还是找到了一种适合我理解的实现方式。

我们初高中的时候，都学过排列，它的概念是这么说的：从 n 个不同的元素中取出m（1≤m≤n）个不同的元素，按照一定的顺序排成一列，这个过程就叫排列。当 m=n 这种特殊情况出现的时候，就是全排列（All Permutation）。如果选择出的这 m 个元素可以有重复的，这样的排列就是为重复排列（否则就是不重复排列。


如果有这样一道题：求 1,2,3,4,5五个数字的全排列，那么该如何实现呢？我们都知道答案是有120种(5*4*3*2*1=120)。

很明显这这里我们会想到用递归来做,以下是我的心路历程：

<!--more-->

![全排列](/images/java/permutation.png)


我这里只列举了3个数字的，不过也差不多一样的意思。代码应该很容易写出来，递归而已! 先回顾下递归三要素

1. 一个问题的解可以分解为几个子问题的解


2. 这个问题与分解之后的子问题，除了数据规模不同，求解思路完全一样
  

3. 存在递归终止条件
  5个数字都参与了全排列则递归终止


所以代码如下:

```java
public class Permutation {

  public static int count = 0;

  /**
   * @param numbers 函数执行前,当前还未参与全排列的数字
   * @param result 函数执行前,已经参与过全排列的数字以及顺序
   */
  public static void allPermutation(ArrayList<Integer> numbers, ArrayList<Integer> result) {

    // 所有数字都参与过了，则排列结束
    if (numbers.isEmpty()) {
      System.out.println(result);
      count++;
      return;
    }

    for (int i = 0; i < numbers.size(); i++) {

      // 从剩下的未排列的数字中选择一个加入结果
      ArrayList<Integer> newResult = (ArrayList<Integer>) result.clone();
      newResult.add(numbers.get(i));

      // 将以选择的数字从未排列的列表中移出
      ArrayList<Integer> newNumbers = (ArrayList<Integer>) numbers.clone();
      newNumbers.remove(i);

      // 递归调用，对于剩余的数字继续生成排列
      allPermutation(newNumbers, newResult);

    }

  }

  public static void main(String[] args) {

    ArrayList<Integer> numbers = new ArrayList<>();
    numbers.add(1);
    numbers.add(2);
    numbers.add(3);
    numbers.add(4);
    numbers.add(5);

    allPermutation(numbers, new ArrayList<>());

    assert count == 120;
  }
    
}

```

我认为这里**最重要的思想是将数字分为了两堆,分别是未参与排列的数字(numbers)以及已经参与过全排列的数字以及顺序(result)。**


万万没想到，CRUD boy也找到了属于自己解题的方式。