# Writing WAGI Modules

WAGI is designed to make it easy to write WAGI modules.
You do not need any special language libraries or imports.
You do not need any special developer tooling.
If you can compile your code to `wasm32-wasi`, you can run your code with WAGI.

WAGI takes a simple approach to mapping HTTP requests to WebAssembly modules.
In short, it uses normal inputs and outputs, like environment variables and standard IO, instead of providing a library.
In this regard, WAGI follows the [Common Gateway Interface (CGI) 1.1](https://tools.ietf.org/html/rfc3875) specification.

WAGI handles input and output using a simple set of concepts:

- Environment variables store most of the HTTP information
- If any data was uploaded, it will come into the Wasm module on standard input (STDIN)
- To communicate back to the client, just print to standard output (STDOUT)
- As usual, you can send error information to standard output (STDOUT), and that will be sent to the WAGI log.

## Hello World

To build a WAGI module, you need to do two things:

- Format your output correctly so WAGI can use it as an HTTP response
- Compile your code to `wasm32-wasi` format

At minimum, a WAGI module needs to output a content type header and an empty line.
Usually, we want to also print out some information that will be displayed to the client.

Here is a minimalist "hello world" example written in Rust:

```rust
fn main() {
    println!("Content-Type: text/plain\n");
    println!("hello world");
}
```

In Rust, you can compile the above with `cargo build --target wasm32-wasi --release` and have a WAGI module ready to use!

### Swift Hello World

A Swift version looks like this:

```swift
print("content-type: text/plain\n\n");
print("hello world\n");
```

You will need the [Swift compiler for Wasm](https://swiftwasm.org/) to compile this. Note that
the Swift compiler will automatically generate the `_start()` method for you.

### AssemblyScript

And here is an [AssemblyScript](https://www.assemblyscript.org) version:

```typescript
import "wasi";
import { Console } from "as-wasi";

Console.log("content-type: text-plain");
Console.log(""); // blank line separates headers from body.
Console.log("hello world");
```

Note that the AssemblyScript compiler generates the function body wrapper for you.
For more, check out the AssemblyScript WASI [docs](https://wasmbyexample.dev/examples/wasi-hello-world/wasi-hello-world.assemblyscript.en-us.html).

Behind the scenes, both of these are compiled to a the WASI format, which includes a function named `_start()`.
WAGI calls that function by default (though you can override this behavior in configuration using the `entrypoint` property.)

Finally, you can execute raw WAT (Web Assembly Text) format. Here's the same example written directly in WAT:

```wat
(module
    ;; This is the example Hello World WAT from the documentation at
    ;; https://github.com/bytecodealliance/wasmtime/blob/main/docs/WASI-tutorial.md
    ;;
    ;; It has been adapted to send CGI headers.
    (import "wasi_snapshot_preview1" "fd_write" (func $fd_write (param i32 i32 i32 i32) (result i32)))
    (memory 1)
    (export "memory" (memory 0))

    (data (i32.const 8) "content-type:text/plain\n\nhello world\n")

    (func $main (export "_start")
        (i32.store (i32.const 0) (i32.const 8))
        (i32.store (i32.const 4) (i32.const 37))

        (call $fd_write
            (i32.const 1)
            (i32.const 0)
            (i32.const 1)
            (i32.const 20)
        )
        drop
    )
)
```

WAT does not need to be compiled to a `.wasm` file to execute.

## Accessing HTTP Information

WAGI translates an HTTP request into a series of environment variables, command line arguments, and STDIN.
It expects output to be written to STDOUT. WAGI examines the output and creates an HTTP response based on it.

For a code example, see [this example module](https://github.com/deislabs/env_wagi).
Here we will cover the mechanics of WAGI without talking about a specific language.
In general, you can use your language's built-in libraries to access this information.

Consider the following HTTP request:

```console
$ curl -vvv -H "HOST:foo.example.com" localhost:3000/env?greet=matt\&foo=bar
*   Trying 127.0.0.1...
* TCP_NODELAY set
* Connected to localhost (127.0.0.1) port 3000 (#0)
> GET /env?greet=matt&foo=bar HTTP/1.1
> Host:foo.example.com
> User-Agent: curl/7.64.1
> Accept: */*
```

WAGI will take that information and present it to the module as follows:

### Arguments

The arguments (aka command line options, flags) will contain the path query string as if the command had been typed in like this:

``` console
/env greet=matt foo=bar
```

Many languages give you access to this data using an `args` or `argv` array.
Some may importing special packages.

### Environment Variables

The above request will result in a whole bunch of environment variables being set:

``` console
REMOTE_ADDR = 127.0.0.1
X_MATCHED_ROUTE = /env
HTTP_HOST = foo.example.com
SERVER_PORT = 80
SCRIPT_NAME = /Users/technosophos/Code/Rust/env_wagi/target/wasm32-wasi/release/env_wagi.wasm
CONTENT_LENGTH = 0
CONTENT_TYPE =
TEST_NAME = test value
SERVER_SOFTWARE = WAGI/1
REMOTE_USER =
GATEWAY_INTERFACE = CGI/1.1
SERVER_NAME = foo.example.com
HTTP_USER_AGENT = curl/7.64.1
AUTH_TYPE =
PATH_TRANSLATED = /env
PATH_INFO = /env
HTTP_ACCEPT = */*
SERVER_PROTOCOL = http
REQUEST_METHOD = GET
REMOTE_HOST = localhost
X_FULL_URL = http://foo.example.com/env?greet=matt&foo=bar
QUERY_STRING = greet=matt&foo=bar
```

Most languages provide a convenient way to access environment variables.
WASI provides an implementation of this OS facility (Which is why we require `wasm32-wasi` as the compile target).

### Standard Out

In previous examples we have seen how you can use `println()` or `console.log()` or other high-level functions to send information to the client.

Underneath the hood, WAGI reads the special STDOUT (standard output) file handle and reformats the result to an HTTP response.

### Standard Input

On operations like HTTP POST, clients send data to the server (WAGI), which in turn passes this information to the WAGI module via STDIN (standard input).

Most languages allow you to read from STDIN directly as if it were a file.
Use your language's built-in libraries to access this information.

### Mapped Volumes

If your configuration declares one or more `volumes` mount, the volumes will be attached to your module as a filesystem.

You can use your language's built-in file IO library to work with these files.

Note that _only the volumes specified_ will be available to your module.
You cannot access files that were not provided in the configuration.
Nor can you traverse from a mounted directory to other parts of the filesystem, including the
parent directory.

## More Examples and Demos

- [env_wagi](https://github.com/deislabs/env_wagi): Dump the environment that WAGI sets up, including env vars and args.
- [hello-wagi-as](https://github.com/deislabs/hello-wagi-as): AssemblyScript example using environment variables and query params.

If you want to understand the details, read the [Common Gateway Interface 1.1](https://tools.ietf.org/html/rfc3875) specification.
While this is not an exact implementation, it is very close.
See the "Differences" section below for the differences.
