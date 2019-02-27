---
layout: post
title:  "二叉树基础"
date:   2019-02-08 00:14:54
categories: 数据结构
tags: BST 递归
excerpt: 遍历、前驱后继、查找、插入、删除、旋转
author: Tizeng
---

* content
{:toc}

首先二叉树的定义如下：

```c++
struct TreeNode {
    int val;
    struct TreeNode *left;
    struct TreeNode *right;
    struct TreeNode *parent;
    TreeNode(int x): val(x), left(NULL), right(NULL) {}
}
```

## 树的基本性质

* 节点的度：一个节点含有的子树的个数称为该节点的度

* 树的度：一棵树中，最大的节点的度称为树的度

* 叶节点或终端节点：度为零的节点

* 非终端节点或分支节点：度不为零的节点

* 节点的层次：从根开始定义起，根为第1层，根的子节点为第2层，以此类推

* 深度：对于任意节点n，n的深度为从根到n的唯一路径长，根的深度为0

* 高度：对于任意节点n，n的高度为从n到一片叶子节点的最长路径长，所有叶子节点的高度为0

* 从根结点到叶结点依次经过的结点（含根、叶结点）形成树的一条路径，最长路径的长度为树的深度

## 二叉树的基本性质

* 普通树的节点个数至少为1，而二叉树的节点个数可以为0；普通树节点的最大分支度没有限制，而二叉树节点的最大分支度为2

* 二叉树的第i层至多拥有`2^(i-1)`个节点

* 深度为k的二叉树至多总共有`2^(k+1)-1`个节点

* 平衡二叉树(Balanced Binary Tree)

    当且仅当任何节点的两棵子树的高度差不大于1的二叉树

* 完全二叉树(Complete Binary Tree)

    对于一颗二叉树，假设其深度为d(d>1)，除了第d层外，其它各层的节点数目均已达最大值，且第d层所有节点从左向右连续地紧密排列，这样的二叉树被称为完全二叉树。

    深度为k的完全二叉树，至少有`2^k`个节点，至多有`2^(k+1)-1`个节点。

* 满二叉树(Full Binary Tree)

    所有叶节点都在最底层的完全二叉树

    一棵深度为k，且有`2^(k+1)-1`个节点的二叉树

* 对任何一棵非空的二叉树T，如果其叶片（终端节点）数为n0，分支度为2的节点数为n2，则`n0=n2+1`

## 二叉搜索树的基本性质

* 根节点的值比所有左子树节点的值大，比所有右节点的值小（相等情况看情况而定）

## 二叉树的遍历

### 前序、中序、后序

前序、中序和后序遍历的区别就在于根节点和左孩子、右孩子的顺序。

```c++
// 前序遍历
void preorder(TreeNode* root){
    if(root == NULL)
        return;
    cout << root->val << endl;
    preorder(root->left);
    preorder(root->right);
}
// 中序遍历
void inorder(TreeNode* root){
    if(root == NULL)
        return;
    inorder(root->left);
    cout << root->val << endl;
    inorder(root->right);
}
// 后序遍历
void postorder(TreeNode* root){
    if(root == NULL)
        return;
    postorder(root->left);
    postorder(root->right);
    cout << root->val << endl;
}
```

还有一种遍历叫层次遍历，具体写在“二叉搜索树的常见套路”中。

### [二叉搜索树的后序遍历序列](https://www.nowcoder.com/practice/a861533d45854474ac791d90e447bafd?tpId=13&tqId=11176&rp=1&ru=%2Fta%2Fcoding-interviews&qru=%2Fta%2Fcoding-interviews%2Fquestion-ranking&tPage=2)

题目描述:

输入一个整数数组，判断该数组是不是某二叉搜索树的后序遍历的结果。如果是则返回`true`,否则返回`false`。假设输入的数组的任意两个数字都互不相同。


这道题考察对后序遍历的理解以及BST性质的运用。

性质1：对于一颗BST，根节点的值比左子树中所有的值都大，而比右子数中所有的值都小，我们可以根据这个性质从一个序列中分别找到哪些值是左子树的哪些是右子数的。

性质2：后序遍历最后才查找根节点，因此遍历结果的最后一个节点一定是根节点。

我们用递归的方式对每个节点来检查它的左子树和右子数是否符合性质1，并在每次递归时利用性质2来完成区分左右子树。

实现代码如下：

```c++
bool VerifySquenceOfBST(vector<int> sequence) {
    if(sequence.size() == 0)
        return false;
    return isBST(sequence, 0, sequence.size() - 1);
}

bool isBST(vector<int>& v, int left, int right){
    if(left >= right)
        return true;
    int idx = left;
    while(v[idx] < v[right]){
        idx++;
    }
    for(int i = left; i < idx; i++){
        if(v[i] > v[right])
            return false;
    }
    for(int i = idx; i < right; i++){
        if(v[i] < v[right])
            return false;
    }
    return isBST(v, left, idx - 1) && isBST(v, idx, right - 1);
}
```

## 节点的前驱和后继

前驱和后继是两个对称的操作。

### 前驱

即二叉树中比这个节点小的节点中，最大的那一个。要得到二叉树中一个节点的前驱，我们还需要两个辅助函数（如果用递归）。这里要分两种情况：

当节点的左孩子存在时，前驱就是左子树中拥有最大值的节点（最右节点）；

当节点的左孩子不存在时，前驱应该为该节点有右孩子的最底层祖先。

```c++
TreeNode* treeMax(TreeNode* node){
    if(node == NULL)
        return NULL
    TreeNode* temp = node;
    while(temp->right != NULL)
        temp = temp->right;
    return temp;
}

TreeNode* leftAncestor(TreeNode* node){
    if(node == NULL)
        return NULL;
    if(node == node->parent->left)
        return leftAncestor(node->parent);
    else
        return node;
}

TreeNode* predecessor(TreeNode* node){
    if(node == NULL)
        return NULL;
    if(node->left != NULL)
        return treeMax(node->left);
    else
        return leftAncestor(node);
}

// 循环写法
TreeNode* predecessor_iter(TreeNode* node){
    if(node == NULL)
        return NULL;
    if(node->left != NULL)
        return treeMax(node->left);
    TreeNode* p = node->parent;
    TreeNode* temp = node;
    while(p != NULL && temp == p->left){
        temp = p;
        p = p->parent;
    }
    return p;
}
```

### 后继

即二叉树中比这个节点大的节点中，最小的那一个。要得到二叉树中一个节点的后继，我们还需要两个辅助函数（如果用递归）。这里要分两种情况:

当节点的右孩子存在时，后继就是右子树中值最小的节点（最左节点）；

当节点的右孩子不存在时，后继应该为该节点有左孩子的最底层祖先。

```c++
TreeNode* treeMin(TreeNode* node){
    if(node == NULL)
        return NULL;
    TreeNode* temp = node;
    while(temp->left != NULL){
        temp = temp->left;
    }
    return temp;
}

TreeNode* rightAncestor(TreeNode* node){
    if(node->parent == NULL)
        return NULL;
    //if(node->val < node->parent->val)
    if(node->parent->left == node) 
        return node->parent;
    else
        return rightAncestor(node->parent);
}

TreeNode* successor(TreeNode* node){
    if(node == NULL)
        return node;
    if(node->right != NULL)
        return treeMin(node->right);
    else
        return rightAncestor(node);
}

// 循环写法
TreeNode* successor_iter(TreeNode* node){
    if(node == NULL)
        return node;
    if(node->right != NULL)
        return treeMin(node);
    TreeNode* p = node->parent;
    TreeNode* temp = node;
    while(p != NULL && temp == p->right){
        temp = p;
        p = p->parent;
    }
    return p;
}
```

## 增删查

### 搜索节点（search）

```c++
TreeNode* search(TreeNode* root, int val){
    if(root == NULL || root->val == val)
        return root;
    if(root->val > val)
        return search(root->left);
    else
        return search(root->right);
}

TreeNode* search_iter(TreeNode* root, int val){
    TreeNode* temp = root;
    while(temp != NULL && temp->val != val){
        if(temp->val > val)
            temp = temp->left;
        else
            temp = temp->right;
    }
    return tmep;
}
```

### 插入节点（insert）

插入节点需要先进行搜索，换言之，我们总是在树中的叶子节点上插入新的节点，而不会在其他节点与节点之间插入。另外还需要考虑树为空的情况。时间复杂度为O(h)。

```c++
void BSTinsert(TreeNode* root, int val){
    TreeNode* node = new TreeNode(val);
    TreeNode* temp = root;
    TreeNode* y;
    while(temp){
        // 记录搜索完毕后的父亲节点
        y = temp;
        if(val > temp->val)
            temp = temp->right;
        else
            temp = temp->left;
    }
    node->parent = y;
    // 判断y是否为空，如果为空则设为根节点
    if(y == NULL)
        root = node;
    else if(val > y->val)
        y->right = node;
    else
        y->left = node;
}
```

### 删除节点（delete）

删除节点比较麻烦，因为一个节点删除后需要别的节点补充它的位置，又要保证操作完成后不违背BST的性质。

为了让代码更易读，我们引入一个新函数`transplant`，用来将一颗子树`v`替换另一颗子树`u`，这意味着节点`u`和它的左右子树都将被抛弃。本质上只修改了两个指针，将被删除的`u`的父亲节点的`left`或`right`，以及嫁接节点`v`的`parent`。

```c++
void transplant(TreeNode* root, TreeNode* u, TreeNode* v){
    if(u->parent == NULL)
        root = v;
    else if(u == u->parent->left)
        u->parent->left = v;
    else
        u->parent->right = v;
    if(v != NULL)
        v->parent = u->parent;
}
```

有了`transplant`函数之后，就可以写删除节点的代码了，要删除节点`del`，我们要分三种情况：

* 1.`del`没有孩子节点，那么将它删除，并用`NULL`替换它就行。
* 2.`del`只有一个孩子，那么只需要将这个孩子节点替换掉`del`就行。
* 3.`del`有两个孩子节点，这样的情况要麻烦一些，要先找到`del`的后继（此时后继必定在节点的右子数中），然后用后继代替`del`的位置，并将原先的左右孩子节点重新接上新节点，同时，如果这里`del`的后继直接是它的右节点，那情况又不一样，需要另做讨论。
    如果后继`y`是`del`的右孩子，说明它没有左孩子，此时直接将`y`嫁接到`del`，然后将`del`原先的左孩子给到`y`，就行了
    如果后继`y`不是`del`的右孩子，我们还需要先将`y`的右孩子替换`y`，然后将`y`和`del`的右孩子连接，最后嫁接`del`。

```c++
void BSTdelete(TreeNode* root, TreeNode* del){
    if(root == NULL || del == NULL)
        return;
    if(del->left == NULL)
        transplant(root, del, del->right);
    else if(del->right = NULL)
        transplant(root, del, del->left);
    else{
        TreeNode* y = successor(del);
        if(y->parent != del){
            transplant(root, y, y->right);
            y->right = del->right;
            y->right->parent = y;
        }
        transplant(root, del, y);
        y->left = del->left;
        y->left->parent = y;
    }
}
```

## 旋转节点（rotation）

旋转二叉树的节点是一个非常重要的操作，经常被用来在不改变二叉树的遍历结果下，调整它的结构来保证平衡。本质上是修改三个节点的六个指针。

```c++
void rotate_left(TreeNode* node){
    if(node == NULL)
        return;
    TreeNode* y = node->right;
    node->right = y->left;
    if(y->left)
        y->left->parent = node;
    y->parent = node->parent;
    
    if(node->parent == NULL)
    else if(node == node->parent->left)
        node->parent->left = y;
    else
        node->parent->right = y;
    y->left = node;
    node->parent = y;
}

void rotate_right(TreeNode* node){
    if(node == NULL)
        return;
    TreeNode* y = node->left;
    node->left = y->right;
    if(y->right)
        y->right->parent = node;
    y->parent = node->parent;

    if(node->parent == NULL)
    else if(node == node->parent->left)
        node->parent->left = y;
    else
        node->parent->right = y;
    y->right = node;
    node->parent = y;
}
```