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

## Security Considerations
1. **Authorization**: Only the vault contract is authorized to call `update_streak` on the streaks contract
2. **Atomicity**: All state changes are atomic - if any step fails, the entire transaction reverts
3. **Balance Tracking**: Vault balances are always updated before any cross-contract calls, ensuring funds are always accounted for
4. **Error Isolation**: Failures in streaks or rewards contracts cannot affect the core vault accounting