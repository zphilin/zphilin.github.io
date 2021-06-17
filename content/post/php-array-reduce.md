---
title: "PHP：数组降维"
date: 2017-06-30T01:37:56+08:00
#lastmod: 2017-08-30T01:37:56+08:00
draft: false
tags: ["PHP"]
categories: ["后台"]
author: "zphilin"

---

php提供的数组操作函数非常丰富，极大方便构建一般的数据结构。
通常在关系型数据库中得到数据结果集均为二维结果，两种常见的数据处理情形有：

1. 只需要某一个字段的数据集
2. 字段A为key，字段B为值

最容易想到的方法便是遍历数组，然后组装。

针对case 1，有[array_column](http://php.net/array_column) 函数可以直接获取到一维数组，但有版本限制，PHP>=5.5

针对case 2，最近翻官网发现有个[array_reduce](http://php.net/array_reduce)，看名字第一眼可能会想到MapReduce，的确也有[array_map](http://php.net/array_column)函数，但意义并不太一样，不过这个reduce函数甚是好用。

现有数据集：

```php
<?php
$ret = [
    ["name"=>"ifreer","score"=>99],
    ["name"=>"Robert","score"=>98],
    ["name"=>"Einstein","score"=>100],
];

//common
foreach($ret as $item){
	$data[$item['name']] = $item['score'];  
}

//by array_reduce
$data = array_reduce($ret,function($return,$item){
	$return[$item['name']] = $item['score'];
  	return $return;
});

print_r($data);

//result
Array
(
    [ifreer] => 99
    [Robert] => 98
    [Einstein] => 100
)

```

两种办法代码量差不多，也很相似，性能未做比较测试，实际情况中也应避免巨大的数据集直接置于内存，可查看[yield]()，只此一记。

*ifreer@2017.06.30*
