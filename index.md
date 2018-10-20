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

### Markdown

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
