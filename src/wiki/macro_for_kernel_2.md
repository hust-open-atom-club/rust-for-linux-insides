# 内核中 Rust 宏的使用手册 (二)

我们已经掌握了 Rust 宏的使用方法。接下来，为了深入理解 Rust 内核中的宏是如何构建和运作的，我们需要了解几个核心构建宏的软件包。

我们将从应用层 (如何用 `quote` 生成代码) 开始，逐渐深入到解析层 (如何用 `syn` 处理输入代码)，最终触及核心原理 (`proc-macro` 库提供的基础 API)。

## Quote 代码生成层

[quote](https://docs.rs/quote/1.0.40/quote/) 库的目的是将 Rust 抽象语法树 (Abstract Syntax Tree, AST) 的片段转换回可编译的 Rust 代码 (Tokens)。它是构建宏输出的核心工具。

Rust 中的过程宏接收一个 Token 流作为输入，执行任意的 Rust 代码以确定如何处理这些 Token，并生成一个 Token 流返回给编译器，以便编译到调用者的 crate 中。`Quasi-quoting` 是解决其中一部分问题的方法——生成返回给编译器的 Token。

`Quasi-quoting` 的想法是，我们编写代码，但将其视为数据。在 `quote!` 宏中，我们可以编写看起来像代码的内容，让编辑器或 IDE 识别。同时，我们还可以对这部分数据进行传递、修改，最后再将其返回给编译器作为 Token，编译到宏调用者的 crate 中。

在内核中，截至目前，该 `quote` 模块的版本为 `1.0.40`。

### quote! 宏

`quote!` 宏对输入进行变量插值后以 `proc_macro2::TokenStream` 的形式输出；在宏中返回 Token 给编译器时，使用 `.into` 将结果转换为 `proc_macro::TokenStream`。这在[后文](#return-type)会讲到其返回值类型的差异。

#### Interpolation

变量插值 (Variable Interpolation) 使用了类似于宏的语法 `#var`。这会在 `quote!` 宏中捕获当前作用域中的 `var` 变量的值。任何实现了 `ToTokens` 特征的类型都可以进行插值。这包括了大多数 Rust 原生类型以及通过 `syn crate` 中的大部分语法树类型。

该语法也可以使用类似于生成宏类似的重复语法，例如：`#(...)*` 或 `#(...), *`。这会遍历任何重复插值的变量元素。

**任何插值的 Token 都保留其 `ToTokens` 实现提供的 `Span` 信息**。

#### Return Type

`quote!` 宏的计算类型为 `proc_macro2::TokenStream` 的表达式；并且 Rust 的过程宏预期的返回类型为 `proc_macro::TokenStream`。

这两种类型的区别在于，**`proc_macro` 类型完全专属于过程宏，并且永远不会出现在过程宏之外的代码中，而 `proc_macro2` 类型可能出现在任何地方，包括测试和非宏代码**。这就是为什么目前过程宏生态也围绕 `proc_macro2` 进行构建，因为这样可以确保库是可单元测试的，并且可在非宏上下文中使用。

#### Example

``` rs
extern crate proc_macro;

use proc_macro::TokenStream;
use quote::quote;

#[proc_macro_derive(HeapSize)]
pub fn derive_heap_size(input: TokenStream) -> TokenStream {
    // Parse the input and figure out what implementation to generate...
    let name = /* ... */;
    let expr = /* ... */;

    let expanded = quote! {
        // The generated impl.
        impl heapsize::HeapSize for #name {
            fn heap_size_of_children(&self) -> usize {
                #expr
            }
        }
    };

    // Hand the output tokens back to the compiler.
    TokenStream::from(expanded)
}
```

上面展示的是一个基本的过程宏结构，在上一章节中我们也有描述。在该构建中，`quote!` 宏会捕获 `#name` 和 `#expr` 对应的变量，然后插值进入，形成一个可编译的完整代码。

通常而言，我们不会一次性地构造出完整的 `TokenStream`。不同的部分可能来自于不同的辅助函数。`quote!` 自身产生的 Token 也实现了 `ToToken`，因此可以被插值到其他 `quote!` 宏中：

``` rs
let type_definition = quote! {...};
let methods = quote! {...};

let tokens = quote! {
    #type_definition
    #methods
};
```

有时候，我们需要以某种方式修改一个变量 (该变量来源于宏输入的某个位置)。直接的修改变量通常没有任何作用，其解决方法通常是**构建一个具有正确值的新 Token，因此 `quote crate` 也提供了 `format_ident!` 来正确的完成这一操作 (关于具体的 `format_ident!` 宏的操作，请参考[后文](#format_ident-宏))：

``` rs
// incorrect
// quote! {
//     let mut _#ident = 0;
// }

let varname = format_ident!("_{}", ident);
quote! {
    let mut #varname = 0;
}
```

当然，在宏中进行构造操作也是十分常见的例子，例如，我们有一个 `field_type` 的变量，其类型为 `syn::Type` 并希望调用该构造函数：

``` rs
// incorrect
quote! {
    let value = #field_type::new();
}
```

这段代码你说完全错误，也不尽然。如果 `field_type` 是 `String` 类型，那么展开后的代码时 `String::new()`，这是能够接受的。如果是类似于 `Vec<i32>` 这样的类型，其展开后的代码是 `Vec<i32>::new()`，这就导致了无效的语法。通常，在宏中，如下方式更为方便：

``` rs
quote! {
    let value = <#field_type>::new();
}
```

当然，这对于 `Trait` 也是一样的。

在 `quote!` 宏中，文档注释和字符串字面量都不会被插值所捕获：

``` rs
quote! {
    /// try to interpolate: #ident
    ///
    /// ...
    #[doc = "try to interpolate: #ident"]
}
```

**构建涉及变量的文档注释的最佳方式是通过在 `quote!` 宏之外格式化文档字符串字面**：

``` rs
let msg = format!(...);
quote! {
    #[doc = #msg]
    ///
    /// ...
}
```

当插值元组或者元组结构体的索引时，通常将其转为 `syn::Index` 的方式进行插值：

``` rs
let i = (0..self.fields.len()).map(syn::Index::from);

// expands to 0 + self.0.heap_size() + self.1.heap_size() + ...
quote! {
    0 #( + self.#i.heap_size() )*
}
```

### quote_spanned! 宏

`quote_spanned!` 与基本的 `quote!` 宏作用相同，都是作用于生成 `TokenStream`，但其增加了一个功能：**显式指定 TokenStream 中所有 Token 的 Span 信息 (代码位置信息)**。

`quote_spanned!` 宏提供了一种 `span` 表达式，该表达式应该尽量简单，如果超过了几个字符的内容，应当使用变量，**在 `=>` 标记前不应该有空格**：

``` rs
let span = /* ... */;

// On one line, use parentheses.
let tokens = quote_spanned!(span=> Box::into_raw(Box::new(#init)));

// On multiple lines, place the span at the top and use braces.
let tokens = quote_spanned! {span=>
    Box::into_raw(Box::new(#init))
};
```

#### Example

下面的过程宏代码会通过 `quote_spanned!` 宏来断言某个特定的 Rust 类型是否实现了 `Sync` 特征，以便安全地在多个线程之间共享引用：

``` rs
let ty_span = ty.span();
let assert_sync = quote_spanned! {ty_span=>
    struct _AssertSync where #ty: Sync;
};
```

`ty.span` 获取了该类型用户源代码定义位置的行和列，而非用户调用宏的哪一行。如果断言失败，用户将看到如下错误，并且其类型输入位置在错误中被高亮显示：

``` bash
error[E0277]: the trait bound `*const (): std::marker::Sync` is not satisfied
  --> src/main.rs:10:21
   |
10 |     static ref PTR: *const () = &();
   |                     ^^^^^^^^^ `*const ()` cannot be shared between threads safely
```

在这个示例中，使用 `span` 的原因是因为 `where` 字句需要带有用户输入类型的行和列信息，这样编译器才能将错误信息正确的显示。

### format_ident! 宏

`format_ident!` 宏用于构建 `Ident` 的格式化宏，其语法是从 `format!` 宏复制过来的，支持位置参数和命名参数：

- `{}` 针对于 [IdentFragment](https://docs.rs/quote/1.0.40/quote/trait.IdentFragment.html)
- `{:o}` 针对于 [Octal](https://doc.rust-lang.org/core/fmt/trait.Octal.html)
- `{:x}` 针对于 [LowerHex](https://doc.rust-lang.org/core/fmt/trait.LowerHex.html)
- `{:X}` 针对于 [UpperHex](https://doc.rust-lang.org/core/fmt/trait.UpperHex.html)
- `{:b}` 针对于 [Binary](https://doc.rust-lang.org/core/fmt/trait.Binary.html)

`format_ident!` 宏支持传递 `span` 用作与最终标识符的跨度，其默认为 `Span::call_site`。

#### Example

``` rs
let my_ident = format_ident!("My{}", "Ident");
assert_eq!(my_ident, "MyIdent");

// If have `r#` prefix, it will be removed by Ident
let raw = format_ident!("r#Raw");
assert_eq!(raw, "r#Raw");

let my_ident_raw = format_ident!("{}Is{}", my_ident, raw);
assert_eq!(my_ident_raw, "MyIdentIsRaw");
```

## Syn 语法解析层

[syn](https://docs.rs/syn/2.0.106/syn/) 库的目的是将 Rust 过程宏的输入 TokenStream 解析成容易操作的、强类型的 抽象语法树 (AST) 结构。它是处理宏输入的核心工具。

syn crate 尽管有一些广泛场景使用的 API，但我们的目的是了解内核如何使用宏，因此只会讲解 syn 中较为常用，且与宏相关的部分。

- Data Structures：syn 提供了一个完整的语法树，可以表示任何有效的 Rust  源代码。该语法树以 `syn::File` 为源头，代表一个完整的源文件，但还有其他入口点可能对过程宏有用，包括 `syn::Item`、`syn::Expr` 和 `syn::Type`。
- Dervies：对于派生宏而言，我们之前见过 `syn::DeriveInput`，其是派生宏的三个合法输入项中的任意一个。
- Parsing：**syn 中的解析功能是围绕具有 `fn(ParseStream) -> Result<T>` 签名的解析函数构建的，每个由 syn 定义的语法树节点都可以单独解析，并且可以作为自定义语法的构建块**。
- Location Information：每个被 syn 解析的 Token 都关联着一个 Span，它会跟踪该 Token 的行号和列号信息，追溯回该 Token 的源代码位置。

目前，内核使用了 `syn v2.0.106` 版本，并在源码中移除了 `unicode-ident`。

## proc-macro 基础 API 层

proc-macro 是 Rust 编译器自带的标准库，它提供了过程宏运行环境所需的基础 API。它位于最底层，是所有宏构建的基础。

## proc-macro2

**`proc_macro` 类型完全专属于过程宏，并且永远不会出现在过程宏之外的代码中，而 [proc_macro2](https://docs.rs/proc-macro2/1.0.101/proc_macro2/) 类型可能出现在任何地方，包括测试和非宏代码。这就是为什么目前过程宏生态也围绕 `proc_macro2` 进行构建，因为这样可以确保库是可单元测试的，并且可在非宏上下文中使用**。

目前，内核使用了 `proc_macro2 v1.0.101` 版本，并在源码中移除了 `unicode-ident`。

## Using in Kernel

在内核中，这三个外部库都是通过 Makefile 中的规则直接进行编译的：

``` Makefile
# rust/Makefile:507
quiet_cmd_rustc_procmacrolibrary = $(RUSTC_OR_CLIPPY_QUIET) PL $@
      cmd_rustc_procmacrolibrary = \
    $(if $(skip_clippy),$(RUSTC),$(RUSTC_OR_CLIPPY)) \
        $(filter-out $(skip_flags),$(rust_common_flags) $(rustc_target_flags)) \
        --emit=dep-info,link --crate-type rlib -O \
        --out-dir $(objtree)/$(obj) -L$(objtree)/$(obj) \
        --crate-name $(patsubst lib%.rlib,%,$(notdir $@)) $<; \
    mv $(objtree)/$(obj)/$(patsubst lib%.rlib,%,$(notdir $@)).d $(depfile); \
    sed -i '/^\#/d' $(depfile)
```

上面这段编译命令则是三个外部库公用的编译逻辑，分别传入不同的参数进行编译。接下来我们首先学习公用部分逻辑。

首先通过 `skip_clippy` 决定最终参与编译的工具，然后通过检查 `skip_flags` 来从 `rust_common_flags` 和 `rustc_target_flags` 中移除指定标志。并且根据目录名生成以类似于 `lib#{dir}.rlib` 的 `rlib` 类型文件 (Rust 静态库)。然后对生成的依赖文件 (.d) 移动到期望的路径，并且删除所有以 `#` 开头的行。

### proc_macro2 build

``` Makefile
# rust/Makefile:517
$(obj)/libproc_macro2.rlib: private skip_clippy = 1
$(obj)/libproc_macro2.rlib: private rustc_target_flags = $(proc_macro2-flags)
```

上面的处理逻辑中，`proc_macro2` 依赖了 `rustc_target_flags` 私有变量，该变量由 `proc_macro2-flags` 构建：

``` Makefile
proc_macro2-cfgs := \
    feature="proc-macro" \
    wrap_proc_macro \
    $(if $(call rustc-min-version,108800),proc_macro_span_file proc_macro_span_location)

proc_macro2-flags := \
    --cap-lints=allow \
    -Zcrate-attr='feature(proc_macro_byte_character,proc_macro_c_str_literals)' \
    $(call cfgs-to-flags,$(proc_macro2-cfgs))
```

这个配置启用了 `feature="proc-macro"`，并且启用了更现代、更详细的 Span 信息。

除此之外，下方是关于 `proc_macro2` 编译的详细命令，并产生了 `libproc_macro2.rlib` 文件。

``` bash
savedcmd_rust/libproc_macro2.rlib := rustc --edition=2021 -Zbinary_dep_depinfo=y -Astable_features -Dnon_ascii_idents -Dunsafe_op_in_unsafe_fn -Wmissing_docs -Wrust_2018_idioms -Wunreachable_pub -Wclippy::all -Wclippy::as_ptr_cast_mut -Wclippy::as_underscore -Wclippy::cast_lossless -Wclippy::ignored_unit_patterns -Wclippy::mut_mut -Wclippy::needless_bitwise_bool -Aclippy::needless_lifetimes -Wclippy::no_mangle_with_rust_abi -Wclippy::ptr_as_ptr -Wclippy::ptr_cast_constness -Wclippy::ref_as_ptr -Wclippy::undocumented_unsafe_blocks -Wclippy::unnecessary_safety_comment -Wclippy::unnecessary_safety_doc -Wrustdoc::missing_crate_level_docs -Wrustdoc::unescaped_backticks --cap-lints=allow -Zcrate-attr='feature(proc_macro_byte_character,proc_macro_c_str_literals)' --cfg='feature="proc-macro"' --cfg='wrap_proc_macro' --cfg='proc_macro_span_file' --cfg='proc_macro_span_location' --emit=dep-info,link --crate-type rlib -O --out-dir ./rust -L./rust --crate-name proc_macro2 rust/proc-macro2/lib.rs; mv ./rust/proc_macro2.d rust/.libproc_macro2.rlib.d; sed -i '/^$(pound)/d' rust/.libproc_macro2.rlib.d
```

### quote build

quote 的编译参数如下所示：

``` Makefile
# rust/Makfile: 522
$(obj)/libquote.rlib: private skip_clippy = 1
$(obj)/libquote.rlib: private skip_flags = $(quote-skip_flags)
$(obj)/libquote.rlib: private rustc_target_flags = $(quote-flags)
```

其中，`skip_flags` 和 `rustc_targe_flags` 作为参数被使用。下面是关于两个参数具体的构建信息：

``` Makefile
# rust/Makefile:92
quote-cfgs := \
    feature="proc-macro"

quote-skip_flags := \
    --edition=2021

quote-flags := \
    --edition=2018 \
    --cap-lints=allow \
    --extern proc_macro2
```

quote 的使用需要启用 `proc-macro` 特性，然后通过 `--extern proc_macro2` 保证 `proc_macro2` 被依赖到 `quote` 编译中。并且限制了所有的 Lint 警告，保证了核心依赖库在编译时不会因警告而中止。

除此之外，下方是关于 `quote` 编译的详细命令，并产生了 `libquote.rlib` 文件。

``` bash
savedcmd_rust/libquote.rlib := rustc -Zbinary_dep_depinfo=y -Astable_features -Dnon_ascii_idents -Dunsafe_op_in_unsafe_fn -Wmissing_docs -Wrust_2018_idioms -Wunreachable_pub -Wclippy::all -Wclippy::as_ptr_cast_mut -Wclippy::as_underscore -Wclippy::cast_lossless -Wclippy::ignored_unit_patterns -Wclippy::mut_mut -Wclippy::needless_bitwise_bool -Aclippy::needless_lifetimes -Wclippy::no_mangle_with_rust_abi -Wclippy::ptr_as_ptr -Wclippy::ptr_cast_constness -Wclippy::ref_as_ptr -Wclippy::undocumented_unsafe_blocks -Wclippy::unnecessary_safety_comment -Wclippy::unnecessary_safety_doc -Wrustdoc::missing_crate_level_docs -Wrustdoc::unescaped_backticks --edition=2018 --cap-lints=allow --extern proc_macro2 --cfg='feature="proc-macro"' --emit=dep-info,link --crate-type rlib -O --out-dir ./rust -L./rust --crate-name quote rust/quote/lib.rs; mv ./rust/quote.d rust/.libquote.rlib.d; sed -i '/^$(pound)/d' rust/.libquote.rlib.d
```

### syn build

syn 的编译参数如下所示：

``` Makefile
# rust/Makefile:528
$(obj)/libsyn.rlib: private skip_clippy = 1
$(obj)/libsyn.rlib: private rustc_target_flags = $(syn-flags)
```

对于 syn 的编译参数就不再进行赘述，其依赖了 `proc_macro2` 和 `quote` 两个库：

``` Makefile
# rust/Makefile:114
syn-flags := \
    --cap-lints=allow \
    --extern proc_macro2 \
    --extern quote
```

除此之外，下方是关于 `syn` 编译的详细命令，并产生了 `libsyn.rlib` 文件。

``` bash
savedcmd_rust/libsyn.rlib := rustc --edition=2021 -Zbinary_dep_depinfo=y -Astable_features -Dnon_ascii_idents -Dunsafe_op_in_unsafe_fn -Wmissing_docs -Wrust_2018_idioms -Wunreachable_pub -Wclippy::all -Wclippy::as_ptr_cast_mut -Wclippy::as_underscore -Wclippy::cast_lossless -Wclippy::ignored_unit_patterns -Wclippy::mut_mut -Wclippy::needless_bitwise_bool -Aclippy::needless_lifetimes -Wclippy::no_mangle_with_rust_abi -Wclippy::ptr_as_ptr -Wclippy::ptr_cast_constness -Wclippy::ref_as_ptr -Wclippy::undocumented_unsafe_blocks -Wclippy::unnecessary_safety_comment -Wclippy::unnecessary_safety_doc -Wrustdoc::missing_crate_level_docs -Wrustdoc::unescaped_backticks --cap-lints=allow --extern proc_macro2 --extern quote --cfg='feature="clone-impls"' --cfg='feature="derive"' --cfg='feature="full"' --cfg='feature="parsing"' --cfg='feature="printing"' --cfg='feature="proc-macro"' --cfg='feature="visit-mut"' --emit=dep-info,link --crate-type rlib -O --out-dir ./rust -L./rust --crate-name syn rust/syn/lib.rs; mv ./rust/syn.d rust/.libsyn.rlib.d; sed -i '/^$(pound)/d' rust/.libsyn.rlib.d
```

---

至此，对于内核中关于宏的依赖库就讲解完毕，下一章中，我们会详细介绍 `macros` 内核库所提供的 API 以及对应的实现。

---

参考链接：

- [quote](https://docs.rs/quote/1.0.40/quote/)
- [syn](https://docs.rs/syn/2.0.106/syn/)
- [proc_macro2](https://docs.rs/proc-macro2/1.0.101/proc_macro2/)
