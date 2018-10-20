## Calling Cardano-Rust from C

There is an implementation of Cardano written in Rust here: [IOHK Cardano-Rust github](https://github.com/input-output-hk/rust-cardano)

Rust can be compiled to many platforms including Windows, Linux, Android and iOS. You can also write functions in Rust that can be called as if they were C functions. Almost all languages allow you to call C library functions. That means you can call the Cardano code from (almost) any language on (almost) any platform.

Obviously data-type representation is different in different languages. We need to write a “bridging” function in Rust, that converts data, calls the functionality, converts data back. In the case of calling from C, we just need “half” the bridge; we only need Rust to talk to C. If we wanted to call the Rust code from another language X, we would need to also make the X-talking-to-C part.

### The Rust Side

I downloaded the code from github, under the top level directory I added a cardano-test subdirectory. I add the cardano-test workspace to the main Cargo.toml:

```
 [workspace]
 members = [
    "cardano",
    "protocol",
    "storage-units",
    "storage",
    "hermes",
    "cardano-test"
]
[dependencies]
printer = { path = "rust-crypto-wasm" }
```

Under cardano-test I created this Cargo.toml, basically a copy of the one in cardano-c, package name is test:

```
[package]
name = "test"
version = "0.1.0"

[lib]
crate-type = ["staticlib"]

[dependencies]
hex = "0.3.2"
cryptoxide = "0.1"
cbor_event = "1.0"
libc = "*"
serde = "1.0"
serde_derive = "1.0"
rand = "0.4"
serde_json = "*"
unicode-normalization = "0.1.7"
cardano = { path = "../cardano" }
storage-units = { path = "../storage-units" }
```

Under cardano-test I make a src and c subdirectories.
I've picked the base58 encode & decode functions from the Cardano-Rust code as an example.

In src/ I create a new Rust code file b58.rs which has 2 special functions that call the Rust-Cardano code, these functions are callable from C:

```
extern crate libc;
use std::os::raw::{c_char};
use std::{ffi, ptr, slice};
use cardano::util::base58::*;
use std::ffi::CStr;
use std::ffi::CString;
  
#[no_mangle]
pub extern "C"
fn my_b58_encode(p: *const u8, p_size: u32, encoded: *mut c_char) -> i8
{
  let p_slice = unsafe { slice::from_raw_parts(p, p_size as usize) };

  let str = encode(p_slice);  /* CALL */

  let bytes = str.as_bytes();
  let cs = CString::new(bytes).unwrap();  // will fail if bytes has "gap" (null) in sequence
  unsafe {
    libc::strcpy( encoded, cs.as_ptr() );
  }

  return 1;
}

#[no_mangle]
pub extern "C"
fn my_b58_decode(encoded: *const c_char, p: *mut u8) -> i8
{
  let c_str: &CStr = unsafe { CStr::from_ptr(encoded) };
  let tmp_str: &str = c_str.to_str().unwrap();  // will fail if "gap" (null) in string, prevent in caller
  let str: String = tmp_str.to_owned();

  let vec = decode(&str);  /* CALL */

  if vec.is_err() {
    return -1; 
  }
 
  let mut tmp_ptr_u8 = p as *mut u8;

  let mut s: u32 = 0;

  for byte in vec.unwrap() {
    s=s+1;
    unsafe {
      *tmp_ptr_u8 = byte;
      tmp_ptr_u8 = tmp_ptr_u8.offset(1);
    }
  }

  return 1;
}
```

I'm not familiar with Rust, perhaps it could be done better, I don't know. The function my_b58_encode takes a C pointer to size bytes, and a pointer to allocated memory for the output. my_b58_decode takes a pointer to a string, and a pointer to allocated and initialised memory for the output.

I create a second file under src/ called lib.rs:

```
extern crate cardano;

pub mod b58;
pub use b58::*;
```

From cardano-test, I execute “cargo build” to make the library for the target platform. You can see platform list: rustc --print target-list. I am using a mac, so I make it for x86_64-apple-darwin:

```
pusheen: cargo build --target x86_64-apple-darwin
   Compiling libc v0.2.43
   Compiling cbor_event v1.0.0
   ...
```

Building produces the library file under target/ in the parent directory (weird right?). The filename is lib[package_name].a (or .so depending on config/platform), the “package_name” is as specified in Cargo.toml, which controls the build.

```
pusheen: ls -lrt ../target/x86_64-apple-darwin/debug/libtest.a
-rw-r--r--  2 pusheen  staff  18725848 19 Oct 12:41 ../target/x86_64-apple-darwin/debug/libtest.a
```

The new functions can be seen in the library:

```
pusheen: nm ../target/x86_64-apple-darwin/debug/libtest.a | grep my_b58_
0000000000000180 T _my_b58_decode
0000000000000000 T _my_b58_encode
```

### The C Side.

Now we need to call the functions from C. Under cardano-test make a header file cardano.h with the function definitions:

```
int8_t my_b58_encode( const uint8_t *bytes, unsigned int size, char *encoded );
int8_t my_b58_decode( const char *encoded, uint8_t *bytes );
```

For convenience I create a soft link to the library and header file:

```
ln -s ../../target/x86_64-apple-darwin/debug/libtest.a ./libtest.a
```

Then I make a C file with a main function that calls our functions:

```
pusheen: cat my_b58_test.c 
#include 
#include 
#include 
#include "../cardano.h"

int8_t bytes_to_hex( uint8_t*, unsigned int, char *);

int main() {

  char *plain1 = "Hello World!";
  char *ebuffer = (char *)malloc(1000);
  memset(ebuffer,'\0',1000);
 
  printf("\nData to encode: %s \n",plain1);
  my_b58_encode((uint8_t *)plain1,strlen(plain1),ebuffer);

  printf("Encoded: %s \n",ebuffer);

  uint8_t *dbuffer = (uint8_t *)malloc(1000);
  memset(dbuffer,'\0',1000);

  my_b58_decode(ebuffer,(uint8_t *)dbuffer);
   
  printf("Decoded again: %s \n\n",dbuffer);

  free(ebuffer);
  free(dbuffer);
}
```

We can compile, link the binary:

```
gcc -g my_b58_test.c libtest.a 
```

Run the binary:

```
pusheen: ./a.out

Data to encode: Hello World! 
Encoded: 2NEpo7TZRRrLZSi2U 
Decoded again: Hello World! 
```

Checking can be done using this handy [online tool](https://www.browserling.com/tools/base58-encode)

Code for this example is [here](https://github.com/HM999/cardano-rust-c-example)

An observation: Be careful to allocate and free from the same language runtime; if you allocated in Rust you must free in Rust, and vice versa, otherwise you will get strange periodic runtime errors. This sounds simple but may not be, initially I tried creating a Rust vec datatype on the back of data at a C pointer, it worked but occasionally it bombed. Seems like Rust manages the memory for vec internally, how exactly I don't know, but looks like sometimes it is freeing and reallocating. That wouldn't be an issue if your whole program was written in Rust, you'd never know, but in this case it's a seggy.

That is the end of this simple example of calling Cardano-Rust from C. [Next we will build a bridge to Android/Java](https://hm999.github.io/cardano-rust-android-example-doc/) and call the functions from there.
