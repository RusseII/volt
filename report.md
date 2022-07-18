# Always Be Growing

# Scope 
https://github.com/volt-protocol/volt-protocol-core/pull/82


# Introduction

Volt was previously an inflation-pegged stablecoin. In the PR under review, they are moving from being pegged by a CPI oracle to being pegged by a VoltSystemOracle. The VoltSystemOracle is configured with a starting price, a start period of when interest should start accruing, and the monthly price change denominated in basis points. Once the VoltSystemOracle has been deployed these values can not be changed. For any changes of these values to happen a new oracle would need deployed and Volt protocol would need configured to use the new oracle.   

I spent a total of 10 hours reviewing the changes split across two days. I have reviewed Volt protocol in the past so I already had a good understanding of the system. 

In this review, I spent a majority of my time reviewing `VoltSystemOracle.sol` & `VoltSystemOracle.t.sol`. I also did a less comprehensive review on the deployment scripts and integration tests. 

*Disclaimer:* This security review does not guarantee against a hack. It is a snapshot in time of brink according to the specific commit by a one person team. Any modifications to the code will require a new security review.

Summary:
No issues found that were not already called out and mitigated in VIP-2.md


# Methodology 

I spent a majority of my time reviewing `VoltSystemOracle.sol` & `VoltSystemOracle.t.sol`. Below is a non-exhaustive list of what was tested and explored.

1. High level protocol overview and line by line code review.
I looked through each of the files updated in this PR to get a full understanding of the changes and to decide where I would spend the majority of my time.

2. Manual verification of the `getCurrentOraclePrice()` and `compoundInterest()` functions. I wanted to ensure that these functions were fully and correctly tested.

Some specific things I looked into: 
* Are the `_calculateDelta` and `_calculateLinearInterpolation` helpers in `VoltSystemOracle.t.sol` correct? 
* Does `getCurrentOraclePrice()` work correctly within a specific period?
* Does `getCurrentOraclePrice()` work correctly across periods?
* Any situations where `getCurrentOraclePrice()` could revert? 
* Does the value returned by`getCurrentOraclePrice()` work correctly if there are multiple time periods between each `compoundInterest()` calls? 
* How long into the future can this oracle be depended on?

3. I explored the impact of what would happen if `compoundInterest()` did not get called for multiple days.
This situation had already been called out and explored by the core team and they will be using a keeper to prevent long periods of time without a `compoundInterest()` call.  As outlined by the team, calling this method in a timely fashion is important to prevent a possible price arbitrage between before the `compoundInterest()` call and after the call. Even in the case of a black swan event where `compoundInterest()` does go multiple days without being called, the arbitrage opportunity is minimal due to the mint/redeem limits & fees. 

I ran some rough numbers on what the impact would be if there was an issue with the keeper and `compoundInterest()` is not called for 24h. Exploiting this on Arbitrum would not make sense unless there was a much longer period because there is a 5 basis fee on both mint and redeem that would be larger than any potential profit. For mainnet USDC PSM it there's currently a 0 basis fee on redeem and will soon be a 0 basis fee on mint, making this theoretically exploitable. There is a rate limit on how much volt can be minted/redeemed that significantly reduces the possible profit from this. 

In the current configuration, the max amount that can be minted at once is 10m, and if flash loaning 10m (dydx w/ 0% fee) to atomically arb this at 20 monthly basis points and 1 day of delay to `compoundInterest()` then possible arb profit would be ~$600.

Might be worth considering to keep a very small fee on mint or redeem if ever increasing the mint cap in the future

# Recommendations:
* Magic strings (addresses) are used a few places in the code. I recommend naming all addresses that are being used in the code. This would help reviewers and devs to understand the significance of each address. This would also help prevent against regression issues in the future where an address is changed only in one place but not in others. 
* I recommend exploring simplifying and clarifying the deployment logic and integration tests. While the current logic appears to be sound and well tested there seems to be some complexity which increases the risk of an error in future deployments. 
* Documentation and namings can be improved. Compounding interval was changed from 1y to 1mo but namings and descriptions have not yet been updated. 
* Consider having a very small bips fee for minting/redeeming volt to negate any potential profit from flash mint/redeems. This is to remove/reduce any possible profit from #3 outlined above.   
