# 从前端视角认识 Rust 编程语言

**编者按：本文由 @祎远 授权分享。**

## 简介

![](https://img.alicdn.com/imgextra/i2/O1CN01MPiHe91a2cebOvget_!!6000000003272-2-tps-770-328.png)

本文站在前端的视角，粗浅地对比了 JS 语言和 Rust 语言在「语言特性」、「设计模式」以及「工程化」方面的异同。希望通过本文，带读者简单一览新时代的编程语言具备哪些设计优势，以及从这些设计中我们可以学到什么。

文中任何一个小结往深入讲都可以有很丰富的内容，但限于篇幅，这里不展开，只是给刚入门 Rust 的前端工程师一个参考。

## 语言特性

### 对象声明


JS 和 Rust 都有 let 和 const 关键字，但含义大不相同。

在 JS 中，let 用来声明一个可变对象并限制其作用域在 let 所在代码块中，const 用来保证对象不被再次赋值，但并非严格意义上的定义“常量”。在同一个作用域中，已经声明过的对象不能再次声明。

```js
// In JS
let foo = [100];
foo = [200];

const foo = [100];
foo = [200]; // Uncaught TypeError: Assignment to constant variable.
foo[0] = 200;

const foo = 100; // Uncaught SyntaxError: Identifier 'foo' has already been declared
```

在 Rust 中，所有对象通过 let 声明，默认为**不可变**，可以添加 **mut** 关键字让对象可变。在同一个作用域中，已经声明过的对象可以再次声明，并且类型允许不同。

```rust
// In Rust
let foo = 100u8;
foo = 200; // error[E0384]: cannot assign twice to immutable variable `foo`

let mut foo = "some text";
foo = "another text";
```

在 Rust 中，const 关键字用于定义一个在编译期就能完全确定的常量。并保证其在运行时无法修改。

```rust
// In Rust
const foo: &'static str = "some constant text at compile time";

const foo: Vec<u32> = vec![123];
// error[E0010]: allocations are not allowed in constants
// error[E0015]: calls in constants are limited to constant functions, tuple structs and tuple variants
```

### 空值处理

JS 中 null 和 undefined 代表空值，编程时必须对空值处理特别小心，因为编码问题导致的导致空指针异常很常见。

```js
// In JS
let foo = null;
foo.bar; // Uncaught TypeError: Cannot read property 'bar' of null

if (foo) {
	foo.bar;
}

foo?.bar;
```

Rust 中通过 **Option** 枚举来管理可选类型，Option 有 Some(T) 或者 None 两个取值。

```rust
// In Rust
struct MyObject {
	bar: u32,
}

let foo: Option<MyObject> = None;

foo.bar; // error[E0609]: no field `bar` on type `Option<MyObject>`

match foo {
	Some(v) => v,
    None => { // handle empty value },
}

if let Some(v) = foo {
	// v
}

// 就不处理，后果自负
foo.unwrap().bar; // panicked at 'called `Option::unwrap()` on a `None` value'
```

利用 Option 这一层包装，推动开发者处理可选对象，可以从根源上消除运行时空指针异常。

### 异常处理

JS 中一般通过 try-catch 捕获错误。

```js
// In JS
try {
  JSON.parse(...);
} catch (err) {
	// handle err
}
```

Rust 中通过 Result 枚举来管理可能存在错误的类型，Result 有 Ok(T) 或者 Err(E) 两个取值。

```rust
// In Rust
let result = json_parse(...);

match result {
	Ok(v) => v,
    Err(err) => // handle error
}

if let Ok(v) = json_parse(...) {
	// v
}

// 就不处理，后果自负
let v = result.unwrap(); // panicked at 'called `Result::unwrap()` on an `Err` value:

json_parse(); // warning: unused `Result` that must be used
```

和 Option 的用法很相似，同样希望开发者有意识地处理异常，并且为自己没有处理的异常负责。

### 参数传递

JS 在传递原始类型时会 Copy 一次，而引用类型均以地址的方式传递，地址复制而指向的内容不复制。

```js
// In JS
function passByRef(arg) {
	arg.foo = 200;
}

const obj = { foo: 100 };
passByRef(obj);
```

Rust 传参时分几种情况：

● 传递「实现了 Copy Trait」的类型时会发生 Copy，原始类型默认实现了 Copy：

```rust
// In Rust
fn pass_by_copy<T: Copy>(arg: T) {

}

pass_by_copy(100u32);
pass_by_copy(String::from("some text")); // error[E0277]: the trait bound `String: Copy` is not satisfied
```

● 通过 Borrow 语义传递借用（引用），这种行为和 JS 是相似的：

```rust
// In Rust
fn pass_by_ref<T>(arg: &T) {

}

let num: u32 = 100;
pass_by_ref(&num);

let string = String::new();
pass_by_ref(string); // error[E0308]: mismatched types
pass_by_ref(&string);
```

● 通过 Move 语义转移所有权（Ownership）

在 Rust 中，一块内存数据有且只能有一个所有者。上述例子中，&string 实际上是将 string 借给了 pass_by_ref，并没有发生所有权转移。

所有权一旦被转移到其他作用域，当前作用域内将无法再次访问：

```rust
// In Rust
let string = String:new();

fn pass_by_ownership<T>(arg: T) {
    // get ownership of arg
    let len = string.len();
	
    // arg dropped here
}

let string = String::new();
pass_by_ownership(string);

let len = string.len(); // error[E0382]: borrow of moved value: `string`

// Q: 为什么 string 被借用了？
// A:
// impl String {
//     pub fn len(&self) -> usize
// }
```

### async / await

JS 和 Rust 都有 async/await 关键字，都用来处理异步任务，但两者底层的实现原理完全不同。

```js
// In JS
async function process() {
	await someHardWork();
}

const promise = process(); // 立即执行
```

Rust 中异步是通过 Future 实现的。

```rust
// In Rust
async fn process() {
	someHardWork().await;
}

let future = process(); // 不执行
future.await; // 开始执行
```

和 JS 相同，Rust 规定 await 关键字必须出现在 async fn 或者 async block 中，那最终由谁来执行异步任务？

```rust
// does async main work?
// error[E0752]: `main` function is not allowed to be `async`
async fn main() {
	// ...
}

use tokio;

fn main() {
    // need a runtime
    tokio::runtime::Builder::new_multi_thread()
        .enable_all()
        .build()
        .unwrap()
        .block_on(async {
            // async code goes here
        })
}

// simplified by Rust macros
#[tokio::main]
async fn main() {
	// ...
}
```

## 编程思想

### OOP vs IOP

JS 中一般使用 class 进行面向对象编程。将要处理的事情当成一个对象，在对象上添加属性或行为。

```js
// In JS
class Person { speak() {} }

class Techer extends Person { teach() {} }

const myTecher = new Techer();
myTecher.speak();
myTecher.teach();
```

而在 Rust 中，既可以使用 OOP 也可以使用 IOP，但 IOP 是语言推荐的方式。这个和 TypeScript 中的 interface 比较相似。

```rust
// In Rust

// OOP
struct Techer {}

impl Techer {
    fn new() -> Self { Self {} }
	fn teach(&self) {}
}

let my_teacher = Techer::new();
my_teacher.teach();


// IOP
trait Speak {
	fn speak(&self);
}

impl Speak for Techer {
	fn speak(&self) {}
}

my_teacher.speak();
```

## 工程相关

### 模块化

模块化是现代编程语言必须具备的能力。

JS 的模块组织方式发展了很多版本，如 AMD、CMD、CJS、ESM，现在最常用的是 CJS 和 ESM。

```js
// CJS
const fs = require('fs');
fs.readFile(...);

module.exports = ...
exports.foo = bar;

// ESM
import fs from 'fs';
fs.readFile(...);

export default xxx;
export { ... };
```

在 Rust 中，通过 use 关键字即可引入，通过 pub 关键字即可导出。

```rust
// In Rust
use std::fs;

fs::read_to_string(...);

pub fn foo() { }
pub struct Object { }
```

### 包管理 NPM vs Cargo

前端的包管理器也有很多，常见的是 npm 和 yarn，每一个包维护一份 package.json。依赖安装后默认自动生成 package-lock.json 文件。依赖版本规范遵循 Semantic Versioning。

```js
// In package.json
{
  "name": "my-package",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "dependencies": {
  	"react": "^16.13.1"
  },
  "devDependencies": {
  	"eslint": "^7.18.0"
  },
  "author": "",
  "license": "ISC"
}


// In package-lock.json
{
  "requires": true,
  "lockfileVersion": 1,
  "dependencies": {
    "ansi-regex": {
      "version": "3.0.0",
      "resolved": "https://registry.npmjs.org/ansi-regex/-/ansi-regex-3.
0.0.tgz",
      "integrity": "sha1-7QMXwyIGT3lGbAKWa922Bas32Zg="
    },
    "cowsay": {
      "version": "1.3.1",
      "resolved": "https://registry.npmjs.org/cowsay/-/cowsay-1.3.1.tgz"
,
      "integrity": "sha512-3PVFe6FePVtPj1HTeLin9v8WyLl+VmM1l1H/5P+BTTDkM
Ajufp+0F9eLjzRnOHzVAYeIYFF5po5NjRrgefnRMQ==",
      "requires": {
        "get-stdin": "^5.0.1",
        "optimist": "~0.6.1",
        "string-width": "~2.1.1",
        "strip-eof": "^1.0.0"
      }
    }
  	// ...
  }
}
```

在 Rust 中，通过 Cargo.toml 文件维护包信息，依赖下载完成后会自动生成 Cargo.lock 文件。依赖版本规范同样遵循 Semantic Versioning。

```rust
// In Cargo.toml

[package]
name = "my-package"
version = "1.0.0"
authors = [""]
description = ""
license = "ISC"
edition = "2018"

[dependencies]
log = "0.4.0" # 同 ^0.4.0

[dev-dependencies]
cmd_lib = "1.2.4"


// In Cargo.lock
# This file is automatically @generated by Cargo.
# It is not intended for manual editing.
version = 3

[[package]]
name = "aead"
version = "0.4.3"
source = "registry+https://github.com/rust-lang/crates.io-index"
checksum = "0b613b8e1e3cf911a086f53f03bf286f52fd7a7258e4fa606f0ef220d39d8877"
dependencies = [
 "generic-array",
]

...
```

### 多包管理 monorepo vs cargo workspace

前端通常需要借助实现了 monorepo 规范的三方工具或者包管理器来进行多包管理，比较常用的就是 Lerna 的实现。

```json
// In lerna.json

{
  "version": "1.1.3",
  "npmClient": "npm",
  "command": {
    "publish": {
      "ignoreChanges": ["ignored-file", "*.md"],
      "message": "chore(release): publish",
      "registry": "https://npm.pkg.github.com"
    },
    "bootstrap": {
      "ignore": "component-*",
      "npmClientArgs": ["--no-package-lock"]
    }
  },
  "packages": ["packages/*"]
}

// 目录结构
packages/
├── foo-pkg
│   └── package.json
├── bar-pkg
│   └── package.json
├── baz-pkg
│   └── package.json
└── qux-pkg
    └── package.json
```

而 Rust 是通过 Cargo WorkSpace 特性来实现的。

```rust
// In Cargo.toml

[workspace]
members = [
    "foo-pkg",
    "bar-pkg",
    "baz-pkg",
    "qux-pkg",
]
exclude = [
    "test-pkg"
]

// 目录结构
./
├── foo-pkg
│   └── Cargo.toml
├── bar-pkg
│   └── Cargo.toml
├── baz-pkg
│   └── Cargo.toml
└── qux-pkg
    └── Cargo.toml
```

### 代码风格 eslint/prettier vs cargo check/format

前端同样需要三方工具，如 eslint 来做静态代码检查、prettier 来做代码格式化。除此之外通常还需要建立对应的一系列配置文件来解决个性化问题。

```bash
$ eslint [options] [file|dir|glob]*
```

而在 Rust 中直接运行 check 或者 format 命令即可。

```bash
$ cargo check
$ cargo format
```

### 代码构建 webpack vs cargo build

前端打包、压缩代码，生成构建产物通常要借助 webpack、rollup 等打包工具以及对应的配置文件。

```js
// webpack.config.js
const path = require('path');

module.exports = {
  entry: './src/index.js',
  output: {
    filename: 'main.js',
    path: path.resolve(__dirname, 'dist'),
  },
};

$ npx webpack --config webpack.config.js
```

在 Rust 中通过 build 命令即可完成。

```bash
$ cargo build
$ cargo build --release
```

### 文档构建 jsdoc vs cargo doc

前端可以借助 jsdoc/typdoc 等工具生成文档，按照工具要求的规范编写注释，即可自动生成文档。

```js
/**
 * Represents a book.
 * @constructor
 * @param {string} title - The title of the book.
 * @param {string} author - The author of the book.
 */
function Book(title, author) {
  
}

$ jsdoc book.js
```

在 Rust 中，也需要按照一定的规范编写注释，然后通过 cargo doc 生成 html 产物。Rust 的注释中可以写入文档测试代码，测试代码可以通过 cargo test --doc 命令执行。

```rust
/// Creates a new `Bytes` from a static slice.
///
/// The returned `Bytes` will point directly to the static slice. There is
/// no allocating or copying.
///
/// # Examples
///
/// ```
/// use bytes::Bytes;
///
/// let b = Bytes::from_static(b"hello");
/// assert_eq!(&b[..], b"hello");
/// ```
#[inline]
#[cfg(not(all(loom, test)))]
pub const fn from_static(bytes: &'static [u8]) -> Bytes {
    Bytes {
        ptr: bytes.as_ptr(),
        len: bytes.len(),
        data: AtomicPtr::new(ptr::null_mut()),
        vtable: &STATIC_VTABLE,
    }
}

$ cargo doc --open
$ cargo test --doc
```

在 CI 过程中包含文档测试，可以保证代码示例的时效性和正确性。

### 单元测试 jest vs cargo test

在 Node.js 中可以使用 assert 模块做断言测试。

```js
// In Node.js
const assert = require('assert').strict;

assert.deepEqual([[[1, 2, 3]], 4, 5], [[[1, 2, '3']], 4, 5]);
```

而前端需要借助各种测试工具库来为单元测试代码提供可以运行的环境，常用的一般是 jest。

```js
// sum.js
function sum(a, b) {
  return a + b;
}
module.exports = sum;

// sum.test.js
const sum = require('./sum');

test('adds 1 + 2 to equal 3', () => {
  expect(sum(1, 2)).toBe(3);
});

$ jest
PASS  ./sum.test.js
✓ adds 1 + 2 to equal 3 (5ms)
```

相反 Rust 就很直接了，可以使用 #[test] 将一个模块或者函数标记为测试，然后通过 cargo test 执行。

```rust
fn sum(a: usize, b: usize) -> usize {
    a + b
}

#[test]
fn test_sum() {
	assert_eq!(sum(1, 2), 3);
}

$ cargo test
running 1 test
test test_sum ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

### 性能测试 benchmark vs cargo bench

前端针对函数的性能测试一般借助前后时间比较，或者 benchmark.js 库来完成。

```js
// ========== In Node.js ==========
const { PerformanceObserver, performance } = require('perf_hooks');

const obs = new PerformanceObserver((items) => {
  console.log(items.getEntries()[0].duration);
  performance.clearMarks();
});
obs.observe({ entryTypes: ['measure'] });
performance.measure('Start to Now');

performance.mark('A');
doSomeLongRunningProcess(() => {
  performance.measure('A to Now', 'A');

  performance.mark('B');
  performance.measure('A to B', 'A', 'B');
});


// ========== In Web Browser ==========
const t0 = performance.now();
for (let i = 0; i < array.length; i++) {
  // some code
}
const t1 = performance.now();
console.log(t1 - t0, 'milliseconds');


// ========== console ==========
console.time('test');
for (let i = 0; i < array.length; i++) {
  // some code
}
console.timeEnd('test');


// ========== benchmark.js ==========
var suite = new Benchmark.Suite;

// add tests
suite.add('RegExp#test', function() {
  /o/.test('Hello World!');
})
.add('String#indexOf', function() {
  'Hello World!'.indexOf('o') > -1;
})
.add('String#match', function() {
  !!'Hello World!'.match(/o/);
})
// add listeners
.on('cycle', function(event) {
  console.log(String(event.target));
})
.on('complete', function() {
  console.log('Fastest is ' + this.filter('fastest').map('name'));
})
// run async
.run({ 'async': true });
```

在 Rust 中只需要使用 #[bench] 标记即可。

```rust
use test::Bencher;

pub fn add_two(a: i32) -> i32 {
    a + 2
}

#[bench]
fn bench_add_two(b: &mut Bencher) {
    b.iter(|| add_two(2));
}

$ cargo bench
running 1 tests
test tests::bench_add_two ... bench:         1 ns/iter (+/- 0)

test result: ok. 0 passed; 0 failed; 0 ignored; 1 measured
```

## 总结&思考

经过上面的对比，不难看出，Rust 无论在语言特性还是工程工具方面，都具备很强的优势。Rust 通过精妙的语言设计，确保代码一致性的同时，也提升了健壮性。工程工具方面，也做到了极致的一体化，一体化不仅为开发者带来了很大的便利，也更有利于整个生态的发展。

回到前端，框架工具的换新率很高，解决相似问题的工具有很多，有人也时常调侃学不动了。我想另一个方面也可能是我们缺少像 Cargo 这样一体化程度很高的工具，还在觉得别人的轮子不好，不断反复造更好的轮子这一过程中。

真希望有一天前端可以不用花太多时间折腾环境、配置工具、啃文档，有更多时间去写出高质量的代码。

