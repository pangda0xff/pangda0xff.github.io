Javascript中必须通过this来访问类成员，可是this的特点就是函数绑在哪个对象上，它就指向那个对象。这个可能困扰过很多的程序员，特别是从C#，Java等语言过来的程序员。

```javascript
function Foo(){ this.message = 'This is message from Foo';
}

Foo.prototype.printMessage = function(){
    console.log(this.message);
} function Foo2(){ this.message = 'This is message from Foo2';
} var foo = new Foo();
foo.printMessage(); var foo2 = new Foo2();
foo2.printMessage = foo.printMessage;
foo2.printMessage();
```

输出为：

This is message from Foo

This is message from Foo2

主要原因就是this改变了，因此Javascript中this的用法，和C++\\C#中的大为不同。如果需要传统方式使用this的函数，可以使用Function.prototype.bind(),指定函数的this值：

```javascript
function Foo(){ this.message = 'This is message from Foo'; this.printMessage = (function(){
        console.log(this.message);
    }).bind(this);
} function Foo2(){ this.message = 'This is message from Foo2';
} var foo = new Foo();
foo.printMessage(); var foo2 = new Foo2();
foo2.printMessage = foo.printMessage;
foo2.printMessage();
```

输出为：

This is message from Foo

This is message from Foo

另外使用call和apply也可以改变函数调用时的this值。

bind函数的主要问题是IE9以后才开始提供。并且一旦开始习惯了Javascript的this用法，这种bind反而会不习惯。在实践中，更多用到的还是保存this：

```javascript
function Foo(){ var that = this; this.message = 'This is message from Foo'; this.printMessage = function(){
        console.log(that.message);
    };
} function Foo2(){ this.message = 'This is message from Foo2';
} var foo = new Foo();
foo.printMessage(); var foo2 = new Foo2();
foo2.printMessage = foo.printMessage;
foo2.printMessage();
```

输出同上。

注意我们是通过that来访问的message(除了that，context和self也是常用的名称)。Javascript一个还算欣慰的地方就是他的闭包上下文始终是在函数定义的地方，因此不管函数被挂上哪个对象上，捕获到的that始终是这个。当然这个地方不算闭包，有闭无包，但原理是相同的。这也是实践中用的最多的方法，推荐使用。