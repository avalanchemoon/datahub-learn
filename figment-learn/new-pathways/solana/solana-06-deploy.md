A **program** is to Solana what a **smart contract** is to other protocols. Once a program has been deployed, any app can interact with it by sending a transaction containing the program instructions to a Solana cluster, which will pass it to the program to be run.

{% hint style="info" %}
[You can learn more about Solana's programs here](https://docs.solana.com/developing/on-chain-programs/overview).
{% endhint %}

# Smart contract review

There is a `program` folder at the app's root. It contains the Rust program `contracts/solana/program/src/lib.rs` and some configuration files to help us compile and deploy it.

**It's a simple program, all it does is increment a number every time it's called.**

Let’s dissect what each part does.

```rust
use borsh::{BorshDeserialize, BorshSerialize};
use solana_program::{
    account_info::{next_account_info, AccountInfo},
    entrypoint,
    entrypoint::ProgramResult,
    msg,
    program_error::ProgramError,
    pubkey::Pubkey,
};
```

[`use` declarations](https://doc.rust-lang.org/reference/items/use-declarations.html) are convenient shortcuts to other code. In this case, the serialize and de-serialize functions from the [borsh](https://borsh.io/) crate. borsh stands for _**B**inary **O**bject **R**epresentation **S**erializer for **H**ashing_.  
A [crate](https://learning-rust.github.io/docs/a4.cargo,crates_and_basic_project_structure.html#Crate) is a collection of source code which can be distributed and compiled together. Learn more about [Cargo, Crates and basic project structure](https://learning-rust.github.io/docs/a4.cargo,crates_and_basic_project_structure.html).

We also `use` portions of the `solana_program` crate :

* A function to return the next `AccountInfo` as well as the  struct for `AccountInfo` ;
* The `entrypoint` macro and related `entrypoint::ProgramResult` ;
* The `msg` macro, for low-impact logging on the blockchain ;
* `program_error::ProgramError` which allows on-chain programs to implement program-specific error types and see them returned by the Solana runtime. A program-specific error may be any type that is represented as or serialized to a u32 integer ;
* The `pubkey::Pubkey` struct.

Next we will use the `derive` macro to generate all the necessary boilerplate code to wrap our `GreetingAccount` struct. This happens behind the scenes during compile time [with any `#[derive()]` macros](https://doc.rust-lang.org/reference/procedural-macros.html#derive-macros). Rust macros are a rather large topic to take in, but well worth the effort to understand. For now, just know that this is a shortcut for boilerplate code that is inserted at compile time.

The struct declaration itself is simple, we are using the `pub` keyword to declare our struct publicly accessible, meaning other programs and functions can use it. The `struct` keyword is letting the compiler know that we are defining a struct named `GreetingAccount` , which has a single field : `counter` with a type of `u32` , an unsigned 32-bit integer. This means our counter cannot be larger than [`4,294,967,295`](https://en.wikipedia.org/wiki/4,294,967,295).

```rust
#[derive(BorshSerialize, BorshDeserialize, Debug)]
pub struct GreetingAccount {
    pub counter: u32,
}
```

Next, we declare an entry point - the `process_instruction` function :

```rust
entrypoint!(process_instruction);

pub fn process_instruction(
    program_id: &Pubkey, 
    accounts: &[AccountInfo],
    _instruction_data: &[u8],
) -> ProgramResult {
    msg!("Hello World Rust program entrypoint");
    let accounts_iter = &mut accounts.iter();
    let account = next_account_info(accounts_iter)?;
```

* The return value of the `process_instruction` entrypoint will be a `ProgramResult` .  
[`Result`](https://doc.rust-lang.org/std/result/enum.Result.html) comes from the `std` crate and is used to express the possibility of error.
* For [debugging](https://docs.solana.com/developing/on-chain-programs/debugging), we can print messages to the Program Log [with the `msg!()` macro](https://docs.rs/solana-program/1.7.3/solana_program/macro.msg.html), rather than use `println!()` which would be prohibitive in terms of computational cost for the network.
* The `let` keyword in Rust binds a value to a variable. By looping through the `accounts` using an [iterator](https://doc.rust-lang.org/book/ch13-02-iterators.html), `accounts_iter` is taking a [mutable reference](https://doc.rust-lang.org/book/ch04-02-references-and-borrowing.html#mutable-references) of each value in `accounts`. Then `next_account_info(accounts_iter)?`will return the next `AccountInfo` or a `NotEnoughAccountKeys` error. Notice the `?` at the end, this is a [shortcut expression](https://doc.rust-lang.org/std/result/#the-question-mark-operator-) in Rust for [error propagation](https://doc.rust-lang.org/book/ch09-02-recoverable-errors-with-result.html#propagating-errors).

```rust
if account.owner != program_id {
  msg!("Greeted account does not have the correct program id");
  return Err(ProgramError::IncorrectProgramId);
}
```

We will perform a security check to see if the account owner has permission. If the `account.owner` public key does not equal the `program_id` we will return an error.

```rust
let mut greeting_account = GreetingAccount::try_from_slice(&account.data.borrow())?;
greeting_account.counter += 1;
greeting_account.serialize(&mut &mut account.data.borrow_mut()[..])?;

msg!("Greeted {} time(s)!", greeting_account.counter);

Ok(())
```

Finally we get to the good stuff where we "borrow" the existing account data, increase the value of `counter` by one and write it back to storage.

* The `GreetingAccount` struct has only one field - `counter`. To be able to modify it, we need to borrow the reference to `account.data` with the `&`[borrow operator](https://doc.rust-lang.org/reference/expressions/operator-expr.html#borrow-operators). 
* The `try_from_slice()` function from `BorshDeserialize`will mutably reference and deserialize the `account.data`. 
* The `borrow()` function comes from the Rust core library, and exists to immutably borrow the wrapped value. 

Taken together, this is saying that we will borrow the account data and pass it to a function that will deserialize it and return an error if one occurs. Recall that `?` is for error propagation.

Next, incrementing the value of `counter` by `1` is simple, using the addition assignment operator : `+=` .

With the `serialize()` function from `BorshSerialize`, the new `counter` value is sent back to Solana in the correct format. The mechanism by which this occurs is the [Write trait](https://doc.rust-lang.org/std/io/trait.Write.html) from the `std::io` crate.

We can then show in the Program Log how many times the count has been incremented by using the `msg!()` macro.

-----------------------------------------

# Set up the Solana CLI

## Install Rust and Solana CLI

So far we've been using Solana's JS API to interact with the blockchain. In this chapter we're going to deploy a Solana program using another Solana developer tool: their Command Line Interface (CLI). We'll install it and use it through our Terminal.

For simplicity, perform both of these installations inside the project root:

[**Install the latest Rust stable**](https://rustup.rs) : 

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

[**Install Solana CLI**](https://docs.solana.com/cli/install-solana-cli-tools) v1.6.6 or later :

```bash
sh -c "$(curl -sSfL https://release.solana.com/stable/install)"
```

-----------------------------------------

## Set up Solana CLI

We need to configure the Solana cluster, create an account, request an airdrop and check that everything is functioning properly.

Set the CLI config URL to the devnet cluster:

```bash
solana config set --url https://api.devnet.solana.com
```

Next, we're going to generate a new keypair using the CLI. Run the following command in your Terminal:

```bash
mkdir solana-wallet
solana-keygen new --outfile solana-wallet/keypair.json
```

You will need **SOL** available in the account to deploy the program, so get an airdrop with:

```bash
solana airdrop 5 $(solana-keygen pubkey solana-wallet/keypair.json)
```

Verify that everything is ok:

```bash
solana config get
solana account $(solana-keygen pubkey solana-wallet/keypair.json)
```

![](../../../.gitbook/assets/solana-deploy-01-v3.gif)

-----------------------------------------
# Deploy a Solana program

{% hint style="info" %}
The program we're going to deploy is an easy but pretty complete program. This program keeps track of the number of times an account has sent a greeting instruction to it.
{% endhint %}


## Building the program

The first thing we're going to do is compile the Rust program to prepare it for the CLI. To do this we're going to use a custom script that's defined in `package.json`. Let's run the script and build the program by running the following command in the terminal (from the project root directory):

{% hint style="warning" %}
This step can take a few minutes!
{% endhint %}

```bash
yarn run solana:build:program
```

When it's successful, you will see the instructions to execute the deploy command with the path to the compiled program named `helloworld.so`. While this would work, we want to specify the keypair we generated just for this purpose, so read on.

```bash
To deploy this program:
  $ solana program deploy /home/zu/project/figment/learn-web3-dapp/dist/solana/program/helloworld.so
Done in 1.39s.
```

{% hint style="info" %}
The `.so` extension does not stand for Solana! It stands for "shared object". You can learn more about Solana Programs in the [developer documentation](https://docs.solana.com/developing/on-chain-programs/overview).
{% endhint %}

-----------------------------------------

## Deploying the program

Now we're going to deploy the program to the devnet cluster. The CLI provides a very simple interface for this :

```bash
solana deploy -v --keypair solana-wallet/keypair.json dist/solana/program/helloworld.so 
```

{% hint style="info" %}
The `-v` Verbose flag is optional, but it will show some related information like the RPC URL and path to the default signer keypair, as well as the expected [**Commitment level**](https://docs.solana.com/implemented-proposals/commitment). When the process completes, the Program Id will be displayed :
{% endhint %}

On success, the CLI will print the **programId** of the deployed contract.

```bash
RPC URL: https://api.devnet.solana.com
Default Signer Path: solana-wallet/keypair.json
Commitment: confirmed
Program Id: 7KwpCaaYXRsjfCTvf85eCVuZDW894zZNN38UMxMpQoaQ
```

![](../../../.gitbook/assets/solana-deploy-02-v3.gif)


-----------------------------------------

# Challenge

{% hint style="tip" %}
Before moving to the next step, we need to check that our program has been correctly deployed! For this, we'll need the `programId` of the program. Copy & paste it into the text input, then try to figure out how to complete the code for `pages/api/solana/checkProgram.ts`.
{% endhint %}

**Take a few minutes to figure this out.**

```tsx
//...
  // Re-create publicKeys from params
  const publicKey = undefined;
  const programInfo = undefined;

  if (programInfo === null) {
      if (fs.existsSync(PROGRAM_SO_PATH)) {
          throw new Error(
            'Program needs to be deployed with `solana program deploy`',
          );
      } else {
        throw new Error('Program needs to be built and deployed');
      }
  } else if (!programInfo.executable) {
    throw new Error(`Program is not executable`);
  }

  res.status(200).json(true);
//...
```

**Need some help?** Check out those two links
* [How to get account Info ?](https://solana-labs.github.io/solana-web3.js/classes/Connection.html#getAccountInfo)  
* [Is an account executable ?](https://solana-labs.github.io/solana-web3.js/modules.html#AccountInfo)

{% hint style="info" %}
You can [**join us on Discord**](https://discord.gg/fszyM7K), if you have questions or want help completing the tutorial.
{% endhint %}

Still not sure how to do this? No problem! The solution is below so you don't get stuck.

----------------------------------

# The solution

```tsx
//...
  const publicKey = new PublicKey(programId);
  const programInfo = await connection.getAccountInfo(publicKey);

  if (programInfo === null) {
      if (fs.existsSync(PROGRAM_SO_PATH)) {
          throw new Error(
            'Program needs to be deployed with `solana program deploy`',
          );
      } else {
        throw new Error('Program needs to be built and deployed');
      }
  } else if (!programInfo.executable) {
    throw new Error(`Program is not executable`);
  }

  res.status(200).json(true);
//...
```

**What happened in the code above?**

* We create a new `PublicKey` instance from the `programId` string formatted address.
* Once we have it, we call the `getAccountInfo` method to check if info is available for this address.
  * If none, then no account is linked to this address, meaning the program has not yet been deployed.
* Then we check if the account's executable property is true. If it is, then the specified account contains a loaded program.
* Finally, we send a value of `true` to the client-side in JSON format.

----------------------------------

# Make sure it works

Once you have the code above saved:
* Copy and paste the generated address in the text input.   
* Click on **Check ProgramId** 

![](../../../.gitbook/assets/pathways/solana/solana-deploy.gif)

For the rest of the challenge we'll keep this programId in the localStorage of our application.

----------------------------------

# Conclusion

So at this point, we've deployed our program to Solana's devnet cluster and checked that it went smoothly. Now it's time to create an account that is owned by our program, to store some data on the Solana cluster! 
