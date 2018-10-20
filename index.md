## Calling Cardano-Rust from C

There is an implementation of Cardano written in Rust here: [IOHK Cardano-Rust github](https://github.com/input-output-hk/rust-cardano)

Rust can be compiled to many platforms including Windows, Linux, Android and iOS. You can also write functions in Rust that can be called as if they were C functions. Almost all languages allow you to call C library functions. That means you can call the Cardano code from (almost) any language on (almost) any platform.

Obviously data-type representation is different in different languages. We need to write a “bridging” function in Rust, that converts data, calls the functionality, converts data back. In the case of calling from C, we just need “half” the bridge; we only need Rust to talk to C. If we wanted to call the Rust code from another language X, we would need to also make the X-talking-to-C part.

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





Markdown is a lightweight and easy-to-use syntax for styling your writing. It includes conventions for

```markdown
Syntax highlighted code block

# Header 1
## Header 2
### Header 3

- Bulleted
- List

1. Numbered
2. List

**Bold** and _Italic_ and `Code` text

[Link](url) and ![Image](src)
```

For more details see [GitHub Flavored Markdown](https://guides.github.com/features/mastering-markdown/).

### Jekyll Themes

Your Pages site will use the layout and styles from the Jekyll theme you have selected in your [repository settings](https://github.com/HM999/cardano-rust-c-example-doc/settings). The name of this theme is saved in the Jekyll `_config.yml` configuration file.

### Support or Contact

Having trouble with Pages? Check out our [documentation](https://help.github.com/categories/github-pages-basics/) or [contact support](https://github.com/contact) and we’ll help you sort it out.
