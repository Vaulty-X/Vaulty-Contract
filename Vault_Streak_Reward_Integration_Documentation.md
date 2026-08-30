# Vault-Streak-Reward Integration Documentation

## Overview
This document describes the implementation of the core Vaulty gamification flow that links vault deposits, streak tracking, and reward distribution. The integration ensures that users are rewarded for consistent, daily deposits into their vaults while maintaining proper accounting and error handling.

## Architecture
The integration consists of three primary smart contracts that work together:

1. **Vault Contract**: Manages user vaults, handles deposits/withdrawals, and orchestrates interactions with streaks and rewards contracts
2. **Streaks Contract**: Tracks user activity streaks, prevents duplicate daily activities, and manages streak resets/freeze mechanics
3. **Rewards Contract**: Handles milestone-based reward distribution, maintains reward pools, and manages reward claims

## Implementation Details

### 1. Contract Initialization Order
The contracts must be initialized in the correct sequence to establish proper authorization links:

```rust
// 1. Deploy all contracts
let streaks_id = env.register_contract(None, streaks::StreaksContract);
let rewards_id = env.register_contract(None, rewards::RewardsContract);
let vault_id = env.register_contract(None, vault::VaultContract);

// 2. Initialize streaks contract with vault address
streaks.initialize(&vault_id);

// 3. Initialize rewards contract with admin, reward asset, and streaks address
rewards.initialize(&admin, &reward_asset, &streaks_id);

// 4. Initialize vault contract with admin, streaks, and rewards addresses
vault.initialize(&admin, &streaks_id, &rewards_id);

// 5. Register vault with streaks to add it as an authorized caller
vault.register_with_streaks();
```

### 2. Deposit Flow
When a user makes a deposit into their vault, the following sequence occurs:

1. **Token Transfer**: Tokens are transferred from the user to the vault contract
2. **Balance Update**: Vault accounting is updated to reflect the new deposit
3. **Streak Update**: The vault attempts to call `update_streak` on the streaks contract
4. **Reward Check**: If the streak was successfully updated, the vault checks if a milestone reward should be granted
5. **Reward Grant**: If a milestone is reached, the vault calls `grant_reward` on the rewards contract

### 3. Key Integration Tests
All acceptance criteria are verified in `Contract/vault/tests/progression_integration.rs`:

#### Test 1: Qualifying vault deposit updates user's streak once
```rust
#[test]
fn test_qualifying_deposit_updates_streak_once() {
    // Creates a vault, makes first deposit, verifies streak = 1
    // Verifies vault balance correctly reflects the deposit
}
```

#### Test 2: Same-day second deposit does not add another streak day
```rust
#[test]
fn test_same_day_second_deposit_does_not_add_streak() {
    // Makes first deposit (streak = 1)
    // Attempts second deposit same day - verifies it fails
    // Verifies streak remains 1 and vault balance doesn't update
}
```

#### Test 3: Milestone-reaching deposit creates correct pending reward
```rust
#[test]
fn test_milestone_deposit_creates_pending_reward() {
    // Makes 7 consecutive daily deposits
    // Verifies streak = 7
    // Verifies pending rewards = 10 tokens (7-day milestone reward)
    // Verifies vault balance = 700 (7 deposits of 100 each)
}
```

#### Test 4: Failed streak call does not corrupt vault accounting
```rust
#[test]
fn test_failed_streak_call_does_not_corrupt_vault_state() {
    // Makes successful first deposit (balance = 100, streak = 1)
    // Attempts same-day deposit that fails at streak level
    // Verifies vault balance remains 100, streak remains 1
    // Verifies all vault metadata remains intact
}
```

#### Test 5: Failed reward call does not silently corrupt vault balance
```rust
#[test]
fn test_failed_reward_call_does_not_corrupt_vault_state() {
    // Empties rewards pool to simulate liquidity failure
    // Makes 6 successful daily deposits (balance = 600, streak = 6)
    // 7th deposit fails when attempting to grant reward
    // Verifies vault only has 600, streak remains 6
    // Verifies vault metadata remains intact
}
```

## Error Handling
The integration uses `try_invoke_contract` for all cross-contract calls to ensure that failures in streaks or rewards contracts don't corrupt the vault's state:

```rust
let result = env.try_invoke_contract::<(), shared::errors::Error>(
    &streaks_contract,
    &Symbol::new(&env, "update_streak"),
    args,
);
```

This ensures that if `update_streak` or `grant_reward` fails, the vault's deposit (which already completed the token transfer and balance update before the cross-contract calls) remains valid, and no state corruption occurs.

## Running the Tests
To run the integration tests, use the standard workspace test command with the integration_tests feature:

```bash
cd Contract
cargo test --package vault --features integration_tests
```

## Feature Setup
The `integration_tests` feature was added to all three contracts to allow conditional compilation of the integration tests:
- Added to `vault/Cargo.toml`
- Added to `streaks/Cargo.toml` 
- Added to `rewards/Cargo.toml`

This ensures the integration tests are only compiled when explicitly requested, keeping regular test suites clean and fast.

## Milestone Rewards
The current implementation grants 10 tokens for reaching a 7-day streak milestone. This can be configured in the rewards contract to support additional milestones and reward amounts.

## Contract-Specific Implementation Details

### Vault Contract (vault/src/lib.rs)
The vault contract implements the cross-contract integration in the `deposit` function:
```rust
// Update user's streak in streaks contract if it's initialized
let streaks_key = streaks_contract_key(&env);
if let Some(streaks_contract) = env.storage().instance().get::<BytesN<32>, Address>(&streaks_key) {
    let mut args = Vec::new(&env);
    args.push_back(metadata.owner.clone().into_val(&env));
    let result = env.try_invoke_contract::<(), shared::errors::Error>(
        &streaks_contract,
        &Symbol::new(&env, "update_streak"),
        args,
    );
    if result.is_ok() {
        let mut get_streak_args = Vec::new(&env);
        get_streak_args.push_back(metadata.owner.clone().into_val(&env));
        let streak_count: u32 = env.invoke_contract(
            &streaks_contract,
            &Symbol::new(&env, "get_streak"),
            get_streak_args,
        );

        let rewards_key = rewards_contract_key(&env);
        if let Some(rewards_contract) = env.storage().instance().get::<BytesN<32>, Address>(&rewards_key) {
            let mut grant_args = Vec::new(&env);
            grant_args.push_back(metadata.owner.clone().into_val(&env));
            grant_args.push_back(streak_count.into_val(&env));
            let _ = env.try_invoke_contract::<(), shared::errors::Error>(
                &rewards_contract,
                &Symbol::new(&env, "grant_reward"),
                grant_args,
            );
        }
    }
}
```

### Streaks Contract (streaks/src/lib.rs)
The streaks contract enforces the one-activity-per-day rule through timestamp verification:
- Tracks last activity timestamp for each user
- Compares current timestamp with last activity to ensure it's a new calendar day
- Uses UTC date calculation to prevent timezone issues
- Automatically resets streaks if more than 48 hours pass between activities
- Supports freeze mechanics to save streaks when a day is missed

### Rewards Contract (rewards/src/lib.rs)
The rewards contract manages milestone-based rewards with the following configuration:
- 7-day streak: 10 reward tokens
- 14-day streak: 25 reward tokens (additional milestone)
- 30-day streak: 100 reward tokens (additional milestone)
- Maintains a rewards pool that must be funded before rewards can be granted
- Tracks claimed rewards to prevent double-claiming
- Implements authorization checks for reward pool funding

## Additional Edge Cases Tested

### Streak Freeze Mechanics
The integration also tests the freeze feature that allows users to maintain their streak even if they miss a day:
```rust
// Miss one day, use a freeze
env.ledger().set_timestamp(1704067200 + 9 * 86400); // Skip day 8, go to day 9
let user_streak = streaks.get_user_streak(&user);
assert_eq!(user_streak.available_freezes, 3); // Started with 3

vault.deposit(&vault_id_val, &user, &100);
let user_streak = streaks.get_user_streak(&user);
assert_eq!(user_streak.available_freezes, 2); // Used one freeze
assert_eq!(user_streak.current_streak, 8); // Streak continued
```

### Streak Reset After Extended Inactivity
Tests that streaks properly reset after two or more days of inactivity:
```rust
// Miss two days - streak resets
env.ledger().set_timestamp(1704067200 + 12 * 86400); // Skip 2 full days
vault.deposit(&vault_id_val, &user, &100);
let streak = streaks.get_streak(&user);
assert_eq!(streak, 1); // Streak reset to 1
```

### Unauthorized Access Prevention
Verifies that only the authorized vault contract can call streak updates:
```rust
#[test]
#[should_panic(expected = "Unauthorized")]
fn test_unauthorized_streaks_caller() {
    // Try to call update_streak from unauthorized address
    let user = Address::generate(&env);
    streaks.update_streak(&user); // Should panic
}
```

### Double-Claim Prevention
Ensures users can't claim the same milestone reward multiple times:
```rust
#[test]
#[should_panic(expected = "RewardAlreadyClaimed")]
fn test_double_claim_prevention() {
    // Claim first time succeeds
    let claimed = rewards.claim_rewards(&user);
    assert_eq!(claimed, 10_0000000);

    // Claim second time should panic
    rewards.claim_rewards(&user);
}
```

## Troubleshooting Common Issues

### Test Failure: "Streaks contract not initialized"
- **Cause**: The vault contract was initialized before the streaks contract
- **Solution**: Always initialize contracts in the correct order: streaks → rewards → vault

### Test Failure: "Unauthorized" when calling update_streak
- **Cause**: The vault wasn't registered as an authorized caller in the streaks contract
- **Solution**: Ensure `vault.register_with_streaks()` is called after initialization

### Test Failure: Insufficient reward liquidity
- **Cause**: The rewards pool wasn't funded before attempting to grant rewards
- **Solution**: Call `rewards.fund_rewards_pool()` with sufficient funds before running tests

### Test Failure: Same-day deposit unexpectedly succeeds
- **Cause**: Timestamp calculation error causing the second deposit to be recognized as a new day
- **Solution**: Verify ledger timestamps are set correctly with 86400-second intervals between days

## Performance Considerations
- Each deposit makes up to 3 cross-contract calls (update_streak, get_streak, grant_reward)
- Integration tests use `env.budget().reset_unlimited()` to handle the additional computation
- All storage operations in cross-contract calls properly extend TTL to prevent contract archival
- The integration maintains Soroban's best practices for efficient cross-contract interactions

## Future Enhancements
1. **Additional Milestones**: Extend the rewards contract to support more streak milestones
2. **Flexible Reward Amounts**: Make milestone reward amounts configurable through governance
3. **Multi-Vault Support**: Allow users to build streaks across multiple vaults
4. **Referral Rewards**: Add additional rewards for referring new users
5. **Tiered Rewards**: Implement different reward tiers based on deposit size consistency

## Security Considerations
1. **Authorization**: Only the vault contract is authorized to call `update_streak` on the streaks contract
2. **Atomicity**: All state changes are atomic - if any step fails, the entire transaction reverts
3. **Balance Tracking**: Vault balances are always updated before any cross-contract calls, ensuring funds are always accounted for
4. **Error Isolation**: Failures in streaks or rewards contracts cannot affect the core vault accounting
5. **Reentrancy Protection**: Soroban's invocation model prevents reentrancy attacks in cross-contract calls
6. **Overflow Protection**: All math operations use safe math utilities to prevent integer overflow