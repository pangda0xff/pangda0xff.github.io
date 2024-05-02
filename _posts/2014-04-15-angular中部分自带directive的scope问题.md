
设想有下面的页面：

```html
<ul>
    <li>
        <input type="text" ng-model="foo"/>
    </li>
    <li ng-repeat="index in \[1,2,3\]">
        <input type="text" ng-model="foo" />
    </li>
</ul>
<script type="text/javascript"> angular.module('main',\[\]); </script>
```

看起来很简单，生成了4个文本框，所有的文本框全部绑定到foo上，所以当任何一个文本框内容发生变化时，其它的文本框也会随之改变。第一个文本框工作正常，但在下面的三个文本框中输入后，其它的文本框并不会发生变化，此时再到第一个文本框中输入，做过修改的文本框也不再随之改变。

造成这种情况的原因在于，ngRepeat在每次枚举的时候生成一个child scope(姑且这么称呼吧)，这个scope的prototype为当前scope，所以当我们修改parent scope的时候。child scope也能响应修改。但js有一个古怪的设计：如果读取的属在当前对象中为undefined，会接着查找prototype的属性，所以当child scope属性未定义时，会通过prototype读到parent scope的值；但当写入属性时，不论当前对象和prototype的状态，直接写入到对象中，此时prototype和对象中的同一属性指向了两个不同的值，子与父的联系被切断了，这就是为什么修改了循环中的文本框内容时，这个文本框不再响应其它文本框的修改。

避免这种情况的方法是使用$parent直接访问parent scope：



<ul>
    <li>
        <input type="text" ng-model="foo"/>
    </li>
    <li ng-repeat="index in \[1,2,3\]">
        <input type="text" ng-model="$parent.foo" />
    </li>
</ul>



注意我们将值绑定在了$parent.foo上。

另外一个头痛的问题在于ngRepeat的生成的scope无法直接访问，如果实在有需要，可以通过函数传递出来：



<ul>
    <li>
        <input type="text" ng-model="foo"/>
    </li>
    <li ng-repeat="index in \[1,2,3\]">
        <input type="text" ng-model="foo" />
        <input type="button" ng-click="updateFoo(this)" value="更新到父项"/>
    </li>
</ul>
<script type="text/javascript"> angular.module('main',\[\])
    
    .controller('testCtrl',\['$scope',function($scope){
        $scope.updateFoo = function(childScope){
            $scope.foo = childScope.foo;
        }
    }\]); </script>



除了ngRepeat,其它一些directive也会有自己的scope，不过还是ngRepeat最难想到吧，有scope的：

*   ng-include
*   ng-switch
*   ng-repeat
*   ng-view
*   ng-controller

参考：[https://github.com/angular/angular.js/wiki/Understanding-Scopes](https://github.com/angular/angular.js/wiki/Understanding-Scopes "https://github.com/angular/angular.js/wiki/Understanding-Scopes")