# spilled map design

本文主要描述 spilled map 设计思路和接口



## Requirement

总体来说是需要一个支持溢出的 Map 结构，满足如下功能

* 可指定该map内存消耗，内存不够时支持溢出
* 其中满足特定规律的KV对（内存可以放下）支持更新
* 检索效率尽可能高
* 更新效率高

## Interface

XXXMap

* insert(k, v), put(k, v), insert(Iterator[(k, v)])
  
* getOrElseUpdate(k, v), get(k)—option(V)
  
* values 
  
* clean
  
* allcanspill
  
* spillAll or release
  
* registerBypassFunction
  
* ​
  
  ​