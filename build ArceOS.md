# ArceOS Record

## Ch0 Hello World

Target：让qemu成功跳转到内核入口，执行指令

```rust
// ch0 code framework
.
├── axorigin
│   ├── Cargo.lock
│   ├── Cargo.toml
│   └── src
│       ├── lang_items.rs
│       └── main.rs
├── linker.lds
├── Makefile
├── qemu.log
├── README.md
└── rust-toolchain.toml
```

### 一、启动内核

qemu-riscv启动内核过程

1. PC初始化为0x1000

2. qemu在0x1000预先放置ROM，执行其中代码，跳转到0x8000_0000
3. qemu在0x8000_0000预先放置OpenSBI，由此启动SBI，进行硬件初始化工作，提供功能调用；M-Mode切换到S-Mode，跳转到0x8020_0000
4. 我们将内核加载到0x8020_0000,启动内核

> [!CAUTION]
>
> riscv 体系结构及平台设计的简洁性：
>
> RISC-V SBI 规范定义了平台固件应当具备的功能和服务接口，多数情况下 SBI 本身就可以代替传统上固件 BIOS/UEFI + BootLoader 的位置和作用
>
> // qemu-riscv 启动
> QEMU 启动 → 加载内置 OpenSBI（固件） → 直接跳转至内核入口 → 内核运行
> // qemu-x86 启动
> QEMU 启动 → 加载 SeaBIOS（传统 BIOS） → 加载 GRUB（BootLoader） → 解析配置文件 → 加载内核 → 内核运行

### 二、建立内核程序入口

```toml
// 准备编译工具链
// rust-roolchain.toml
[toolchain]
profile = "minimal"
channel = "nightly"
components = ["rust-src", "llvm-tools-preview", "rustfmt", "clippy"]
targets = ["riscv64gc-unknown-none-elf"]
```

```asm
// 将内核入口放置在image文件开头
// linker.lds
OUTPUT_ARCH(riscv)

BASE_ADDRESS = 0x80200000;

ENTRY(_start)
SECTIONS
{
    . = BASE_ADDRESS;
    _skernel = .;

    .text : ALIGN(4K) {
        _stext = .;
        // 内核入口标记：*(.text.boot)
        *(.text.boot)                      
        *(.text .text.*)
        . = ALIGN(4K);
        _etext = .;
    }
    ...
}
```

```rust
// axorigin/src/main.rs
#![no_std]
#![no_main]
mod lang_items;                                         // 需要实现panic handler处理不可恢复异常

#[unsafe(no_mangle)]                                    // 要求编译器保持_start函数名称
#[unsafe(link_section = ".text.boot")]                  // 将_start放置在链接文件标记处
unsafe extern "C" fn _start() -> ! {
    unsafe {
        core::arch::asm!("wfi", options(noreturn),);    // 嵌入一条汇编
    }
}

// axorigin/src/lang_items.rs
use core::panic::PanicInfo;

#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    loop {}
}
```

```shell
# 编译axorigin
cargo build --manifest-path axorigin/Cargo.toml \
    --target riscv64gc-unknown-none-elf --target-dir ./target --release
# 将编译得到的ELF文件转为二进制格式
rust-objcopy --binary-architecture=riscv64 --strip-all -O binary \
    target/riscv64gc-unknown-none-elf/release/axorigin \
    target/riscv64gc-unknown-none-elf/release/axorigin.bin
# 运行qemu
qemu-system-riscv64 -m 128M -machine virt -bios default -nographic \
    -kernel target/riscv64gc-unknown-none-elf/release/axorigin.bin \
    -D qemu.log -d in_asm
```

---

## Ch1 Hello ArceOS

Target：以组件化方式解耦Unikernel内核，实现输出

```rust
// ch1 code framework
.
├── axconfig
│   ├── Cargo.toml
│   ├── src
│   │   └── lib.rs
│   └── tests
│       └── test_align.rs
├── axhal
│   ├── Cargo.lock
│   ├── Cargo.toml
│   ├── linker.lds
│   └── src
│       ├── lib.rs
│       ├── riscv64
│       │   ├── boot.rs
│       │   ├── console.rs
│       │   └── lang_items.rs
│       └── riscv64.rs
├── axorigin
│   ├── Cargo.lock
│   ├── Cargo.toml
│   └── src
│       └── main.rs
├── axsync
│   ├── Cargo.toml
│   ├── src
│   │   ├── bootcell.rs
│   │   └── lib.rs
│   └── tests
│       └── test_bootcell.rs
├── Cargo.lock
├── Cargo.toml
├── Makefile
├── qemu.log
├── README.md
├── rust-toolchain.toml
└── spinlock
    ├── Cargo.toml
    ├── src
    │   ├── lib.rs
    │   └── raw.rs
    └── tests
        └── test_raw.rs
```

### 一、Unikernel与组件化

Unikernel：单内核，将应用程序与kernel编译为单一image

Monolithic：内核态，用户态（Linux

Microkernel：仅在内核保留最基础功能，其他服务在用户态

组件化：组件作为模块封装功能，提供接口，各个组件构成操作系统的基本元素；以构建 crate 的方式来构建组件，通过 dependencies+features 的方式组合组件->` ArceOS = 组件仓库 + 组合方式 `

### 二、解耦Unikernel

1. 根据功能将Unikernel分为系统层(axhal)和应用层(axorigin)

系统层：封装对体系结构和具体硬件的支持和操作，对外提供硬件操作接口

应用层：实现具体功能，在Ch1中为调用axhal提供的console功能，输出信息

```rust
// 系统层实现内核启动：
// 1. 清零BSS区域
// 2. 建立栈支持函数调用
// 3. 跳转到rust_entry

// axhal/src/boot.rs
#[unsafe(no_mangle)]
#[unsafe(link_section = ".text.boot")]
unsafe extern "C" fn _start() -> ! {
    // a0 = hartid
    // a1 = dtb
    core::arch::asm!("
        la a3, _sbss
        la a4, _ebss
        ble a4, a3, 2f
1:
        sd zero, (a3)
        add a3, a3, 8
        blt a3, a4, 1b
2:

        la      sp, boot_stack_top      // setup boot stack（定义在linker.lds

        la      a2, {entry}
        jalr    a2                      // call rust_entry(hartid, dtb)
        j       .",
        entry = sym super::rust_entry,
        options(noreturn),
    )
}

// 通过axhal的rust_entry跳转到axorigin的main入口
// axhal/src/lib.rs
#![no_std]
#![no_main]

mod lang_items;
mod boot;
pub mod console;

unsafe extern "C" fn rust_entry(_hartid: usize, _dtb: usize) {
    unsafe extern "C" {
        fn main(hartid: usize, dtb: usize);
    }
    main(_hartid, _dtb);
}
```

```rust
// axhal使用SBI提供的功能调用，实现字符输出
// axhal与SBI交互的过程也体现了其作为系统层封装硬件操作，对外提供硬件操作接口的功能
// axhal/src/console.rs
use core::fmt::{Error, Write};

struct Console;

pub fn putchar(c: u8) {
    #[allow(deprecated)]
    sbi_rt::legacy::console_putchar(c as usize);
}

pub fn write_bytes(bytes: &[u8]) {
    for c in bytes {
        putchar(*c);
    }
}

impl Write for Console {
    fn write_str(&mut self, s: &str) -> Result<(), Error> {
        write_bytes(s.as_bytes());
        Ok(())
    }
}

pub fn __print_impl(args: core::fmt::Arguments) {
    Console.write_fmt(args).unwrap();
}

#[macro_export]
macro_rules! ax_print {
    ($($arg:tt)*) => {
        $crate::console::__print_impl(format_args!($($arg)*));
    }
}

#[macro_export]
macro_rules! ax_println {
    () => { $crate::print!("\n") };
    ($($arg:tt)*) => {
        $crate::console::__print_impl(format_args!("{}\n", format_args!($($arg)*)));
    }
}
```

2. 将两个组件进行组合，形成可运行Unikernel

```toml
// axhal/Cargo.toml
[package]
name = "axhal"
version = "0.1.0"
edition = "2021"

[dependencies]
sbi-rt = { version = "0.0.2", features = ["legacy"] }

// axorigin/Cargo.toml  
[package]
name = "axorigin"
version = "0.1.0"
edition = "2021"

[dependencies]
axhal = { path = "../axhal" }

```

### 三、组件测试/模块测试

组件测试相当于 crate 级的测试，直接在 crate 根目录下建立 tests 模块，对组件进行基于公开接口的黑盒测试；模块测试对应于单元测试，在模块内建立子模块tests，对模块的内部方法进行白盒测试

> [!CAUTION]
>
> make run 和 make test编译执行环境不同，make run编译目标为riscv架构，运行在qumu上，make test相当于将测试作为应用直接运行于x86 linux(your pc)上

```makefile
# 互相隔离的环境，test下不能有RUSTFLAGS环境变量
ifeq ($(filter $(MAKECMDGOALS),test),)  # not run `cargo test`
RUSTFLAGS := -C link-arg=-T$(LD_SCRIPT) -C link-arg=-no-pie
export RUSTFLAGS
endif

test:
	cargo test --workspace --exclude "axorigin" -- --nocapture
	
# 为了便于测试，增加测试模块功能：
ifeq ($(filter test test_mod,$(MAKECMDGOALS)),)  # not run `cargo test` 或 `cargo test_mod`
RUSTFLAGS := -C link-arg=-T$(LD_SCRIPT) -C link-arg=-no-pie
export RUSTFLAGS
endif

test_mod:
ifndef MOD
		@printf "    $(YELLOW_C)Error$(END_C): Please specify a module using MOD=<module_name>\n"
		@printf "    Example: make test_mod MOD=axhal\n"
else
		@printf "    $(GREEN_C)Testing$(END_C) module: $(MOD)\n"
		cargo test --package $(MOD) -- --nocapture
endif
```

```toml
# 支持test，工程 Workspace 级的 Cargo.toml
// ArceOS/Cargo.toml
[workspace]
resolver = "2"

members = [
    "axorigin",
    "axhal",
]

[profile.release]
lto = true
```

```rust
// axhal需要屏蔽体系结构差异，原来在riscv64环境下的实现代码应该被隔离
// axhal/src/lib.rs
#![no_std]

#[cfg(target_arch = "riscv64")]
mod riscv64;
#[cfg(target_arch = "riscv64")]
pub use self::riscv64::*;

// axhal/src/riscv64.rs
mod lang_items;
mod boot;
pub mod console;

unsafe extern "C" fn rust_entry(_hartid: usize, _dtb: usize) {
    unsafe extern "C" {
        fn main(hartid: usize, dtb: usize);
    }
    main(_hartid, _dtb);
}

// axhal/Cargo.toml
[target.'cfg(target_arch = "riscv64")'.dependencies]
sbi-rt = { version = "0.0.2", features = ["legacy"] }
```

`axhal`隔离riscv64环境后：

```shell
 Cargo.toml                            | 10 ++++++++++
 Makefile                              |  7 ++++++-
 axhal/Cargo.toml                      |  2 +-
 axhal/src/lib.rs                      | 15 ++++-----------
 axhal/src/riscv64.rs                  | 10 ++++++++++
 axhal/src/{ => riscv64}/boot.rs       |  0
 axhal/src/{ => riscv64}/console.rs    |  0
 axhal/src/{ => riscv64}/lang_items.rs |  0
 8 files changed, 31 insertions(+), 13 deletions(-)
```

### 四、添加组件

1. 全局配置组件 `axconfig`

   ```rust
   // 全局常量 与 工具函数
   // axconfig/src/lib.rs
   #![no_std]
   
   pub const PAGE_SHIFT: usize = 12;
   pub const PAGE_SIZE: usize = 1 << PAGE_SHIFT;
   pub const PHYS_VIRT_OFFSET: usize = 0xffff_ffc0_0000_0000;
   pub const ASPACE_BITS: usize = 39;
   
   pub const SIZE_1G: usize = 0x4000_0000;
   pub const SIZE_2M: usize = 0x20_0000;
   
   #[inline]
   pub const fn align_up(val: usize, align: usize) -> usize {
       (val + align - 1) & !(align - 1)
   }
   #[inline]
   pub const fn align_down(val: usize, align: usize) -> usize {
       (val) & !(align - 1)
   }
   #[inline]
   pub const fn align_offset(addr: usize, align: usize) -> usize {
       addr & (align - 1)
   }
   #[inline]
   pub const fn is_aligned(addr: usize, align: usize) -> bool {
       align_offset(addr, align) == 0
   }
   #[inline]
   pub const fn phys_pfn(pa: usize) -> usize {
       pa >> PAGE_SHIFT
   }
   ```

2. 自旋锁 `SpinRaw`

   ```rust
   // 并没有真正实现自旋锁，目前处于单线程状态不会出现数据争用问题
   // 自旋锁的作用是为了包装mut全局变量，假装实现同步保护
   // spinlock/src/lib.rs
   #![no_std]
   mod raw;
   pub use raw::{SpinRaw, SpinRawGuard}
   
   // spinlock/src/raw.rs
   use core::cell::UnsafeCell;
   use core::ops::{Deref, DerefMut};
   
   pub struct SpinRaw<T> {
       data: UnsafeCell<T>,
   }
   
   pub struct SpinRawGuard<T> {
       data: *mut T,
   }
   
   unsafe impl<T> Sync for SpinRaw<T> {}
   unsafe impl<T> Send for SpinRaw<T> {}
   
   impl<T> SpinRaw<T> {
       #[inline(always)]
       pub const fn new(data: T) -> Self {
           Self {
               data: UnsafeCell::new(data),
           }
       }
   
       #[inline(always)]
       pub fn lock(&self) -> SpinRawGuard<T> {
           SpinRawGuard {
               data: unsafe { &mut *self.data.get() },
           }
       }
   }
   
   impl<T> Deref for SpinRawGuard<T> {
       type Target = T;
       #[inline(always)]
       fn deref(&self) -> &T {
           unsafe { &*self.data }
       }
   }
   
   impl<T> DerefMut for SpinRawGuard<T> {
       #[inline(always)]
       fn deref_mut(&mut self) -> &mut T {
           unsafe { &mut *self.data }
       }
   }
   ```

   > [!NOTE]
   >
   > UnsafeCell<T>是实现 **内部可变性（Interior Mutability）** 的核心机制
   >
   > - **常规规则**：
   >   Rust 默认通过引用规则禁止通过不可变引用（`&T`）修改数据，只能通过可变引用（`&mut T`）修改，且同一作用域内只能存在一个可变引用。
   >
   >   ```rust
   >   let x = 42;
   >   let y = &x;
   >   *y = 10; // 错误：不能通过不可变引用修改数据
   >   ```
   >
   > - **`UnsafeCell` 的魔法**：
   >   通过 `UnsafeCell<T>` 包裹数据，允许在不可变引用（`&UnsafeCell<T>`）的上下文中修改内部数据：
   >
   >   ```rust
   >   use core::cell::UnsafeCell;
   >           
   >   let cell = UnsafeCell::new(42);
   >   let ptr = cell.get(); // 获取 *mut T 裸指针
   >   unsafe { *ptr = 10; } // 允许修改
   >   ```

3. 初始化组件`axsync`

   ```rust
   // BootOnceCell 是对 lazy_static 的替代实现
   // 初始化阶段单线程：仅在启动阶段由单个线程调用 init()
   // 初始化后只读：初始化完成后，所有线程仅通过 get() 读取数据
   // axsync/src/lib.rs
   #![no_std]
   
   mod bootcell;
   pub use bootcell::BootOnceCell;
   
   // axsync/src/bootcell.rs
   use core::cell::OnceCell;
   
   pub struct BootOnceCell<T> {
       inner: OnceCell<T>,
   }
   
   impl<T> BootOnceCell<T> {
       pub const fn new() -> Self {
           Self {
               inner: OnceCell::new()
           }
       }
   
       pub fn init(&self, val: T) {
           let _ = self.inner.set(val);
       }
   
       pub fn get(&self) -> &T {
           self.inner.get().unwrap()
       }
   
       pub fn is_init(&self) -> bool {
           self.inner.get().is_some()
       }
   }
   
   unsafe impl<T> Sync for BootOnceCell<T> {}
   ```
   
---

## Ch2 内存管理1

Target：组织Unikernel为四层系统架构，引入虚拟内存和内存分配

```shell
# Ch2 Code Framework
.
├── axalloc
│   ├── Cargo.toml
│   └── src
│       ├── early
│       │   └── tests.rs
│       ├── early.rs
│       └── lib.rs
├── axconfig
│   ├── Cargo.toml
│   ├── src
│   │   └── lib.rs
│   └── tests
│       └── test_align.rs
├── axhal
│   ├── Cargo.lock
│   ├── Cargo.toml
│   ├── linker.lds
│   └── src
│       ├── lib.rs
│       ├── riscv64
│       │   ├── boot.rs
│       │   ├── console.rs
│       │   ├── lang_items.rs
│       │   └── paging.rs
│       └── riscv64.rs
├── axorigin
│   ├── Cargo.lock
│   ├── Cargo.toml
│   └── src
│       └── main.rs
├── axruntime
│   ├── Cargo.toml
│   └── src
│       └── lib.rs
├── axstd
│   ├── Cargo.toml
│   └── src
│       └── lib.rs
├── axsync
│   ├── Cargo.toml
│   ├── src
│   │   ├── bootcell.rs
│   │   └── lib.rs
│   └── tests
│       └── test_bootcell.rs
├── Cargo.lock
├── Cargo.toml
├── Makefile
├── page_table
│   ├── Cargo.toml
│   ├── src
│   │   └── lib.rs
│   └── tests
│       └── test_ealy.rs
├── qemu.log
├── README.md
├── rust-toolchain.toml
└── spinlock
    ├── Cargo.toml
    ├── src
    │   ├── lib.rs
    │   └── raw.rs
    └── tests
        └── test_raw.rs
```

### 一、内核框架构建

原来的系统只有系统层axhal和应用层axorigin，我们需要另外增加一层管理组织组件的axstd层，它负责了应用和系统的隔离，抽象了系统为应用提供的功能；另外axhal负责了与硬件体系相关的工作，另外一些组件的初始化（硬件体系无关）需要一个位于axhal之上的抽象层来负责，因此引入了axruntime提供运行时管理

```rust
// 启动过程axhal -> axruntime -> axorigin (axstd作为axorigin的依赖载入)
// axhal/src/riscv64.rs
mod lang_items;
mod boot;
pub mod console;
mod paging;

unsafe extern "C" fn rust_entry(hartid: usize, dtb: usize) {
    extern "C" {
        fn rust_main(hartid: usize, dtb: usize);
    }
    rust_main(hartid, dtb);
}

// axruntime/src/lib.rs
#![no_std]

pub use axhal::ax_println as println;

#[no_mangle]
pub extern "C" fn rust_main(_hartid: usize, _dtb: usize) -> ! {
    extern "C" {
        fn _skernel();
        fn main();
    }

    println!("\nArceOS is starting ...");
    // We reserve 2M memory range [0x80000000, 0x80200000) for SBI,
    // but it only occupies ~194K. Split this range in half,
    // requisition the higher part(1M) for early heap.
    axalloc::early_init(_skernel as usize - 0x100000, 0x100000);    // 加载组件启用内存分配器
    unsafe { main(); }
    loop {}
}

// axorigin/src/main.rs
#![no_std]
#![no_main]

use axstd::{String, println};

#[no_mangle]
pub fn main(_hartid: usize, _dtb: usize) {
    let s = String::from("from String");
    println!("\nHello, ArceOS![{}]", s);
}
```

### 二、引入分页机制

为什么需要虚拟内存和分页机制：物理地址空间是硬件平台生产构造时就已经确定的，而虚拟地址空间则是内核可以根据实际需要灵活定义和实时改变的，这是将来内核很多重要机制的基础。按照近年来流行的说法，分页机制赋予了内核“软件定义”地址空间的能力。

内核可以开启CPU的MMU的分页机制，通过MMU提供的地址映射功能，使内核见到的是虚拟地址空间

我们启用Sv39分页机制，这是rCore就涉及的内容，不多赘述

```rust
// page_table/src/lib.rs
#![no_std]
/*
 * RiscV64 PTE format:
 * | XLEN-1  10 | 9             8 | 7 | 6 | 5 | 4 | 3 | 2 | 1 | 0
 *       PFN      reserved for SW   D   A   G   U   X   W   R   V
 */
use axconfig::{phys_pfn, PAGE_SHIFT, ASPACE_BITS};

const _PAGE_V : usize = 1 << 0;     /* Valid */
const _PAGE_R : usize = 1 << 1;     /* Readable */
const _PAGE_W : usize = 1 << 2;     /* Writable */
const _PAGE_E : usize = 1 << 3;     /* Executable */
const _PAGE_U : usize = 1 << 4;     /* User */
const _PAGE_G : usize = 1 << 5;     /* Global */
const _PAGE_A : usize = 1 << 6;     /* Accessed (set by hardware) */
const _PAGE_D : usize = 1 << 7;     /* Dirty (set by hardware)*/

const PAGE_TABLE: usize = _PAGE_V;
pub const PAGE_KERNEL_RO: usize = _PAGE_V | _PAGE_R | _PAGE_G | _PAGE_A | _PAGE_D;
pub const PAGE_KERNEL_RW: usize = PAGE_KERNEL_RO | _PAGE_W;
pub const PAGE_KERNEL_RX: usize = PAGE_KERNEL_RO | _PAGE_E;
pub const PAGE_KERNEL_RWX: usize = PAGE_KERNEL_RW | _PAGE_E;

#[derive(Debug)]
pub enum PagingError {}
pub type PagingResult<T = ()> = Result<T, PagingError>;
const PAGE_PFN_SHIFT: usize = 10;
const ENTRIES_COUNT: usize = 1 << (PAGE_SHIFT - 3);

#[derive(Clone, Copy)]
#[repr(transparent)]
pub struct PTEntry(u64);

impl PTEntry {
    pub fn set(&mut self, pa: usize, flags: usize) {
        self.0 = Self::make(phys_pfn(pa), flags);
    }

    fn make(pfn: usize, prot: usize) -> u64 {
        ((pfn << PAGE_PFN_SHIFT) | prot) as u64
    }
}

pub struct PageTable<'a> {
    level: usize,
    table: &'a mut [PTEntry],
}

impl PageTable<'_> {
    pub fn init(root_pa: usize, level: usize) -> Self {
        let table = unsafe {
            core::slice::from_raw_parts_mut(root_pa as *mut PTEntry, ENTRIES_COUNT)
        };
        Self { level, table }
    }

    const fn entry_shift(&self) -> usize {
        ASPACE_BITS - (self.level + 1) * (PAGE_SHIFT - 3)
    }
    const fn entry_size(&self) -> usize {
        1 << self.entry_shift()
    }
    pub const fn entry_index(&self, va: usize) -> usize {
        (va >> self.entry_shift()) & (ENTRIES_COUNT - 1)
    }

    pub fn map(&mut self, mut va: usize, mut pa: usize,
        mut total_size: usize, best_size: usize, flags: usize
    ) -> PagingResult {
        let entry_size = self.entry_size();
        while total_size >= entry_size {
            let index = self.entry_index(va);
            if entry_size == best_size {
                self.table[index].set(pa, flags);
            } else {
                let mut pt = self.next_table_mut(index)?;
                pt.map(va, pa, entry_size, best_size, flags)?;
            }
            total_size -= entry_size;
            va += entry_size;
            pa += entry_size;
        }
        Ok(())
    }

    fn next_table_mut(&mut self, _index: usize) -> PagingResult<PageTable> {
        unimplemented!();
    }
}
```

实现组件page_table后，需要在axhal层init时映射内核空间，并使mmu启用分页

```rust
// axhal/src/riscv64/paging.rs
use axconfig::{SIZE_1G, phys_pfn};
use page_table::{PAGE_KERNEL_RWX, PageTable};
use riscv::register::satp;

unsafe extern "C" {
    unsafe fn boot_page_table();
}

pub unsafe fn init_boot_page_table() {
    let mut pt: PageTable = PageTable::init(boot_page_table as usize, 0);
    let _ = pt.map(0x8000_0000, 0x8000_0000, SIZE_1G, SIZE_1G, PAGE_KERNEL_RWX);
    let _ = pt.map(
        0xffff_ffc0_8000_0000,
        0x8000_0000,
        SIZE_1G,
        SIZE_1G,
        PAGE_KERNEL_RWX,
    );
}

pub unsafe fn init_mmu() {
    let page_table_root = boot_page_table as usize;
    unsafe {
        satp::set(satp::Mode::Sv39, 0, phys_pfn(page_table_root));
        riscv::asm::sfence_vma_all();
    }
}

// boot时调用函数初始化
// axhal/src/riscv64/boot.rs
use crate::riscv64::paging;

#[no_mangle]
#[link_section = ".text.boot"]
unsafe extern "C" fn _start() -> ! {
    // a0 = hartid
    // a1 = dtb
    core::arch::asm!("
        mv      s0, a0                  // save hartid
        mv      s1, a1                  // save DTB pointer
        la a3, _sbss
        la a4, _ebss
        ble a4, a3, 2f
1:
        sd zero, (a3)
        add a3, a3, 8
        blt a3, a4, 1b
2:
        la      sp, boot_stack_top      // setup boot stack
        call    {init_boot_page_table}  // setup boot page table
        call    {init_mmu}              // enabel MMU
        li      s2, {phys_virt_offset}  // fix up virtual high address
        add     sp, sp, s2              // readjust stack address
        mv      a0, s0                  // restore hartid
        mv      a1, s1                  // restore DTB pointer
        la      a2, {entry}
        add     a2, a2, s2              // readjust rust_entry address
        jalr    a2                      // call rust_entry(hartid, dtb)
        j       .",
        init_boot_page_table = sym paging::init_boot_page_table,
        init_mmu = sym paging::init_mmu,
        phys_virt_offset = const axconfig::PHYS_VIRT_OFFSET,
        entry = sym super::rust_entry,
        options(noreturn),
    )
}
```

### 三、启用动态内存分配

在第一部分的框架代码中我们使用了String，但是String的实现需要动态内存分配支持，这本来是rust std库提供的功能，我们需要实现内核级的动态内存堆管理的支持，因此我们引入了axalloc组件，它的初始化在axruntime中完成。

全局内存分配器 GlobalAllocator 实现 GlobalAlloc Trait，它包含两个功能：字节分配和页分配，分别用于响应对应请求。区分两种请求的策略是，请求分配的大小是页大小的倍数且按页对齐，就视作申请页；否则就是按字节申请分配。我们在GlobalAllocator的下层实现一个early allocator提供底层内存分配的抽象封装

```rust
// axalloc/src/lib.rs
#![no_std]
use axconfig::PAGE_SIZE;
use core::alloc::Layout;
use core::ptr::NonNull;
use spinlock::SpinRaw;

extern crate alloc;
use alloc::alloc::GlobalAlloc;

mod early;
use early::EarlyAllocator;

#[derive(Debug)]
pub enum AllocError {
    InvalidParam,
    MemoryOverlap,
    NoMemory,
    NotAllocated,
}
pub type AllocResult<T = ()> = Result<T, AllocError>;

// 通过 #[global_allocator] 属性将 GlobalAllocator 注册为全局分配器，使 String 等类型默认使用它
#[cfg_attr(not(test), global_allocator)]
static GLOBAL_ALLOCATOR: GlobalAllocator = GlobalAllocator::new();

struct GlobalAllocator {
    // 使用实现的自旋锁组件，防止争用
    early_alloc: SpinRaw<EarlyAllocator>,
}

impl GlobalAllocator {
    pub const fn new() -> Self {
        Self {
            early_alloc: SpinRaw::new(EarlyAllocator::uninit_new()),
        }
    }

    pub fn early_init(&self, start: usize, size: usize) {
        self.early_alloc.lock().init(start, size)
    }
}

impl GlobalAllocator {
    fn alloc_bytes(&self, layout: Layout) -> *mut u8 {
        if let Ok(ptr) = self.early_alloc.lock().alloc_bytes(layout) {
            ptr.as_ptr()
        } else {
            alloc::alloc::handle_alloc_error(layout)
        }
    }
    fn dealloc_bytes(&self, ptr: *mut u8, layout: Layout) {
        self.early_alloc
            .lock()
            .dealloc_bytes(NonNull::new(ptr).expect("dealloc null ptr"), layout)
    }
    fn alloc_pages(&self, layout: Layout) -> *mut u8 {
        if let Ok(ptr) = self.early_alloc.lock().alloc_pages(layout) {
            ptr.as_ptr()
        } else {
            alloc::alloc::handle_alloc_error(layout)
        }
    }
    fn dealloc_pages(&self, _ptr: *mut u8, _layout: Layout) {
        unimplemented!();
    }
}

unsafe impl GlobalAlloc for GlobalAllocator {
    unsafe fn alloc(&self, layout: Layout) -> *mut u8 {
        if layout.size() % PAGE_SIZE == 0 && layout.align() == PAGE_SIZE {
            self.alloc_pages(layout)
        } else {
            self.alloc_bytes(layout)
        }
    }

    unsafe fn dealloc(&self, ptr: *mut u8, layout: Layout) {
        if layout.size() % PAGE_SIZE == 0 && layout.align() == PAGE_SIZE {
            self.dealloc_pages(ptr, layout)
        } else {
            self.dealloc_bytes(ptr, layout)
        }
    }
}

pub fn early_init(start: usize, len: usize) {
    GLOBAL_ALLOCATOR.early_init(start, len)
}
```

底层内存分配器

> [!TIP]
>
> `Layout` 是 Rust 标准库中定义的一个结构体，用于描述内存分配的 **布局要求**，包含两个核心字段：
>
> - **`size: usize`**：需要分配的内存块大小。
> - **`align: usize`**：内存块的最小对齐要求
>
> ``` 
> // 分配一个 16 字节、按 8 字节对齐的内存块
> let layout = Layout::from_size_align(16, 8).unwrap();
> ```
>
> ##### `byte_pos` 和 `page_pos` 的意义
>
> `EarlyAllocator` 通过两个指针管理内存区域：
>
> - **`byte_pos`**：从 **低地址向高地址** 分配小粒度内存（字节级）。
> - **`page_pos`**：从 **高地址向低地址** 分配大粒度内存（页级）
>
> **设计特点**：
>
> - 仅在所有字节分配都被释放时（`count == 0`），才将 `byte_pos` 重置到 `start`。
> - 不支持部分释放，适用于 **批量分配后一次性释放** 的场景（如临时缓冲区）。
> - page_dealloc为支持

```rust
// axalloc/src/early.rs
#![allow(dead_code)]

#[cfg(test)]
mod tests;

use crate::{AllocError, AllocResult};
use axconfig::{PAGE_SIZE, align_down, align_up};
use core::alloc::Layout;
use core::ptr::NonNull;

#[derive(Default)]
pub struct EarlyAllocator {
    start: usize,
    end: usize,
    count: usize,
    byte_pos: usize,
    page_pos: usize,
}

impl EarlyAllocator {
    pub fn init(&mut self, start: usize, size: usize) {
        self.start = start;
        self.end = start + size;
        self.byte_pos = start;
        self.page_pos = self.end;
    }
    pub const fn uninit_new() -> Self {
        Self {
            start: 0,
            end: 0,
            count: 0,
            byte_pos: 0,
            page_pos: 0,
        }
    }
}

impl EarlyAllocator {
    pub fn alloc_pages(&mut self, layout: Layout) -> AllocResult<NonNull<u8>> {
        assert_eq!(layout.size() % PAGE_SIZE, 0);
        let next = align_down(self.page_pos - layout.size(), layout.align());
        if next <= self.byte_pos {
            alloc::alloc::handle_alloc_error(layout)
        } else {
            self.page_pos = next;
            NonNull::new(next as *mut u8).ok_or(AllocError::NoMemory)
        }
    }

    pub fn total_pages(&self) -> usize {
        (self.end - self.start) / PAGE_SIZE
    }
    pub fn used_pages(&self) -> usize {
        (self.end - self.page_pos) / PAGE_SIZE
    }
    pub fn available_pages(&self) -> usize {
        (self.page_pos - self.byte_pos) / PAGE_SIZE
    }
}

impl EarlyAllocator {
    pub fn alloc_bytes(&mut self, layout: Layout) -> AllocResult<NonNull<u8>> {
        let start = align_up(self.byte_pos, layout.align());
        let next = start + layout.size();
        if next > self.page_pos {
            alloc::alloc::handle_alloc_error(layout)
        } else {
            self.byte_pos = next;
            self.count += 1;
            NonNull::new(start as *mut u8).ok_or(AllocError::NoMemory)
        }
    }

    pub fn dealloc_bytes(&mut self, _ptr: NonNull<u8>, _layout: Layout) {
        self.count -= 1;
        if self.count == 0 {
            self.byte_pos = self.start;
        }
    }

    fn total_bytes(&self) -> usize {
        self.end - self.start
    }
    fn used_bytes(&self) -> usize {
        self.byte_pos - self.start
    }
    fn available_bytes(&self) -> usize {
        self.page_pos - self.byte_pos
    }
}
```

## Ch3 Basic Component

Target:增加基础组件，扩展子系统

```shell
 # Code 
 Cargo.lock                               |  42 ++++++++++++++++++++++++++++++++++++++
 Cargo.toml                               |   2 +-
 axconfig/src/lib.rs                      |   8 ++++++++
 axdtb/Cargo.toml                         |   7 +++++++
 axdtb/src/lib.rs                         | 118 ++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
 axdtb/src/util.rs                        |  42 ++++++++++++++++++++++++++++++++++++++
 axdtb/tests/sample.dtb                   | Bin 0 -> 326 bytes
 axdtb/tests/sample.dts                   |  18 +++++++++++++++++
 axdtb/tests/test_dtb.rs                  |  51 ++++++++++++++++++++++++++++++++++++++++++++++
 axhal/Cargo.toml                         |   2 ++
 axhal/src/riscv64.rs                     |  16 +++++++++++++++
 axhal/src/riscv64/lang_items.rs          |   6 ++++--
 axhal/src/riscv64/misc.rs                |   4 ++++
 axhal/src/riscv64/time.rs                |  23 +++++++++++++++++++++
 axlog/Cargo.toml                         |   8 ++++++++
 axlog/src/lib.rs                         |  89 ++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
 axorigin/src/main.rs                     |  10 +++++++--
 axruntime/Cargo.toml                     |   3 +++
 axruntime/src/lib.rs                     | 118 ++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++--
 axstd/Cargo.toml                         |   1 +
 axstd/src/lib.rs                         |   2 ++
 axstd/src/time.rs                        |  31 ++++++++++++++++++++++++++++
 buddy_allocator/Cargo.toml               |   6 ++++++
 buddy_allocator/src/lib.rs               |   3 +++
 buddy_allocator/src/linked_list.rs       | 113 +++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
 buddy_allocator/src/linked_list/tests.rs |  30 +++++++++++++++++++++++++++
 26 files changed, 746 insertions(+), 7 deletions(-)
```



### 一、打破循环依赖

我们希望在ch3引入axlog日志组件，但是这会出现循环依赖，无法通过编译：组件 axruntime 在初始化时，将会初始化 axhal 和 axlog 这两个组件。对于 axhal 和 axlog 这两个组件来说，一方面，axhal 组件需要日志功能，所以依赖 axlog 组件；与此同时，axlog 必须依赖 axhal 提供的标准输出或写文件功能以实现日志输出，所以 axlog 又反过来依赖 axhal。这就在二者之间形成了循环依赖。

我们使用extern ABI的方式声明外部函数，并在crate中直接调用避免循环依赖；创建组件crate_interface使用过程宏封装extern ABI来对这种方式提供抽象。

```rust
// 通过crate_interface提供的宏解决循环依赖问题
// axlog/src/lib.rs
// 定义跨crate接口Logif
#[crate_interface::def_interface]
pub trait LogIf {
    fn write_str(s: &str);
    fn get_time() -> core::time::Duration;
}

// 在 axhal 中定义 LogIfImpl 实现 LogIf 接口
// /axhal/src/riscv64.rs
struct LogIfImpl;

#[crate_interface::impl_interface]
impl axlog::LogIf for LogIfImpl {
    fn write_str(s: &str) {
        console::write_bytes(s.as_bytes());
    }

    fn get_time() -> core::time::Duration {
        time::current_time()
    }
}

// 通过宏 call_interface!(LogIf::XXX) 调用 LogIf 的实现
// axlog/src/lib.rs
pub fn init() {
    extern crate alloc;

    let now = crate_interface::call_interface!(LogIf::get_time());
    let s = alloc::format!("Logging startup time: {}.{:06}",
        now.as_secs(), now.subsec_micros());
    crate_interface::call_interface!(LogIf::write_str(s.as_str()));
}
```

```rust
// 第一次接触过程宏，详细看看如何自定义过程宏
// extern/crate_interface
#![doc = include_str!("../README.md")]  // 将 README.md 内容作为 crate 文档
use proc_macro::TokenStream;           // 过程宏输入/输出的标准类型
use proc_macro2::Span;                 // 用于追踪代码位置
use quote::{format_ident, quote};      // 代码生成工具
use syn::parse::{Error, Parse, ParseStream, Result}; // 解析宏输入的工具
use syn::punctuated::Punctuated;       // 带分隔符的列表（如逗号分隔的参数）
use syn::{parenthesized, parse_macro_input, Token}; // 解析语法结构的工具
use syn::{Expr, FnArg, ImplItem, ImplItemFn, ItemImpl, ItemTrait, Path, PathArguments, PathSegment, TraitItem, Type}; // 各类 AST 节点类型

// 将 syn 的 Error 转换为编译器错误输出
fn compiler_error(err: Error) -> TokenStream {
    err.to_compile_error().into() // 将错误转为可编译的 TokenStream
}

#[proc_macro_attribute]
pub fn def_interface(attr: TokenStream, item: TokenStream) -> TokenStream {
    if !attr.is_empty() { // 检查属性参数是否为空（如 #[def_interface] 不能带参数）
        return compiler_error(Error::new(
            Span::call_site(),
            "expect an empty attribute: `#[def_interface]`",
        ));
    }
    let ast = syn::parse_macro_input!(item as ItemTrait); // 解析输入为 Trait 的 AST
    let trait_name = &ast.ident; // 获取 Trait 名称（如 MyTrait）
    let vis = &ast.vis;         // 获取 Trait 的可见性（如 pub）
    let mut extern_fn_list = vec![]; // 存储生成的外部函数声明
    for item in &ast.items { // 遍历 Trait 中的每个项（如方法）
        if let TraitItem::Fn(method) = item { // 仅处理方法项
            let mut sig = method.sig.clone(); // 复制方法签名（如 fn my_method(arg: u32)）
            let fn_name = &sig.ident;          // 获取方法名（如 my_method）
            sig.ident = format_ident!("__{}_{}", trait_name, fn_name); // 生成唯一函数名，如 __MyTrait_my_method
            sig.inputs = syn::punctuated::Punctuated::new(); // 清空输入参数（后续重新填充）
            // 仅保留类型参数（移除 self 等接收者）
            for arg in &method.sig.inputs {
                if let FnArg::Typed(_) = arg { // 过滤掉 Receiver（如 &self）
                    sig.inputs.push(arg.clone());
                }
            }
            // 生成外部函数声明代码块
            let extern_fn = quote! { pub #sig; };
            extern_fn_list.push(extern_fn);
        }
    }
    let mod_name = format_ident!("__{}_mod", trait_name); // 生成模块名，如 __MyTrait_mod
    quote! { // 生成最终代码
        #ast // 原样保留 Trait 定义
        #[doc(hidden)] // 隐藏生成的模块
        #[allow(non_snake_case)] // 允许非蛇形命名
        #vis mod #mod_name { // 定义模块
            use super::*;     // 引入父模块内容
            extern "Rust" {   // 声明外部函数
                #(#extern_fn_list)* // 插入所有生成的外部函数
            }
        }
    }
    .into() // 转为 TokenStream
}

#[proc_macro_attribute]
pub fn impl_interface(attr: TokenStream, item: TokenStream) -> TokenStream {
    if !attr.is_empty() { // 检查属性参数是否为空
        return compiler_error(Error::new_spanned(...));
    }
    let mut ast = syn::parse_macro_input!(item as ItemImpl); // 解析为 Impl 块 AST
    // 提取 Trait 名称（如 impl MyTrait for MyStruct 中的 MyTrait）
    let trait_name = if let Some((_, path, _)) = &ast.trait_ {
        &path.segments.last().unwrap().ident
    } else { ... };
    // 提取结构体名称（如 MyStruct）
    let impl_name = if let Type::Path(path) = &ast.self_ty.as_ref() {
        path.path.get_ident().unwrap()
    } else { ... };
        for item in &mut ast.items { // 遍历 Impl 块中的每个方法
        if let ImplItem::Fn(method) = item {
            let (attrs, vis, sig, stmts) = // 解构方法属性、可见性、签名和语句
                (&method.attrs, &method.vis, &method.sig, &method.block.stmts);
            let fn_name = &sig.ident; // 方法名（如 my_method）
            let extern_fn_name = format_ident!("__{}_{}", trait_name, fn_name); // 生成外部函数名
            let mut new_sig = sig.clone(); // 复制方法签名
            new_sig.ident = extern_fn_name.clone(); // 修改函数名为外部名称
            new_sig.inputs = Punctuated::new(); // 清空参数（后续填充）
                        let mut args = vec![]; // 存储参数名（如 arg1, arg2）
            let mut has_self = false; // 是否包含 self 参数
            for arg in &sig.inputs { // 遍历方法参数
                match arg {
                    FnArg::Receiver(_) => has_self = true, // 检测 self
                    FnArg::Typed(ty) => { // 处理类型参数
                        args.push(ty.pat.clone()); // 提取参数名（如 arg）
                        new_sig.inputs.push(arg.clone()); // 添加到新签名
                    }
                }
            }
            // 生成调用实际方法的代码（区分有无 self）
            let call_impl = if has_self {
                quote! { let _impl: #impl_name = #impl_name; _impl.#fn_name(#(#args),*) }
            } else {
                quote! { #impl_name::#fn_name(#(#args),*) }
            };
                        let item = quote! { // 生成新的方法实现
                #[inline]
                #(#attrs)* // 保留原属性（如 #[cfg]）
                #vis
                #sig
                {
                    { // 内部块定义外部函数
                        #[inline]
                        #[export_name = #extern_fn_name] // 强制符号名
                        extern "Rust" #new_sig { // 定义外部函数
                            #call_impl // 调用实际方法
                        }
                    }
                    #(#stmts)* // 原方法语句
                }
            }
            .into();
            *method = syn::parse_macro_input!(item as ImplItemFn); // 替换原方法
        }
    }
    quote! { #ast }.into() // 返回修改后的 Impl 块
}

#[proc_macro]
pub fn call_interface(item: TokenStream) -> TokenStream {
    let call = parse_macro_input!(item as CallInterface); // 解析为自定义结构体
    let args = call.args; // 参数列表（如 42, "test"）
    let mut path = call.path.segments; // 路径（如 MyTrait::my_method）
    // 校验路径格式（至少 Trait 和方法名）
    if path.len() < 2 {
        return compiler_error(...);
    }
    let fn_name = path.pop().unwrap(); // 弹出方法名（如 my_method）
    let trait_name = path.pop().unwrap(); // 弹出 Trait 名（如 MyTrait）
    let extern_fn_name = format_ident!("__{}_{}", trait_name.ident, fn_name.ident); // 生成外部函数名
    // 构造模块路径（如 __MyTrait_mod）
    path.push_value(PathSegment {
        ident: format_ident!("__{}_mod", trait_name.ident),
        arguments: PathArguments::None,
    });
    // 生成 unsafe 调用代码
    quote! { unsafe { #path::#extern_fn_name(#args) } }.into()
}

struct CallInterface { path: Path, args: Punctuated<Expr, Token![,]> }

impl Parse for CallInterface {
    fn parse(input: ParseStream) -> Result<Self> {
        let path: Path = input.parse()?; // 解析路径（如 MyTrait::my_method）
        let args = if input.peek(Token![,]) { // 处理逗号分隔参数
            input.parse::<Token![,]>()?;
            input.parse_terminated(Expr::parse, Token![,])?
        } else if !input.is_empty() { // 处理括号参数（如 (42)）
            parenthesized!(content in input);
            content.parse_terminated(Expr::parse, Token![,])?
        } else { Punctuated::new() };
        Ok(CallInterface { path, args })
    }
}
```

### 二、其他组件

#### axlog

封装并扩展crate.io中的crate - log。按照 crate log 的实现要求，为 定义的全局日志实例 Logger 实现 trait Log 接口。这个外部的 crate log 本身是一个框架，实现了日志的各种通用功能，但是如何对日志进行输出需要基于所在的环境，这个 trait Log 就是通用功能与环境交互的接口

```rust
// axlog/src/lib.rs
macro_rules! with_color {
    ($color_code:expr, $($arg:tt)*) => {{
        format_args!("\u{1B}[{}m{}\u{1B}[m", $color_code as u8, format_args!($($arg)*))
    }};
}

#[repr(u8)]
#[allow(dead_code)]
enum ColorCode {
    Red = 31, Green = 32, Yellow = 33, Cyan = 36, White = 37, BrightBlack = 90,
}

impl Log for Logger {
    #[inline]
    fn enabled(&self, _metadata: &Metadata) -> bool {
        true
    }

    fn log(&self, record: &Record) {
        let level = record.level();
        let line = record.line().unwrap_or(0);
        let path = record.target();
        let args_color = match level {
            Level::Error => ColorCode::Red,
            Level::Warn => ColorCode::Yellow,
            Level::Info => ColorCode::Green,
            Level::Debug => ColorCode::Cyan,
            Level::Trace => ColorCode::BrightBlack,
        };
        let now = call_interface!(LogIf::get_time);

        print_fmt(with_color!(
            ColorCode::White,
            "[{:>3}.{:06} {path}:{line}] {args}\n",
            now.as_secs(),
            now.subsec_micros(),
            path = path,
            line = line,
            args = with_color!(args_color, "{}", record.args()),
        ));
    }

    fn flush(&self) {}
}
```

#### axdtb

内核最初启动时从 SBI 得到两个参数分别在 a0 和 a1 寄存器中。其中 a1 寄存器保存的是 dtb 的开始地址，而 dtb 就是 fdt 的二进制形式，它的全称 device tree blob。由于它已经在内存中放置好，内核启动时就可以直接解析它的内容获取信息。我们引入组件axdtb让内核自己解析 fdt 设备树来获得硬件平台的配置情况，作为后面启动过程的基础

```rust
// axdtb/src/lib.rs
impl DeviceTree {
    pub fn parse(
        &self, mut pos: usize,
        mut addr_cells: usize,
        mut size_cells: usize,
        cb: &mut dyn FnMut(String, usize, usize, Vec<(String, Vec<u8>)>)
    ) -> DeviceTreeResult<usize> {
        let buf = unsafe {
            core::slice::from_raw_parts(self.ptr as *const u8, self.totalsize)
        };

        // check for DT_BEGIN_NODE
        if buf.read_be_u32(pos)? != OF_DT_BEGIN_NODE {
            return Err(DeviceTreeError::ParseError(pos))
        }
        pos += 4;

        let raw_name = buf.read_bstring0(pos)?;
        pos = align_up(pos + raw_name.len() + 1, 4);

        // First, read all the props.
        let mut props = Vec::new();
        while buf.read_be_u32(pos)? == OF_DT_PROP {
            let val_size = buf.read_be_u32(pos+4)? as usize;
            let name_offset = buf.read_be_u32(pos+8)? as usize;

            // get value slice
            let val_start = pos + 12;
            let val_end = val_start + val_size;
            let val = buf.subslice(val_start, val_end)?;

            // lookup name in strings table
            let prop_name = buf.read_bstring0(self.off_strings + name_offset)?;

            let prop_name = str::from_utf8(prop_name)?.to_owned();
            if prop_name == "#address-cells" {
                addr_cells = val.read_be_u32(0)? as usize;
            } else if prop_name == "#size-cells" {
                size_cells = val.read_be_u32(0)? as usize;
            }

            props.push((prop_name, val.to_owned()));

            pos = align_up(val_end, 4);
        }

        // Callback for parsing dtb
        let name = str::from_utf8(raw_name)?.to_owned();
        cb(name, addr_cells, size_cells, props);

        // Then, parse all its children.
        while buf.read_be_u32(pos)? == OF_DT_BEGIN_NODE {
            pos = self.parse(pos, addr_cells, size_cells, cb)?;
        }

        if buf.read_be_u32(pos)? != OF_DT_END_NODE {
            return Err(DeviceTreeError::ParseError(pos))
        }

        pos += 4;

        Ok(pos)
    }
}
```

## Ch4 内存管理2

### 一、实现多级页表

扩展实现Ch2的页表机制，之前的 map 实现十分简单，只是映射了 1G，并且 va/pa 地址以及总长度 total_size 都是按 1G 对齐的。但是如果地址范围可能出现不对齐的情况，这是指开始/结束地址没有按照 best_size 对齐或者总长度就小于 best_size。例如我们期望按照 2M 的粒度进行映射（即 best_size 是 2M，如上图所示），但是起止地址都没有按照 2M 对齐，那就需要把映射范围分成三个部分：把中间按照 2M 对齐的部分截出来，按照 2M 的 best_size 进行映射；把前后两个剩余部分按照 4K 的粒度单独映射