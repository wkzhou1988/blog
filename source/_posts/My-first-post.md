---
title: My first post
date: 2020-05-07 11:17:27
tags:
---

`public class Solution {

​    bool compare(TreeNode s, TreeNode t)

​    {

​        if (s == null && t == null)

​            return true;



​        if (!(s != null && t != null))

​            return false;



​        if (s.val == t.val && compare(s.left, t.left) && compare(s.right, t.right))

​            return true;

​        return false;

​    }



​    public bool IsSubtree(TreeNode s, TreeNode t)

​    {

​        if (compare(s, t))

​            return true;

​        else if (s.left != null && IsSubtree(s.left, t))

​            return true;

​        else if (s.right != null && IsSubtree(s.right, t))

​            return true;

​        return false;

​    }

}`