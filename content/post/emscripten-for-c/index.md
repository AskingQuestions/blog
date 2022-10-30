---
title: Using Emscripten for porting C libraries to the web/serverless
description: Emscripten is pretty cool for porting C libraries to the web. Here are some notes on how I used it (Mainly written for myself 🙂)
slug: emscripten-for-c
date: 2022-03-06 00:00:00+0000
image: emscripten-for-c.png
categories:
  - Web
tags:
  - Emscripten
  - C
  - WebAssembly
  - Workers
---

Doing this on linux (Ubuntu WSL2) with the latest emscripten built from source.

## Step 1 - Install emscripten

> https://emscripten.org/docs/getting_started/downloads.html

```sh
# Get the emsdk repo
git clone https://github.com/emscripten-core/emsdk.git

# Enter that directory
cd emsdk

# Download and install the latest SDK tools.
./emsdk install latest

# Make the "latest" SDK "active" for the current user. (writes .emscripten file)
./emsdk activate latest

# Activate PATH and other environment variables in the current terminal
source ./emsdk_env.sh
```

## Step 2 - Find/clone your C library

In my case I wanted to port `libredwg` to the a cloudflare worker.

So I cloned the repo:

```sh
git clone your-c-library

cd your-c-library
```

## Step 3 - Build your C library

> https://emscripten.org/docs/compiling/Building-Projects.html

In `libredwg`'s case I needed to `./configure` and `make`. With emscripten we need to use `emconfigure` and `emmake` instead.

```sh
# Run emconfigure with the normal configure command as an argument.
emconfigure ./configure

# Run emmake with the normal make to generate wasm object files.
emmake make
```

## Step 4 - Build your C library to wasm for testing

This is where things get a bit weird. In my case I took an existing program from `libredwg` named `dwg2dxf` and copied it into my own .c file. I also included the necessary headers and linked against the generated .o files in Step 3. I pretty much just manually did what `make` does in order to not have to deal with problems.

The command looks like this:

```sh
emcc dwg2dxf.c ../src/*.o -o dwg2dxf.html -L../  -I../include -I../src -O2 --pre-js testem.js
```

- `dwg2dxf.c` is my own .c file that I copied and modified from `libredwg`'s `dwg2dxf.c`
- `../src/*.o` are the object files generated in Step 3
- `dwg2dxf.html` is the output file
- `-L../` is the path to the library
- `-I../include` is a path to the headers
- `-I../src` is a path to the headers
- `-O2` is the optimization level
- `--pre-js testem.js` is an emscripten-specific option that allows you to run some javascript before the wasm is executed. In my case, I used it to pass args to the compiled program and load files into `MEMFS`.

My `testem.js` looks like this:

```js
var Module = {
  arguments: ["/test.dwg"],
  print: function (text) {
    console.log("stdout: " + text);
  },
  printErr: function (text) {
    console.log("stderr: " + text);
  },
  preRun: function () {
    var data = fs.readFileSync("./test.dwg");
    var stream = FS.open("/test.dwg", "w+");
    FS.write(stream, data, 0, data.length, 0);
    FS.close(stream);
  },
  postRun: function () {
    let stream2 = FS.open("/test.dxf", "r");
    let stat = FS.stat("/test.dxf");
    console.log(stat.size);
    var buf = new Uint8Array(stat.size);
    FS.read(stream2, buf, 0, stat.size, 0);
    FS.close(stream2);

    fs.writeFileSync("./test.dxf", buf);
  },
};
```

When testing you may need to use the following to speed up the node startup time:

```sh
node --liftoff --no-wasm-tier-up test.cjs
```

## Extras

A useful command to view the generated wasm:

```sh
wasm-objdump test.wasm -d > dump.txt
```

> You may have to apt-get install `wabt` to get `wasm-objdump`

## Deploying to a cloudflare worker

Hopefully this process will be easier in the future but for now the high-level steps are:

1. Create a fetch handler

```js
//            v-- This is your emscripten-generated module
import createMyModule from "../lib/dwg2dxf_module.js";

export default {
  async fetch(request, env, ctx) {
    const formData = await request.formData();
    const file = formData.get("file");

    const fileData = await file.arrayBuffer();
    let outputBuf;

    var ModuleInit = {
      arguments: ["/t.dwg"],
      print: function (text) {},
      printErr: function (text) {},
      preRun: function () {
        let FS = ModuleInit.FS;
        var data = Buffer.from(fileData);
        var stream = FS.open("/t.dwg", "w+");
        FS.write(stream, data, 0, data.length, 0);
        FS.close(stream);
      },
      postRun: function () {
        let FS = ModuleInit.FS;
        let stream2 = FS.open("/t.dxf", "r");
        let stat = FS.stat("/t.dxf");
        var buf = new Uint8Array(stat.size);
        FS.read(stream2, buf, 0, stat.size, 0);
        FS.close(stream2);

        outputBuf = buf;
      },
    };
    await createMyModule(ModuleInit);

    return new Response(outputBuf);
  },
};
```

2. Modify your emscripten-generated module to use the `import yourWasm from "your-wasm.wasm"` instead of the native node `fs` module.

```js
// Find var wasmBinary; and replace it with:
var wasmBinary = yourWasm;
```

3. You'll also need to strip out any parts of the generated module that attempt to load the wasm file from the filesystem or network.

4. Add `node_compat = true` to your `wrangler.toml` file.

```toml
node_compat = true
```
