上一篇文章 我们初步了解了 solana中程序之间如何交互。并完成了程序中转账sol。
现在我们来了解Solana中SPL TOKEN 的相关知识。这篇文章我们将实现合约程序内创建代币、mint代币、转账代币，以及如何更新代币名称、简称、logo。

## Token Program
在以太坊中部署一个代币，就需要新部署一个合约。solana中由于程序和状态是分开存储的，所以我们不需要新部署一个合约程序，solana官方已经有对应的2个代币合约程序（`Token Program`）：
1. [Token Program](https://github.com/solana-program/token) 包含基本的代币功能（铸币、转账等）

2. [Token Extension Program (Token 2022)](https://github.com/solana-program/token-2022) 在 Token Program 的基础上， 通过“扩展”添加了额外的功能，比如  转账税、持币生息、交易费用。

### 创建代币-Token Mint
Token Program 和 Token 2022 都有一个 Mint账户 结构实现，创建一个Mint账户，每个MInt账户就代表一个代币，Mint账户地址就是代币地址。

#### Mint账户结构
创建MInt账户需要定义以下主要参数：
- supply：总供应量，即代币数量
- decimals：代币小数位
- authority：可以铸造代币的权限地址
- freeze_authority：可以冻结代币账户的权限地址，使某个用户无法转账代币。

> 熟悉以太坊的代币开发的话，会发现一个问题，这里没有定义代币的 name 、symbol等信息，solana中这些信息是存放在另外的程序中，[metaplex](https://www.metaplex.com/)协议定义了代币的这些信息如何保存，后面我们将看到如何交互。

比如solana上 USDT 的代币地址是 Es9vMFrzaCERmJfrF4H2FYD4KCoNkY11McCe8BenwNYB ，可以在solana浏览器中[查看详情](https://solscan.io/token/Es9vMFrzaCERmJfrF4H2FYD4KCoNkY11McCe8BenwNYB)。

#### 使用 anchor-spl 与Token Program交互
接下来我们在anchor程序中调用Token Program来创建一个Mint账户。`anchor-spl` 是anchor中的一个crate（可以理解为一个封装好的程序代码），使用它可以简化与Token Program的交互。
使用前需要在我们的anchor程序中添加 `anchor-spl` ：

```js
//初始化anchor程序
anchor init create_mint
//安装anchor-spl
cargo add anchor-spl
```

![image.png](https://img.learnblockchain.cn/attachments/2025/05/nACIppuX68312cade6772.png)
我们成功安装了anchor-spl v0.31.1版本，然后在 Cargo.toml 文件中添加：

```js
[features]
idl-build = [
    "anchor-lang/idl-build",
    "anchor-spl/idl-build",
]

    [dependencies]
anchor-lang = "0.31.1"
anchor-spl = "0.31.1"

```

然后替换lib,rs中的代码为：

```js
use anchor_lang::prelude::*;
//引入anchor-spl的token_interface模块
use anchor_spl::token_interface::{Mint, TokenInterface};
 
declare_id!("2pNAGP3UHPrEFQ9q46DTNofA9xGesMjV231aTxCuTjw2");
 
#[program]
pub mod create_token {
    use super::*;
 
    pub fn create_mint(ctx: Context<CreateMint>) -> Result<()> {
        msg!("Created Mint Account: {:?}", ctx.accounts.mint.key());
        Ok(())
    }
}
 
#[derive(Accounts)]
pub struct CreateMint<'info> {
    #[account(mut)]
    pub signer: Signer<'info>,
    //此处通过账户约束来定义mint账户
    #[account(
        init,
        payer = signer,
        mint::decimals = 6, //定义代币的小数位
        mint::authority = signer.key(), //定义代币的铸造权限地址，此处我们设置为signer
        mint::freeze_authority = signer.key(), //定义代币的冻结权限地址，此处我们设置为signer
    )]
    pub mint: InterfaceAccount<'info, Mint>,
    pub token_program: Interface<'info, TokenInterface>,
    pub system_program: Program<'info, System>,
}

```
在 `create_mint` 方法的上下文 `CreateMint` 中我们定义了一个 `mint` 账户，它的类型是 `InterfaceAccount`，它的作用是提供一个统一的接口来处理 Token 账户，无论该账户是由Token Program 或 Token 2022 程序创建的。这样可以使你的 Anchor 程序更加灵活，能够兼容不同类型的 Token 账户。

然后我们写一个测试脚本调用 `create_mint` 方法来创建一个 mint账户。将下面的测试代码替换到 /tests/create_mint.ts 文件中。替换后需要安装 @solana/spl-token。

```js
npm install @solana/spl-token
```
测试代码

```js
import * as anchor from "@coral-xyz/anchor";
import { Program } from "@coral-xyz/anchor";
import { CreateToken } from "../target/types/create_token";
import { TOKEN_2022_PROGRAM_ID, getMint } from "@solana/spl-token";
 
describe("token-example", () => {
  anchor.setProvider(anchor.AnchorProvider.env());
 
  const program = anchor.workspace.CreateToken as Program<CreateToken>;
  const mint = anchor.web3.Keypair.generate();
 
  it("Is initialized!", async () => {
    const tx = await program.methods
      .createMint()
      .accounts({
        mint: mint.publicKey,
        tokenProgram: TOKEN_2022_PROGRAM_ID,
      })
      .signers([mint])
      .rpc({ commitment: "confirmed" });
    console.log("Your transaction signature", tx);
 
    const mintAccount = await getMint(
      program.provider.connection,
      mint.publicKey,
      "confirmed",
      TOKEN_2022_PROGRAM_ID,
    );
 
    console.log("Mint Account", mintAccount);
  });
});
```
anchor中启动本地节点：
```js
//启动本地节点
solana-test-validator
```
另外开打一个终端再运行测试脚本：
```js
//运行测试脚本
anchor test --skip-local-validator
```
成功创建 `mint账户` ，地址为：DEA8573ueDk5aAo9NuomhMthRurvH3senGyQX9rx7zTT
![image.png](https://img.learnblockchain.cn/attachments/2025/05/chvcVo9E68326d588ad92.png)
我们可以在[solscan](https://solscan.io/)中连接本地节点，查看刚才的这笔交易和mint账户的详情。

![image.png](https://img.learnblockchain.cn/attachments/2025/05/s90TsKvi68326fe916b53.png)
可以看到代币的 `onwer` 是 Token 2022 Program，和以太坊合约中经常用到的onwer并不一样 , 此处代表普通钱包账户调用` Token 2022 Program` 可以修改此`mint账户`，但是也不可能所有普通钱包账户调用都可以修改，所以会有另外一个字段 `Authority` ，其中：
- Mint Authority：此账户可以铸造代币，此处被设置为交易部署账户
- Freeze Authority：此账户可以冻结代币账户，此处被设置为交易部署账户

#### 更新代币元数据 metadata（name、symbol等）
我们成功创建了一个代币，但是我们发现并没有设置过名称、符号、logo这些信息，那如何在solscan浏览器和一些钱包中配置呢？

solana代币的名称、符号、logo这些信息是保存在`元数据 metadata`中的，而`原始Token Program` 和 `Token Program 2022` 的保存方式也有一些不同。



**1. 原始 Token Program (SPL Token Program):**
* 原始的 Token Program 本身没有内置的功能来存储代币的名称、符号或 Logo。
* 为了给代币添加这些元数据，通常需要依赖 Metaplex Token Metadata 标准。
* 当你铸造一个使用原始 Token Program 的代币并为其创建元数据时，实际上是创建了一个单独的 Metaplex Metadata 账户，并将它与你的代币 Mint 账户关联起来。
* 更新名称和 Logo 的方式: 更新代币的名称或 Logo (或指向 Logo 的 URI) 需要调用 Metaplex 程序 的指令，而不是 Token Program 的指令。你需要持有该 Metaplex Metadata 账户的 Update Authority 权限才能进行更新。

我们写一个程序，在程序中创建原始代币并设置元数据，代码中的上下文结构 `CreateTokenMint` 新增传入一个 `metadata_account` PDA账户，并在cpi调用Metaplex程序的 create_metadata_accounts_v3方法时，通过上下文传值过去，同时把代币的token_name等元数据也传值过去。元数据是保存在 `metadata_account`账户中的。


```js
#![allow(clippy::result_large_err)]

use {
    anchor_lang::prelude::*,
    anchor_spl::{
        metadata::{
            create_metadata_accounts_v3, mpl_token_metadata::types::DataV2,
            CreateMetadataAccountsV3, Metadata,
        },
        token::{Mint, Token},
    },
};

declare_id!("GwvQ53QTu1xz3XXYfG5m5jEqwhMBvVBudPS8TUuFYnhT");

#[program]
pub mod create_token {
    use super::*;

    pub fn create_token_mint(
        ctx: Context<CreateTokenMint>,
        _token_decimals: u8,
        token_name: String,
        token_symbol: String,
        token_uri: String,
    ) -> Result<()> {
        msg!("Creating metadata account...");
        msg!(
            "Metadata account address: {}",
            &ctx.accounts.metadata_account.key()
        );

        // Cross Program Invocation (CPI)
        // Invoking the create_metadata_account_v3 instruction on the token metadata program
        create_metadata_accounts_v3(
            CpiContext::new(
                ctx.accounts.token_metadata_program.to_account_info(),
                CreateMetadataAccountsV3 {
                    metadata: ctx.accounts.metadata_account.to_account_info(),
                    mint: ctx.accounts.mint_account.to_account_info(),
                    mint_authority: ctx.accounts.payer.to_account_info(),
                    update_authority: ctx.accounts.payer.to_account_info(),
                    payer: ctx.accounts.payer.to_account_info(),
                    system_program: ctx.accounts.system_program.to_account_info(),
                    rent: ctx.accounts.rent.to_account_info(),
                },
            ),
            DataV2 {
                name: token_name,
                symbol: token_symbol,
                uri: token_uri,
                seller_fee_basis_points: 0,
                creators: None,
                collection: None,
                uses: None,
            },
            false, // Is mutable
            true,  // Update authority is signer
            None,  // Collection details
        )?;

        msg!("Token mint created successfully.");

        Ok(())
    }
}

#[derive(Accounts)]
#[instruction(_token_decimals: u8)]
pub struct CreateTokenMint<'info> {
    #[account(mut)]
    pub payer: Signer<'info>,

    /// CHECK: Validate address by deriving pda
    #[account(
        mut,
        seeds = [b"metadata", token_metadata_program.key().as_ref(), mint_account.key().as_ref()],
        bump,
        seeds::program = token_metadata_program.key(),
    )]
    pub metadata_account: UncheckedAccount<'info>,
    // Create new mint account
    #[account(
        init,
        payer = payer,
        mint::decimals = _token_decimals,
        mint::authority = payer.key(),
    )]
    pub mint_account: Account<'info, Mint>,

    pub token_metadata_program: Program<'info, Metadata>,
    pub token_program: Program<'info, Token>,
    pub system_program: Program<'info, System>,
    pub rent: Sysvar<'info, Rent>,
}
```

**2. Token Program 2022:**
* Token Program 2022 引入了许多扩展 (Extensions)。
* 其中一个重要的扩展是 Metadata Pointer 和 Metadata Extension。
* 通过启用这些扩展，你可以直接将代币的名称、符号、URI (通常指向包含 Logo 和其他详细信息的 JSON 文件) 等元数据存储在 代币的 Mint 账户本身 内，而不是一个独立的 Metaplex 账户。
* 更新名称和 Logo 的方式: 更新这些内置的元数据需要调用 Token Program 2022 程序 的特定指令，这些指令用于处理 Metadata Extension。你需要持有在 Mint 账户的 Metadata Extension 中定义的 Update Authority 权限才能进行更新。

接下来我们将之前创建mint 账户的程序修改一下，创建mint账户的同时，并设置metadata。


```js
use anchor_lang::prelude::*;
use anchor_lang::solana_program::rent::{
    DEFAULT_EXEMPTION_THRESHOLD, DEFAULT_LAMPORTS_PER_BYTE_YEAR,
};
use anchor_lang::system_program::{transfer, Transfer};
use anchor_spl::token_interface::{
    token_metadata_initialize, Mint, Token2022, TokenMetadataInitialize,
};
use spl_token_metadata_interface::state::TokenMetadata;
use spl_type_length_value::variable_len_pack::VariableLenPack;
 
declare_id!("2pNAGP3UHPrEFQ9q46DTNofA9xGesMjV231aTxCuTjw2");
 
#[program]
pub mod create_token {
    use super::*;
 
    pub fn create_mint(ctx: Context<CreateMint>, args: TokenMetadataArgs) -> Result<()> {
        let TokenMetadataArgs { name, symbol, uri } = args;

        // 定义 token metadata
        let token_metadata = TokenMetadata {
            name: name.clone(),
            symbol: symbol.clone(),
            uri: uri.clone(),
            ..Default::default()
        };

        // 计算mint账户额外保存元数据所需要的空间大小 
        //Token-2022 扩展标准要求前2 bytes为类型字段：标识这是一个元数据扩展，接着2 bytes为长度字段：帮助程序正确读取元数据，所以需要额外加上4bytes。
        let data_len = 4 + token_metadata.get_packed_len()?;

        // 计算豁免租金，存入2年以上的sol，可以豁免租金。
        let lamports = data_len as u64 * DEFAULT_LAMPORTS_PER_BYTE_YEAR * DEFAULT_EXEMPTION_THRESHOLD as u64;

        // 向 mint account 转账对应的sol
        transfer(
            CpiContext::new(
                ctx.accounts.system_program.to_account_info(),
                Transfer {
                    from: ctx.accounts.payer.to_account_info(),
                    to: ctx.accounts.mint_account.to_account_info(),
                },
            ),
            lamports,
        )?;

        // 设置代币的 metadata
        token_metadata_initialize(
            CpiContext::new(
                ctx.accounts.token_program.to_account_info(),
                TokenMetadataInitialize {
                    program_id: ctx.accounts.token_program.to_account_info(),
                    mint: ctx.accounts.mint_account.to_account_info(),
                    metadata: ctx.accounts.mint_account.to_account_info(),
                    mint_authority: ctx.accounts.payer.to_account_info(),
                    update_authority: ctx.accounts.payer.to_account_info(),
                },
            ),
            name,
            symbol,
            uri,
        )?;
        Ok(())
    }
}
 

#[derive(Accounts)]
pub struct CreateMint<'info> {
    #[account(mut)]
    pub payer: Signer<'info>,

    #[account(
        init,
        payer = payer,
        mint::decimals = 2,
        mint::authority = payer,
        //设置可以修改元数据的账户
        extensions::metadata_pointer::authority = payer,
        //指定元数据存储的账户，指向 mint account 自己
        extensions::metadata_pointer::metadata_address = mint_account,
    )]
    pub mint_account: InterfaceAccount<'info, Mint>,
    pub token_program: Program<'info, Token2022>,
    pub system_program: Program<'info, System>,
}


#[derive(AnchorDeserialize, AnchorSerialize)]
pub struct TokenMetadataArgs {
    pub name: String,
    pub symbol: String,
    pub uri: String,
}
```


### 铸造代币
我们现在创建了代币，但是总量还是0，现在需要铸造一定数量的代币。

> 和以太坊的代币不同，以太坊中一般都是在初始化方法中指定代币的总量，solana中并不存在部署合约后并初始化的方法，可以在部署合约后访问指定的方法来模拟初始化操作。

#### 什么是关联Token账户 ATA ？
另外一个和以太坊不同的地方是，solidity可以通过map(address=>uint256)结构，直接记录用户地址的余额。但是solana的账户模型设计是需要一个单独的代币账户来保存余额，相当于每个用户钱包在每个代币中都会有另外一个单独的账户来保存余额等信息，这个账户可以是任意创建的账户，但是为了方便查找等操作，solana中每个用户钱包在每个代币中会有一个默认且唯一的代币账户来保存相关信息，这个账户称为`关联代币账户` （`Associated Token Account 简称：ATA`）。

ATA是一个PDA账户，由`用户账户地址`、`代币mint地址`和`代币程序地址`组成种子，由`ATA 程序`生成。


```js
//ts前端测试文件中创建ATA账户
const ata = getAssociatedTokenAddress(
        mint.publicKey,//代币mint账户地址
        wallet.publicKey,//用户钱包地址
        false,//是否允许不在曲线上，一般来说用户钱包通常是false，pda账户则是true
        TOKEN_2022_PROGRAM_ID // 代币程序地址，这里我们使用Token Program 2022
);
```
现在我们在之前的程序中为默认钱包创建一个ATA账户，并mint一定数量的代币，使用此ATA账户来接收。


```js
use anchor_lang::prelude::*;
use anchor_spl::associated_token::AssociatedToken;
use anchor_lang::solana_program::rent::{
    DEFAULT_EXEMPTION_THRESHOLD, DEFAULT_LAMPORTS_PER_BYTE_YEAR,
};
use anchor_lang::system_program::{transfer, Transfer};
use anchor_spl::token_interface::{
    self,token_metadata_initialize, Mint,MintTo, Token2022, TokenMetadataInitialize,TokenInterface,TokenAccount,
};
use spl_token_metadata_interface::state::TokenMetadata;
use spl_type_length_value::variable_len_pack::VariableLenPack;
 
declare_id!("2pNAGP3UHPrEFQ9q46DTNofA9xGesMjV231aTxCuTjw2");
 
#[program]
pub mod create_token {
    use super::*;
 
    pub fn create_mint(ctx: Context<CreateMint>, args: TokenMetadataArgs) -> Result<()> {
        let TokenMetadataArgs { name, symbol, uri } = args;

        // Define token metadata
        let token_metadata = TokenMetadata {
            name: name.clone(),
            symbol: symbol.clone(),
            uri: uri.clone(),
            ..Default::default()
        };

        // Add 4 extra bytes for size of MetadataExtension (2 bytes for type, 2 bytes for length)
        let data_len = 4 + token_metadata.get_packed_len()?;

        // Calculate lamports required for the additional metadata
        let lamports =
            data_len as u64 * DEFAULT_LAMPORTS_PER_BYTE_YEAR * DEFAULT_EXEMPTION_THRESHOLD as u64;

        // Transfer additional lamports to mint account
        transfer(
            CpiContext::new(
                ctx.accounts.system_program.to_account_info(),
                Transfer {
                    from: ctx.accounts.payer.to_account_info(),
                    to: ctx.accounts.mint_account.to_account_info(),
                },
            ),
            lamports,
        )?;

        // Initialize token metadata
        token_metadata_initialize(
            CpiContext::new(
                ctx.accounts.token_program.to_account_info(),
                TokenMetadataInitialize {
                    program_id: ctx.accounts.token_program.to_account_info(),
                    mint: ctx.accounts.mint_account.to_account_info(),
                    metadata: ctx.accounts.mint_account.to_account_info(),
                    mint_authority: ctx.accounts.payer.to_account_info(),
                    update_authority: ctx.accounts.payer.to_account_info(),
                },
            ),
            name,
            symbol,
            uri,
        )?;
        Ok(())
    }

    //新增铸造方法
    pub fn mint_token(ctx: Context<MintToken>, amount: u64) -> Result<()> {
        let cpi_accounts = MintTo {
            mint: ctx.accounts.mint.to_account_info().clone(),
            to: ctx.accounts.signer_ata.to_account_info().clone(),
            authority: ctx.accounts.signer.to_account_info(),
        };
        let cpi_program = ctx.accounts.token_program.to_account_info();
        let cpi_context = CpiContext::new(cpi_program, cpi_accounts);
        token_interface::mint_to(cpi_context, amount)?;
        msg!("Mint Token");
        Ok(())
    }
}
 

#[derive(Accounts)]
pub struct CreateMint<'info> {
    #[account(mut)]
    pub payer: Signer<'info>,

    #[account(
        init,
        payer = payer,
        mint::decimals = 2,
        mint::authority = payer,
        extensions::metadata_pointer::authority = payer,
        extensions::metadata_pointer::metadata_address = mint_account,
    )]
    pub mint_account: InterfaceAccount<'info, Mint>,
    pub token_program: Program<'info, Token2022>,
    pub system_program: Program<'info, System>,
}


#[derive(AnchorDeserialize, AnchorSerialize)]
pub struct TokenMetadataArgs {
    pub name: String,
    pub symbol: String,
    pub uri: String,
}

 //新增铸造方法上下文结构体
#[derive(Accounts)]
pub struct MintToken<'info> {
    #[account(mut)]
    pub signer: Signer<'info>,
    #[account(mut)]
    pub mint: InterfaceAccount<'info, Mint>,
    #[account(
        init,
        payer = signer,
        associated_token::mint = mint,
        associated_token::authority = signer,
    )]
    pub signer_ata: InterfaceAccount<'info, TokenAccount>,
    pub token_program: Interface<'info, TokenInterface>,

    pub system_program: Program<'info, System>,
    pub associated_token_program: Program<'info, AssociatedToken>,
}
```
在新增的铸造方法的上下文结构体中我们用 账户约束 定义一个ATA账户，由这个账户来接收铸造出来的代币。我们需要通过CPI调用代币程序的 `mint_to` 方法来完成铸造。

测试代码：

```js
import * as anchor from "@coral-xyz/anchor";
import { Program } from "@coral-xyz/anchor";
import { CreateToken } from "../target/types/create_token";
import { 
  TOKEN_2022_PROGRAM_ID, 
  ASSOCIATED_TOKEN_PROGRAM_ID,
  getAssociatedTokenAddress 
} from "@solana/spl-token";
import { 
  PublicKey, 
  Keypair, 
  SystemProgram, 
  LAMPORTS_PER_SOL 
} from "@solana/web3.js";
 
describe("token-example", () => {
  const provider = anchor.AnchorProvider.env();
  anchor.setProvider(provider);
 
  const program = anchor.workspace.CreateToken as Program<CreateToken>;
  const mint = Keypair.generate();

  const wallet = provider.wallet as anchor.Wallet;
  
  
  const metadata = {
    name: 'my token',
    symbol: 'mt',
    uri: 'https://raw.githubusercontent.com/solana-developers/opos-asset/main/assets/DeveloperPortal/metadata.json',
  };
 
  it("Is initialized!", async () => {
    const tx = await program.methods
      .createMint(metadata)
      .accounts({
        payer: provider.wallet.publicKey,
        mintAccount: mint.publicKey,
        tokenProgram: TOKEN_2022_PROGRAM_ID,
        systemProgram: SystemProgram.programId,
      })
      .signers([mint])
      .rpc({ commitment: "confirmed" });
    
    console.log("Your transaction signature", tx);
    console.log("mintAccount", mint.publicKey);
  });

  it("Mint Token!", async () => {
      const ata = await getAssociatedTokenAddress(
        mint.publicKey,
        wallet.publicKey,
        false,
        TOKEN_2022_PROGRAM_ID
      );

      const amount = new anchor.BN(50000000);

      const tx = await program.methods
      .mintToken(amount)
      .accounts({
        mint: mint.publicKey,
        signerAta: ata,
        tokenProgram: TOKEN_2022_PROGRAM_ID,
      })
      .rpc();

    console.log("Mint Token! Transaction signature:", tx);
  });

  
});
```
运行后可以看到我们成功创建了 `signerAta` ATA账户，并铸造了50万代币给到这个账户。

### 转账代币

程序中进行代币转账，可以使用anchor_spl中封装好的 `transfer_checked`方法，也是发起一笔cpi调用。
我们将在之前的程序中增加 `transfer_token` 方法来进行代币转账，对应的示例程序如下：

在 `transfer_token` 方法的上下文结构体中，需要传入转出代币的账户（此处使用签名者账号）和对应的ATA账户，还有接收代币的账户和对应的ATA账户(此处接收账号还未创建ATA账号，所以需要创建)

```js
use anchor_lang::prelude::*;
use anchor_spl::associated_token::AssociatedToken;
use anchor_lang::solana_program::rent::{
    DEFAULT_EXEMPTION_THRESHOLD, DEFAULT_LAMPORTS_PER_BYTE_YEAR,
};
use anchor_lang::system_program::{transfer, Transfer};
use anchor_spl::token_interface::{
    self,token_metadata_initialize, Mint,MintTo, Token2022, TokenMetadataInitialize,TokenInterface,TokenAccount,TransferChecked,
};
use spl_token_metadata_interface::state::TokenMetadata;
use spl_type_length_value::variable_len_pack::VariableLenPack;
 
declare_id!("2pNAGP3UHPrEFQ9q46DTNofA9xGesMjV231aTxCuTjw2");
 
#[program]
pub mod create_token {
    use super::*;
 
    pub fn create_mint(ctx: Context<CreateMint>, args: TokenMetadataArgs) -> Result<()> {
        let TokenMetadataArgs { name, symbol, uri } = args;

        // Define token metadata
        let token_metadata = TokenMetadata {
            name: name.clone(),
            symbol: symbol.clone(),
            uri: uri.clone(),
            ..Default::default()
        };

        // Add 4 extra bytes for size of MetadataExtension (2 bytes for type, 2 bytes for length)
        let data_len = 4 + token_metadata.get_packed_len()?;

        // Calculate lamports required for the additional metadata
        let lamports =
            data_len as u64 * DEFAULT_LAMPORTS_PER_BYTE_YEAR * DEFAULT_EXEMPTION_THRESHOLD as u64;

        // Transfer additional lamports to mint account
        transfer(
            CpiContext::new(
                ctx.accounts.system_program.to_account_info(),
                Transfer {
                    from: ctx.accounts.payer.to_account_info(),
                    to: ctx.accounts.mint_account.to_account_info(),
                },
            ),
            lamports,
        )?;

        // Initialize token metadata
        token_metadata_initialize(
            CpiContext::new(
                ctx.accounts.token_program.to_account_info(),
                TokenMetadataInitialize {
                    program_id: ctx.accounts.token_program.to_account_info(),
                    mint: ctx.accounts.mint_account.to_account_info(),
                    metadata: ctx.accounts.mint_account.to_account_info(),
                    mint_authority: ctx.accounts.payer.to_account_info(),
                    update_authority: ctx.accounts.payer.to_account_info(),
                },
            ),
            name,
            symbol,
            uri,
        )?;
        Ok(())
    }


    pub fn mint_token(ctx: Context<MintToken>, amount: u64) -> Result<()> {
        let cpi_accounts = MintTo {
            mint: ctx.accounts.mint.to_account_info().clone(),
            to: ctx.accounts.signer_ata.to_account_info().clone(),
            authority: ctx.accounts.signer.to_account_info(),
        };
        let cpi_program = ctx.accounts.token_program.to_account_info();
        let cpi_context = CpiContext::new(cpi_program, cpi_accounts);
        token_interface::mint_to(cpi_context, amount)?;
        msg!("Mint Token");
        Ok(())
    }

    pub fn transfer_token(ctx: Context<TransferToken>, amount: u64) -> Result<()> {
        let cpi_accounts = TransferChecked {
            from: ctx.accounts.from.to_account_info().clone(),
            mint: ctx.accounts.mint.to_account_info().clone(),
            to: ctx.accounts.to_ata.to_account_info().clone(),
            authority: ctx.accounts.signer.to_account_info(),
        };
        let cpi_program = ctx.accounts.token_program.to_account_info();
        let cpi_context = CpiContext::new(cpi_program, cpi_accounts);
        token_interface::transfer_checked(cpi_context, amount, ctx.accounts.mint.decimals)?;
        msg!("Transfer Token");
        Ok(())
    }
}
 

#[derive(Accounts)]
pub struct CreateMint<'info> {
    #[account(mut)]
    pub payer: Signer<'info>,

    #[account(
        init,
        payer = payer,
        mint::decimals = 2,
        mint::authority = payer,
        extensions::metadata_pointer::authority = payer,
        extensions::metadata_pointer::metadata_address = mint_account,
    )]
    pub mint_account: InterfaceAccount<'info, Mint>,
    pub token_program: Program<'info, Token2022>,
    pub system_program: Program<'info, System>,
}


#[derive(AnchorDeserialize, AnchorSerialize)]
pub struct TokenMetadataArgs {
    pub name: String,
    pub symbol: String,
    pub uri: String,
}


#[derive(Accounts)]
pub struct MintToken<'info> {
    #[account(mut)]
    pub signer: Signer<'info>,
    #[account(mut)]
    pub mint: InterfaceAccount<'info, Mint>,
    #[account(
        init,
        payer = signer,
        associated_token::mint = mint,
        associated_token::authority = signer,
    )]
    pub signer_ata: InterfaceAccount<'info, TokenAccount>,
    pub token_program: Interface<'info, TokenInterface>,

    pub system_program: Program<'info, System>,
    pub associated_token_program: Program<'info, AssociatedToken>,
}

#[derive(Accounts)]
pub struct TransferToken<'info> {
    #[account(mut)]
    pub signer: Signer<'info>,
    #[account(mut)]
    pub from: InterfaceAccount<'info, TokenAccount>,
    pub to: SystemAccount<'info>,
    #[account(
        init,
        associated_token::mint = mint,
        payer = signer,
        associated_token::authority = to
    )]
    pub to_ata: InterfaceAccount<'info, TokenAccount>,
    #[account(mut)]
    pub mint: InterfaceAccount<'info, Mint>,
    pub token_program: Interface<'info, TokenInterface>,
    pub system_program: Program<'info, System>,
    pub associated_token_program: Program<'info, AssociatedToken>,
}
```

测试文件代码：


```js
import * as anchor from "@coral-xyz/anchor";
import { Program } from "@coral-xyz/anchor";
import { CreateToken } from "../target/types/create_token";
import { 
  TOKEN_2022_PROGRAM_ID, 
  ASSOCIATED_TOKEN_PROGRAM_ID,
  getAssociatedTokenAddress, 
  getAssociatedTokenAddressSync
} from "@solana/spl-token";
import { 
  PublicKey, 
  Keypair, 
  SystemProgram, 
  LAMPORTS_PER_SOL 
} from "@solana/web3.js";
 
describe("token-example", () => {
  const provider = anchor.AnchorProvider.env();
  anchor.setProvider(provider);
 
  const program = anchor.workspace.CreateToken as Program<CreateToken>;
  

  const wallet = provider.wallet as anchor.Wallet;
  
  const mint = Keypair.generate();
  const ata = getAssociatedTokenAddressSync(
        mint.publicKey,
        wallet.publicKey,
        false,
        TOKEN_2022_PROGRAM_ID
  );
  const recipient = Keypair.generate();
  const recipient_ata = getAssociatedTokenAddressSync(
        mint.publicKey,
        recipient.publicKey,
        false,
        TOKEN_2022_PROGRAM_ID
  );
  
  const metadata = {
    name: 'my token',
    symbol: 'mt',
    uri: 'https://raw.githubusercontent.com/solana-developers/opos-asset/main/assets/DeveloperPortal/metadata.json',
  };
 
  it("Is initialized!", async () => {
    const tx = await program.methods
      .createMint(metadata)
      .accounts({
        payer: provider.wallet.publicKey,
        mintAccount: mint.publicKey,
        tokenProgram: TOKEN_2022_PROGRAM_ID,
        systemProgram: SystemProgram.programId,
      })
      .signers([mint])
      .rpc({ commitment: "confirmed" });
    
    console.log("Your transaction signature", tx);
    console.log("mintAccount", mint.publicKey);
  });

  it("Mint Token!", async () => {
      

      const amount = new anchor.BN(50000000);

      const tx = await program.methods
      .mintToken(amount)
      .accounts({
        mint: mint.publicKey,
        signerAta: ata,
        tokenProgram: TOKEN_2022_PROGRAM_ID,
      })
      .rpc();

      

    console.log("Mint Token! Transaction signature:", tx);
  });


  it("transfer Token!", async () => {
      

      const amount = new anchor.BN(10000000);

      const tx = await program.methods
      .transferToken(amount)
      .accounts({
        from: ata,
        to: recipient.publicKey,
        toAta:recipient_ata,
        mint: mint.publicKey,
        tokenProgram: TOKEN_2022_PROGRAM_ID,
      })
      .rpc();

      

    console.log("transfer Token! Transaction signature:", tx);
  });

  
});
```

我们新创建一个 recipient 账号，并计算出对应的ATA账户。运行测试代码，可以看到成功转移10万代币。

```js
//构建
anchor build
//运行测试代码
anchor test
```

### 总结

本节我们在anchor程序中创建代币、铸造代币、转账代币，并了解了什么是 Mint账户以及元数据，还有什么是ATA账户。



