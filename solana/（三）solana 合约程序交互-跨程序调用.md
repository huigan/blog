通过之前的学习，我i们已经可以写一个简单的solana合约程序了，但是区块链有一个很重要的特性，就是可组合性，每个程序都可以互相调用对方的接口来完成复杂的操作。

比如热门应用 **pump.fun**（一个代币公平发射平台），用户支付0.2个sol，就可以在pump.fun的程序中创建一个SPL Token代币，并可以在pump程序中交易，一旦买入的sol达到一定数量，则会在Raydium（去中心化交易所）中添加流动性，使代币可以在Raydium中进行交易（整个逻辑比这个要复杂很多，后面我们再学习怎么实现一个pump.fun的应用）。

这里我们的程序有3个地方需要调用外部程序：
1. 调用系统程序，从用户的账户中转移0.2个`sol`到指定的账户。
2. 调用SPL Token程序创建一个代币
3. 调用Raydium程序添加流动性。

接下来的文章将一步一步实现这3个功能。

要实现上面的功能，我们就需要用到solana中一个很重要的功能，`跨程序调用（CPI:Cross Program Invocation）`。

## 跨程序调用 CPI
跨程序调用（Cross Program Invocation，CPI）是指一个程序调用另一个程序的指令。

构建CPI指令需要：

* **程序地址**：指定要调用的程序
* **账户**：列出指令读取或写入的每个账户，包括其他程序
* **指令数据**：指定要在程序上调用的指令，以及指令所需的任何数据（函数参数）


### 调用系统程序进行sol转账
我们先实现转账sol的功能，合约中转移sol主币其实是在当前程序中调用系统程序，所以也属于跨程序调用了。
现在我们使用anchor框架的 `CpiContext` 和辅助函数来构建 CPI 指令：

> 手动构建 CPI 指令调用相对比较复杂，但是如果没有对应的 crate 构建相应的指令时，我们只能自己手动构建。


```rust
use anchor_lang::prelude::*;
use anchor_lang::system_program::{transfer, Transfer};

declare_id!("9AvUNHjxscdkiKQ8tUn12QCMXtcnbR9BVGq3ULNzFMRi");

#[program]
pub mod cpi {
    use super::*;

    pub fn sol_transfer(ctx: Context<SolTransfer>, amount: u64) -> Result<()> {
        let from_pubkey = ctx.accounts.sender.to_account_info();
        let to_pubkey = ctx.accounts.recipient.to_account_info();
        let program_id = ctx.accounts.system_program.to_account_info();

        let cpi_context = CpiContext::new(
            program_id,
            Transfer {
                from: from_pubkey,
                to: to_pubkey,
            },
        );

        transfer(cpi_context, amount)?;
        Ok(())
    }
}

#[derive(Accounts)]
pub struct SolTransfer<'info> {
    #[account(mut)]
    sender: Signer<'info>,
    #[account(mut)]
    recipient: SystemAccount<'info>,
    system_program: Program<'info, System>,
}
```

`CpiContext::new(program, accounts)` 其中 ：
第一个参数 program 为需要调用的程序ID。
第二个为转移sol所需的账户结构列表。

最后通过anchor封装的 system_program::transfer 进行sol转账操作，通过CPI实现了从sender账户给recipient账号转移sol的功能。

但是如果你熟悉以太坊开发，经常会需要把eth主币转到合约地址中存储的场景，我们这个示例只是从用户账户转给了其他用户账户，solana中的程序和状态是分离的，如果想要实现类似功能，就需要用到程序派生地址PDA。

**什么是程序派生地址 PDA？**


**Program Derived Address (PDA)** 主要有以下特性：

* **确定性账户地址**：PDA 是一种地址，并提供了一种机制，可以使用可选的 seed、bump seed 和程序 ID的组合来确定性地创建地址。
* **支持程序签名**：PDA看起来像公钥，但没有私钥。这意味着无法为该地址生成有效的签名。然而，Solana 运行时允许程序无需私钥即可为 PDA "签名"。


> 您可以将 PDA 理解为一种在链上从预定义输入（例如字符串、数字和其他账户地址）创建类似哈希表结构的方式。这种方法的好处在于，它消除了需要跟踪确切地址的需求。相反，您只需记住用于推导地址的特定输入即可。

下面的代码实现了用户账户和PDA之间互转sol的功能：
```rust
use anchor_lang::prelude::*;
use anchor_lang::system_program::{transfer, Transfer};

declare_id!("BrcdB9sV7z9DvF9rDHG263HUxXgJM3iCQdF36TcxbFEn");

#[program]
pub mod cpi {
    use super::*;

    pub fn sol_transfer(ctx: Context<SolTransfer>, amount: u64) -> Result<()> {
        let from_pubkey  = ctx.accounts.user.to_account_info();
        let to_pubkey= ctx.accounts.pda_account.to_account_info();
        let program_id = ctx.accounts.system_program.to_account_info();

        let cpi_context = CpiContext::new(
            program_id,
            Transfer {
                from: from_pubkey,
                to: to_pubkey,
            },
        );

        transfer(cpi_context, amount)?;
        Ok(())
    }
    
     pub fn sol_transfer2(ctx: Context<SolTransfer>, amount: u64) -> Result<()> {
        let from_pubkey = ctx.accounts.pda_account.to_account_info();
        let to_pubkey = ctx.accounts.user.to_account_info();
        let program_id = ctx.accounts.system_program.to_account_info();

        let bump_seed = ctx.bumps.pda_account;
        let signer_seeds: &[&[&[u8]]] = &[&[b"pda", &[bump_seed]]];

        let cpi_context = CpiContext::new(
            program_id,
            Transfer {
                from: from_pubkey,
                to: to_pubkey,
            },
        )
        .with_signer(signer_seeds);

        transfer(cpi_context, amount)?;
        Ok(())
    }
}

#[derive(Accounts)]
pub struct SolTransfer<'info> {
    #[account(
        mut,
        seeds = [b"pda"],
        bump,
    )]
    pda_account: SystemAccount<'info>,
    #[account(mut)]
    user: SystemAccount<'info>,
    system_program: Program<'info, System>,
}
```


在ctx的 `SolTransfer` 结构体中定义了一个PDA账户，由 `seeds` 、 `bump` 和 `program_id` 构成，`seeds`中的 “pda” 可替换为其他字符。

```js
#[derive(Accounts)]
pub struct SolTransfer<'info> {
#[account(
    mut,
    seeds = [b"pda"],
    bump,
)]
pda_account: SystemAccount<'info>,
#[account(mut)]
user: SystemAccount<'info>,
system_program: Program<'info, System>,
}
```
通过 solana playground 调用指令时，可以看到 pda_account 需选择From seed方式，并在seed中填写pda生成的。这样就可以实现所有用户转账sol都会到同一个PDA账户中。
![image.png](https://img.learnblockchain.cn/attachments/2025/05/uIgmFIss6824b9fe9caf4.png)

`sol_transfer` 和 `sol_transfer2` 分别实现了 用户账户->PDA账户 和 PDA账户->用户账户 转sol的功能。
但是`sol_transfer2` 中构建CPI的时候需要用到 `.with_signer(signer_seeds)` 用于指定 PDA seeds 作为签名者，这样确保只能由此程序转移此PDA账户中的资产。

## 总结
现在我们初步了解了solana程序之间是如何调用的，通过跨程序调用 CPI 实现程序之间的可组合性。
构建CPI指令的核心参数：

* **程序地址**：指定要调用的程序
* **账户**：列出指令读取或写入的每个账户，包括其他程序
* **指令数据**：指定要在程序上调用的指令，以及指令所需的任何数据（函数参数）

然后我们又学习了程序派生地址 PDA，可以确定性的生成一个账户地址，并由程序来控制。

下面我们将了解更核心的概念，solana中Token代币相关的知识。

欢迎关注微信公众号获取最新文章，wx：向日葵web3


![](https://img.learnblockchain.cn/attachments/2025/05/8a2MgLxd682ddb1d10884.jpg)



