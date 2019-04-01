---
layout: post
title:  "友塔游戏笔试笔记"
categories: 笔经
tags: test
excerpt: 整理一下笔试题
author: Tizeng
---

* content
{:toc}

题目一：

![yotta1](https://github.com/tizengyan/images/raw/master/yotta1.png)

这道题本质上是排序，但是并不是按普通的大小顺序，而是按输入的一串数字的顺序来决定谁大谁小，越靠前的数字就越小，而没有出现的一律比出现的小，且按自身的顺序排序。那么我们很容易想到利用`sort`函数来进行排序，剩下的问题就是如何按题目的要求定义好`comp`。

实现代码如下：

```c++
string s2;
vector<int> table(256);

bool comp(char a, char b) {
    if (table[a] == 0 && table[b] == 0)
        return a < b;
    else if (table[a] == 0 && table[b] != 0)
        return 1;
    else if (table[a] != 0 && table[b] == 0)
        return 0;
    else
        return table[a] < table[b];
}

void mySort(string& s1) {
    int cnt = 1;
    for (int i = 0; i < s2.length(); i++) {
        table[s2[i]] = cnt;
        cnt++;
    }
    sort(s1.begin(), s1.end(), comp);
}

int main() {
    string s1;
    cin >> s1 >> s2;
    if (s2.length() == 0) {
        sort(s1.begin(), s1.end(), comp);
    }
    else {
        mySort(s1);
    }
    cout << s1 << endl;
    return 0;
}
```

题目二：

![yotta2](https://github.com/tizengyan/images/raw/master/yotta2.png)

我们首先用一个结构体来存每个学生的信息，并利用`sort`先按成绩，后按姓名字母顺序给学生排序，然后就可以开始根据每个专业的录取人数记录录取信息了。这道题的难点在于当第一第二志愿人满时，如何正确的给报了相关志愿学生的分数做级差

```c++
struct Info {
    string name;
    int score;
    int h1, h2, h3;
    Info(string n, int s, int h1_, int h2_, int h3_) { name = n; score = s; h1 = h1_; h2 = h2_; h3 = h3_; }
};

vector<Info> students;
vector<int> k;
int m;

void init() {
    int s, n;
    cin >> s >> m >> n;
    for (int i = 0; i < s; i++) {
        string name;
        int score, h1, h2, h3;
        cin >> name >> score >> h1 >> h2 >> h3;
        Info info(name, score, h1, h2, h3);
        students.push_back(info);
        //cout << students[i].name << ' ' << students[i].score << endl;
    }
    for (int i = 0; i < n; i++) {
        int input;
        cin >> input;
        k.push_back(input);
        //cout << k[i] << ' ';
    }
    //cout << endl;
}

bool comp1(Info info1, Info info2) {
    if (info1.score != info2.score)
        return info1.score > info2.score;
    char* a = (char*)info1.name.data();
    char* b = (char*)info2.name.data();
    return strcmp(a, b);
}

void solve1() {
    sort(students.begin(), students.end(), comp1);
    vector<int> table(k.size(), 0);
    bool flag1 = 1, flag2 = 1;
    //for (size_t i = 0; i < students.size(); i++) {
    while(!students.empty()){
        cout << students[0].name << ',';
        if (table[students[0].h1] < k[students[0].h1]) {
            cout << students[0].h1 << endl;
            table[students[0].h1]++;
        }
        else if (table[students[0].h2] < k[students[0].h2]) {
            if (flag1){
                students[0].score = students[0].score - m;
                sort(students.begin(), students.end(), comp1);
                flag1 = 0;
                continue;
            }
            flag1 = 1;
            cout << students[0].h2 << endl;
            table[students[0].h2]++;
        }
        else if(table[students[0].h3] < k[students[0].h3]) {
            if (flag2) {
                students[0].score = students[0].score - m;
                sort(students.begin(), students.end(), comp1);
                flag2 = 0;
                continue;
            }
            flag2 = 1;
            cout << students[0].h3 << endl;
            table[students[0].h3]++;
        }
        else
            cout << "failed" << endl;
        students.erase(students.begin());
    }
}

int main() {
    init();
    solve1();
    system("pause");
    return 0;
}
```

题目三：

![yotta3](https://github.com/tizengyan/images/raw/master/yotta3.png)

这道题的输入是一排英雄，这些英雄有两种状态，即觉醒和不觉醒，分别有不同的价值（战斗力）和费用（能量消耗），那么意味着我如果选了其中一个英雄的觉醒状态就不能选非觉醒状态，反之亦然，也就是说这两种状态互斥，那么我们可以把它看成一个分组的背包问题，每个英雄的两种状态自成一组，一共有输入的英雄数量的组，然后用分组的背包问题的思路去解决：

```c++
#include <iostream>
#include <fstream>
#include <vector>

using namespace std;

vector<int> w1, w2, v1, v2;
int n, C;

void init() {
    ifstream infile;
    infile.open("input.txt");
    int w, val;
    infile >> n >> C;
    cout << "n = " << n << ", C = " << C << endl;
    int i = 0;
    while (i < n) {
        infile >> w >> val;
        w1.push_back(w);
        v1.push_back(val);
        infile >> w >> val;
        w2.push_back(w + w1[i]);
        v2.push_back(val + v1[i++]);
    }
    infile.close();
}

int solve() {
    vector<int> f(C + 1);
    for (int i = 0; i < n; i++) { // 组的数目就是输入英雄的数目
        for (int j = C; j >= 0; j--) {
            if (j >= w1[i]) // 未觉醒组
                f[j] = max(f[j], f[j - w1[i]] + v1[i]);
            if (j >= w2[i]) // 觉醒组
                f[j] = max(f[j], f[j - w2[i]] + v2[i]);
        }
    }
    return f[C];
}

int main() {
    init();
    cout << solve() << endl;
    system("pause");
    return 0;
}
```