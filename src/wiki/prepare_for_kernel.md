# 内核进入许可证

上一章我们成功搭建了 Rust for Linux 的源码环境，现在我们将正式开始深入学习内核源码。在此之前，我们首先需要了解 Rust 在内核中的组织结构 (只包含目录)：

``` bash
rust
├── bindings/
├── helpers/
├── kernel/
├── macros/
├── pin-init/
├── proc-macro2/
├── quote/
├── syn/
└── uapi/
```

在这个结构中，`rust/kernel` 是 Rust 实现内核逻辑 的主要目录。而除 `rust/kernel` 之外的所有目录，都是 `Rust for Linux` 项目 提供的 外部支持模块，它们共同为内核代码的编写提供了必要的工具和抽象。例如：`pin-init` 模块：这是一个重要的辅助模块，由 `Rust for Linux` 上游直接维护。

因此，在真正进入内核 (rust/kernel) 的学习之前，我们必须先掌握这些主要的外部支持模块。只有理解了这些“许可证”模块的功能和用法，我们才能在后续的源码学习中，准确而深入地理解代码的含义和设计思路。

> 这就是为什么我将这个章节取名为：内核进入许可证。掌握这些基础模块，是深入内核源码的先决条件。

## Bindings

在 Rust for Linux 项目的初期，首要也是最关键的任务是解决 Rust 代码如何安全、高效地调用现有 C 语言内核函数 的问题。由于完全重写整个 Linux 内核是不切实际的，复用成熟的 C 语言代码库 成为了必然的选择。

为了实现这一目标，rust-bindgen 提供了可行的、自动化的解决方案。bindgen 是一个强大的工具，它通过命令行直接将 C 语言的头文件 (Header Files) 转换为 Rust 的外部函数接口 (Foreign Function Interface, FFI) 绑定代码。

``` Makefile
# rust/Makefile:444
quiet_cmd_bindgen = BINDGEN $@
      cmd_bindgen = \
	$(BINDGEN) $< $(bindgen_target_flags) --rust-target 1.68 \
		--use-core --with-derive-default --ctypes-prefix ffi --no-layout-tests \
		--no-debug '.*' --enable-function-attribute-detection \
		-o $@ -- $(bindgen_c_flags_final) -DMODULE \
		$(bindgen_target_cflags) $(bindgen_target_extra)
```

上面给出了 rust 是如何使用 bindgen 来生成 Rust 外部函数接口绑定代码。在内核中，主要有三个地方会生成 FFI 代码，分别位于两个目录中：`bindings/` 和 `uapi/`。

一些通用配置，例如：`--use-core` 指导内核生成的 FFI 接口不会使用 `std` 标准库，而是使用 `core` 库 (实际上，也不会使用 `core:ffi` 而是使用 `crate:ffi`，在[ffi](#ffi)小节中有详细解释)，这是由于内核的环境是 `no-std` 的；`--ctypes-prefix ffi` 会为 C 类型添加 `ffi::` 前缀，以明确其来自 Rust 的 core::ffi` 模块 ...

`$@` 则是目标产物文件，`--` 分隔符后的内容是传递给 `Clang` 编译器的参数，用于解析 C 语言头文件内容。

``` Makefile
# rust/Makefile:370
bindgen_skip_c_flags := -mno-fp-ret-in-387 -mpreferred-stack-boundary=% \
  -mskip-rax-setup ...

bindgen_extra_c_flags = -w --target=$(BINDGEN_TARGET)
ifeq ($(shell expr $(libclang_maj_ver) \< 16), 1)
bindgen_extra_c_flags += -enable-trivial-auto-var-init-zero-knowing-it-will-be-removed-from-clang
endif

# rust/Makefile:415
bindgen_c_flags = $(filter-out $(bindgen_skip_c_flags), $(c_flags)) \
	$(bindgen_extra_c_flags)
endif

ifdef CONFIG_LTO
bindgen_c_flags_lto = $(filter-out $(CC_FLAGS_LTO), $(bindgen_c_flags))
else
bindgen_c_flags_lto = $(bindgen_c_flags)
endif

# rust/Makefile:425
# `-fno-builtin` is passed to avoid `bindgen` from using `clang` builtin
# prototypes for functions like `memcpy` -- if this flag is not passed,
# `bindgen`-generated prototypes use `c_ulong` or `c_uint` depending on
# architecture instead of generating `usize`.
bindgen_c_flags_final = $(bindgen_c_flags_lto) -fno-builtin -D__BINDGEN__
```

`bindgen_c_flags_final` 保证了最终传递给 bindgen 的 C 编译环境是最小化、最纯净且完全针对 bindgen 生成而优化的，避免了不必要的架构优化细节来干扰 FFI 类型的正确映射。

首先 `bindgen_skip_c_flags` 列出了一组需要从标准内核编译标志中排除的 C 标志；通常来说，bindgen 理应支持 GNU GCC 后端，并且完全兼容 Clang 驱动生成；为了能够 Clang 的正常工作，因此跳过不必要的标志。

而后，`bindgen_c_flags` 会根据给出的 `bindgen_skip_c_flags` 从 `c_flags` 中剔除掉相应的标志，并且添加上 bindgen 额外需要的标志。如果内核开启了 LTO 优化，那么相关的 LTO 标志就必须被移除，使其最终获得一个不含 LTO 的标志集合。并且还需要禁止内敛函数的生成，如下则是一个经过编译验证的 `bindgen_c_flags_final` 的完整内容，读者可用选择性查看：

``` Makefile
-Wp,-MMD,rust/bindings/.bindings_generated.rs.d -nostdinc -I./arch/x86/include -I./arch/x86/include/generated -I./include -I./include -I./arch/x86/include/uapi -I./arch/x86/include/generated/uapi -I./include/uapi -I./include/generated/uapi -include ./include/linux/compiler-version.h -include ./include/linux/kconfig.h -include ./include/linux/compiler_types.h -D__KERNEL__ --target=x86_64-linux-gnu -fintegrated-as -Werror=unknown-warning-option -Werror=ignored-optimization-argument -Werror=option-ignored -Werror=unused-command-line-argument -Werror -std=gnu11 -fshort-wchar -funsigned-char -fno-common -fno-PIE -fno-strict-aliasing -mno-sse -mno-mmx -mno-sse2 -mno-3dnow -mno-avx -mno-sse4a -fcf-protection=branch -fno-jump-tables -m64 -falign-loops=1 -mno-80387 -mno-fp-ret-in-387 -mstack-alignment=8 -mskip-rax-setup -march=x86-64 -mtune=generic -mno-red-zone -mcmodel=kernel -mstack-protector-guard-reg=gs -mstack-protector-guard-symbol=__ref_stack_chk_guard -Wno-sign-compare -fno-asynchronous-unwind-tables -mretpoline-external-thunk -mindirect-branch-cs-prefix -mfunction-return=thunk-extern -fpatchable-function-entry=16,16 -fno-delete-null-pointer-checks -O2 -fstack-protector-strong -fomit-frame-pointer -ftrivial-auto-var-init=zero -fno-stack-clash-protection -falign-functions=16 -fstrict-flex-arrays=3 -fms-extensions -fno-strict-overflow -fno-stack-check -fno-builtin-wcslen -Wall -Wextra -Wundef -Werror=implicit-function-declaration -Werror=implicit-int -Werror=return-type -Werror=strict-prototypes -Wno-format-security -Wno-trigraphs -Wno-frame-address -Wno-address-of-packed-member -Wmissing-declarations -Wmissing-prototypes -Wframe-larger-than=2048 -Wno-gnu -Wno-microsoft-anon-tag -Wno-format-overflow-non-kprintf -Wno-format-truncation-non-kprintf -Wno-pointer-sign -Wcast-function-type -Wimplicit-fallthrough -Werror=date-time -Werror=incompatible-pointer-types -Wenum-conversion -Wunused -Wno-unused-but-set-variable -Wno-unused-const-variable -Wno-format-overflow -Wno-override-init -Wno-pointer-to-enum-cast -Wno-tautological-constant-out-of-range-compare -Wno-unaligned-access -Wno-enum-compare-conditional -Wno-missing-field-initializers -Wno-type-limits -Wno-shift-negative-value -Wno-enum-enum-conversion -Wno-sign-compare -Wno-unused-parameter    -DKBUILD_MODFILE='"rust/bindings_generated"' -DKBUILD_BASENAME='"bindings_generated"' -DKBUILD_MODNAME='"bindings_generated"' -D__KBUILD_MODNAME=bindings_generated -fno-builtin -D__BINDGEN__ 
```

> 为什么需要移除 LTO 优化和内联函数构建？ 
> **LTO (Link-Time Optimization) 标志告诉编译器延迟许多优化，直到整个程序或模块的代码都可见时（即在链接阶段）。这旨在生成最优化的最终机器码。而 bindgen 阶段只依赖 Clang 编译器前端来解析 C 语言 AST 并理解类型和函数前面，并不需要实际的代码生成和优化。因此 LTO 阶段是不必要的**。  
> **Clang 编译器会为标准 C 语言函数库使用内置原型，而非去查找内核头文件中的实际定义。内置原型通常使用标准的 C 类型，而 C 语言的类型原型比如 `unsigned long` 则会取决于具体架构实现 (这对应了 Rust 的 `c_ulong` 或 `c_uint`)。因此，禁止内置原型以强制 bindgen 使用内核或宏定义提供的带正确的 `size_t` (`usize`) 参数的函数原型，从而使得 Rust FFI 绑定更精准、更符合 Rust 的语言规范**。

### bindings_generated

`$<` 代指了我们所依赖的文件 `rust/bindings/bindings_helper.h`；bindgen 会根据这一个头文件中的所有内容来生成对应接口绑定代码。`$(bindgen_target_flags)` 指定了 bindgen 的参数，其具体指定在：

``` Makefile
# rust/Makefile:452
$(obj)/bindings/bindings_generated.rs: private bindgen_target_flags = \
    $(shell grep -Ev '^#|^$$' $(src)/bindgen_parameters)
```

`$(obj)/bindings/bindings_generated.rs` 是我们最终的生成产物，所有的 FFI 代码都存放在该文件中，而这个最终产物依赖于 `bindgen_target_flags` 规则，其指向了 `rust/bindgen_parameters` 文件。因此，bindgen 的参数会从 `bindgen_parameters` 中获取。

``` Makefile
# rust/Makefile:454
$(obj)/bindings/bindings_generated.rs: private bindgen_target_extra = ; \
    sed -Ei 's/pub const RUST_CONST_HELPER_([a-zA-Z0-9_]*)/pub const \1/g' $@
```

`bindgen_target_extra` 指明了在生成 `bindings_generated.rs` 后会通过 `sed` 命令移除 `RUST_CONST_HELPER_` 前缀，这使得常量更为简洁，更符合 Rust for Linux 的命名规范。

``` bash
sed -Ei 's/pub const RUST_CONST_HELPER_([a-zA-Z0-9_]*)/pub const \1/g' rust/bindings/bindings_generated.rs
```

内核中完整的编译命令可用在下方查看，也可以在编译后的内核中查看 `.rust/bindings/bindings_generated.rs.cmd`：

``` Makefile
savedcmd_rust/bindings/bindings_generated.rs := bindgen rust/bindings/bindings_helper.h --blocklist-type __kernel_s?size_t --blocklist-type __kernel_ptrdiff_t --opaque-type xregs_state --opaque-type desc_struct --opaque-type arch_lbr_state --opaque-type local_apic --opaque-type alt_instr --opaque-type x86_msi_data --opaque-type x86_msi_addr_lo --opaque-type kunit_try_catch --opaque-type spinlock --no-doc-comments --blocklist-function __list_.*_report --blocklist-item ARCH_SLAB_MINALIGN --blocklist-item ARCH_KMALLOC_MINALIGN --blocklist-item VM_MERGEABLE --blocklist-item VM_READ --blocklist-item VM_WRITE --blocklist-item VM_EXEC --blocklist-item VM_SHARED --blocklist-item VM_MAYREAD --blocklist-item VM_MAYWRITE --blocklist-item VM_MAYEXEC --blocklist-item VM_MAYEXEC --blocklist-item VM_PFNMAP --blocklist-item VM_IO --blocklist-item VM_DONTCOPY --blocklist-item VM_DONTEXPAND --blocklist-item VM_LOCKONFAULT --blocklist-item VM_ACCOUNT --blocklist-item VM_NORESERVE --blocklist-item VM_HUGETLB --blocklist-item VM_SYNC --blocklist-item VM_ARCH_1 --blocklist-item VM_WIPEONFORK --blocklist-item VM_DONTDUMP --blocklist-item VM_SOFTDIRTY --blocklist-item VM_MIXEDMAP --blocklist-item VM_HUGEPAGE --blocklist-item VM_NOHUGEPAGE --with-derive-custom-struct .*=MaybeZeroable --with-derive-custom-union .*=MaybeZeroable --rust-target 1.68 --use-core --with-derive-default --ctypes-prefix ffi --no-layout-tests --no-debug '.*' --enable-function-attribute-detection -o rust/bindings/bindings_generated.rs -- -Wp,-MMD,rust/bindings/.bindings_generated.rs.d -nostdinc -I./arch/x86/include -I./arch/x86/include/generated -I./include -I./include -I./arch/x86/include/uapi -I./arch/x86/include/generated/uapi -I./include/uapi -I./include/generated/uapi -include ./include/linux/compiler-version.h -include ./include/linux/kconfig.h -include ./include/linux/compiler_types.h -D__KERNEL__ --target=x86_64-linux-gnu -fintegrated-as -Werror=unknown-warning-option -Werror=ignored-optimization-argument -Werror=option-ignored -Werror=unused-command-line-argument -Werror -std=gnu11 -fshort-wchar -funsigned-char -fno-common -fno-PIE -fno-strict-aliasing -mno-sse -mno-mmx -mno-sse2 -mno-3dnow -mno-avx -mno-sse4a -fcf-protection=branch -fno-jump-tables -m64 -falign-loops=1 -mno-80387 -mno-fp-ret-in-387 -mstack-alignment=8 -mskip-rax-setup -march=x86-64 -mtune=generic -mno-red-zone -mcmodel=kernel -mstack-protector-guard-reg=gs -mstack-protector-guard-symbol=__ref_stack_chk_guard -Wno-sign-compare -fno-asynchronous-unwind-tables -mretpoline-external-thunk -mindirect-branch-cs-prefix -mfunction-return=thunk-extern -fpatchable-function-entry=16,16 -fno-delete-null-pointer-checks -O2 -fstack-protector-strong -fomit-frame-pointer -ftrivial-auto-var-init=zero -fno-stack-clash-protection -falign-functions=16 -fstrict-flex-arrays=3 -fms-extensions -fno-strict-overflow -fno-stack-check -fno-builtin-wcslen -Wall -Wextra -Wundef -Werror=implicit-function-declaration -Werror=implicit-int -Werror=return-type -Werror=strict-prototypes -Wno-format-security -Wno-trigraphs -Wno-frame-address -Wno-address-of-packed-member -Wmissing-declarations -Wmissing-prototypes -Wframe-larger-than=2048 -Wno-gnu -Wno-microsoft-anon-tag -Wno-format-overflow-non-kprintf -Wno-format-truncation-non-kprintf -Wno-pointer-sign -Wcast-function-type -Wimplicit-fallthrough -Werror=date-time -Werror=incompatible-pointer-types -Wenum-conversion -Wunused -Wno-unused-but-set-variable -Wno-unused-const-variable -Wno-format-overflow -Wno-override-init -Wno-pointer-to-enum-cast -Wno-tautological-constant-out-of-range-compare -Wno-unaligned-access -Wno-enum-compare-conditional -Wno-missing-field-initializers -Wno-type-limits -Wno-shift-negative-value -Wno-enum-enum-conversion -Wno-sign-compare -Wno-unused-parameter    -DKBUILD_MODFILE='"rust/bindings_generated"' -DKBUILD_BASENAME='"bindings_generated"' -DKBUILD_MODNAME='"bindings_generated"' -D__KBUILD_MODNAME=bindings_generated -fno-builtin -D__BINDGEN__ -DMODULE  ; sed -Ei 's/pub const RUST_CONST_HELPER_([a-zA-Z0-9_]*)/pub const \1/g' rust/bindings/bindings_generated.rs
```

### bindings_helpers_generated

在 `bindings_helpers_generated` 的生成中，主要是通过将 `rust/helpers/helpers.c` 文件中的所有内容转为 FFI 绑定。

`bindgen_target_flags` 与 `bindings_generated` 的有所不同，如下所示：

``` Makefile
# rust/Makefile:470
$(obj)/bindings/bindings_helpers_generated.rs: private bindgen_target_flags = \
    --blocklist-type '.*' --allowlist-var '' \
    --allowlist-function 'rust_helper_.*'
```

``` Makefile
# rust/Makefile:473
$(obj)/bindings/bindings_helpers_generated.rs: private bindgen_target_cflags = \
    -I$(objtree)/$(obj) -Wno-missing-prototypes -Wno-missing-declarations
```

在 `bindings_helpers_generated` 的生成中，额外依赖了 `bindgen_target_cflags` 参数，这是因为编译一个源文件需要找到对应所需的所有头文件。并且，**关闭 `missing prototypes` 和 `missing declarations` 警告，这是为了 bindgen 解析时可能会遇到源文件中某些函数可能只有定义而没有显式的原型声明，防止这种情况下产生的不必要警告导致编译流程中断或失败**。

``` Makefile
# rust/Makefile:475
$(obj)/bindings/bindings_helpers_generated.rs: private bindgen_target_extra = ; \
    sed -Ei 's/pub fn rust_helper_([a-zA-Z0-9_]*)/#[link_name="rust_helper_\1"]\n    pub fn \1/g' $@
```

对于 `bindgen_target_extra` 参数，是为了后续的生成文件中，添加 `#[link_name=]` 标记，在不改变底层 C 符号 (rust_helper_foo) 的情况下，重命名 Rust 侧的函数接口 (foo)：

``` rs
extern "C" {
    #[link_name="rust_helper_atomic_read"]
    pub fn atomic_read(v: *const atomic_t) -> ffi::c_int;
}
```

内核中完整的编译命令可用在下方查看，也可以在编译后的内核中查看 `.rust/bindings/bindings_helpers_generated.rs.cmd`：

``` Makefile
savedcmd_rust/bindings/bindings_helpers_generated.rs := bindgen rust/helpers/helpers.c --blocklist-type '.*' --allowlist-var '' --allowlist-function 'rust_helper_.*' --rust-target 1.68 --use-core --with-derive-default --ctypes-prefix ffi --no-layout-tests --no-debug '.*' --enable-function-attribute-detection -o rust/bindings/bindings_helpers_generated.rs -- -Wp,-MMD,rust/bindings/.bindings_helpers_generated.rs.d -nostdinc -I./arch/x86/include -I./arch/x86/include/generated -I./include -I./include -I./arch/x86/include/uapi -I./arch/x86/include/generated/uapi -I./include/uapi -I./include/generated/uapi -include ./include/linux/compiler-version.h -include ./include/linux/kconfig.h -include ./include/linux/compiler_types.h -D__KERNEL__ --target=x86_64-linux-gnu -fintegrated-as -Werror=unknown-warning-option -Werror=ignored-optimization-argument -Werror=option-ignored -Werror=unused-command-line-argument -Werror -std=gnu11 -fshort-wchar -funsigned-char -fno-common -fno-PIE -fno-strict-aliasing -mno-sse -mno-mmx -mno-sse2 -mno-3dnow -mno-avx -mno-sse4a -fcf-protection=branch -fno-jump-tables -m64 -falign-loops=1 -mno-80387 -mno-fp-ret-in-387 -mstack-alignment=8 -mskip-rax-setup -march=x86-64 -mtune=generic -mno-red-zone -mcmodel=kernel -mstack-protector-guard-reg=gs -mstack-protector-guard-symbol=__ref_stack_chk_guard -Wno-sign-compare -fno-asynchronous-unwind-tables -mretpoline-external-thunk -mindirect-branch-cs-prefix -mfunction-return=thunk-extern -fpatchable-function-entry=16,16 -fno-delete-null-pointer-checks -O2 -fstack-protector-strong -fomit-frame-pointer -ftrivial-auto-var-init=zero -fno-stack-clash-protection -falign-functions=16 -fstrict-flex-arrays=3 -fms-extensions -fno-strict-overflow -fno-stack-check -fno-builtin-wcslen -Wall -Wextra -Wundef -Werror=implicit-function-declaration -Werror=implicit-int -Werror=return-type -Werror=strict-prototypes -Wno-format-security -Wno-trigraphs -Wno-frame-address -Wno-address-of-packed-member -Wmissing-declarations -Wmissing-prototypes -Wframe-larger-than=2048 -Wno-gnu -Wno-microsoft-anon-tag -Wno-format-overflow-non-kprintf -Wno-format-truncation-non-kprintf -Wno-pointer-sign -Wcast-function-type -Wimplicit-fallthrough -Werror=date-time -Werror=incompatible-pointer-types -Wenum-conversion -Wunused -Wno-unused-but-set-variable -Wno-unused-const-variable -Wno-format-overflow -Wno-override-init -Wno-pointer-to-enum-cast -Wno-tautological-constant-out-of-range-compare -Wno-unaligned-access -Wno-enum-compare-conditional -Wno-missing-field-initializers -Wno-type-limits -Wno-shift-negative-value -Wno-enum-enum-conversion -Wno-sign-compare -Wno-unused-parameter    -DKBUILD_MODFILE='"rust/bindings_helpers_generated"' -DKBUILD_BASENAME='"bindings_helpers_generated"' -DKBUILD_MODNAME='"bindings_helpers_generated"' -D__KBUILD_MODNAME=bindings_helpers_generated -fno-builtin -D__BINDGEN__ -DMODULE -I./rust -Wno-missing-prototypes -Wno-missing-declarations ; sed -Ei 's/pub fn rust_helper_([a-zA-Z0-9_]*)/$(pound)[link_name="rust_helper_\1"]\n    pub fn \1/g' rust/bindings/bindings_helpers_generated.rs
```

### uapi_generated

`uapi_generated` 通过将 `rust/uapi/uapi_helper.h` 头文件中的所有内容转为 FFI 绑定。主要依赖了一条规则：

``` Makefile
# rust/Makefile:460
$(obj)/uapi/uapi_generated.rs: private bindgen_target_flags = \
    $(shell grep -Ev '^#|^$$' $(src)/bindgen_parameters)
```

很显然，这与 `bindings_generated` 的依赖规则一致，都是依赖 `rust/bindgen_parameters` 文件中的编译规则来生成 FFI 代码。

内核中完整的编译命令可用在下方查看，也可以在编译后的内核中查看 `.rust/uapi/bindings_helpers_generated.rs.cmd`：

``` Makefile
savedcmd_rust/uapi/uapi_generated.rs := bindgen rust/uapi/uapi_helper.h --blocklist-type __kernel_s?size_t --blocklist-type __kernel_ptrdiff_t --opaque-type xregs_state --opaque-type desc_struct --opaque-type arch_lbr_state --opaque-type local_apic --opaque-type alt_instr --opaque-type x86_msi_data --opaque-type x86_msi_addr_lo --opaque-type kunit_try_catch --opaque-type spinlock --no-doc-comments --blocklist-function __list_.*_report --blocklist-item ARCH_SLAB_MINALIGN --blocklist-item ARCH_KMALLOC_MINALIGN --blocklist-item VM_MERGEABLE --blocklist-item VM_READ --blocklist-item VM_WRITE --blocklist-item VM_EXEC --blocklist-item VM_SHARED --blocklist-item VM_MAYREAD --blocklist-item VM_MAYWRITE --blocklist-item VM_MAYEXEC --blocklist-item VM_MAYEXEC --blocklist-item VM_PFNMAP --blocklist-item VM_IO --blocklist-item VM_DONTCOPY --blocklist-item VM_DONTEXPAND --blocklist-item VM_LOCKONFAULT --blocklist-item VM_ACCOUNT --blocklist-item VM_NORESERVE --blocklist-item VM_HUGETLB --blocklist-item VM_SYNC --blocklist-item VM_ARCH_1 --blocklist-item VM_WIPEONFORK --blocklist-item VM_DONTDUMP --blocklist-item VM_SOFTDIRTY --blocklist-item VM_MIXEDMAP --blocklist-item VM_HUGEPAGE --blocklist-item VM_NOHUGEPAGE --with-derive-custom-struct .*=MaybeZeroable --with-derive-custom-union .*=MaybeZeroable --rust-target 1.68 --use-core --with-derive-default --ctypes-prefix ffi --no-layout-tests --no-debug '.*' --enable-function-attribute-detection -o rust/uapi/uapi_generated.rs -- -Wp,-MMD,rust/uapi/.uapi_generated.rs.d -nostdinc -I./arch/x86/include -I./arch/x86/include/generated -I./include -I./include -I./arch/x86/include/uapi -I./arch/x86/include/generated/uapi -I./include/uapi -I./include/generated/uapi -include ./include/linux/compiler-version.h -include ./include/linux/kconfig.h -include ./include/linux/compiler_types.h -D__KERNEL__ --target=x86_64-linux-gnu -fintegrated-as -Werror=unknown-warning-option -Werror=ignored-optimization-argument -Werror=option-ignored -Werror=unused-command-line-argument -Werror -std=gnu11 -fshort-wchar -funsigned-char -fno-common -fno-PIE -fno-strict-aliasing -mno-sse -mno-mmx -mno-sse2 -mno-3dnow -mno-avx -mno-sse4a -fcf-protection=branch -fno-jump-tables -m64 -falign-loops=1 -mno-80387 -mno-fp-ret-in-387 -mstack-alignment=8 -mskip-rax-setup -march=x86-64 -mtune=generic -mno-red-zone -mcmodel=kernel -mstack-protector-guard-reg=gs -mstack-protector-guard-symbol=__ref_stack_chk_guard -Wno-sign-compare -fno-asynchronous-unwind-tables -mretpoline-external-thunk -mindirect-branch-cs-prefix -mfunction-return=thunk-extern -fpatchable-function-entry=16,16 -fno-delete-null-pointer-checks -O2 -fstack-protector-strong -fomit-frame-pointer -ftrivial-auto-var-init=zero -fno-stack-clash-protection -falign-functions=16 -fstrict-flex-arrays=3 -fms-extensions -fno-strict-overflow -fno-stack-check -fno-builtin-wcslen -Wall -Wextra -Wundef -Werror=implicit-function-declaration -Werror=implicit-int -Werror=return-type -Werror=strict-prototypes -Wno-format-security -Wno-trigraphs -Wno-frame-address -Wno-address-of-packed-member -Wmissing-declarations -Wmissing-prototypes -Wframe-larger-than=2048 -Wno-gnu -Wno-microsoft-anon-tag -Wno-format-overflow-non-kprintf -Wno-format-truncation-non-kprintf -Wno-pointer-sign -Wcast-function-type -Wimplicit-fallthrough -Werror=date-time -Werror=incompatible-pointer-types -Wenum-conversion -Wunused -Wno-unused-but-set-variable -Wno-unused-const-variable -Wno-format-overflow -Wno-override-init -Wno-pointer-to-enum-cast -Wno-tautological-constant-out-of-range-compare -Wno-unaligned-access -Wno-enum-compare-conditional -Wno-missing-field-initializers -Wno-type-limits -Wno-shift-negative-value -Wno-enum-enum-conversion -Wno-sign-compare -Wno-unused-parameter    -DKBUILD_MODFILE='"rust/uapi_generated"' -DKBUILD_BASENAME='"uapi_generated"' -DKBUILD_MODNAME='"uapi_generated"' -D__KBUILD_MODNAME=uapi_generated -fno-builtin -D__BINDGEN__ -DMODULE  
```

### ffi

在 Rust for Linux 中，`rust/ffi.rs` 准确的定义了**如何将 C 语言中用于 FFI 的基本类型精确地映射到 Rust 语言的固定大小或平台依赖类型，并提供了一个绑定宏**。

**Rust 的标准 `core::ffi` 模块提供了平台默认的 C ABI 映射。然而，Linux 内核环境（由于其特殊的编译配置和目标）可能需要偏离这些默认映射。这个文件通过自己实现这些类型别名，确保了 Rust FFI 调用时使用的类型与 C 内核编译时使用的类型完全一致**。

> 因此，Rust for Linux 不使用 `core:ffi` 而是使用 `crate::ffi`。

``` rust
macro_rules! alias {
    ($($name:ident = $ty:ty;)*) => {$(
        #[allow(non_camel_case_types, missing_docs)]
        pub type $name = $ty;

        // Check size compatibility with `core`.
        const _: () = assert!(
            ::core::mem::size_of::<$name>() == ::core::mem::size_of::<::core::ffi::$name>()
        );
    )*}
}
```

通过设计一个 `alias!` 宏为 C 类型定义 Rust 别名，并且允许使用非惯用的 C 命名风格，并避免文档缺失警告。**尽管内核不直接使用 `core::ffi`，但还是通过断言检查作为一道兼容性护栏，确保内核的自定义映射至少在内存大小上没有与平台标准发生严重冲突**。

``` rust
use std;

# macro_rules! alias {
#     ($($name:ident = $ty:ty;)*) => {$(
#         #[allow(non_camel_case_types, missing_docs)]
#         pub type $name = $ty;

#         // Check size compatibility with `core`.
#         const _: () = assert!(
#             ::core::mem::size_of::<$name>() == ::core::mem::size_of::<::core::ffi::$name>()
#         );
#     )*}
# }
alias! {
  c_char = u8;
}

const test_c: c_char = 0;
println!("Successful using c_char type: {test_c}");
```

而后，在构建系统中，通过如下编译指令将 `core:ffi` 替换为 `crate::ffi`：

``` Makefile
# rust/Makefile:659
$(obj)/ffi.o: private skip_gendwarfksyms = 1
$(obj)/ffi.o: $(src)/ffi.rs

(obj)/bindings.o: private rustc_target_flags = --extern ffi
$(obj)/bindings.o: $(src)/bindings/lib.rs \
    $(obj)/ffi.o \
    $(obj)/bindings/bindings_generated.rs \
    $(obj)/bindings/bindings_helpers_generated.rs

$(obj)/uapi.o: private rustc_target_flags = --extern ffi
$(obj)/uapi.o: private skip_gendwarfksyms = 1
$(obj)/uapi.o: $(src)/uapi/lib.rs \
    $(obj)/ffi.o \
    $(obj)/uapi/uapi_generated.rs
```

`ffi.o` 是编译 FFI 类型定义模块，通过 `skip_gendwarfksyms` 告知内核编译系统在处理 `ffi.o` 时**跳过 DWARF 内核符号信息，因为该模块只包含类型别名和宏定义，没有复杂的函数或全局变量需要导出到内核符号表 (ksyms) 或进行复杂的调试信息生成，因此跳过此步骤可以节省构建时间**。

随后在编译 `bindings` 和 `uapi` 这两个 FFI 实现时，通过 `--extern ffi` 表明这两个模块需要链接或依赖名为 ffi 的外部 crate。这样，Rust for Linux 的 bindings 生成就使用上了内核自定义的 `ffi` 类型系统。
