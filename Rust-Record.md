# Rust-Record

###### as_ref()

```rust
let group = parent.as_ref().map_or_else(
    // 当 parent 为 None 时的处理逻辑
    || {
        let session = Session::new(pid);    // 创建新会话
        ProcessGroup::new(pid, &session)    // 基于新会话创建进程组
    },
    // 当 parent 为 Some 时的处理逻辑
    |p| p.group(),                          // 直接使用父进程的进程组
);
```

###### unwrap_or_else

```rust
let testcases = option_env!("AX_TESTCASES_LIST")     // 步骤1：编译时读取环境变量
    .unwrap_or_else(|| "Please specify...")         // 步骤2：处理空值，提供默认提示
    .split(',')                                     // 步骤3：按逗号分割字符串
    .filter(|&x| !x.is_empty());                    // 步骤4：过滤空字符串
```

