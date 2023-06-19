看v4-periphery里面的GeomeanOracle.sol
```solidity
 function getHooksCalls() public pure override returns (Hooks.Calls memory) {
        return Hooks.Calls({
            beforeInitialize: true,
            afterInitialize: true,
            beforeModifyPosition: true,
            afterModifyPosition: false,
            beforeSwap: true,
            afterSwap: false,
            beforeDonate: false,
            afterDonate: false
        });
    }
```
这个hook实现了四个函数
```solidity
 function beforeInitialize(address, IPoolManager.PoolKey calldata key, uint160)
        external
        view
        override
        poolManagerOnly
        returns (bytes4)
    {
        // This is to limit the fragmentation of pools using this oracle hook. In other words,
        // there may only be one pool per pair of tokens that use this hook. The tick spacing is set to the maximum
        // because we only allow max range liquidity in this pool.
        if (key.fee != 0 || key.tickSpacing != poolManager.MAX_TICK_SPACING()) revert OnlyOneOraclePoolAllowed();
        return GeomeanOracle.beforeInitialize.selector;
    }
```
看这个注释，是限制，只有tickSpace是最大值，才能使用这个hook

```solidity
 function afterInitialize(address, IPoolManager.PoolKey calldata key, uint160, int24)
        external
        override
        poolManagerOnly
        returns (bytes4)
    {
        bytes32 id = key.toId();
        (states[id].cardinality, states[id].cardinalityNext) = observations[id].initialize(_blockTimestamp());
        return GeomeanOracle.afterInitialize.selector;
    }

/// @notice Initialize the oracle array by writing the first slot. Called once for the lifecycle of the observations array
/// @param self The stored oracle array
/// @param time The time of the oracle initialization, via block.timestamp truncated to uint32
/// @return cardinality The number of populated elements in the oracle array
/// @return cardinalityNext The new length of the oracle array, independent of population
    function initialize(Observation[65535] storage self, uint32 time)
    internal
    returns (uint16 cardinality, uint16 cardinalityNext)
    {
        self[0] = Observation({
        blockTimestamp: time,
        tickCumulative: 0,
        secondsPerLiquidityCumulativeX128: 0,
        initialized: true
        });
        return (1, 1);
    }
```
这个比较清晰，当pool初始化完成后，初始化第一个Obervation。
```solidity
 function beforeModifyPosition(
    address,
    IPoolManager.PoolKey calldata key,
    IPoolManager.ModifyPositionParams calldata params
) external override poolManagerOnly returns (bytes4) {
    if (params.liquidityDelta < 0) revert OraclePoolMustLockLiquidity();
    int24 maxTickSpacing = poolManager.MAX_TICK_SPACING();
    if (
        params.tickLower != TickMath.minUsableTick(maxTickSpacing)
        || params.tickUpper != TickMath.maxUsableTick(maxTickSpacing)
    ) revert OraclePositionsMustBeFullRange();
    _updatePool(key);
    return GeomeanOracle.beforeModifyPosition.selector;
}

    function beforeSwap(address, IPoolManager.PoolKey calldata key, IPoolManager.SwapParams calldata)
    external
    override
    poolManagerOnly
    returns (bytes4)
    {
        _updatePool(key);
        return GeomeanOracle.beforeSwap.selector;
    }
 
```
beforeModifyPosition和beforeSwap都调用了_updatePool。
```solidity
  /// @dev Called before any action that potentially modifies pool price or liquidity, such as swap or modify position
    function _updatePool(IPoolManager.PoolKey calldata key) private {
        bytes32 id = key.toId();
        (, int24 tick,) = poolManager.getSlot0(id);

        uint128 liquidity = poolManager.getLiquidity(id);

        (states[id].index, states[id].cardinality) = observations[id].write(
            states[id].index, _blockTimestamp(), tick, liquidity, states[id].cardinality, states[id].cardinalityNext
        );
    }
/// @notice Writes an oracle observation to the array
/// @dev Writable at most once per block. Index represents the most recently written element. cardinality and index must be tracked externally.
/// If the index is at the end of the allowable array length (according to cardinality), and the next cardinality
/// is greater than the current one, cardinality may be increased. This restriction is created to preserve ordering.
/// @param self The stored oracle array
/// @param index The index of the observation that was most recently written to the observations array
/// @param blockTimestamp The timestamp of the new observation
/// @param tick The active tick at the time of the new observation
/// @param liquidity The total in-range liquidity at the time of the new observation
/// @param cardinality The number of populated elements in the oracle array
/// @param cardinalityNext The new length of the oracle array, independent of population
/// @return indexUpdated The new index of the most recently written element in the oracle array
/// @return cardinalityUpdated The new cardinality of the oracle array
    function write(
        Observation[65535] storage self,
        uint16 index,
        uint32 blockTimestamp,
        int24 tick,
        uint128 liquidity,
        uint16 cardinality,
        uint16 cardinalityNext
    ) internal returns (uint16 indexUpdated, uint16 cardinalityUpdated) {
    unchecked {
        Observation memory last = self[index];

        // early return if we've already written an observation this block
        if (last.blockTimestamp == blockTimestamp) return (index, cardinality);

        // if the conditions are right, we can bump the cardinality
        if (cardinalityNext > cardinality && index == (cardinality - 1)) {
            cardinalityUpdated = cardinalityNext;
        } else {
            cardinalityUpdated = cardinality;
        }

        indexUpdated = (index + 1) % cardinalityUpdated;
        self[indexUpdated] = transform(last, blockTimestamp, tick, liquidity);
    }
    }
```
这个write里面我没太看懂，为什么要bump