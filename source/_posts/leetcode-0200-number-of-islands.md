---
title: "[leetcode] 0200 number of islands"
date: 2020-03-05 21:11:50
categories:
tags: leetcode
---

https://leetcode.com/problems/number-of-islands/

有一個二維陣列，裡面的值如果是 1 代表陸地，0 代表是水。如果一個陸地在另一塊陸地的上下左右方則視為相連，對角線不視為相連。假設陣列外都是水，找出島的數量有多少。

我們可以從左上掃到右下，如果遇到 1 則向外尋找，這裡有兩個要考慮的點：
*   向外尋找的方式，這裡應該可以使用 DFS 或 BFS
*   在向外尋找時，要避免往回重複尋找。如果該陣列可以被修改的話，我們就將找過的地方將它的值改成別的值即可。

第一個解法先用 DFS，並且修改它的值代表已經找過

    class Solution {
        void dfs(vector<vector<char>>& grid, int i, int j) {
            if (i < 0 || j < 0 || i >= grid.size() || j >= grid[0].size() || grid[i][j] != '1') return;
            grid[i][j] = 'x';
            dfs(grid, i+1, j);
            dfs(grid, i-1, j);
            dfs(grid, i, j+1);
            dfs(grid, i, j-1);
        }
    public:
        int numIslands(vector<vector<char>>& grid) {
            if (grid.empty()) return 0;
            int cnt = 0;
            for (int i=0; i<grid.size(); i++) {
                for (int j=0; j<grid[0].size(); j++) {
                    if (grid[i][j] == '1') {
                        dfs(grid, i, j);
                        cnt++;
                    }
                }
            }
            return cnt;
        }
    };

由於每個點我們只會經過一次，所以這個演算法的複雜度是 O(n^2)

在這個解法裡，我們將拜訪過的值設成 'x'，當然我們也可以設成 '0'，但是設成 'x' 的好處是我們仍然知道那個地方本來是塊陸地，如果我們要復原的話就將 'x' 改回 '1' 即可。

第二個解法用 BFS， 修改它的值代表已經找過

    class Solution {
        void push_cadidates(vector<vector<char>>& grid, int i, int j, queue<pair<int,int>>& q) {
            if (i < 0 || j < 0 || i >= grid.size() || j >= grid[0].size() || grid[i][j] != '1') return;
            q.push({i,j});
            grid[i][j] = 'x';
        }
        void bfs(vector<vector<char>>& grid, queue<pair<int,int>>& q) {
            while (!q.empty()) {
                int qsize = q.size();
                for (int i=0; i<qsize; i++) {
                    auto p = q.front();
                    q.pop();
                    push_cadidates(grid, p.first-1, p.second, q);
                    push_cadidates(grid, p.first+1, p.second, q);
                    push_cadidates(grid, p.first, p.second-1, q);
                    push_cadidates(grid, p.first, p.second+1, q);
                }
            }
        }
    public:
        int numIslands(vector<vector<char>>& grid) {
            if (grid.empty()) return 0;
            int cnt = 0;
            for (int i=0; i<grid.size(); i++) {
                for (int j=0; j<grid[0].size(); j++) {
                    if (grid[i][j] == '1') {
                        queue<pair<int,int>> q;
                        q.push({i,j});
                        grid[i][j] = 'x';
                        bfs(grid, q);
                        cnt++;
                    }
                }
            }
            return cnt;
        }
    };

在 leetcode 上的討論還有使用 union find，但我對 union find 很不熟，這裡就先跳過。

這題算是相當簡單的題目，也不容易出錯，算是要好好把握的題目。

