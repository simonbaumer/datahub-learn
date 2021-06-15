# 6. Deploy a program

A _program_ is to Solana what a _smart contract_ is to other protocols. Once a program has been deployed, any app can interact with it by sending a transaction to a Solana cluster that will pass it to the program.

Solana programs can be written in C or in Rust. You can learn more about Solana's programs [here](https://docs.solana.com/developing/on-chain-programs/overview).

So far we've been using Solana's JS API to interact with the blockchain. In this chapter we're going to deploy a Solana program using another Solana developer tool: their CLI. We'll install it and use it through our terminal.

## Install Rust and configure the Solana CLI

* Install the latest Rust stable from [https://rustup.rs/](https://rustup.rs/)
* Install Solana v1.6.6 or later from [https://docs.solana.com/cli/install-solana-cli-tools](https://docs.solana.com/cli/install-solana-cli-tools)

Set the CLI config url to the devnet cluster by running this command in your Terminal:

```text
solana config set --url devnet
```

If this is your first time using the Solana CLI, you will need to generate a new keypair. It will be written to `~/.config/solana/id.json` and will be used every time you use the CLI. Run the following command in your Terminal:

```text
solana-keygen new
```

## Understanding the hello-world program

There is a `program` folder at the app's root. It contains the Rust program `src/lib.rs` and some configuration files to help us compile and deploy it.

**It's a "dummy" program: all it does is increment a number every time it's called.**

Let’s disect what each part does.

```rust
use byteorder::{ByteOrder, LittleEndian};
use solana_program::{
    account_info::{next_account_info, AccountInfo},
    entrypoint,
    entrypoint::ProgramResult,
    msg,
    program_error::ProgramError,
    pubkey::Pubkey,
};
use std::mem;

entrypoint!(process_instruction);
```

We start by doing some standard includes and declaring an entry point which will be the process\_instruction function:

```rust
fn process_instruction(
    program_id: &Pubkey,
    accounts: &[AccountInfo],
    _instruction_data: &[u8],
```

The program\_id is the public key where the contract is stored and the accountInfo is the appAccount to say hello to.

```rust
) -> ProgramResult {
    msg!("Helloworld Rust program entrypoint");
    let accounts_iter = &mut accounts.iter();
    let account = next_account_info(accounts_iter)?;
```

The ProgramResult is where the magic happens and we start by printing a fairly pointless message and then select the accountInfo by looping through although in practice there is likely only one.

```rust
if account.owner != program_id {
  msg!("Greeted account does not have the correct program id");
  return Err(ProgramError::IncorrectProgramId);
}
```

Security check to see if the account owner has permission.

```rust
if account.try_data_len()? < mem::size_of::<u32>() {
  msg!("Greeted account data length too small for u32");
  return Err(ProgramError::InvalidAccountData);
}
```

Check to see if data is available to store a u32 integar.

```rust
let mut data = account.try_borrow_mut_data()?;
let mut num_greets = LittleEndian::read_u32(&data);
num_greets += 1;
LittleEndian::write_u32(&mut data[0..], num_greets);
msg!("Hello!");
Ok(())
```

Finally we get to the good stuff where we “borrow” the existing appAccount data, increase that value by one and rewrite it back.

## Building the program

The first thing we're going to do is compile the Rust program to prepare it for the CLI. To do this we're going to use a custom script that's in `package.json`:

```rust
"build:program-rust": "cargo build-bpf --manifest-path=program/Cargo.toml --bpf-out-dir=dist/program"
```

* `cargo` is Rust’s build system and package manager \([docs](https://doc.rust-lang.org/book/ch01-03-hello-cargo.html)\), like what `npm` is to Javascript.
* `build-bpf` is the cargo command we're going to run to build/compile the program.
* We pass it a manifest file and the desired output

Let's build the program by running the following command in your Terminal:

```rust
$ npm run build:program-rust
```

{% hint style="warning" %}
This step can take 5 or 10 minutes!
{% endhint %}

If it's successful you should see a new folder in your app which contains the compiled contract: `hellow-world.so`.

{% hint style="info" %}
The `.so` extension does not stand for Solana! It stands for "shared object". The helloworld program we wrote is a Rust program compiled to Berkley Packet Format \(BPF\) and stored as an Executable and Linkable Format \(ELF\) shared object.

You can read more about Solana Programs [here](https://docs.solana.com/developing/on-chain-programs/overview).
{% endhint %}

## Deploying the program

Next we're going to deploy the program to the devnet cluster. The CLI provides a very simple interface for this:

```bash
solana program deploy dist/program/helloworld.so
```

The first time you run this, the CLI should error out because the account trying to deploy this program **does not have enough funds to spend**. You can fix this by going back a few steps in the React app to the "Fund" step and pasting in the input the public key that you see in the Terminal error. Click "Fund" a few times to get enough devnet tokens to be able to deploy the program

![](../../../.gitbook/assets/screen-shot-2021-06-14-at-8.20.30-pm.png)

If you see

> Error: Deploying program failed: Error processing Instruction 1: custom program error: 0x1

simply re-run the deploy command until it succeeds. If you run out of funds, go back to the "Fund" step and get more!

On success, the CLI will print the programId of the deployed contract.

```bash
Program Id: DwpsLw56wmAr3FMZiWHHK47vwZx9LYreT9r32Sn9tBZ5
```

If you visit [https://explorer.solana.com](https://explorer.solana.com/), change the cluster to devnet and paste this Program Id string, you should see a page like this:

{% hint style="warning" %}
Make sure you select Devnet on the Solana Explorer!
{% endhint %}

![](../../../.gitbook/assets/screen-shot-2021-06-14-at-3.52.29-pm.png)

Notice that the field `Executable` is now set to `Yes` because the address we're looking up is for a program which can be called and executed.

## Save the program and its author secret keys

Before we move to the next step we need to save two important variables

1. In your terminal run `cat ~/.config/solana/id.json` and copy its output. In `src/components/Call.jsx` assign it to the constant `PAYER_SECRET_KEY`. This is the Keypair information of the author of the program \(you!\). We will need to pass it to transactions we make to the program  to authenticate ourselves as the owner of the program.
2. In the directory, find `dist/program/helloworld-keypair.json` and copy its content. In `Program.jsx` assign it to the constant `PROGRAM_SECRET_KEY`. This is the Keypair information of the  program itself. We will need it to generate the program's public key that will be used to call the program.

## Next

So at this point, we've deployed our dummy smart contract to Solana's devnet cluster. We're finally ready for the big moment: interacting with the program by calling from the UI!
