该啃的总要啃，最重要的swap
[PoolSwapTest.sol](https://github.com/Uniswap/v4-core/blob/main/contracts/test/PoolSwapTest.sol)

swap 方法
```solidity
 delta =
            abi.decode(manager.lock(abi.encode(CallbackData(msg.sender, testSettings, key, params))), (BalanceDelta));
```
调用poolManager的lock方法。
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
压一个状态进lockedBy。

然后调用lockAcquired
```solidity
function lockAcquired(uint256, bytes calldata rawData) external returns (bytes memory) {
        require(msg.sender == address(manager));

        CallbackData memory data = abi.decode(rawData, (CallbackData));

        BalanceDelta delta = manager.swap(data.key, data.params);

        if (data.params.zeroForOne) {
            if (delta.amount0() > 0) {
                if (data.testSettings.settleUsingTransfer) {
                    if (data.key.currency0.isNative()) {
                        manager.settle{value: uint128(delta.amount0())}(data.key.currency0);
                    } else {
                        IERC20Minimal(Currency.unwrap(data.key.currency0)).transferFrom(
                            data.sender, address(manager), uint128(delta.amount0())
                        );
                        manager.settle(data.key.currency0);
                    }
                } else {
                    // the received hook on this transfer will burn the tokens
                    manager.safeTransferFrom(
                        data.sender,
                        address(manager),
                        uint256(uint160(Currency.unwrap(data.key.currency0))),
                        uint128(delta.amount0()),
                        ""
                    );
                }
            }
            if (delta.amount1() < 0) {
                if (data.testSettings.withdrawTokens) {
                    manager.take(data.key.currency1, data.sender, uint128(-delta.amount1()));
                } else {
                    manager.mint(data.key.currency1, data.sender, uint128(-delta.amount1()));
                }
            }
        } else {
            if (delta.amount1() > 0) {
                if (data.testSettings.settleUsingTransfer) {
                    if (data.key.currency1.isNative()) {
                        manager.settle{value: uint128(delta.amount1())}(data.key.currency1);
                    } else {
                        IERC20Minimal(Currency.unwrap(data.key.currency1)).transferFrom(
                            data.sender, address(manager), uint128(delta.amount1())
                        );
                        manager.settle(data.key.currency1);
                    }
                } else {
                    // the received hook on this transfer will burn the tokens
                    manager.safeTransferFrom(
                        data.sender,
                        address(manager),
                        uint256(uint160(Currency.unwrap(data.key.currency1))),
                        uint128(delta.amount1()),
                        ""
                    );
                }
            }
            if (delta.amount0() < 0) {
                if (data.testSettings.withdrawTokens) {
                    manager.take(data.key.currency0, data.sender, uint128(-delta.amount0()));
                } else {
                    manager.mint(data.key.currency0, data.sender, uint128(-delta.amount0()));
                }
            }
        }

        return abi.encode(delta);
    }
```
从这个方法从上向下看，进入了
```solidity
    BalanceDelta delta = manager.swap(data.key, data.params);
```
```solidity
 /// @inheritdoc IPoolManager
    function swap(PoolKey memory key, IPoolManager.SwapParams memory params)
        external
        override
        noDelegateCall
        onlyByLocker
        returns (BalanceDelta delta)
    {
        if (key.hooks.shouldCallBeforeSwap()) {
            if (key.hooks.beforeSwap(msg.sender, key, params) != IHooks.beforeSwap.selector) {
                revert Hooks.InvalidHookResponse();
            }
        }

        // Set the total swap fee, either through the hook or as the static fee set an initialization.
        uint24 totalSwapFee;
        if (key.fee.isDynamicFee()) {
            totalSwapFee = IDynamicFeeManager(address(key.hooks)).getFee(key);
            if (totalSwapFee >= 1000000) revert FeeTooLarge();
        } else {
            // clear the top 4 bits since they may be flagged for hook fees
            totalSwapFee = key.fee & Fees.STATIC_FEE_MASK;
        }

        uint256 feeForProtocol;
        uint256 feeForHook;
        Pool.SwapState memory state;
        PoolId id = key.toId();
        (delta, feeForProtocol, feeForHook, state) = pools[id].swap(
            Pool.SwapParams({
                fee: totalSwapFee,
                tickSpacing: key.tickSpacing,
                zeroForOne: params.zeroForOne,
                amountSpecified: params.amountSpecified,
                sqrtPriceLimitX96: params.sqrtPriceLimitX96
            })
        );

        _accountPoolBalanceDelta(key, delta);
        // the fee is on the input currency

        unchecked {
            if (feeForProtocol > 0) {
                protocolFeesAccrued[params.zeroForOne ? key.currency0 : key.currency1] += feeForProtocol;
            }
            if (feeForHook > 0) {
                hookFeesAccrued[address(key.hooks)][params.zeroForOne ? key.currency0 : key.currency1] += feeForHook;
            }
        }

        if (key.hooks.shouldCallAfterSwap()) {
            if (key.hooks.afterSwap(msg.sender, key, params, delta) != IHooks.afterSwap.selector) {
                revert Hooks.InvalidHookResponse();
            }
        }

        emit Swap(
            id,
            msg.sender,
            delta.amount0(),
            delta.amount1(),
            state.sqrtPriceX96,
            state.liquidity,
            state.tick,
            totalSwapFee
        );
    }
```
```solidity
 if (key.hooks.shouldCallBeforeSwap()) {
            if (key.hooks.beforeSwap(msg.sender, key, params) != IHooks.beforeSwap.selector) {
                revert Hooks.InvalidHookResponse();
            }
        }

        // Set the total swap fee, either through the hook or as the static fee set an initialization.
        uint24 totalSwapFee;
        if (key.fee.isDynamicFee()) {
            totalSwapFee = IDynamicFeeManager(address(key.hooks)).getFee(key);
            if (totalSwapFee >= 1000000) revert FeeTooLarge();
        } else {
            // clear the top 4 bits since they may be flagged for hook fees
            totalSwapFee = key.fee & Fees.STATIC_FEE_MASK;
        }
```
上面的代码比较简单，检查是否要调用shouldCallBeforeSwap，设置fee。如果fee是动态的，就从DynamicFeeManager里面获取fee

```solidity
 (delta, feeForProtocol, feeForHook, state) = pools[id].swap(
            Pool.SwapParams({
                fee: totalSwapFee,
                tickSpacing: key.tickSpacing,
                zeroForOne: params.zeroForOne,
                amountSpecified: params.amountSpecified,
                sqrtPriceLimitX96: params.sqrtPriceLimitX96
            })
        );
```
```solidity
   /// @dev Executes a swap against the state, and returns the amount deltas of the pool
    function swap(State storage self, SwapParams memory params)
        internal
        returns (BalanceDelta result, uint256 feeForProtocol, uint256 feeForHook, SwapState memory state)
    {
        if (params.amountSpecified == 0) revert SwapAmountCannotBeZero();

        Slot0 memory slot0Start = self.slot0;
        if (self.slot0.sqrtPriceX96 == 0) revert PoolNotInitialized();
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

        SwapCache memory cache = SwapCache({
            liquidityStart: self.liquidity,
            protocolFee: params.zeroForOne ? (slot0Start.protocolSwapFee % 16) : (slot0Start.protocolSwapFee >> 4),
            hookFee: params.zeroForOne ? (slot0Start.hookSwapFee % 16) : (slot0Start.hookSwapFee >> 4)
        });

        bool exactInput = params.amountSpecified > 0;

        state = SwapState({
            amountSpecifiedRemaining: params.amountSpecified,
            amountCalculated: 0,
            sqrtPriceX96: slot0Start.sqrtPriceX96,
            tick: slot0Start.tick,
            feeGrowthGlobalX128: params.zeroForOne ? self.feeGrowthGlobal0X128 : self.feeGrowthGlobal1X128,
            liquidity: cache.liquidityStart
        });

        // continue swapping as long as we haven't used the entire input/output and haven't reached the price limit
        while (state.amountSpecifiedRemaining != 0 && state.sqrtPriceX96 != params.sqrtPriceLimitX96) {
            StepComputations memory step;

            step.sqrtPriceStartX96 = state.sqrtPriceX96;

            (step.tickNext, step.initialized) =
                self.tickBitmap.nextInitializedTickWithinOneWord(state.tick, params.tickSpacing, params.zeroForOne);

            // ensure that we do not overshoot the min/max tick, as the tick bitmap is not aware of these bounds
            if (step.tickNext < TickMath.MIN_TICK) {
                step.tickNext = TickMath.MIN_TICK;
            } else if (step.tickNext > TickMath.MAX_TICK) {
                step.tickNext = TickMath.MAX_TICK;
            }

            // get the price for the next tick
            step.sqrtPriceNextX96 = TickMath.getSqrtRatioAtTick(step.tickNext);

            // compute values to swap to the target tick, price limit, or point where input/output amount is exhausted
            (state.sqrtPriceX96, step.amountIn, step.amountOut, step.feeAmount) = SwapMath.computeSwapStep(
                state.sqrtPriceX96,
                (
                    params.zeroForOne
                        ? step.sqrtPriceNextX96 < params.sqrtPriceLimitX96
                        : step.sqrtPriceNextX96 > params.sqrtPriceLimitX96
                ) ? params.sqrtPriceLimitX96 : step.sqrtPriceNextX96,
                state.liquidity,
                state.amountSpecifiedRemaining,
                params.fee
            );

            if (exactInput) {
                // safe because we test that amountSpecified > amountIn + feeAmount in SwapMath
                unchecked {
                    state.amountSpecifiedRemaining -= (step.amountIn + step.feeAmount).toInt256();
                }
                state.amountCalculated = state.amountCalculated - step.amountOut.toInt256();
            } else {
                unchecked {
                    state.amountSpecifiedRemaining += step.amountOut.toInt256();
                }
                state.amountCalculated = state.amountCalculated + (step.amountIn + step.feeAmount).toInt256();
            }

            // if the protocol fee is on, calculate how much is owed, decrement feeAmount, and increment protocolFee
            if (cache.protocolFee > 0) {
                // A: calculate the amount of the fee that should go to the protocol
                uint256 delta = step.feeAmount / cache.protocolFee;
                // A: subtract it from the regular fee and add it to the protocol fee
                unchecked {
                    step.feeAmount -= delta;
                    feeForProtocol += delta;
                }
            }

            if (cache.hookFee > 0) {
                // step.feeAmount has already been updated to account for the protocol fee
                uint256 delta = step.feeAmount / cache.hookFee;
                unchecked {
                    step.feeAmount -= delta;
                    feeForHook += delta;
                }
            }

            // update global fee tracker
            if (state.liquidity > 0) {
                unchecked {
                    state.feeGrowthGlobalX128 += FullMath.mulDiv(step.feeAmount, FixedPoint128.Q128, state.liquidity);
                }
            }

            // shift tick if we reached the next price
            if (state.sqrtPriceX96 == step.sqrtPriceNextX96) {
                // if the tick is initialized, run the tick transition
                if (step.initialized) {
                    int128 liquidityNet = Pool.crossTick(
                        self,
                        step.tickNext,
                        (params.zeroForOne ? state.feeGrowthGlobalX128 : self.feeGrowthGlobal0X128),
                        (params.zeroForOne ? self.feeGrowthGlobal1X128 : state.feeGrowthGlobalX128)
                    );
                    // if we're moving leftward, we interpret liquidityNet as the opposite sign
                    // safe because liquidityNet cannot be type(int128).min
                    unchecked {
                        if (params.zeroForOne) liquidityNet = -liquidityNet;
                    }

                    state.liquidity = liquidityNet < 0
                        ? state.liquidity - uint128(-liquidityNet)
                        : state.liquidity + uint128(liquidityNet);
                }

                unchecked {
                    state.tick = params.zeroForOne ? step.tickNext - 1 : step.tickNext;
                }
            } else if (state.sqrtPriceX96 != step.sqrtPriceStartX96) {
                // recompute unless we're on a lower tick boundary (i.e. already transitioned ticks), and haven't moved
                state.tick = TickMath.getTickAtSqrtRatio(state.sqrtPriceX96);
            }
        }

        (self.slot0.sqrtPriceX96, self.slot0.tick) = (state.sqrtPriceX96, state.tick);

        // update liquidity if it changed
        if (cache.liquidityStart != state.liquidity) self.liquidity = state.liquidity;

        // update fee growth global
        if (params.zeroForOne) {
            self.feeGrowthGlobal0X128 = state.feeGrowthGlobalX128;
        } else {
            self.feeGrowthGlobal1X128 = state.feeGrowthGlobalX128;
        }

        unchecked {
            if (params.zeroForOne == exactInput) {
                result = toBalanceDelta(
                    (params.amountSpecified - state.amountSpecifiedRemaining).toInt128(),
                    state.amountCalculated.toInt128()
                );
            } else {
                result = toBalanceDelta(
                    state.amountCalculated.toInt128(),
                    (params.amountSpecified - state.amountSpecifiedRemaining).toInt128()
                );
            }
        }
    }
```
```solidity

        Slot0 memory slot0Start = self.slot0;
        if (self.slot0.sqrtPriceX96 == 0) revert PoolNotInitialized();
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

        SwapCache memory cache = SwapCache({
            liquidityStart: self.liquidity,
            protocolFee: params.zeroForOne ? (slot0Start.protocolSwapFee % 16) : (slot0Start.protocolSwapFee >> 4),
            hookFee: params.zeroForOne ? (slot0Start.hookSwapFee % 16) : (slot0Start.hookSwapFee >> 4)
        });

        bool exactInput = params.amountSpecified > 0;

        state = SwapState({
            amountSpecifiedRemaining: params.amountSpecified,
            amountCalculated: 0,
            sqrtPriceX96: slot0Start.sqrtPriceX96,
            tick: slot0Start.tick,
            feeGrowthGlobalX128: params.zeroForOne ? self.feeGrowthGlobal0X128 : self.feeGrowthGlobal1X128,
            liquidity: cache.liquidityStart
        });
```
上面都是做一些检测，以及存储swap状态
```solidity
 // continue swapping as long as we haven't used the entire input/output and haven't reached the price limit
        while (state.amountSpecifiedRemaining != 0 && state.sqrtPriceX96 != params.sqrtPriceLimitX96) {
            StepComputations memory step;

            step.sqrtPriceStartX96 = state.sqrtPriceX96;

            (step.tickNext, step.initialized) =
                self.tickBitmap.nextInitializedTickWithinOneWord(state.tick, params.tickSpacing, params.zeroForOne);

            // ensure that we do not overshoot the min/max tick, as the tick bitmap is not aware of these bounds
            if (step.tickNext < TickMath.MIN_TICK) {
                step.tickNext = TickMath.MIN_TICK;
            } else if (step.tickNext > TickMath.MAX_TICK) {
                step.tickNext = TickMath.MAX_TICK;
            }

            // get the price for the next tick
            step.sqrtPriceNextX96 = TickMath.getSqrtRatioAtTick(step.tickNext);

            // compute values to swap to the target tick, price limit, or point where input/output amount is exhausted
            (state.sqrtPriceX96, step.amountIn, step.amountOut, step.feeAmount) = SwapMath.computeSwapStep(
                state.sqrtPriceX96,
                (
                    params.zeroForOne
                        ? step.sqrtPriceNextX96 < params.sqrtPriceLimitX96
                        : step.sqrtPriceNextX96 > params.sqrtPriceLimitX96
                ) ? params.sqrtPriceLimitX96 : step.sqrtPriceNextX96,
                state.liquidity,
                state.amountSpecifiedRemaining,
                params.fee
            );

            if (exactInput) {
                // safe because we test that amountSpecified > amountIn + feeAmount in SwapMath
                unchecked {
                    state.amountSpecifiedRemaining -= (step.amountIn + step.feeAmount).toInt256();
                }
                state.amountCalculated = state.amountCalculated - step.amountOut.toInt256();
            } else {
                unchecked {
                    state.amountSpecifiedRemaining += step.amountOut.toInt256();
                }
                state.amountCalculated = state.amountCalculated + (step.amountIn + step.feeAmount).toInt256();
            }

            // if the protocol fee is on, calculate how much is owed, decrement feeAmount, and increment protocolFee
            if (cache.protocolFee > 0) {
                // A: calculate the amount of the fee that should go to the protocol
                uint256 delta = step.feeAmount / cache.protocolFee;
                // A: subtract it from the regular fee and add it to the protocol fee
                unchecked {
                    step.feeAmount -= delta;
                    feeForProtocol += delta;
                }
            }

            if (cache.hookFee > 0) {
                // step.feeAmount has already been updated to account for the protocol fee
                uint256 delta = step.feeAmount / cache.hookFee;
                unchecked {
                    step.feeAmount -= delta;
                    feeForHook += delta;
                }
            }

            // update global fee tracker
            if (state.liquidity > 0) {
                unchecked {
                    state.feeGrowthGlobalX128 += FullMath.mulDiv(step.feeAmount, FixedPoint128.Q128, state.liquidity);
                }
            }

            // shift tick if we reached the next price
            if (state.sqrtPriceX96 == step.sqrtPriceNextX96) {
                // if the tick is initialized, run the tick transition
                if (step.initialized) {
                    int128 liquidityNet = Pool.crossTick(
                        self,
                        step.tickNext,
                        (params.zeroForOne ? state.feeGrowthGlobalX128 : self.feeGrowthGlobal0X128),
                        (params.zeroForOne ? self.feeGrowthGlobal1X128 : state.feeGrowthGlobalX128)
                    );
                    // if we're moving leftward, we interpret liquidityNet as the opposite sign
                    // safe because liquidityNet cannot be type(int128).min
                    unchecked {
                        if (params.zeroForOne) liquidityNet = -liquidityNet;
                    }

                    state.liquidity = liquidityNet < 0
                        ? state.liquidity - uint128(-liquidityNet)
                        : state.liquidity + uint128(liquidityNet);
                }

                unchecked {
                    state.tick = params.zeroForOne ? step.tickNext - 1 : step.tickNext;
                }
            } else if (state.sqrtPriceX96 != step.sqrtPriceStartX96) {
                // recompute unless we're on a lower tick boundary (i.e. already transitioned ticks), and haven't moved
                state.tick = TickMath.getTickAtSqrtRatio(state.sqrtPriceX96);
            }
        }
```
上面是重要的内容，一个while循环。
遇到第一个方法是
```solidity
(step.tickNext, step.initialized) =
                self.tickBitmap.nextInitializedTickWithinOneWord(state.tick, params.tickSpacing, params.zeroForOne);
```
```solidity
 /// @notice Returns the next initialized tick contained in the same word (or adjacent word) as the tick that is either
    /// to the left (less than or equal to) or right (greater than) of the given tick
    /// @param self The mapping in which to compute the next initialized tick
    /// @param tick The starting tick
    /// @param tickSpacing The spacing between usable ticks
    /// @param lte Whether to search for the next initialized tick to the left (less than or equal to the starting tick)
    /// @return next The next initialized or uninitialized tick up to 256 ticks away from the current tick
    /// @return initialized Whether the next tick is initialized, as the function only searches within up to 256 ticks
    function nextInitializedTickWithinOneWord(
        mapping(int16 => uint256) storage self,
        int24 tick,
        int24 tickSpacing,
        bool lte
    ) internal view returns (int24 next, bool initialized) {
        unchecked {
            int24 compressed = tick / tickSpacing;
            if (tick < 0 && tick % tickSpacing != 0) compressed--; // round towards negative infinity

            if (lte) {
                (int16 wordPos, uint8 bitPos) = position(compressed);
                // all the 1s at or to the right of the current bitPos
                uint256 mask = (1 << bitPos) - 1 + (1 << bitPos);
                uint256 masked = self[wordPos] & mask;

                // if there are no initialized ticks to the right of or at the current tick, return rightmost in the word
                initialized = masked != 0;
                // overflow/underflow is possible, but prevented externally by limiting both tickSpacing and tick
                next = initialized
                    ? (compressed - int24(uint24(bitPos - BitMath.mostSignificantBit(masked)))) * tickSpacing
                    : (compressed - int24(uint24(bitPos))) * tickSpacing;
            } else {
                // start from the word of the next tick, since the current tick state doesn't matter
                (int16 wordPos, uint8 bitPos) = position(compressed + 1);
                // all the 1s at or to the left of the bitPos
                uint256 mask = ~((1 << bitPos) - 1);
                uint256 masked = self[wordPos] & mask;

                // if there are no initialized ticks to the left of the current tick, return leftmost in the word
                initialized = masked != 0;
                // overflow/underflow is possible, but prevented externally by limiting both tickSpacing and tick
                next = initialized
                    ? (compressed + 1 + int24(uint24(BitMath.leastSignificantBit(masked) - bitPos))) * tickSpacing
                    : (compressed + 1 + int24(uint24(type(uint8).max - bitPos))) * tickSpacing;
            }
        }
    }
```
上面的方法，我没太看懂，为什么要发奋bitPos和wordPos，以及为什么做一下&。但注释很清晰，就是下一个初始化的tick。

```solidity
   /// @notice Computes the result of swapping some amount in, or amount out, given the parameters of the swap
    /// @dev The fee, plus the amount in, will never exceed the amount remaining if the swap's `amountSpecified` is positive
    /// @param sqrtRatioCurrentX96 The current sqrt price of the pool
    /// @param sqrtRatioTargetX96 The price that cannot be exceeded, from which the direction of the swap is inferred
    /// @param liquidity The usable liquidity
    /// @param amountRemaining How much input or output amount is remaining to be swapped in/out
    /// @param feePips The fee taken from the input amount, expressed in hundredths of a bip
    /// @return sqrtRatioNextX96 The price after swapping the amount in/out, not to exceed the price target
    /// @return amountIn The amount to be swapped in, of either currency0 or currency1, based on the direction of the swap
    /// @return amountOut The amount to be received, of either currency0 or currency1, based on the direction of the swap
    /// @return feeAmount The amount of input that will be taken as a fee
    function computeSwapStep(
        uint160 sqrtRatioCurrentX96,
        uint160 sqrtRatioTargetX96,
        uint128 liquidity,
        int256 amountRemaining,
        uint24 feePips
    ) internal pure returns (uint160 sqrtRatioNextX96, uint256 amountIn, uint256 amountOut, uint256 feeAmount) {
        unchecked {
            bool zeroForOne = sqrtRatioCurrentX96 >= sqrtRatioTargetX96;
            bool exactIn = amountRemaining >= 0;

            if (exactIn) {
                uint256 amountRemainingLessFee = FullMath.mulDiv(uint256(amountRemaining), 1e6 - feePips, 1e6);
                amountIn = zeroForOne
                    ? SqrtPriceMath.getAmount0Delta(sqrtRatioTargetX96, sqrtRatioCurrentX96, liquidity, true)
                    : SqrtPriceMath.getAmount1Delta(sqrtRatioCurrentX96, sqrtRatioTargetX96, liquidity, true);
                if (amountRemainingLessFee >= amountIn) {
                    sqrtRatioNextX96 = sqrtRatioTargetX96;
                } else {
                    sqrtRatioNextX96 = SqrtPriceMath.getNextSqrtPriceFromInput(
                        sqrtRatioCurrentX96, liquidity, amountRemainingLessFee, zeroForOne
                    );
                }
            } else {
                amountOut = zeroForOne
                    ? SqrtPriceMath.getAmount1Delta(sqrtRatioTargetX96, sqrtRatioCurrentX96, liquidity, false)
                    : SqrtPriceMath.getAmount0Delta(sqrtRatioCurrentX96, sqrtRatioTargetX96, liquidity, false);
                if (uint256(-amountRemaining) >= amountOut) {
                    sqrtRatioNextX96 = sqrtRatioTargetX96;
                } else {
                    sqrtRatioNextX96 = SqrtPriceMath.getNextSqrtPriceFromOutput(
                        sqrtRatioCurrentX96, liquidity, uint256(-amountRemaining), zeroForOne
                    );
                }
            }

            bool max = sqrtRatioTargetX96 == sqrtRatioNextX96;

            // get the input/output amounts
            if (zeroForOne) {
                amountIn = max && exactIn
                    ? amountIn
                    : SqrtPriceMath.getAmount0Delta(sqrtRatioNextX96, sqrtRatioCurrentX96, liquidity, true);
                amountOut = max && !exactIn
                    ? amountOut
                    : SqrtPriceMath.getAmount1Delta(sqrtRatioNextX96, sqrtRatioCurrentX96, liquidity, false);
            } else {
                amountIn = max && exactIn
                    ? amountIn
                    : SqrtPriceMath.getAmount1Delta(sqrtRatioCurrentX96, sqrtRatioNextX96, liquidity, true);
                amountOut = max && !exactIn
                    ? amountOut
                    : SqrtPriceMath.getAmount0Delta(sqrtRatioCurrentX96, sqrtRatioNextX96, liquidity, false);
            }

            // cap the output amount to not exceed the remaining output amount
            if (!exactIn && amountOut > uint256(-amountRemaining)) {
                amountOut = uint256(-amountRemaining);
            }

            if (exactIn && sqrtRatioNextX96 != sqrtRatioTargetX96) {
                // we didn't reach the target, so take the remainder of the maximum input as fee
                feeAmount = uint256(amountRemaining) - amountIn;
            } else {
                feeAmount = FullMath.mulDivRoundingUp(amountIn, feePips, 1e6 - feePips);
            }
        }
    }
```

这个方法，一开始先判断是不是买1，以及是不是指定买的数量。
如果指定输入的数量

    计算去掉费用后的数量amountRemainingLessFee
    如果是买1，就根据当前价格，目标价格，流动性，算出需要输入的数量amountIn
    如果amountRemainingLessFee大于amountIn
        说明输入足够，下一个价格直接就是目标价格
        不然 就根据输入数量amountRemainingLessFee，当前价格，流动性计算出下一个价格是多少

如果指定输出的数量

    根据当前价格，目标价格，流动性计算出输出的数量amountOut
    如果剩余需要输出的数量大于amountOut，
        下一个价格是目标价格
    反之 
        根据剩余需要输出的数量，当前价格，流动性算出下一个价格

下一步就是根据下一个价格，补全输入的数量和输出的数量

判断目标价格是不是下一个价格 存为max

如果是买1
    
    如果max为真并且指定输入的数量
        amountIn就是amountIn，否则根据当前价格，下一个价格和流动性计算amountIn
接下来的意思都差不多，就是根据前面得到的下一个价格，计算出数量，就不一一赘述了

如果指定的是输出数量并且输出的数量大于需要输出的数量，
    就将输出数量设成需要输出的数量
如果指定的是输入数量并且下一个价格不是目标价格，说明这是最后一次循环了，
    费用就是剩余的输入数量-计算得到的输入数量
否则
    费用就是根据输入数量计算得到的

```solidity
if (exactInput) {
                // safe because we test that amountSpecified > amountIn + feeAmount in SwapMath
                unchecked {
                    state.amountSpecifiedRemaining -= (step.amountIn + step.feeAmount).toInt256();
                }
                state.amountCalculated = state.amountCalculated - step.amountOut.toInt256();
            } else {
                unchecked {
                    state.amountSpecifiedRemaining += step.amountOut.toInt256();
                }
                state.amountCalculated = state.amountCalculated + (step.amountIn + step.feeAmount).toInt256();
            }

            // if the protocol fee is on, calculate how much is owed, decrement feeAmount, and increment protocolFee
            if (cache.protocolFee > 0) {
                // A: calculate the amount of the fee that should go to the protocol
                uint256 delta = step.feeAmount / cache.protocolFee;
                // A: subtract it from the regular fee and add it to the protocol fee
                unchecked {
                    step.feeAmount -= delta;
                    feeForProtocol += delta;
                }
            }

            if (cache.hookFee > 0) {
                // step.feeAmount has already been updated to account for the protocol fee
                uint256 delta = step.feeAmount / cache.hookFee;
                unchecked {
                    step.feeAmount -= delta;
                    feeForHook += delta;
                }
            }

            // update global fee tracker
            if (state.liquidity > 0) {
                unchecked {
                    state.feeGrowthGlobalX128 += FullMath.mulDiv(step.feeAmount, FixedPoint128.Q128, state.liquidity);
                }
            }
```
上面就是计算amount,计算fee
```solidity
// shift tick if we reached the next price
            if (state.sqrtPriceX96 == step.sqrtPriceNextX96) {
                // if the tick is initialized, run the tick transition
                if (step.initialized) {
                    int128 liquidityNet = Pool.crossTick(
                        self,
                        step.tickNext,
                        (params.zeroForOne ? state.feeGrowthGlobalX128 : self.feeGrowthGlobal0X128),
                        (params.zeroForOne ? self.feeGrowthGlobal1X128 : state.feeGrowthGlobalX128)
                    );
                    // if we're moving leftward, we interpret liquidityNet as the opposite sign
                    // safe because liquidityNet cannot be type(int128).min
                    unchecked {
                        if (params.zeroForOne) liquidityNet = -liquidityNet;
                    }

                    state.liquidity = liquidityNet < 0
                        ? state.liquidity - uint128(-liquidityNet)
                        : state.liquidity + uint128(liquidityNet);
                }

                unchecked {
                    state.tick = params.zeroForOne ? step.tickNext - 1 : step.tickNext;
                }
            } else if (state.sqrtPriceX96 != step.sqrtPriceStartX96) {
                // recompute unless we're on a lower tick boundary (i.e. already transitioned ticks), and haven't moved
                state.tick = TickMath.getTickAtSqrtRatio(state.sqrtPriceX96);
            }
```
```solidity
 /// @notice Transitions to next tick as needed by price movement
    /// @param self The Pool state struct
    /// @param tick The destination tick of the transition
    /// @param feeGrowthGlobal0X128 The all-time global fee growth, per unit of liquidity, in token0
    /// @param feeGrowthGlobal1X128 The all-time global fee growth, per unit of liquidity, in token1
    /// @return liquidityNet The amount of liquidity added (subtracted) when tick is crossed from left to right (right to left)
    function crossTick(State storage self, int24 tick, uint256 feeGrowthGlobal0X128, uint256 feeGrowthGlobal1X128)
        internal
        returns (int128 liquidityNet)
    {
        unchecked {
            TickInfo storage info = self.ticks[tick];
            info.feeGrowthOutside0X128 = feeGrowthGlobal0X128 - info.feeGrowthOutside0X128;
            info.feeGrowthOutside1X128 = feeGrowthGlobal1X128 - info.feeGrowthOutside1X128;
            liquidityNet = info.liquidityNet;
        }
    }
```
```solidity
  (self.slot0.sqrtPriceX96, self.slot0.tick) = (state.sqrtPriceX96, state.tick);

        // update liquidity if it changed
        if (cache.liquidityStart != state.liquidity) self.liquidity = state.liquidity;

        // update fee growth global
        if (params.zeroForOne) {
            self.feeGrowthGlobal0X128 = state.feeGrowthGlobalX128;
        } else {
            self.feeGrowthGlobal1X128 = state.feeGrowthGlobalX128;
        }

        unchecked {
            if (params.zeroForOne == exactInput) {
                result = toBalanceDelta(
                    (params.amountSpecified - state.amountSpecifiedRemaining).toInt128(),
                    state.amountCalculated.toInt128()
                );
            } else {
                result = toBalanceDelta(
                    state.amountCalculated.toInt128(),
                    (params.amountSpecified - state.amountSpecifiedRemaining).toInt128()
                );
            }
        }
```