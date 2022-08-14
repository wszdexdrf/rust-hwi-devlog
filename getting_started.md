# Getting started with rust-hwi

## About

[bitcoindevkit/rust-hwi](https://github.com/bitcoindevkit/rust-hwi) is a sub-project for [bitcoindevkit](https://bitcoindevkit.org/) (BDK) which is used to interact with hardware wallets using the Rust programming language. It is a wrapper around [bitcoin-core/HWI](https://github.com/bitcoin-core/HWI) and it's behaviour is closely linked with the same.

## Fundamentals

As mentioned before, rust-hwi is a wrapper around bitcoin-core/HWI. The functions in it, when called, pass on the arguements to related functions in bitcoin-core/HWI. More information about the functions and their arguements is available on rust-hwi [docs](https://docs.rs/hwi/latest/hwi/index.html) and bitcoin-core/HWI [docs](https://hwi.readthedocs.io/en/stable/).

rust-hwi uses `PyO3` to call the Python functions from Rust. Let us take an example from the documentation:

```rust
use bitcoin::util::bip32::{ChildNumber, DerivationPath};
use hwi::error::Error;
use hwi::interface::HWIClient;
use hwi::types;
use std::str::FromStr;

fn main() -> Result<(), Error> {
    // Find information about devices
    let devices = HWIClient::enumerate()?;
    let device = devices.first().expect("No devices");
    // Create a client for a device
    let client = HWIClient::get_client(&device, true, types::HWIChain::Test)?;
    // Display the address from path
    let derivation_path = DerivationPath::from_str("m/44'/1'/0'/0/0")?;
    let hwi_address =
        client.display_address_with_path(&derivation_path, types::HWIAddressType::Tap)?;
    println!("{}", hwi_address.address);
    Ok(())
}
```

In the first line we call `HWIClient::enumerate()`. This function is equivalent to HWI's `enumerate` function, which returns a list of connected devices. Here, `HWIClient::enumerate()` returns a `Vec<HWIDevice>`, where `HWIDevice` is a struct representing a single device and contains all the information related to it, for example fingerprint, path, etc.

Then we store the first device available into `device`, which is straightforward. We then call `HWIClient::get_client()` and pass the reference to device info and the Chain we are going to use that device on. (the boolean is for setting `expert` mode, which allows for some more functions and detailed information. That is implemented in bitcoin-core/HWI, see the [docs](https://hwi.readthedocs.io/en/latest/usage/cli-usage.html#cmdoption-hwi-expert)) There are 4 chains available, as usual, Main, Test, Regtest & Signet.

HWI contains a base class known as `HardwareWalletClient`. All the functions and their arguements are defined in it. Harware Wallet developers create their own implementations of the base class. The function `get_client()` returns a instance of `HWIClient` struct, which contains a reference to the Python object of `HardwareWalletClient` (and also a reference to the Python code of HWI itself). rust-hwi's `HWIClient` tries to mimic the base class `HardwareWalletClient` and thus all the functions used for communicating with a hardware wallet belong to `HWIClient`.

In the next line we generate a derivation path and use the client instance we created to get a address for the aforementioned path. We then print out the address, and return a `Ok`.

## Integration with BDK

BDK is an amazing project. It is one of the easiest ways to integrate Bitcoin wallet features into any application. rust-hwi aims to help bdk to work with hardware wallets. One of the ways to do so is to implement a Custom Signer.

Let us look at a basic example from BDK's docs:

```rust
use bdk::{FeeRate, Wallet, SyncOptions, SignOptions};
use bdk::database::MemoryDatabase;
use bdk::blockchain::ElectrumBlockchain;
use bdk::electrum_client::Client;

use bdk::wallet::AddressIndex::New;

fn main() -> Result<(), bdk::Error> {
    let client = Client::new("ssl://electrum.blockstream.info:60002")?;
    let wallet = Wallet::new(
        "wpkh([c258d2e4/84h/1h/0h]tpubDDYkZojQFQjht8Tm4jsS3iuEmKjTiEGjG6KnuFNKKJb5A6ZUCUZKdvLdSDWofKi4ToRCwb9poe1XdqfUnP4jaJjCB2Zwv11ZLgSbnZSNecE/0/*)",
        Some("wpkh([c258d2e4/84h/1h/0h]tpubDDYkZojQFQjht8Tm4jsS3iuEmKjTiEGjG6KnuFNKKJb5A6ZUCUZKdvLdSDWofKi4ToRCwb9poe1XdqfUnP4jaJjCB2Zwv11ZLgSbnZSNecE/1/*)"),
        bdk::bitcoin::Network::Testnet,
        MemoryDatabase::default(),
    )?;
    let blockchain = ElectrumBlockchain::from(client);

    wallet.sync(&blockchain, SyncOptions::default())?;

    let send_to = wallet.get_address(New)?;
    let (mut psbt, details) = {
        let mut builder =  wallet.build_tx();
        builder
            .add_recipient(send_to.script_pubkey(), 50_000)
            .enable_rbf()
            .do_not_spend_change()
            .fee_rate(FeeRate::from_sat_per_vb(5.0));
        builder.finish()?
    };

    let finalized = wallet.sign(&mut psbt, SignOptions::default())?;

    Ok(())
}
```

This creates a wallet instance with the given descriptors, and uses a electrum backend to sync the wallet. It then creates a new transaction and then signs it using the same wallet.

If we were to do this using a harware wallet, how would we do this?

First we create a client instance of the device.

```rust
let devices = HWIClient::enumerate().unwrap();
let client = HWIClient::get_client(
    devices
        .first()
        .expect("No devices found. Either plug in a hardware wallet, or start a simulator."),
    true,
    types::HWIChain::Test,
)
.unwrap();
```

We would then need a descriptor from the device for bdk.

```rust
let descriptors = client.get_descriptors(None).unwrap();
```

We would now need to create a instance of custom signer. A basic version is provided in bdk/wallet/hardwaresigner/signer/HWISigner. The basic signer has no extra features, it just takes a psbt and simply hands it over to the hardware wallet.

```rust
let custom_signer = HWISigner::from_device(devices.first().unwrap(), types::HWIChain::Test).unwrap();
```

We now create a wallet instance using the descriptor from the hardware wallet and add the custom signer.

```rust
let mut wallet = Wallet::new(
    &descriptors.internal[0],
    Some(&descriptors.receive[0]),
    Network::Testnet,
    MemoryDatabase::default(),
)?;
wallet.add_signer(
    KeychainKind::External,
    SignerOrdering(200),
    Arc::new(custom_signer),
);
```

Rest everything remains the same!

```rust
let client = bdk::electrum_client::Client::new("ssl://electrum.blockstream.info:60002")?;
let blockchain = bdk::blockchain::ElectrumBlockchain::from(client);
wallet.sync(&blockchain, bdk::SyncOptions::default())?;

let send_to = wallet.get_address(New)?;
let mut tx_builder = wallet.build_tx();
tx_builder
    .add_recipient(send_to.script_pubkey(), 50_000)
    .enable_rbf();
let (mut psbt, _tx_details) = tx_builder.finish()?;
let finalized = wallet.sign(&mut psbt, SignOptions::default())?;
```
