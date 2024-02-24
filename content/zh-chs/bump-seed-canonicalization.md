---
title: Bump 种子规范化
objectives:
- 解释使用非规范化的 bump 派生地址（PDAs）相关的漏洞
- 使用 Anchor 的 `seeds` 和 `bump` 约束初始化 PDA，自动使用规范化的 bump
- 使用 Anchor 的 `seeds` 和 `bump` 约束确保在未来指令中派生 PDA 时始终使用规范化的 bump
---

# 总结

- [**`create_program_address`**](https://docs.rs/solana-program/latest/solana_program/pubkey/struct.Pubkey.html#method.create_program_address) 函数派生 PDA 时不会搜索 **规范化的 bump**。这意味着存在多个有效的 bumps，每个都会产生不同的地址。
- 使用 [**`find_program_address`**](https://docs.rs/solana-program/latest/solana_program/pubkey/struct.Pubkey.html#method.find_program_address) 确保使用最高的有效 bump，或者称为规范化的 bump，进行派生，从而创建了一种确定性的方法，根据特定的种子查找地址。
- 在初始化时，你可以使用 Anchor 的 `seeds` 和 `bump` 约束，以确保账户验证结构中的 PDA 派生始终使用规范化的 bump。
- Anchor 允许你在验证 PDA 地址时使用 `bump = <some_bump>` 约束来 **指定 bump**。
- 由于 `find_program_address` 可能成本较高，最佳实践是将派生的 bump 存储在账户数据字段中，以便在以后重新派生地址以进行验证时进行引用。

    ```rust
    #[derive(Accounts)]
    pub struct VerifyAddress<'info> {
    	#[account(
        	seeds = [DATA_PDA_SEED.as_bytes()],
    	    bump = data.bump
    	)]
    	data: Account<'info, Data>,
    }
    ```

# 概述

Bumps 种子是一个介于 0 到 255 之间（包括边界值）的数字，用于确保使用 [`create_program_address`](https://docs.rs/solana-program/latest/solana_program/pubkey/struct.Pubkey.html#method.create_program_address) 派生的地址是一个有效的 PDA。**规范化的 bump（canonical bump）** 是产生有效 PDA 的最高 bump 值。在 Solana 中的标准做法是 *始终使用规范化的 bump* 来派生 PDAs，这既是为了安全性也是为了方便性。

## 使用 `create_program_address` 进行不安全的 PDA 派生

给定一组种子，`create_program_address` 函数将大约有 50% 的几率生成一个有效的 PDA。bump 种子是额外添加的字节，作为一种“bump”派生地址到有效领域的种子。由于存在 256 个可能的 bump 种子，并且该函数大约 50% 的时间生成有效的 PDA，对于给定的输入种子集，有许多有效的 bumps。

您可以想象，当使用种子作为在已知信息和账户之间进行映射的方式时，这可能会导致在定位账户时产生混淆。使用规范的 bump 作为标准可以确保您始终能够找到正确的账户。更重要的是，它避免了由于允许多个 bumps 而导致的安全漏洞。

在下面的示例中，`set_value` 指令使用作为指令数据传递的 `bump` 来派生一个 PDA。然后，该指令使用 `create_program_address` 函数派生 PDA，并检查 `address` 是否与 `data` 账户的公钥匹配。

```rust
use anchor_lang::prelude::*;

declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");

#[program]
pub mod bump_seed_canonicalization_insecure {
    use super::*;

    pub fn set_value(ctx: Context<BumpSeed>, key: u64, new_value: u64, bump: u8) -> Result<()> {
        let address =
            Pubkey::create_program_address(&[key.to_le_bytes().as_ref(), &[bump]], ctx.program_id).unwrap();
        if address != ctx.accounts.data.key() {
            return Err(ProgramError::InvalidArgument.into());
        }

        ctx.accounts.data.value = new_value;

        Ok(())
    }
}

#[derive(Accounts)]
pub struct BumpSeed<'info> {
    data: Account<'info, Data>,
}

#[account]
pub struct Data {
    value: u64,
}
```

在指令派生 PDA 并检查传入的账户的同时，这是好的，但它允许调用者传入任意的 bump。根据程序的上下文，这可能导致不希望的行为或潜在的利用。

例如，如果种子映射旨在强制执行 PDA 与用户之间的一对一关系，那么该程序将无法正确执行。用户可以多次调用程序，并传入许多有效的 bump，每次产生不同的 PDA。

## 推荐使用 `find_program_address` 进行派生

解决这个问题的一个简单方法是让程序只接受规范的 bump，并使用 `find_program_address` 来派生 PDA。

[`find_program_address`](https://docs.rs/solana-program/latest/solana_program/pubkey/struct.Pubkey.html#method.find_program_address) *始终使用规范的 bump*。该函数通过调用 `create_program_address` 进行迭代，从 bump 为 255 开始，并在每次迭代中递减 bump。一旦找到有效地址，函数就会返回派生的 PDA 和用于派生它的规范 bump。

这确保了输入种子与它们产生的地址之间的一对一映射。

```rust
pub fn set_value_secure(
    ctx: Context<BumpSeed>,
    key: u64,
    new_value: u64,
    bump: u8,
) -> Result<()> {
    let (address, expected_bump) =
        Pubkey::find_program_address(&[key.to_le_bytes().as_ref()], ctx.program_id);

    if address != ctx.accounts.data.key() {
        return Err(ProgramError::InvalidArgument.into());
    }
    if expected_bump != bump {
        return Err(ProgramError::InvalidArgument.into());
    }

    ctx.accounts.data.value = new_value;
    Ok(())
}
```

## 使用 Anchor 的 `seeds` 和 `bump` 约束

Anchor 提供了一种方便的方式，在账户验证结构中使用 `seeds` 和 `bump` 约束来派生 PDAs。这些甚至可以与 `init` 约束结合使用，以在预期地址上初始化账户。为了保护程序免受我们在本课程中讨论的漏洞的影响，Anchor 甚至不允许您使用除了规范 bump 以外的任何方式来初始化 PDA 上的账户。相反，它使用 `find_program_address` 来派生 PDA，并随后执行初始化。

```rust
use anchor_lang::prelude::*;

declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");

#[program]
pub mod bump_seed_canonicalization_recommended {
    use super::*;

    pub fn set_value(ctx: Context<BumpSeed>, _key: u64, new_value: u64) -> Result<()> {
        ctx.accounts.data.value = new_value;
        Ok(())
    }
}

// initialize account at PDA
#[derive(Accounts)]
#[instruction(key: u64)]
pub struct BumpSeed<'info> {
  #[account(mut)]
  payer: Signer<'info>,
  #[account(
    init,
    seeds = [key.to_le_bytes().as_ref()],
    // derives the PDA using the canonical bump
    bump,
    payer = payer,
    space = 8 + 8
  )]
  data: Account<'info, Data>,
  system_program: Program<'info, System>
}

#[account]
pub struct Data {
    value: u64,
}
```

如果您不初始化账户，仍然可以使用 `seeds` 和 `bump` 约束来验证 PDAs。这只是重新派生 PDA 并将派生的地址与传入的账户的地址进行比较。

在这种情况下，Anchor *允许* 您指定要用于派生 PDA 的 bump，形式为 `bump = <some_bump>`。这里的意图不是让您使用任意的 bump，而是让您优化您的程序。`find_program_address` 的迭代性质使其成本高昂，因此最佳实践是在初始化 PDA 时将规范 bump 存储在 PDA 账户的数据中，以便您在后续指令中验证 PDA 时引用存储的 bump。

当您指定要使用的 bump 时，Anchor 使用提供的 bump 而不是 `find_program_address`。在账户数据中存储 bump 的这种模式确保您的程序始终使用规范 bump 而不会降低性能。

```rust
use anchor_lang::prelude::*;

declare_id!("CVwV9RoebTbmzsGg1uqU1s4a3LvTKseewZKmaNLSxTqc");

#[program]
pub mod bump_seed_canonicalization_recommended {
    use super::*;

    pub fn set_value(ctx: Context<BumpSeed>, _key: u64, new_value: u64) -> Result<()> {
        ctx.accounts.data.value = new_value;
        // store the bump on the account
        ctx.accounts.data.bump = *ctx.bumps.get("data").unwrap();
        Ok(())
    }

    pub fn verify_address(ctx: Context<VerifyAddress>, _key: u64) -> Result<()> {
        msg!("PDA confirmed to be derived with canonical bump: {}", ctx.accounts.data.key());
        Ok(())
    }
}

// initialize account at PDA
#[derive(Accounts)]
#[instruction(key: u64)]
pub struct BumpSeed<'info> {
  #[account(mut)]
  payer: Signer<'info>,
  #[account(
    init,
    seeds = [key.to_le_bytes().as_ref()],
    // derives the PDA using the canonical bump
    bump,
    payer = payer,
    space = 8 + 8 + 1
  )]
  data: Account<'info, Data>,
  system_program: Program<'info, System>
}

#[derive(Accounts)]
#[instruction(key: u64)]
pub struct VerifyAddress<'info> {
  #[account(
    seeds = [key.to_le_bytes().as_ref()],
    // guranteed to be the canonical bump every time
    bump = data.bump
  )]
  data: Account<'info, Data>,
}

#[account]
pub struct Data {
    value: u64,
    // bump field
    bump: u8
}
```

如果您在 `bump` 约束上未指定 bump，Anchor 将仍然使用 `find_program_address` 使用规范 bump 来派生 PDA。因此，您的指令将产生不确定数量的计算预算。已经有可能超出计算预算的程序应该谨慎使用这个功能，因为程序的预算可能偶尔和不可预测地会超出。

另一方面，如果您只需要验证传入的 PDA 的地址而不初始化账户，您将被迫让 Anchor 派生规范 bump 或者使您的程序面临不必要的风险。在这种情况下，请尽管性能稍微下降，还是使用规范 bump。

# 实验

为了演示当您不检查规范 bump 时可能发生的安全漏洞，让我们使用一个允许每个程序用户及时“领取”奖励的程序来进行工作。

## 1. 设置

首先获取 [此存储库](https://github.com/Unboxed-Software/solana-bump-seed-canonicalization/tree/starter) 上 `starter` 分支上的代码。

请注意，程序中有两个指令以及 `tests` 目录中的单个测试。

程序中的指令包括：

1. `create_user_insecure`
2. `claim_insecure`

`create_user_insecure` 指令简单地在使用签名者的公钥和传入的 bump 派生的 PDA 上创建一个新账户。

`claim_insecure` 指令向用户铸造 10 个代币，然后标记账户的奖励为已领取，以防止他们再次领取。

然而，该程序并未明确检查所涉及的 PDA 是否使用了规范 bump。

在继续之前，请查看程序以了解其功能。

## 2. 测试不安全的指令

由于指令并未明确要求 `user` PDA 使用规范 bump，因此攻击者可以在每个钱包中创建多个账户，并领取超出应允许的奖励。

`tests` 目录中的测试创建一个名为 `attacker` 的新密钥对来代表攻击者。然后，它循环遍历所有可能的 bumps 并调用 `create_user_insecure` 和 `claim_insecure`。最后，测试期望攻击者能够多次领取奖励，并且获得超过每个用户分配的 10 个代币。

```typescript
it("Attacker can claim more than reward limit with insecure instructions", async () => {
    const attacker = Keypair.generate()
    await safeAirdrop(attacker.publicKey, provider.connection)
    const ataKey = await getAssociatedTokenAddress(mint, attacker.publicKey)

    let numClaims = 0

    for (let i = 0; i < 256; i++) {
      try {
        const pda = createProgramAddressSync(
          [attacker.publicKey.toBuffer(), Buffer.from([i])],
          program.programId
        )
        await program.methods
          .createUserInsecure(i)
          .accounts({
            user: pda,
            payer: attacker.publicKey,
          })
          .signers([attacker])
          .rpc()
        await program.methods
          .claimInsecure(i)
          .accounts({
            user: pda,
            mint,
            payer: attacker.publicKey,
            userAta: ataKey,
          })
          .signers([attacker])
          .rpc()

        numClaims += 1
      } catch (error) {
        if (
          error.message !== "Invalid seeds, address must fall off the curve"
        ) {
          console.log(error)
        }
      }
    }

    const ata = await getAccount(provider.connection, ataKey)

    console.log(
      `Attacker claimed ${numClaims} times and got ${Number(ata.amount)} tokens`
    )

    expect(numClaims).to.be.greaterThan(1)
    expect(Number(ata.amount)).to.be.greaterThan(10)
})
```

运行 `anchor test` 来查看此测试是否通过，显示攻击者成功。由于测试对每个有效的 bump 调用指令，因此运行时间可能较长，请耐心等待。

```bash
  bump-seed-canonicalization
Attacker claimed 129 times and got 1290 tokens
    ✔ Attacker can claim more than reward limit with insecure instructions (133840ms)
```

## 3. 创建安全指令

让我们通过创建两个新指令来演示修补漏洞：

1. `create_user_secure`
2. `claim_secure`

在编写账户验证或指令逻辑之前，让我们创建一个新的用户类型 `UserSecure`。这种新类型将规范 bump 添加为结构体的一个字段。

```rust
#[account]
pub struct UserSecure {
    auth: Pubkey,
    bump: u8,
    rewards_claimed: bool,
}
```

接下来，让我们为每个新指令创建账户验证结构。它们将与不安全版本非常相似，但会让 Anchor 处理 PDA 的派生和反序列化。

```rust
#[derive(Accounts)]
pub struct CreateUserSecure<'info> {
    #[account(mut)]
    payer: Signer<'info>,
    #[account(
        init,
        seeds = [payer.key().as_ref()],
        // derives the PDA using the canonical bump
        bump,
        payer = payer,
        space = 8 + 32 + 1 + 1
    )]
    user: Account<'info, UserSecure>,
    system_program: Program<'info, System>,
}

#[derive(Accounts)]
pub struct SecureClaim<'info> {
    #[account(
        seeds = [payer.key().as_ref()],
        bump = user.bump,
        constraint = !user.rewards_claimed @ ClaimError::AlreadyClaimed,
        constraint = user.auth == payer.key()
    )]
    user: Account<'info, UserSecure>,
    #[account(mut)]
    payer: Signer<'info>,
    #[account(
        init_if_needed,
        payer = payer,
        associated_token::mint = mint,
        associated_token::authority = payer
    )]
    user_ata: Account<'info, TokenAccount>,
    #[account(mut)]
    mint: Account<'info, Mint>,
    /// CHECK: mint auth PDA
    #[account(seeds = ["mint".as_bytes().as_ref()], bump)]
    pub mint_authority: UncheckedAccount<'info>,
    token_program: Program<'info, Token>,
    associated_token_program: Program<'info, AssociatedToken>,
    system_program: Program<'info, System>,
    rent: Sysvar<'info, Rent>,
}
```

最后，让我们为这两个新指令实现指令逻辑。`create_user_secure` 指令只需要设置 `user` 账户数据上的 `auth`、`bump` 和 `rewards_claimed` 字段即可。

```rust
pub fn create_user_secure(ctx: Context<CreateUserSecure>) -> Result<()> {
    ctx.accounts.user.auth = ctx.accounts.payer.key();
    ctx.accounts.user.bump = *ctx.bumps.get("user").unwrap();
    ctx.accounts.user.rewards_claimed = false;
    Ok(())
}
```

`claim_secure` 指令需要向用户铸造 10 个代币，并将 `user` 账户的 `rewards_claimed` 字段设置为 `true`。

```rust
pub fn claim_secure(ctx: Context<SecureClaim>) -> Result<()> {
    token::mint_to(
        CpiContext::new_with_signer(
            ctx.accounts.token_program.to_account_info(),
            MintTo {
                mint: ctx.accounts.mint.to_account_info(),
                to: ctx.accounts.user_ata.to_account_info(),
                authority: ctx.accounts.mint_authority.to_account_info(),
            },
            &[&[
                    b"mint".as_ref(),
                &[*ctx.bumps.get("mint_authority").unwrap()],
            ]],
        ),
        10,
    )?;

    ctx.accounts.user.rewards_claimed = true;

    Ok(())
}
```

## 4. 测试安全指令

让我们继续编写一个测试来展示攻击者无法再使用新指令多次领取奖励。

请注意，如果您像旧测试一样循环使用多个 PDAs，甚至无法将非规范 bump 传递给指令。但是，您仍然可以循环遍历各种 PDAs，并在最后检查只发生了一次领取，总计为 10 个代币。您的最终测试将如下所示：

```typescript
it.only("Attacker can only claim once with secure instructions", async () => {
    const attacker = Keypair.generate()
    await safeAirdrop(attacker.publicKey, provider.connection)
    const ataKey = await getAssociatedTokenAddress(mint, attacker.publicKey)
    const [userPDA] = findProgramAddressSync(
      [attacker.publicKey.toBuffer()],
      program.programId
    )

    await program.methods
      .createUserSecure()
      .accounts({
        payer: attacker.publicKey,
      })
      .signers([attacker])
      .rpc()

    await program.methods
      .claimSecure()
      .accounts({
        payer: attacker.publicKey,
        userAta: ataKey,
        mint,
        user: userPDA,
      })
      .signers([attacker])
      .rpc()

    let numClaims = 1

    for (let i = 0; i < 256; i++) {
      try {
        const pda = createProgramAddressSync(
          [attacker.publicKey.toBuffer(), Buffer.from([i])],
          program.programId
        )
        await program.methods
          .createUserSecure()
          .accounts({
            user: pda,
            payer: attacker.publicKey,
          })
          .signers([attacker])
          .rpc()

        await program.methods
          .claimSecure()
          .accounts({
            payer: attacker.publicKey,
            userAta: ataKey,
            mint,
            user: pda,
          })
          .signers([attacker])
          .rpc()

        numClaims += 1
      } catch {}
    }

    const ata = await getAccount(provider.connection, ataKey)

    expect(Number(ata.amount)).to.equal(10)
    expect(numClaims).to.equal(1)
})
```

```bash
  bump-seed-canonicalization
Attacker claimed 119 times and got 1190 tokens
    ✔ Attacker can claim more than reward limit with insecure instructions (128493ms)
    ✔ Attacker can only claim once with secure instructions (1448ms)
```

如果您在所有的 PDA 派生中都使用 Anchor，那么避免这种特定的漏洞就非常简单。然而，如果您最终做了任何“非标准”的操作，请小心设计您的程序以明确使用规范 bump！

如果您想查看最终的解决方案代码，可以在 [相同的存储库](https://github.com/Unboxed-Software/solana-bump-seed-canonicalization/tree/solution) 的 `solution` 分支上找到它。

# 挑战

与本单元的其他课程一样，避免这种安全漏洞的机会在于审查您自己或其他程序。

花些时间审查至少一个程序，并确保所有的 PDA 派生和检查都使用规范 bump。

请记住，如果您在别人的程序中发现了错误或漏洞，请立即通知他们！如果您在自己的程序中发现了错误或漏洞，请务必立即修复。

## 完成实验了吗？

将您的代码推送到 GitHub，并[告诉我们您对本课程的看法](https://form.typeform.com/to/IPH0UGz7#answers-lesson=d3f6ca7a-11c8-421f-b7a3-d6c08ef1aa8b)！