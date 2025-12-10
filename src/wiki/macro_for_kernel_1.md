# 内核中 Rust 宏的使用手册 (一)

在上一节中，我们介绍了内核的 bindings 模块，其用于生成 FFI 绑定代码，并且不再使用 `core::ffi` 而是通过自定义类型系统使 FFI 生成的代码与平台实现无关。本节，我们将会介绍 Rust for Linux 中宏的使用。在这之前，我们先简单了解 Rust 宏的基本使用。

## Rust Macro for Standard

在 Rust 编程中，我们无时无刻都在接触宏，比如我们想要打印一个变量，则会用到 `println!`。Rust 的宏分为了两种类型：声明式宏 (declarative macros) 和过程式宏 (procedural macros)。其中，过程式宏又分为三种类型：

- 派生宏 (derived macros)：类似于 `#\[derive]` 这样的派生宏，用于在结构体和枚举上添加属性的代码，常见于 `#[derive(Debug)]`
- 类特性宏 (Attribute-like macros)：用于定义可在任何项上使用的自定义属性
- 类函数宏 (Function-like macros)：类似于函数调用的宏

> 为什么已经有了函数的情况下还需要宏？  
> 如果了解过 C++ 语言的读者就很容易明白一个概念：元编程技术 (Metaprogramming, 用来编写代码的代码)。元编程能够显著的减少编写和维护代码的行数，当然这也导致了后续维护的复杂性和可读性较差。在 Rust 中，函数签名必须声明函数拥有的参数数量和类型，而宏可以接受一个可变数量的参数。  
> Rust 的宏和函数之间的另一个重要区别是：在文件中调用宏之前必须定义宏或声明引入作用域，而函数则可以在任意地方定义并在任意地方调用。

### Declarative Macros

Rust 中使用最为广泛的宏是声明式宏，也被称为通过示例定义的宏。本质上来说，声明式宏就类似于通过 Rust `match` 表达式与模式进行比较，然后运行匹配对应模式的相关联的代码。**这些操作均在编译器进行完成**。

要定义一个声明式宏，Rust 中提供了  `macro_rules!` 构造器用于操作，我们通过一个简化的 `vec!` 宏来学习如何使用：

``` rust
#[macro_export]
macro_rules! vec {
	( $( $x:expr ),* ) => {
		{
			let mut temp_vec = Vec::new();
			$(
				temp_vec.push($x);
			)*
			temp_vec
		}
	};
}
```

`#[macro_export]` 注解表示这个宏允许被 `crate` 引入作用域，其作用类似于 C 语言的 `export` 关键字。`macro_rules!` 后跟上一个宏的名字，该名字则是宏的整体宏名，后续使用则是 `name!`。

完成整体的声明部分后，我们来分析宏的主体部分。首先我们可以通过如下的一个伪代码来了解声明式宏的主体部分结构：

``` bash
#[macro_export]
macro_rules! macro_name {
	pattern 1 => expression 1,
	pattern 2 => {
		statement 1;
		statement 2;
		...
		expression 2
	},
	_ => expression 3
}
```

因此，`($($x:expr), *)` 则是第一个模式，首先我们使用 `()` 来包裹整个模式，然后通过 `$` 在 Rust 的宏系统中声明一个变量，该符号表明这是一个宏变量，而非普通的 Rust 变量。`$()` 包含了与模板匹配的 Rust 代码，而 `$x` 则是会从匹配到的 `expr` 表达式中捕获对应的值，`,` 表示在该声明式宏中，Rust 的传入的每个参数都使用逗号进行分割，而 `*` 表明了该模式匹配零个或多个与之前相匹配的任意内容。

``` rust
# #[macro_export]
# macro_rules! vec {
#   ( $( $x:expr ),* ) => {
		{
			let mut temp_vec = Vec::new();
			$(
				temp_vec.push($x);
			)*
			temp_vec
		}
# };
# }
```

当模式匹配正确，上方的语句就会随之生成对应的代码，`$(temp_vec.push($x);)*` 表示会根据匹配到的个数，生成对应个数的重复内容，`$x` 则会根据匹配的值进行变化。

``` rust
# #[macro_export]
# macro_rules! vec {
# 	( $( $x:expr ),* ) => {
# 		{
# 			let mut temp_vec = Vec::new();
# 			$(
# 				temp_vec.push($x);
# 			)*
# 			temp_vec
# 		}
# 	};
# }

let x = vec![1, 2, 3];
println!("{:?}", x);
```

对于具体的 Rust 生成式宏的语法，可以详细参考 [The Rust Reference - Macros Declaration](https://doc.rust-lang.org/reference/macros-by-example.html) 进行查看。

### Procedural Macros

过程式宏在 Rust 中经常使用，从形式上来看，过程宏与函数较为相像，但是过程宏是使用源代码作为输入参数，基于代码进行一系列操作后，再输出一段全新的代码。**值得注意的是，过程宏中的派生宏输出的代码并不会替换之前的代码**。

**在创建过程宏时，它的定义必须要放入一个独立的包中，并且包的类型也是特殊的**。

> 过程宏放入独立包的原因在于它必须先被编译后才能使用，如果过程宏和使用它的代码在一个包，就必须先单独对过程宏的代码进行编译，然后再对我们的代码进行编译，但悲剧的是 Rust 的编译单元是包，因此你无法做到这一点。

假如我们要创建一个派生宏，类似于：

``` rs
#[derive(HelloMacro)]
struct Dog;

#[derive(HelloMacro)]
struct Cat;

fn main() {
  Dog::hello_macro();
  Cat::hello_macro();
}
```

这个小节会分别通过手动实现和实现派生宏来进行对比，用于分析过程宏的作用与实现。

#### 手动实现

手动实现十分简单，我们只需要利用 Rust 提供的 `trait` 机制即可：

``` rust
trait HelloMacro {
  fn hello_macro();
}

struct Dog;

impl HelloMacro for Dog {
  fn hello_macro() {
    println!("Hello, macro! This is Dog!");
  }
}

struct Cat;

impl HelloMacro for Cat {
  fn hello_macro() {
    println!("Hello, macro! This is Cat!");
  }
}

Cat::hello_macro();
Dog::hello_macro();
```

相信读者已经发现了手动实现的问题在哪，**如果想要实现不同的招呼内容，就需要为每一个类型都实现一次对应的特征，并且 Rust 不支持反射，我们也无法在运行时获得类型名**。

#### 派生宏实现

如果我们实现了过程宏，就不再出现手动实现的问题。为了更好的显示，代码均存放在同一个文件中，**但读者在实际书写时，切记过程宏必须以一个包为结构进行编写，并且派生宏的名字一定要以 `derive` 作为后缀**。

``` rs
extern crate proc_macro;

use proc_macro::TokenStream;
use quote::quote;
use syn;
use syn::DeriveInput;

#[proc_macro_derive(HelloMacro)]
pub fn hello_macro_derive(input: TokenStream) -> TokenStream {
    // 基于 input 构建 AST 语法树
    let ast:DeriveInput = syn::parse(input).unwrap();

    // 构建特征实现代码
    impl_hello_macro(&ast)
}
```

对于绝大多数过程宏而言，这段代码是通用的，只在 `impl_xxx_macro` 中的实现有所区别。`proc_macro` 是 Rust 的官方库，包含了编译器的 API，可以用于读取和操作 Rust 源代码。

我们通过 `#[proc_macro_derive(HelloMacro)]` 来标记 `hello_macro_derive` 函数；当用户使用 `#[derive(HelloMacro)]` 时，则相当于变相的调用了 `hello_macro_derive` 函数；`#[proc_macro_derive(HelloMacro)]` 相当于一个链接器，将用户的类型与过程宏联系在一起。

`syn` 将字符串形式的 Rust 代码解析为一个 AST 树的数据结构，该数据结构可以在随后的 `impl_hello_macro` 函数中进行操作。最后，操作的结果又会被 `quote` 包转换回 Rust 代码。这些包非常关键，可以帮我们节省大量的精力，否则你需要自己去编写支持代码解析和还原的解析器，这可不是一件简单的任务！

我们可以简单看一下 `input` 和 `syn::parse` 的内容：

``` rs
Input: TokenStream {
    Ident {
        ident: "struct",
        span: #0 bytes(87..93),
    },
    Ident {
        ident: "Cat",
        span: #0 bytes(94..97),
    },
    Punct {
        ch: ';',
        spacing: Alone,
        span: #0 bytes(97..98),
    },
}

Ast: DeriveInput {
    // --snip--
    vis: Visibility,
    ident: Ident {
        ident: "Cat",
        span: #0 bytes(94..97)
    },
    generics: Generics,
    // Data是一个枚举，分别是DataStruct，DataEnum，DataUnion，这里以 DataStruct 为例
    data: Data(
        DataStruct {
            struct_token: Struct,
            fields: Fields,
            semi_token: Some(
                Semi
            )
        }
    )
}
```

对于 `proc_macro`、`syn` 和 `quote` 的详细分析均在后续的文章中进行分析，此处不做过多讲述。

``` rs
fn impl_hello_macro(ast: &syn::DeriveInput) -> TokenStream {
  let name = &ast.ident;
  let gen = quote! {
    impl HelloMacro for #name {
      fn hello_macro() {
          println!("Hello, Macro! My name is {}!", stringify!(#name));
      }
    }
  };
  gen.into()
}
```

从上文中得知，`ast.ident` 实际上就是类名，因此通过 `quote!` 宏重建代码，并最终通过 `into()` 函数生成 `TokenStream`。通过运行 `cargo expand`，我们也可以看到对应的展开宏：

``` rs
struct Cat;
impl HelloMacro for Cat {
    fn hello_macro() {
        {
            ::std::io::_print(format_args!("Hello, Macro! My name is {0}!\n", "Cat"));
        };
    }
}
```

从展开的代码也能看出 derive 宏的特性，`struct Cat;` 被保留了，也就是说最后 `impl_hello_macro()` 返回的 token 被加到结构体后面，这和类属性宏可以修改输入的 token 是不一样的，input 的 token 并不能被修改。

#### 类属性宏

类属性过程宏跟 `derive` 宏类似，但是前者**允许我们定义自己的属性**。除此之外，`derive` 只能用于结构体和枚举，而类属性宏可以用于其它类型项，例如函数。

假设我们在开发一个 `web` 框架，当用户通过 `HTTP GET` 请求访问 `/` 根路径时，使用 index 函数为其提供服务：

``` rs
#[route(GET, "/")]
fn index() {}
```

如上所示，代码功能非常清晰、简洁，这里的 `#[route]` 属性就是一个过程宏，它的定义函数大概如下：

``` rs
#[proc_macro_attribute]
pub fn route(attr: TokenStream, item: TokenStream) -> TokenStream {}
```

与 `derive` 宏不同，类属性宏的定义函数有两个参数：

- 第一个参数时用于说明属性包含的内容：`Get`, `/` 部分
- 第二个是属性所标注的类型项，在这里是 fn index() {...}，注意，函数体也被包含其中

除此之外，类属性宏跟 `derive` 宏的工作方式并无区别：创建一个包，类型是 `proc-macro`，接着实现一个函数用于生成想要的代码。

#### 类函数宏

类函数宏可以让我们定义像函数那样调用的宏，从这个角度来看，它跟声明宏 `macro_rules` 较为类似。

区别在于，`macro_rules` 的定义形式与 `match` 匹配非常相像，而类函数宏的定义形式则类似于之前讲过的两种过程宏：

``` rs
#[proc_macro]
pub fn sql(input: TokenStream) -> TokenStream {}
```

而使用形式则类似于函数调用：

``` rs
let sql = sql!(SELECT * FROM posts WHERE id=1);
```

大家可能会好奇，为何我们不使用声明宏 `macro_rules` 来定义呢？原因是这里需要对 `SQL` 语句进行解析并检查其正确性，这个复杂的过程是 `macro_rules` 难以对付的，而过程宏相比起来就会灵活的多。

在下一章中，我们会介绍 `queto` 包的使用方式，从使用到原理，一步步深入内核各个组件。

--- 

参考链接

- [Rust Book](https://doc.rust-lang.org/book/ch20-05-macros.html)
- [Rust Course](https://course.rs/advance/macro.html)
- [Rust Reference - Macro](https://doc.rust-lang.org/reference/macros-by-example.html)
