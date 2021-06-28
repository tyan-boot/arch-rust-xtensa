# Archlinux Rust for ESP(xtensa)

provide `PKGBUILD` for [MabezDev/rust-xtensa](https://github.com/MabezDev/rust-xtensa)

## How to build rust toolchain

* download this repo and unzip, [click this](https://github.com/tyan-boot/arch-rust-xtensa/archive/refs/heads/master.zip)
* `cd rust`
* `makepkg -s`
* `pacman -U rust-xtensa-git-*.pkg.tar.zst`

This will build rust toolchain and llvm and install it. You will need at least 10GB ram and 25GB disk to build.

For more about build, refer to [MabezDev/rust-xtensa](https://github.com/MabezDev/rust-xtensa)

__As `MabezDev/rust-xtensa` update llvm upstream to `espressif`, the `llvm` directory is no longer requried, 
you can still build `llvm-project` yourself, and add `--llvm-root=/opt/llvm-xtensa` to `rust/PKGBUILD`, 
but this will break `install` due to the lack of `libunwind` source code in `src/llvm-project/libunwind`, you will need to fix it.__

## Prebuilt

Download from [release](https://github.com/tyan-boot/arch-rust-xtensa/releases).

There is no guarantee that prebuilt package will work due to archlinux update policy. If you encounter error, try build yourself or open an issue.

## How to use toolchain

This package will install rust toolchain and `cargo` in `/opt/rust-xtensa`, you can call `cargo` and `rustc` directly from `/opt/rust-xtensa/bin`.

Or link to `rustup`:

```shell
rustup toolchain link xtensa /opt/rust-xtensa
```

And call with `cargo +xtensa build` or override using `rustup override set xtensa`.

To build to xtensa, use `xtensa-esp32-none-elf`/`xtensa-esp32s2-none-elf`/`xtensa-esp8266-none-elf` target.

### Option 1, cargo-xbuild

Install cargo-xbuild with `cargo install cargo-xbuild`, this tool will help you to compile crate like `core`, `alloc` to xtensa target.

example
```shell
export XARGO_RUST_SRC=/opt/rust-xtensa/lib/rustlib/src/rust/library
cargo xbuild --target xtensa-esp32-none-elf
```


### Option 2, cargo unstable build-std

Cargo also have a unstable feature `build-std`, works similar to `cargo-xbuild`.

example
```shell
cargo build -Z build-std --target xtensa-esp32-none-elf
```

__NOTE__ `build-std` doesn't work for `esp8266` target, but work fine for `esp32` and `esp32s2`

## Tips

1. add `#![no_std]` to your `lib.rs` or `main.rs`.
2. `#![no_std]` doesn't meant to the end of world, you can still use `extern crate core` to add `core` library to your crate.
3. add a panic handler, for examples, `panic-abort`/`panic-halt`.
4. if you need alloc feature, like `vec`, add `extern crate alloc`, and you will need to provide a `GlobalAlloc`, impl it or find a crate.
In my project, i impl it like this:
    ```rust
    use core::alloc::{GlobalAlloc, Layout};
    
    mod ffi {
        extern "C" {
            pub fn malloc(size: usize) -> *mut u8;
        
            pub fn free(ptr: *mut u8);
        }
    }
    
    pub struct IdfAlloc;
    
    unsafe impl GlobalAlloc for IdfAlloc {
        unsafe fn alloc(&self, layout: Layout) -> *mut u8 {
            ffi::malloc(layout.size())
        }
    
        unsafe fn dealloc(&self, ptr: *mut u8, _layout: Layout) {
            ffi::free(ptr);
        }
    }
    
    #[global_allocator]
    static ALLOC: IdfAlloc = IdfAlloc;
    ```
    this need link to `newlib` of `esp-idf` to work.

5. For API like `wifi`, `bluetooth`, `GPIO`, i just write ffi bindings to `esp-idf` and link to them.