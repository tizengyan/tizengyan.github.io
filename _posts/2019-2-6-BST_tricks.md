---
layout: post
title:  "二叉搜索树的一些常见套路"
date:   2019-02-06 20:34:54
categories: 数据结构
tags: BST 递归
excerpt: 层次遍历、反转（镜像）、合并、双向链表、子结构、深度
author: Tizeng
---

* content
{:toc}

## 1.层次遍历

层次遍历是指从上到下对每一层，从左到右依次输出节点的值（如果存在）。这种遍历方式有别于前面三种常用的遍历，但同样非常重要。常用的方法是利用队列，先将根节点加入队列，然后将根节点的左右孩子节点（如果存在）加入队列，弹出根节点并遍历，之后对队列中的所有节点做相同的操作，即可得到我们想要的结果。

代码实现如下：

```c++
void levelTraversal(TreeNode* root){
    if(root == NULL)
        return;
    queue<TreeNode* > q;
    q.push(root);
    while(!q.empty()){
        int size = q.size();
        for(int i = 0; i < size; i++){
            TreeNode* temp = q.front();
            q.pop();
            cout << temp->val << endl;
            if(temp->left)
                q.push(temp->left);
            if(temp->right)
                q.push(temp->right);
        }
    }
}
```

层次遍历也有用递归实现的方法，但是不太常见。而且这样写有一个限制，最后返回的是一个二维数组，其中每一行储存了二叉树一层所有节点的值。写法比较巧妙，不容易想到。

```c++
vector<vector<int> > res;

vector<vector<int>> levelOrderBottom(TreeNode* root) {
    DFS(root, 0);
    //reverse(res.begin(), res.end());
    return res;
}

void DFS(TreeNode* root, int level){
    if(root == NULL)
        return;
    if(res.size() == level)
        res.push_back(vector<int>());
    res[level].push_back(root->val);
    DFS(root->left, level + 1);
    DFS(root->right, level + 1);
}
```

## 2.二叉树的反转（镜像）

要求反转一颗二叉树，用递归可以以非常短的代码实现，但是前提是对递归和二叉树有基本的理解。

```c++
void reverseBST(TreeNode* root){
    if(root == NULL)
        return;
    TreeNode* temp = root->left;
    root->left = root->right;
    root->right = temp;
    reverseBST(root->left);
    reverseBST(root->right);
}
```

另一种思路是用循环，这里需要用上面讲到的二叉树的层次遍历。由于这里不需要像层次遍历那样每层从左致右依次输出节点，只要保证每个节点都遍历到了就可以，因此用栈和队列都可以，但是这里个人习惯用队列来做，因为比较符合层次遍历的逻辑。

```c++
void Mirror(TreeNode *root) {
    if(root == NULL)
        return;
    queue<TreeNode* > q;
    q.push(root);
    while(!q.empty()){
        int size = q.size();
        for(int i = 0; i < size; i++){
            TreeNode* temp = q.front();
            q.pop();
            if(temp->left)
                q.push(temp->left);
            if(temp->right)
                q.push(temp->right);
            TreeNode* temp1 = temp->left;
            temp->left = temp->right;
            temp->right = temp1;
        }
    }
}
```

## 3.树的深度

题目描述输入一棵二叉树，求该树的深度。从根结点到叶结点依次经过的**结点**（含根、叶结点）形成树的一条路径，最长路径的长度为树的深度（根节点深度为1，注意深度有时定义会有差别，要视情况而定）。

这题本质考察对递归的理解，我们设想一种基本情况，即只有一个节点的树，它的深度是1，那么可以推断出一棵树的深度就是自己左右子树中深度最大的那颗树的深度加一，从而写出递归代码。

```c++
int treeDepth(TreeNode* root){
    if(root == NULL)
        return 0;
    if(root->left == NULL && root->right == NULL)
        return 1;
    return 1 + max(treeDepth(root->left), treeDepth(root->right));
}
```

### 扩展题：判断一棵二叉树是否平衡

题目描述：输入一棵二叉树，判断该二叉树是否是平衡二叉树。

回顾平衡二叉树的概念：所有节点的左右子树的高度相差不超过1的二叉树。这里树的高度说的其实就是根节点的深度，因此根据定义，我们很容易写出如下代码：

```c++
bool IsBalanced_Solution(TreeNode* root) {
    if(root == NULL)
        return true;
    bool res = false;
    if(abs(treeDepth(root->left) - treeDepth(root->right)) <= 1)
        res = true;
    return IsBalanced_Solution(root->left) && IsBalanced_Solution(root->right) && res;
}
```

但是这种思路并不高效，重复计算了很多节点的信息，类似于用递归求斐波那契数列，当树体积庞大时会严重影响效率，我们需要更快的算法。
如果能从低端的叶子节点开始后序遍历，一边遍历一边累积深度，如果出现两边子树深度差超过1，则直接返回`false`，直到最后遍历到根节点。
这种思路对递归的理解有一定要求，否则就算告诉你这样做，也很难写出实际的代码（比如我）。我们回想一下之前普通的计算树的深度的函数，其实已经在做我们需要的操作了，只是累积深度时没有进行比较，我们可以把这部分内容加到这个函数里面。

```c++
bool IsBalanced_Solution(TreeNode* root) {
    return isBalanced(root) != -1;
}

int isBalanced(TreeNode* root){
    if(root == NULL)
        return 0;
    int left = isBalanced(root->left);
    int right = isBalanced(root->right);
    if(abs(left - right) > 1 || left == -1 || right == -1)
        return -1;
    return 1 + max(left, right);
}
```

`isBalanced`函数的本质其实就是返回树的深度，只是同时会检查树的平衡性，一旦有节点的左右子树不满足平衡条件，就返回-1，base case为遍历到`NULL`时，返回深度0。我们也可以设置一个`bool res`的全局变量，初始化为`true`，计算深度时如果有节点不满足平衡条件，则设`res`为`false`。

## 4.合并两颗二叉树

即将两颗二叉树重叠后相加，都有节点的地方将节点的值相加，如果只有其中一颗树有节点则直接替换掉另一颗树的NULL。这个问题我知道的有两个思路，一是先层次遍历两颗树，将结果存入二维数组，并将NIL节点以0存入，这样可以保证两个二维数组的维度相同，然后将两个二维数组相加，再将其复原成一颗新的二叉树；另一个思路是利用递归直接对两颗二叉树的节点进行操作，创造出一颗新的二叉树。这里我们用第二种方法，代码要简单许多。

思路很简单，如果重叠时两棵树的节点`root1`和`root2`都存在，就新建一个值为两个节点值的和的新节点，并将新节点的左孩子设为递归函数输入`root->left`、`root2->left`的返回结果，将右节点设为递归函数输入`root1->right`、`root2->right`的返回结果。

```c++
TreeNode* mergeTrees(TreeNode* root1, TreeNode* root2){
    if(!root1)
        return root2;
    else if(!root2)
        return root1;
    TreeNode* node = new TreeNode(root1->val + root2->val);
    node->left = mergeTrees(root1->left, root2->left);
    node->right = mergeTrees(root1->right, root2->right);
    return node;
}
```

还有另一种更加简洁的写法，思路是一样的：

```c++
TreeNode* mergeTrees(TreeNode* t1, TreeNode* t2) {
    if(t1 != NULL && t2 != NULL){
        TreeNode* root = new TreeNode(t1->val + t2->val);
        root->left = mergeTrees(t1->left, t2->left);
        root->right = mergeTrees(t1->right, t2->right);
        return root;
    }
    else{
        return t1 ? t1 : t2;
    }
}
```

## 5.二叉树与双向链表

输入一棵二叉搜索树，将该二叉搜索树转换成一个排序的双向链表。要求不能创建任何新的结点，只能调整树中结点指针的指向。

我们要利用二叉搜索树的基本性质来做这道题，如果要将一个BST转化成一个双向链表，则每个节点的`left`指针应该指向该节点的前驱，`right`指针指向该节点的后继。
然后对它的左右子树做相同的操作，是一个典型的递归过程。
由于将一个BST进行**中序遍历**后，一定会得到一个排好序的序列，我们可以利用这个性质，对中序遍历的函数加以修改，用一个`pre`指针记录前一个节点，然后将前一个节点（如果存在）的`right`指向当前节点，将当前节点的`left`指向`pre`。这里要注意传入函数的`pre`指针需要传入引用，主函数调用时也要初始化一个变量，不能直接以`NULL`作为参数调用，否则会报错。
调用完毕后，还需要根据根节点找到双向链表的头结点，只需要一直往左查找即可。

下面是实现代码：

```c++
TreeNode* Convert(TreeNode* root){
    if(root == NULL)
        return NULL;
    TreeNode* pre = nullptr;
    convert(root, pre);
    TreeNode* temp = root;
    while(temp->left)
        temp = temp->left;
    return temp;
}

void convert(TreeNode* cur, TreeNode*& pre){
    if(cur == NULL)
        return;
    convert(cur->left, pre);
    cur->left = pre;
    if(pre)
        pre->right = cur;
    pre = cur;
    convert(cur->right, pre);
}
```

这里`convert`函数的第二个参数要传入指针的引用，还不是很清楚为什么。

## 6.树的子结构

输入两棵二叉树A，B，判断B是不是A的子结构。（我们约定空树不是任意一个树的子结构）

这道题本质上还是考察二叉树的遍历，只要思路清晰，莫慌，不难写出答案。我们需要在A中找寻是否存在B的子结构，分两步：先从B根节点的值开始，搜索A中是否存在和它相等的节点，如果找到，那么再在这个子树中依次比较每个节点，看其他值是否也相等。

注意这里如果第一步用循环，会面临当找到第一个节点的值和B的根节点值相等时，无法判断接下来是往左搜索还是往右搜索的情况，因此我选择用递归。

在第二步依次检查节点时，有两点要注意：1.优先讨论当输入的两个指针为`NULL`的情况，再比较它们的值；2.优先检查`root2`是否为`NULL`，如果是，则说明一路查下来没有发现值不一样的节点，已经到达了叶子节点，返回`true`，之后再检查`root1`，如果为`NULL`，则说明A的这个子树已经没有节点可以继续遍历，而B还没有遍历完，应该返回`false`。如果先检查`root1`，逻辑会出错。

```c++
bool HasSubtree(TreeNode* root1, TreeNode* root2){
    if(root1 == NULL || root2 == NULL)
        return false;
    bool res = false;
    if(root1->val == root2->val)
        res = check(root1, root2);
    if(!res)
        res = HasSubtree(root1->left, root2);
    if(!res)
        res = HasSubtree(root1->right, root2);
    return res;
}

bool check(TreeNode* root1, TreeNode* root2){
    if(root2 == NULL)
        return true;
    if(root1 == NULL)
        return false;
    if(root1->val != root2->val)
        return false;
    return check(root1->left, root2->left) && check(root1->right, root2->right);
}
```

## 7.重建二叉树

题目描述：输入某二叉树的前序遍历和中序遍历的结果，请重建出该二叉树。假设输入的前序遍历和中序遍历的结果中都不含重复的数字。例如输入前序遍历序列{1,2,4,7,3,5,6,8}和中序遍历序列{4,7,2,1,5,3,8,6}，则重建二叉树并返回。

这题考察对三种基本遍历的应用，利用遍历的结果反推二叉树的结构。先由前序遍历得到根节点，然后根据中序遍历的结果推测出左子树和右子树，以此类推，利用递归的特点把左右子树接上。

```c++
TreeNode* reConstructBinaryTree(vector<int> pre,vector<int> vin) {
    if(pre.size() == 0 || vin.size() == 0)
        return NULL;
    return construct(pre, vin, 0, pre.size() - 1, 0, vin.size() - 1);
}

TreeNode* construct(vector<int>& pre, vector<int>& inorder, int l1, int r1, int l2, int r2){
    if(l1 > r1 || l2 > r2)
        return NULL;
    TreeNode* root = new TreeNode(pre[l1]);
    int count = 0;
    for(int i = l2; i <= r2; i++){
        if(inorder[i] != pre[l1])
            count++;
        else
            break;
    }
    root->left = construct(pre, inorder, l1 + 1, l1 + count, l2, l2 + count - 1);
    root->right = construct(pre, inorder, l1 + 1 + count, r1, l2 + count + 1, r2);
    return root;
}
```

## 8.二叉树中和为某一值的路径（[LeetCode 113](https://leetcode.com/problems/path-sum-ii/)）

输入一颗二叉树的跟节点和一个整数，打印出二叉树中结点值的和为输入整数的所有路径。路径定义为从树的根结点开始往下一直到叶结点所经过的结点形成一条路径。(注意: 在返回值的list中，数组长度大的数组靠前)

这题如果充分利用递归的性质，代码可以很简洁，首先base case是遍历到`NULL`时，直接返回，而只要节点不为`NULL`，就将此时的`n`减去其值，并将值存入`path`，然后判断有没有到达根节点且`n`刚好为0，如果是，则说明这是一条满足条件的路径，把它加入`ans`，如果不是，有两种情况，一是已经到达叶子节点，但是`n`不为0，另一种是还未到达叶子节点（无论`n`是否为0），这两种情况我们都继续遍历当前节点的左右子树，完毕后将之前推入`path`的值弹出，回到上一层继续遍历。

```c++
vector<vector<int>> pathSum(TreeNode* root, int sum) {
    vector<vector<int> > res;
    vector<int> path;
    findPath(root, sum, res, path);
    return res;
}

void findPath(TreeNode* node, int sum, vector<vector<int> >& res, vector<int>& path){
    if(node == NULL)
        return;
    path.push_back(node->val);
    sum -= node->val;
    if(sum == 0 && node->left == NULL && node->right == NULL)
        res.push_back(path);
    findPath(node->left, sum, res, path);
    findPath(node->right, sum, res, path);
    path.pop_back();
}
```

这道题还有一些变种，如[LeetCode 437](https://leetcode.com/problems/path-sum-iii/)，这题同样是找路径和为某一值的数量，但现在规定路径不一定要从根节点开始也不需要在叶子节点结束，且要求的是满足和为输入`sum`的路径数量。这里有两层递归，第一层是从上至下遍历二叉树的所有节点（DFS），然后再对每一个节点下面的所有路径做检查：

```c++
int pathSum(TreeNode* root, int sum) {
    if(root == NULL)
        return 0;
    return findPath(root, sum) + pathSum(root->left, sum) + pathSum(root->right, sum);
}

int findPath(TreeNode* node, int sum){
    int res=0;
    if(node == NULL)
        return 0;
    if(sum == node->val)
        res++;
    res += findPath(node->left, sum - node->val);
    res += findPath(node->right, sum - node->val);
    return res;
}
```

## 9.最低公共祖先（[LeetCode 235](https://leetcode.com/problems/lowest-common-ancestor-of-a-binary-search-tree/)）

题目描述：输入一颗二叉搜索树的根节点和两个其他节点，要求输出这两个节点的最低公共祖先。

要注意这里输入的是一颗二叉搜索树，因此要充分利用BST的性质，如果两个节点的值都比根节点小，那么应该继续往左查，反之应该继续往右查，而如果出现一个在左一个在右的情况，则说明找到了最低的公共祖先。

先来看一下循环的写法：

```c++
TreeNode* lowestCommonAncestor(TreeNode* root, TreeNode* p, TreeNode* q) {
    if(root == NULL)
        return NULL;
    TreeNode* temp = root;
    while(temp){
        if(temp->val > p->val && temp->val > q->val)
            temp = temp->left;
        else if(temp->val < p->val && temp->val < q->val)
            temp = temp->right;
        else
            return temp;
    }
    return temp;
}
```

再补充递归的写法，递归的写法通常更简洁，但是写的时候要费点脑筋（至少对我来说）：

```c++
TreeNode* lowestCommonAncestor(TreeNode* root, TreeNode* p, TreeNode* q) {
    if(root == NULL)
        return NULL;
    if(root->val < p->val && root->val < q->val)
        return lowestCommonAncestor(root->right, p, q);
    else if(root->val > p->val && root->val > q->val)
        return lowestCommonAncestor(root->left, p, q);
    return root;
}
```