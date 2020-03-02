---
title: "[leetcode] 0665 Non-decreasing Array"
date: 2020-03-02 21:25:09
categories: leetcode
tags:
---

https://leetcode.com/problems/non-decreasing-array/

給定一個整數陣列，檢查它是否 non-decreasing。檢查的過程中，可以允許修改一個元素的值使它符合條件。

舉例來說 `1, 3, 3, 2, 4` 這是可被接受的數列，因為只要將 2 改成 3 即可。

並且從上面的例子可以看到，當出現沒有排好的元素時，該如何處理？

當 `nums[i] < nums[i-1]` 發生時，我們可以考慮改 `nums[i]` 或是 `nums[i-1]`。
*   如果 `nums[i-1] == nums[i-2]`，此時就只能將 `nums[i]` 改大，因為我們無法同時改 `nums[i-1]` 與 `nums[i-2]`，這樣就改超過一個元素。
*   如果 `nums[i-1] != nums[i-2]`，我們可以將 `nums[i]` 的值變大或是將 `nums[i-1]` 的值變小。我們會希望儘量將 `nums[i-1]` 變小，因為如果將 `num[i]` 變大將有可能使後面的元素違反限制。
    *   如果 `nums[i] < nums[i-2]`，那麼將 `nums[i-1]` 的值變小沒有用，因為 `nums[i]` 仍然比 `nums[i-2]` 還小，違反限制，此時我們只能將 `nums[i]` 改大
    *   如果 `nums[i] >= num[i-2]`，將 `nums[i-1]` 改小成 `nums[i]`

根據這個想法，解法如下：

    class Solution {
    public:
        bool checkPossibility(vector<int>& nums) {
            bool firstErr = false;
            for (int i = 1; i < nums.size(); i++) {
                if (nums[i] < nums[i-1]) {
                    if (firstErr) {
                        return false;
                    } else {
                        firstErr = true;
                        if (i < 2) {
                            nums[i-1] = nums[i];
                        } else {
                            if (nums[i-1] == nums[i-2]) {
                                nums[i] = nums[i-1];
                            } else {
                                if (nums[i] < nums[i-2]) {
                                    nums[i] = nums[i-1];
                                } else {
                                    nums[i-1] = nums[i];
                                }
                            }
                        }
                    }
                }
            }
            return true;
        }
    };


由於 `i-2` 是個很擾民的檢查，加上如果陣列元素只有兩個的情況一定是 true，所以整理一下：

    class Solution {
    public:
        bool checkPossibility(vector<int>& nums) {
            bool firstErr = false;
            if (nums.size() < 2) return true;
            if (nums[1] < nums[0]) {
                nums[0] = nums[1];
                firstErr = true;
            }
            for (int i = 2; i < nums.size(); i++) {
                if (nums[i] < nums[i-1]) {
                    if (firstErr) {
                        return false;
                    } else {
                        firstErr = true;
                        if (nums[i-1] == nums[i-2]) {
                            nums[i] = nums[i-1];
                        } else {
                            if (nums[i] < nums[i-2]) {
                                nums[i] = nums[i-1];
                            } else {
                                nums[i-1] = nums[i];
                            }
                        }
                    }
                }
            }
            return true;
        }
    };

在那群 if-else 裡，可以發現，只有當 `nums[i-1] != nums[i-2] && nums[i] >= nums[i-2]` 時，我們才會設定 `nums[i-1] = nums[i]`，其它情況都是 `nums[i] = nums[i-1]`

    class Solution {
    public:
        bool checkPossibility(vector<int>& nums) {
            bool firstErr = false;
            if (nums.size() < 2) return true;
            if (nums[1] < nums[0]) {
                nums[0] = nums[1];
                firstErr = true;
            }
            for (int i = 2; i < nums.size(); i++) {
                if (nums[i] < nums[i-1]) {
                    if (firstErr) {
                        return false;
                    } else {
                        firstErr = true;
                        if (nums[i-1] != nums[i-2] && nums[i] >= nums[i-2]) {
                            nums[i-1] = nums[i];
                        } else {
                            nums[i] = nums[i-1];
                        }
                    }
                }
            }
            return true;
        }
    };

面試當中，也有可能會被問到，如何不更改原本陣列的值達到同樣的功能。這時候就需要一個額外的變數 `comp` 儲存可能會被更新的 `nums[i]` 的值，這時候要很小心地將每個 branch 都檢查一次。

    class Solution {
    public:
        bool checkPossibility(vector<int>& nums) {
            bool firstErr = false;
            if (nums.size() < 2) return true;
            int comp = nums[1];
            if (nums[1] < nums[0]) {
                firstErr = true;
            }
            for (int i = 2; i < nums.size(); i++) {
                if (nums[i] < comp) {
                    if (firstErr) {
                        return false;
                    } else {
                        firstErr = true;
                        if (nums[i-1] != nums[i-2] && nums[i] >= nums[i-2]) {
                            comp = nums[i];
                        } else {
                            comp = nums[i-1];
                        }
                    }
                } else {
                    comp = nums[i-1];
                }
            }
            return true;
        }
    };

寫完之後，就要餵一些數據測試看看，根據前面考慮的細節，底下是一些可以測看看的數據：
*   `1, 3, 2, 4`
*   `3, 5, 1, 3`
*   `3, 3, 2, 2`

這是個考驗細心跟耐心的題目，第一次寫很容易就在一些條件檢查上搞糊塗，最好是寫下來比較清楚。
