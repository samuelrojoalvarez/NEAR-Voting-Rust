# NEAR-Voting-Rust
Writing a voting contract for unlocking transfers.
The purpose of this contract is solely for validators to vote on whether to unlock token transfer. Validators can call vote to vote for yes with the amount of stake they wish to put on the vote. If there are more than 2/3 of the stake at any given moment voting for yes, the voting is done. After the voting is finished, no one can further modify the contract.

# GOAL: Writing a voting contract for unlocking transfers.

Learning to write a very simple yet a very popular contract, which is a part of the core-contracts in near protocol. This particular voting contract will be written in the Rust programming language, so if you are not familiar with it - do not worry. As long as you follow the steps mentioned in the quest, you will be able to successfully deploy and run the contract.

However there are certain prerequisites that you would need to install in order to successfully build and deploy the contract on the near testnet. You can refer to the NEAR Protocol Rust setup quest, complete that setup and then start with the quest.

This contract is one of the core contracts from the near repo and [here](https://github.com/near/core-contracts) is the GitHub link from where you can clone the entire repo but for this quest, we will be focusing on the voting contract.

Let's begin!


## Importing dependencies

Let's us take a look at the lib.rs file, which is where the main contract code will go.
We will start with importing all the required crates.

```
use near_sdk::borsh::{self, BorshDeserialize, BorshSerialize};
use near_sdk::json_types::{U128, U64};
use near_sdk::{env, near_bindgen, AccountId, Balance, EpochHeight};
use std::collections::HashMap;

#[global_allocator]
static ALLOC: near_sdk::wee_alloc::WeeAlloc = near_sdk::wee_alloc::WeeAlloc::INIT;
```

The keyword 'use' is used to import the various crates from near_sdk as well as the HashMap from the standard collections. These crates consist of functionality that we would be consuming in our code. 

P.S: Crates in rust are similar to packages that can be imported to be used in our contract. This helps us to avoid writing all the necessary functionality ourselves. Instead, we can make use of the existing functionality by using crates.

A note about #[global_allocator]
Allocators are the way that programs in Rust obtain memory from the system at runtime. The attribute allows Rust programs to set their allocator to the system allocator, as well as define new allocators by implementing the GlobalAlloc trait.


## Writing the main structure

The main structure of the contract is called "VotingContract". Structures in Rust are similar to classes, which will encompass all the required variables and functions required for the voting contract.

**Borsch-Serialize** and **Deserialize** are macros from the near SDK which can be used to serialize and deserialize the structure to and from the binary format.

Here is the code for our main structure

```
#[near_bindgen]
#[derive(BorshDeserialize, BorshSerialize)]
pub struct VotingContract {
    /// How much each validator votes
    votes: HashMap<AccountId, Balance>,
    /// Total voted balance so far.
    total_voted_stake: Balance,
    /// When the voting ended. `None` means the poll is still open.
    result: Option<WrappedTimestamp>,
    /// Epoch height when the contract is touched last time.
    last_epoch_height: EpochHeight,
}

```
A quick look at the structure:
* The structure is public, therefore the keyword _pub_.
* near_bindgen is used so because once we wrap a struct in #[near_bindgen], it generates a smart contract compatible with the NEAR blockchain
* The struct has 4 variables called _votes_ , _total_voted_stake_ , _result_ and _last_epoch_height_ .
* The variable types have been taken from the near_sdk, except for **HashMap**, which is from the standard collection.

If you think most of the hard work is done, you are completely wrong - we are just getting started :P. However, do not be overwhelmed by Rust - the more contracts you read and the more you work with, you will get into the flow of things.

## Writing our first implementation

In Rust, there are _structures_ and there are _implementations_. The implementations, denoted by the keyword _impl_ are used to implement the methods within the structure.

The first implementation is a default implementation which says that before using the voting contract it should be initialised. So, if we try to call any functionality within the voting contract without initialising at first, we will get an error.

Here is the code for the implementation:

```
impl Default for VotingContract {
    fn default() -> Self {
        env::panic(b"Voting contract should be initialized before usage")
    }
}
```

Let us look at the next set of implementation where we define all the the next set of functions used to initialise and perform some actions for the VotingContract.

1. Our first function is called _new_, which is a constructor. It doesn't take any aruguments and it returns self object. **Self** is an equivalent of the voting contract itself. It's basically creating a new instance of this class.
The **init** macro at the top of the _new_ function is used to initialise the state of the contract and checks if it doesn't have any previous state.

We will wrap all the functions within our implementation, so start with the code snippet below.

```
#[near_bindgen]
impl VotingContract {
  //ALL FUNCTIONS HERE
}

```

## Writing our first function

So, the way we work with Rust contracts is Near is, we deploy the class/structure first within the contract and then initialise it. Let us write the constructor now.The initial contract state comes from the env variable, so we check if the contract has been initialized earlier by checking the state.
Then we assign default values to the structure variables.
Here is our first function.

```
#[init]
    pub fn new() -> Self {
        assert!(!env::state_exists(), "The contract is already initialized");
        VotingContract {
            votes: HashMap::new(),
            total_voted_stake: 0,
            result: None,
            last_epoch_height: 0,
        }
   }
```
Remember to add this function within your implementation defined in the previous quest.
Now that we have completed our constructor, let us get onto writing some more functions.

## Function to vote or withdraw the vote - Part1

Let us start with the main function - vote.
The vote function consists of two arguments - _self_ and _is_vote_

Whenever we attach a &mut keyword with the argument, it would mean that the argument is mutable.
In this case, the **mut** keyword is associated with the structure itself, therefore, this refers to a function
where there could be state change happening on the blockchain.

In simple terms, this means that the state of the data stored on chain could be modified in this function.

This function is slightly longer, so let us break this down into 3 parts.
Before that, here is a look at the function.

```
/// Method for validators to vote or withdraw the vote.
    /// Votes for if `is_vote` is true, or withdraws the vote if `is_vote` is false.
    pub fn vote(&mut self, is_vote: bool) {
        self.ping();
        
        if self.result.is_some() {
            return;
        }
        let account_id = env::predecessor_account_id();
        let account_stake = if is_vote {
            let stake = 10;
            //COMMENTED: let stake = env::validator_stake(&account_id);
            assert!(stake > 0, "{} is not a validator", account_id);
            stake
        } else {
            0
        };
        let voted_stake = 7;
        //COMMENTED: let voted_stake self.votes.remove(&account_id).unwrap_or_default();
        assert!(
            voted_stake <= self.total_voted_stake,
            "invariant: voted stake {} is more than total voted stake {}",
            voted_stake,
            self.total_voted_stake
        );
        self.total_voted_stake = self.total_voted_stake + account_stake - voted_stake;
        if account_stake > 0 {
            self.votes.insert(account_id, account_stake);
            self.check_result();
        }
    }
```
NOTE: You do not need to write the parts into code again, these are just explanations.
Part1: Checking for the result. This particular code snippet checks if the result of the voting exists i.e has the voting process been completed.
Think of _is_some_ as literally - is some value there?
We return from the function if a result for the voting already exists.

```
if self.result.is_some() {
            return;
}
```

Let's move to Part 2 in the next quest.


## Function to vote or withdraw the vote - Part2

Refer to the entire function from the previous quest. We will explore the second part of the function.

Here is the code snippet:

```
let account_id = env::predecessor_account_id();
        let account_stake = if is_vote {
            let stake = 10;
            //COMMENTED: let stake = env::validator_stake(&account_id);
            assert!(stake > 0, "{} is not a validator", account_id);
            stake
        } else {
            0
        };
```

The above code snippet does the following things:
* Gets the account id of the previous account caller. 
* The **//COMMENTED** line is the actual line of code that gets the stake of the account, who is a validator. In our case, we have substituted this with a value of 10, just to let the functions pass while testing them from CLI.
* If the stake returned is not greater than 0, it throws an error that the account calling this function is not a validator.

There is an initial check if the validator has decided to vote or no.
This is quite simple right? Let us move to the next part.

# Function to vote or withdraw the vote - Part3

In the third part of the function, if the validator has voted - we will check if the value of the voted stake.
If by adding the votes of this validator, the total stake for result has been met, the check result is called to check the results.
As per the problem statement: Once the majority of the stakeholder vote, the transfers would be unlocked.

Here is the part-3 code.

```
        let voted_stake = 7;
        //COMMENTED: let voted_stake = self.votes.remove(&account_id).unwrap_or_default();
        assert!(
            voted_stake <= self.total_voted_stake,
            "invariant: voted stake {} is more than total voted stake {}",
            voted_stake,
            self.total_voted_stake
        );
        self.total_voted_stake = self.total_voted_stake + account_stake - voted_stake;
        if account_stake > 0 {
            self.votes.insert(account_id, account_stake);
            self.check_result();
        }
 ```

We have assigned "7" as the _voted_stake_ just for testing purposes. Otherwise, the validators actual votes are obtained from the **votes** map.

Let us take a quick look at one last function and then, the rest of the helper functions(getters/checkers), we would leave it upto you to decode :).

## Function ping to update the votes according to current stake of validators.

Let us take a look at the function ping.
The ping function is used to update the votes according to the current stake of validators.

Here is the code for the function.

```
/// Ping to update the votes according to current stake of validators.
    pub fn ping(&mut self) {
        assert!(self.result.is_none(), "Voting has already ended");
        let cur_epoch_height = env::epoch_height();
        if cur_epoch_height != self.last_epoch_height {
            let votes = std::mem::take(&mut self.votes);
            self.total_voted_stake = 10;
            //COMMENTED self.total_voted_stake = 0;
            for (account_id, _) in votes {
                let account_current_stake = 5;
                //env::validator_stake(&account_id);
                self.total_voted_stake += account_current_stake;
                if account_current_stake > 0 {
                    self.votes.insert(account_id, account_current_stake);
                }
            }
            self.check_result();
            self.last_epoch_height = cur_epoch_height;
        }
    }
```

The above function does the following:
* It checks first if the voting has ended by checking the result object.
* epoch_height gives us the height of the epoch.
* It calculats for the total voted stake based on the account's current stake.
* It adds the current account stake(only if the account has chosen true) to the votes Hashmap.
* Post the calculation, there is a call to check for the result and the epoch height is updated to the current one.

The above function might seem a little intimdating but as you rad through the whole contract once or twice, you will grasp the workflow quite easily.

In the next quest, we will be sharing a few getting/checker functions in the contract that you could add as is.

NOTE: An epoch is usually defined as the period of time it takes for a specific number of blocks to be finalized on the chain

## Adding the getter and checker functions

The first function that we have been using for a while is the check_result.
As the name suggests, this functions checks for the result of the voting process.

Here is the code:
```
/// Check whether the voting has ended.
    fn check_result(&mut self) {
        assert!(
            self.result.is_none(),
            "check result is called after result is already set"
        );
        let total_stake = env::validator_total_stake();
        if self.total_voted_stake > 2 * total_stake / 3 {
            self.result = Some(U64::from(env::block_timestamp()));
        }
    }
```

The next three getter functions are used to get the result, the voted_stake and the votes.

```
/// Get the timestamp of when the voting finishes. `None` means the voting hasn't ended yet.
    pub fn get_result(&self) -> Option<WrappedTimestamp> {
        self.result.clone()
    }

    /// Returns current a pair of `total_voted_stake` and the total stake.
    /// Note: as a view method, it doesn't recompute the active stake. May need to call `ping` to
    /// update the active stake.
    pub fn get_total_voted_stake(&self) -> (U128, U128) {
        (
            self.total_voted_stake.into(),
            env::validator_total_stake().into(),
        )
    }

    /// Returns all active votes.
    /// Note: as a view method, it doesn't recompute the active stake. May need to call `ping` to
    /// update the active stake.
    pub fn get_votes(&self) -> HashMap<AccountId, U128> {
        self.votes
            .iter()
            .map(|(account_id, stake)| (account_id.clone(), (*stake).into()))
            .collect()
    }
```
It is important to note that all the functions up until now need to be added within the VotingContract implementation(impl).

Let us take a quick look at the tests for the contract. We will not be explaining how to write the tests but you can take a quick look at them and add them below the implementation.

## Adding tests to the contract

The following unit tests are used to test the contract code by creating a VMContext that can call the contract functions without actually having the need to deploy the contract on chain.

Here are the tests.

```
#[cfg(not(target_arch = "wasm32"))]
#[cfg(test)]
mod tests {
    use super::*;
    use near_sdk::MockedBlockchain;
    use near_sdk::{testing_env, VMContext};
    use std::collections::HashMap;
    use std::iter::FromIterator;

    fn get_context(predecessor_account_id: AccountId) -> VMContext {
        get_context_with_epoch_height(predecessor_account_id, 0)
    }

    fn get_context_with_epoch_height(
        predecessor_account_id: AccountId,
        epoch_height: EpochHeight,
    ) -> VMContext {
        VMContext {
            current_account_id: "alice_near".to_string(),
            signer_account_id: "bob_near".to_string(),
            signer_account_pk: vec![0, 1, 2],
            predecessor_account_id,
            input: vec![],
            block_index: 0,
            block_timestamp: 0,
            account_balance: 0,
            account_locked_balance: 0,
            storage_usage: 1000,
            attached_deposit: 0,
            prepaid_gas: 2 * 10u64.pow(14),
            random_seed: vec![0, 1, 2],
            is_view: false,
            output_data_receivers: vec![],
            epoch_height,
        }
    }

    #[test]
    #[should_panic(expected = "is not a validator")]
    fn test_nonvalidator_cannot_vote() {
        let context = get_context("bob.near".to_string());
        let validators = HashMap::from_iter(
            vec![
                ("alice_near".to_string(), 100),
                ("bob_near".to_string(), 100),
            ]
            .into_iter(),
        );
        testing_env!(context, Default::default(), Default::default(), validators);
        let mut contract = VotingContract::new();
        contract.vote(true);
    }

    #[test]
    #[should_panic(expected = "Voting has already ended")]
    fn test_vote_again_after_voting_ends() {
        let context = get_context("alice.near".to_string());
        let validators = HashMap::from_iter(vec![("alice.near".to_string(), 100)].into_iter());
        testing_env!(context, Default::default(), Default::default(), validators);
        let mut contract = VotingContract::new();
        contract.vote(true);
        assert!(contract.result.is_some());
        contract.vote(true);
    }

    #[test]
    fn test_voting_simple() {
        let context = get_context("test0".to_string());
        let validators = (0..10)
            .map(|i| (format!("test{}", i), 10))
            .collect::<HashMap<_, _>>();
        testing_env!(
            context,
            Default::default(),
            Default::default(),
            validators.clone()
        );
        let mut contract = VotingContract::new();

        for i in 0..7 {
            let mut context = get_context(format!("test{}", i));
            testing_env!(
                context.clone(),
                Default::default(),
                Default::default(),
                validators.clone()
            );
            contract.vote(true);
            context.is_view = true;
            testing_env!(
                context,
                Default::default(),
                Default::default(),
                validators.clone()
            );
            assert_eq!(
                contract.get_total_voted_stake(),
                (U128::from(10 * (i + 1)), U128::from(100))
            );
            assert_eq!(
                contract.get_votes(),
                (0..=i)
                    .map(|i| (format!("test{}", i), U128::from(10)))
                    .collect::<HashMap<_, _>>()
            );
            assert_eq!(contract.votes.len() as u128, i + 1);
            if i < 6 {
                assert!(contract.result.is_none());
            } else {
                assert!(contract.result.is_some());
            }
        }
    }

    #[test]
    fn test_voting_with_epoch_change() {
        let validators = (0..10)
            .map(|i| (format!("test{}", i), 10))
            .collect::<HashMap<_, _>>();
        let context = get_context("test0".to_string());
        testing_env!(
            context,
            Default::default(),
            Default::default(),
            validators.clone()
        );
        let mut contract = VotingContract::new();

        for i in 0..7 {
            let context = get_context_with_epoch_height(format!("test{}", i), i);
            testing_env!(
                context,
                Default::default(),
                Default::default(),
                validators.clone()
            );
            contract.vote(true);
            assert_eq!(contract.votes.len() as u64, i + 1);
            if i < 6 {
                assert!(contract.result.is_none());
            } else {
                assert!(contract.result.is_some());
            }
        }
    }

    #[test]
    fn test_validator_stake_change() {
        let mut validators = HashMap::from_iter(vec![
            ("test1".to_string(), 40),
            ("test2".to_string(), 10),
            ("test3".to_string(), 10),
        ]);
        let context = get_context_with_epoch_height("test1".to_string(), 1);
        testing_env!(
            context,
            Default::default(),
            Default::default(),
            validators.clone()
        );

        let mut contract = VotingContract::new();
        contract.vote(true);
        validators.insert("test1".to_string(), 50);
        let context = get_context_with_epoch_height("test2".to_string(), 2);
        testing_env!(
            context,
            Default::default(),
            Default::default(),
            validators.clone()
        );
        contract.ping();
        assert!(contract.result.is_some());
    }

    #[test]
    fn test_withdraw_votes() {
        let validators =
            HashMap::from_iter(vec![("test1".to_string(), 10), ("test2".to_string(), 10)]);
        let context = get_context_with_epoch_height("test1".to_string(), 1);
        testing_env!(
            context,
            Default::default(),
            Default::default(),
            validators.clone()
        );
        let mut contract = VotingContract::new();
        contract.vote(true);
        assert_eq!(contract.votes.len(), 1);
        let context = get_context_with_epoch_height("test1".to_string(), 2);
        testing_env!(
            context,
            Default::default(),
            Default::default(),
            validators.clone()
        );
        contract.vote(false);
        assert!(contract.votes.is_empty());
    }

    #[test]
    fn test_validator_kick_out() {
        let mut validators = HashMap::from_iter(vec![
            ("test1".to_string(), 40),
            ("test2".to_string(), 10),
            ("test3".to_string(), 10),
        ]);
        let context = get_context_with_epoch_height("test1".to_string(), 1);
        testing_env!(
            context,
            Default::default(),
            Default::default(),
            validators.clone()
        );

        let mut contract = VotingContract::new();
        contract.vote(true);
        assert_eq!((contract.get_total_voted_stake().0).0, 40);
        validators.remove(&"test1".to_string());
        let context = get_context_with_epoch_height("test2".to_string(), 2);
        testing_env!(
            context,
            Default::default(),
            Default::default(),
            validators.clone()
        );
        contract.ping();
        assert_eq!((contract.get_total_voted_stake().0).0, 0);
    }
}
```

Viola! You have your contract up and running now.

You can tun the ./build.sh script to build your contract and use 'near deploy' to deploy your contract to a testnet account of your choice!

You can use the 'near call' command on CLI to call some of the contract functions and check the output for yourself!
Happy Learning!
