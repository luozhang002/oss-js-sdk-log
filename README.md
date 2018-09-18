## oss js sdk浏览器端如何收集用户日志
### 背景
开发者使用oss js sdk开发应用产品时，一旦产品面向广大用户，由于产品运行在不同的终端，用户的操作千差万异，用户一旦遇到问题，由于没有相关日志信息，而且很难模拟用户当时的操作环境，问题也就很难定位，站在开发者角度希望有一套方案能够协助解决这种问题。对SDK提供方来说，我们是坚决不收集用户信息的。收集用户信息的责任应该由开发者授权，这里提供一套方案来指导开发者如何收集用户信息。事实上oss js sdk提供的调试信息本身已经暴露相关日志信息。因此在SDK里面把调试信息的接口暴露出来即可。至于如何收集这些信息，如何上传这些信息，上传到什么日志分析平台，开发者完全自主选择。详细方案和使用可参考下面的解决方案。

### 解决方案

在js sdk原来的调试信息里面把调试信息加入级别备注，比如warning 、error、success等，然后挂载到js sdk暴露出的对象原型上，开发者只需要在业务代码中劫持debug函数信息，如何劫持看下文，然再结合第三方日志收集工具比如logline把日志收集到localstoreage 、indexed db 、或者websql中，客户可以根据logline提供的功能，可以选择存储日志的方式、记录日志、读取日志、清除日志等，关于如何上传日志，只需要通过获取日志后，业务方自己写逻辑就可以了，此处建议使用阿里云的STS服务，更加强大、稳定、可靠。

### js sdk改动

在js sdk把debug函数暴露到实例原型上，同时debug函数里面把调试信息加入级别备注，比如warning 、error、success等，目前已经代码提交到6.X系列版本。

### 客户改动

*  ali-oss的版本必须是6.X系列版本。
* 开发者在业务代码里劫持debug函数，重写，获取里面的debug信息，然后可以配合第三方日志工具进行本地存储，比如logine，具体文档可移步：https://github.com/latel/logline

### 示例

下面是基于ali-oss 6.x版本，告诉用户如何收集日志，可控制存储什么级别的日志，如何获取日志为例进行演示，用户获取日志后可自行撰写上传逻辑。注意：js sdk源码中debug函数最后一个参数是控制日志的级别的目前就三个info、waring 、error三种。

#### 第一步 
开发环境中使用6.X系列版本

#### 第二步
劫持debug信息，并调用logine并选择一种方式存储日志信息。代码如下：
```javascript
/ 引入ali-oss 6.X 
const OSS = require('ali-oss');
// 日志收集引入第三方日志收集工具logline
const Logline = require('logline');
// 选择日志协议，web端主要有三种存储方式：websql、indexeddb、localstorage
Logline.using(Logline.PROTOCOL.LOCALSTORAGE）
const sdkLog = new Logline('sdk');
// 劫持OSS.prototype.debug上的日志
const _debug = OSS.prototype.debug;
OSS.prototype.debug = (...params) => {
    console.log(...params);
    // 获取params最后一个参数的数值是info还是error，如果是info就调用sdkLog.info error的话就调用 sdkLog.error
    const info = params[params.length - 1];
    // 解析params参数 ["request %s %s, with headers %j, !!stream: %s", "GET", "http://luozhang002.oss-cn-zhangjiakou.aliyuncs.com/?max-keys=100", {…}, false, "info"]
    // 规则是 %s 或者%s, 替换 字符, %j 或者%j ,替换为对象, 或者把%j对象单独打印出来
    // 解析逻辑 (开发者也可以自己写，这里先提供简单的解析代码，可供参考)
    // 最终要存储的总的信息 params.length的长度约定的至少为2，为2 直接取第一个，其他的长度需要另外处理
   let resultInfo = '';
   if (params.length === 2) {
     resultInfo = params[0];
   } else {
     let k = 1;
     const firstInfo = params[0].split(' ');
     const newInfo = [];
     for (let j = 0; j < firstInfo.length; j++) {
       if (firstInfo[j] === '%s' || firstInfo[j] === '%s,') {
          newInfo.push(params[k]);
          k++;
       } else if (firstInfo[j] === '%j' || firstInfo[j] === '%j,') {
          newInfo.push(JSON.stringify(params[k]));
          k++;
       }  else {
          newInfo.push(firstInfo[j]);
       }
     }
     resultInfo = newInfo.join(' ');
  } // end else
  if (info === 'info') {
    sdkLog.info('info', resultInfo);
  } else {
    sdkLog.error('error', resultInfo);
  }
  // 之前debug的逻辑还是保留的，localstoreage.debug = 'ali-oss'
  _debug(...params);
};
```

#### 第三步 
获取所有日志信息并上传。
```javascript
collect all logs
Logline.all(function(logs) {
    // process logs here这里可以获取到所有的日志是一个大数组，用户可以自行把数据上传到任何地方OSS、STS或者其他日志分析系统
  console.log(logs)
});
// collet logs within .3 days
Logline.get('.3d', function(logs) {    
    // process logs here
});

```
#### 获取到的日志
![log](https://img.alicdn.com/tfs/TB1ITipEY9YBuNjy0FgXXcxcXXa-2844-838.png)

### 总结
总之，在浏览器端客户如果想收集日志信息，开发者需要在业务代码中劫持debug函数信息，再结合第三方日志收集工具比如logline把日志收集到localstoreage 、indexed db 、或者websql中，然后再获取日志。