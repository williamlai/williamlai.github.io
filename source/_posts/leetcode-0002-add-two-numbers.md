---
title: Leetcode 0002 Add Two Numbers
date: 2022-10-28 16:49:25
categories: leetcode
tags:
- Leetcode
- Rust
- CPP
---

# Idea

It's a trivial problem. Two inputs are linked lists, and the output is a linked list. I'll just create a linked list as an output whose memory comes from the heap. For each digit, the value is the summation of the digit in list1, the digit in list2, and the carry. If the sum is larger or equal to 10, then modularize it to 10 and mark carry as 1. After summation for 1 digit, step to the next digit and take care of NULL pointers.

<!-- more -->

# Solution

## Rust

Here is the Rust solution.

```
impl Solution {
    pub fn add_two_numbers(l1: Option<Box<ListNode>>, l2: Option<Box<ListNode>>) -> Option<Box<ListNode>> {
        let mut dummy_head = ListNode::new(0);
        let mut current = &mut dummy_head;
        let mut idx1 = l1;
        let mut idx2 = l2;
        
        let mut carry: i32 = 0;
        
        while idx1 != None || idx2 != None {
            let sum = match (&idx1, &idx2) {
                (Some(node1), Some(node2)) => node1.val + node2.val + carry,
                (Some(node1), None) => node1.val + carry,
                (None, Some(node2)) => node2.val + carry,
                (None, None) => carry,
            };
            
            carry = sum / 10;
            
            current.next = Some(Box::new(ListNode::new(sum % 10)));
            current = current.next.as_mut().unwrap();
            
            idx1 = if idx1 != None { idx1.unwrap().next } else { idx1 };
            idx2 = if idx2 != None { idx2.unwrap().next } else { idx2 };
        }
        
        if carry > 0 {
            current.next = Some(Box::new(ListNode::new(carry)));
        }
        
        dummy_head.next
    }
}
```

When dealing with pointers, it's usually designed as an `Option` enumeration. `Some` denotes there is a value, and `None` denotes a NULL pointer. It's a good design because we can only extract value when it's `Some`. We can see how it works when we do the summation in the match expression.

Since the output size is unknown, so the output is created from Box. It means that it uses memory in a heap.

## CPP

Here is the CPP solution.

```
class Solution {
public:
    ListNode* addTwoNumbers(ListNode* l1, ListNode* l2) {
        ListNode *pDummyHead = new ListNode(0);
        ListNode *pCurrent = pDummyHead;
        ListNode *pIdx1 = l1;
        ListNode *pIdx2 = l2;
        
        int carry = 0;
        
        while (pIdx1 != NULL || pIdx2 != NULL) {
            int sum = carry;
            if (pIdx1 != NULL) {
                sum += pIdx1->val;
            }
            if (pIdx2 != NULL) {
                sum += pIdx2->val;
            }

            pCurrent->next = new ListNode(sum % 10);
            pCurrent = pCurrent->next;
            carry = sum / 10;
            
            if (pIdx1 != NULL) {
                pIdx1 = pIdx1->next;
            }
            if (pIdx2 != NULL) {
                pIdx2 = pIdx2->next;
            }
        }
        if (carry > 0) {
            pCurrent->next = new ListNode(carry);
        }
        return pDummyHead->next;
    }
};
```
