# 如何开发一个JQuery插件

用插件来拓展JQuery十分方便，将功能封装到插件中可以节省大部分的开发时间。

<h2 id = '1'>开始</h2>
要编写一个 jQuery 插件，需要为 jQuery.fn 对象增加一个新的函数属性，属性名就是插件的名字

```javascript
jQuery.fn.myPlugin = function() {

    // 插件的具体内容放在这里

};
```
为了确保你的插件不与其它使用 $ 的库发生冲突，有一个最佳实践： 把 jQuery 传递给 IIFE（立即调用函数），并通过它映射成 $ ，这样就避免了在执行的作用域里被其它库所覆盖。

```javascript
(function( $ ) {
    $.fn.myPlugin = function() {
    
        // 插件的具体内容放在这里
    
    };
})( jQuery );
```
<h2 id='2'>上下文</h2>
现在，已经有了外壳，可以开始编写真正的插件代码了。但在这之前，我们来介绍下上下文。在插件函数的立即作用域中，关键字 this 指向调用插件的 jQuery 对象。这是个经常出错的地方，因为有些情况下 jQuery 接受一个回调函数，此时 this 指向原生的 DOM 元素。这常常导致开发者在 jQuery 函数中对 this 关键字多作一次无必要的包装。

```javascript
(function( $ ){
    $.fn.myPlugin = function() {
    
        // 没有必要再作 $(this) ，因为"this"已经是 jQuery 对象了
        // $(this) 与 $($('#element')) 是相同的
           
        this.fadeIn('normal', function(){
            // 在这里 this 关键字指向 DOM 元素
        });		    		    
    };  		
})( jQuery );

   
$('#element').myPlugin();
```
<h2 id='3'>基础</h2>
现在理解了 jQuery 插件的上下文以后， 我们来写一个真正能做点儿事儿的插件。

```javascript
(function( $ ){

  $.fn.maxHeight = function() {
  
    var max = 0;

    this.each(function() {
      max = Math.max( max, $(this).height() );
    });

    return max;
  };
})( jQuery );

--
var tallest = $('div').maxHeight(); // 返回最高 div 的高度
```
这个简单的插件利用` .height()` 来返回页面中最高 div 的高度
<h2 id='4'>保持 chainability</h2>
前面的例子返回了页面上最高 div 的一个整数值，但很多时候插件只是以某种方式修改元素集合，并把它们传给调用链的下一个方法。 这正是 jQuery 设计的漂亮之处，也是它如此流行的原因之一。为保持插件的 chainability ，必须确保插件返回 this 关键字。
 
```javascript
(function( $ ){

  $.fn.lockDimensions = function( type ) {  

    return this.each(function() {

      var $this = $(this);

      if ( !type || type == 'width' ) {
        $this.width( $this.width() );
      }

      if ( !type || type == 'height' ) {
        $this.height( $this.height() );
      }

    });

  };
})( jQuery );

--
$('div').lockDimensions('width').css('color', 'red');
```


<h2 id='5'>默认设置和选项</h2>
对于那些提供许多选项、更复杂、更可配置的插件，最佳实践是提供一个默认设置，它可在插件调用时（通过 `$.extend`）被扩展。这样调用插件时无需大量参数， 只要一个对象参数，内容为你希望不同于默认值的那部分设置。做法如下：
 
```javascript
(function( $ ){

  $.fn.tooltip = function( options ) {  

    // Create some defaults, extending them with any options that were provided
    var settings = $.extend( {
      'location'         : 'top',
      'background-color' : 'blue'
    }, options);

    return this.each(function() {        

      // Tooltip plugin code here

    });

  };
})( jQuery );

--
$('div').tooltip({
  'location' : 'left'
});    
```

在本例中，用给定选项调用 `tooltip` 插件后， 默认的 `location` 设置被覆盖为 `"left"`， 但 `bacground-color` 设置仍为默认值 `"blue"`。最终的设置对象看起来这样的：

```javascript
{
  'location'         : 'left',
  'background-color' : 'blue'
}
```
这是一个非常好的方式， 可以提供一个高度可配置的插件，又不必强制开发者定义所有选项。

<h2 id='6'>名称空间</h2>
合理地为插件定义名称空间是插件开发中很重要的一部分。 正确的定义名称空间可以确保你的插件很难被其它插件或同一页面中的其它代码所覆盖。名称空间也可以帮你跟踪你的方法、事件和数据。

<h4 id='6.1'>插件方法</h4>
单个插件永远不要在 `jQuery.fn` 对象中声明一个以上的名称空间

```javascript
(function( $ ){

  $.fn.tooltip = function( options ) { 
    // 这
  };
  $.fn.tooltipShow = function( ) {
    // 不
  };
  $.fn.tooltipHide = function( ) { 
    // 好
  };
  $.fn.tooltipUpdate = function( content ) { 
    // !!!  
  };

})( jQuery );
```
这非常不好，因为它搞乱了 $.fn 名称空间。 要修正这个问题，你应该把所有插件方法收集到一个对象定义当中，并通过传递方法名称字符串调用.
 
```javascript
(function( $ ){

  var methods = {
    init : function( options ) { 
      // 这 
    },
    show : function( ) {
      // 很
    },
    hide : function( ) { 
      // 好
    },
    update : function( content ) { 
      // !!! 
    }
  };

  $.fn.tooltip = function( method ) {
    
    // Method calling logic
    if ( methods[method] ) {
      return methods[ method ].apply( this, Array.prototype.slice.call( arguments, 1 ));
    } else if ( typeof method === 'object' || ! method ) {
      return methods.init.apply( this, arguments );
    } else {
      $.error( 'Method ' +  method + ' does not exist on jQuery.tooltip' );
    }    
  
  };

})( jQuery );

// 调用  init 方法
$('div').tooltip(); 

// 调用  init 方法
$('div').tooltip({
  foo : 'bar'
});

--
// 调用 hide 方法
$('div').tooltip('hide'); 

--
// 调用 update 方法
$('div').tooltip('update', 'This is the new tooltip content!');
```

 
这种插件架构使你可以在插件的父闭包中封装所有方法，调用时先传方法名称字符串，接下来再把你需要的其它参数传给该方法。这种封装和架构是 jQuery 插件社区的一个标准，已经被无数插件所使用，包括 jQueryUI 中的插件和小部件。

<h4 id='6.2'>事件</h4>

`bind` 方法有个鲜为人知的特性：它支持为绑定事件定义名称空间。如果你的插件要绑定事件，最好为其定义名称空间。这样，回头想 `unbind` 的时候就不会影响到相同事件类型上的其它已绑定事件。要为事件定义名称空间，把 `".<namespace>"` 附到要绑定的事件类型后面即可。

```javascript
(function( $ ){

  var methods = {
     init : function( options ) {

       return this.each(function(){
         $(window).bind('resize.tooltip', methods.reposition);
       });

     },
     destroy : function( ) {

       return this.each(function(){
         $(window).unbind('.tooltip');
       })

     },
     reposition : function( ) { 
       // ... 
     },
     show : function( ) { 
       // ... 
     },
     hide : function( ) {
       // ... 
     },
     update : function( content ) { 
       // ...
     }
  };

  $.fn.tooltip = function( method ) {
    
    if ( methods[method] ) {
      return methods[method].apply( this, Array.prototype.slice.call( arguments, 1 ));
    } else if ( typeof method === 'object' || ! method ) {
      return methods.init.apply( this, arguments );
    } else {
      $.error( 'Method ' +  method + ' does not exist on jQuery.tooltip' );
    }    
  
  };

})( jQuery );


--
$('#fun').tooltip();
// Some time later...
$('#fun').tooltip('destroy');
```
在本例中，当 `tooltip` 被 `init` 方法初始化的时候，它把 `reposition` 方法绑定到 `window` 对象的 `resize` 事件上，名称空间为 `"tooltip"`。 之后，如果开发者想要销毁对象，可以把插件的名称空间（即 `"tooltip"`）传给 `unbind` 方法，以便解除本插件对所有事件的绑定。这使得我们可以安全的地解除本插件的事件绑定，避免意外影响插件之外绑定的事件。

<h4 id='6.3'>数据</h4>
插件开发中，你可能经常需要维护状态，或检查你的插件是否已在给定元素上做过初始化。jQuery data 方法是针对每个元素跟踪变量的好办法。不过最好能用单一对象容纳所有变量并用单一名称空间访问此对象，而不是分别跟踪一堆不同名字的数据。

```javascript
(function( $ ){

  var methods = {
     init : function( options ) {

       return this.each(function(){
         
         var $this = $(this),
             data = $this.data('tooltip'),
             tooltip = $('<div />', {
               text : $this.attr('title')
             });
         
         // If the plugin hasn't been initialized yet
         if ( ! data ) {
         
           /*
             Do more setup stuff here
           */

           $(this).data('tooltip', {
               target : $this,
               tooltip : tooltip
           });

         }
       });
     },
     destroy : function( ) {

       return this.each(function(){

         var $this = $(this),
             data = $this.data('tooltip');

         // Namespacing FTW
         $(window).unbind('.tooltip');
         data.tooltip.remove();
         $this.removeData('tooltip');

       })

     },
     reposition : function( ) { // ... },
     show : function( ) { // ... },
     hide : function( ) { // ... },
     update : function( content ) { // ...}
  };

  $.fn.tooltip = function( method ) {
    
    if ( methods[method] ) {
      return methods[method].apply( this, Array.prototype.slice.call( arguments, 1 ));
    } else if ( typeof method === 'object' || ! method ) {
      return methods.init.apply( this, arguments );
    } else {
      $.error( 'Method ' +  method + ' does not exist on jQuery.tooltip' );
    }    
  
  };

})( jQuery );
```
`data` 方法可以帮你在插件的多次方法调用之间跟踪变量和状态。 把数据置于单一对象中，并为其定义名称空间有利于集中访问插件的所有属性，同时也减少了名称空间以便需要时删除。

<h2 id='7'>总结及最佳实践</h2>
编写 jQuery 插件使库更加高效。把你最聪明、最有用的功能抽象成可重用代码，这将节省你的时间，进一步提高开发效率。下面是本文档的简要总结以及你开发下一个 jQuery 插件时的注意事项：

* 总是把插件包装在闭包中 { /* plugin goes here */ })( jQuery );
* 不在插件函数的立即作用域中额外包装 this 关键字
* 总是让插件函数返回 this 关键字以保持 chainability ，除非插件有真正的返回值。
* 不要传给插件大量参数，应该传一个可以覆盖插件默认选项的设置对象。
* 在单个插件中，不要让一个以上的名称空间搞乱了 jQuery.fn 对象。
* 总是为方法、事件和数据定义名称空间。

最后附上一个推荐的JQuery 插件模板供参考：

```javascript
/*
 * 插件注释开头写插件名称、版本、作者、描述等信息
 * for example, for example:
 * ------------------------------------------------------------------------
 * jQuery-DistPlugin Version 0.1
 * Created by LancerSung on 2017/4/7
 * 一个jQuery插件模板
 *
 */

(function($, window, document, undefined) {

    /**
     * 将插件名称存放在变量中，方便后续修改插件名称（一般不会修改）
     * @type {String}
     */
    var pluginName = 'distPlugin';

    /**
     * 插件构造函数
     * @param {DOM Element} element --插件挂载的DOM元素
     * @param {Object} options --传给构造函数的参数
     */
    function Plugin(element, options) {

        // 存储原始元素
        this.el = element;

        // 存储jquery对象
        this.$el = $(element);

        //构造一个设置对象，包含默认设置以及用户自定义设置
        this.options = $.extend({}, $.fn[pluginName].defaults, options);

        // 初始化插件实例
        this.init();
    }

    /**
     * 用必要的方法来设置插件原型
     * 最佳实践是在原型中实现‘init'和‘destroy’方法
     */
    $.extend(Plugin.prototype, {

        /**
         * 初始化插件实例.
         * 设置相关属性, 存储相关节点引用, 添加监听事件等
         *
         * 当为插件绑定事件时，最好为其定义名称空间
         * 元素可能在同一事件类型上绑定多个监听
         * 这样，回头想 unbind 的时候就不会影响到相同事件类型上的其它已绑定监听
         *
         * @example
         * this.$someSubElement.on('click.' + pluginName, function() {
         *      // 逻辑处理
         * });
         *
         */
        init: function() {

            // this._bindEvents();
        },

        /**
         *  'destroy' 方法用来销毁插件，释放插件占用的资源，移除相关事件监听…
         *
         * 记得‘unbind’你绑定的事件监听
         *
         * @example
         * this.$someSubElement.off('.' + pluginName);
         *
         * 上面这个例子是用来解除插件相关的所有监听
         *
         */
        destroy: function() {
            //解除绑定
            //this._unbindEvents();

            // 移除所有插件绑定的数据
            this.$el.removeData();
        },

        /**
         * 编写插件的公有方法
         * 公有方法可以像下面这么调用：
         *
         * @example
         * $('#element').distPlugin('somePublicMethod', 'Here', 1001);
         *
         * @param  {[type]} foo [some parameter]
         * @param  {[type]} bar [some other parameter]
         * @return {[type]}
         */
        somePublicMethod: function(foo, bar) {

            // 调用原型内部的私有方法（伪私有）
            this._pseudoPrivateMethod();

            // 调用原型外部的私有方法（真私有），使用call或者apply来调用
            _privateMethod.call(this);
        },

        /**
         * 用来实现类似getter 方法的公有方法.
         * 可以像调用普通共有函数一样调用:
         *
         * @example
         * $('#element').distPlugin('someGetterMethod');
         *
         * 来获取某些对插件的调用者来说有用的信息
         *
         * @return {[type]} Return something
         */
        someGetterMethod: function() {

        },

        /**
         * 以下划线来命名函数标明它是私有方法，
         * 通过插件内部的逻辑处理带下划线的函数将无法在外部直接调用
         * for example:
         *
         *  @example
         *  $('#element').distPlugin('_pseudoPrivateMethod');  // 外部调用这个函数将不会被执行
         */
        _pseudoPrivateMethod: function() {

        }

        /***
         * 绑定事件
         */
        /*
         *  _bindEvents: function() {
         *    var plugin = this;
         *    plugin.$el.on('click'+'.'+pluginName, function() {
         *      _privateMethod.call(plugin);
         *    }
         *  }
         */
        /***
         * 解除事件绑定
         */
        /*
        *_unbindEvents: function() {
        *   this.$el.off('.'+pluginName);
        * }
        *
        *
         */

    });

    /**
     * 这是个私有函数. 只有插件实例对象可以调用它
     * @return {[type]}
     */
    var _privateMethod = function() {
        console.log("privateMethod");
        console.log(this);
    };

    /**
     * 拓展Jquery插件
     * 使用data避免插件在给定元素上做重复初始化
     * 对含有下划线的方法过滤阻止外部调用
     * @example
     * $('#element').distPlugin({
     *     defaultOption: 'this options overrides a default plugin option',
     *     additionalOption: 'this is a new option'
     * });
     */
    $.fn[pluginName] = function(options) {
        var args = arguments;

        if (options === undefined || typeof options === 'object') {
            // 创建一个插件实例, 遍历选中的节点, 并将创建的插件实例保存至节点的data中
            this.each(function() {
                if (!$.data(this, 'plugin_' + pluginName)) {
                    $.data(this, 'plugin_' + pluginName, new Plugin(this, options));
                }
            });
            return this;
        } else if (typeof options === 'string' && options[0] !== '_' && options !== 'init') {
            //在选中的节点上调用公共方法
            if (Array.prototype.slice.call(args, 1).length == 0 && $.inArray(options, $.fn[pluginName].getters) != -1) {
                //调用类getter方法，这样链式调用则被破坏会返回某个感兴趣的值而非jquery对象
                var instance = $.data(this[0], 'plugin_' + pluginName);
                return instance[options].apply(instance, Array.prototype.slice.call(args, 1));
            } else {
                // 在节点上调用指定的方法
                this.each(function() {
                    var instance = $.data(this, 'plugin_' + pluginName);
                    if (instance instanceof Plugin && typeof instance[options] === 'function') {
                        instance[options].apply(instance, Array.prototype.slice.call(args, 1));
                    }
                });
                return this;
            }
        }
    };

    /**
     * 指定这个插件的getter方法
     * @type {Array}
     */
    $.fn[pluginName].getters = ['someGetterMethod'];

    /**
     * 默认配置
     */
    $.fn[pluginName].defaults = {
        defaultOption: "I'm a default option"
    };

})(jQuery, window, document);
```
