---
title: LeetCode刷题：数组
date: 2023-04-06 21:35:04
tags: 数组
categories: 面向LeetCode刷题
mathjax: False
---

&emsp;&emsp;LeetCode刷题篇将从本篇开头，将会覆盖数组、树、堆、栈、队列、图等数据结构，也会包含一些刷题常用的算法，例如排序、二分查找、递归、贪心、动态规划等。每个部分都会单独拿出来形成一篇内容，我会记录一下常见的题型和解题思路，加深一下理解。

* [88. 合并两个有序数组（简单）](https://leetcode.cn/problems/merge-sorted-array/)

&emsp;&emsp;给你两个按非递减顺序排列的整数数组 nums1 和 nums2，另有两个整数 m 和 n ，分别表示 nums1 和 nums2 中的元素数目。请你合并 nums2 到 nums1 中，使合并后的数组同样按 非递减顺序排列。注意：最终，合并后数组不应由函数返回，而是存储在数组 nums1 中。为了应对这种情况，nums1 的初始长度为 m + n，其中前 m 个元素表示应合并的元素，后 n 个元素为 0 ，应忽略。nums2 的长度为 n 。

```Python
class Solution:
    def merge(self, nums1: List[int], m: int, nums2: List[int], n: int) -> None:
        """
        Do not return anything, modify nums1 in-place instead.
        """
        i, j = m - 1, n - 1
        k = m + n - 1
        while k>=0:
            if i==-1:
                nums1[k] = nums2[j]
                j-=1
            elif j==-1:
                nums1[k] = nums1[i]
                i-=1
            elif nums1[i] < nums2[j]:
                nums1[k] = nums2[j]
                j-=1
            else:
                nums1[k] = nums1[i]
                i-=1
            k-=1
```

&emsp;&emsp;本题关键：1.数组的大小已经固定成 m + n 了，不需要返回值，需要我们原地操作。2.题目要求将两个原本非递减的数组合并成一个非递减的数组，在本题的设置下，顺序比较大小并插入数字比较困难，而逆序从 nums1 末尾开始比较大小进行插入则水到渠成。3.注意边界情况，不要溢出。


* [27. 移除元素（简单）](https://leetcode.cn/problems/remove-element/description/)

&emsp;&emsp;给你一个数组 nums 和一个值 val，你需要原地移除所有数值等于 val 的元素，并返回移除后数组的新长度。不要使用额外的数组空间，你必须仅使用 O(1) 额外空间并原地修改输入数组。元素的顺序可以改变。你不需要考虑数组中超出新长度后面的元素。

```Python
class Solution:
    def removeElement(self, nums: List[int], val: int) -> int:
        left, right = 0, len(nums)-1
        while left <= right:
            if nums[left] == val:
                nums[left] = nums[right]
                right -= 1
            elif nums[right] == val:
                right -= 1
                left += 1
            else:
                left += 1
        return left
```

&emsp;&emsp;本题关键：1.本题同样要求原地操作，从数组中取出目标值，但是根据题意，其实只是将目标值移动到数组的末尾。2.双指针法，左指针按顺序依次比较，右指针用于指向数组末尾的目标值，最终返回的左指针就是“新”数组的长度。

* [26. 删除有序数组中的重复项（简单）](https://leetcode.cn/problems/remove-duplicates-from-sorted-array/)
  
&emsp;&emsp; 给你一个升序排列的数组 nums ，请你原地删除重复出现的元素，使每个元素只出现一次，返回删除后数组的新长度。元素的相对顺序应该保持一致。由于在某些语言中不能改变数组的长度，所以必须将结果放在数组nums的第一部分。更规范地说，如果在删除重复项之后有 k 个元素，那么 nums 的前 k 个元素应该保存最终结果。将最终结果插入 nums 的前 k 个位置后返回 k 。不要使用额外的空间，你必须在原地修改输入数组 并在使用 O(1) 额外空间的条件下完成。

```Python
class Solution:
    def removeDuplicates(self, nums: List[int]) -> int:
        fast, slow = 0, 0
        while fast < len(nums):
            if nums[slow] != nums[fast]:
                slow += 1
                nums[slow] = nums[fast]
            fast += 1
        return slow+1
```

&emsp;&emsp;本题关键：1.要求与27题一样，需要原地操作，虽然是从数组中删除重复项，但根本还是将不重复的值挪到前面去。2.双指针法，慢指针用于指向当前的值，快指针按顺序寻找与慢指针不重复的值，最终慢指针将指向“新”数组的末尾。