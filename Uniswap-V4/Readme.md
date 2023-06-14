swap 
Pool.sol    从374col开始
if (params.amountSpecified == 0) revert SwapAmountCannotBeZero();
如果指定要买的数量为0，直接报交换的数量不能为0的错误

Slot0 memory slot0Start = self.slot0;
struct Slot0 {
    // the current price
    uint160 sqrtPriceX96;
    // the current tick
    int24 tick;
    uint8 protocolSwapFee;
    uint8 protocolWithdrawFee;
    uint8 hookSwapFee;
    uint8 hookWithdrawFee;
}

if (self.slot0.sqrtPriceX96 == 0) revert PoolNotInitialized();
如果价格为0，说明池子还没有被初始化  报错
```
if (params.zeroForOne) {
    if (params.sqrtPriceLimitX96 >= slot0Start.sqrtPriceX96) {
        revert PriceLimitAlreadyExceeded(slot0Start.sqrtPriceX96, params.sqrtPriceLimitX96);
    }
    if (params.sqrtPriceLimitX96 <= TickMath.MIN_SQRT_RATIO) {
        revert PriceLimitOutOfBounds(params.sqrtPriceLimitX96);
    }
} else {
    if (params.sqrtPriceLimitX96 <= slot0Start.sqrtPriceX96) {
        revert PriceLimitAlreadyExceeded(slot0Start.sqrtPriceX96, params.sqrtPriceLimitX96);
    }
    if (params.sqrtPriceLimitX96 >= TickMath.MAX_SQRT_RATIO) {
        revert PriceLimitOutOfBounds(params.sqrtPriceLimitX96);
    }
}
```
如果是从token0交换到token1， 如果价格大于当前的价格，直接
```
SwapCache memory cache = SwapCache({
    liquidityStart: self.liquidity,
    protocolFee: params.zeroForOne ? (slot0Start.protocolSwapFee % 16) : (slot0Start.protocolSwapFee >> 4),
    hookFee: params.zeroForOne ? (slot0Start.hookSwapFee % 16) : (slot0Start.hookSwapFee >> 4)
});
```
For swap fees, the upper 4 bits are the fee for trading 1 for 0, and the lower 4 are for 0 for 1 and are taken as a percentage of the lp swap fee.
```
      (step.tickNext, step.initialized) =
                self.tickBitmap.nextInitializedTickWithinOneWord(state.tick, params.tickSpacing, params.zeroForOne);
```