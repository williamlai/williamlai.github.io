---
title: Leetcode 0001 Two Sum
date: 2022-10-24 08:57:12
categories: leetcode
tags:
- Leetcode
- Rust
---

# Idea

It's a pretty fundamental problem. So I'm gonna solve this problem with Rust.

First, create a hash map. For each number *n* of index *idx* in the array, find *n* in the hash map. We have the answer if *n* is in the hash map. If not, then insert *target - n* as key and *idx* as value into hash map.

<!-- more -->

# Solution

```
use std::collections::HashMap;

impl Solution {
    pub fn two_sum(nums: Vec<i32>, target: i32) -> Vec<i32> {
        let mut hash_map = HashMap::new();
        for (idx, n) in nums.iter().enumerate() {
            match hash_map.get(n) {
                Some(&val) => return vec![val as i32, idx as i32],
                None => hash_map.insert(target - n, idx),
            };
        }
        vec![]
    }
}
```

We can reduce calculation time by creating hash map as following `HashMap::with_capacity(nums.len)`.

I return an empty vector in the end. We can replace it with `unreachable!()` to make it more readable.