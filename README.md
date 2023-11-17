# IPriceOracle

`IPriceOracle` is a minimal interface for querying for on-chain prices.

Internally, price oracle contracts can resolve pricing requests in any way they like. Oracles should of course refrain from using price feeds that can be manipulated by attackers, which in practice means centralised oracles like Chainlink, TWAPs, [medians](https://github.com/euler-xyz/median-oracle), and variations thereof.

The interface does not expose any policies or configuration parameters to the users. Instead, those are handled internally by the implementation contract. If parameters need to be modified, this should be handled by making the oracle contract controllable by governance or some other mechanism. If multiple simultaneous configurations are desired, separate price oracle contracts should be used.


## Currencies

In forex, prices are specified for pairs of currencies called base and quote, like this: `BASE/QUOTE`. The price represents the amount of the quote currency that it takes to buy 1 unit of the base currency. For example, if the price for `EUR/USD` is 1.1, then it will take 1.1 USD to buy 1 EUR.

In the `IPriceOracle` methods, token addresses for both `base` and `quote` should be passed in. This makes it unambiguous what the returned values represent, and allow oracles to implement [cross pricing](#crosses) logic.

### ISO codes

Users can request prices based on fiat currencies instead of token addresses by casting numeric [ISO 4217](https://en.wikipedia.org/wiki/ISO_4217) currency codes into addresses. For example, USD has the ISO 4217 code of 840, so `address(840)` which corresponds to `0x0000000000000000000000000000000000000348` can be used instead.

For pricing purposes, values denominated in these special fiat addresses are assumed to have 18 decimals.


## Crosses

Pricing oracles can optionally implement cross-pricing logic. This allows a user to request pricing for a token pair that doesn't directly have a pricing feed available.

As a hypothetical example, if a user requests a price for `DAI/EUR`, the price oracle could retrieve pricing for `DAI/USDC` from a Uniswap TWAP, and then another price for the `USDC/USD` from an internal pegged price, and then finally another price for `EUR/USD` from Chainlink. The price oracle essentially multiplies these prices together, inverting when necessary.

Oracles should support inverting pairs, so that if `EUR/USD` is supported, then `USD/EUR` should also be supported. Note that when inverting, [bid and ask](#spreads) values should be swapped. In more complicated crosses, care should be taken to use the correct bid or ask value at each leg of the cross.


## Amounts

The `getQuote` and `getQuotes` methods take an `inAmount` parameter. This has two purposes:

* Reduces the impact of [precision loss](#precision-loss)
* Optionally allows a price oracle to widen the spread between bid and ask values as successively larger amounts are requested

### Precision Loss

Consider the case where a user requests a quote for a pair such as SHIB/USDC. This represents the worst-case for precision loss because each unit of SHIB has a very small value, and because USDC's amount representation has a small number of decimals (6).

If, without precision loss, the price for SHIB/USDC should be `0.000008936`, then converting 1 unit of SHIB into USDC would yield `0.000008` (rounding down) or `0.000009` (rounding up). If this conversion was used as a price, then it would be significantly incorrect and furthermore would not reflect many dramatic price changes.

To solve this, a larger amount of the base asset should be converted. For example, if using the one-sided pricing methods, `1e12` SHIB may be requested with the `inAmount` parameter, which will effectively treat USDC as an 18 decimal place token.

If using two-sided pricing methods, in many cases there will be a known amount (ie, the size of a user's collateral or liability), and this amount can be used directly as `inAmount` to leverage the price oracle's [rounding](#rounding) and [spread](#spreads) behaviours.

### Spreads

For prices to make sense logically, there normally needs to be two values: a bid and an ask. The bid is always lower than the ask. If these two values were equivalent, then trading activity would occur until a [market clearing](https://en.wikipedia.org/wiki/Market_clearing) condition is reached, leaving a non-zero gap between the bid and ask.

The `getQuotes` function supports returning separate bid and ask values, which are referred to as two-sided prices. The meaning of the gap between these two values is defined by the oracle implementation, and could be a combination of any of the following:

* Actual market price quotes (ideally averaged over time using TWAPs etc)
* Confidence level estimations of pricing sources
* Difference between mean and median
* Simply hard-coded to 0

In the last case, the oracle is simply ignoring bid-ask spreads and assuming the spread is 0 for the purposes of pricing.

For oracles that ignore bid-ask spreads, the bid and ask values can just be copies of what is returned from the `getQuote` method.

For oracles that *do* implement bid-ask spreads, `getQuote`'s result should be derived from the bid-ask values by taking their *mid-point*. This is simply an average between the bid and ask values. If possible, this should be a geometric average of the underlying prices. However, in most cases taking an arithmetic average between the two output amounts will be acceptably accurate.


### Rounding

When implementing two-sided prices, the price oracle must ensure that amounts are rounded away from the [mid-point](#spreads). In other words, bid amounts are to be rounded down, and ask amounts rounded up. This pessimises the value that would be received through a conversion, and can be relied upon by oracle users.


## Interface

    interface IPriceOracle {
        /// @notice General description of this oracle.
        function name() external view returns (string memory);

        /// @notice Describes the pricing configuration currently used for computing quotes for this pair.
        function describe(address base, address quote) external view returns (string memory);

        /// @notice One-sided price: How much quote token you would get for inAmount of base token, assuming no price spread
        function getQuote(uint inAmount, address base, address quote) external view returns (uint outAmount);

        /// @notice Two-sided price: How much quote token you would get/spend for selling/buying inAmount of base token
        function getQuotes(uint inAmount, address base, address quote) external view returns (uint bidOutAmount, uint askOutAmount);

        error PO_BaseUnsupported();
        error PO_QuoteUnsupported();
        error PO_Overflow();
        error PO_NoPath();

        address internal constant PO_USD = address(840);
        address internal constant PO_EUR = address(978);
        address internal constant PO_JPY = address(392);
        address internal constant PO_GBP = address(826);
        address internal constant PO_CHF = address(756);
        address internal constant PO_CAD = address(124);
        address internal constant PO_AUD = address(36);
        address internal constant PO_NZD = address(554);
        address internal constant PO_CNY = address(156);
        address internal constant PO_XAU = address(959);
        address internal constant PO_XAG = address(961);
    }
