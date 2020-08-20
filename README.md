🌈 Eth→NEAR Fungible Tokens 🌈
=============================

Send an [ERC20] token over the [Rainbow Bridge], get an [NEP21] token on [NEAR]


Background
==========

You can think of the [Rainbow Bridge] as having three main pieces:

1. Clients. These get raw NEAR data into Ethereum and vice versa. These are light clients that run as smart contracts in each blockchain, with external relays to pipe the data in each direction.
2. Provers. These are smart contracts that allow making assertions about data stored in the clients.
3. Contract pairs. These provide an interface to Dapps that want to send certain kinds of data or calls from one blockchain to the other. They use the provers to ensure each operation is valid.

This repository contains a **contract pair** to allow a specific fungible token to be sent from Ethereum to NEAR.

On the Ethereum side, this is implemented as a `TokenLocker` Solidity contract. This contract given an ERC20 token contract address on initialization. Then Ethereans can send some amount of that ERC20 token to the contract and it will lock them, emitting a `Locked` event upon success.

On the NEAR side, there's a matching `MintableFungibleToken` Rust contract. When NEARkats call its `mint` function, the function calls Provers to verify that the expected `Locked` event was emitted on Ethereum at least 25 blocks ago.

This could, for example, allow you to convert an ERC20 token like DAI into a wrapped DAI token on NEAR, perhaps called nDAI or nearDAI. The contracts also provide a means to send this wrapped ERC20 back to Ethereum, using the provers & client on Ethereum to verify that the wrapped token was correctly burned on the NEAR side before unlocking DAI in the `TokenLocker` contract.

Here's a schematic representation, where "Transfer script" could be understood to be JavaScript in a user's browser or a CLI call:

![TRANSFER SCRIPT calls 'approve' on ERC20 then 'lockToken' on LOCKER. LOCKER calls 'safeTransferFrom' on ERC20 then emits 'Locked' event. TRANSFER SCRIPT then notes the block of the 'Locked' event. TRANSFER SCRIPT then waits for this block to finalize over the bridge and extracts a proof. TRANSFER SCRIPT calls 'mint' on MINTABLE FUNGIBLE TOKEN. MINTABLE FUNGIBLE TOKEN checks that the event was not used before and that it's not too far in the past, then calls 'verify_log_entry' on PROVER. PROVER calls 'verify_trie_proof' on itself and then calls 'block_hash_safe' on ETH ON NEAR CLIENT. ETH ON NEAR CLIENT calls 'on_block_hash' on PROVER which calls 'finish_mint' on MINTABLE FUNGIBLE TOKEN](erc20-to-near.png)

For more detail about Rainbow Bridge, refer to [this introductory blog post](https://near.org/blog/eth-near-rainbow-bridge/)

  [ERC20]: https://eips.ethereum.org/EIPS/eip-20
  [Rainbow Bridge]: https://github.com/near/rainbow-bridge
  [NEP21]: https://github.com/nearprotocol/NEPs/pull/21
  [NEAR]: https://near.org/


How to use this repository
==========================

If you want to send an ERC20 token from Ethereum to NEAR:

1. Clone this repository
2. Compile the contracts (see `build.sh` script in token directories)
3. Deploy the contracts

Note that you don't need to modify the contracts, since all the info they need is passed as initialization parameters.

If you want to create a new **contract pair** for something other than sending ERC20 tokens from Ethereum to NEAR, you can clone this repository and use it as a starting point.


Deploy
======

This requires three steps:

1. Create NEAR account
2. Deploy TokenLocker contract to Ethereum
3. Deploy MintableFungibleToken contract to NEAR


Create NEAR account
-------------------

Every smart contract in NEAR has its [own associated account][NEAR accounts]. Before you deploy your TokenLocker to Ethereum, you'll need to know the NEAR account name where the matching MintableFungibleToken will live.

### Step 1: Install near-cli

[near-cli] is a command line interface (CLI) for interacting with the NEAR blockchain. For best ergonomics you probably want to install it globally:

    npm install --global near-cli

Ensure that it's installed with `near --version`


### Step 2: Create an account for the contract

Each account on NEAR can have at most one contract deployed to it. If you've already created an account such as `your-name.testnet`, you can deploy your contract to `my-wrapped-erc20-token.your-name.testnet`. Assuming you've already created an account on [NEAR Wallet], here's how to create `my-wrapped-erc20-token.your-name.testnet`:

1. Authorize NEAR CLI, following the commands it gives you:

      near login

2. Create a subaccount (replace `YOUR-NAME` below with your actual account name):

      near create-account my-wrapped-erc20-token.YOUR-NAME.testnet --masterAccount YOUR-NAME.testnet

  [NEAR accounts]: https://docs.near.org/docs/concepts/account
  [NEAR Wallet]: https://wallet.testnet.near.org/


Deploy TokenLocker to Ethereum
------------------------------

You'll need three data:

1. The Ethereum contract address of the ERC20 token that you want to send to NEAR
2. The account name you created on NEAR above, such as `my-wrapped-erc20-token.YOUR-NAME.testnet`
3. The Prover contract address that will be used when transferring tokens from NEAR back to Ethereum. The Ropsten contract maintained by NEAR is at 0x???

Once you have these, deploy the contract to NEAR as you normally would, initializing your contract with the data above in the given order.

Save the new contract address for your freshly-deployed TokenLocker contract in an environment variable. You'll use it when deploying your NEAR contract.

On Mac/Linux:

    TLADDR=0xYourContractAddress

On Windows:

    set TLADDR=0xYourContractAddress

Check that you set it correctly on Mac/Linux with:

    echo $TLADDR

on Windows:

    echo %TLADDR%


Deploy MintableFungibleToken to NEAR
------------------------------------

In addition to the TLADDR saved above, you will also need to know the address of the Prover on NEAR used to verify that tokens were locked correctly on Ethereum. The address for the TestNet contract maintained by NEAR is ???

To see all the options you can use when deploying a NEAR contract:

    near deploy --help

On Mac/Linux, final command will look like:

    near deploy --accountId=my-wrapped-erc20-token.your-name.testnet --wasmFile=./mintable-fungible-token/res/mintable_fungible_token.wasm --initFunction=new --initArgs='{ "prover_account": "???", "locker_address": "'$TLADDR'" }'

and on Windows:

    near deploy --accountId=my-wrapped-erc20-token.your-name.testnet --wasmFile=.\mintable-fungible-token\res\mintable_fungible_token.wasm --initFunction=new --initArgs="{ \"prover_account\": \"???\", \"locker_address\": \"%TLADDR%\" }"
