---
title: "K-Means 聚类算法"
date: 2015-06-16T10:17:10+08:00
lastmod: 2015-06-16T10:17:10+08:00
draft: false
keywords: [K-means, 聚类, 数据分析]
tags: [K-means, 聚类, 数据分析]
categories: [数据分析]
mathjax: true
mathjaxEnableSingleDollar: false
mathjaxEnableAutoNumber: false
hideHeaderAndFooter: false
comments: true
toc: true
---

>**聚类**是在给定的数据集合中寻找同类的数据子集合，每一个子集合形成一个类簇，同类簇中的数据具有更大的相似性。聚类算法大体上可分为**基于划分**的方法、**基于层次**的方法、**基于密度**的方法、**基于网格**的方法以及**基于模型**的方法。

## 基本概念
术语 **k-means** 最早是由 James MacQueen 在 1967 年提出的，这一观点可以追溯到 1957 年 Hugo Steinhaus 所提出的想法。1957 年，斯图亚特·劳埃德最先提出这一标准算法，当初是作为一门应用于脉码调制的技术,直到 1982 年，这一算法才在贝尔实验室被正式提出。1965 年，E.W.Forgy 发表了一个本质上是相同的方法，1975 年和 1979 年，Hartigan 和 Wong 分别提出了一个更高效的版本。
<!--more-->
目前 k-means 算法是一种得到最广泛使用的**基于划分的聚类算法**，把 n 个对象分为 k 个簇，以使簇内具有较高的相似度。相似度的计算根据一个簇中对象的平均值来进行。它与处理混合正态分布的最大期望算法很相似，因为他们都试图找到**数据中自然聚类的中心**。

算法首先随机地选择 k 个对象，每个对象初始地代表了一个簇的平均值或中心。对剩余的每个对象根据其与各个簇中心的距离，将它赋给最近的簇，然后重新计算每个簇的平均值。这个过程不断重复，直到准则函数收敛。

它假设对象属性来自于空间向量，并且目标是使各个群组内部的均方误差总和最小。假设有 k 个群组 Si, i=1,2,...,k。μi 是群组 Si 内所有元素 xj 的重心，或叫中心点。

K 值的选择应该在 2 和 √n 之间，其中 n 为数据个数。

## K-means算法的性能分析

### 优点
1. k-平均算法是解决聚类问题的一种经典算法，算法**简单、快速**。
2. 对处理大数据集，该算法是相对可伸缩的和高效率的，因为它的复杂度大约是 O(nkt)，其中 n 是所有对象的数目，k 是簇的数目,t 是迭代的次数。通常 k 远小于 n。这个算法经常以**局部最优**结束。
3. 算法尝试找出使平方误差函数值最小的 k 个划分。当簇是密集的、球状或团状的，而簇与簇之间区别明显时，它的聚类效果很好。

### 缺点
1. k-means 只有在簇的平均值被定义的情况下才能使用，不适用于某些应用，如涉及有分类属性的数据不适用。
2. 要求用户必须事先给出要生成的簇的数目 k。
3. 对初值敏感，对于不同的初始值，可能会导致不同的聚类结果。
4. 不适合于发现非凸面形状的簇，或者大小差别很大的簇。
5. 对于"噪声"和孤立点数据敏感，少量的该类数据能够对平均值产生极大影响。


## K-means 算法基本步骤
1. 从 n 个数据对象任意选择 k 个对象作为初始聚类中心；
2. 计算每个对象与这些中心对象的距离；并根据最小距离对相应对象进行划分；
3. 重新计算每个中心对象；
4. 重复 2、3 直至满足准则收敛函数。


## 鸢尾花数据集测试实例
```c
#include<stdio.h>
#include<stdlib.h>
#include<time.h>
#include<string.h>
#include<math.h>

// 鸢尾花数据集的定义
typedef struct
{
    float x ;   // 花萼长度
    float y ;   // 花萼宽度
    float i ;   // 花瓣长度
    float j ;   // 花瓣宽度
    int type ;  // 属种
    /**
        1--Iris-setosa          0---第一类
        2--Iris-versicolor      1---第二类
        3--Iris-virginica       2---第三类
    */
} Iris ;

Iris *iris ;      // 保存数据
Iris *irisCopy ;  // 数据副本


// 文件操作，读取鸢尾花数据集，有3组数据，每组 50 个
void ReadData()
{
    FILE *fp ;  // 文件指针
    char s[50],t[20] ;
    int i = 0 ;
    iris = (Iris *)malloc(150*sizeof(Iris)) ;
    fp = fopen("iris.data","r") ;
    // 读取文件内容，保存进iris
    while(fgets(s,50,fp)!=NULL){
        iris[i].x = atof(s) ;
        iris[i].y = atof(&s[4]) ;
        iris[i].i = atof(&s[8]) ;
        iris[i].j = atof(&s[12]) ;
        if(i<50){
            iris[i].type = 1 ;
        }
        else if(i<100){
            iris[i].type = 2 ;
        }
        else{
            iris[i].type = 3 ;
        }
        //printf("%d,%f,%f,%f,%f,%d\n",i,iris[i].x,iris[i].y,iris[i].i,iris[i].j,iris[i].type) ;
        i++ ;
    }
}

// k 个随机质心的生成
Iris *Random(int k,int n)
{
    int i,j,a ;
    int r[k] ;      // 保存随机数
    Iris *center ;  // 保存质心数据
    center = (Iris *)malloc(k*sizeof(Iris)) ;
    srand( (unsigned)time( NULL ) );
    // 随机生成整数，防止重复
    for(i=0;i<k;i++){
        a = rand()%n ;
        for(j=0;j<i;j++){
            if(r[j]==a){
                a = rand()%n ;
                j = 0 ;
            }
        }
        r[i] = a ;
    }
    // ReadData() ;
    // 赋值给质心
    for(i=0;i<k;i++){
        center[i].x = iris[r[i]].x ;
        center[i].y = iris[r[i]].y ;
        center[i].i = iris[r[i]].i ;
        center[i].j = iris[r[i]].j ;
        center[i].type = iris[r[i]].type ;
        // printf("%d,",r[i]) ;
        // printf("%f,%f,%f,%f,%d\n",center[i].x,center[i].y,center[i].i,center[i].j,center[i].type) ;
    }
    return center ;
}


// 创建数据集副本
void DataCopy()
{
    int i = 0;
    irisCopy = (Iris *)malloc(150*sizeof(Iris)) ;
    for(i;i<150;i++){
        irisCopy[i].x = iris[i].x ;
        irisCopy[i].y = iris[i].y ;
        irisCopy[i].i = iris[i].i ;
        irisCopy[i].j = iris[i].j ;
        irisCopy[i].type = iris[i].type ;
    }
}

// 创建质心副本
Iris *centerCopy(Iris *center,int k)
{

    int i = 0 ;
    Iris *cenCopy ;   // 质心副本
    cenCopy = (Iris *)malloc(k*sizeof(Iris)) ;
    for(i;i<k;i++){
        cenCopy[i].x = center[i].x ;
        cenCopy[i].y = center[i].y ;
        cenCopy[i].i = center[i].i ;
        cenCopy[i].j = center[i].j ;
        cenCopy[i].type = center[i].type ;
    }
    return cenCopy ;
}

// 计算质心
Iris ComputingCenter(Iris *c,int top)
{
    Iris c_ir ;  // 质点
    int m ;
    float i=0,j=0,x=0,y=0 ;
    // 求和
    for(m=0;m<top;m++){
        x = x+c[m].x ;
        y = y+c[m].y ;
        i = i+c[m].i ;
        j = j+c[m].j ;
    }
    // 给质点赋值
    c_ir.x = x/top ;
    c_ir.y = y/top ;
    c_ir.i = i/top ;
    c_ir.j = j/top ;
    return c_ir ;
}

/**********************************************************/
// 比较原质心和生成的质心是否一致
int Compare(Iris *center,Iris *centerCopy,int k)
{
    int i ;
    for(i=0;i<k;i++){
        if((centerCopy[i].x!=center[i].x)||(centerCopy[i].y!=center[i].y)||(centerCopy[i].i!=center[i].i)||(centerCopy[i].j!=center[i].j)){
            // 判断每对质心是不是变化
            return 0 ;  // 变化返回0
        }
    }
    return 1 ;  // 不变化返回1
}

/**********************************************************/
// 比较与哪个质心相近，返回与第几个质心最近
int Belong(Iris *center,Iris ir,int k)
{
    double a[k] ;  // 保存距离的平方
    int i ,min;
    for(i=0;i<k;i++){
        a[i] = (center[i].x - ir.x)*(center[i].x - ir.x) + (center[i].y - ir.y)*(center[i].y - ir.y) + (center[i].i - ir.i)*(center[i].i - ir.i) + (center[i].j - ir.j)*(center[i].j - ir.j) ;
        // 计算距离的平方
        // printf("%f\n",a[i]) ;
    }
    min = 0 ;
    for(i=0;i<k;i++){
        if(a[min]>a[i])
            min = i ;
    }
    return min ;
}

int main()
{
    Iris cla[3][150] ;
    int ncla[3][150] = {-1} ;
    Iris *center,*cenCopy ;
    int top[3] ;
    int i,j,h=0 ;
    ReadData() ;
    center = Random(3,150) ;
    DataCopy() ;
    cenCopy = centerCopy(center,3) ;
    cenCopy[0].x  = 0.0 ;
    for(i=0;i<3;i++){
        top[i] = 0 ;
    }
    while(!Compare(cenCopy,center,3)){  // 类中心不再变化停止循环
        cenCopy = centerCopy(center,3) ;
        for(i=0;i<3;i++){
            top[i] = 0 ;
        }
        for(i=0;i<150;i++){
            j = Belong(center,iris[i],3) ;
            ncla[j][top[j]] = i ;  // 源数据所在位置
            cla[j][top[j]].x = iris[i].x ;
            cla[j][top[j]].y = iris[i].y ;
            cla[j][top[j]].i = iris[i].i ;
            cla[j][top[j]].j = iris[i].j ;
            cla[j][top[j]].type = j ;
            top[j]++ ;
        }
        for(i=0;i<3;i++){
            center[i] = ComputingCenter(cla[i],top[i]) ;  // 重新计算类中心
            // printf("%d,%f,%f,%f,%f\n",i,center[i].x,center[i].y,center[i].i,center[i].j) ;
        }
        h++ ;  // 统计循环次数
    }
    for(i=0;i<3;i++){
        printf("\n第%d类数据(%d个)：\n",i,top[i]) ;
        for(j=0;j<top[i];j++){
            printf("%d\t",ncla[i][j]) ;
        }
    }
    printf("\n迭代次数：%d\n",h) ;
    return 0 ;
}
```

输出结果：
{{< figure src="/images/post/K-means聚类算法/20150616004340650.png" class="center" title="输出结果" alt="输出结果" >}}

## 参考
1. Jiawei Han 等著，《数据挖掘概念与技术》，2008
2. 吴夙慧等，k-means 算法研究综述，2011