---
title-meta: "Vault Guardians Security Review"
author-meta: "theonomiMC"
documentclass: article
papersize: a4
fontsize: 11pt
mainfont: "DejaVu Sans"
sansfont: "DejaVu Sans"
monofont: "DejaVu Sans Mono"
geometry:
  - margin=22mm
colorlinks: true
linkcolor: blue
urlcolor: blue
header-includes:
  - |
    ```{=latex}
    \usepackage{graphicx}
    \usepackage{titlesec}
    \usepackage{fancyhdr}
    \usepackage{fvextra}
    \usepackage{xcolor}
    \definecolor{codeblue}{HTML}{355C7D}

    \titlespacing*{\section}{0pt}{28pt}{12pt}
    \titlespacing*{\subsection}{0pt}{24pt}{10pt}
    \titlespacing*{\subsubsection}{0pt}{16pt}{8pt}

    \RecustomVerbatimEnvironment{verbatim}{Verbatim}{
        breaklines=true,
        breakanywhere=true,
        fontsize=\small,
        formatcom=\color{codeblue}
    }

    \pagestyle{fancy}
    \fancyhf{}
    \fancyhead[L]{\small Vault Guardians}
    \fancyhead[R]{\small Security Review}
    \fancyfoot[L]{\small theonomiMC}
    \fancyfoot[R]{\thepage}
    \setlength{\headheight}{15pt}
    \setcounter{tocdepth}{2}
    \renewcommand{\contentsname}{Table of Contents}
    ```
---

```{=latex}
\begin{titlepage}
\centering

\vspace*{1cm}

{\Huge\bfseries VAULT GUARDIANS\par}

\vspace{0.7cm}

{\LARGE Smart Contract Security Review\par}

\vspace{0.5cm}

{\large Security Review\par}

\vspace{1cm}

\includegraphics[width=0.75\textwidth]{vault-guardians-logo.png}

\vfill

{\Large\bfseries theonomiMC\par}

\vspace{0.3cm}

\href{https://github.com/theonomiMC}{github.com/theonomiMC}

\vspace{1cm}

Review period: November 10--17, 2026\par
Report date: November 17, 2026\par
Network: Ethereum\par

\end{titlepage}

\tableofcontents
\clearpage
```

# Security Audit Report - Vault Guardians

## Protocol summary

Vault Guardians lets users deposit supported ERC20 tokens into ERC4626 vaults managed by guardians. A guardian chooses how much the vault holds directly, supplies to Aave V3, and invests in Uniswap V2 liquidity.

The README allows three investment choices: Aave, Uniswap, and holding tokens without investing. Allocation values must add up to 1,000.

A guardian registers by depositing a stake into a new vault. Other users can then deposit into that vault. Users exit through `withdraw()` or `redeem()`. A guardian can change the allocation and leave through `quitGuardian()`.

The README describes guardian rewards as performance fees. The implementation mints fee shares on deposits. That difference is discussed separately from the known asset-accounting issue.

## Roles and permissions

| Role                    | Permissions                                                                                                                                 |
| ----------------------- | ------------------------------------------------------------------------------------------------------------------------------------------- |
| Depositor               | Deposit assets, mint shares, withdraw assets, redeem shares, and approve another account to spend shares.                                   |
| Guardian                | Create a WETH vault, register supported non-WETH vaults after becoming a WETH guardian, update the registered vault's allocation, and quit. |
| VaultGuardians contract | Call restricted vault functions, including allocation updates and deactivation.                                                             |
| Protocol owner / DAO    | Update the stake price and the fee denominator. Receive tokens swept from the VaultGuardians contract.                                      |
| Any account             | Call `rebalanceFunds()` on an active vault and call `sweepErc20s()`. The sweep recipient is fixed to `owner()`.                             |

Guardians are not given permission to send vault assets to arbitrary destinations. Their investment choices are limited to the supported protocols.

## Scope

```
./src/
#-- abstract
|   #-- AStaticTokenData.sol
|   #-- AStaticUSDCData.sol
|   #-- AStaticWethData.sol
#-- dao
|   #-- VaultGuardianGovernor.sol
|   #-- VaultGuardianToken.sol
#-- interfaces
|   #-- IVaultData.sol
|   #-- IVaultGuardians.sol
|   #-- IVaultShares.sol
|   #-- InvestableUniverseAdapter.sol
#-- protocol
|   #-- VaultGuardians.sol
|   #-- VaultGuardiansBase.sol
|   #-- VaultShares.sol
|   #-- investableUniverseAdapters
|       #-- AaveAdapter.sol
|       #-- UniswapAdapter.sol
#-- vendor
    #-- DataTypes.sol
    #-- IPool.sol
    #-- IUniswapV2Factory.sol
    #-- IUniswapV2Router01.sol
```

The fork setup selects Ethereum block `18,377,723`. Fork tests require an archive-capable mainnet RPC endpoint configured as `mainnet`.

## Findings summary

| Severity           |  Count |
| ------------------ | -----: |
| High               |      2 |
| Medium             |      3 |
| Low                |      4 |
| Informational      |      9 |
| **Total findings** | **18** |

These are proposed audit severities, not judging results. The count is based on the code review; All PoCs were run on a mainnet fork at block 18,377,723 or on unit mocks, as listed in the run conditions table.

## H-01: Incorrect WETH LP reference blocks exits

**Severity:** High

### Description

The `VaultShares` constructor looks up the LP token using the vault asset and WETH:

```solidity
i_uniswapLiquidityToken = IERC20(
    i_uniswapFactory.getPair(address(constructorData.asset), address(i_weth))
);
```

For a WETH vault, this requests the WETH/WETH pair. That pair does not exist, so the stored LP address is zero.

The adapter actually invests WETH into the WETH/USDC pair. Deposits can therefore succeed while the vault stores the wrong LP reference.

The `divestThenInvest` modifier reads the stored LP token balance before running the requested operation. Reading `balanceOf()` from the zero address fails.

### Impact

WETH vault users can deposit and receive shares, but `withdraw()` and `redeem()` revert. Rebalancing also reverts, and the guardian cannot quit because quitting calls `redeem()`.

Every guardian must first create a WETH vault before registering a supported non-WETH vault. Therefore, guardians managing other supported assets are also exposed to the blocked WETH stake.

The failure occurs before the vault can return idle WETH or unwind its investments.

### Proof of concept

Use `test_wethVaultIncorrectLpReference` in `WethFork.t.sol` on the original contracts.

The test creates a WETH vault, checks that the WETH/WETH pair and stored LP address are zero, deposits user funds, and expects all four exit or rebalance paths to revert.

The broad `expectRevert()` checks should be accompanied by the trace showing the invalid LP balance read. A revert by itself does not establish its cause.

<details>
<summary>Use this fork test: </summary>

```solidity
function test_wethVaultIncorrectLpReference() public {
    // step 1. guardian created weth vault
    deal(address(weth), guardian, mintAmount);
    uint256 stake = vaultGuardians.getGuardianStakePrice();

    vm.startPrank(guardian);
    weth.approve(address(vaultGuardians), stake);
    address wethVault = vaultGuardians.becomeGuardian(allocationData);
    vm.stopPrank();

    wethVaultShares = VaultShares(wethVault);

    // step 2. root cause
    IUniswapV2Factory factory = IUniswapV2Factory(
        IUniswapV2Router01(uniswapRouter).factory()
    );
    assertEq(factory.getPair(wethAddress, wethAddress), address(0)); // (WETH, WETH) pair doesn't exist
    assertTrue(factory.getPair(wethAddress, usdcAddress) != address(0));
    assertEq(wethVaultShares.getUniswapLiquidtyToken(), address(0)); // vault saved incorrect reference

    // step 3. user can deposit
    deal(address(weth), user, 10 ether);
    vm.startPrank(user);
    weth.approve(wethVault, 10 ether);
    wethVaultShares.deposit(10 ether, user);
    vm.stopPrank();

    uint256 userShares = wethVaultShares.balanceOf(user);
    assertGt(userShares, 0);

    // step 4. all withdrawals reverts
    vm.startPrank(user);
    vm.expectRevert();
    wethVaultShares.withdraw(1 ether, user, user);
    vm.expectRevert();
    wethVaultShares.redeem(userShares, user, user);
    vm.stopPrank();

    vm.expectRevert();
    wethVaultShares.rebalanceFunds();

    vm.startPrank(guardian);
    wethVaultShares.approve(address(vaultGuardians), type(uint256).max);
    vm.expectRevert();
    vaultGuardians.quitGuardian();
    vm.stopPrank();
}
```

</details>

### Recommended mitigation

Select the counterparty token using the same logic as the Uniswap adapter: USDC for WETH vaults and WETH for other supported assets.

Initialize the LP reference from that pair and validate that the returned address is nonzero and contains contract code before allowing vault creation.

```diff
VaultShares {
    ...
+ error VaultShares__InvalidUniswapPair();
constructor( ... )
    {
        ...
-       i_uniswapLiquidityToken = IERC20(i_uniswapFactory.getPair(address(constructorData.asset), address(i_weth)));
+        IERC20 counterPartyToken = constructorData.asset == i_weth ? i_tokenOne : i_weth;
+        address liquidityToken = i_uniswapFactory.getPair(address(constructorData.asset),  address(counterPartyToken));
+        if (liquidityToken == address(0) || liquidityToken.code.length == 0) {
+            revert VaultShares__InvalidUniswapPair();
+        }
+        i_uniswapLiquidityToken = IERC20(liquidityToken);
    }
```

## H-02: Missing router approvals block Uniswap divestment

**Severity:** High

### Description

`_uniswapDivest()` calls the router to remove liquidity and then swap the received counterparty token back into the vault asset. It does not set the allowances needed for these operations.

1. `removeLiquidity()` needs permission to transfer the vault's LP tokens.
2. The reverse swap needs permission to transfer the counterparty tokens received from removing liquidity.

The vault never approves its LP tokens for the router. The counterparty approval made during investment is spent by liquidity provision and cannot be relied on for a later exit.

```solidity
function _uniswapDivest(IERC20 token, uint256 liquidityAmount) internal returns (uint256 amountOfAssetReturned) {
    IERC20 counterPartyToken = token == i_weth ? i_tokenOne : i_weth;

    // (1) no approval of the LP token before removeLiquidity()
    (uint256 tokenAmount, uint256 counterPartyTokenAmount) = i_uniswapRouter.removeLiquidity({ ... });

    s_pathArray = [address(counterPartyToken), address(token)];
    // (2) no approval of counterPartyToken before the reverse swap
    uint256[] memory amounts = i_uniswapRouter.swapExactTokensForTokens({ ... });
    ...
}
```

### Impact

When the vault holds Uniswap LP tokens, removing liquidity fails. This blocks `withdraw()`, `redeem()`, `rebalanceFunds()`, and `quitGuardian()`.

The modifier performs the divestment before paying the user, so an available idle balance does not avoid the failure.

Tested on a mainnet fork (block 18,377,723): LINK vault, allocation 500/250/250. It is independent of H-01.

**Proof of Concept:**

<details>
<summary>Use these fork tests: </summary>

```solidity
// helper
 function _createLinkVaultWithUserDeposit()
    internal
    returns (address linkVault, address lp, uint256 userShares)
{
    uint256 stake = vaultGuardians.getGuardianStakePrice();

    deal(address(weth), guardian, stake);
    deal(address(link), guardian, stake);

    vm.startPrank(guardian);
    weth.approve(address(vaultGuardians), stake);
    vaultGuardians.becomeGuardian(allocationData);
    link.approve(address(vaultGuardians), stake);
    linkVault = vaultGuardians.becomeTokenGuardian(
        allocationData,
        IERC20(linkAddress)
    );
    vm.stopPrank();
    linkVaultShares = VaultShares(linkVault);

    IUniswapV2Factory factory = IUniswapV2Factory(
        IUniswapV2Router01(uniswapRouter).factory()
    );
    lp = factory.getPair(linkAddress, wethAddress);
    assertTrue(lp != address(0));
    assertEq(linkVaultShares.getUniswapLiquidtyToken(), lp);

    // deposit
    deal(address(link), user, 10 ether);
    vm.startPrank(user);
    link.approve(linkVault, 10 ether);
    linkVaultShares.deposit(10 ether, user);
    vm.stopPrank();

    userShares = linkVaultShares.balanceOf(user);
    assertGt(userShares, 0);
}

function test_vaultMissingLpApprovalBlocksExit() public {
    (
        address linkVault,
        address lp,
        uint256 userShares
    ) = _createLinkVaultWithUserDeposit();

    IERC20 lpToken = IERC20(lp);
    uint256 lpBefore = lpToken.balanceOf(linkVault);
    assertGt(lpBefore, 0, "vault holds LP tokens");
    assertEq(
        lpToken.allowance(linkVault, uniswapRouter),
        0,
        "router has no LP allowance"
    );

    bytes memory err = abi.encodeWithSignature(
        "Error(string)",
        "ds-math-sub-underflow"
    );

    // All withdrawals revert
    vm.startPrank(user);
    vm.expectRevert(err);
    linkVaultShares.withdraw(1 ether, user, user);
    vm.expectRevert(err);
    linkVaultShares.redeem(userShares, user, user);
    vm.stopPrank();

    vm.expectRevert();
    linkVaultShares.rebalanceFunds();

    vm.startPrank(guardian);
    linkVaultShares.approve(address(vaultGuardians), type(uint256).max);
    vm.expectRevert(err);
    vaultGuardians.quitGuardian(IERC20(linkAddress));
    vm.stopPrank();

    // Thus balances remain unchanged
    assertEq(linkVaultShares.balanceOf(user), userShares);
    assertEq(lpToken.balanceOf(linkVault), lpBefore);
}
// After approving the LP token, the withdrawal still reverts because the counterparty token allowance is insufficient.
function test_vaultMissingCounterpartyApprovalBlocksExit() public {
    (
        address linkVault,
        address lp,
        uint256 userShares
    ) = _createLinkVaultWithUserDeposit();

    vm.prank(linkVault);
    IERC20(lp).approve(uniswapRouter, type(uint256).max);

    emit log_named_uint(
        "leftover WETH allowance",
        weth.allowance(linkVault, uniswapRouter)
    );

    uint256 lpBefore = IERC20(lp).balanceOf(linkVault);

    vm.prank(user);
    vm.expectRevert(
        abi.encodeWithSignature(
            "Error(string)",
            "TransferHelper: TRANSFER_FROM_FAILED"
        )
    );
    linkVaultShares.withdraw(1 ether, user, user);

    assertEq(linkVaultShares.balanceOf(user), userShares);
    assertEq(IERC20(lp).balanceOf(linkVault), lpBefore);
}

```

</details>

The tested expectations are `ds-math-sub-underflow` for the first failure and `TransferHelper: TRANSFER_FROM_FAILED` for the second.

### Recommended mitigation

Use `SafeERC20.forceApprove()` immediately before each operation:

- Approve the correct LP token for `liquidityAmount` before removing liquidity.
- Approve the counterparty token for the amount received before the reverse swap.

After fixing both allowances, add success checks for withdrawal, redemption, rebalancing, and quitting. The tested files do not contain a complete success regression for all four paths.

```diff
function _uniswapDivest(IERC20 token, uint256 liquidityAmount) internal returns (uint256 amountOfAssetReturned) {
    IERC20 counterPartyToken = token == i_weth ? i_tokenOne : i_weth;

+   IERC20 liquidityToken = IERC20(
+       i_uniswapFactory.getPair(address(token), address(counterPartyToken))
+   );
+   liquidityToken.forceApprove(address(i_uniswapRouter), liquidityAmount);

    (uint256 tokenAmount, uint256 counterPartyTokenAmount) = i_uniswapRouter.removeLiquidity({ ... });
    s_pathArray = [address(counterPartyToken), address(token)];

+   counterPartyToken.forceApprove(address(i_uniswapRouter), counterPartyTokenAmount);

    uint256[] memory amounts = i_uniswapRouter.swapExactTokensForTokens({ ... });
    ...
}
```

## M-01: Zero strategy amounts cause valid operations to revert

**Severity:** Medium

### Description

`VaultShares.updateHoldingAllocation()` accepts allocation ratios whose sum is 1,000, including ratios with a zero Uniswap or Aave allocation. However, `_investFunds()` always calls both `_uniswapInvest()` and `_aaveInvest()`, even when the calculated investment amount is zero.

For `(hold, Uniswap, Aave) = (500, 0, 500)`, the Uniswap adapter attempts a swap with zero input. For `(500, 500, 0)`, the Aave adapter calls `supply()` with amount zero. These external protocols reject the respective operations, reverting the entire deposit. A zero allocation should disable investment in that strategy.

```solidity
function _investFunds(uint256 assets) private {
        ....
@>      _uniswapInvest(IERC20(asset()), uniswapAllocation);
@>      _aaveInvest(IERC20(asset()), aaveAllocation);
    }
```

The real Uniswap router rejects a zero-input swap. Aave rejects a zero-amount supply. Choosing not to use a strategy therefore causes an external call to revert instead of skipping that strategy.

### Impact

Some accepted allocations cannot be used to create a funded vault or make deposits. This includes the hold-only option described in the README.

After H-01 and H-02 are fixed, withdrawals can still fail during the reinvestment step if one strategy receives zero. Changing back to a working allocation can restore operation. Quitting deactivates reinvestment, but it still depends on successful divestment.

### Proof of concept

The included tests are:

- `test_zeroUniswapAllocationCallsAdapter`
- `test_zeroAaveAllocationCallsAdapter`
- `test_zeroAaveAllocationBlocksWithdrawals`

The guardian fixture creates a vault with a working allocation. Each test then sets one strategy allocation to zero and checks the resulting zero-amount external call and deposit revert.

<details>
<summary>Code</summary>

```solidity
function test_zeroUniswapAllocationCallsAdapter() public hasGuardian {
    // 1. Allocation update succeeds: total remains 1000.
    vm.prank(guardian);
    vaultGuardians.updateHoldingAllocation(weth, AllocationData(500, 0, 500));

    AllocationData memory allocation = wethVaultShares.getAllocationData();
    assertEq(allocation.uniswapAllocation, 0);

    // 2. Prepare a valid user deposit.
    uint256 amount = 10 ether;
    deal(address(weth), user, amount);

    vm.prank(user);
    weth.approve(address(wethVaultShares), amount);

    uint256 userBalanceBefore = weth.balanceOf(user);
    uint256 supplyBefore = wethVaultShares.totalSupply();

    // 3. Require a router call whose first argument, amountIn, is zero.
    // expectCall matches this calldata prefix.
    vm.expectCall(
        uniswapRouter,
        abi.encodeWithSelector(
            IUniswapV2Router01.swapExactTokensForTokens.selector,
            uint256(0)
        )
    );

    vm.startPrank(user);
    vm.expectRevert();
    wethVaultShares.deposit(amount, user);
    vm.stopPrank();

    // 4. Reverted deposit must leave no partial changes.
    assertEq(weth.balanceOf(user), userBalanceBefore);
    assertEq(wethVaultShares.balanceOf(user), 0);
    assertEq(wethVaultShares.totalSupply(), supplyBefore);
}

function test_zeroAaveAllocationCallsAdapter() public hasGuardian {
    vm.prank(guardian);
    vaultGuardians.updateHoldingAllocation(
        weth,
        AllocationData(500, 500, 0)
    );

    AllocationData memory allocation = wethVaultShares.getAllocationData();
    assertEq(allocation.aaveAllocation, 0);

    uint256 amount = 10 ether;
    deal(address(weth), user, amount);

    vm.prank(user);
    weth.approve(address(wethVaultShares), amount);

    uint256 supplyBefore = wethVaultShares.totalSupply();

    // Require the Aave supply call to receive amount = 0.
    vm.expectCall(
        aavePool,
        abi.encodeWithSignature(
            "supply(address,uint256,address,uint16)",
            address(weth),
            uint256(0),
            address(wethVaultShares),
            uint16(0)
        )
    );

    vm.startPrank(user);
    vm.expectRevert();
    wethVaultShares.deposit(amount, user);
    vm.stopPrank();

    // 4. Reverted deposit must leave no partial changes.
    assertEq(weth.balanceOf(user), amount);
    assertEq(wethVaultShares.balanceOf(user), 0);
    assertEq(wethVaultShares.totalSupply(), supplyBefore);
}

function test_zeroAaveAllocationBlocksWithdrawals() public {
    deal(address(weth), guardian, mintAmount);

    vm.startPrank(guardian);
    weth.approve(address(vaultGuardians), mintAmount);
    wethVaultShares = VaultShares(
        vaultGuardians.becomeGuardian(allocationData)
    );
    vm.stopPrank();

    address userB = makeAddr("user B");
    uint256 amount = 10 ether;
    deal(address(weth), user, amount);
    deal(address(weth), userB, amount);

    vm.startPrank(user);

    weth.approve(address(wethVaultShares), amount);
    wethVaultShares.deposit(amount, user);
    wethVaultShares.withdraw(1 ether, user, user);
    assertEq(weth.balanceOf(user), 1 ether);
    vm.stopPrank();

    vm.startPrank(userB);
    weth.approve(address(wethVaultShares), amount);
    wethVaultShares.deposit(amount, userB);
    vm.stopPrank();

    // change allocation
    vm.prank(guardian);
    vaultGuardians.updateHoldingAllocation(
        weth,
        AllocationData(500, 500, 0)
    );

    assertTrue(wethVaultShares.getAllocationData().aaveAllocation == 0);

    uint256 sharesBefore = wethVaultShares.balanceOf(userB);
    uint256 assetsBefore = weth.balanceOf(userB);

    vm.prank(userB);
    vm.expectRevert();
    wethVaultShares.withdraw(1 ether, userB, userB);

    vm.prank(userB);
    vm.expectRevert();
    wethVaultShares.redeem(sharesBefore, userB, userB);

    assertEq(wethVaultShares.balanceOf(userB), sharesBefore);
    assertEq(weth.balanceOf(userB), assetsBefore);

    vm.prank(guardian);
    vaultGuardians.updateHoldingAllocation(
        weth,
        allocationData
    );

    vm.prank(userB);
    wethVaultShares.withdraw(1 ether, userB, userB);

    assertEq(weth.balanceOf(userB), assetsBefore + 1 ether);
}

```

</details>

_The withdrawal test must run on a branch with H-01 and H-02 fixed, while leaving M-01 unfixed. Its initial successful withdrawal otherwise fails before reaching the issue under test._

### Recommended mitigation

Only call an investment adapter when its calculated investment amount is greater than zero:

```diff
-  _uniswapInvest(IERC20(asset()), uniswapAllocation);
-  _aaveInvest(IERC20(asset()), aaveAllocation);
+  if (uniswapAllocation > 0) _uniswapInvest(IERC20(asset()), uniswapAllocation);
+  if (aaveAllocation > 0) _aaveInvest(IERC20(asset()), aaveAllocation);
```

Test zero Uniswap, zero Aave, hold-only, and small deposits whose investment amounts round to zero. A nonzero Uniswap amount can still produce a zero swap input when divided by two; handle strategy minimum amounts explicitly.

## M-02: Inherited mint bypasses vault rules

**Severity:** Medium

### Description

`VaultShares` overrides `deposit()` but inherits `mint()` from ERC4626.

The inherited `mint()` transfers assets and creates shares without the extra rules implemented in `deposit()`.

| Rule                               | `deposit()` | Inherited `mint()` |
| ---------------------------------- | ----------- | ------------------ |
| Require an active vault            | Yes         | No                 |
| Mint guardian and DAO fee shares   | Yes         | No                 |
| Invest the incoming assets         | Yes         | No                 |
| Apply the vault's reentrancy guard | Yes         | No                 |

### Impact

Users can enter through `mint()` without paying the deposit-based share fee. They can also add assets after the guardian quits and the vault becomes inactive.

Assets entering through `mint()` remain idle until a later operation invests them. Inactive vaults do not rebalance through the normal active-only entry point.

The missing guard is a code difference, not a separately proven reentrancy exploit in the included tests.

### Proof of concept

Insert these tests into `test/unit/concrete/VaultSharesTest.t.sol`. The vault uses a hold-only allocation, so the known `totalAssets()` issue does not change the result.

- `test_mintSkipsTheFee`
  1. user calls `deposit()`. The guardian and the DAO each receive `shares / cut` fee shares (control check).
  2. userB calls `mint()` for the same number of shares. The guardian and the DAO receive **0** fee shares.
- `test_mintWorksAfterGuardianQuits`
  1. The guardian calls `quitGuardian()`. The vault becomes inactive.
  2. `deposit()` reverts with `VaultShares__NotActive`.
  3. `mint()` succeeds and new shares are created.

<details><summary>code</summary>

```solidity
address internal userB = makeAddr("userB");
...
// helpers
function _deposit(address who, uint256 assets) internal returns (uint256 shares) {
    weth.mint(assets, who);
    vm.startPrank(who);
    weth.approve(address(wethVaultShares), assets);
    shares = wethVaultShares.deposit(assets, who);
    vm.stopPrank();
}
function _mintShares(address who, uint256 shares) internal returns (uint256 assetsPaid) {
    uint256 needed = wethVaultShares.previewMint(shares);
    weth.mint(needed, who);
    vm.startPrank(who);
    weth.approve(address(wethVaultShares), needed);
    assetsPaid = wethVaultShares.mint(shares, who);
    vm.stopPrank();
}

function _feeHolderShares() internal view returns (uint256 guardianShares, uint256 daoShares) {
    guardianShares = wethVaultShares.balanceOf(wethVaultShares.getGuardian());
    daoShares = wethVaultShares.balanceOf(wethVaultShares.getVaultGuardians());
}
function test_mintSkipsTheFee() public {
    // Create vault
    weth.mint(mintAmount, guardian);
    vm.startPrank(guardian);
    weth.approve(address(vaultGuardians), mintAmount);
    address wethVault = vaultGuardians.becomeGuardian(AllocationData(1000, 0, 0));
    wethVaultShares = VaultShares(wethVault);
    vm.stopPrank();

    assertEq(
        wethVaultShares.totalAssets(),
        vaultGuardians.getGuardianStakePrice()
    );

    uint256 cut = wethVaultShares.getGuardianAndDaoCut();
    // deposit() pays fee shares to the guardian and to the DAO.
    (uint256 g0, uint256 d0) = _feeHolderShares();
    uint256 depositShares = _deposit(user, 10 ether);
    (uint256 g1, uint256 d1) = _feeHolderShares();
    assertGt(g1 - g0, 0, "deposit should pay a fee");
    assertEq(g1 - g0, depositShares / cut);
    assertEq(d1 - d0, depositShares / cut);
    // Bug: mint() of the same size pays no fee at all.
    _mintShares(userB, depositShares);
    (uint256 g2, uint256 d2) = _feeHolderShares();
    emit log_named_uint("fee shares paid by deposit (each side)", g1 - g0);
    emit log_named_uint("fee shares paid by mint    (each side)", g2 - g1);
    assertEq(g2, g1, "guardian got no fee from mint");
    assertEq(d2, d1, "DAO got no fee from mint");
}

function test_mintWorksAfterGuardianQuits() public {
    // Create vault
    weth.mint(mintAmount, guardian);
    vm.startPrank(guardian);
    weth.approve(address(vaultGuardians), mintAmount);
    address wethVault = vaultGuardians.becomeGuardian(AllocationData(1000, 0, 0));
    wethVaultShares = VaultShares(wethVault);
    vm.stopPrank();

    assertEq(
        wethVaultShares.totalAssets(),
        vaultGuardians.getGuardianStakePrice()
    );

    // Guardian quits. The vault becomes inactive.
    vm.startPrank(guardian);
    wethVaultShares.approve(address(vaultGuardians), type(uint256).max);
    vaultGuardians.quitGuardian();
    vm.stopPrank();
    assertFalse(wethVaultShares.getIsActive(), "vault should be inactive");

    // deposit() is blocked, as the code comment promises.
    weth.mint(1 ether, user);
    vm.startPrank(user);
    weth.approve(address(wethVaultShares), 1 ether);
    vm.expectRevert(VaultShares.VaultShares__NotActive.selector);
    wethVaultShares.deposit(1 ether, user);
    vm.stopPrank();

    // Bug: mint() is not blocked.
    uint256 supplyBefore = wethVaultShares.totalSupply();
    _mintShares(userB, 1 ether);
    emit log_named_uint("UserB still mints after deactivated",wethVaultShares.balanceOf(userB));
    assertGt(wethVaultShares.balanceOf(userB), 0, "mint worked in an inactive vault");
    assertGt(wethVaultShares.totalSupply(), supplyBefore);
}
```

Expected outputs:

```bash
[PASS] test_mintSkipsTheFee() (gas: 4452888)
Logs:
  fee shares paid by deposit (each side): 10019999999999999
  fee shares paid by mint    (each side): 0

[PASS] test_mintWorksAfterGuardianQuits() (gas: 4441124)
Logs:
  UserB still mints after deactivated: 1000000000000000000
```

</details>

### Recommended mitigation

Override `mint()` to enforce the active-vault check, reentrancy protection, fee policy, and investment behavior used by `deposit()`.

Preserve ERC4626 checks, including `maxMint(receiver)`, asset rounding, and share allowance behavior. Put shared entry logic in an internal function so that `deposit()` and `mint()` apply the same rules.

## M-03: Repeated rebalancing makes holders pay repeated trading costs

**Severity:** Medium

**PoC qualification:** The test compares two runs from the same snapshot. The known `totalAssets()` issue affects both runs in the same way, so the difference between them is the cost of the extra rebalances.

### Description

Anyone can call `rebalanceFunds()` on an active vault:

```solidity
function rebalanceFunds()
    public isActive divestThenInvest nonReentrant {}
```

The modifier removes all invested positions and invests the remaining assets again, even when the allocation has not changed.

Each Uniswap cycle involves swaps and their trading costs. Repeating the cycle makes the vault pay those costs repeatedly. The caller pays transaction gas; holders bear the vault's trading costs.

`withdraw()` and `redeem()` also use the full cycle, so restricting only `rebalanceFunds()` would leave another way to trigger unnecessary trading.

### Impact

An unrelated account can repeatedly force trading that reduces the value available to holders. The amount depends on the allocation, pool conditions, vault size, and how much of the pool's fee income returns to the vault through its own LP position.

The user deposits 100 WETH in each run. Without extra rebalances, redemption returns 107.363210318217719061 WETH. After ten extra rebalances, it returns 104.824217008993634916 WETH.

The extra rebalances reduce the user's redemption proceeds by 2.538993309224084145 WETH, approximately 2.36% compared with the baseline.

The baseline returns more than the deposit because of the known totalAssets() issue. This effect is the same in both runs, so it does not change the difference. The loss is about 0.254 WETH per call, which is in line with two Uniswap swaps with the 0.3% pool fee on the Uniswap part of the vault. Ten calls used 3,495,820 gas. The caller pays the gas, the holders pay the loss.

### Proof of concept

It compares two branches from a snapshot:

1. Deposit and redeem without extra rebalances.
2. Repeat the same deposit, perform ten rebalances from another address, and redeem.

Its allocation is `(200, 800, 0)`. On the original contracts, zero Aave supply prevents even registration. H-01 and H-02 also block WETH exits. This test therefore requires H-01, H-02, and M-01 fixed first.

<details>
<summary> Insert this test in WethFork.t.sol</summary>

```solidity
function test_rebalance_griefing_valueLoss() public {
    deal(address(weth), guardian, mintAmount);

    vm.startPrank(guardian);
    weth.approve(address(vaultGuardians), mintAmount);
    VaultShares vault = VaultShares(
        vaultGuardians.becomeGuardian(AllocationData(200, 800, 0))
    );
    vm.stopPrank();

    deal(address(weth), user, 200 ether);
    uint256 id = vm.snapshotState();

    vm.startPrank(user);
    weth.approve(address(vault), 100 ether);
    vault.deposit(100 ether, user);
    uint256 userShares0 = vault.balanceOf(user);
    uint256 out0 = vault.redeem(userShares0, user, user);
    vm.stopPrank();

    assertTrue(vm.revertToState(id), "snapshot restore failed");

    address attacker = makeAddr("attacker");

    vm.startPrank(user);
    weth.approve(address(vault), 100 ether);
    vault.deposit(100 ether, user);
    vm.stopPrank();

    uint256 g = gasleft();
    for (uint256 i = 0; i < 10; i++) {
        vm.prank(attacker);
        vault.rebalanceFunds();
    }
    uint256 gasUsed = g - gasleft();

    vm.startPrank(user);
    uint256 userSharesN = vault.balanceOf(user);
    uint256 outN = vault.redeem(userSharesN, user, user);
    vm.stopPrank();

    uint256 loss = out0 > outN ? out0 - outN : 0;
    uint256 lossPercent = (loss * 10_000) / out0; // basis points (e.g. 100 = 1%)

    emit log_named_uint("out0:", out0);
    emit log_named_uint("outN:", outN);
    emit log_named_uint("Loss (out0 - outN):", loss);
    emit log_named_uint("Loss % (basis points):", lossPercent);
    emit log_named_uint("Avg loss per rebalance:", loss / 10);
    emit log_named_uint("Gas used for 10 rebalances:", gasUsed);
    assertLt(
        outN,
        out0,
        "extra rebalances should reduce redemption proceeds"
    );
}
```

Recorded output from the test run with the final `assertLt(outN, out0)` check:

```bash
[PASS] test_rebalance_griefing_valueLoss() (gas: 7928964)
Logs:
  out0:: 107363210318217719061
  outN:: 104824217008993634916
  Loss (out0 - outN):: 2538993309224084145
  Loss % (basis points):: 236
  Avg loss per rebalance:: 253899330922408414
  Gas used for 10 rebalances:: 3495820
```

</details>

### Recommended mitigation

Use idle assets first for exits and divest only the shortfall. Avoid a full portfolio rebuild on every withdrawal or redemption.

Limit full rebalances through permissions and/or a suitable interval and meaningful rebalance conditions. A rebalance should serve a portfolio need rather than repeat trades without a strategy change. Test repeated small exits as well as direct rebalance calls.

## L-01: A shared raw stake amount prevents normal USDC registration

**Severity:** Low

### Description

`VaultGuardiansBase` uses the same raw stake value for all supported tokens:

```solidity
uint256 internal s_guardianStakePrice = 10 ether;
```

Registration transfers `s_guardianStakePrice` units of the selected token. No decimal conversion is applied.

| Token | Decimals | amount (`1e19` raw units) |
| ----- | -------: | ------------------------: |
| WETH  |       18 |                   10 WETH |
| LINK  |       18 |                   10 LINK |
| USDC  |        6 |   10,000,000,000,000 USDC |

Token decimals describe units; they do not establish equal economic value between WETH, LINK, and USDC.

### Impact

The default stake makes normal USDC guardian registration impractical. The included fork test also checks that the requested raw USDC stake exceeds USDC's total supply at the selected historical block.

Changing the shared stake to suit USDC would change the requirement for WETH and LINK too. The configuration cannot independently express appropriate stakes for all three assets.

### Proof of concept

<details>
<summary> Insert this test in <code>WethFork.t.sol</code></summary>

```solidity
function test_stakeIgnoresTokenDecimals() public {
    uint256 stake = vaultGuardians.getGuardianStakePrice();
    uint256 usdcSupply = IERC20(usdcAddress).totalSupply();
    emit log_named_uint("stake (raw)", stake);
    emit log_named_uint("USDC total supply (raw)", usdcSupply);

    assertEq(IERC20Metadata(usdcAddress).decimals(), 6);
    assertGt(stake, usdcSupply); // stake is larger than all USDC that exists

    // guardian needs a WETH vault first
    deal(address(weth), guardian, mintAmount);
    vm.startPrank(guardian);
    weth.approve(address(vaultGuardians), stake);
    vaultGuardians.becomeGuardian(allocationData);

    deal(usdcAddress, guardian, 1_000_000e6);
    IERC20(usdcAddress).approve(address(vaultGuardians), stake);

    deal(linkAddress, guardian, 100e18);
    IERC20(linkAddress).approve(address(vaultGuardians), stake);
    uint256 balanceBefore = IERC20(linkAddress).balanceOf(guardian);

    address linkVault = vaultGuardians.becomeTokenGuardian(
        allocationData,
        IERC20(linkAddress)
    );

    vm.expectRevert(bytes("ERC20: transfer amount exceeds balance")); // transferFrom of 1e19 raw USDC fails
    vaultGuardians.becomeTokenGuardian(allocationData, IERC20(usdcAddress));
    vm.stopPrank();

    assertEq(VaultShares(linkVault).asset(), linkAddress);
    assertEq(
        address(
            vaultGuardians.getVaultFromGuardianAndToken(
                guardian,
                IERC20(linkAddress)
            )
        ),
        linkVault
    );

    // Confirm that the stake was taken.
    assertEq(
        IERC20(linkAddress).balanceOf(guardian),
        balanceBefore - stake
    );
}
```

</details>
<br>

The test checks USDC decimals and historical supply, creates the required WETH guardian, successfully creates a LINK vault with enough LINK, and expects USDC registration to fail with insufficient balance.

The supply claim is limited to the fork block checked by the test. A revert with an artificially assigned balance alone is not enough evidence; the unit conversion and supply comparison explain the problem.

### Recommended mitigation

Configure the stake separately for each supported token in that token's native units. Validate changes and document whether stakes are fixed token quantities or intended to represent comparable economic values.

## L-02: Registering again replaces the first vault in the registry

**Severity:** Low

### Description

`_becomeTokenGuardian()` writes a new vault into the guardian/token mapping without checking whether a vault is already registered:

```solidity
s_guardians[msg.sender][token] = IVaultShares(address(tokenVault));
```

The same guardian can therefore create a second vault for the same asset while the first remains active.

Allocation updates and `quitGuardian()` use the mapping, so they reach only the latest vault.

### Impact

The first vault loses access to the normal allocation-update and deactivation routes. After quitting the second vault, the mapping is cleared; a second quit does not find the first vault.

This does not by itself prove that users' assets or the guardian's stake are locked. The old vault still exists, and its direct exit functions must be assessed separately after fixing the adapter blockers.

This finding concerns loss of management access. It does not claim the known governance-token farming issue as a new finding.

### Proof of concept

**Use the test after H-01 and H-02 are fixed.**

It checks that two different active vaults are created, the mapping points to the second, quitting deactivates only the second, the mapping becomes zero, and another quit reverts while the first remains active.

<details>
<summary> Insert this test in <code>WethFork.t.sol</code></summary>

```solidity
function test_doubleBecomeGuardian() public {
    // 1. Guardian creates the first WETH vault.
    deal(address(weth), guardian, mintAmount);

    vm.startPrank(guardian);
    weth.approve(address(vaultGuardians), mintAmount);
    VaultShares firstVault = VaultShares(
        vaultGuardians.becomeGuardian(allocationData)
    );
    vm.stopPrank();

    // 2. Same guardian creates another WETH vault without quitting.
    deal(address(weth), guardian, mintAmount);

    vm.startPrank(guardian);
    weth.approve(address(vaultGuardians), mintAmount);
    VaultShares secondVault = VaultShares(
        vaultGuardians.becomeGuardian(allocationData)
    );
    vm.stopPrank();

    assertTrue(address(firstVault) != address(secondVault));
    assertTrue(firstVault.getIsActive());
    assertTrue(secondVault.getIsActive());

    // Registry now points to the second vault.
    assertEq(
        address(
            vaultGuardians.getVaultFromGuardianAndToken(guardian, weth)
        ),
        address(secondVault)
    );

    // 4. Approve shares and quit: this closes ONLY the second vault.
    vm.startPrank(guardian);
    secondVault.approve(
        address(vaultGuardians),
        secondVault.balanceOf(guardian)
    );
    vaultGuardians.quitGuardian();
    vm.stopPrank();

    assertFalse(secondVault.getIsActive());
    assertTrue(firstVault.getIsActive());

    // Guardian's registry entry has been cleared.
    assertEq(
        address(
            vaultGuardians.getVaultFromGuardianAndToken(guardian, weth)
        ),
        address(0)
    );

    // 5. A second quit cannot reach the first vault.
    vm.startPrank(guardian);
    vm.expectRevert(
        abi.encodeWithSignature(
            "VaultGuardiansBase__NotAGuardian(address,address)",
            guardian,
            address(weth)
        )
    );
    vaultGuardians.quitGuardian();
    vm.stopPrank();

    assertTrue(firstVault.getIsActive());
}
```

</details>

### Recommended mitigation

If only one active vault per guardian and asset is intended, reject registration when an entry already exists. Allow a new registration after the previous vault has been properly deactivated.

If multiple vaults are intended, track them separately and let authorized management calls identify the target vault explicitly.

## L-03: Deposit fees do not match the stated performance fee

**Severity:** Low

### Description

The README describes a performance fee. The code instead mints guardian and DAO shares on every deposit, using:

```solidity
_mint(i_guardian, shares / i_guardianAndDaoCut);
_mint(i_vaultGuardians, shares / i_guardianAndDaoCut);
```

### Impact

Fee shares are minted to the guardian and DAO on every deposit, even when the vault earns no profit. These additional shares dilute depositors’ ownership. The dilution increases as cut decreases. The default cut is 1000, and DAO updates affect only newly created vaults.

This is a mismatch between the stated fee policy and the implementation. The report does not treat an agreed deposit fee as an exploit.

### Proof of concept

Measured in unit tests using mocks and the hold-only allocation `(1000, 0, 0)`. No assets are invested in external strategies:

- A user deposits 10 WETH and redeems at once. The user receives 9.990009990009990010 WETH. The loss is about 0.00999 WETH (about 0.1%).
- In a separate test, a second user deposits 10 WETH while the first user keeps their shares. The first user's redeemable value decreases from 9.990009990009990010 WETH to 9.983354420396392416 WETH.

```solidity
address internal userB = makeAddr("userB");
...
function test_depositThenRedeemLosesMoneyWithoutProfit() public {
    weth.mint(mintAmount, guardian);
    vm.startPrank(guardian);
    weth.approve(address(vaultGuardians), mintAmount);
    address wethVault = vaultGuardians.becomeGuardian(AllocationData(1000, 0, 0));
    wethVaultShares = VaultShares(wethVault);
    vm.stopPrank();

    emit log_named_uint("guardian and DAO cut (shares / cut)", wethVaultShares.getGuardianAndDaoCut());

    uint256 amountIn = 10 ether;
    uint256 shares = _deposit(user, amountIn);
    vm.prank(user);
    uint256 amountOut = wethVaultShares.redeem(shares, user, user);

    emit log_named_uint("assets in ", amountIn);
    emit log_named_uint("assets out", amountOut);
    assertLt(amountOut, amountIn, "user lost money with zero profit in the vault");
    emit log_named_uint("loss", amountIn - amountOut);
}

function test_newDepositDilutesExistingHolderWithoutProfit() public {
    weth.mint(mintAmount, guardian);
    vm.startPrank(guardian);
    weth.approve(address(vaultGuardians), mintAmount);
    address wethVault = vaultGuardians.becomeGuardian(AllocationData(1000, 0, 0));
    wethVaultShares = VaultShares(wethVault);
    vm.stopPrank();

    uint256 userShares = _deposit(user, 10 ether);
    uint256 valueBefore = wethVaultShares.previewRedeem(userShares);
    // Nothing happens in the vault: no yield, no donation, no price change.
    _deposit(userB, 10 ether);
    uint256 valueAfter = wethVaultShares.previewRedeem(userShares);

    emit log_named_uint("User shares worth before UserB deposit", valueBefore);
    emit log_named_uint("User shares worth after  UserB deposit", valueAfter);
    assertLt(valueAfter, valueBefore, "existing holder was diluted by the fee shares");
}
```

Expected outputs:

```bash
[PASS] test_depositThenRedeemLosesMoneyWithoutProfit() (gas: 4405688)
Logs:
  guardian and DAO cut (shares / cut): 1000
  assets in : 10000000000000000000
  assets out: 9990009990009990010
  loss: 9990009990009990

[PASS] test_newDepositDilutesExistingHolderWithoutProfit() (gas: 4441209)
Logs:
  User shares worth before UserB deposit: 9990009990009990010
  User shares worth after  UserB deposit: 9983354420396392416
```

### Recommended mitigation

If the intended fee is a performance fee, remove unconditional fee-share minting from deposits. Charge fees only on measured investment gains, using correct asset accounting and a high-water mark that accounts for deposits and withdrawals.

If a deposit fee is intended instead, document it explicitly and apply it consistently to both `deposit()` and `mint()`. Deducting fee shares from the entering depositor would avoid diluting existing holders through additional fee-share issuance, but it would still be a deposit fee.

## L-04: External protocol failures block withdrawals of idle funds

**Severity:** Low

### Description

`withdraw()`, `redeem()` and `rebalanceFunds()` use the `divestThenInvest` modifier. It always removes the whole position from both Uniswap and Aave before the withdrawal runs. There is no `try/catch`, and idle funds are not used first:

```solidity
if (uniswapLiquidityTokensBalance > 0) _uniswapDivest(...);
if (aaveAtokensBalance > 0) _aaveDivest(...);
```

If one of these calls reverts, the whole withdrawal reverts. This is true even when the user's amount is fully covered by the idle balance of the vault.

The same is true for deposits. `_investFunds()` calls Aave `supply()`, so a paused or frozen reserve, or a reached supply cap, makes every `deposit()` revert. _(This part comes from the Aave rules, not from a test.)_

### Impact

If an external withdrawal call reverts, users cannot withdraw even amounts fully covered by the vault's idle balance. Withdrawals remain blocked while that external call keeps reverting.

The PoC simulates an Aave withdrawal failure. It does not demonstrate a specific real-world pause or freeze scenario.

### Proof of concept

Test `test_idleFundsLocked_whenAaveReverts` (fork, with the H-01 and H-02 fixes applied, because exits are blocked by them in the original code):

1. A guardian creates a WETH vault with allocation `(800, 100, 100)`. A user deposits 100 WETH.
2. A small `withdraw(1 ether)` works (control). The vault still holds much more than 1 WETH idle.
3. The Aave `withdraw` call is mocked to revert with `"paused"`.
4. `withdraw()` and `redeem()` revert with `"paused"`.
5. After clearing the mocked failure, a withdrawal succeeds again.

<details>
<summary> Insert this test in <code>WethFork.t.sol</code>
</summary>

```solidity
function test_idleFundsLocked_whenAaveReverts() public {
    // Create vault
    deal(address(weth), guardian, mintAmount);
    vm.startPrank(guardian);
    weth.approve(address(vaultGuardians), mintAmount);
    VaultShares vault = VaultShares(
        vaultGuardians.becomeGuardian(AllocationData(800, 100, 100))
    );
    vm.stopPrank();

    // Deposit 100 ether to vault
    deal(address(weth), user, 100 ether);
    vm.startPrank(user);
    weth.approve(address(vault), 100 ether);
    vault.deposit(100 ether, user);
    uint256 assetsBefore = weth.balanceOf(user);
    vault.withdraw(1 ether, user, user);
    assertEq(weth.balanceOf(user), assetsBefore + 1 ether);
    vm.stopPrank();

    assertGt(weth.balanceOf(address(vault)), 1 ether);
    emit log_named_uint(
        "idle WETH in vault",
        weth.balanceOf(address(vault))
    );

    vm.mockCallRevert(
        aavePool,
        abi.encodeWithSelector(
            bytes4(keccak256("withdraw(address,uint256,address)"))
        ),
        "paused"
    );

    vm.prank(user);
    vm.expectRevert(bytes("paused"));
    vault.withdraw(1 ether, user, user);

    uint256 shares = vault.balanceOf(user);
    vm.prank(user);
    vm.expectRevert(bytes("paused"));
    vault.redeem(shares, user, user);

    // Removing the simulated Aave failure restores withdrawals.
    vm.clearMockedCalls();

    uint256 balanceBeforeRecovery = weth.balanceOf(user);

    vm.prank(user);
    vault.withdraw(1 ether, user, user);

    uint256 balanceAfterRecovery = weth.balanceOf(user);

    assertEq(balanceAfterRecovery, balanceBeforeRecovery + 1 ether);

    emit log_named_uint(
        "User WETH balance before recovery withdrawal",
        balanceBeforeRecovery
    );
    emit log_named_uint(
        "User WETH balance after recovery withdrawal",
        balanceAfterRecovery
    );
    emit log_named_uint(
        "WETH received after clearing the mocked failure",
        balanceAfterRecovery - balanceBeforeRecovery
    );
}
```

Expected output:

```bash
[PASS] test_idleFundsLocked_whenAaveReverts() (gas: 6031652)
Logs:
  idle WETH in vault: 87188276167963637928
  User WETH balance before recovery withdrawal: 1000000000000000000
  User WETH balance after recovery withdrawal: 2000000000000000000
  WETH received after clearing the mocked failure: 1000000000000000000
```

</details>

### Recommended mitigation

Use idle assets first and divest only the shortfall. If a strategy fails, allow a withdrawal only when enough assets are available to pay it. Keep the remaining strategy positions correctly included in asset accounting.

## I-01: Parameter events contain incorrect information

**Severity:** Informational

### Description and impact

`updateGuardianStakePrice()` changes the stored stake before emitting the event. Its old-value field therefore contains the new value.

`updateGuardianAndDaoCut()` has the same ordering issue and emits `VaultGuardians__UpdatedStakePrice` instead of the fee-update event.

The parameter updates still take effect, but event-based monitoring can report misleading values or miss fee changes.

### Recommended mitigation

Save the old value before changing storage. Emit the matching event with the actual old and new values. Add event checks for both setters.

## I-02: Registration fee comments do not match the implementation

**Severity:** Informational

### Description and impact

`GUARDIAN_FEE = 0.1 ether` is unused. The comment for `becomeGuardian()` says the caller must send an ETH fee, but the function is not payable and does not collect that fee.

The implementation and comments give different registration requirements. No financial-loss claim is made without confirming that the ETH fee is an intended requirement.

### Recommended mitigation

Remove the unused constant and outdated fee comments if registration should not charge ETH. Otherwise, implement the intended fee collection and specify its recipient and exact amount.

## I-03: Unused errors and imports

**Severity:** Informational

### Description and impact

The list includes these unused declarations:

- `VaultGuardians__TransferFailed`
- `VaultGuardiansBase__NotEnoughWeth`
- `VaultGuardiansBase__CantQuitGuardianWithNonWethVaults`
- `VaultGuardiansBase__FeeTooSmall`
- The `IERC20` import in `InvestableUniverseAdapter.sol`

Unused declarations make the code harder to maintain and can suggest checks that are not actually present. No direct security impact or deployed-bytecode saving is established by their removal alone.

### Recommended mitigation

Remove unused declarations after checking that they do not point to a missing intended validation. Report any missing validation separately, with its actual consequences.

## I-04: Functions used only externally can use external visibility

**Severity:** Informational

### Description and impact

`setNotActive()` and `rebalanceFunds()` are public but are not called internally in the reviewed source. Marking them `external` would express their intended use more clearly.

This is optional code cleanup. Their current visibility does not demonstrate a security vulnerability, and no gas saving is claimed without measurement.

### Recommended mitigation

Use `external` if internal calls are not needed. Consider inheritance requirements before changing visibility.

## I-05: The guardian stake is not locked while the guardian is active

**Severity:** Informational

### Description and impact

The guardian receives ordinary transferable vault shares in exchange for the registration stake. The reviewed design does not reserve a minimum guardian share balance while the vault remains active.

After unrelated exit blockers are fixed, the guardian can potentially redeem or transfer these shares without calling `quitGuardian()`. The registry and management permissions are not tied to a minimum remaining stake.

No separate user-fund loss or slashing bypass has been demonstrated. This is an informational observation unless the specification explicitly requires the stake to remain committed while the guardian is active.

This was found by reading the code and was not tested.

### Recommended mitigation

If the stake should stay in the vault while the guardian is active, lock a minimum number of guardian shares (no redeem and no transfer below it). Otherwise, change the README.

## I-06: The README says the protocol is upgradeable, but the contracts are not

**Severity:** Informational

### Description and impact

The README says: _(The protocol is upgradeable so that if any of the platforms in the investable universe change, or we want to add more, we can do so.)_.

In the code there is no upgrade mechanism:

- No contract uses a proxy (UUPS, transparent or beacon) and there is no `initialize()` function.
- All contracts set their values in constructors. The Aave pool and the Uniswap router are `immutable`, so a vault cannot change them.
- `VaultShares` vaults are created with `new`.

What the DAO can change is only the stake price and the fee cut (`updateGuardianStakePrice()` and `updateGuardianAndDaoCut()`), and the fee cut changes only the vaults created after the change. New logic needs a new deployment. Existing directly deployed vaults cannot receive replacement implementation logic through a native upgrade mechanism. Deploying corrected contracts does not modify existing vaults.

Migration requires a functioning withdrawal or recovery route from the old vaults. If an existing vault's exit paths are blocked, a new deployment alone does not recover its funds.

This does not cause a loss by itself. It is important for the recommended fixes: a bug in `VaultShares` cannot be fixed in vaults that already exist.

### Recommended mitigation

Say in the README what "upgradeable" means (new deployments, or a real proxy pattern). If real upgrades are intended, add a proxy pattern with access control and a migration plan.

## I-07: There are no limits for the fee cut

**Severity:** Informational

### Description and impact

`updateGuardianAndDaoCut()` accepts any value. `VaultShares` divides by this value on every deposit:

```solidity
_mint(i_guardian, shares / i_guardianAndDaoCut);
_mint(i_vaultGuardians, shares / i_guardianAndDaoCut);
```

- With `0`, registration attempts that reach the initial stake deposit revert because fee calculation divides by zero. New guardian registration remains blocked until the DAO restores a valid value. Existing vaults retain their previously configured immutable cut.
- With `1`, the guardian and the DAO each receive as many shares as the depositor, so the depositor owns only one third of the new shares.

Only the DAO can set this value, so this is a trusted-role mistake and not an attack. It was found by reading the code and was not tested.

### Recommended mitigation

Check that the value is inside a sensible range (for example, at least a minimum and never zero). Add tests for the limits.

---

## I-08: The governance setup has weak defaults

**Severity:** Informational

### Description and impact

From reading `VaultGuardianGovernor`, `VaultGuardianToken` and the deploy script:

- `proposalThreshold()` is not set, so it is `0`. An account with no tokens can create proposals.
- There is no enforced timelock between a successful vote and execution. A successful proposal can be executed without an additional timelock delay.
- The token starts with zero supply (the comment in the deploy script says it mints the total supply, but the constructor mints nothing). All voting power comes from guardians who register.
- The known issue M-1 (voting delay and period are set in seconds but counted in blocks) is not repeated here.

These are settings of the system. No separate loss is shown. No independent governance takeover or financial loss is demonstrated by these settings. The known governance-token farming issue is excluded from this finding.

### Recommended mitigation

Set a proposal threshold, add a timelock between the vote and the execution, and decide how the first tokens are distributed.

---

## I-09: Missing checks in the constructors

**Severity:** Informational

### Description and impact

- `AaveAdapter` and `UniswapAdapter` do not check that their addresses are not zero.
- `VaultShares` reads the aToken with `getReserveData(asset).aTokenAddress` and does not check that the reserve exists. If the aToken address is zero, every `withdraw()` and `redeem()` reverts, as in H-01. The tested PoCs do not demonstrate this configuration failure for all supported assets. Validate each supported asset against the configured Aave pool at the selected deployment or fork block.

### Recommended mitigation

Revert in the constructors if an address is zero. In `VaultShares`, revert if the aToken is zero, the same way as the Uniswap pair check in the H-01 fix.

## PoC run conditions

| Finding | Environment | Required fixes / conditions                                                                                  |
| ------- | ----------- | ------------------------------------------------------------------------------------------------------------ |
| H-01    | Fork        | Original contracts.                                                                                          |
| H-02    | Fork        | Original contracts, LINK vault. The second test adds LP approval to reach the counterparty approval failure. |
| M-01    | Fork        | First two tests: original contracts. Withdrawal test: H-01 and H-02 fixed, M-01 unfixed.                     |
| M-02    | Unit mocks  | Original contracts, hold-only allocation `(1000, 0, 0)`.                                                     |
| M-03    | Fork        | H-01, H-02 and M-01 fixed. Correct asset accounting is also needed to isolate trading costs.                 |
| L-01    | Fork        | Original contracts. The test checks registration, not exits.                                                 |
| L-02    | Fork        | H-01 and H-02 fixed because the test calls `quitGuardian()`.                                                 |
| L-03    | Unit mocks  | Original contracts, hold-only allocation `(1000, 0, 0)`.                                                     |
| L-04    | Fork        | H-01 and H-02 fixed. Aave withdrawal failure is mocked.                                                      |
| I-01    | Unit mocks  | Dedicated event tests are not included in this report.                                                       |

---

## Excluded warnings and known issues

**Known issues (audit-data).** These are not reported: H-1 (slippage and deadline), H-2 (`totalAssets()` ignores invested funds), H-3 (governance token farming), M-1 (governor timing), L-1 (LINK vault name and symbol), L-2 (unassigned return in `_aaveDivest()`).

The known H-1 names only `_uniswapInvest()`. The same missing slippage protection (`amountOutMin = 0`, `deadline = block.timestamp`) is also in `_uniswapDivest()`, which runs in `withdraw()`, `redeem()` and the public `rebalanceFunds()`. A caller can start these swaps in the same transaction as a pool price change, so no mempool access is needed. We treated this as part of H-1.

**Checked and not reported:**

| Topic                                              | Result                                                                                                                                                                                                    |
| -------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Deposits that give zero shares                     | Not found in a fuzz test of deposits, withdrawals, redemptions and donations on the hold-only mock.                                                                                                       |
| Fee rounding (small deposits)                      | Small deposits can produce zero fee shares due to rounding. No practical exploit was demonstrated.                                                                                                        |
| Inflation and donation attack on the first deposit | Not practical. The first deposit is the guardian stake (`10^19` raw units), and the DAO receives fee shares that the guardian cannot redeem. This was reasoned from the code and fuzzed only on the mock. |
| Reentrancy                                         | The `divestThenInvest` modifier runs outside `nonReentrant`, but WETH, USDC, LINK, Aave and Uniswap give no callback to the vault. No usable path was found.                                              |
| External call in the constructor                   | The called contracts are fixed and trusted.                                                                                                                                                               |
| Allowances left after investing (Aave and Uniswap) | Ten rebalance cycles completed without allowance errors after the missing approvals were fixed.                                                                                                           |
| Governance flash loans                             | Code review found no way to use a same-transaction flash loan to bypass the voting snapshot. This was not tested with a dedicated PoC.                                                                    |

**Limits of this review:**

- Fork tests use Ethereum block `18,377,723`. Results can differ at other blocks.
- The invariant fuzz tests ran only on the mock with the hold-only allocation. The mocks do not model Uniswap and Aave.
- Findings that depend on fixes of other findings say so in the table above.

---

## Recommended fix order

1. Fix **H-01** and **H-02** to restore exit paths.
2. Fix **M-01** so zero strategy amounts do not block valid operations.
3. Fix **M-02** so `deposit()` and `mint()` enforce consistent rules.
4. Correct the known asset-accounting issue and repeat the **M-03** measurement.
5. Address **M-03** and confirm the intended fee policy for **L-03**.
6. Address **L-01**, **L-02** and **L-04**.
7. Address the informational findings.

After each fix, check the intended behavior: valid operations succeed, invalid operations revert, and the reported bypass or unintended loss no longer occurs. Verify withdrawals, redemptions, rebalancing and guardian exits.
