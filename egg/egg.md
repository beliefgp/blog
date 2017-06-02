# Egg

egg框架，基于egg-core、egg-cluster等库，进行重写封装。

## EggApplication

继承自EggCore类，扩展了一些属性、方法，application、agent类的父类。

### constructor

- ContextCookies   
  Cookies类，```ctx.cookies``` 是此类实例。   
  具体参考[egg-cookies](https://github.com/eggjs/egg-cookies)

- ContextLogger  
  Logger类，```ctx.logger```、```ctx.coreLogger```都是此类实例，此类主要是在打印日志时，加入了```ctx.userId```、```ctx.ip```、```ctx.method```、```ctx.url```、请求用时等一些请求信息，底层还是调用的```app.logger```或```app.coreLogger```对象。       
  具体参考[egg-logger](https://github.com/eggjs/egg-logger)
  [.EggContextLogger](https://github.com/eggjs/egg-logger/blob/master/lib/egg/context_logger.js)

- ContextHttpClient
  HttpClient类，```ctx.httpclient```是此类实例。    
  此类是基于```app.curl```方法的封装，```app.curl```是```app.httpclient.request```的简写，```app.httpclient```是urllib库的HttpClient类的实例     
  具体参考(urllib)[http://github.com/node-modules/urllib]

- messenger 
  Messenger类实例，进程通信对象。   
  Messenger类是在(sendmessage)[https://github.com/node-modules/sendmessage]库的基础上进行的封装，然后配合[egg-cluster](https://github.com/eggjs/egg-cluster)库中逻辑，实现功能。    

- cluster(clientClass, options)
  [参见](https://eggjs.org/zh-cn/advanced/cluster-client.html)

- BaseContextClass
  重写EggCore.BaseContextClass，区别在于加入this.logger属性，主要是日志可以记录具体的触发写入的代码所在的文件地址，底层还是调用的```ctx.logger```对象。

  ```js
  class BaseContextClass {
    get logger() {
      if (!this[LOGGER]) this[LOGGER] = new BaseContextLogger(this.ctx, this.pathName);
      return this[LOGGER];
    }
  }

  //用法
  module.exports = app => {
    return class extends app.Service {
      constructor(...args){
        super(...args);
        this.pathName = __filename;
      }

      getVersion(){
        this.logger('获取版本');
      }
    }
  }
  ```

- Controller
  重写EggCore.Controller，同BaseContextClass。

- Service
  重写EggCore.Service，同BaseContextClass。

- 执行加载配置逻辑
  ```js
  // 加载所有配置(plugins、config)
  this.loader.loadConfig();

  // 注册app启动后执行函数。
  this.ready(() => this.dumpConfig());

  /**
   * 将当前运行配置(plugin、config)，写入对应的文件，方便调试定位
   * 默认路径在：baseDir/run下面
   * application进程对应application.json
   * agent进程对应agent.json
   */
  dumpConfig() {
    const rundir = this.config.rundir;
    let ignoreList;
    try {
      // support array and set
      ignoreList = Array.from(this.config.dump.ignore);
    } catch (_) {
      ignoreList = [];
    }
    const dumpFile = path.join(rundir, `${this.type}_config.json`);
    try {
      /* istanbul ignore if */
      if (!fs.existsSync(rundir)) fs.mkdirSync(rundir);
      const json = extend(true, {}, { config: this.config, plugins: this.plugins });
      utils.convertObject(json, ignoreList);
      fs.writeFileSync(dumpFile, JSON.stringify(json, null, 2));
    } catch (err) {
      this.coreLogger.warn(`dumpConfig error: ${err.message}`);
    }
  }
  ```

### toJSON()

application对象转json输出，便于调试。

### httpclient

http请求模块，具体参考(urllib)[http://github.com/node-modules/urllib]

### curl(url, opts)

发起http请求，httpclient.request方法别称。

### loggers

日志对象

### getLogger(name)

获取日志对象，默认有 coreLogger、logger、agentLogger。

### logger

logger对象，写入app.config.logger.appLogName配置的文件中，默认为[appInfo.name]-web.log

### coreLogger

coreLogger对象，写入app.config.logger.coreLogName配置的文件中，默认为egg-web.log文件。



