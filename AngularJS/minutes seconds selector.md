在一个项目中需要一个用来输入分钟数和秒数的控件，然而调查了一些开源项目后并未发现合适的控件。在Angular Bootstrap UI中有一个类似的控件TimePicker，但是它并没有深入到分钟和秒的精度。

因此，决定参考它的源码然后自己进行实现。

最终的效果如下：

![](https://github.com/destiny1020/js-learning-notes-cn/blob/master/images/minute-second-picker.png)

首先是该directive的定义：

```js
app.directive('minuteSecondPicker', function() {
    return {
        restrict: 'EA',
        require: ['minuteSecondPicker', '?^ngModel'],
        controller: 'minuteSecondPickerController',
        replace: true,
        scope: {
            validity: '='
        },
        templateUrl: 'partials/directives/minuteSecondPicker.html',
        link: function(scope, element, attrs, ctrls) {
            var minuteSecondPickerCtrl = ctrls[0],
                ngModelCtrl = ctrls[1];

            if(ngModelCtrl) {
                minuteSecondPickerCtrl.init(ngModelCtrl, element.find('input'));
            }
        }
    };
});
```

在以上的link函数中，ctrls是一个数组：
ctrls[0]是定义在本directive上的controller实例，ctrls[1]是ngModelCtrl，即ng-model对应的controller实例。这个顺序实际上是通过require: ['minuteSecondPicker', '?^ngModel']定义的。

注意到第一个依赖就是directive本身的名字，此时会将该directive中controller声明的对应实例传入。第二个依赖的写法有些奇怪："?^ngModel"，?的含义是即使没有找到该依赖，也不要抛出异常，即该依赖是一个可选项。^的含义是查找父元素的controller。

然后，定义该directive中用到的一些默认设置，通过constant directive实现：

```js
app.constant('minuteSecondPickerConfig', {
    minuteStep: 1,
    secondStep: 1,
    readonlyInput: false,
    mousewheel: true
});
```

紧接着是directive对应的controller，它的声明如下：

```js
app.controller('minuteSecondPickerController', ['$scope', '$attrs', '$parse', 'minuteSecondPickerConfig', 
    function($scope, $attrs, $parse, minuteSecondPickerConfig) {
	...
}]);
```

在directive的link函数中，调用了此controller的init方法：

```js
   this.init = function(ngModelCtrl_, inputs) {
        ngModelCtrl = ngModelCtrl_;
        ngModelCtrl.$render = this.render;

        var minutesInputEl = inputs.eq(0),
            secondsInputEl = inputs.eq(1);

        var mousewheel = angular.isDefined($attrs.mousewheel) ? 
            $scope.$parent.$eval($attrs.mousewheel) : minuteSecondPickerConfig.mousewheel;
        if(mousewheel) {
            this.setupMousewheelEvents(minutesInputEl, secondsInputEl);
        }

        $scope.readonlyInput = angular.isDefined($attrs.readonlyInput) ?
            $scope.$parent.$eval($attrs.readonlyInput) : minuteSecondPickerConfig.readonlyInput;
        this.setupInputEvents(minutesInputEl, secondsInputEl);
    };
```

init方法接受的第二个参数是inputs，在link函数中传入的是：element.find('input')。
所以第一个输入框用来输入分钟，第二个输入框用来输入秒。

然后，检查是否覆盖了mousewheel属性，如果没有覆盖则使用在constant中设置的默认mousewheel，并进行相关设置如下：

```js
    // respond on mousewheel spin
    this.setupMousewheelEvents = function(minutesInputEl, secondsInputEl) {
        var isScrollingUp = function(e) {
            if(e.originalEvent) {
                e = e.originalEvent;
            }

            // pick correct delta variable depending on event
            var delta = (e.wheelData) ? e.wheelData : -e.deltaY;
            return (e.detail || delta > 0);
        };

        minutesInputEl.bind('mousewheel wheel', function(e) {
            $scope.$apply((isScrollingUp(e)) ? $scope.incrementMinutes() : $scope.decrementMinutes());
            e.preventDefault();
        });

        secondsInputEl.bind('mousewheel wheel', function(e) {
            $scope.$apply((isScrollingUp(e)) ? $scope.incrementSeconds() : $scope.decrementSeconds());
            e.preventDefault();
        });
    };
```

init方法中的最后一个设置是input的响应：

```js
    // respond on direct input
    this.setupInputEvents = function(minutesInputEl, secondsInputEl) {
        if($scope.readonlyInput) {
            $scope.updateMinutes = angular.noop;
            $scope.updateSeconds = angular.noop;
            return;
        }

        var invalidate = function(invalidMinutes, invalidSeconds) {
            ngModelCtrl.$setViewValue(null);
            ngModelCtrl.$setValidity('time', false);
            $scope.validity = false;
            if(angular.isDefined(invalidMinutes)) {
                $scope.invalidMinutes = invalidMinutes;
            }
            if(angular.isDefined(invalidSeconds)) {
                $scope.invalidSeconds = invalidSeconds;
            }
        };

        $scope.updateMinutes = function() {
            var minutes = getMinutesFromTemplate();

            if(angular.isDefined(minutes)) {
                selected.minutes = minutes;
                refresh('m');
            } else {
                invalidate(true);
            }
        };

        minutesInputEl.bind('blur', function(e) {
            if(!$scope.invalidMinutes && $scope.minutes < 10) {
                $scope.$apply(function() {
                    $scope.minutes = pad($scope.minutes);
                });
            }
        });

        $scope.updateSeconds = function() {
            var seconds = getSecondsFromTemplate();

            if(angular.isDefined(seconds)) {
                selected.seconds = seconds;
                refresh('s');
            } else {
                invalidate(undefined, true);
            }
        };

        secondsInputEl.bind('blur', function(e) {
            if(!$scope.invalidSeconds && $scope.seconds < 10) {
                $scope.$apply(function() {
                    $scope.seconds = pad($scope.seconds);
                });
            }
        });
    };
```

此方法中，声明了用于设置输入非法的invalidate函数，它会在scope中暴露一个validity = false属性让页面有机会做出合适的反应。

如果用户使用了一个变量来表示minuteStep或者secondStep，那么还需要设置相应的watchers：

```js
    var minuteStep = minuteSecondPickerConfig.minuteStep;
    if($attrs.minuteStep) {
        $scope.parent.$watch($parse($attrs.minuteStep), function(value) {
            minuteStep = parseInt(value, 10);
        });
    }

    var secondStep = minuteSecondPickerConfig.secondStep;
    if($attrs.secondStep) {
        $scope.parent.$watch($parse($attrs.secondStep), function(value) {
            secondStep = parseInt(value, 10);
        });
    }
```

完整的directive实现代码如下：

```js
var app = angular.module("minuteSecondPickerDemo");

app.directive('minuteSecondPicker', function() {
    return {
        restrict: 'EA',
        require: ['minuteSecondPicker', '?^ngModel'],
        controller: 'minuteSecondPickerController',
        replace: true,
        scope: {
            validity: '='
        },
        templateUrl: 'partials/directives/minuteSecondPicker.html',
        link: function(scope, element, attrs, ctrls) {
            var minuteSecondPickerCtrl = ctrls[0],
                ngModelCtrl = ctrls[1];

            if(ngModelCtrl) {
                minuteSecondPickerCtrl.init(ngModelCtrl, element.find('input'));
            }
        }
    };
});

app.constant('minuteSecondPickerConfig', {
    minuteStep: 1,
    secondStep: 1,
    readonlyInput: false,
    mousewheel: true
});

app.controller('minuteSecondPickerController', ['$scope', '$attrs', '$parse', 'minuteSecondPickerConfig', 
    function($scope, $attrs, $parse, minuteSecondPickerConfig) {

    var selected = {
            minutes: 0,
            seconds: 0
        },
        ngModelCtrl = {
            $setViewValue: angular.noop
        };

    this.init = function(ngModelCtrl_, inputs) {
        ngModelCtrl = ngModelCtrl_;
        ngModelCtrl.$render = this.render;

        var minutesInputEl = inputs.eq(0),
            secondsInputEl = inputs.eq(1);

        var mousewheel = angular.isDefined($attrs.mousewheel) ? 
            $scope.$parent.$eval($attrs.mousewheel) : minuteSecondPickerConfig.mousewheel;
        if(mousewheel) {
            this.setupMousewheelEvents(minutesInputEl, secondsInputEl);
        }

        $scope.readonlyInput = angular.isDefined($attrs.readonlyInput) ?
            $scope.$parent.$eval($attrs.readonlyInput) : minuteSecondPickerConfig.readonlyInput;
        this.setupInputEvents(minutesInputEl, secondsInputEl);
    };

    var minuteStep = minuteSecondPickerConfig.minuteStep;
    if($attrs.minuteStep) {
        $scope.parent.$watch($parse($attrs.minuteStep), function(value) {
            minuteStep = parseInt(value, 10);
        });
    }

    var secondStep = minuteSecondPickerConfig.secondStep;
    if($attrs.secondStep) {
        $scope.parent.$watch($parse($attrs.secondStep), function(value) {
            secondStep = parseInt(value, 10);
        });
    }

    // respond on mousewheel spin
    this.setupMousewheelEvents = function(minutesInputEl, secondsInputEl) {
        var isScrollingUp = function(e) {
            if(e.originalEvent) {
                e = e.originalEvent;
            }

            // pick correct delta variable depending on event
            var delta = (e.wheelData) ? e.wheelData : -e.deltaY;
            return (e.detail || delta > 0);
        };

        minutesInputEl.bind('mousewheel wheel', function(e) {
            $scope.$apply((isScrollingUp(e)) ? $scope.incrementMinutes() : $scope.decrementMinutes());
            e.preventDefault();
        });

        secondsInputEl.bind('mousewheel wheel', function(e) {
            $scope.$apply((isScrollingUp(e)) ? $scope.incrementSeconds() : $scope.decrementSeconds());
            e.preventDefault();
        });
    };

    // respond on direct input
    this.setupInputEvents = function(minutesInputEl, secondsInputEl) {
        if($scope.readonlyInput) {
            $scope.updateMinutes = angular.noop;
            $scope.updateSeconds = angular.noop;
            return;
        }

        var invalidate = function(invalidMinutes, invalidSeconds) {
            ngModelCtrl.$setViewValue(null);
            ngModelCtrl.$setValidity('time', false);
            $scope.validity = false;
            if(angular.isDefined(invalidMinutes)) {
                $scope.invalidMinutes = invalidMinutes;
            }
            if(angular.isDefined(invalidSeconds)) {
                $scope.invalidSeconds = invalidSeconds;
            }
        };

        $scope.updateMinutes = function() {
            var minutes = getMinutesFromTemplate();

            if(angular.isDefined(minutes)) {
                selected.minutes = minutes;
                refresh('m');
            } else {
                invalidate(true);
            }
        };

        minutesInputEl.bind('blur', function(e) {
            if(!$scope.invalidMinutes && $scope.minutes < 10) {
                $scope.$apply(function() {
                    $scope.minutes = pad($scope.minutes);
                });
            }
        });

        $scope.updateSeconds = function() {
            var seconds = getSecondsFromTemplate();

            if(angular.isDefined(seconds)) {
                selected.seconds = seconds;
                refresh('s');
            } else {
                invalidate(undefined, true);
            }
        };

        secondsInputEl.bind('blur', function(e) {
            if(!$scope.invalidSeconds && $scope.seconds < 10) {
                $scope.$apply(function() {
                    $scope.seconds = pad($scope.seconds);
                });
            }
        });
    };

    this.render = function() {
        var time = ngModelCtrl.$modelValue ? {
            minutes: ngModelCtrl.$modelValue.minutes,
            seconds: ngModelCtrl.$modelValue.seconds
        } : null;

        // adjust the time for invalid value at first time
        if(time.minutes < 0) {
            time.minutes = 0;
        }
        if(time.seconds < 0) {
            time.seconds = 0;
        }

        var totalSeconds = time.minutes * 60 + time.seconds;
        time = {
            minutes: Math.floor(totalSeconds / 60),
            seconds: totalSeconds % 60
        };

        if(time) {
            selected = time;
            makeValid();
            updateTemplate();
        }
    };

    // call internally when the model is valid
    function refresh(keyboardChange) {
        makeValid();
        ngModelCtrl.$setViewValue({
            minutes: selected.minutes,
            seconds: selected.seconds
        });
        updateTemplate(keyboardChange);
    }

    function makeValid() {
        ngModelCtrl.$setValidity('time', true);
        $scope.validity = true;
        $scope.invalidMinutes = false;
        $scope.invalidSeconds = false;
    }

    function updateTemplate(keyboardChange) {
        var minutes = selected.minutes,
            seconds = selected.seconds;

        $scope.minutes = keyboardChange === 'm' ? minutes : pad(minutes);
        $scope.seconds = keyboardChange === 's' ? seconds : pad(seconds);
    }

    function pad(value) {
        return ( angular.isDefined(value) && value.toString().length < 2 ) ? '0' + value : value;
    }

    function getMinutesFromTemplate() {
        var minutes = parseInt($scope.minutes, 10);
        return (minutes >= 0) ? minutes : undefined;
    }

    function getSecondsFromTemplate() {
        var seconds = parseInt($scope.seconds, 10);
        if(seconds >= 60) {
            seconds = 59;
        }

        return (seconds >= 0) ? seconds : undefined;
    }

    $scope.incrementMinutes = function() {
        addSeconds(minuteStep * 60);
    };

    $scope.decrementMinutes = function() {
        addSeconds(-minuteStep * 60);
    };

    $scope.incrementSeconds = function() {
        addSeconds(secondStep);
    };

    $scope.decrementSeconds = function() {
        addSeconds(-secondStep);
    };

    function addSeconds(seconds) {
        var newSeconds = selected.minutes * 60 + selected.seconds + seconds;
        if(newSeconds < 0) {
            newSeconds = 0;
        }

        selected = {
            minutes: Math.floor(newSeconds / 60),
            seconds: newSeconds % 60
        };

        refresh();
    }

    $scope.previewTime = function(minutes, seconds) {
        var totalSeconds = parseInt(minutes, 10) * 60 + parseInt(seconds, 10),
            hh = pad(Math.floor(totalSeconds / 3600)),
            mm = pad(minutes % 60),
            ss = pad(seconds);

        return hh + ':' + mm + ':' + ss;
    };
}]);
```

对应的Template实现：

```html
<table>
    <tbody>
        <tr class="text-center">
            <td>
                <a ng-click="incrementMinutes()" class="btn btn-link">
                    <span class="glyphicon glyphicon-chevron-up"></span>
                </a>
            </td>
            <td>&nbsp;</td>
            <td>
                <a ng-click="incrementSeconds()" class="btn btn-link">
                    <span class="glyphicon glyphicon-chevron-up"></span>
                </a>
            </td>
            <td>&nbsp;</td>
        </tr>
        <tr>
            <td style="width:50px;" class="form-group" ng-class="{'has-error': invalidMinutes}">
                <input type="text" ng-model="minutes" ng-change="updateMinutes()" class="form-control text-center" ng-mousewheel="incrementMinutes()" ng-readonly="readonlyInput" maxlength="3">
            </td>
            <td>:</td>
            <td style="width:50px;" class="form-group" ng-class="{'has-error': invalidSeconds}">
                <input type="text" ng-model="seconds" ng-change="updateSeconds()" class="form-control text-center" ng-mousewheel="incrementSeconds()" ng-readonly="readonlyInput" maxlength="2">
            <td>
            <!-- preview column -->
            <td>
                <span class="label label-primary" ng-show="validity">
                    {{ previewTime(minutes, seconds) }}
                </span>
            </td>
        </tr>
        <tr class="text-center">
            <td>
                <a ng-click="decrementMinutes()" class="btn btn-link">
                    <span class="glyphicon glyphicon-chevron-down"></span>
                </a>
            </td>
            <td>&nbsp;</td>
            <td>
                <a ng-click="decrementSeconds()" class="btn btn-link">
                    <span class="glyphicon glyphicon-chevron-down"></span>
                </a>
            </td>
            <td>&nbsp;</td>
        </tr>
    </tbody>
</table>
```

测试代码(即前面截图dialog的源代码)：

```html
<div class="modal-header">
    <h3 class="modal-title">Highlight on <span class="label label-primary">{{ movieName }}</span></h3>
</div>
<div class="modal-body">

    <div class="row">
        <div id="highlight-start" class="col-xs-6">
            <h4>Start Time:</h4>
            <minute-second-picker ng-model="startTime" validity="startTimeValidity"></minute-second-picker>
        </div>

        <div id="highlight-end" class="col-xs-6">
            <h4>End Time:</h4>
            <minute-second-picker ng-model="endTime" validity="endTimeValidity"></minute-second-picker>
        </div>
    </div>
    <div class="row">
        <div class="col-xs-2">
            Tags:
        </div>
        <div class="col-xs-10">
            <tags model="tags" src="s as s.name for s in sourceTags" options="{ addable: 'true' }"></tags>
        </div>
    </div>
</div>
<div class="modal-footer">
    <button class="btn btn-primary" ng-click="ok()" ng-disabled="!startTimeValidity || !endTimeValidity || durationIncorrect(endTime, startTime)">OK</button>
    <button class="btn btn-warning" ng-click="cancel()">Cancel</button>
</div>
```







