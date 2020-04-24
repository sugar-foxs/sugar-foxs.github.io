---
layout:     post
title:      java获取集合的交集，差集，并集的不同方式
subtitle:   
date:       2020-01-18
author:     sugar-foxs
catalog: 	true
tags: java
---

本文主要介绍java获取set的交集，差集，并集的不同方式。
<!-- more -->

# 使用java原生集合实现

```java
/**
 * @author guchunhui
 **/
public class Test1 {
    public static void main(String[] args) {
        Set<Integer> set1 = new HashSet<>();
        set1.add(1);
        set1.add(2);
        set1.add(3);
        Set<Integer> set2 = new HashSet<>();
        set2.add(1);
        set2.add(4);
        set2.add(5);

        // 交集
        Set<Integer> set3 = new HashSet<>(set1);
        set3.retainAll(set2);
        System.out.println(set3); // 输出 [1]

        // 差集
        Set<Integer> set4 = new HashSet<>(set1);
        set4.removeAll(set2);
        System.out.println(set4);// 输出 [2,3]

        // 并集
        Set<Integer> set5 = new HashSet<>(set1);
        set5.addAll(set2);
        System.out.println(set5);// 输出 [1,2,3,4,5]
    }
}
```

# 使用guava实现
```java
/**
 * @author guchunhui
 **/
public class Test2 {
    public static void main(String[] args) {
        Set<Integer> set1 = Sets.newHashSet(1,2,3);
        Set<Integer> set2 = Sets.newHashSet(1,4,5);

        // 交集
        Sets.SetView<Integer> intersection = Sets.intersection(set1, set2);
        System.out.println(intersection); //输出 [1]

        // 差集, 返回第一个集合有。但在第二个集合中没有的元素
        Sets.SetView<Integer> diff = Sets.difference(set1, set2);
        System.out.println(diff); // 输出 [2,3]

        Sets.SetView<Integer> diff2 = Sets.difference(set2, set1);
        System.out.println(diff2); // 输出 [4,5]

        // 并集
        Sets.SetView<Integer> union = Sets.union(set1, set2);
        System.out.println(union); //输出 [1,2,3,4,5]
    }
}
```

- 再看看guava其他实现
```java
/**
 * @author guchunhui
 **/
public class Test3 {
    public static void main(String[] args) {
        Set<Integer> set1 = Sets.newHashSet(1,2,3);
        Set<Integer> set2 = Sets.newHashSet(1,4,5);

        // 除了交集，差集，并集，guava还提供了其他实用功能

        Set<List<Integer>> listSet = Sets.cartesianProduct(set1,set2);
        // 返回 [[1, 1], [1, 4], [1, 5], [2, 1], [2, 4], [2, 5], [3, 1], [3, 4], [3, 5]]
        // 从结果应该就能看出这个方法是干什么用的。从两个set[1,2,3] [1,4,5] 中各取出一个元素组成list的所有可能。
        System.out.println(listSet);

        // list也有类似功能
        List<Integer> list1 = Lists.newArrayList(1,2,4);
        List<Integer> list2 = Lists.newArrayList(1,5,6);

        List<List<Integer>> lists = Lists.cartesianProduct(list1,list2);
        // 返回 [[1, 1], [1, 5], [1, 6], [2, 1], [2, 5], [2, 6], [4, 1], [4, 5], [4, 6]]
        System.out.println(lists);

        // 我们在看看实际工作中经常用的Map
        Map<String,Integer> map1 = Maps.newHashMap();
        map1.put("1",1);
        map1.put("3",3);
        map1.put("4",4);

        Map<String,Integer> map2 = Maps.newHashMap();
        map2.put("1",1);
        map2.put("2",2);
        map2.put("4",5);

        MapDifference<String,Integer> difference = Maps.difference(map1,map2);
        System.out.println(difference.areEqual()); // 输出 false
        System.out.println(difference.entriesInCommon());// 输出 {1=1}
        System.out.println(difference.entriesOnlyOnLeft());// 输出 {3=3}
        System.out.println(difference.entriesOnlyOnRight());// 输出 {2=2}
        // 返回具有相同key，但value不一样，将不一样的value放入ValueDifference。
        System.out.println(difference.entriesDiffering());// 输出 {4=(4,5)}

        // 还有一种map是实际工作中经常需要用到的，value是List类型
        // 我们可以使用Multimap,ArrayListMultimap或者LinkedListMultimap;
        Multimap<String,Integer> multimap = ArrayListMultimap.create();

        // 相同key的值并不会覆盖，而是加入list。
        multimap.put("a",1);
        multimap.put("a",2);
        multimap.put("a",3);

        for(String key:multimap.keySet()) {
            System.out.println(key);//输出a
            Collection<Integer> value = multimap.get(key);
            System.out.println(value);//输出 [1,2,3]
        }
    }
}
```

## guava中其他集合实现
- Multiset
```java
    Multiset<Long> set = HashMultiset.create();
    set.add(1L);
    set.add(1L);
    set.add(2L);
    set.add(4L);
    for (Multiset.Entry<Long> entry:set.entrySet()) {
        System.out.println(entry.getElement()+":"+entry.getCount());
    }
```
>输出：  
1:2  
2:1  
4:1  

- BiMap
```java
    BiMap<String, String> biMap = HashBiMap.create();
    biMap.put("a","b");
    biMap.put("a","c");// 上一个b会被覆盖
    biMap.put("b","c");// 这个会报错，因为value也不可重复
    biMap.put("b","d");
    for (BiMap.Entry<String,String> entry:biMap.entrySet()) {
        System.out.println(entry.getKey()+":"+entry.getValue());
    }
```
> 输出：  
a:c  
b:d  

- Table
```java
    Table<String,String,String> table = HashBasedTable.create();
    table.put("r1","c1","1");
    table.put("r1","c2","2");
    table.put("r1","c3","3");
    table.put("r2","c1","4");
    table.put("r2","c2","5");
    table.put("r2","c3","6");
    for (Map.Entry<String,Map<String,String>> entry:table.rowMap().entrySet()){
        System.out.println(entry.getKey());
        System.out.println(entry.getValue());
    }
    for (String s:table.columnKeySet()){
        System.out.println(s);
    }
    for (Table.Cell<String,String,String> cell:table.cellSet()) {
        System.out.println(cell);
    }
```
>输出:  
r1  
{c1=1, c2=2, c3=3}  
r2  
{c1=4, c2=5, c3=6}  
c1  
c2  
c3  
(r1,c1)=1  
(r1,c2)=2  
(r1,c3)=3  
(r2,c1)=4  
(r2,c2)=5  
(r2,c3)=6  

