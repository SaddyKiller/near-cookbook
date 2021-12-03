# How to cook NEAR
Tutorial for first steps with NEAR cli and api)


Ingredients:
	1 - NEAR account (Wallet)
	2 - NEAR CLI
	3 - NEAR contract
	4 - Basic Web app + NEAR API  

Ingredient 1 - NEAR account

First of all, we need an NEAR account, so follow the link https://wallet.testnet.near.org (it’s a NEAR playground for developers, and called “testnet”), on the page click Create Account and follow the instructions. As a result of preparing ingredient - you get an account name - <username>.testnet, the account is filled with 200 NEAR (it’s a testNEAR for test purposes).
When you done - proceed to ingredient 2 - NEAR CLI. 

Ingredient 2 - NEAR CLI

To get it - you need to install NodeJS and NPM (node package manages):
	I’ll show you 2 ways: 
	1. Use pre-built installer for your platform:  https://nodejs.org/en/download/
	2. Install via package manager: https://nodejs.org/en/download/package-manager/Then install via npm “near-cli”:
	npm install -g near-cli

Now, test installation by running command: near login, as a result - you system opens webpage where you can complete login procedure with you NEAR account (Wallet). 

Ingredient 3 - NEAR contract

To prepare that ingredient you need:
	1. Create sub account
	2. Prepare environment for building contract
	3. Create contract sources 
	4. Build contract and Deploy contract to sub account you created at step 1.
	5. Test deployed contract.

	1. Create sub account - do do this we use near-cli command:
near create-account <subaccname>.<username>.testnet --masterAccount <username>.testnet  --initialBalance <numberOfNear>
	explains the params - 
	subaccname - any name you want to call you sub account, use lowercase.
	username - it’s a first ingredient - name of your NEAR account
	numberOfNear - it’s number of near you want to send from your main account to sub

	
You can add you sub account name to enviroment variable by running command: 
export SUBACC= <subaccname>.<username>.testnet
and use $SUBACC instead of <subaccname>.<username>.testnet
but we continue use <subaccname>.<username>.testnet

	2. Prepairing enviroment - building contract use Rust and WebAssembly - here some docs (https://www.rust-lang.org/tools/install, https://rustwasm.github.io/docs/book/):
	Install Rustup with command - curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
	Add shell environments with command - source $HOME/.cargo/env
	Add wasm (WebAssembly) target to your toolchain - rustup target add wasm32-unknown-unknown
	3. Create contract sources - 
	create project folder with structure:		.
├ Cargo.toml
└ src
   └ lib.rs

	Now feel them with NEAR tutorial sources (https://docs.near.org/docs/develop/contracts/rust/intro):
cargo.toml 
```[package]
name = "rust-counter-tutorial"
version = "0.1.0"
authors = ["NEAR Inc <hello@near.org>"]
edition = "2018"

[lib]
crate-type = ["cdylib", "rlib"]

[dependencies]
near-sdk = "3.1.0"

[profile.release]
codegen-units = 1
# Tell `rustc` to optimize for small code size.
opt-level = "z"
lto = true
debug = false
panic = "abort"
# Opt into extra safety checks on arithmetic operations https://stackoverflow.com/a/64136471/249801
overflow-checks = true```

lib.rs

```//! This contract implements simple counter backed by storage on blockchain.
//!
//! The contract provides methods to [increment] / [decrement] counter and
//! [get it's current value][get_num] or [reset].
//!
//! [increment]: struct.Counter.html#method.increment
//! [decrement]: struct.Counter.html#method.decrement
//! [get_num]: struct.Counter.html#method.get_num
//! [reset]: struct.Counter.html#method.reset

use near_sdk::borsh::{self, BorshDeserialize, BorshSerialize};
use near_sdk::{env, near_bindgen};

near_sdk::setup_alloc!();

// add the following attributes to prepare your code for serialization and invocation on the blockchain
// More built-in Rust attributes here: https://doc.rust-lang.org/reference/attributes.html#built-in-attributes-index
#[near_bindgen]
#[derive(Default, BorshDeserialize, BorshSerialize)]
pub struct Counter {
    // See more data types at https://doc.rust-lang.org/book/ch03-02-data-types.html
    val: i8, // i8 is signed. unsigned integers are also available: u8, u16, u32, u64, u128
}

#[near_bindgen]
impl Counter {
    /// Returns 8-bit signed integer of the counter value.
    ///
    /// This must match the type from our struct's 'val' defined above.
    ///
    /// Note, the parameter is `&self` (without being mutable) meaning it doesn't modify state.
    /// In the frontend (/src/main.js) this is added to the "viewMethods" array
    /// using near-cli we can call this by:
    ///
    /// ```bash
    /// near view counter.YOU.testnet get_num
    /// ```
    pub fn get_num(&self) -> i8 {
        return self.val;
    }

    /// Increment the counter.
    ///
    /// Note, the parameter is "&mut self" as this function modifies state.
    /// In the frontend (/src/main.js) this is added to the "changeMethods" array
    /// using near-cli we can call this by:
    ///
    /// ```bash
    /// near call counter.YOU.testnet increment --accountId donation.YOU.testnet
    /// ```
    pub fn increment(&mut self) {
        // note: adding one like this is an easy way to accidentally overflow
        // real smart contracts will want to have safety checks
        // e.g. self.val = i8::wrapping_add(self.val, 1);
        // https://doc.rust-lang.org/std/primitive.i8.html#method.wrapping_add
        self.val += 1;
        let log_message = format!("Increased number to {}", self.val);
        env::log(log_message.as_bytes());
        after_counter_change();
    }

    /// Decrement (subtract from) the counter.
    ///
    /// In (/src/main.js) this is also added to the "changeMethods" array
    /// using near-cli we can call this by:
    ///
    /// ```bash
    /// near call counter.YOU.testnet decrement --accountId donation.YOU.testnet
    /// ```
    pub fn decrement(&mut self) {
        // note: subtracting one like this is an easy way to accidentally overflow
        // real smart contracts will want to have safety checks
        // e.g. self.val = i8::wrapping_sub(self.val, 1);
        // https://doc.rust-lang.org/std/primitive.i8.html#method.wrapping_sub
        self.val -= 1;
        let log_message = format!("Decreased number to {}", self.val);
        env::log(log_message.as_bytes());
        after_counter_change();
    }

    /// Reset to zero.
    pub fn reset(&mut self) {
        self.val = 0;
        // Another way to log is to cast a string into bytes, hence "b" below:
        env::log(b"Reset counter to zero");
    }
}

// unlike the struct's functions above, this function cannot use attributes #[derive(…)] or #[near_bindgen]
// any attempts will throw helpful warnings upon 'cargo build'
// while this function cannot be invoked directly on the blockchain, it can be called from an invoked function
fn after_counter_change() {
    // show helpful warning that i8 (8-bit signed integer) will overflow above 127 or below -128
    env::log("Make sure you don't overflow, my friend.".as_bytes());
}

/*
 * the rest of this file sets up unit tests
 * to run these, the command will be:
 * cargo test --package rust-counter-tutorial -- --nocapture
 * Note: 'rust-counter-tutorial' comes from cargo.toml's 'name' key
 */

// use the attribute below for unit tests
#[cfg(test)]
mod tests {
    use super::*;
    use near_sdk::MockedBlockchain;
    use near_sdk::{testing_env, VMContext};

    // part of writing unit tests is setting up a mock context
    // in this example, this is only needed for env::log in the contract
    // this is also a useful list to peek at when wondering what's available in env::*
    fn get_context(input: Vec<u8>, is_view: bool) -> VMContext {
        VMContext {
            current_account_id: "alice.testnet".to_string(),
            signer_account_id: "robert.testnet".to_string(),
            signer_account_pk: vec![0, 1, 2],
            predecessor_account_id: "jane.testnet".to_string(),
            input,
            block_index: 0,
            block_timestamp: 0,
            account_balance: 0,
            account_locked_balance: 0,
            storage_usage: 0,
            attached_deposit: 0,
            prepaid_gas: 10u64.pow(18),
            random_seed: vec![0, 1, 2],
            is_view,
            output_data_receivers: vec![],
            epoch_height: 19,
        }
    }

    // mark individual unit tests with #[test] for them to be registered and fired
    #[test]
    fn increment() {
        // set up the mock context into the testing environment
        let context = get_context(vec![], false);
        testing_env!(context);
        // instantiate a contract variable with the counter at zero
        let mut contract = Counter { val: 0 };
        contract.increment();
        println!("Value after increment: {}", contract.get_num());
        // confirm that we received 1 when calling get_num
        assert_eq!(1, contract.get_num());
    }

    #[test]
    fn decrement() {
        let context = get_context(vec![], false);
        testing_env!(context);
        let mut contract = Counter { val: 0 };
        contract.decrement();
        println!("Value after decrement: {}", contract.get_num());
        // confirm that we received -1 when calling get_num
        assert_eq!(-1, contract.get_num());
    }

    #[test]
    fn increment_and_reset() {
        let context = get_context(vec![], false);
        testing_env!(context);
        let mut contract = Counter { val: 0 };
        contract.increment();
        contract.reset();
        println!("Value after reset: {}", contract.get_num());
        // confirm that we received -1 when calling get_num
        assert_eq!(0, contract.get_num());
    }
}```

4. Build contract with  running command inside a project directory: 	cargo build --target wasm32-unknown-unknown —release
	project compiled at - <projectdirectory>/target/wasm32-unknown-unknown/release/rust_counter_tutorial.wasm
	- rust_counter_tutorial - it’s a name of the project from Cargo.toml, so be careful, at next step
	Deploying contract  - command: 

	near deploy --wasmFile <compiledContract> --accountId <subaccname>.<username>.testnet

    <compiledContract> - relative path to the file from previous step:
	    target/wasm32-unknown-unknown/release/rust_counter_tutorial.wasm
    <subaccname>.<username>.testnet - name of the sub account we created earlier.


5. Test deployed contract by running commands:

Notice - when you run command of contract that - changing some data in contract - you need to run near call, else near view
	near call <subaccname>.<username>.testnet increment --accountId <subaccname>.<username>.testnet
	near call <subaccname>.<username>.testnet decrement --accountId <subaccname>.<username>.testnet

	near view <subaccname>.<username>.testnet get_num --accountId <subaccname>.<username>.testnet

Ingredient 4 - Basic Web app + NEAR API 

     For basic web app we use react. To create template do:
        Install create-react-app, by command - npm i -g create-react-app
        Create new project with - create-react-app <projectname>
            projectname - its project and folder name where it create it.
    Open project folder by command: cd ./<projectname>

    Add near-api-js to dependencies
    by command npm i near-api-js

    Replace ./src/index.js file content with:

import React from 'react';
import ReactDOM from 'react-dom';
import './index.css';
import App from './App';

import * as nearAPI from 'near-api-js';

const CONTRACT_NAME = process.env.CONTRACT_NAME || 'subaccname.saddykiller.testnet';

const contractConfig = {
  networkId: 'testnet',
  nodeUrl: 'https://rpc.testnet.near.org',
  contractName: CONTRACT_NAME,
  walletUrl: 'https://wallet.testnet.near.org',
  helperUrl: 'https://helper.testnet.near.org'
};

// Initializing contract
async function initContract() {
  // get network configuration values from config.js
  // based on the network ID we pass to getConfig()
  const nearConfig = contractConfig;

  // create a keyStore for signing transactions using the user's key
  // which is located in the browser local storage after user logs in
  const keyStore = new nearAPI.keyStores.BrowserLocalStorageKeyStore();

  // Initializing connection to the NEAR testnet
  const near = await nearAPI.connect({ keyStore, ...nearConfig });

  // Initialize wallet connection
  const walletConnection = new nearAPI.WalletConnection(near);

  // Load in user's account data
  let currentUser;
  if (walletConnection.getAccountId()) {
    currentUser = {
      // Gets the accountId as a string
      accountId: walletConnection.getAccountId(),
      // Gets the user's token balance
      balance: (await walletConnection.account().state()).amount,
    };
  }

  // Initializing our contract APIs by contract name and configuration
  const contract = await new nearAPI.Contract(
    // User's accountId as a string
    walletConnection.account(),
    // accountId of the contract we will be loading
    // NOTE: All contracts on NEAR are deployed to an account and
    // accounts can only have one contract deployed to them. 
    nearConfig.contractName,
    {
      // View methods are read-only – they don't modify the state, but usually return some value
      // Change methods can modify the state, but you don't receive the returned value when called
      viewMethods:['get_num'],
      changeMethods: ['increment', 'decrement'],
      // Sender is the account ID to initialize transactions.
      // getAccountId() will return empty string if user is still unauthorized
      sender: walletConnection.getAccountId(),
    }
  );

  return { contract, currentUser, nearConfig, walletConnection };
}

window.nearInitPromise = initContract().then(
  ({ contract, currentUser, nearConfig, walletConnection }) => {
    ReactDOM.render(
      <App
        contract={contract}
        currentUser={currentUser}
        nearConfig={nearConfig}
        wallet={walletConnection}
      />,
      document.getElementById('root')
    );
  }
);

- here we add a contract init and define methods that it has, divided the in 2 groups as with near-cli call and view methods:
     
     nearConfig.contractName,
    {
      // View methods are read-only – they don't modify the state, but usually return some value
      // Change methods can modify the state, but you don't receive the returned value when called
      viewMethods:['get_num'],
      changeMethods: ['increment', 'decrement'],
      // Sender is the account ID to initialize transactions.
      // getAccountId() will return empty string if user is still unauthorized
      sender: walletConnection.getAccountId(),
    }


and replace content of src/App.js with 

import logo from './logo.svg';
import './App.css';
import Big from "big.js";
import React, { useState } from "react";

const BOATLOAD_OF_GAS = Big(3)
  .times(10 ** 13)
  .toFixed();

function App({ contract, currentUser, nearConfig, wallet }) {

  const [counter, setCounter] = useState(0);
  
  const getNum = () => {
    if (!currentUser) return;
    return contract
      .get_num({
        account_id: currentUser.accountId,
      })
      .then(num => {
        setCounter(num)
      })
      .catch((e) => {
        console.error(e);
      });
  };

  const incNum = () => {
    if (!currentUser) return;
    return contract
      .increment({
        account_id: currentUser.accountId,
      })
      .then(() => getNum())
      .catch((e) => {
        console.error(e);
      });
  };

  const decNum = () => {
    if (!currentUser) return;
    return contract
      .decrement({
        account_id: currentUser.accountId,
      })
      .then(() => getNum())
      .catch((e) => {
        console.error(e);
      });
  };

  const signIn = () => {
    wallet
      .requestSignIn(nearConfig.contractName, "Vote! It's your voice!")
      .then((res) =>
        contract.storage_deposit(
          {
            account_id: currentUser.accountId,
          },
          BOATLOAD_OF_GAS,
          Big("0.00125")
            .times(10 ** 24)
            .toFixed()
        )
      )
      .catch((e) => console.log(e));
  };

  const signOut = () => {
    wallet.signOut();
    window.location.replace(window.location.origin + window.location.pathname);
  };

  return (
    <div className="App">
      <header className="App-header">
        <img src={logo} className="App-logo" alt="logo" />
        {currentUser ? (
          <button className="App-link" onClick={signOut}>Log out</button>
        ) : (
          <>
            <h3>Login to vote!</h3>
            <button onClick={signIn}>Log in</button>
          </>
        )}     

        <p>
          Current counter {counter}
        </p>
        <button onClick={getNum}>Get</button>
        <button onClick={incNum}>Increment</button>
        <button onClick={decNum}>Decrement</button>
        
      </header>
    </div>
  );
}

export default App;


then run command - npm run start, now you can interact with you contract via WEB)))
Thank you for trying my receipt of NEAR cooking!
Now we are DONE, ENJOY NEAR and have a nice day!
