---
title: 关闭账户和复活攻击
objectives:
- 解释与错误关闭程序账户相关的各种安全漏洞
- 使用本地 Rust 安全地关闭程序账户
- 使用 Anchor 的 `close` 约束安全地关闭程序账户
---

# TL;DR

- **关闭账户** 不当操作会为重新初始化/复活攻击创造机会。
- 当账户不再免租时，Solana runtime 会**垃圾收集账户**。关闭账户涉及将存储在账户中以免租形式的 lamports 转移到您选择的另一个账户中。
- 您可以使用 Anchor 的 `#[account(close = <address_to_send_lamports>)]` 约束来安全地关闭账户，并将账户鉴别器设置为 `CLOSED_ACCOUNT_DISCRIMINATOR`。

```rust
#[account(mut, close = receiver)]
pub data_account: Account<'info, MyData>,
#[account(mut)]
pub receiver: SystemAccount<'info>
```

# 概述

虽然听起来很简单，但正确关闭账户可能会有些棘手。如果您不按照特定步骤操作，攻击者可能会绕过关闭账户的过程。

为了更好地了解这些攻击向量，让我们深入探讨每种情况。

## 不安全的账户关闭

在本质上，关闭一个账户涉及将其 lamports 转移到一个单独的账户，从而触发 Solana runtime 对第一个账户进行垃圾收集。这将把账户的 `owner` 字段从所属的合约重置为归属 `system program`。

请看下面的示例。这个指令需要两个账户：

1. `account_to_close` - 要关闭的账户
2. `destination` - 应该接收关闭账户 lamports 的账户

这个程序逻辑旨在通过简单地增加 `destination` 账户的 lamports，数量为存储在 `account_to_close` 中的金额，从而关闭一个账户，并将 `account_to_close` 的 lamports 设置为 0。使用这个程序，在一个完整的交易被处理后，`account_to_close` 将被运行时垃圾收集。

```rust
use anchor_lang::prelude::*;

declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");

#[program]
pub mod closing_accounts_insecure {
    use super::*;

    pub fn close(ctx: Context<Close>) -> ProgramResult {
        let dest_starting_lamports = ctx.accounts.destination.lamports();

        **ctx.accounts.destination.lamports.borrow_mut() = dest_starting_lamports
            .checked_add(ctx.accounts.account_to_close.to_account_info().lamports())
            .unwrap();

        **ctx
            .accounts
            .account_to_close
            .to_account_info()
            .lamports
            .borrow_mut() = 0;

        Ok(())
    }
}

#[derive(Accounts)]
pub struct Close<'info> {
    account_to_close: Account<'info, Data>,
    destination: AccountInfo<'info>,
}

#[account]
pub struct Data {
    data: u64,
}
```

然而，垃圾收集直到交易完成才会发生。由于一个交易中可以有多个指令，这为攻击者提供了一个机会，可以调用关闭账户的指令，但也在交易中包含一个转账以退还账户的租金豁免 lamports。结果是账户*将不会*被垃圾收集，为攻击者在程序中引发意外行为甚至耗尽协议提供了一条途径。

## 安全的账户关闭

关闭这个漏洞的两个最重要的步骤是将账户数据清零，并添加一个表示账户已关闭的账户鉴别器（account discriminator）。你需要*同时*做这两件事情来避免意外的程序行为。

一个数据被清零的账户仍然可以用于某些情况，特别是如果它是一个 PDA，其派生地址在程序中用于验证目的。然而，如果攻击者无法访问先前存储的数据，损害可能会受到潜在的限制。

为了进一步保护程序，关闭的账户应该被赋予一个账户鉴别器，将其标记为“closed”，并且所有指令应该对所有传入的账户进行检查，如果账户标记为已关闭，则返回错误。

看下面的示例。这个程序通过单个指令将 lamports 转移出一个账户，清零账户数据，并设置一个账户鉴别器，希望在垃圾收集之前阻止后续的指令再次使用这个账户。如果没有做到这些事情中的任何一项，都将导致安全漏洞。

```rust
use anchor_lang::prelude::*;
use std::io::Write;
use std::ops::DerefMut;

declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");

#[program]
pub mod closing_accounts_insecure_still_still {
    use super::*;

    pub fn close(ctx: Context<Close>) -> ProgramResult {
        let account = ctx.accounts.account.to_account_info();

        let dest_starting_lamports = ctx.accounts.destination.lamports();

        **ctx.accounts.destination.lamports.borrow_mut() = dest_starting_lamports
            .checked_add(account.lamports())
            .unwrap();
        **account.lamports.borrow_mut() = 0;

        let mut data = account.try_borrow_mut_data()?;
        for byte in data.deref_mut().iter_mut() {
            *byte = 0;
        }

        let dst: &mut [u8] = &mut data;
        let mut cursor = std::io::Cursor::new(dst);
        cursor
            .write_all(&anchor_lang::__private::CLOSED_ACCOUNT_DISCRIMINATOR)
            .unwrap();

        Ok(())
    }
}

#[derive(Accounts)]
pub struct Close<'info> {
    account: Account<'info, Data>,
    destination: AccountInfo<'info>,
}

#[account]
pub struct Data {
    data: u64,
}
```

请注意，上面的示例使用了 Anchor 的 `CLOSED_ACCOUNT_DISCRIMINATOR`。这只是一个帐户鉴别器，其中每个字节都是 `255`。该判别器本身没有任何固有含义，但如果您将其与帐户验证检查相结合，每当传递具有此鉴别器的帐户到指令时返回错误，您将阻止您的程序无意中处理已关闭的帐户的指令。

### 手动强制撤销资金

还有一个小问题。虽然将帐户数据清零并添加一个 “closed” 帐户鉴别器的做法可以防止您的程序被利用，但用户仍然可以在指令结束之前退还帐户的 lamports，从而使帐户不被垃圾回收。这导致一个或多个帐户处于一种无法使用但也无法被垃圾回收的悬空状态（limbo state）。

为了处理这种边缘情况，您可以考虑添加一个指令，允许**任何人**撤销带有 “closed” 帐户区分器标记的帐户的资金。该指令唯一的帐户验证是确保被撤销资金的帐户已标记为关闭。它可能看起来像这样：

```rust
use anchor_lang::__private::CLOSED_ACCOUNT_DISCRIMINATOR;
use anchor_lang::prelude::*;
use std::io::{Cursor, Write};
use std::ops::DerefMut;

...

    pub fn force_defund(ctx: Context<ForceDefund>) -> ProgramResult {
        let account = &ctx.accounts.account;

        let data = account.try_borrow_data()?;
        assert!(data.len() > 8);

        let mut discriminator = [0u8; 8];
        discriminator.copy_from_slice(&data[0..8]);
        if discriminator != CLOSED_ACCOUNT_DISCRIMINATOR {
            return Err(ProgramError::InvalidAccountData);
        }

        let dest_starting_lamports = ctx.accounts.destination.lamports();

        **ctx.accounts.destination.lamports.borrow_mut() = dest_starting_lamports
            .checked_add(account.lamports())
            .unwrap();
        **account.lamports.borrow_mut() = 0;

        Ok(())
    }

...

#[derive(Accounts)]
pub struct ForceDefund<'info> {
    account: AccountInfo<'info>,
    destination: AccountInfo<'info>,
}
```

由于任何人都可以调用此指令，这可以作为一种威慑，防止尝试复活攻击，因为攻击者正在为帐户租金豁免付费，但其他任何人都可以为自己索取退还的帐户中的 lamports。

虽然这并非必需，但这可以帮助消除与这些“悬空”帐户相关的空间和 lamports 的浪费。

## 使用 Anchor 的 `close` 约束

幸运的是，Anchor 使用 `#[account(close = <target_account>)]` 约束可以使所有这些变得更简单。该约束处理了安全关闭帐户所需的所有操作：

1. 将帐户的 lamports 转移到指定的 `<target_account>`
2. 清零帐户数据
3. 将帐户鉴别器设置为 `CLOSED_ACCOUNT_DISCRIMINATOR` 变体

您只需将其添加到希望关闭的帐户的帐户验证结构中即可。

```rust
#[derive(Accounts)]
pub struct CloseAccount {
    #[account(
        mut, 
        close = receiver
    )]
    pub data_account: Account<'info, MyData>,
    #[account(mut)]
    pub receiver: SystemAccount<'info>
}
```

`force_defund` 指令是一个可选项，如果您想要使用它，您将需要自行实现。

# 实验

为了阐明攻击者如何利用复活攻击（revival attack），让我们使用一个简单的彩票程序来进行实验，该程序使用程序账户状态来管理用户参与彩票的情况。

## 1. 设置

首先，从以下仓库的 `starter` 分支获取代码：[这里](https://github.com/Unboxed-Software/solana-closing-accounts/tree/starter)。

该代码包含程序中的两个指令和 `tests` 目录中的两个测试。

程序指令如下：

1. `enter_lottery`
2. `redeem_rewards_insecure`

当用户调用 `enter_lottery` 时，程序将初始化一个账户来存储有关用户彩票参与情况的一些状态。

由于这是一个简化的示例而不是一个完整的彩票程序，一旦用户参与了彩票，他们可以随时调用 `redeem_rewards_insecure` 指令。此指令将根据用户参与彩票的次数铸造相应数量的奖励代币给用户。发行奖励后，程序会关闭用户的彩票记录。

请花一分钟时间熟悉一下程序代码。`enter_lottery` 指令简单地在映射到用户的 PDA 上创建一个帐户，并在其中初始化一些状态。

`redeem_rewards_insecure` 指令执行一些帐户和数据验证，向给定的代币账户发行代币，然后通过移除其 lamports 关闭彩票账户。

然而，请注意 `redeem_rewards_insecure` 指令**只**转移出帐户的 lamports，使得该帐户容易受到复活攻击。

## 2. 测试不安全的程序

成功阻止其帐户关闭的攻击者随后可以多次调用 `redeem_rewards_insecure`，索取超过其应得奖励的奖励。

一些起始测试已经编写，展示了这种漏洞。查看 `tests` 目录中的 `closing-accounts.ts` 文件。`before` 函数中有一些设置，然后有一个简单的测试为 `attacker` 创建一个新的彩票条目。

最后，有一个测试演示了攻击者如何在索取奖励后保持帐户存活，并再次索取奖励。该测试如下所示：

```typescript
it("attacker  can close + refund lottery acct + claim multiple rewards", async () => {
    // claim multiple times
    for (let i = 0; i < 2; i++) {
      const tx = new Transaction()
      // instruction claims rewards, program will try to close account
      tx.add(
        await program.methods
          .redeemWinningsInsecure()
          .accounts({
            lotteryEntry: attackerLotteryEntry,
            user: attacker.publicKey,
            userAta: attackerAta,
            rewardMint: rewardMint,
            mintAuth: mintAuth,
            tokenProgram: TOKEN_PROGRAM_ID,
          })
          .instruction()
      )

      // user adds instruction to refund dataAccount lamports
      const rentExemptLamports =
        await provider.connection.getMinimumBalanceForRentExemption(
          82,
          "confirmed"
        )
      tx.add(
        SystemProgram.transfer({
          fromPubkey: attacker.publicKey,
          toPubkey: attackerLotteryEntry,
          lamports: rentExemptLamports,
        })
      )
      // send tx
      await sendAndConfirmTransaction(provider.connection, tx, [attacker])
      await new Promise((x) => setTimeout(x, 5000))
    }

    const ata = await getAccount(provider.connection, attackerAta)
    const lotteryEntry = await program.account.lotteryAccount.fetch(
      attackerLotteryEntry
    )

    expect(Number(ata.amount)).to.equal(
      lotteryEntry.timestamp.toNumber() * 10 * 2
    )
})
```

这个测试做了以下几件事情：
1. 调用 `redeem_rewards_insecure` 来兑现用户的奖励。
2. 在同一个事务中，添加一个指令来退还用户的 `lottery_entry`，在它实际上被关闭之前。
3. 成功重复步骤 1 和 2，为第二次兑现奖励。

理论上，您可以无限次地重复步骤 1-2，直到以下情况之一发生：a) 程序没有更多的奖励可给予，或者 b) 有人注意到并修复了这个漏洞。在任何真实的程序中，这显然是一个严重的问题，因为它允许恶意攻击者耗尽整个奖励池。

## 3. 创建一个 `redeem_rewards_secure` 指令

为了防止这种情况发生，我们将创建一个新的指令，使用 Anchor 的 `close` 约束来安全地关闭彩票账户。如果您愿意，可以随时尝试自己实现。

新的账户验证结构名为 `RedeemWinningsSecure` 应该如下所示：

```rust
#[derive(Accounts)]
pub struct RedeemWinningsSecure<'info> {
    // program expects this account to be initialized
    #[account(
        mut,
        seeds = [user.key().as_ref()],
        bump = lottery_entry.bump,
        has_one = user,
        close = user
    )]
    pub lottery_entry: Account<'info, LotteryAccount>,
    #[account(mut)]
    pub user: Signer<'info>,
    #[account(
        mut,
        constraint = user_ata.key() == lottery_entry.user_ata
    )]
    pub user_ata: Account<'info, TokenAccount>,
    #[account(
        mut,
        constraint = reward_mint.key() == user_ata.mint
    )]
    pub reward_mint: Account<'info, Mint>,
    ///CHECK: mint authority
    #[account(
        seeds = [MINT_SEED.as_bytes()],
        bump
    )]
    pub mint_auth: AccountInfo<'info>,
    pub token_program: Program<'info, Token>
}
```

它应该与原始的 `RedeemWinnings` 账户验证结构完全相同，只是在 `lottery_entry` 账户上有一个额外的 `close = user` 约束。这将告诉 Anchor 通过清零数据、将其 lamports 转移到 `user` 账户，并将账户鉴别器设置为 `CLOSED_ACCOUNT_DISCRIMINATOR` 来关闭该账户。如果程序已尝试关闭它，最后一步将防止再次使用该账户。

然后，我们可以在新的 `RedeemWinningsSecure` 结构中创建一个 `mint_ctx` 方法，以帮助向代币程序进行铸造 CPI。

```Rust
impl<'info> RedeemWinningsSecure <'info> {
    pub fn mint_ctx(&self) -> CpiContext<'_, '_, '_, 'info, MintTo<'info>> {
        let cpi_program = self.token_program.to_account_info();
        let cpi_accounts = MintTo {
            mint: self.reward_mint.to_account_info(),
            to: self.user_ata.to_account_info(),
            authority: self.mint_auth.to_account_info()
        };

        CpiContext::new(cpi_program, cpi_accounts)
    }
}
```

最后，新的安全指令的逻辑应如下所示：

```rust
pub fn redeem_winnings_secure(ctx: Context<RedeemWinningsSecure>) -> Result<()> {

    msg!("Calculating winnings");
    let amount = ctx.accounts.lottery_entry.timestamp as u64 * 10;

    msg!("Minting {} tokens in rewards", amount);
    // program signer seeds
    let auth_bump = *ctx.bumps.get("mint_auth").unwrap();
    let auth_seeds = &[MINT_SEED.as_bytes(), &[auth_bump]];
    let signer = &[&auth_seeds[..]];

    // redeem rewards by minting to user
    mint_to(ctx.accounts.mint_ctx().with_signer(signer), amount)?;

    Ok(())
}
```

这段逻辑简单地计算生命用户的奖励并进行了奖励转移。然而，由于账户验证结构中的 `close` 约束，攻击者不应该能够多次调用此指令。

## 4. 测试程序

为了测试我们的新安全指令，让我们创建一个新的测试，尝试调用 `redeemingWinningsSecure` 两次。我们期望第二次调用会抛出错误。

```typescript
it("attacker cannot claim multiple rewards with secure claim", async () => {
    const tx = new Transaction()
    // instruction claims rewards, program will try to close account
    tx.add(
      await program.methods
        .redeemWinningsSecure()
        .accounts({
          lotteryEntry: attackerLotteryEntry,
          user: attacker.publicKey,
          userAta: attackerAta,
          rewardMint: rewardMint,
          mintAuth: mintAuth,
          tokenProgram: TOKEN_PROGRAM_ID,
        })
        .instruction()
    )

    // user adds instruction to refund dataAccount lamports
    const rentExemptLamports =
      await provider.connection.getMinimumBalanceForRentExemption(
        82,
        "confirmed"
      )
    tx.add(
      SystemProgram.transfer({
        fromPubkey: attacker.publicKey,
        toPubkey: attackerLotteryEntry,
        lamports: rentExemptLamports,
      })
    )
    // send tx
    await sendAndConfirmTransaction(provider.connection, tx, [attacker])

    try {
      await program.methods
        .redeemWinningsSecure()
        .accounts({
          lotteryEntry: attackerLotteryEntry,
          user: attacker.publicKey,
          userAta: attackerAta,
          rewardMint: rewardMint,
          mintAuth: mintAuth,
          tokenProgram: TOKEN_PROGRAM_ID,
        })
        .signers([attacker])
        .rpc()
    } catch (error) {
      console.log(error.message)
      expect(error)
    }
})
```

运行 `anchor test` 来查看测试是否通过。输出将类似于以下内容：

```bash
  closing-accounts
    ✔ Enter lottery (451ms)
    ✔ attacker can close + refund lottery acct + claim multiple rewards (18760ms)
AnchorError caused by account: lottery_entry. Error Code: AccountDiscriminatorMismatch. Error Number: 3002. Error Message: 8 byte discriminator did not match what was expected.
    ✔ attacker cannot claim multiple rewards with secure claim (414ms)
```

请注意，这并不防止恶意用户完全退还 lamports 到他们的账户 - 它只是保护我们的程序免受在应关闭时意外重复使用该账户。到目前为止，我们还没有实现 `force_defund` 指令，但我们可以。如果你感兴趣，可以自己尝试一下！

关闭账户的最简单和最安全的方法是使用 Anchor 的 `close` 约束。如果您需要更多定制行为而无法使用此约束，请确保复制其功能，以确保您的程序是安全的。

如果您想查看最终的解决方案代码，您可以在[同一个存储库的 `solution` 分支](https://github.com/Unboxed-Software/solana-closing-accounts/tree/solution)上找到它。

# 挑战

与本单元的其他课程一样，避免此安全漏洞的机会在于审计您自己或其他程序。

花些时间审查至少一个程序，并确保当账户关闭时，它们不容易受到复活攻击的影响。

请记住，如果您发现其他程序中的漏洞或攻击，请立即通知它们！如果您发现自己的程序中有漏洞或攻击，请务必立即修补它。

## 完成了实验吗？

将您的代码推送到 GitHub，并[告诉我们您对本课程的看法](https://form.typeform.com/to/IPH0UGz7#answers-lesson=e6b99d4b-35ed-4fb2-b9cd-73eefc875a0f)！