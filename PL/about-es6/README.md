# ES6 新特性


Ref [mozilla's es6 in depth](https://hacks.mozilla.org/category/es6-in-depth/)

## for-of loop

Via https://hacks.mozilla.org/2015/04/es6-in-depth-iterators-and-the-for-of-loop/

for-of 只是一个语法糖。
内部需要借助 iterator，凡是实现 iterator 接口的 object 都可以使用 for-of。

所谓的 iterator 接口只是需要 object 包含一个 key 为 `Symbol.iterator` 的函数字段，该函数执行返回 iterator 对象。

iterator 对象需要实现 `next` 方法，以及两个可选的方法： `return` 和 `throw(exc)`。

如果在 for-of 过程中 break 或者 return 或者 exception 了， `return` 方法就会被调用。

下面是一个简单的例子：（斐波那契数）

``` javascript
class Fibonacci {
    constructor() {
        this.a = 0;
        this.b = 1;

    }

    [Symbol.iterator]() {
        return {
            next: () => {
                const t = this.b;
                this.b = this.a + this.b;
                this.a = t;
                return { done: false, value: this.b - this.a };
            },
            return: () => {
              console.log("exit");
            }
        };
    }
}
```

在 for-of 中使用它：

``` javascript
for(var fi of new Fibonacci()) {
  if(fi > 10) {
    break;
  }

  console.log(fi);
}
```

[babel](http://babeljs.io) 会将上述代码转义成下面这样：

``` javascript
var _iteratorNormalCompletion = true;
var _didIteratorError = false;
var _iteratorError = undefined;

try {

    for (var _iterator = new Fibonacci()[Symbol.iterator](), _step; !(_iteratorNormalCompletion = (_step = _iterator.next()).done); _iteratorNormalCompletion = true) {
        var fi = _step.value;

        if (fi > 10) {
            break;
        }

        console.log(fi);
    }
} catch (err) {
    _didIteratorError = true;
    _iteratorError = err;
} finally {
    try {
        if (!_iteratorNormalCompletion && _iterator["return"]) {
            _iterator["return"]();
        }
    } finally {
        if (_didIteratorError) {
            throw _iteratorError;
        }
    }
}
```

可以看到，当 for-of 不是正常结束的时候，`return` 就会执行（当然，如果定义了的话）。

至于说 `throw(exc)`, 下一节再看。

## 生成器
- https://hacks.mozilla.org/2015/05/es6-in-depth-generators
- https://hacks.mozilla.org/2015/07/es6-in-depth-generators-continued

语法： `function* (...args) { ... }` ，称之为 **生成器函数**（generator function）。

可以在 _生成器函数_ 中使用 yield 方法达到 iterator 效果。

这里的 `yield` 和 ruby 中的 `yield` 并没有本质区别。

函数 yield 过程：

(函数调用开始)start of function --> (`.next()`)yield --> (`.next()`)yield --> .... --> (`.next()`)end of function

### `yield` 返回值

`yield [expression]` 也可以返回一个值： `var v = yield someExpression`。
这个值从 `.next(value)` 通过参数  `value` 传过来。

start of function --> (`.next(undefined)`)yield --> (`.next(v)`)yield --> .... --> (`.next(v)`)end of function

``` javascript
var g = getSomeGenerator();
// new created generator is suspended.
//  when first call next, the value passed to next is meaningless.
// and the generator run to first yield, and suspended again.
g.next(someValue);

// when call next again, the value passed to it is the last-encountered yield's  return value.
g.next(anotherValue);
```

``` javascript
function* range(start, end) {
  try {
    for(var i = start; i < end; i++)
      console.log(yield i);
  }
  finally {
    console.log("clean up");
  }

}

var rng = range(0, 10);

rng.next("what"); // no output
rng.return("what"); // output 'what'
rng.next("what"); // output 'what'
```

### `.return` 方法

前面说过，iterator 在迭代的过程中，可能会 break，会 return，也会 error，这个时候 iterator 的 `return` 方法会被调用来对 iterator 作一些收尾工作，这个收尾工作可能是关闭文件，断开数据库连接等等。

Generator 作为一种 Iterator，也可以定义 `return` 方法。`try {} finally {}` 中的 `finally` 就是。。（不要问我，我也不知道为啥要这么设计）。
如果 Generator 中有使用 try-finally，那么finally 块就相当于 `return`。。

``` javascript
function* range(start, end) {
  try {
    for(var i = start; i < end; i++)
      console.log(yield i);
  }
  finally {
    console.log("clean up");
  }

}

var rng = range(0, 10);

rng.next("what"); // no output
rng.next("what"); // output 'what'
rng.return(); // output 'clean up'
rng.next("what"); // no output
```

### `.throw` 方法

Generator 的调用者也可以直接 `throw(exception)`，这相当于 `yield someExpression` 在执行过程中出错，`exception` 会在 Generator  中抛出。

``` javascript
function* range(start, end) {
  try {
    for(var i = start; i < end; i++)
      console.log(yield i);
  } catch (e) {
    console.log(`error: ${e}`);
  }
  finally {
    console.log("clean up");
  }

}

var rng = range(0, 10);

rng.next("what"); // no output
rng.throw("what"); // output 'error: what' and 'clean up'
rng.next("what"); // no output
```

### `yield*`

往往需要在一个 generator 中调用另一个 generator，

``` javascript
function* concat(generatora, generatorb) {
  for (var a of generatora) yield a;
  for (var b of generatorb) yield b;
}
```

`yield*` 更适用于这种的情况。

``` javascript
function* concat(generatora, generatorb) {
  yield* a;
  yield* b;
}
```

### TODO：生成器结合 `Promise`

## Template String

模板字符串功能在其他语言已经很常见了：

``` javascript
// javascript
`hello, ${yourname}`
```

``` scala
// scala
s"hello, $yourname"
```

``` ruby
# ruby
"hello, #{yourname}"
```

``` elixir
# elixir
~s(hello, #{yourname})
```

但是，从本质上来说，javascript 的 **template string** 和 elixir 的 [sigils](http://elixir-lang.org/getting-started/sigils.html#custom-sigils) 更类似，可以自定义 template 处理方法。

``` javascript
function SaferHTML(templateData) {
  var s = templateData[0];
  for (var i = 1; i < arguments.length; i++) {
    var arg = String(arguments[i]);

    // Escape special characters in the substitution.
    s += arg.replace(/&/g, "&amp;")
            .replace(/</g, "&lt;")
            .replace(/>/g, "&gt;");

    // Don't escape special characters in the template.
    s += templateData[i];
  }
  return s;
}

let yourname = "<myname>";
let greeting = SaferHTML`<p>hello, ${yourname}</p>`; // => <p>hello, &lt;myname&gt;</p>
```

### Rest parameters and defaults && Destructuring

这两个功能不再细说，其他语言早已经实现了。

### Arrow Function

类似于 _coffeescript_ 中的函数简写形式。
值得注意的一点是 _arrow function_ 里面的 `this` 是 context 中的 `this`。

### Symbol

符号类型是新加的一种类型。

其他语言如，Ruby，Elixir 都有这个概念。

但是 Symbol 主要是 ES6 为了做兼容而想出来的一种不失优雅的解决方案。
不过以后应该会派上更大的用场。

