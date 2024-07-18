# [HIGH-01] RestakeManager::calculateTVLs incorrectly calculates the withdraw queue balances, leading to heavily distorted TVL results

## Submission Link
https://github.com/code-423n4/2024-04-renzo-findings/issues/275

## Impact
RestakeManager calculates TVLs incorrectly leading to a distorted tvl value, which is used in essential protocol functions. The consequences can be:

* minting fewer/more shares when depositing/withdrawing depending on the `withdrawQueue` balances
* providing inflated/deflated mint rates by `BalancerRateProvider::getRate()`
* DOS on deposits and withdraws

## Vulnerability details
[`RestakeManager::calculateTVLs()` is used to calculate the total value](https://github.com/code-423n4/2024-04-renzo/blob/519e518f2d8dec9acf6482b84a181e403070d22d/contracts/RestakeManager.sol#L274) currently locked in the protocol. It factors:

* the value of all tokens in all operators
* the latest version of the contract adds the balances of the withdraw queue for each token

The calculated TVL returned from that function is essential when determining the mint/burn ratios for users and thus is vital for the proper functioning of the protocol. Currently it's being used in the following contracts:

* `RestakeManager::deposit` ([link](https://github.com/code-423n4/2024-04-renzo/blob/519e518f2d8dec9acf6482b84a181e403070d22d/contracts/RestakeManager.sol#L500-L504)) & `RestakeManager::depositETH` ([link](https://github.com/code-423n4/2024-04-renzo/blob/519e518f2d8dec9acf6482b84a181e403070d22d/contracts/RestakeManager.sol#L594)) - to check max limits and the amount of ezETH(shares) to mint
* `BalancerRateProvider::getRate` ([link](https://github.com/code-423n4/2024-04-renzo/blob/519e518f2d8dec9acf6482b84a181e403070d22d/contracts/RateProvider/BalancerRateProvider.sol#L31))  - Contract to Provide Mint Rate of LRT(ezETH)
* `WithdrawQueue::withdraw` ([link](https://github.com/code-423n4/2024-04-renzo/blob/519e518f2d8dec9acf6482b84a181e403070d22d/contracts/Withdraw/WithdrawQueue.sol#L217)) - calculates the ezETH to burn when withdrawing

The problem is in the following [new lines of the `calculateTVLs()` logic](https://github.com/code-423n4/2024-04-renzo/blob/519e518f2d8dec9acf6482b84a181e403070d22d/contracts/RestakeManager.sol#L316-L321):

```solidity
function calculateTVLs() public view returns (uint256[][] memory, uint256[] memory, uint256) {
        ....

        // Iterate through the ODs
        uint256 odLength = operatorDelegators.length;

        ....

        for (uint256 i = 0; i < odLength; ) {
            // Track the TVL for this OD
            uint256 operatorTVL = 0;

            ....

            uint256 tokenLength = collateralTokens.length;
            for (uint256 j = 0; j < tokenLength; ) {

                ....

                // Set the value in the array for this OD
                operatorValues[j] = renzoOracle.lookupTokenValue(
                    collateralTokens[j],
                    operatorBalance
                );

                ....

                // record token value of withdraw queue
                if (!withdrawQueueTokenBalanceRecorded) {
                    totalWithdrawalQueueValue += renzoOracle.lookupTokenValue(
 ---->                  collateralTokens[i],
 ---->                  collateralTokens[j].balanceOf(withdrawQueue)
                    );
                }

                unchecked {
                    ++j;
                }
            }

    }
```

As you can see above the function loops through every operator using the `i` index and inside every loop there is second nested loop that iterates through all tokens using the `j` index.

The problem is in the calculation of `withdrawQueue` token balances:

```solidity
                // record token value of withdraw queue
                if (!withdrawQueueTokenBalanceRecorded) {
                    totalWithdrawalQueueValue += renzoOracle.lookupTokenValue(
                     
                      //@audit - wrong index  - uses i instead of j
                      collateralTokens[i],
                      collateralTokens[j].balanceOf(withdrawQueue)
                    );
                }
```

Instead of using the index `j` for the `collateralTokens[]` array it uses the `i` index which corresponds to the operators array.

Depending on the difference in count between the operators and tokens, the consequences can be various:

* Complete DOS of the deposit/withdraw functions -  In case the `operatorDelegators` array is bigger than the `collateralTokens` array the function will revert with `array out-of-bounds` error.
* Minting/Burning more/less shares than necessary - In case the `operatorDelegators` array is smaller than `collateralTokens` array it will repeatedly accumulate the first token only -> which can  lead to lower/higher balances for the `withdrawQueue`, which will make the deposit to TVL ratio bigger/smaller.

## Tools Used
Manual review

## Recommended Mitigation Steps
Fix the issue by using the token index instead of the operators one:

```solidity
if (!withdrawQueueTokenBalanceRecorded) {
    totalWithdrawalQueueValue += renzoOracle.lookupTokenValue(
        collateralTokens[j], // <- use j, not i
        collateralTokens[j].balanceOf(withdrawQueue)
    );
}
```

## Assessed type
Other

# [HIGH-02] User can extract protocol value by sandwiching oracle and rebasing transactions

## Submission Link
https://github.com/code-423n4/2024-04-renzo-findings/issues/325

# Lines of code
https://github.com/code-423n4/2024-04-renzo/blob/519e518f2d8dec9acf6482b84a181e403070d22d/contracts/RestakeManager.sol#L491-L495 https://github.com/code-423n4/2024-04-renzo/blob/519e518f2d8dec9acf6482b84a181e403070d22d/contracts/Withdraw/WithdrawQueue.sol#L206 https://github.com/code-423n4/2024-04-renzo/blob/519e518f2d8dec9acf6482b84a181e403070d22d/contracts/Withdraw/WithdrawQueue.sol#L279

# Vulnerability details
## Impact
User can extract protocol value each time there is a oracle price change or rebasing (in stETH).

## Proof of Concept
Attacker can spot oracle price change transaction or rebasing stETH one in the mempool and frontrun, depositing tokens and minting ezETH and then do a backrun and imidiatelly call withdraw on the new price withdrawing more of the same token or a different one.

This can be profitable because there are no fees upon depositing/withdrawing and because the 2 step withdrawal process calculates how much tokens can be claimed by the user in the first step immidiately after a deposit (with no cooldown time between deposit and withdraw).

Let's imagine the following scenario:

User deposit 100e+18 of tokenC and mints 100 ezETH (price of 1 tokenC == 1 ETH)

`totalSupply of ezETH`: 3000 ezETH

TVL of tokens in ETH:

* tokenA: 1000e+18
* tokenB: 1000e+18
* tokenC: 1000e+18

## Case 1: TokenC price decreases from 1 ETH to 0.95 ETH
User will place a withdraw with his 100 ezETH

`totalSupply of ezETH`: 3000 ezETH

TVL of tokens in ETH:

* tokenA: 1000e+18
* tokenB: 1000e+18
* tokenC: 950e+18

```
`TotalTVL` = A + B + C = 2950 ETH
100 ezETH in ETH = TotalTVL * burnEzEthAmount / ezEthTotalSupply
100 ezETH in ETH = 2950e+18 * 100e+18 / 3000e+18
100 ezETH in ETH = 98.333333 ETH
```

This is the calculation happening inside:

```solidity
uint256 amountToRedeem = renzoOracle.calculateRedeemAmount(
    _amount,
    ezETH.totalSupply(),
    totalTVL
);
```

Depending on the exit asset the user will have different results.

The following calculations of the out token are happening in:

```solidity
if (_assetOut != IS_NATIVE) {
    // Get ERC20 asset equivalent amount
    amountToRedeem = renzoOracle.lookupTokenAmountFromValue(
        IERC20(_assetOut),
        amountToRedeem
    );
}
```

### **1.1 Profit in same token**:
```
exit tokenC amount = ETH representation of provided ezETH * 1e+18 / price of exit token (tokenC)
exit tokenC amount = 98.333333e+18 ETH * 1e+18 / 0.95e+18
exit tokenC amount = 103.50877e+18

Profit = exit token amount - amount he would have had if just held
Profit in tokenC = 103.50877e+18 - 100e+18 = 3.50877e+18
```

**Result in underlying ETH:** user is better of doing the sandwitch attack

### **1.2 Profit in native currency**:
His 100 tokenC worth 100 ETH after the oracle price update his tokens would be worth 95 ETH

```
Profit in native ETH = ETH representation of provided ezETH - ETH value if he just held his 100 tokenC after update
Profit in native ETH = 98.333333e+18 - 95 ETH
Profit in native ETH = 3.333333 ETH
```

**Result in underlying ETH:** user is better of doing the sandwitch attack

## Case 2: TokenC price increases from 1 ETH to 1.05 ETH
User will place a withdraw with his 100 ezETH

`totalSupply of ezETH`: 3000 ezETH

TVL of tokens in ETH:

* tokenA: 1000e+18
* tokenB: 1000e+18
* tokenC: 1050e+18

```
Total TVL in ETH = A + B + C = 3050 ETH
100 ezETH in ETH = TotalTVL * burnEzEthAmount / ezEthTotalSupply
100 ezETH in ETH = 3050e+18 * 100e+18 / 3000e+18
100 ezETH in ETH = 101.66667 ETH
```

### **2.1 Profit in same token**:
```
exit tokenC amount = ETH representation of provided ezETH * 1e+18 / price of exit token (tokenC)
exit tokenC amount = 101.66667e+18 ETH * 1e+18 / 1.05e+18
exit tokenC amount = 96.8254e+18

Profit = exit tokenC amount - amount he would have had if just held
Profit in tokenC = 96.8254e+18 - 100e+18 = -3.1746e+18
```

**Result in underlying ETH the user is better of holding**

### **2.2 Profit in native currency**:
His 100 tokenC worth 100 ETH after the oracle price update his tokens would be worth 105 ETH

```
Profit in native ETH = ETH representation of provided ezETH - ETH value if he just held his 100 tokenC after update
Profit in native ETH = 101.66667 ETH - 105 ETH
Profit in native ETH = -3.333333 ETH
```

**Result in underlying ETH the user is better of holding**

### Summary
User is better of doing nothing if the price of his token increases.

If the price decreases he can make a profit by doing the attack.

User makes a profit by not spending any time in the protocol with the exception of the cooldown withdraw period that does not affect the realised profit but just locks the profits for X amount of time after which the user will claim them.

## Tools Used
Manual Review

## Recommended Mitigation Steps
Do not allow a user to place a withdraw order if he just deposited. Have a cooldown period between these 2 operations.

There might be other more appropriate solutions to this issue but further research should be done in order not to introduce another vulnerability.

I think that moving the calculation for the tokenOut amount to the claim function instead of being in the withdraw placing order one and adding a slippage check to allow the user to control how much token he will get when he claims in case the price of ezETH he burned have changed. But this second mitigation proposal should be checked for further possible edge cases.

## Assessed type
MEV

# [HIGH-03] ReentrancyGuard causes all native currency funds to get locked inside the EigenPod contract

## Submission Link
https://github.com/code-423n4/2024-04-renzo-findings/issues/570

# Lines of code
https://github.com/code-423n4/2024-04-renzo/blob/519e518f2d8dec9acf6482b84a181e403070d22d/contracts/Delegation/OperatorDelegator.sol#L265-L269 https://github.com/code-423n4/2024-04-renzo/blob/519e518f2d8dec9acf6482b84a181e403070d22d/contracts/Delegation/OperatorDelegator.sol#L501

# Vulnerability details
## Impact
OperatorDelegator contracts are not able to make partial or full withdrawals of the ETH in their EigenPod contracts coming from the beacon chain. Effectively locking all the collaterals for each node (32 ETH each) + all profits.

## Proof of Concept
The issue is that both [`OperatorDelegator.sol:completeQueuedWithdrawal()`](https://github.com/code-423n4/2024-04-renzo/blob/519e518f2d8dec9acf6482b84a181e403070d22d/contracts/Delegation/OperatorDelegator.sol#L265-L269) and [`receive()`](https://github.com/code-423n4/2024-04-renzo/blob/519e518f2d8dec9acf6482b84a181e403070d22d/contracts/Delegation/OperatorDelegator.sol#L501) have `nonReentrant` modifier:

```solidity
    modifier nonReentrant() {
        _nonReentrantBefore();
        _;
        _nonReentrantAfter();
    }

    function _nonReentrantBefore() private {
        // On the first call to nonReentrant, _status will be _NOT_ENTERED
        require(_status != _ENTERED, "ReentrancyGuard: reentrant call");

        // Any calls to nonReentrant after this point will fail
        _status = _ENTERED;
    }

    function _nonReentrantAfter() private {
        // By storing the original value once again, a refund is triggered (see
        // https://eips.ethereum.org/EIPS/eip-2200)
        _status = _NOT_ENTERED;
    }
```

When we first make a call to [`OperatorDelegator.sol:completeQueuedWithdrawal()`](https://github.com/code-423n4/2024-04-renzo/blob/519e518f2d8dec9acf6482b84a181e403070d22d/contracts/Delegation/OperatorDelegator.sol#L265-L269) the _status will become _ENTERED, then this line will get executed:

```solidity
function completeQueuedWithdrawal(...) external nonReentrant onlyNativeEthRestakeAdmin {
  ...
  delegationManager.completeQueuedWithdrawal(withdrawal, tokens, middlewareTimesIndex, true);
  ...
}
```

EigenLayer will try to send the the native currency which will trigger `receive()` but _status will already equal _ENTERED which will result in a revert here:

```solidity
    function _nonReentrantBefore() private {
        // On the first call to nonReentrant, _status will be _NOT_ENTERED
@>      require(_status != _ENTERED, "ReentrancyGuard: reentrant call");

        // Any calls to nonReentrant after this point will fail
        _status = _ENTERED;
    }
```

## Tools Used
Manual Review

## Recommended Mitigation Steps
Remove the nonReentrant modifier from the receive function.

## Assessed type
DoS

# [HIGH-04] Withdrawing rebasing tokens can cause revert or loss of funds

## Submission Link
https://github.com/code-423n4/2024-04-renzo-validation/issues/873

## Disclaimer
We placed a medium severity for this finding but the judges decided that it is a duplicate of a High. The label in the repo was not changed for some reason.

# Lines of code
https://github.com/code-423n4/2024-04-renzo/blob/519e518f2d8dec9acf6482b84a181e403070d22d/contracts/Withdraw/WithdrawQueue.sol#L206 https://github.com/code-423n4/2024-04-renzo/blob/519e518f2d8dec9acf6482b84a181e403070d22d/contracts/Withdraw/WithdrawQueue.sol#L279

# Vulnerability details
## Impact
When rebasing tokens like stETH change their balance, this can cause reverts or to claim a different amount of stETH than the deserved one by the user at the time of calling [`claim()`](https://github.com/code-423n4/2024-04-renzo/blob/519e518f2d8dec9acf6482b84a181e403070d22d/contracts/Withdraw/WithdrawQueue.sol#L279).

## Proof of Concept
The stETH balance of the holder is based on the rewards or the slashing happening in the Lido protocol. There can be an increase or a decrease of the balance so we will explore both scenarios.

When user calls [`withdraw()`](https://github.com/code-423n4/2024-04-renzo/blob/519e518f2d8dec9acf6482b84a181e403070d22d/contracts/Withdraw/WithdrawQueue.sol#L206) he specifies the tokenOut (in our case stETH) that he would like to [`claim()`](https://github.com/code-423n4/2024-04-renzo/blob/519e518f2d8dec9acf6482b84a181e403070d22d/contracts/Withdraw/WithdrawQueue.sol#L279) after the cooldown period expires.

In [`withdraw()`](https://github.com/code-423n4/2024-04-renzo/blob/519e518f2d8dec9acf6482b84a181e403070d22d/contracts/Withdraw/WithdrawQueue.sol#L206) the tokenOut amount is recorded in the withdrawal struct and it also increases the `claimReserve[_assetOut]` with the same amount:

```solidity
        // add withdraw request for msg.sender
        withdrawRequests[msg.sender].push(
            WithdrawRequest(
                _assetOut,
                withdrawRequestNonce,
                amountToRedeem,
                _amount,
                block.timestamp
            )
        );

        // add redeem amount to claimReserve of claim asset
        claimReserve[_assetOut] += amountToRedeem;
```

Now lets explore both scenarios.

### Scenario 1 - increasing stETH balance
Bob calls [`withdraw()`](https://github.com/code-423n4/2024-04-renzo/blob/519e518f2d8dec9acf6482b84a181e403070d22d/contracts/Withdraw/WithdrawQueue.sol#L206) and burns enough ezETH to record in the protocol that he must claim 20 stETH

Alice does the same.

Currently we have 2 withdrawal orders for 40 stETH total `claimReserve[_assetOut]` currently is 40e18 Let's say the balance of WithdrawQueue is also 40 stETH

Rebasing happens and increases the balance of the contract with 10% (just for the sake of the example). The balance of WithdrawQueue is 44 stETH.

Instead of Bob and Alice claiming 22 stETH each they will get 20 stETH and 4 stETH will be left inside the protocol.

### Scenario 2 - decreasing stETH balance
Bob and Alice call [`withdraw()`](https://github.com/code-423n4/2024-04-renzo/blob/519e518f2d8dec9acf6482b84a181e403070d22d/contracts/Withdraw/WithdrawQueue.sol#L206) and burn enough ezETH to record in the protocol that each must get 20 stETH.

`claimReserve[_assetOut]` currently is 40e18 Let's say the balance of WithdrawQueue is also 40 stETH

Rebasing happens and decreases the balance of the contract with 10% The balance of WithdrawQueue is 36 stETH

Alice gets her 20 stETH but Bob cannot claim until more capital is deployed to withdrawQueue despite "locking" the funds that need to be claimed with `claimReserve[_assetOut]`.

## Tools Used
Manual Review

## Recommended Mitigation Steps
Possible solution is on [`withdraw()`](https://github.com/code-423n4/2024-04-renzo/blob/519e518f2d8dec9acf6482b84a181e403070d22d/contracts/Withdraw/WithdrawQueue.sol#L206) to convert stETH to wstETH which is non rebasing. This way users will get the amounts they deserve.

## Assessed type
Other

# [MEDIUM-01] Token deposits and withdraws below the buffer deficit will fail

## Submission Link
https://github.com/code-423n4/2024-04-renzo-findings/issues/182

## Impact
All deposit transactions with amounts that are less than the buffer deficit will revert. When deficits are high it's possible that most of the deposits will be DOSed.

Withdraw completions will also be prevented in case the amount left does not constitute a whole share.

## Vulnerability details
In the latest version of `RestakeManager` the `deposit()` function has been modified so that the buffer deficit inside `withdrawQueue` is checked and in case there is any, the deposit amount is used to fill that buffer:

https://github.com/code-423n4/2024-04-renzo/blob/519e518f2d8dec9acf6482b84a181e403070d22d/contracts/RestakeManager.sol#L546-L556

```solidity
function deposit(
        IERC20 _collateralToken,
        uint256 _amount,
        uint256 _referralId
    ) public nonReentrant notPaused {
       
        .....

        // Get the value of the collateral token being deposited
        uint256 collateralTokenValue = renzoOracle.lookupTokenValue(_collateralToken, _amount);

        .....

        // Check the withdraw buffer and fill if below buffer target
        uint256 bufferToFill = depositQueue.withdrawQueue().getBufferDeficit(
            address(_collateralToken)
        );

        if (bufferToFill > 0) {
            bufferToFill = (_amount <= bufferToFill) ? _amount : bufferToFill;
            // update amount to send to the operator Delegator
            _amount -= bufferToFill;

            ....

            // fill Withdraw Buffer via depositQueue
            depositQueue.fillERC20withdrawBuffer(address(_collateralToken), bufferToFill);
        }

        // Approve the tokens to the operator delegator
        _collateralToken.safeApprove(address(operatorDelegator), _amount);

        // Call deposit on the operator delegator
        operatorDelegator.deposit(_collateralToken, _amount);

        ....
    }
```

The amount that is left after the buffer has been filled is sent to the operator delegator to be deposited through the `StrategyManager` contract inside `EigenLayer`.

https://github.com/code-423n4/2024-04-renzo/blob/519e518f2d8dec9acf6482b84a181e403070d22d/contracts/Delegation/OperatorDelegator.sol#L143-L169

```solidity
 function _deposit(IERC20 _token, uint256 _tokenAmount) internal returns (uint256 shares) {
        // Approve the strategy manager to spend the tokens
        _token.safeApprove(address(strategyManager), _tokenAmount);

        // Deposit the tokens via the strategy manager
        return
            strategyManager.depositIntoStrategy(tokenStrategyMapping[_token], _token, _tokenAmount);
    }
```

The `strategyManager.depositIntoStrategy()` function internally calls the `deposit()` function of the respective strategy where the funds actually get stored:

https://github.com/Layr-Labs/eigenlayer-contracts/blob/15b679b08e7a3589ff83a5e84076ca4e0a00ec0b/src/contracts/core/StrategyManager.sol#L333

```solidity
function _depositIntoStrategy(
        address staker,
        IStrategy strategy,
        IERC20 token,
        uint256 amount
    ) internal onlyStrategiesWhitelistedForDeposit(strategy) returns (uint256 shares) {
        ....

        // deposit the assets into the specified strategy and get the equivalent amount of shares in that strategy
        shares = strategy.deposit(token, amount);

        ....

        return shares;
    }
```

`Strategy.deposit()` calculates the shares to mint.

https://github.com/Layr-Labs/eigenlayer-contracts/blob/dbfa12128a41341b936f3e8da5d6da58c6233877/src/contracts/strategies/StrategyBase.sol#L115-L118

```solidity
function deposit(
        IERC20 token,
        uint256 amount
    ) external virtual override onlyWhenNotPaused(PAUSED_DEPOSITS) onlyStrategyManager returns (uint256 newShares) {
        ....
        newShares = (amount * virtualShareAmount) / virtualPriorTokenBalance;

        // extra check for correctness / against edge case where share rate can be massively inflated as a 'griefing' sort of attack
        require(newShares != 0, "StrategyBase.deposit: newShares cannot be zero");

        ....

        return newShares;
    }
```

As you can see `Strategy.deposit()` reverts in case the `newShares` to mint is 0.

And this is exactly the problem that will block user deposits inside `StakeManager`. The issue arises from the fact that `StakeManager.deposit()` never checks before the call to `operatorDelegator.deposit()` if the `_amount` [that is left after the buffer deficit has been deducted is 0](https://github.com/code-423n4/2024-04-renzo/blob/519e518f2d8dec9acf6482b84a181e403070d22d/contracts/RestakeManager.sol#L548-L549):

```solidity
// update amount to send to the operator Delegator
   _amount -= bufferToFill;
```

If `_amount` is zero this means that the call chain `StakeManager.deposit()` -> `strategyManager.depositIntoStrategy()` -> `Strategy.deposit()` passes 0 as `amoun`t and finally [the shares calculation inside `Strategy.deposit()` equals to 0 which as explained above reverts the execution](https://github.com/Layr-Labs/eigenlayer-contracts/blob/dbfa12128a41341b936f3e8da5d6da58c6233877/src/contracts/strategies/StrategyBase.sol#L115C9-L118C83).

```solidity
newShares = (amount * virtualShareAmount) / virtualPriorTokenBalance;

// extra check for correctness / against edge case where share rate can be massively inflated as a 'griefing' sort of attack
require(newShares != 0, "StrategyBase.deposit: newShares cannot be zero");
```

Considering that the buffer deficits will probably be set quite high in order to cover the rising number of accounts that would want to withdraw, the chances are high that a lot of deposits won't be above that deficit, meaning no value will be left after that to send to EigenLayer, making all those deposits fail and leaving the buffer empty because of everything explained above

EigenPod withdraws through `EigenPod::completeQueuedWithdrawal()` [are also affected by the same problem](https://github.com/code-423n4/2024-04-renzo/blob/519e518f2d8dec9acf6482b84a181e403070d22d/contracts/Delegation/OperatorDelegator.sol#L307-L308). Although there is a 0 amount check here, still no validation is made if the amount left after the buffer has been filled is enough for minting of a whole share.

## Tools Used
Manual Review

## Recommended Mitigation Steps
Make sure to validate inside `RestakeManager.deposit()` that the amount of shares to be minted from the leftover amount are above 0 before calling `operatorDelegator.deposit()`:

```solidity
function deposit(
        IERC20 _collateralToken,
        uint256 _amount,
        uint256 _referralId
    ) public nonReentrant notPaused {

     ....

        if (bufferToFill > 0) {
            bufferToFill = (_amount <= bufferToFill) ? _amount : bufferToFill;
            // update amount to send to the operator Delegator
            _amount -= bufferToFill;

            ....
        }

+        uint256 sharesToMint = 
operatorDelegator.tokenStrategyMapping[_collateralToken].underlyingToSharesView(_amount );

+        if(sharesToMint > 0){

+            // Approve the tokens to the operator delegator
+            _collateralToken.safeApprove(address(operatorDelegator), _amount);

+            // Call deposit on the operator delegator
+            operatorDelegator.deposit(_collateralToken, _amount);
        }

        // Calculate how much ezETH to mint
        uint256 ezETHToMint = renzoOracle.calculateMintAmount(
            totalTVL,
            collateralTokenValue,
            ezETH.totalSupply()
        );

        .....
}

The same remediation applies for `OperatorDelegator::completeQueuedWithdrawal()`
```

## Assessed type
Dos

# [MEDIUM-02] WithdrawQueueAdmin is not able to pause withdrawals

## Submission Link
https://github.com/code-423n4/2024-04-renzo-findings/issues/123

# Lines of code
https://github.com/code-423n4/2024-04-renzo/blob/519e518f2d8dec9acf6482b84a181e403070d22d/contracts/Withdraw/WithdrawQueue.sol#L139-L141 https://github.com/code-423n4/2024-04-renzo/blob/519e518f2d8dec9acf6482b84a181e403070d22d/contracts/Withdraw/WithdrawQueue.sol#L147-L149 https://github.com/code-423n4/2024-04-renzo/blob/519e518f2d8dec9acf6482b84a181e403070d22d/contracts/Withdraw/WithdrawQueue.sol#L39-L42

# Vulnerability details
## Impact
According to the Renzo docs the withdraw functionality can be paused by an admin in case of emergencies but by doing so non of the withdraw functions get paused.

## Proof of Concept
WithdrawQueue.sol inherits OpenZeppelin's PausableUpgradeable contract and implements [`pause()`](https://github.com/code-423n4/2024-04-renzo/blob/519e518f2d8dec9acf6482b84a181e403070d22d/contracts/Withdraw/WithdrawQueue.sol#L139-L141) and [`unpause()`](https://github.com/code-423n4/2024-04-renzo/blob/519e518f2d8dec9acf6482b84a181e403070d22d/contracts/Withdraw/WithdrawQueue.sol#L147-L149) functions that can only be called the the [`WithdrawQueueAdmin`](https://github.com/code-423n4/2024-04-renzo/blob/519e518f2d8dec9acf6482b84a181e403070d22d/contracts/Withdraw/WithdrawQueue.sol#L39-L42)

Currently none of the functions in this contract have a check or a modifier connected with the pause state.

## Tools Used
Manual Review

## Recommended Mitigation Steps
Add whenNotPaused modifier to the withdraw functions

## Assessed type
Access Control

# [MEDIUM-03] Missing slippage check on deposit and withdraw functions

## Submission Link
https://github.com/code-423n4/2024-04-renzo-findings/issues/481

# Lines of code
https://github.com/code-423n4/2024-04-renzo/blob/519e518f2d8dec9acf6482b84a181e403070d22d/contracts/Withdraw/WithdrawQueue.sol#L206 https://github.com/code-423n4/2024-04-renzo/blob/519e518f2d8dec9acf6482b84a181e403070d22d/contracts/RestakeManager.sol#L491-L495

# Vulnerability details
## Impact
By not having a minOut amount for the minted ezETH tokens that the depositor will get his transaction can get executed on a different price suprising the user with minted amount of tokens.

## Proof of Concept
I can have tokenA that has a price of 1.02 ETH, I make a [`deposit()`](https://github.com/code-423n4/2024-04-renzo/blob/519e518f2d8dec9acf6482b84a181e403070d22d/contracts/RestakeManager.sol#L491-L495) but before my transaction gets executed the oracle price have changed to 0.98 ETH and I get minted less ezETH than I expected (4% less - which can be the profit for a year doing normal ETH stake).

## Tools Used
Manual Review

## Recommended Mitigation Steps
Introduce a minMintAmount or another function parameter in the deposit/withdraw functions in order to guarantee the user no surprises.

## Assessed type
Oracle

# [MEDIUM-04] Possibility of DOS in functions that use calculateTVLs() due to scalability issues

## Submission Link
https://github.com/code-423n4/2024-04-renzo-findings/issues/560

# Lines of code
https://github.com/code-423n4/2024-04-renzo/blob/519e518f2d8dec9acf6482b84a181e403070d22d/contracts/RestakeManager.sol#L274

# Vulnerability details
## Impact
Functions that use [`RestakeManager:calculateTVLs()`](https://github.com/code-423n4/2024-04-renzo/blob/519e518f2d8dec9acf6482b84a181e403070d22d/contracts/RestakeManager.sol#L274) can revert due to the increasing gas consumption with the scale of the protocol thus blocking main protocol functionalities.

## Proof of Concept
[`RestakeManager.sol:calculateTVLs()`](https://github.com/code-423n4/2024-04-renzo/blob/519e518f2d8dec9acf6482b84a181e403070d22d/contracts/RestakeManager.sol#L274) is currently used in:

1. [`RestakeManager.sol:depositETH()`](https://github.com/code-423n4/2024-04-renzo/blob/519e518f2d8dec9acf6482b84a181e403070d22d/contracts/RestakeManager.sol#L594)
2. [`RestakeManager.sol:deposit()`](https://github.com/code-423n4/2024-04-renzo/blob/519e518f2d8dec9acf6482b84a181e403070d22d/contracts/RestakeManager.sol#L473)
3. [`RestakeManager.sol:depositTokenRewardsFromProtocol()`](https://github.com/code-423n4/2024-04-renzo/blob/519e518f2d8dec9acf6482b84a181e403070d22d/contracts/RestakeManager.sol#L652)
4. [`xRenzoBridge.sol:sendPrice()`](https://github.com/code-423n4/2024-04-renzo/blob/519e518f2d8dec9acf6482b84a181e403070d22d/contracts/Bridge/L1/xRenzoBridge.sol#L215) -[`BalanceRateProvider.sol.getRate()`](https://github.com/code-423n4/2024-04-renzo/blob/519e518f2d8dec9acf6482b84a181e403070d22d/contracts/RateProvider/BalancerRateProvider.sol#L31)
5. [`WithdrawQueue.sol:withdraw()`](https://github.com/code-423n4/2024-04-renzo/blob/519e518f2d8dec9acf6482b84a181e403070d22d/contracts/Withdraw/WithdrawQueue.sol#L206)

[`RestakeManager.sol:calculateTVLs()`](https://github.com/code-423n4/2024-04-renzo/blob/519e518f2d8dec9acf6482b84a181e403070d22d/contracts/RestakeManager.sol#L274) loops for each operator through all accepted tokens.

Let's say we support 3 tokens and 5 operators = 15 iterations

On each iteration we make 1 call to EigenLayer + 1 call to an oracle.

On the first operator iteration we make additional calls for each token to see the tvl in the withdrawalQueue contract.

so far we have: withdrawal queue value = 3 tokens * 1 call = 3 oracle calls 3 tokens * 5 operators = 15 iterations * (1 oracle call + 1 EigenLayer call) = 30 calls In total 33 calls.

If we double the operators count and the tokens count the total calls will become: withdrawal queue value = 6 tokens * 1 call = 6 oracle calls 6 tokens * 10 operators = 60 iterations * (1 oracle call + 1 EigenLayer call) = 120 calls In total 126 calls.

With scale the gas consumption increases times 4.

At this rate [`RestakeManager:calculateTVLs()`](https://github.com/code-423n4/2024-04-renzo/blob/519e518f2d8dec9acf6482b84a181e403070d22d/contracts/RestakeManager.sol#L274) will become very expensive to call and can cause DOS due to reaching the block gas limit.

## Tools Used
Manual Review

## Recommended Mitigation Steps
This fix is a non trivial fix and it involves making significant design changes to the code.

What I can think of is moving such kind of logic off-chain by implementing a custom oracle in order to track TVLs of operators and total TVL of the protocol.

By taking the TVL value from the oracle we can use the same mint logic of ezETH (based on the inflation of value made by the deposit).

The other option I think of is to compare ezETH oracle price to the deposited token oracle price.

Further research should be done on the possible mitigation solutions.

Despite the decision of the protocol on this finding, gas optimisations could be performed to lower the consumed gas. Consider getting all token prices once from the oracles and then using them to determine the value of the totalTVL and of each operator, instead making the same oracle price calls for each operator.

## Assessed type
DoS

# [MEDIUM-05] Hardcoded heartbeat can cause the usage of stale prices

## Submission Link
https://github.com/code-423n4/2024-04-renzo-findings/issues/562

# Lines of code
https://github.com/code-423n4/2024-04-renzo/blob/519e518f2d8dec9acf6482b84a181e403070d22d/contracts/Oracle/RenzoOracle.sol#L71-L81 https://github.com/code-423n4/2024-04-renzo/blob/519e518f2d8dec9acf6482b84a181e403070d22d/contracts/Oracle/RenzoOracle.sol#L85-L98

# Vulnerability details
## Impact
Hardcoding heartbeat to 24h + 60 seconds can lead to using very stale prices.

## Proof of Concept
Oracle price feeds can have different heartbeat intervals for different tokens and heartbeats can change over time.

[`RenzoOracle.sol:lookupTokenValue()`](https://github.com/code-423n4/2024-04-renzo/blob/519e518f2d8dec9acf6482b84a181e403070d22d/contracts/Oracle/RenzoOracle.sol#L71-L81) and [`RenzoOracle.sol:lookupTokenAmountFromValue()`](https://github.com/code-423n4/2024-04-renzo/blob/519e518f2d8dec9acf6482b84a181e403070d22d/contracts/Oracle/RenzoOracle.sol#L85-L98) use [`MAX_TIME_WINDOW`](https://github.com/code-423n4/2024-04-renzo/blob/519e518f2d8dec9acf6482b84a181e403070d22d/contracts/Oracle/RenzoOracle.sol#L26)

```solidity
/// @dev The maxmimum staleness allowed for a price feed from chainlink
uint256 constant MAX_TIME_WINDOW = 86400 + 60; // 24 hours + 60 seconds
```

```solidity
    function lookupTokenValue(IERC20 _token, uint256 _balance) public view returns (uint256) {
        AggregatorV3Interface oracle = tokenOracleLookup[_token];
        if (address(oracle) == address(0x0)) revert OracleNotFound();

        (, int256 price, , uint256 timestamp, ) = oracle.latestRoundData();
@>      if (timestamp < block.timestamp - MAX_TIME_WINDOW) revert OraclePriceExpired();
        if (price <= 0) revert InvalidOraclePrice();

        // Price is times 10**18 ensure value amount is scaled
        return (uint256(price) * _balance) / SCALE_FACTOR;
    }
```

If a token has a heartbeat of 1 hour then the price that we get from the oracle can be stale up to 23 hours which can cause significant economical damage to the protocol if the price becomes stale during a period of volatility

## Tools Used
Manual Review

## Recommended Mitigation Steps
Use the heartbeat of the specific oracle and have a way to update it if needed in the future.

## Assessed type
Oracle

