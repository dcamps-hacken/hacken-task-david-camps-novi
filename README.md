## SMART CONTRACT SECURITY REPORT

This document is the result of an exhaustive revision of the provided `Staking.sol` smart contract. All of it has been produced by manual checking of the source code, without implementing additional testing or using external auditing tools.

The content of this report is organized in the following sections, by order:

1. Documentation
2. Global variables and constructor
3. Staking
4. Withdrawing and rewards
5. Token
6. Events
7. Other general issues
8. Gas optimization

Whilst the first 6 sections contain the functions related to a block of logic and their respective issues, the 7th section `Other general issues` includes notes of transversal matter, and the latest `Gas optimization` includes some easy-direct changes that can be introduced to save some gas (note that not 100% of gas optimizations have been suggested since are not so clear/worth/UX-friendly).

In some cases, a code example or suggestion has been added for further clarification of the recommendations.

<br>

## DOCUMENTATION

Documentation of the processes and smart contract logic are not provided with the code (e.g. a white paper), making it harder for any reader, user, developer or auditor to understand the intention of the authors. Therefore, many assumptions have been made to navigate the different processes.

Additionally, the use of NatSpec is highly recommended to break-down the smart contract logic into the different functions and parameters. Following the same idea, having well-documented functions and processes facilitates the integration of the code, its readability and logic review for the team members or outsiders of the protocol.

<br>

## GLOBAL VARIABLES AND CONSTRUCTOR

<br>

1. Unused variables
   > Variables `tokenDecimals`, `earlyWithdrawFee`, `_operators` and `levelPeriods` should be either used in the code (e.g. some notes suggest a fee in some cases but never implemented) or deleted

<br>

2. Variables can be declared `constant`
   > The variables `tokenDecimals`, `earlyWithdrawFee`, `rewardRate` can be declared as constants<br>
   > This formula saves some gas

<br>

3. Variables can be declared `immutable`
   > Some variables are set during in the constructor but never change: `stakingToken`, `rewardToken` and `uniswapV2Router`

<br>

```solidity
IERC20DetailedBurnable public immutable stakingToken;
IERC20DetailedMintable public immutable rewardToken;
IUniswapV2Router public immutable uniswapV2Router;

uint256 public constant REWARD_RATE;

constructor(address _stakingToken, address _rewardToken, address _routerAddress) {
    stakingToken = IERC20DetailedBurnable(_stakingToken);
    rewardToken = IERC20DetailedMintable(_rewardToken);
    uniswapV2Router = IUniswapV2Router(_routerAddress);
}
```

<br>

## STAKING

The deducted logic from the contract code is that a user can:

- Stake some tokens for a certain time period using `stake()`
- Add tokens to the stake within the same time period using `stake()`
- Prolong the current staking period using `prolong()`

<br>

1. It is recommended to explicitly show these three possibilities using three different functions (e.g. `stake()`, `increaseStake()`, `prolong()`) so that possibilities and expectations are clear for the user or even future adopters of this code, and delete the `_stake()` function.
   > This would allow to design a more clear logic in the all three functions and the whole application: there is no transfer of ERC20 in the case of `prolong()`; the function call to `_validatePeriod(periods)` would only be called for `stake()` and `prolong()` since it doesn't play any role in the `increaseStake()` and thus the input parameter can be validated immediately.

- Affected functions: `stake()`, `prolong()`, `_stake()`

<br>

2. Lack of input validation routines or checks later than recommended
   > Functions `stake()` and `prolong()` should make sure the input param `period` is checked at the top <br>
   > Function `_validatePeriod()` should be tested at this level rather than in `_stake()`

- Affected functions: `stake()`, `prolong()`

<br>

3. NatSpec in `prolong()` mentions the existence of a fee but it is not translated into code (same in `emergencyWithdraw()`)

<br>

## WITHDRAWING AND REWARDS

<br>

1. The variables `totalStaked` and `stakes[msg.sender].amount` are not updated when a user unstakes using `withdraw()`
   > These values should be updated as in `emergencyWithdraw()`

<br>

```solidity
function withdraw() external {
    ...
    totalStaked -= stakes[msg.sender].amount;
    stakes[msg.sender].amount = 0;
    _withdraw(msg.sender, stakes[msg.sender].amount);
}
```

<br>

2. Use a pull-over-push approach
   > The function `_collectRewards()` transfers the amount of reward tokens to the user in a push approach in three cases: `stake()`, `prolong()`, `withdraw()`.<br>
   > It is recommended to let users get their transactions in a pull approach calling `harvest()` themselves and remove the call to `_collectRewards()` from the other functions. <br>
   > The bool isDirect from `_collectRewards()` would be then removed.

- Affected functions: `stake()`, `prolong()`, `withdraw()`, `harvest()`, `_collectRewards()`

<br>

3. The function `_nextRewardDate()` needs further development and clear logic
   > The unclarity of its wording, return values and conditional statements provide many questions about what the expected return is: while its name suggests it would return a timestamp of the moment an account can claim a reward, in `allStakeInfo()` the value `nextRewardSeconds` may suggest it should return how many seconds are left until the next claim (among others). Additionally, the logic of the smart contract allows the collection of rewards at any moment, so the concept of having a `rewardDate` does not make much sense. Deeper description of the function intention and reward process should be added in order to provide better feedback. <br>
   > The check of `lastRewardClaims[account] == 0` suggests that, if a user has not claimed anything before, the next reward date is `0`, which may indicate that there is nothing to claim. However, `lastRewardClaims[account] == 0` is also valid when a user has tokens staked but has no claimed then before, which makes this condition not acceptable. Using a condition like `stakeInfo.activeUntil == 0` would work as an option to check if a user has no stake, solving this issue. <br>
   > Using well-structured conditional statement blocks would also help understanding correctly the function

<br>

```solidity
function _nextRewardDate(address account) private view returns (uint256) {
    StakeInfo memory stakeInfo = stakes[account];
    if (stakeInfo.activeUntil == 0) {
        return 0;
    } else if (block.timestamp > stakeInfo.activeUntil) {
        return stakeInfo.activeUntil ;
    } else {
        uint256 passedPeriods = (block.timestamp - stakeInfo.startedAt) / _DAY;
        return ((passedPeriods + 1) * _DAY) + stakeInfo.startedAt;
    }
}
```

<br>

4. In `_rewardAmount()` the staking period passed is not calculated properly, since it is missing the staking process start `startedAt`.

<br>

```solidity
function _rewardAmount(address account) private view returns (uint256 periodsPassed, uint256 reward){
    ...
    periodsPassed = (time - lastRewardClaims[account] - stakeInfo.startedAt) / _DAY;
    ...
}
```

<br>

5. Access control

> Currently, `updateRewardsPool()` has no access control and therefore anyone could send tokens to the contract using that function. Narrowing down its use to a contract `owner` or adding other roles would be recommended.

<br>

6. Impossibility to recover tokens

   > In case the project has any issues or it is finalized, there is no way to recover the remaining tokens from the contract.

<br>

7. Price manipulation via direct transfer of tokens is possible

   > If the logic of calculating an amount of `rewardTokens` goes through `toRewardTokens()`, it should also be noted that currently the price of the swap can be manipulated if tokens are sent to the contract address using `updateRewardsPool()` or using a direct transaction to the contract. Access control implementation and/or further development is recommended.

<br>

8. Use of illogical wording in `collectRewards()`

   > The use of a negative wording in `notDirect` for a bool when it is `true` is illogical and should be renamed `direct` or pick another name

<br>

```solidity
function _collectRewards(address account, bool direct) private {...}
```

<br>

9. Use `require()` over `if`

   > In `_collectRewards()` the whole content of the method is inside a conditional statement, which is not a recomended pattern since we will not notice whether the function is fulfilling that condition or not. A better approach would be to use a requirement instead. However, such condition has already been checked in the functions that are currently calling `_collectRewards()`: it's use is redundant and should be removed. <br>
   > A second conditional statement in `_collectRewards()` checks if `(notDirect && periods == 0)` and just `return` as the outcome of it. Again, fulfilling this condition will not be noticeable. Using a requirement is a better option.

<br>

10. The variable `lastRewardClaims[account]` is not reset

    > This variable increases in `_collectRewards()` every time a reward is claimed. However, once a whole staking period has passed (after `activeUntil`), it should reset to `0` so that the reward amounts of new staking processes can be properly set.

<br>

11. Unclear reward mechanism

    > The reward calculation and mechanism is unclear because of the lack of documentation, inappropriate wording and inconsistent logic. It is recommended to further break down the reward mechanism and tokenomics, produce exhaustive documentation and translate that into consistent code : according to `_withdraw()` and `_collectRewards()`, the logic is that a user can stake `stakingTokens` to farm `rewardTokens`. However, in many cases this logic is broken: in `_rewardAmount()` the amount of farmed `rewardTokens` is calculated, but it is later used in `_collectRewards()` as an input value of `toRewardTokens()` to return an amount of `stakingTokens`. It then appears that there are two methods to calculate the amount of reward tokens.

<br>

12. The function `updateRewardsPool()` is sending `stakingTokens` instead of `rewardTokens`.
    > According to Uniswap documentation, the input parameters are not introduced in the right order

<br>

## TOKEN

<br>

1. Consider preventing the transfer of tokens to the same contract address:
   > In order to avoid locking tokens in the contract, adding a modifier is a good solution to prevent it.

- Affected functions: `_stake()`, `_withdraw()`, `updateRewardsPool()`, `_collectRewards()`

<br>

```solidity
modifier validAddress (address account) {
    require(account != address(this), "tokens can't be sent to this contract")
}
function _withdraw(address account, uint256 _amount) validAddress (account) private {...}
```

<br>

2. Use OpenZeppelin's `safeERC20` wrapper functions instead of the regular `transfer()` and `transferFrom()`
   > Some ERC20 implementations of `transfer()` and `transferFrom()` can return false on failure instead of reverting. It is recommended to wrap such calls into a require() statement or use `safeERC20` wrapper. <br>
   > Althought the ERC20 suggests returning `true` on success for these functions, some tokens are non-compliant in this regard. It is recommended to use `safeERC20` wrapper. <br>
   > The interfaces and smart contract must be updated so that `safeERC20` can be used instead of the regular functions.

- Affected functions: `_stake()`, `_withdraw()`, `updateRewardsPool()`, `_collectRewards()`

<br>

## EVENTS

<br>

1. Bool `isEarly` from `event Withdraw()` is not reflecting its meaning
   > The logic of the `event Withdraw(address indexed account, uint256 sum, bool isEarly)` reflects the willingness to check if a withdrawal is called early using `emergencyWithdraw()` or normally using `withdraw()`. However, there is no difference in the emission of this event since the bool is hardcoded to `_withdraw()`.

- Affected functions: `withdraw()`, `emergencyWithdraw()`, `_withdraw()`

<br>

```solidity
function withdraw() external {
    ...
    _withdraw(msg.sender, stakes[msg.sender].amount, false);
}
function emergencyWithdraw() external {
    ...
    _withdraw(msg.sender, stakes[msg.sender].amount, true);
}
function _withdraw(address account, uint256 _amount, bool isEarly) private {
    ...
    emit Withdraw(account, _amount, isEarly);
}
```

<br>

2. The event `RewardPoolUpdated()` is not tracking who is updating the reward pool.
   > It is recommended to add the contributor in the event

<br>

## OTHER GENERAL ISSUES

<br>

1. Lack of return value checks may translate into unexpected results
   > Several functions do not check the return value, making the code error-prone, which can lead to unexpected results <br>
   > A bool can be added as a return value for the functions called, to be later checked

- Affected functions: `stake()`, `prolong()`, `_stake()`, `collectRewards()`, `harvest()`, `withdraw()`, `emergencyWithdraw()`, `_withdraw()` (and the corresponding functions they call)

<br>

2. Unnecessary calculations are performed in `getStakingPeriod()`

   > The variable period is already stored in the struct `StakeInfo`, so there is no need to calculate it

<br>

3. There is a typo in the spelling of `_DAYS_IN_MONT`, which should be `_DAYS_IN_MONTH`

<br>

4. In `_YEAR_IN_DAYS` it is declared that a year has `360` days instead of `365`.

   > It is not specified if it is done like this for rounding issues or it is just a typo. If kept like this, it may lead to incorrect expectations from the users.

<br>

5. Variables `_DAY`, `_DAY_IN_MONT`, `_YEAR_IN_DAYS` do not self-explain and are prompt to errors

   > Suggested names could be `_SECONDS_PER_DAY`, `_DAYS_PER_MONTH`, `_DAYS_PER_YEAR`

<br>

6. There is an inconsistency in the use of `periods`, `periodsPassed`, `passedPeriods` to refer to the same concept across the whole contract

<br>

7. Naming of variables, params and functions is inconsistent according to the [Solidity Style Guide](https://docs.soliditylang.org/en/latest/style-guide.html#style-guide)

   > In most cases, input parameters are named directly (e.g. `account`, `amount`). However, there are other cases in which the underscore naming is used (e.g. `_amount` in `_withdraw()`). <br>
   > The same happens with the private functions, which are mostly named `_functionName`, while `toRewardTokens` does not follow this naming. <br>
   > The same goes for the mapping names, one of them being `_mappingName` instead of `mappingName`. <br>
   > Being consistent it is recommended, using for all input values `_paramName` or `paramName`.

<br>

8. Division rounding
   > In several places, unnecessary calculations are used to calculate times. It is recommended to stick to default units (seconds) to handle different data/functions unless it is a must to use days, years, etc. This solution would be better than using multipliers as in `_rewardAmount()`.

- Affected functions: `getStakingPeriod()`, `_rewardAmount()`, `_nextRewardDate()`, `_stake()`, `_collectRewards()`

<br>

9. Re-entrancy possibilities

   > In several cases, the [checks-effects-interaction pattern](https://docs.soliditylang.org/en/latest/security-considerations.html#use-the-checks-effects-interactions-pattern) is not followed, and calls to external contracts are performed before applying its effects in `updateRewardsPool()`, `_stake()`, `_withdraw()`, `_collectRewards()` (e.g. emission of events). <br>
   > The named pattern should be followed, considering also the application of OpenZeppelin's ReentrancyGuard.

<br>

10. Locking of pragma
    > Contracts should be deployed with the same compiler version used to test them. <br>
    > Locking pragma avoids accidentally deploy using a later compiler, which adds the risk of having undiscovered bugs. <br>
    > Outsiders who want to deploy the contract will be aware of which version the original authors intended to use.

<br>

```solidity
pragma solidity 0.8.12;
```

<br>

## GAS OPTIMIZATION

<br>

1. Variables loaded as `storage` can be summoned as `memory`

- Affected functions: `allStakeInfo()`, `prolong()`, `_rewardAmount()`, `_nextRewardDate()`

<br>

```solidity
function prolong(uint256 period) external {
    StakeInfo memory stakeInfo = stakes[msg.sender];
    ...
}
```

<br>

2. State variables can be set to `private` visibility

   > Some variables such as the token contracts (`IERC20DetailedBurnable public stakingToken` and so on) can be set as `private` instead of `public` to save some gas if the authors do not intent them to be accessed from outside of the contract.

<br>

3. Over declaration of variables in `allStakeInfo()`
   > A new variable `reward` is created unnecessarily to get the value of `_rewardAmount()` since `rewards` can be directly used to saving some gas
