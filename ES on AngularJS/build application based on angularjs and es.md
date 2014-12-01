# 使用AngularJS为Elasticsearch创建一个前端 #

如果使用Elasticsearch使用应用的数据源，我们可以很方便的使用AngularJS结合Elasticsearch提供的相关模块为它创建一个前端。

以创建一个简单的员工信息花名册为例。

## 准备工作 ##

准备工作分为以下两个方面：

### 添加前端依赖 ###

1. 安装Bower
2. 在bower.json中添加对于AngularJS和Elasticsearch的依赖：

```json
"dependencies": {
    "angular": "~1.2.15",
    "elasticsearch": "~2.4.0"
}
```

### 准备运行时环境 ###

对于简单的应用和Demo，可以直接使用Node环境中提供的http-server，非常简单快捷。

1. 安装Node
2. 安装http-server，通过命令：npm install -g http-server

## 配置AngularJS以及ES Client ##

### 配置AngularJS ###

```js
var rosterApp = angular.module('rosterApp', ['elasticsearch']);
```

### 配置Elasticsearch Client ###

对于一个只读的应用，我们只需要将搜索的功能暴露出来。可以通过自定义一个factory来实现：

```js
rosterApp.factory('rosterService',
    ['$q', 'esFactory', '$location', function($q, elasticsearch, $location){
        var client = elasticsearch({
            host: $location.host() + ":9200"
        });

        var search = function(term, offset){
            var deferred = $q.defer();
            var query = {
                "match": {
                    "_all": term
                }
            };

            client.search({
                "index": 'roster',
                "type": 'employee',
                "body": {
                    "size": 10,
                    "from": (offset || 0) * 10,
                    "query": query
                }
            }).then(function(result) {
                var ii = 0, hits_in, hits_out = [];
                hits_in = (result.hits || {}).hits || [];
                for(;ii < hits_in.length; ii++){
                    hits_out.push(hits_in[ii]._source);
                }
                deferred.resolve(hits_out);
            }, deferred.reject);

            return deferred.promise;
        };

        return {
            "search": search
        };
    }]
);
```

在以上定义的factory中，我们只暴露了search方法。该方法调用了client的search方法完成基于关键字的搜索。

## 控制器(Controller) ##

下面需要做的就是实现一个控制器来完成页面与上述search方法的交互：

```js
rosterApp.controller('rosterCtrl',
    ['rosterService', '$scope', '$location', function(rosterService, $scope, $location){
        $scope.employees = [];
        $scope.page = 0;
        $scope.allResults = false;
	    $scope.searchTerm = '';

        $scope.search = function(){
            $scope.page = 0;
            $scope.employees = [];
            $scope.allResults = false;
            $location.search({'q': $scope.searchTerm});
            $scope.loadMore();
        };

        $scope.loadMore = function(){
            rosterService.search($scope.searchTerm, $scope.page++).then(function(results){
                if(results.length !== 10){
                    $scope.allResults = true;
                }

                var ii = 0;
                for(;ii < results.length; ii++){
                    $scope.employees.push(results[ii]);
                }
            });
        };

        $scope.loadMore();
    }]
);
```

search方法用来执行搜索，loadMore方法用于加载更多记录。

## 页面 ##

页面主要用于展示Employee对象的具体属性。

```html
<div ng-app='rosterApp' ng-controller='rosterCtrl'>
	<header>
    	<h1>Employee Search</h1>
    </header>
    <section class='searchField'>
       	<form ng-submit='search()'>
           	<input ng-model='searchTerm' type='text'>
            <input type='submit' value='Search for employees'>
        </form>
    </section>
    <section class='results'>
       <div class='no-employees' ng-hide='employees.length'>No results</div>
       <article class='employee' ng-cloak ng-repeat='employee in employees'>
           <h2>
              <a ng-href='{{employee.link}}'>{{employee.name}}</a>
           </h2>
           <ul>
              <li ng-repeat='historyItem in employee.historyItems'>{{ historyItem }}</li>
           </ul>
           <p>
              {{employee.introduction}}
              <a ng-href='{{employee.link}}'>More information...</a>
           </p>
       </article>
       <div class='load-more' ng-cloak ng-hide='allResults'>
           	<a ng-click='loadMore()'>More...</a>
       </div>
    </section>
</div>
```

## 运行 ##

进入工程目录，运行http-server。访问http://localhost:8080。

当需要使用不同的查询类型时，通过修改search方法中query的创建部分即可。