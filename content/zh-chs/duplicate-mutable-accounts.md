---
title: 重复的可变账户
objectives:
- 解释与需要两个相同类型的可变账户相关的安全风险以及如何避免这些风险
- 使用长格式 Rust 实现检查重复可变账户
- 使用 Anchor 约束实现检查重复可变账户
---

# 总结

- 当一个指令需要两个相同类型的可变账户时，攻击者可以两次传入相同的账户，导致账户以意想不到的方式被修改。
- 要在 Rust 中检查重复的可变账户，只需比较两个账户的公钥，如果它们相同，则抛出错误。

  ```rust
  if ctx.accounts.account_one.key() == ctx.accounts.account_two.key() {
      return Err(ProgramError::InvalidArgument)
  }
  ```

- 在 Anchor 中，您可以使用 `constraint` 来为一个账户添加显式约束，检查它是否与另一个账户相同。

# 概述

重复的可变账户（Duplicate Mutable Accounts）指的是一个指令需要两个相同类型的可变账户。在这种情况下，您应该验证两个账户是不同的，以防止将同一个账户传递给指令两次。

由于程序将每个账户视为独立的，两次传入相同的账户可能导致第二个账户以意想不到的方式发生变化。这可能导致非常微小的问题，或者是灾难性的问题 - 这实际上取决于代码更改了哪些数据以及这些账户如何使用。无论如何，这是所有开发人员都应该意识到的一个漏洞。

## 没有检查

例如，想象一个程序，在单个指令中更新了 `user_a` 和 `user_b` 的 `data` 字段。该指令设置给 `user_a` 的值与 `user_b` 的值不同。如果没有验证 `user_a` 和 `user_b` 是不同的账户，程序将会首先更新 `user_a` 账户的 `data` 字段，然后在假设 `user_b` 是一个独立账户的前提下，第二次更新 `data` 字段为另一个值。

您可以在下面的代码中看到这个例子。在这里没有检查以验证 `user_a` 和 `user_b` 不是同一个账户。如果将相同的账户传递给 `user_a` 和 `user_b`，则会导致账户的 `data` 字段被设置为 `b`，即使意图是在不同的账户上分别设置值为 `a` 和 `b`。根据 `data` 代表的内容，这可能是一个轻微的意外副作用，或者可能意味着严重的安全风险。允许 `user_a` 和 `user_b` 是相同的账户可能会导致

```rust
use anchor_lang::prelude::*;

declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");

#[program]
pub mod duplicate_mutable_accounts_insecure {
    use super::*;

    pub fn update(ctx: Context<Update>, a: u64, b: u64) -> Result<()> {
        let user_a = &mut ctx.accounts.user_a;
        let user_b = &mut ctx.accounts.user_b;

        user_a.data = a;
        user_b.data = b;
        Ok(())
    }
}

#[derive(Accounts)]
pub struct Update<'info> {
    user_a: Account<'info, User>,
    user_b: Account<'info, User>,
}

#[account]
pub struct User {
    data: u64,
}
```

## 在指令中添加检查

要解决这个问题，简单地在指令逻辑中添加一个检查，验证 `user_a` 的公钥是否与 `user_b` 的公钥不同，如果它们相同则返回错误。

```rust
if ctx.accounts.user_a.key() == ctx.accounts.user_b.key() {
    return Err(ProgramError::InvalidArgument)
}
```

这个检查确保了 `user_a` 和 `user_b` 不是同一个账户。

```rust
use anchor_lang::prelude::*;

declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");

#[program]
pub mod duplicate_mutable_accounts_secure {
    use super::*;

    pub fn update(ctx: Context<Update>, a: u64, b: u64) -> Result<()> {
        if ctx.accounts.user_a.key() == ctx.accounts.user_b.key() {
            return Err(ProgramError::InvalidArgument.into())
        }
        let user_a = &mut ctx.accounts.user_a;
        let user_b = &mut ctx.accounts.user_b;

        user_a.data = a;
        user_b.data = b;
        Ok(())
    }
}

#[derive(Accounts)]
pub struct Update<'info> {
    user_a: Account<'info, User>,
    user_b: Account<'info, User>,
}

#[account]
pub struct User {
    data: u64,
}
```

## 使用 Anchor 的 `constraint`

如果您使用 Anchor，一个更好的解决方案是将检查添加到账户验证结构而不是指令逻辑中。

您可以使用 `#[account(..)]` 属性宏和 `constraint` 关键字为账户添加手动约束。`constraint` 关键字将检查其后的表达式是否评估为 true 或 false，如果表达式评估为 false，则返回错误。

下面的示例将检查从指令逻辑移到账户验证结构中，方法是为 `#[account(..)]` 属性添加一个 `constraint`。

```rust
use anchor_lang::prelude::*;

declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");

#[program]
pub mod duplicate_mutable_accounts_recommended {
    use super::*;

    pub fn update(ctx: Context<Update>, a: u64, b: u64) -> Result<()> {
        let user_a = &mut ctx.accounts.user_a;
        let user_b = &mut ctx.accounts.user_b;

        user_a.data = a;
        user_b.data = b;
        Ok(())
    }
}

#[derive(Accounts)]
pub struct Update<'info> {
    #[account(constraint = user_a.key() != user_b.key())]
    user_a: Account<'info, User>,
    user_b: Account<'info, User>,
}

#[account]
pub struct User {
    data: u64,
}
```

# 实验

让我们通过创建一个简单的石头、剪刀、布程序来进行练习，演示如何在程序中未检查重复的可变账户会导致未定义行为。

该程序将初始化“玩家（player）”账户，并有一个单独的指令，需要两个玩家账户来表示开始一场石头、剪刀、布游戏。

- 一个 `initialize` 指令用于初始化一个 `PlayerState` 账户
- 一个 `rock_paper_scissors_shoot_insecure` 指令，需要两个 `PlayerState` 账户，但不检查传递给指令的账户是否不同
- 一个 `rock_paper_scissors_shoot_secure` 指令，与 `rock_paper_scissors_shoot_insecure` 指令相同，但添加了一个约束，确保两个玩家账户不同

## 1. Starter

开始之前，请在 [该仓库](https://github.com/unboxed-software/solana-duplicate-mutable-accounts/tree/starter) 的 `starter` 分支上下载起始代码。起始代码包括一个具有两个指令和测试文件的样板设置。

`initialize` 指令用于初始化一个新的 `PlayerState` 账户，该账户存储玩家的公钥和一个设置为 `None` 的 `choice` 字段。

`rock_paper_scissors_shoot_insecure` 指令需要两个 `PlayerState` 账户，并要求每个玩家从 `RockPaperScissors` 枚举中选择，但不检查传递给指令的账户是否不同。这意味着单个账户可以在指令中用于两个 `PlayerState` 账户。

```rust
use anchor_lang::prelude::*;
use borsh::{BorshDeserialize, BorshSerialize};

declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");

#[program]
pub mod duplicate_mutable_accounts {
    use super::*;

    pub fn initialize(ctx: Context<Initialize>) -> Result<()> {
        ctx.accounts.new_player.player = ctx.accounts.payer.key();
        ctx.accounts.new_player.choice = None;
        Ok(())
    }

    pub fn rock_paper_scissors_shoot_insecure(
        ctx: Context<RockPaperScissorsInsecure>,
        player_one_choice: RockPaperScissors,
        player_two_choice: RockPaperScissors,
    ) -> Result<()> {
        ctx.accounts.player_one.choice = Some(player_one_choice);

        ctx.accounts.player_two.choice = Some(player_two_choice);
        Ok(())
    }
}

#[derive(Accounts)]
pub struct Initialize<'info> {
    #[account(
        init,
        payer = payer,
        space = 8 + 32 + 8
    )]
    pub new_player: Account<'info, PlayerState>,
    #[account(mut)]
    pub payer: Signer<'info>,
    pub system_program: Program<'info, System>,
}

#[derive(Accounts)]
pub struct RockPaperScissorsInsecure<'info> {
    #[account(mut)]
    pub player_one: Account<'info, PlayerState>,
    #[account(mut)]
    pub player_two: Account<'info, PlayerState>,
}

#[account]
pub struct PlayerState {
    player: Pubkey,
    choice: Option<RockPaperScissors>,
}

#[derive(Clone, Copy, BorshDeserialize, BorshSerialize)]
pub enum RockPaperScissors {
    Rock,
    Paper,
    Scissors,
}
```

## 2. 测试 `rock_paper_scissors_shoot_insecure` 指令

测试文件包含调用 `initialize` 指令两次以创建两个玩家账户的代码。

添加一个测试，通过将 `playerOne.publicKey` 作为 `playerOne` 和 `playerTwo` 都传递给 `rock_paper_scissors_shoot_insecure` 指令来调用它。

```typescript
describe("duplicate-mutable-accounts", () => {
	...
	it("Invoke insecure instruction", async () => {
        await program.methods
        .rockPaperScissorsShootInsecure({ rock: {} }, { scissors: {} })
        .accounts({
            playerOne: playerOne.publicKey,
            playerTwo: playerOne.publicKey,
        })
        .rpc()

        const p1 = await program.account.playerState.fetch(playerOne.publicKey)
        assert.equal(JSON.stringify(p1.choice), JSON.stringify({ scissors: {} }))
        assert.notEqual(JSON.stringify(p1.choice), JSON.stringify({ rock: {} }))
    })
})
```

运行 `anchor test` 以查看交易成功完成，尽管在指令中将同一个账户用作两个账户。由于在指令中将 `playerOne` 账户用作两个玩家，注意 `choice` 存储在 `playerOne` 账户上也被覆盖，并错误地设置为 `scissors`。

```bash
duplicate-mutable-accounts
  ✔ Initialized Player One (461ms)
  ✔ Initialized Player Two (404ms)
  ✔ Invoke insecure instruction (406ms)
```

允许重复账户不仅在游戏中没有太多意义，而且会导致未定义的行为。如果我们进一步构建这个程序，程序只选择了一个选项，因此无法与第二个选项进行比较。游戏每次都会以平局结束。对于人类来说，`playerOne` 的选择是石头还是剪刀也不清楚，所以程序的行为是奇怪的。

## 3. 添加 `rock_paper_scissors_shoot_secure` 指令

接下来，返回到 `lib.rs` 并添加一个 `rock_paper_scissors_shoot_secure` 指令，该指令使用 `#[account(...)]` 宏添加额外的 `constraint` 来检查 `player_one` 和 `player_two` 是否为不同的账户。

```rust
#[program]
pub mod duplicate_mutable_accounts {
    use super::*;
		...
        pub fn rock_paper_scissors_shoot_secure(
            ctx: Context<RockPaperScissorsSecure>,
            player_one_choice: RockPaperScissors,
            player_two_choice: RockPaperScissors,
        ) -> Result<()> {
            ctx.accounts.player_one.choice = Some(player_one_choice);

            ctx.accounts.player_two.choice = Some(player_two_choice);
            Ok(())
        }
}

#[derive(Accounts)]
pub struct RockPaperScissorsSecure<'info> {
    #[account(
        mut,
        constraint = player_one.key() != player_two.key()
    )]
    pub player_one: Account<'info, PlayerState>,
    #[account(mut)]
    pub player_two: Account<'info, PlayerState>,
}
```

## 7. 测试 `rock_paper_scissors_shoot_secure` 指令

为了测试 `rock_paper_scissors_shoot_secure` 指令，我们将两次调用该指令。首先，我们将使用两个不同的玩家账户调用该指令，以检查该指令是否按预期工作。然后，我们将使用 `playerOne.publicKey` 作为两个玩家账户调用该指令，我们预期会失败。

```typescript
describe("duplicate-mutable-accounts", () => {
	...
    it("Invoke secure instruction", async () => {
        await program.methods
        .rockPaperScissorsShootSecure({ rock: {} }, { scissors: {} })
        .accounts({
            playerOne: playerOne.publicKey,
            playerTwo: playerTwo.publicKey,
        })
        .rpc()

        const p1 = await program.account.playerState.fetch(playerOne.publicKey)
        const p2 = await program.account.playerState.fetch(playerTwo.publicKey)
        assert.equal(JSON.stringify(p1.choice), JSON.stringify({ rock: {} }))
        assert.equal(JSON.stringify(p2.choice), JSON.stringify({ scissors: {} }))
    })

    it("Invoke secure instruction - expect error", async () => {
        try {
        await program.methods
            .rockPaperScissorsShootSecure({ rock: {} }, { scissors: {} })
            .accounts({
                playerOne: playerOne.publicKey,
                playerTwo: playerOne.publicKey,
            })
            .rpc()
        } catch (err) {
            expect(err)
            console.log(err)
        }
    })
})
```

运行 `anchor test` 来查看指令是否按预期工作，并且使用 `playerOne` 账户两次是否返回预期的错误。

```bash
'Program Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS invoke [1]',
'Program log: Instruction: RockPaperScissorsShootSecure',
'Program log: AnchorError caused by account: player_one. Error Code: ConstraintRaw. Error Number: 2003. Error Message: A raw constraint was violated.',
'Program Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS consumed 5104 of 200000 compute units',
'Program Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS failed: custom program error: 0x7d3'
```

简单的约束就足以关闭这个漏洞。虽然有些牵强，但这个例子说明了如果你在编写程序时假设两个相同类型的账户将是不同的账户实例，但没有明确地将约束写入程序中，可能会发生奇怪的行为。始终考虑你期望从程序中获得的行为以及是否明确。

如果你想查看最终的解决方案代码，可以在 [该仓库](https://github.com/Unboxed-Software/solana-duplicate-mutable-accounts/tree/solution) 的 `solution` 分支上找到它。

# 挑战

就像本单元中的其他课程一样，避免这种安全漏洞的机会在于审查自己或其他程序。

花一些时间审查至少一个程序，并确保任何具有两个相同类型的可变账户的指令都受到适当的约束，以避免重复。

请记住，如果你在别人的程序中发现了错误或漏洞，请立即通知他们！如果你在自己的程序中发现了错误或漏洞，请务必立即修复它。

## 完成实验了吗？

将你的代码推送到 GitHub 并 [告诉我们你对这节课的看法](https://form.typeform.com/to/IPH0UGz7#answers-lesson=9b759e39-7a06-4694-ab6d-e3e7ac266ea7)！