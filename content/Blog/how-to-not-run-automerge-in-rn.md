---
title: How to (not) run Automerge in React-native
publish: true
tags:
  - react-native
  - automerge
  - c++
  - android
  - programming
description: My painful journey to get the Automerge library to work in React native
date: 2025-08-02
---
 TL;DR: I don't know how how to run [Automerge](https://automerge.org/) in RN, I've tried [⚛️ 🏎 react-native-webassembly](https://github.com/cawfree/react-native-webassembly) but it's not yet possible due to technical reasons.

There is hope. Callstack published new RN library [callstackincubator/polygen](https://github.com/callstackincubator/polygen) which allows to run webassembly in RN, but I have yet to try it.

## Problem

I wanted to create an app that has one big note, that is shared and synced between my devices. There are two layers to it:
- Presentation: A single markdown formatted note, which can be done using https://github.com/Expensify/react-native-live-markdown
- Storage and sync: There were no obvious answers here, I didn't want to relay on cloud providers. I found a nice library: https://automerge.org/ which has a nice selling point: 
	- Automatic merging: Automerge is a Conflict-Free Replicated Data Type (CRDT
	- Network-agnostic: Allows the use of any connection-orientated protocol
	- Portable: Imlemented in Javascript-WASM and Rust, with FFI bindings

## Choosing the right tool for the job

This WASM part caught my eye since there are libraries that claim to bring WASM to react-native:

[⚛️ 🏎 react-native-webassembly](https://github.com/cawfree/react-native-webassembly) - A c++ turbo module that is a wrapper around [wasm3](https://github.com/wasm3/wasm3) - A fast WebAssembly interpreter and the most universal WASM runtime.

[callstackincubator/polygen](https://github.com/callstackincubator/polygen) - from the callstack website: "Polygen instead performs Ahead-of-time compilation of WebAssembly modules into C/C++ code using the wonderful [`wasm2c`](https://github.com/WebAssembly/wabt/blob/main/wasm2c/README.md) tool. After that, Polygen generates additional glue code so that the compiled WebAssembly module can be used from JavaScript code."

So both projects are essentially wrappers around other libraries. I decided to go with the `react-native-webassembly` since it offered Android support, while polygen (as I'm writing it) iOS only.

## react-native-webassembly

Of course adding this library to the project was not simple `npm install react-native-webassembly` failed to compile

![[Pasted image 20250802231954.png]]

I got somewhat cryptic error:

```
error: error in backend: failed to perform tail call elimination on a call site marked musttail
```

Quick google led me to a [simmilar issue](https://github.com/llvm/llvm-project/issues/56679) on PowerPC, but I'm compiling for android... two searches later I got it: [Musttail is causing crash when compiling wasm3 for android armv7-a](https://github.com/llvm/llvm-project/issues/57069) so I read through the issue and I saw `wasm3` being mentioned

![[Pasted image 20250802232519.png]]

The issue is with the configuration of `wasm3`, I fixed it the same way the @bald-man did, disabling `musttail`. Here's the whole patch, feel free to take it:

```git
diff --git a/node_modules/react-native-webassembly/cpp/m3_config_platforms.h b/node_modules/react-native-webassembly/cpp/m3_config_platforms.h
index 50b86ac..8deaf8d 100644
--- a/node_modules/react-native-webassembly/cpp/m3_config_platforms.h
+++ b/node_modules/react-native-webassembly/cpp/m3_config_platforms.h
@@ -82,7 +82,7 @@
 #  endif
 # endif
 
-# if M3_COMPILER_HAS_ATTRIBUTE(musttail)
+# if M3_COMPILER_HAS_ATTRIBUTE(musttail) && !defined(__arm__)
 #   define M3_MUSTTAIL __attribute__((musttail))
 # else
 #   define M3_MUSTTAIL
```

**Open questions**

- Why this is an issue. https://github.com/llvm/llvm-project/pull/109943 is closed in llvm, did the fix not propagate? No idea
- What are the consequences of this fix, what is the `musttail`?

## wasm

Before I proceed with the steps that I took, I need to explain what is WASM.

WASM - it's a low level assembly language, that has both binary and text format. It runs the code in a sandboxed environment. That means that by default it does not have access to any Web APIs. Those are given by providing imports when loading WASM module.

Here's the example WASM module in WAT (text) format:

```wasm
(module
  (func $i (import "my_namespace" "imported_func") (param i32))
  (func (export "exported_func")
    i32.const 42
    call $i))
```

You can convert any `.wasm` binary into such text format using: https://webassembly.github.io/wabt/demo/wasm2wat/

### How react-native-webassemly loads WASM

In `react-native-webassembly` those imports can be provided like this:

```ts
const module = await WebAssembly.instantiate<{
  exported_func: () => number;
  // ...
}>(bufferSource, {
  // Define the scope of the import functions.
  my_namespace: {
    imported_func: (value: number) => console.error(value),
  },
});
```

Those functions are then passed to `wasm3::module` by the [c++ turbomodule](https://github.com/cawfree/react-native-webassembly/blob/main/cpp/react-native-webassembly.cpp#L385-L412) somwehere in lines [367-411](https://github.com/cawfree/react-native-webassembly/blob/main/cpp/react-native-webassembly.cpp#L367-L411) but I don't quite understand how he gets the callback to call the function he wants (I suspect it has to do something with `_doSomethingWithFunction`). My understanding goes like this:

Imports are passed as callback from the JS side to the `RNWebassembly_instantiate` function.

```ts
reactNativeWebAssembly.RNWebassembly_instantiate({
	iid,
	bufferSource: bufferSourceBase64,
	stackSizeInBytes,
	callback: ({ func, args, module }) => {
		...
		return maybeFunction(...args.map(parseFloat));
	},
});
```

Then on the native side the callback is passed to `m3_LinkRawFunctionEx`:

```cpp
Function callback = params.getProperty(runtime, "callback").asObject(runtime).asFunction(runtime);

std::shared_ptr<facebook::jsi::Function> fn = std::make_shared<facebook::jsi::Function>(std::move(callback));

/* export initialization */
for (u32 i = 0; i < io_module->numFunctions; ++i) {
    ...
          m3_LinkRawFunctionEx(io_module, M3GetResolvedModuleName(f).data(), functionName->data(), signature.data(), &_doSomethingWithFunction, static_cast<void*>(id->data()));
```

And the imported function which needs to be called is called using `_doSomethingWithFunction`:


```cpp
m3ApiRawFunction(_doSomethingWithFunction)
{
	...
    resultDict.setProperty(*context.rt, "module", facebook::jsi::String::createFromUtf8(*context.rt, std::string(_ctx->function->import.moduleUtf8)));
    resultDict.setProperty(*context.rt,   "func", facebook::jsi::String::createFromUtf8(*context.rt, functionName));
    resultDict.setProperty(*context.rt,   "args", ConvertStringArrayToJSIArray(*context.rt, result, length));
    
    Value callResult = originalFunction.call(*context.rt, resultDict);
    ...
}
```

## Loading automerge in RN

### First try

This was my first attempt:

```ts
const AutomergeWasm = require('@/assets/automerge.wasm');
type AutomergeModuleType = typeof import("@automerge/automerge/slim");
const module = await WebAssembly.instantiate<AutomergeModuleType>(AutomergeWasm);
```

Guess what's wrong? I forgot imports 🫠, just for reference here's the error that I've got:

```txt
error unknown value_type
```

not telling me much...

### Second try

Adding the imports was not that straightforward (or at least I did not know a better way). So I've just copied the `web/automerge_wasm.js`and  exported `__wbg_get_imports` which provides those imports for the web. 

```js
function __wbg_get_imports() {
    const imports = {};
    imports.wbg = {};
    imports.wbg.__wbindgen_object_drop_ref = function(arg0) {
        takeObject(arg0);
    };
    ...
```
https://www.npmjs.com/package/@automerge/automerge-wasm

And... again...

```txt
error unknown value_type
```

Side note: To check what imports are needed I put the `.wasm` file into https://webassembly.github.io/wabt/demo/wasm2wat/ and looked at the `import` statements, then a quick `grep -r __wbindgen_object_drop_ref` quickly found the missing functions.

```wasm
  (import "wbg" "__wbindgen_string_get" (func $wasm_bindgen::__wbindgen_string_get::hbef6b8ade2155369 (type $t87)))
```

### Debugging time

How to debug this thing? I've decided to go straight into modifying the `node_modules/react-native-webassembly/cpp/react-native-webassembly.cpp` and then recompiling with `npx expo run:android`, which proved to work nicely.

I added logs and turns out that it didn't even run the 'import' part, it failed on parsing the module:

```cpp
wasm3::module mod = env.parse_module(buffer, decoded.length());
```

So I went to google once again and found this issue: https://github.com/wasm3/wasm3/issues/352

![Reference Types are not supported yet](PublicMedia/vshymanskyy-reference-types.png)

Which is a bummer. 

Question: What are **reference types**? And how do I know that the `.wasm` file has them?

WASM docs states:

> _Reference types_ classify first-class references to objects in the runtime [store](https://webassembly.github.io/reference-types/core/exec/runtime.html#store).
> $$ \text{reftype} ::= \text{funcref}\ |\ \text{externref} $$
> The type [funcref](https://webassembly.github.io/reference-types/core/syntax/types.html#syntax-reftype) denotes the infinite union of all references to [functions](https://webassembly.github.io/reference-types/core/syntax/modules.html#syntax-func), regardless of their [function types](https://webassembly.github.io/reference-types/core/syntax/types.html#syntax-functype).
> The type [externref](https://webassembly.github.io/reference-types/core/syntax/types.html#syntax-reftype) denotes the infinite union of all references to objects owned by the [embedder](https://webassembly.github.io/reference-types/core/intro/overview.html#embedder) and that can be passed into WebAssembly under this type.
> Source: https://webassembly.github.io/reference-types/core/syntax/types.html#syntax-reftype

How to spot a `reftype` in the text format? Per the [WASM spec](https://webassembly.github.io/reference-types/core/text/types.html#reference-types):
 ![[Pasted image 20250803171212.png]]

So i went back to the https://webassembly.github.io/wabt/demo/wasm2wat/ loaded wasm and:

Bingo! 

```wasm
 (module $automerge_wasm.wasm
  (type $t0 (func))
  (type $t1 (func (result i32)))
  (type $t2 (func (result externref)))
  (type $t3 (func (param i32)))
  (type $t4 (func (param i32) (result i32)))
  (type $t5 (func (param i32) (result i32 i32)))
  (type $t6 (func (param i32) (result i32 i32 i32)))
  (type $t7 (func (param i32) (result i64)))
  (type $t8 (func (param i32) (result f64)))
  (type $t9 (func (param i32) (result externref)))
```


## Conclusions

When I started writing this I had no idea what the WASM was and how to load it into react-native. Even though I failed catastrophically this was a nice learning experience. I'm sharing it here as a future reference for myself and to turn those X hours of fighting to run this thing into something concrete.

In the future I'll probably look into some other library like https://github.com/yjs/yjs which also offers CRDTs but this time there are some people which got it to work in react-native https://github.com/yjs/yjs/issues/381 which may be easier and I might actually get some work done 😅

 👀 turns out the reference-types are not supported everywhere and at [some point you had to explicitly enable them](https://github.com/wasm-bindgen/wasm-bindgen/issues/4211) in `wasm-bindgen`.
