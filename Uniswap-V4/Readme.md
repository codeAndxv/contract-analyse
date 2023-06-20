swap 
Pool.sol    从374col开始
if (params.amountSpecified == 0) revert SwapAmountCannotBeZero();
如果指定要买的数量为0，直接报交换的数量不能为0的错误
```solidity
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
```

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
如果是从token0交换到token1， 就是卖出token0，买入token1，会使token如果限制价格大于当前的价格，直接
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


看不懂的是这个变量  
```
/// @inheritdoc IPoolManager
address[] public override lockedBy;
/// @member nonzeroDeltaCount The number of entries in the currencyDelta mapping that have a non-zero value
/// @member currencyDelta The amount owed to the locker (positive) or owed to the pool (negative) of the currency
struct LockState {
    uint256 nonzeroDeltaCount;
    mapping(Currency currency => int256) currencyDelta;
}

/// @dev Represents the state of the locker at the given index. Each locker must have net 0 currencies owed before
/// releasing their lock. Note this is private because the nested mappings cannot be exposed as a public variable.
mapping(uint256 index => LockState) private lockStates;
```
看懂了这个变量，这个变量相当于状态栈，每次在进入lock方法时，把调用用户存入栈中，当退出时，检查金额是否正确，保证该用户的账户没有从pool里面拿走coin
```solidity
  /// @inheritdoc IPoolManager
    function lock(bytes calldata data) external override returns (bytes memory result) {
        uint256 id = lockedBy.length;
        lockedBy.push(msg.sender);

        // the caller does everything in this callback, including paying what they owe via calls to settle
        result = ILockCallback(msg.sender).lockAcquired(id, data);

        unchecked {
            LockState storage lockState = lockStates[id];
            if (lockState.nonzeroDeltaCount != 0) revert CurrencyNotSettled();
        }

        lockedBy.pop();
    }
```

TickMath.sol
这个是跟价格有关系的。V4版本从之前的k=xy变成了tick形式。
```
/// @dev The minimum tick that may be passed to #getSqrtRatioAtTick computed from log base 1.0001 of 2**-128
int24 internal constant MIN_TICK = -887272;
/// @dev The maximum tick that may be passed to #getSqrtRatioAtTick computed from log base 1.0001 of 2**128
int24 internal constant MAX_TICK = -MIN_TICK;
这里把价格分为从2^-128到2^128，并使用log1.0001。就是以0.01%作为一个台阶。
/// @notice Calculates sqrt(1.0001^tick) * 2^96
/// @dev Throws if |tick| > max tick
/// @param tick The input tick for the above formula
/// @return sqrtPriceX96 A Fixed point Q64.96 number representing the sqrt of the ratio of the two assets (currency1/currency0)
/// at the given tick
function getSqrtRatioAtTick(int24 tick) internal pure returns (uint160 sqrtPriceX96) 

```
实现太复杂，只看注释。就是把tick做1.0001^,相当于还原了2^x。然后做根号，我不理解为什么要根号，相当于把价格范围变为了2^-64~2^64。后面乘以2^96也好理解，2^-x都是小数。

不太明白为什么要用地址来验证，就像下面这样
```solidity
   function shouldCallBeforeSwap(IHooks self) internal pure returns (bool) {
        return uint256(uint160(address(self))) & BEFORE_SWAP_FLAG != 0;
    }
```
难道还要使用工具来生成特别的地址吗，如何使用多个hook函数，那么生成的地址也要计算好久
