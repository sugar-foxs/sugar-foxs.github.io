---
layout:     post
title:      spring事务传播机制
subtitle:   
date:       2019-12-24
author:     sugar-foxs
catalog: 	true
tags:
    - spring
---

本文主要介绍spring事务传播机制。

<!-- more -->

- spring事务传播机制有以下几种：
    - REQUIRED：如果有事务, 那么加入事务, 没有的话新建一个(默认情况下)。
    - NOT_SUPPORTED：不为这个方法开启事务，相当于没有事务。
    - REQUIRES_NEW：不管是否存在事务,都创建一个新的事务,原来的挂起,新的执行完毕,继续执行老的事务。
    - MANDATORY：必须在一个已有的事务中执行,否则抛出异常。
    - NEVER：必须在一个没有事务的方法中执行,否则抛出异常(与Propagation.MANDATORY相反)。
    - SUPPORTS：如果其他bean调用这个方法,在其他bean中声明事务,那就用事务.如果其他bean没有声明事务,那就不用事务。
    - NESTED：如果当前存在事务，则在嵌套事务内执行。如果当前没有事务，则执行与REQUIRED类似的操作。

> 假设方法a调用方法b,两个方法分别加上事务不同的传播机制。看会发生什么。

## REQUIRED
- 两个都是REQUIRED机制，b方法发现a已经有事务了，就不再新建事务了，加入a事务。
- a使用NOT_SUPPORTED,b使用REQUIRED,b方法发现a没有事务，会自己新建一个事务。

## NOT_SUPPORTED
- 这个相当于没有Spring事务，每条执行语句单独执行，单独提交。

## REQUIRES_NEW
- a使用REQUIRED，b使用REQUIRES_NEW，b会新建一个独立事务，a的事务挂起，b的事务先提交，然后a的事务执行再提交。
- 外层事务的回滚不影响REQUIRES_NEW事务的执行，即a事务回滚，b事务不会回滚，不受影响。

## MANDATORY
- a使用NOT_SUPPORTED,b使用MANDATORY，会发生错误。
- MANDATORY必须在已有的事务中执行。

## NEVER
- a使用REQUIRED,b使用NEVER，即a使用事务，b使用NEVER传播机制会发生错误。
- 因为b使用了NEVER机制，必须在一个没有事务的方法中执行，否则报错。

## SUPPORTS
- a使用REQUIRED，b使用SUPPORTS,b会复用a的事务。
- a使用NOT_SUPPORTED，b使用SUPPORTS,b便不会使用事务。
- 是否使用事务取决于调用方法是否有事务，如果有则直接用，如果没有则不使用事务。

## NESTED
- a使用REQUIRED，b使用NESTED,b直接在a事务的基础上创建一个嵌套事务，本质上还是同一个事务，做一次提交。
- a使用NOT_SUPPORTED，b使用NESTED，b直接创建一个新的事务，单独提交。