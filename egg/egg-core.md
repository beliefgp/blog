# Egg核心库

该库提供了egg最底层的类库、方法，包括路由、文件加载逻辑等。


## EggCore

该类是egg app的基类，继承自koa的application。

### constructor

- BaseContextClass   
  Service、Controller等对象的基类。
  这就是在service、controller中```this.ctx```、```this.app```、```this.config```的根源。

  ```js
  // lib/util/base_context_class.js
  class BaseContextClass {
    constructor(ctx) {
      this.ctx = ctx;
      this.app = ctx.app;
      this.config = ctx.app.config;
      this.service = ctx.service;
    }
  }
  ```
- Controller  
  controller基类，指向BaseContextClass

- Service   
  service基类，指向BaseContextClass
 
- loader   
  EggLoader实例，文件加载工具类，可通过```Symbol.for('egg#loader')```重写。
  ```js
  const Loader = this[Symbol.for('egg#loader')];
  this.loader = new Loader({
    baseDir: options.baseDir,
    app: this,
    plugins: options.plugins,
    logger: this.console,
  });
  ```

- 执行_initReady()方法
  加载启动方法，此方法是app启动关键。详细参见[ready-callback](https://github.com/node-modules/ready-callback)
  > 可以通过```app.ready(fn)```，注册到ready的方法队列中，当执行```app.ready(true)```时，会按顺序执行队列中所有方法。

  ```js
  _initReady() {
    /**
     * require('ready-callback')({ timeout: 10000 }) 实例化ready对象，
     * 通过mixin(this)方法，将ready-callback的方法代理到app上
     */
    require('ready-callback')({ timeout: 10000 }).mixin(this);

    this.on('ready_stat', data => {
      this.console.info('[egg:core:ready_stat] end ready task %s, remain %j', data.id, data.remain);
    }).on('ready_timeout', id => {
      this.console.warn('[egg:core:ready_timeout] 10 seconds later %s was still unable to finish.', id);
    });

    this.ready(() => debug('egg emit ready, application started'));
  }


  /**
   * ready-callback
   * 实例化对象时，在下轮事件循环中，如果cache.size === 0，则触发this.ready(true)，执行所有注册的回调方法
   * 后续egg-cluster组件会使用到。
   */
  class Ready extends EventEmitter {
    constructor(opt) {
      super();
      ready.mixin(this);

      this.opt = opt || {};
      this.isError = false;
      this.cache = new Map();

      setImmediate(() => {
        // fire callback directly when no registered ready callback
        if (this.cache.size === 0) {
          debug('Fire callback directly');
          this.ready(true);
        }
      });
    }

    /**
     * 注册前置函数
     * 此方法在前置队列中，增加标记(构造函数中的触发逻辑就会被终断)，返回一个可触发ready(true)的function。
     * 前置函数执行完，手动执行返回的function，执行后续逻辑
     * app.beforeStart 就是依赖此逻辑。
     */
    readyCallback(name, opt) {
      opt = Object.assign({}, defaults, this.opt, opt);
      const cacheKey = uuid.v1();
      opt.name = name || cacheKey;
      const timer = setTimeout(() => this.emit('ready_timeout', opt.name), opt.timeout);
      const cb = once(err => {
        if (err != null && !(err instanceof Error)) {
          err = new Error(err);
        }
        clearTimeout(timer);
        // won't continue to fire after it's error
        if (this.isError === true) return;
        // fire callback after all register
        setImmediate(() => this.readyDone(cacheKey, opt, err));
      });
      cb.id = opt.name;
      this.cache.set(cacheKey, cb);
      return cb;
    }

    /**
     * 检测队列中是否还有未执行前置函数，若没有，则触发ready(true)。
     */
    readyDone(id, opt, err) {
      if (err != null && !opt.isWeakDep) {
        this.isError = true;
        return this.emit('error', err);
      }

      this.cache.delete(id);

      this.emit('ready_stat', {
        id: opt.name,
        remain: getRemain(this.cache),
      });

      if (this.cache.size === 0) {
        this.ready(true);
      }
      return this;
    }
  }

  module.exports = opt => new Ready(opt);
  ```
  
### use(fn)
注册中间件，注入app.middleware中

### beforeStart(fn)
app启动前，执行方法。

```js
beforeStart(scope) {
  if (!is.function(scope)) {
    throw new Error('beforeStart only support function');
  }

  // get filename from stack
  const name = utils.getCalleeFromStack(true);

  // 参见：_initReady()中的介绍
  const done = this.readyCallback(name);

  // ensure scope excutes after load completed
  process.nextTick(() => {
    co(function* () {
      yield utils.callFn(scope);
    }).then(() => done(), done);
  });
}

```

### beforeClose(fn)
app关闭时，执行方法。

### config
app.config，指向loader.config。

### type
option.type，用以区分application、agent，后续记录最终配置是会用到。

### router
app.router，路由对象，首次访问时，将路由注入到app中间件中。

### url
指向router.url方法

### del
指向router.delete方法

### [Symbol.for('egg#loader')]
返回 EggLoader对象

### get、post、put、delete等
路由方法。在router.js中，app.get等都是直接挂载到了路由实例上。
```js
// application对象代理所有路由方法
utils.methods.concat([ 'all', 'resources', 'register', 'redirect' ]).forEach(method => {
  EggCore.prototype[method] = function(...args) {
    this.router[method](...args);
    return this;
  };
});

```

## EggLoader

loader，提供加载Controller、Service、Config等约定目录文件的加载方法  
包含以下属性、方法([官方说明文档](https://eggjs.org/zh-cn/advanced/loader.html))：

### app
application实例

### pkg
项目的package.json

### serverEnv
服务器环境，就是所说的pro、local等，通过执行getServerEnv方法获取。

### appInfo
包含application基础信息的对象，通过执行```getAppInfo()```方法获取。

- name：项目package.json中的name，通过执行getAppname()方法获取。
- baseDir：当前项目根目录
- env：同serverEnv
- HOME：home路径，通过执行getHomedir()方法获取，一般在部署服务器时用到(存放日志等)。
- pkg：同上面的pkg
- root：根路径，存放日志等公共文件使用
  > env === 'local' || env === 'unittest' ? baseDir : home  
  
### eggPaths 
所有框架路径，通过执行```getEggPaths()```方法获取。**越底层的框架，路径越靠前**，这样在按路径顺序加载合并配置时，越上层的框架的配置优先级越高。

### getServerEnv()  
获取服务器环境
> 读取顺序：
> 1. 项目目录下的config/env文件  
> 2. process.env.EGG_SERVER_ENV  
> 3. process.env.NODE_ENV(默认local，test转为unittest，production转为pro)

### getAppname()    
获取项目package.json中的name。

### getHomedir()    
获取home路径
> 读取顺序：
> 1. process.env.EGG_HOME  
> 2. os.userInfo().homedir || os.homedir  
> 3. process.env.HOME  
> 4. /home/admin  

### getAppInfo()    
获取包含application基础信息的对象。

### getEggPaths()   
获取所有框架路径，通过遍历app原型链上约定的属性```Symbol.for('egg#eggPath')```，逐级遍历获取。  

```js
// 框架约定继承方式
class Application extends egg.Application {
  get [Symbol.for('egg#eggPath')]() {
    return path.dirname(__dirname);
  }
}

// lib/loader/egg-loader.js
getEggPaths() {
  const EggCore = require('../egg');
  const eggPaths = [];

  let proto = this.app;
  while (proto) {
    proto = Object.getPrototypeOf(proto);
    if (proto === Object.prototype || proto === EggCore.prototype) {
      break;
    }

    const eggPath = proto[Symbol.for('egg#eggPath')];
    const realpath = fs.realpathSync(eggPath);
    if (eggPaths.indexOf(realpath) === -1) {
      // 用unshift，加入队列头部，保证了越底层的框架，路径越靠前
      eggPaths.unshift(realpath);
    }
  }

  return eggPaths;
}
```

### **loadFile(filepath)**
加载文件方法。文件的exports基本都是通过此方法获取的。

```js
loadFile(filepath) {
  if (!fs.existsSync(filepath)) {
    return null;
  }

  // 通过require方式加载模块
  const ret = utils.loadFile(filepath);

  let inject = Array.prototype.slice.call(arguments, 1);
  if (inject.length === 0) inject = [ this.app ];

  // 支持方式module.exports = app => {};
  return isFunction(ret) ? ret.apply(null, inject) : ret;
}
```
### **getLoadUnits()**
获取所有插件、框架、及当前项目路径，以便加载对应的模块。**注意路径顺序**，将影响同类别配置在合并中，名称重复的属性覆盖的优先级。
> 路径顺序：插件路径orderPlugins(在loadPlugin时创建)，框架路径eggPaths，当前项目路径baseDir  
**因为一般是顺序遍历合并，后面的会覆盖前面的同名配置，所以，路径越靠后，优先级越高，也就是 app(当前项目) > framework > plugin**

### FileLoader  
模块加载对象，将属性挂载在全局对象上(如```app.controller```)，启动即加载属性。  
多级目录转为级联式调用，文件名默认转为驼峰式模块名。通过loadFile方法，获得模块exports，根据其用途，进行相应的适配转换。  
调用方法：```new FileLoader(option).load()```。
> 例如：service/open/login_config.js ==> service.open.loginConfig      
> **注意**：如果框架或插件之间，存在同路径文件，   
> plugin1/app/service/open/login_config.js   
> plugin2/app/service/open/login_config.js   
> 默认将会抛出一个异常，将option.override设为true，可使后加载的覆盖掉前面的，加载优先级问题，参考getLoadUnits()方法说明。   
> 详见文件lib/loader/file_loader.js

### ContextLoader  
模块加载对象，继承自FileLoader，将属性挂载在context对象上。通过改对象加载的属性均为**请求级别**，在每个请求中首次访问时进行实例化，并缓存，生命周期仅持续至请求结束。  
调用方法：```new ContextLoader(option).load()```。
> 详见lib/loader/context_loader.js。

```js
/**
  * 在context对象原型上，扩展属性，在第一次调用时，实例化。
  * 在每一个新请求时，Koa都会通过Object.creat()的方式重建context对象，保证了service的生命周期
  * 整个service级联对象，文件夹所在的属性，均会转为ClassLoader实例，在原型增加其子属性，加入缓存逻辑。
  * property = 'service';
  */
Object.defineProperty(app.context, property, {
  get() {
    if (!this[CLASSLOADER]) {
      this[CLASSLOADER] = new Map();
    }
    const classLoader = this[CLASSLOADER];

    let instance = classLoader.get(property);
    if (!instance) {
      instance = getInstance(target, this);
      classLoader.set(property, instance);
    }
    return instance;
  },
});

/**
  * 实例化service，ctx对象通过此种方法，层层传递下去
  * a/b/c.js ==> a.b.c
  * a ==> ClassLoader实例({ _ctx = ctx; b = new ClassLoader(); })
  * b ==> ClassLoader实例({ _ctx = ctx; c = new C(_ctx); })
  */
function getInstance(values, ctx) {
  // 如果对象是通过文件export导出，EXPORTS值为true，如果是文件夹名称转换来的，EXPORTS值为undefined
  const Class = values[EXPORTS] ? values : null;
  let instance;
  if (Class) {
    if (is.class(Class)) {
      instance = new Class(ctx);
    } else {
      instance = Class;
    }
  } else if (is.primitive(values)) {
    instance = values;
  } else {
    instance = new ClassLoader({ ctx, properties: values });
  }
  return instance;
}
```

### loadToApp(directory, property, LoaderOptions)  
通过FileLoader对象，加载目录[directory]下所有模块，并挂载到app对象上的[property]属性上。

### loadToContext(directory, property, opt)  
通过FileLoader对象，加载目录[directory]下所有模块，并挂载到ctx对象上的[property]属性上。

### loadPlugin()  
加载框架(```eggPaths```)及当前项目下(```baseDir```)、当前环境变量(```EGG_PLUGINS```)、以及实例化Application时(```startCluster(options)```)方式启动，内部会实例化Application，并把options传递过去)以传参形式添加的(```new Application({ plugins: [] })```)所有的插件。
> 1. 插件支持根据不同环境而单独配置：plugin.default.js(可简写为plugin.js)、plugin.[serverEnv].js。  
> 2. 插件配置合并优先级：实例化Application传入的 > 环境变量EGG_PLUGINS > 当前项目plugin.js配置 > 框架中配置   
> 3. 创建orderPlugins对象，存储需要加载的插件(配置中开启的插件、开启的插件所依赖的插件)的信息，供getLoadUnits方法读取。

### loadConfig()
加载所有config，并进行合并。   
config文件加载时，通过module.exports = appInfo => {};模式导出时，参数是appInfo，而不是app对象！
> 调用getLoadUnits()获取所有路径，先按顺序合并所有的config.default.js，再合并所有的config.[serverENv].js
```js
const config = this.loadFile(filepath, this.appInfo, extraInject);
```

### loadCustomApp()   
加载执行所有路径下的app.js

### loadCustomAgent()   
加载执行所有路径下的agent.js

### loadAgentExtend()  
加载所有路径下的app/extend/agent.js，将模块exports的属性，挂载到```app```对象上。  
> 通过Object.defineProperty、Object.getOwnPropertyDescriptor进行copy属性，防止getter、setter无法正常copy。

### loadApplicationExtend()   
加载所有路径下的app/extend/application.js，将模块exports的属性，挂载到```app```对象上。

### loadRequestExtend()   
加载所有路径下的app/extend/request.js，将模块exports的属性，挂载到```app.request```对象上。

### loadResponseExtend()   
加载所有路径下的app/extend/response.js，将模块exports的属性，挂载到```app.response```对象上。

### loadContextExtend()    
加载所有路径下的app/extend/context.js，将模块exports的属性，挂载到```app.context```对象上。

### loadHelperExtend()   
将所有路径下的app/extend/helper.js，将模块exports的属性，挂载到```app.Helper.prototype```(app.Helper 是继承自BaseContextClass的类，所以要扩展在原型上)对象上。

### loadMiddleware()   
加载所有路径下的中间件，通过```loadToApp```方法，挂载到```app.middlewares```，此时只是将所有中间件方法挂在到了app对象上而已，还没有加入koa中间件队列。   
根据读取config中coreMiddleware与middleware两个数组的合并结果，进行加载中间件。如果配置中，中间件名称不存在或者重复，将抛出一个异常。   
[官方中间件文档](https://eggjs.org/zh-cn/basics/middleware.html)

### loadService()   
加载所有路径下的app/service中的文件，通过loadToContext方法，挂载到app.service。

### loadController(options)
加载当前项目(```baseDir```)下的controller，包装成koa中间件模式，并以传参形式注入上下文环境(context)，通过```loadToApp```方法，挂载到```app.controller```上。

```js
function methodToMiddleware(Controller, key) {
  return function* classControllerMiddleware() {
    const controller = new Controller(this);
    const r = controller[key](this);
    if (is.generator(r) || is.promise(r)) {
      yield r;
    }
  };
}
```

### loadRouter()   
加载当前项目(```baseDir```)下的router.js。


  
    