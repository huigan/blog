上一篇 [（一）solana - 开发环境之windows安装、编译、测试](https://learnblockchain.cn/article/14760)

我们安装好了solana开发环境，现在我们可以使用Anchor框架进行开发了，新手建议使用anchor框架进行学习，相比原生开发会简单很多。

使用anchor框架有2种方式：

- Solana Playground
- 本地开发

# 一、Solana Playground 介绍

Solana Playground  是一个基于浏览器的开发环境，可以快速开发、部署和测试 Solana 程序！
如果熟悉solidity开发，就知道相当于以太坊的remix浏览器开发环境，开发一些简单的合约非常方便。

> 之所以介绍这个，是因为solana的程序和状态是分开保存的，solana账户概念和以太坊的很不一样，通过Solana Playground可以直观的了解solana的账户模型。我刚开始了解的时候被 程序账户、程序派生账户PDA、用户账户、token账户、关联token账户ATA等弄得晕头转向。但是不要担心，通过本系列一步一步了解下来就会清楚solana为什么会有怎么多的账户类型。

浏览器打开 : https://beta.solpg.io/
整体布局和remix基本差不多。
![image.png](https://img.learnblockchain.cn/attachments/2025/04/Ox3uFBKD68102d2f702ba.png)
## 1、创建内置钱包
开始需要创建或者导入钱包，默认连接的是 devnet 链，2种方式获取测试sol：
下方 Playground 终端运行

```
solana airdrop 5
```
或  通过此链接 [Web Faucet](https://faucet.solana.com/) 获取。

## 2、创建Anchor项目
点击左侧 Create a new project 按钮，输入项目名称并选择 anchor框架。
演示程序代码位于 `src/lib.rs` 文件中
![image.png](https://img.learnblockchain.cn/attachments/2025/04/YRbuRIA1681031ca43554.png)

Anchor 框架大量使用 [Rust 宏](https://rust-book.cs.brown.edu/ch20-06-macros.html) 来减少 样板代码并简化进行 Solana 程序开发所需的常见安全检查的实现。如果使用原生开发，则会麻烦很多。

接下来我们来大概了解每一行的含义，重要概念将在后面重点介绍：
```rust
//引入 anchor 框架的预导入模块，提供了开发 Solana 程序所需的基本功能和宏，相比原生开发，可以简化很多代码
use anchor_lang::prelude::*;

//此处使用anchor的宏 设置程序的链上地址。运行build构建指令的时候，它会被自动生成并更新。
declare_id!("11111111111111111111111111111111");

// #[program] 属性定义hello_anchor{}是一个模块，类似于solidity中的 contract {}代码块
#[program]
mod hello_anchor {
    use super::*;
    //此处定义一条指令，后续主要通过此方法与合约交互。
    //参数 ctx: Context<Initialize>：提供对此指令所需账户的访问， `Initialize` 结构体定义了ctx中可以访问的账户。
    //参数data: u64：需要传入一个无符号64位整数。
    pub fn initialize(ctx: Context<Initialize>, data: u64) -> Result<()> {
        ctx.accounts.new_account.data = data;
        msg!("Changed data to: {}!", data); 
        Ok(())
    }
}
//#[derive(Accounts)]宏用于注释一个结构体，定义initialize指令所需的账户，其中每个字段代表一个单独的账户。
#[derive(Accounts)]
pub struct Initialize<'info> {
    // new_account 是我们要存储数据的账号， 使用 `#[account()]` 属性指定了账户的约束条件,如 `init`(初始化)、`payer`(支付租金)、`space` 占用空间大小
    #[account(init, payer = signer, space = 8 + 8)]
    pub new_account: Account<'info, NewAccount>,
    //signer 为签名账户，为交易签名并支付费用。
    #[account(mut)]
    pub signer: Signer<'info>,
    //system_program 为系统程序账号，用于创建PDA账户
    pub system_program: Program<'info, System>,
}

//此处定义new_account账号有那些字段，定义了一个data字段。
#[account]
pub struct NewAccount {
    data: u64
}
```
看完了代码结构，我们来实际构建和交互一下：
### 构建
点击build按钮进行构建，成功后可以看到declare_id!宏中的ID也更新了：

![image.png](https://img.learnblockchain.cn/attachments/2025/04/a7eKfAqn68108f005c9a0.png)

### 部署
点击 Deploy按钮部署，成功后，点击左侧第三个test按钮，Playground会帮我们生成一个指令交互界面
> 由于solana为了防止垃圾账户占用资源，会有一个租金支付，程序部署成功后会有几个sol的租金产生，程序销毁后即可返还。

![image.png](https://img.learnblockchain.cn/attachments/2025/04/pdOzFj0w6810900224a22.png)

### 交互
可以看到initialize指令，传值分2部分：Args（参数）、Accounts（程序需要的账户）
- Args中的data即我们在initialize指令方法中定义的data参数，传入任意无符号整数即可。
- Accounts 部分稍微复杂一点，也是理解solana账号逻辑的关键：前面提到solana的程序和状态是分开的，交互的时候需要开发者传入对应的账户。
  `newAccount`  ：点击输入框，可通过 RANDOM 随机生成（也可通过seek方式固定生成）。
  `signer`：默认填写为当前钱包地址。
  `systemProgram`：系统程序账号为 11111111111111111111111111111111


![image.png](https://img.learnblockchain.cn/attachments/2025/04/dayVixfb68109a58d515f.png)
然后点击Test按钮，即可发起一笔交易。
可通过下方的Accounts功能中的fetch All ,获取当前程序中的所有NewAccount类型账户来查看数据，可以看到 **271KjUJzDEjUcsXWEo3gmQp5Xg6Hp3FKF8w9etZxDEWj** 下面的**data**字段设置为1了。

![image.png](https://img.learnblockchain.cn/attachments/2025/04/OsJlwGfE68109a41728a0.png)

# 二、Anchor本地开发

Solana Playground浏览器开发确实很方便，但是开发一个复杂的合约，不管是从部署还是测试等方面来说，还是使用本地环境比较好。
[上一篇](https://learnblockchain.cn/article/14760) 我们已经本地初始化了一个anchor程序，第一步我们先来了解anchor的目录结构：

```shell
//初始化一个anchor项目
anchor init counter
```

![image.png](https://img.learnblockchain.cn/attachments/2025/04/XVb0Xn9s6810dbc339f93.png)
## 1、目录结构
我们先来介绍几个常用的目录，其他的后面可以慢慢了解。

- `/app` 默认是一个空文件夹，用于存放前端代码（后面学习solana前端开发的时候我们在学习）。
- `/programs` 目录存放项目的 Anchor 程序。单个工作区可以存在多个程序，当前只有一个counter程序。
- `/programs/counter` 目录为counter程序的目录 
- `/target` 目录包含构建输出。主要的子文件夹包括：
    * `/deploy`：包含程序的密钥对和程序二进制文件，运行build后，代码中的declare_id!宏设置的值需要和密钥对的公钥一致。
    * `/idl`：程序的 JSON IDL，和以太坊合约的ABI文件作用一致，供客户端交互使用。
    * `/types`：包含 IDL 的 TypeScript 类型。
- `/tests` 目录存放项目的测试文件。anchor会创建一个默认的测试文件。
-  `Anchor.toml` 是anchor项目的配置文件，也是非常重要的一个文件，我们先简单了解几个重要的配置，详细的配置可以查看文档 <https://www.anchor-lang.com/docs/manifest>
```toml
[toolchain]
package_manager = "yarn"

[features]
resolution = true
skip-lint = false

[programs.localnet]
# 程序ID(公钥)
counter = "BsWBY6XfX9r6YSs4YN8z2T6JDWvw16SKTtZuDKTEzVzE"

[registry]
url = "https://api.apr.dev"

[provider]
# 指定要连接到的集群，可选 "localnet","Devnet", "Mainnet" 或自定义 RPC URL
cluster = "localnet"
# 指定私钥文件的路径，用于签名交易。
wallet = "~/.config/solana/id.json"

[scripts]
# 定义测试脚本的命令
test = "yarn run ts-mocha -p ./tsconfig.json -t 1000000 tests/**/*.ts"
```
## 2、程序结构
### 宏
想要看的懂Anchor框架中的程序，就需要了解几个重要宏的作用：
> 了解了这些宏，基本就能掌握solana合约程序的运行逻辑和的整体结构了。
* `declare_id`: 定义程序的链上地址（程序 ID），anchor框架build后可自动生成；也可以通过自定义生成指定前缀地址的密钥对，然后替换/target/deploy中的程序密钥对，来生成合约靓号的目的。。
* `#[program]`: 定义指令模块，主要程序逻辑都存放在这里面，模块可以包含多条可被调用的函数方法，即指令；类似solidity中 contract合约的概念，contract中又包含多个方法。。
* `#[derive(Accounts)]`: 定义指令运行所需的账户列表，solana的程序和状态是分开存储的，所以需要传入指令运行所需要操作的账号，此宏基本都是用在方法第一个参数context类型。
* `#[account]`: 定义 账户类型，就是我们合约逻辑中需要存储数据的结构类型。

```rust
use anchor_lang::prelude::*;

declare_id!("11111111111111111111111111111111");

#[program]
mod hello_anchor {
    use super::*;
    pub fn initialize(ctx: Context<Initialize>, data: u64) -> Result<()> {
        ctx.accounts.new_account.data = data;
        msg!("Changed data to: {}!", data); 
        Ok(())
    }
}

#[derive(Accounts)]
pub struct Initialize<'info> {
    #[account(init, payer = signer, space = 8 + 8)]
    pub new_account: Account<'info, NewAccount>,
    #[account(mut)]
    pub signer: Signer<'info>,
    pub system_program: Program<'info, System>,
}

#[account]
pub struct NewAccount {
    data: u64
}
```


### 函数方法
在模块中我们可以定义多个不同指令的函数方法，之前anchor默认创建的程序中已经定义了一个函数方法：`pub fn initialize(ctx: Context<Initialize>, data: u64)` ，这个方法用于初始化账号和账号的data字段。

**指令上下文**
重点介绍一下 指令上下文 概念。
`ctx` 是一个 `Context<T>` 类型的包含上下文信息的结构体，**每个函数的第一个参数都是这种类型**，使用泛型 `T` 来指定指令函数所需的具体账户集合，开发者需要定义一个结构体来作为 `Context`的泛型参数，比如示例代码中的 `Context<Initialize>`，`Initialize` 在下面定义为：
```rust
#[derive(Accounts)]
pub struct Initialize<'info> {
    #[account(init, payer = signer, space = 8 + 8)]
    pub new_account: Account<'info, NewAccount>,
    #[account(mut)]
    pub signer: Signer<'info>,
    pub system_program: Program<'info, System>,
}
```
需要用使用 `#[derive(Accounts)] `宏来注释，其中字段可以自定义，比如 `new_account` 可以更换为 `newuser`，`signer`更换为 `payer`等。也可以增加一个字段名为user2的PDA账户：
```rust
#[derive(Accounts)]
pub struct Initialize<'info> {
    #[account(init, payer = signer, space = 8 + 8)]
    pub new_account: Account<'info, NewAccount>,
    
    #[account(init, payer = signer, space = 8 + 8)]
    pub user2: Account<'info, NewAccount>,
    
    #[account(mut)]
    pub signer: Signer<'info>,
    pub system_program: Program<'info, System>,
}
```
通过`ctx.accounts`可以获取指令逻辑所需的账户，比如 `ctx.accounts.new_account`或 `ctx.accounts.user2`。
>    'info 是生命周期参数，避免悬垂引用问题，不熟悉的可以搜索一下rust的生命周期，此处开发者可以无需过度关注。

**Initialize 结构**
1. `new_account`: 自定义PDA数据账户，用于存储计数器数据，字段由 `NewAccount`定义。

**#[account(...)]** 是账号约束：定义了账户必须满足的额外条件，其中：
-    init: 表示此账号需要初始化
-    payer: 指定由下面的类型为Signer的signer账号支付账户创建费用
-    space: 分配8+8字节的存储空间
2. `signer`: 交易签名者账户，用于支付交易费用
3. `system_program`: Solana系统程序账户，用于创建PDA账户






**新增函数方法**
`initialize` 方法只能创建账号，不能更新`new_account`中的 `data`，如果`new_account`传入的账号相同，再次调用此方法，会发现调用失败，错误信息为：

```
Allocate: account Address { address: 7hZh8namm9LG6Yy64RJXadjnZqXAZiUP97cRnx8bhUH2, base: None } already in use
```

所以我们需要新增一个 `updateaccount` 更新方法，同时对比和 `initialize` 方法的不同：

```rust
use anchor_lang::prelude::*;

declare_id!("11111111111111111111111111111111");

#[program]
mod hello_anchor {
    use super::*;
    pub fn initialize(ctx: Context<Initialize>, data: u64) -> Result<()> {
        ctx.accounts.new_account.data = data;
        msg!("Changed data to: {}!", data); 
        Ok(())
    }
    
    pub fn updateaccount(ctx: Context<UpdateAccountStruct>,data:u64) -> Result<()> {
        ctx.accounts.exist_account.data = data;
        msg!("exist_account data: {:?}", ctx.accounts.exist_account.data);
        Ok(())
    }
}

#[derive(Accounts)]
pub struct Initialize<'info> {
    #[account(init, payer = signer, space = 8 + 8)]
    pub new_account: Account<'info, NewAccount>,
    #[account(mut)]
    pub signer: Signer<'info>,
    pub system_program: Program<'info, System>,
}

#[derive(Accounts)]
pub struct UpdateAccountStruct<'info> {
    #[account(mut)]
    pub exist_account:Account<'info,Newaccount>,
}

#[account]
pub struct NewAccount {
    data: u64
}
```

可以看到第一个参数类型也是 `Context`，但是泛型类型使用了一个新的 `UpdateAccountStruct`，

```rust
#[derive(Accounts)]
pub struct UpdateAccountStruct<'info> {
    #[account(mut)]
    pub exist_account:Account<'info,Newaccount>,
}
```
和 `Initialize` 相比，第一个字段也是定义了一个数据账户，但是账户约束少了很多，只是使用了 `mut`表示可修改，不再使用 `init`、`payer`、`space`。因为此函数只是需要更新已存在账户的数据，不需要初始化了，也就不再需要由其他账户来支付创建费用，所有 `signer`、`system_program`也都不再需要了。

## 3、测试
接下来我们写一个测试脚本，来测试我们的新合约代码是否可以正常运行。
使用以下代码替换` /tests/counter.ts`中的代码：


```
import * as anchor from "@coral-xyz/anchor";
import { Program } from "@coral-xyz/anchor";
import { Counter } from "../target/types/counter";
import { BN } from "bn.js";

describe("counter", () => {
  // Configure the client to use the local cluster.
  const provider = anchor.AnchorProvider.env();
  anchor.setProvider(anchor.AnchorProvider.env());

  const program = anchor.workspace.counter as Program<Counter>;

  const signer = provider.wallet;
  const data = new BN(111);
  const newAccountPair = new anchor.web3.Keypair();


  it("Is initialized!", async () => {
    // Add your test here.
    const tx = await program.methods.initialize(data).accounts({
      newAccount:  newAccountPair.publicKey,
      signer:signer.publicKey,
    
    }).signers([newAccountPair]).rpc();
    console.log("Your transaction signature", tx);

    const newaccount = await program.account.newaccount.fetch(newAccountPair.publicKey);

    console.log("newaccount", newaccount.data.toString());

    const data2 = new BN(222);
    const tx2 = await program.methods.updateaccount(data2).accounts({
      existAccount:  newAccountPair.publicKey,
    }).rpc();
    
    const newaccount2 = await program.account.newaccount.fetch(newAccountPair.publicKey);

    console.log("newaccount", newaccount2.data.toString());

  });
});
```
测试中，我们先创建了一个数据账号 `newAccountPair`，通过 `initialize`方法初始化数据账号，并将`data`设置为111，然后又调用 `updateaccount`，将数据账号 `newAccountPair`的 `data`修改为222。

运行测试命令：

```
anchor test 
```

## 总结
我们通过anchor框架实现了一个简单的保存数据的合约程序，并了解了 宏的使用、函数方法和参数结构体的定义。
