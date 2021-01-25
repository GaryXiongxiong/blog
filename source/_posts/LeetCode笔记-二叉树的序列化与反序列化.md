---
title: LeetCode笔记 二叉树的序列化与反序列化
date: 2020-09-19 15:16:24
tags:
- Algorithm
categories: 
- blog
---

## 题目
序列化是将一个数据结构或者对象转换为连续的比特位的操作，进而可以将转换后的数据存储在一个文件或者内存中，同时也可以通过网络传输到另一个计算机环境，采取相反方式重构得到原数据。

请设计一个算法来实现二叉树的序列化与反序列化。这里不限定你的序列 / 反序列化算法执行逻辑，你只需要保证一个二叉树可以被序列化为一个字符串并且将这个字符串反序列化为原始的树结构。

示例: 
```
你可以将以下二叉树：

    1
   / \
  2   3
     / \
    4   5

序列化为 "[1,2,3,null,null,4,5]"
```
来源：力扣（LeetCode）
链接：https://leetcode-cn.com/problems/serialize-and-deserialize-binary-tree
著作权归领扣网络所有。商业转载请联系官方授权，非商业转载请注明出处。

## 思路

大体思路为BFS，不过过程中需要处理字节点为null的情况。所以突发奇想引入Java中的Optional类来简化算法

## 解决

```java
/**
 * Definition for a binary tree node.
 * public class TreeNode {
 *     int val;
 *     TreeNode left;
 *     TreeNode right;
 *     TreeNode(int x) { val = x; }
 * }
 */

import java.util.*;

public class Codec {
    public String serialize(TreeNode root) {
        if(null==root){
            return "[]";
        }
        Queue<Optional<TreeNode>> queue = new LinkedList<>();
        StringBuilder sb = new StringBuilder("[");
        queue.offer(Optional.of(root));
        int nullCounter = 0;
        while(!queue.isEmpty()){
            Optional<TreeNode> curNode = queue.poll();
            if(curNode.isEmpty()){
                nullCounter--;
                sb.append("null,");
            }
            else{
                Optional<TreeNode> left = Optional.ofNullable(curNode.get().left);
                Optional<TreeNode> right = Optional.ofNullable(curNode.get().right);
                if(left.isEmpty()){
                    nullCounter++;
                }
                if(right.isEmpty()){
                    nullCounter++;
                }
                queue.offer(left);
                queue.offer(right);
                sb.append(curNode.get().val+",");
            }
            if(nullCounter==queue.size()){
                break;
            }
        }
        sb.delete(sb.length()-1,sb.length());
        sb.append("]");
        return sb.toString();
    }

    public TreeNode deserialize(String data) {
        data = data.replace("[","").replace("]","");
        LinkedList<String> elements = new LinkedList<>(Arrays.asList(data.split(",")));
        Queue<Optional<TreeNode>> queue = new LinkedList<>();
        String rootElement = elements.pollFirst();
        if(rootElement==null||"null".equals(rootElement)||"".equals(rootElement)){
            return null;
        }
        TreeNode root = new TreeNode(Integer.parseInt(rootElement));
        queue.offer(Optional.ofNullable(root));
        while(!elements.isEmpty()&&!queue.isEmpty()){
            Optional<TreeNode> curNode = queue.poll();
            if(!curNode.isEmpty()){
                String left = elements.poll();
                String right = elements.poll();
                if(null!=left&&!"null".equals(left)){
                    curNode.get().left = new TreeNode(Integer.parseInt(left));
                }
                else{
                    curNode.get().left = null;
                }

                if(null!=right&&!"null".equals(right)){
                    curNode.get().right = new TreeNode(Integer.parseInt(right));
                }
                else{
                    curNode.get().right = null;
                }
                queue.offer(Optional.ofNullable(curNode.get().left));
                queue.offer(Optional.ofNullable(curNode.get().right));
            }
        }
        return root;

    }
}

// Your Codec object will be instantiated and called as such:
// Codec codec = new Codec();
// codec.deserialize(codec.serialize(root));
```