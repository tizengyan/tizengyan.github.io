---
layout: post
title:  "字符串的常见套路"
date:   2019-02-25 10:10:54
categories: 考点
tags: string
excerpt: 整理一下字符串的一些套路  
author: Tizeng
---

* content
{:toc}

## 1.char*和char[]的区别

一般来讲，我们有两种方式声明一个字符数组：

```c++
char* s1 = "Hello";
char s2[10] = "Hello";
```

这两种写法的区别是，`s1`是一个指针变量（4 bytes），它被存在栈中指向我们初始化的字符串，而此时字符串在我们的代码内存中，因此我们无法通过下标对任何字符做修改，因此这里编译器会建议我们在前面加上`const`，但是可以让这个指针指向别的字符串。`s2`声明为一个数组（10 bytes），和其他类型的数组如整形数组并无区别。

## 2.转换成字符数组

很多函数需要我们输入的是一个`char*`的字符数组而非`string`，这时我们就需要将二者相互转换的方法。

### string转char*

第一种可行的方法是使用`c_str()`，它会返回一个`const char*`。

```c++
std::string str = "string";
const char* cstr = str.c_str();
```

第二种是使用`data()`，并将结果转换为`char*`：

```c++
string str = "abc";
char* p = (char*)str.data();
```

### char*转string

如果我们需要将`char*`转换成`string`，可以利用`string`的构造函数：

```c++
const char *s = "Hello, World!";
string str(s);
```

## 3.字符串比较

这是很常规的一个操作，使用`strcmp`函数实现，它依次比较两个字符串的字符，返回值有三种可能：

* `<0`: the first character that does not match has a lower value in ptr1 than in ptr2

* `0`: the contents of both strings are equal

* `>0`: the first character that does not match has a greater value in ptr1 than in ptr2

注意这里输入的应该是`char*`。例如：

```c++
bool comp(Info info1, Info info2) {
    // 先按分数排序
    if (info1.score != info2.score)
        return info1.score > info2.score;
    char* a = (char*)info1.name.data();
    char* b = (char*)info2.name.data();
    // 再按名字字母顺序排序
    return strcmp(a, b);
}
```

如果我们需要比较两个`string`类型的字符串，可以用`string::compare`：

```c++
str1.compare(str2)
```

类似的，如果它们完全相同返回`0`，第一个不相同的字符值比第二个大返回`>0`，反之小于零。如果需要，也可以输入参数`pos`和`len`比较子串。

## 4.子串（string）

使用`std::string::substr`在字符串中提取子串，该函数定义为`string substr(size_t pos, size_t len) const;`。下面是用法：

```c++
int main () {
    std::string str="We think in generalities, but we live in details."; // (quoting Alfred N. Whitehead)
    std::string str2 = str.substr (3,5);     // "think"
    std::size_t pos = str.find("live");      // position of "live" in str
    std::string str3 = str.substr (pos);     // get from "live" to the end
    std::cout << str2 << ' ' << str3 << '\n';
    return 0;
}
```

输出：

> think live in details.

## 5.反转

顾名思义，将输入的字符串完全反向，然后输出。如`abcd`输出`dcba`。

要实现这个并不难，只需要循环的交换头尾元素即可。

```c++
void reverseString(vector<char>& s) {
    int i = 0, j = s.size() - 1;
    if(s.size() == 0)
        return;
    while(i < j){
        swap(s[i], s[j]);
        i++;
        j--;
    }
}
```

### 反转一个整数

#### 转换成字符串实现

我们可以用翻转字符串的思路来实现，先将输入的整数转换成字符串，用翻转字符串的方法翻转过后再转换回整数。
整数转换成字符串可以直接用`to_string`函数，当然还有其他方法：

使用`itoa()`：

```c++
int a = 10;
char *intStr = itoa(a);
string str = string(intStr);
```

或者使用字符串流：

```c++
int a = 10;
stringstream ss;
ss << a;
string str = ss.str();
```

#### 直接对整数操作实现

思路很简单，初始化`sum = 0`，通过对输入的整数对10取余得到个位数，加上`sum * 10`，然后除以10抹掉最后一位，重复上述操作，直到取到0为止。

```c++
int reverse(int x) {
    long long temp = (long long)x;
    if(x < 0)
        temp = -1 * temp;
    long long res = 0;
    while(temp){
        long long dig = temp % 10;
        res = res * 10 + dig;
        temp /= 10;
    }
    // 如果题目要求答案范围不超出+-2^31
    if(res > pow(2,31) || res < -pow(2, 31) + 1)
        return 0;
    return x > 0 ? res : res * -1;
}
```

## 6.删去字符串中重复的字符

```c++
string s;
cin >> s;
int start = 0;
for (int i = 1; i < s.size(); i++) {
    if (s[i] == s[i - 1]) {
        start++;
    }
    else {
        s[i - start] = s[i];
    }
}
cout << s << endl;
```

## 7.旋转字符串（或数组）

旋转分左旋和右旋，说简单点就是左移和右移，每次移动最后一位给到开头补齐，要完成这个操作，至少有三种方法。

### 方法1

通过三次反转完成。以右移为例，如果如果要把 1234567 右移三位，会得到 5671234 ，如果我们直接对原数（或字符串）翻转，会得到 7654321，可以看到前三位和后四位的元素已经和右移的结果一样，只是顺序反了，这时我们只需要分别翻转前三位和后四位，就可以得到右移的结果。因此我们需要对普通的翻转函数进行修改。让它可以只翻转字符串中的一部分：

```c++
void reverse(string& s, int left, int right){
    if(left < 0 || right >= s.size())
        return;
    while(left < right){
        swap(s[left], s[right]);
        left++;
        right--;
    }
}

void rotate_right(string& s, int k){
    int n = s.size();
    if(n <= 1)
        return;
    k %= n; // 保证 k 比 n 小
    reverse(s, 0, n - 1);
    reverse(s, 0, k - 1);
    reverse(s, k, n - 1);
}

void rotate_left(string& s, int k){
    int n = s.size();
    if(n <= 1)
        return;
    k %= n; // 保证 k 比 n 小
    reverse(s, 0, n - 1);
    reverse(s, 0, n - k - 1);
    reverse(s, n - k, n - 1);
}
```

可以看到上面的左移函数其实能用右移函数直接表示，因此改写为：

```c++
void rotate_left(string& s, int k){
    rotate_right(s, s.size() - k);
}
```

这种方法的时间复杂度是O(n)，消耗常数的空间。

### 方法2

声明一个新的数组，然后按左右移动的顺序依次改写得到答案，这样做时间和空间复杂度都是O(n)。这里可以用求余的性质：

```c++
nums[(i + k)%n] = numsCopy[i];
```

### 方法3

将数组中前 k 位和后 k 位的元素调换位置，如果是要右移的话，那么前 k 个元素已经就位，剩下的需要把右边剩下的元素向右移三位，我们可以在子数组中重复刚才的操作，再次交换，直到只剩下一个元素，这个过程很明显可以用递归实现，但是需要小心其中的细节：

```c++
void rotate(vector<int>& nums, int k) {
    if(k == 0 || nums.size() == 1)
        return;
    int n = nums.size();
    k = k % n;
    rotate_recursion(nums, 0, n - 1, k);
}

void rotate_recursion(vector<int> &nums, int l, int r, int k){
    if(k == 0)
        return;
    int n = nums.size();
    k = k % (r - l + 1);
    if(l > n - 1)
        return;
    int left = l;
    for(int i = 0; i < k; i++){
        if(l > n - 1 || r - k + 1 < left)
            break;
        swap(nums[l++], nums[r++ - k + 1]);
    }
    rotate_recursion(nums, left + k, n - 1, k);
}
```

## 8.翻转句子中单词的顺序

即将输入的一句英文中的单词顺序反向后输出，例如`I am a student.`翻转之后变成`student. a am I`。

```c++

```

## 9.字符串中的最长重复子串

