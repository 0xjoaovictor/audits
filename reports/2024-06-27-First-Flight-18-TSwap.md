<table>
    <tr><th></th><th></th></tr>
    <tr>
        <td><img src="../assets/tswap.png" width="250" height="250" /></td>
        <td>
            <h1>First Flight #18: T-Swap</h1>
            <h2>DEXk</h2>
            <p>Prepared by: 0xjoaovictor</p>
            <p>Date: June 27 2024</p>
        </td>
    </tr>
</table>

# About **First Flight #18: T-Swap**

This project is meant to be a permissionless way for users to swap assets between each other at a fair price. You can think of T-Swap as a decentralized asset/token exchange (DEX). T-Swap is known as an Automated Market Maker (AMM) because it doesn't use a normal "order book" style exchange, instead it uses "Pools" of an asset. It is similar to Uniswap.

# Summary & Scope

The following contracts were in scope:
- src/PoolFactory.sol
- src/TSwapPool.sol

# Summary of Findings

| ID     | Title                        | Severity      | Fixed |
| ------ | ---------------------------- | ------------- | ----- |
| [H-01] | `TSwapPool::sellPoolTokens` don't sell the pool tokens correctly | High | ✓ |
| [M-01] | `TSwapPool::deposit` function don't verify if deadline is late | Medium | ✓ |
| [L-01] | `TSwapPool::swapExactInput` don't return the output amount | Low | ✓ |

# Detailed Findings

## [H-01] `TSwapPool::sellPoolTokens` don't sell the pool tokens correctly

### Summary

When `TSwapPool::sellPoolTokens` is called, depending on the balance of the user the swap fails.

### Vulnerability Details

Inside the `TSwapPool::sellPoolTokens`, the function `TSwapPool::swapExactOutput` is called, as we can see below:

```solidity
function sellPoolTokens(
        uint256 poolTokenAmount
    ) external returns (uint256 wethAmount) {
        return
            swapExactOutput( // here
                i_poolToken,
                i_wethToken,
                poolTokenAmount,
                uint64(block.timestamp)
            );
    }
```
But when the `TSwapPool::swapExactOutput` is called, is passed the third parameter `poolTokenAmount`, it means that user want to receive this quantity in weth token, but user may not have this quantity in pool tokens to sell, getting an error of `ERC20InsufficientBalance`.

We need to call `TSwapPool::swapExactInput` instead of `TSwapPool:swapExactOutput` because we only know the quantity that user want to sell in pool token.

### Inpact

The user may not sell your pool tokens getting and `ERC20InsufficientBalance` error.

#### Tools Used

Solidity and Foundry

### Proof of Concept

Add the folloing PoC to `test/unit/TSwapPool.t.sol`:

```solidity
    function testSellPoolTokens() public {
        // Add funds to the pool
        vm.startPrank(liquidityProvider);
        weth.approve(address(pool), 100e18);
        poolToken.approve(address(pool), 100e18);
        pool.deposit(100e18, 100e18, 100e18, uint64(block.timestamp));
        vm.stopPrank();

        vm.startPrank(user);
        poolToken.approve(address(pool), 100_000e18);

        uint256 userPoolTokenInitialBalance = poolToken.balanceOf(user);
        uint256 userWethInitialBalance = weth.balanceOf(user);
        uint256 expectedUserWethFinalBalance = userWethInitialBalance
            + pool.getOutputAmountBasedOnInput(
                userPoolTokenInitialBalance, poolToken.balanceOf(address(pool)), weth.balanceOf(address(pool))
            );

        pool.sellPoolTokens(userPoolTokenInitialBalance);

        assertEq(poolToken.balanceOf(user), 0, "The balance of pool tokens should be 0");
        assertEq(
            weth.balanceOf(user) == expectedUserWethFinalBalance,
            true,
            "The balance of WETH should be equal the expected balance"
        );
    }
```

### Recommendations

You need to call the `TSwapPool:swapExactInput` to make the swap correctly, for example:

```diff
function sellPoolTokens(
        uint256 poolTokenAmount
    ) external returns (uint256 wethAmount) {
-        return
-            swapExactOutput(
-                i_poolToken,
-                i_wethToken,
-                poolTokenAmount,
-                uint64(block.timestamp)
-            );
+        uint256 minOutputAmount = getOutputAmountBasedOnInput(
+           poolTokenAmount, i_poolToken.balanceOf(address(this)), i_wethToken.balanceOf(address(this))
+        );

+        return swapExactInput(
+                i_poolToken, 
+                poolTokenAmount, 
+                i_wethToken, 
+                minOutputAmount, 
+                uint64(block.timestamp)
+            );
    }
```

## [M-01] `TSwapPool::deposit` function don't verify if deadline is late

### Summary

In the `TSwapPool::deposit` function we don't have a verification if deadline is late.

### Vulnerability Details

The function `TSwapPool::deposit` don't verify the deadline parameter, as we can see below:

```solidity
    function deposit(
        uint256 wethToDeposit,
        uint256 minimumLiquidityTokensToMint,
        uint256 maximumPoolTokensToDeposit,
        uint64 deadline
    )
        external
        revertIfZero(wethToDeposit) // here we don't have a modifier to verify the deadline
        returns (uint256 liquidityTokensToMint)
```

### Impact

Because of the lack os this check the function `TSwapPool::deposit` will be accept deposits after the deadline

### Tools Used

* Solidity and Foundry

### Proof of Concept

Add the following PoC to `test/unit/TSwapPool.t.sol`:

```solidity
    function testDepositWithDeadlineLate() public {
        vm.startPrank(liquidityProvider);
        weth.approve(address(pool), 100e18);
        poolToken.approve(address(pool), 100e18);

        uint64 deadlineLate = uint64(0);
        pool.deposit(100e18, 100e18, 100e18, deadlineLate);

        assertEq(pool.balanceOf(liquidityProvider), 100e18);
        assertEq(weth.balanceOf(liquidityProvider), 100e18);
        assertEq(poolToken.balanceOf(liquidityProvider), 100e18);

        assertEq(weth.balanceOf(address(pool)), 100e18);
        assertEq(poolToken.balanceOf(address(pool)), 100e18);
    }
```

### Recommendations

You can use the existent modifier `revertIfDeadlinePassed` in the `TSwapPool::deposit`:

```diff
    function deposit(
        uint256 wethToDeposit,
        uint256 minimumLiquidityTokensToMint,
        uint256 maximumPoolTokensToDeposit,
        uint64 deadline
    )
        external
        revertIfZero(wethToDeposit)
+       revertIfDeadlinePassed(deadline)
        returns (uint256 liquidityTokensToMint)
    {
...
```

## [L-01] `TSwapPool::swapExactInput` don't return the output amount

### Summary

When `TSwapPool::swapExactInput` is called it's make the swap but don't return the output amount to the user.

### Vulnerability Details

When `TSwapPool::swapExactInput` is called we don't have the output amount returned as we can see below:

```solidity
    function swapExactInput(
        IERC20 inputToken,
        uint256 inputAmount,
        IERC20 outputToken,
        uint256 minOutputAmount,
        uint64 deadline
    )
        public
        revertIfZero(inputAmount)
        revertIfDeadlinePassed(deadline)
        returns (uint256 output) // this variable is not used
    {
        uint256 inputReserves = inputToken.balanceOf(address(this));
        uint256 outputReserves = outputToken.balanceOf(address(this));

        uint256 outputAmount = getOutputAmountBasedOnInput(
            inputAmount,
            inputReserves,
            outputReserves
        );

        if (outputAmount < minOutputAmount) {
            revert TSwapPool__OutputTooLow(outputAmount, minOutputAmount);
        }

        _swap(inputToken, inputAmount, outputToken, outputAmount);
    }
```

### Impact

Because of lack this return, we don't have the output amount that user receives when make the swap

### Tools Used

Solidity and Foundry

### Proof of Concept

Add the following PoC to `test/unit/TSwapPool.t.sol`:

```solidity
    function testDepositSwapExactInput() public {
        // Add funds to the pool
        vm.startPrank(liquidityProvider);
        weth.approve(address(pool), 100e18);
        poolToken.approve(address(pool), 100e18);
        pool.deposit(100e18, 100e18, 100e18, uint64(block.timestamp));
        vm.stopPrank();

        vm.startPrank(user);
        poolToken.approve(address(pool), 10e18);
        // After we swap, there will be ~110 tokenA, and ~91 WETH
        // 100 * 100 = 10,000
        // 110 * ~91 = 10,000
        uint256 expected = 9e18;

        uint256 userInitialPoolTokenBalance = poolToken.balanceOf(user);
        uint256 userInitialWethBalance = weth.balanceOf(user);

        uint256 outputAmount = pool.swapExactInput(poolToken, userInitialPoolTokenBalance, weth, expected, uint64(block.timestamp));

        uint256 userFinalPoolTokenBalance = poolToken.balanceOf(user);
        uint256 userFinalWethBalance = weth.balanceOf(user);

        assertEq(userFinalPoolTokenBalance, 0, "The balance of pool tokens should be 0");
        assertEq(userFinalWethBalance > userInitialWethBalance, true, "The balance of WETH should be greater than the initial balance");
        assertEq(outputAmount >= expected, true, "The output amount should be greather than or equal to the expected amount");
    }
```

### Recommendations

In the `TSwapPool::swapExactInput` you could:

* Rename from `returns (uint256 output)` to `returns (uint256 outputAmount)`
* And remove the other declaration from `uint256 outputAmount = getOutputAmountBasedOnInput(` to `outputAmount = getOutputAmountBasedOnInput(`

```diff
    function swapExactInput(
        IERC20 inputToken,
        uint256 inputAmount,
        IERC20 outputToken,
        uint256 minOutputAmount,
        uint64 deadline
    )
        public
        revertIfZero(inputAmount)
        revertIfDeadlinePassed(deadline)
-      returns (uint256 output)
+      returns (uint256 outputAmount)
    {
        uint256 inputReserves = inputToken.balanceOf(address(this));
        uint256 outputReserves = outputToken.balanceOf(address(this));

-      uint256 outputAmount = getOutputAmountBasedOnInput(
+      outputAmount = getOutputAmountBasedOnInput(
            inputAmount,
            inputReserves,
            outputReserves
        );

        if (outputAmount < minOutputAmount) {
            revert TSwapPool__OutputTooLow(outputAmount, minOutputAmount);
        }

        _swap(inputToken, inputAmount, outputToken, outputAmount);
    }
```
