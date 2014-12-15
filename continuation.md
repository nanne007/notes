---
layout: post
title: Continuation
tags: [cps continuation]
---

去年接触到 continuation 这个概念，前前后后也看了一些文章，但总是不得要领，最近又重新整理了思路，记录在这里。


### CPS

在利用 continuation 编程之前，需要认识一种编程风格：Continuation Passing Style(CPS)。
在 CPS 中，每一个过程（或者说函数）都接受一个额外的参数，这个参数代表了 *对该过程调用结果的处理* 。

举个例子：

以下这段代码以递归的形式计算前 n 个数的乘积。

``` racket
(define (factorial n)
 (if (= n 0)
  1     ; NOT tail-recursive
  (* n (factorial (- n 1)))))
```

如何把它变成 CPS 形式的呢？

首先给 `factorial` 加一个额外的参数 `k`，
这个 `k` 代表了当 `factorial` 调用结束后要执行的动作。

``` racket
; postfix a & to represent the cps version of a function
(define (factorial& n k)
 ???)
```

接下来的唯一“复杂”的地方就是 `(* n (factorial (- n 1)))` 了。
这个表达式中，先有 `(- n 1)` 调用，然后是 `factorial`，乘法是最后一步计算。
需要做的同上：给 `factorial` 调用加上一个 `k` 参数。

``` racket
(define (factorial& n k)
 (if (= n 0)
  1
  (factorial& (- n 1) (lambda (fact) ; use a lambda to represent the computation
					   (* n fact)))))
```

`k` 在这里用一个 lambda 表示，lambda 封装了 `factorial` 调用之后的动作，并接收调用结果作为参数 `fact`。

得到的这个就是 CPS 形式的 `factorial`。
调用它的时候，需要显式提供一个过程用来接收过程调用结果，
比如这样： `(factorial& 10 (lambda (x) (display x) (newline)))`。

我仿照 [continuation sandwich][continuation_sandwich] 写了一个中文类比：

> 假设现在你走进**厨房**，想做一碗**西红柿鸡蛋面**给自己吃，或者给女朋友吃。
> 无论是做给谁吃，还是想拿它做其他事情，暂且把它写在纸条上揣进兜里。
> 现在，你从冰箱里拿出来两个鸡蛋，一份面，当然还有可爱的西红柿，然后花个十分钟做好这份面。
> 这个时候，你再从兜里把纸条拿出来，还原当时的情景，看看自己做面是想干什么：依旧在厨房里，想着一碗西红柿鸡蛋面，做好了给女朋友吃。
> 接着这个思路，这个时候你会发现一份热腾腾的西红柿鸡蛋面已经摆在了你的面前，可以该干嘛干嘛了。

*参数 `k` 和纸条上的想法就是所说的 continuation。
可能你也会发现 javascript 最为“出名”的 [callback][callback] 也是 continuation。*


> 在 Racket 中，`=`、`*` 都是过程调用，严格来说，也需要做 CPS 变换。

> 下面给出详细的 CPS 转换过程，如果不感兴趣或者已熟谙于心，可以直接跳过。:)

#### `factorial` 完整的 CPS 变换


``` racket
(define (factorial& n k)
 (=& n 0 (lambda (b)
		  (if b
			  (k 1)
			  ???))))
```

接下来变换 `(* n (factorial (- n 1)))`。
前面提到过，这个表达式中，先有 `(- n 1)` 调用，然后是 `factorial`，乘法是最后一步计算。
依照先后顺序，可以依次作如下变换：

``` racket
; cps of (- n 1)
(define (factorial& n k)
 (=& n 0 (lambda (b)
		  (if b
			  (k 1)
			  (-& n 1 (lambda (nm1)
					   ???)))))

; cps of (factorial (- n 1))
(define (factorial& n k)
 (=& n 0 (lambda (b)
		  (if b
			  (k 1)
			  (-& n 1 (lambda (nm1)
					   (factorial& nm1 (lambda (factor)
										???)))))))

; cps of (* n (factorial (- n 1))), and that's it!
(define (factorial& n k)
 (=& n 0 (lambda (b)
		  (if b
			  (k 1)
			  (-& n 1 (lambda (nm1)
					   (factorial& nm1 (lambda (fact)
										(*& n fact k))))))))
```

### call/cc

continuation 就是这么个玩意儿，Scheme 提供了 call-with-current-continuation(call/cc) 来利用它编程。

call/cc 接受一个函数 `f` 作为参数，并把当前的 continuation 打包成函数，传递给 `f`，continuation 只能进行函数应用操作。


``` racket
; define a function f, take a function argument k
(define (f k)
  (k 1)
  (display 2))

(display (call/cc f))

(display 3)
```

分析下这个程序，先确定调用 `call/cc` 时的 *current continuation*：

``` racket
(lambda (x)
 (display x)
 (display 3))
```

参数 `x` 是 `call/cc` 调用的返回值，所谓的 *current continuation* 就是 `call/cc` 之后需要执行的代码块。

之后，这样的一个 continuation 就作为参数 `k` 传递给了 `call/cc` 的参数 `f`。

``` racket
(f (lambda (x)
    (display x)
    (display 3)))
```

在 `f` 的执行流中，`k` 的应用使得 `k` 中的执行流取代了之后的执行流 (即：`(lambda (y) (display 2))`)，
程序的运行由 `k` 中的执行流决定：

``` racket
((lambda (x)
  (display x)
  (display 3))
 1)
```


也可以保存 `call/cc` 创建出来的 continuation，以便反复使用。

``` racket
(define cont #f)
(display (call/cc (lambda (k) (set! cont k))))

(cont 1)
(cont 2)
(cont 3)
```

这就是 `call/cc`。


### delimited continuation(定界延续)

`call/cc` 无法控制 continuation 的边界，`call/cc` 调用之后的执行流都包含在 continuation 之内。
**delimited continuation** 是用来解决这个问题的，它将 continuation 控制在某一个范围内，比如说，

``` racket
(reset (display 0)
       (display (+ 1 (shift k (begin
                       (k 0)
                       (display 3)))))
       (display 2))
(display 4)
```

`shift` 就类似于 `call/cc` 的参数 `f` ，接收一个 continuation `k`，
而`reset` 则界定了 continuation 能够作用的范围。
在 `reset` 中调用 `shift` 时，`shift` 中的 continuation 就被限制在 `reset` 决定的范围里。

不过这里需要注意一点：`shift` 中，当 `(k 0)` 调用完毕后，执行流会继续往下执行 `(display 3)`，然后从这里跳出 `reset`。
`shift` 的使用使得执行流从 `reset` 转换到 `shift`，并以此结束 `reset`。



### next...

网上有很多相关的资料，我也考过许多，不过 Jim Mcbeath 的[delimited continuations][delimited continuations] 让我真正明白 continuation 做了一件什么事情。
下一步，想搞清楚为什么要有 continuation，具体实现上对闭包的处理，以及和尾递归优化的关系。



[continuation]: http://en.wikipedia.org/wiki/Continuation
[CPS]: http://en.wikipedia.org/wiki/Continuation-passing_style
[callback]: http://en.wikipedia.org/wiki/Callback_(computer_programming)
[continuation_sandwich]: http://en.wikipedia.org/wiki/Continuation#cite_note-cont-sandwich-3
[call-with-current-continuation]: http://en.wikipedia.org/wiki/Call-with-current-continuation
[delimited continuations]: http://jim-mcbeath.blogspot.com/2010/08/delimited-continuations.html

[Continuations in Scheme(draft)]: http://phillipwright.info/drafts/continuations.htm
