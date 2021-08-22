---
layout: post
categories: Algorithm
tag: arrays
title: Some Array Practices
---
## Generate Array Elements by Rule
> There is an array generated by a rule.
> The first item is 1. If k is in the array, then k*3 +1 and k*2+1 are in the array.
>
> The array is sorted. There are no duplicate values.
> Please write a function that accepts an input N. It should return the index N of the array.
>
> For example [1, 3, 4, 7, 9, 10, 13, 15, 19, 21, 22, 27, ...] n=10, return 22
```Java
public static int generate(int n) {
        int[] count = new int[n + 1];
        int[] result = new int[n + 1];
        for (int i = 0; i <= n; i++) {
            count[i] = 2;
        }
        result[0] = 1;
        int min = 0;
        for (int i = 1; i <= n; i++) {
            min = result[i - 1] * 3 + 2;
            int min_i_1 = -1; //the index of generated 3*j + 1
            int min_i_2 = -1; //the index of generated 2*j + 1
            int tmp = 0;
            for (int j = 0; j < i; j++) {
               if (count[j] == 2) tmp = 2 * result[j] + 1;
               else if (count[j] == 1) tmp = 3 * result[j] + 1;
               else continue;
               if (tmp < min) {
                   min = tmp;
                   if (count[j] == 2) {
                       min_i_2 = j;
                       min_i_1 = -1;
                   } else if (count[j] == 1) {
                       min_i_1 = j;
                       min_i_2 = -1;
                   }
               }
            }
            result[i] = min;
            if (min_i_1 >= 0) {
                count[min_i_1] = 0;
            }
            if (min_i_2 >= 0) {
                count[min_i_2] = 1;
            }
        }
        return result[n];
    }
```
<!--more-->
## 5782. Maximum Alternating Subsequence Sum
>The alternating sum of a 0-indexed array is defined as the sum of the elements at even indices minus the sum of the elements at odd indices.
>
> For example, the alternating sum of [4,2,5,3] is (4 + 5) - (2 + 3) = 4.
>
> Given an array nums, return the maximum alternating sum of any subsequence of nums (after reindexing the elements of the subsequence).
>
> A subsequence of an array is a new array generated from the original array by deleting some elements (possibly none)
> without changing the remaining elements' relative order.
>
> For example, [2,7,4] is a subsequence of [4,2,3,7,2,1,4] (the underlined elements), while [2,4,2] is not.
>
> eg. [4,2,5,3] -> 7 (optimal [4,2,5])
>
> [5,6,7,8] -> 8 (optimal [8])
>
> [6,2,1,2,4,5] -> 10 (optimal [6,1,5])
```Java
    public static long maxAlternatingSum(int[] nums) {
        int n =  nums.length;
        long result = nums[0];
        for (int i = 1; i <n; i++) {
            //always find a pair of numbers that the difference is greater than 0,
            // similar to sell stock problem with out selling time limit
            result += Math.max((nums[i] - nums[i - 1]), 0);
        }
        return result;
    }

```

## 5780. Remove One Element to Make the Array Strictly Increasing
> Given a 0-indexed integer array nums, return true if it can be made strictly increasing after removing exactly one element, or false otherwise.
>
> If the array is already strictly increasing, return true.
> The array nums is strictly increasing if nums[i - 1] < nums[i] for each index (1 <= i < nums.length).
>
> eg. [1,2,10,5,7] -> true
>
> [100,21,100] -> true
>
> [2,3,1,2] -> false
```Java
public static boolean canBeIncreasing(int[] nums) {
        int removeIdx = -1;
        for (int i = 0; i < nums.length - 1; i++) {
            if (removeIdx >= 0 && nums[i] >= nums[i + 1]) {
                return false;
            }
            if (nums[i] >= nums[i + 1]) removeIdx = i;
        }
        return (removeIdx <= 0 || removeIdx + 1 == nums.length - 1 || nums[removeIdx - 1] < nums[removeIdx + 1]);
    }
```