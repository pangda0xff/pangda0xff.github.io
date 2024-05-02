angular中的各种关于事件处理的directive。比如ngClick，ngKeydown，都是在ngEventDirs.js定义的，打开看一下，400多行的源文件，实际代码只有这20行：

```javascript
var ngEventDirectives = {};
forEach(
  'click dblclick mousedown mouseup mouseover mouseout mousemove mouseenter mouseleave keydown keyup keypress submit focus blur copy cut paste'.split(' '),
  function(name) {
    var directiveName = directiveNormalize('ng-' + name);
    ngEventDirectives[directiveName] = ['$parse', function($parse) {
      return {
        compile: function($element, attr) {
          var fn = $parse(attr[directiveName]);
          return function(scope, element, attr) {
            element.on(lowercase(name), function(event) {
              scope.$apply(function() {
                fn(scope, {$event:event});
              });
            });
          };
        }
      };
    }];
  }
);
```

剩下的全部是注释。事件处理相关的directive 在api文档页面中占去了好大一片，实际上是通过一个循环20行就搞定了，有点毁三观啊。。。

关于如何取得event object的问题，在ngClick这样的directive中，event是无效的，不过可以使用$event。这个对象文档中没有定义。也难怪stackoverflow上有人说拿不到事件对象。其实是可以的，而且$event的内容还挺丰富的。