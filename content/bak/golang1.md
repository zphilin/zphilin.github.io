---
title: "初识Golang"
date: 2018-07-25T01:37:56+08:00
#lastmod: 2017-08-30T01:37:56+08:00
draft: false
tags: ["Golang","协程"]
categories: ["后台"]
author: "zphilin"

---
# 初识Golang

在一两年前，有想要看下Go，然后断断续续开始从网上找相关资料，但始终好像没找到合适的，尤其是中文相关资料。最开始找了《the way to go》，《go in action》但基本每次都是过一遍语法，然后就丢一边，不用几个月基本就忘了。

最近从网上买了一本《Go语言编程》，翻阅一番后，有些地方很是疑惑，比如就在第一章介绍流程控制，其中一句：`“在有返回值的函数中，不允许将“最终的”return语句包含在if...else...结构中，否则会编译失败”`，然后给出了失败示例：

```go
func example(x int) int {
	if x == 0 {
		return 5
  	} else {
		return x
	}
}
```

这是其他语言最普遍的用法，如果说return不能在条件中结束函数使其返回，一定要在最后加个return，这语法显得太不“高级”了。而且也完全打破了我们日常推荐的编程习惯：条件分支满足直接返回，避免多层嵌套，使逻辑更加清晰。自己写了一遍，亲测之后基于go1.5,这段代码完全没有任何异常。只能说出书还是得谨慎些。

至此，我想我需要一本更好的参考资料，有时候初学者第一步资料其实真的挺重要，如果一开始就错了或打击了学习积极性后面就比较麻烦，再次看到《the way to go》 只不过现在有中文译版了叫《[GO入门指南](https://github.com/Unknwon/the-way-to-go_ZH_CN/issues/261) 》其中有网友提出没有ebook之类的pdf或epub格式方便离线阅读，*作者回应：导出并不是一件容易的事*。哈哈，那作为一名程序员，作为学习，干脆用Go来实现生成pdf如何？

基于对markdown的了解，很多软件支持直接从md格式导出为html或pdf等便携格式。通过进一步摸索，发现了一些工具，原理都是使用[Pandoc](http://www.pandoc.org/) 只需一个简单的命令即可完成，但却有很多依赖需要安装，也是在找寻的过程中，误入了图灵社区一人写的文章，按照文章的方法，一步步一堆东西装下来上G，最后却还没成功，这年头写文章发博客都这么随意，不管结果？

重新理了下思路，然后决定采用速战速觉的方法，将《Go入门指南》的md文件down下来，然后通过Go编写程序合并这些文件为一个文件，再通过markdown工具直接导出html或pdf，这算是开启Go程序之旅吧。

因为已经用python写了个脚步跑过一遍合并，所以按照同样的思想想来用Go也是几分钟的事。然鹅，采用Go来做的时候断断续续却花了两三个小时才完成。主要遇到的问题有：

1. IO函数不熟悉，读写用起来不方便
2. 包只要import就必须使用，调试过程中容易反复注解
3. 变量只要定义就必须使用
4. 强类型编译型语言，各种类型容易对不上
5. 简短申明方式`a := 666`冒号总是忘打
6. 返回值问题！
   1. `too many arguments to return`
   2. ` not enough arguments to return`
   3. 函数最后直接return 会返回默认的已定义命名返回值，其实我只是想简单返回一个`空`
7. 几乎所有的资料基本都一个套路就是讲语法，IO使用的介绍都木有，网上中文搜索结果基本又都是相互copy

以上都是一些小的，表面的问题，也并没有真正体现Go的优势之处，虽然相对于C已经非常脚本化，但如果从PHP过来，还真是瞬间感受到PHP的灵活之处，可以说是相当傻瓜化了。
只所以有这么多小问题，固然和强类型编译型语言有关。Go自带并发编程优势，goroutines相较与其他的通过多进程和线程的方式，自行管理更轻量级的“协程”，执行同步编码的方式却能达到异步的效果。后面有机会用的时候，再来具体分析吧，以下便是所写的合并代码。这本书相对来说讲的算是比较全面的了，值得一阅，最后附上这本自制书的pdf版：[Go入门指南](https://github.com/zphilin/gitbook-tech/blob/master/gobook.pdf)

```go
package main

import (
	"os"
	"io/ioutil"
	"strings"
)

const fileroot = "/Users/xfreer/go/src/github.com/gobook/eBook/"

func Listdir(path string, suffix string) (files []string, err error) {
    files = make([]string, 0, 100)
    dir, err := ioutil.ReadDir(path)
    if err != nil {
		return nil, err
    }
  
    for _, fi := range dir {
		if fi.IsDir() {
	    	continue
		}
		files = append(files, fi.Name())
    }
  
    return files, nil
}

func main() {
    dir, _ := Listdir(fileroot, "md")
    newfile := "/Users/xfreer/go/src/gobook.md"
    for _,filename := range dir{
		if strings.Contains(filename,"directory") {
	    	continue
		}
		if strings.Contains(filename,"preface") {
	    	continue
		}

		filepath := fileroot+filename
		b,_ :=ioutil.ReadFile(filepath)
//		ioutil.WriteFile(newfile,b,0777)
		nf,_ :=os.OpenFile(newfile,os.O_WRONLY,0644)
		n,_ :=nf.Seek(0,os.SEEK_END)
		nf.WriteAt(b,n)
		defer nf.Close()
    }
}
```

*ifreer@2017.06.25*
