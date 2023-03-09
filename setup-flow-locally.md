---
description: Setting up FLOW Developer Environment Locally
---

# Setup Flow Locally

<figure><img src=".gitbook/assets/image (5).png" alt=""><figcaption></figcaption></figure>

### 👝 The Wallet

There are a few different options for wallets to choose from:

1. [Lilico](https://lilico.app/) - A non-custodial browser extension wallet (Only works with Chromium browsers, sorry Firefox)
2. [portto](https://portto.com/) - A custodial iOS, and Android wallet
3. [Finoa](https://www.finoa.io/flow/) - An institutional grade custodial wallet (Requires KYC)
4. [Dapper](https://www.meetdapper.com/) - A custodial web wallet (Requires KYC)

We recommend using either [Lilico](https://lilico.app/) or [portto](https://portto.com/). I am going to be using Lilico as it's easier to just use a browser extension than a mobile wallet during development, in my opinion.

Once you install the extension, it will take you through setting up a new FLOW wallet. Go ahead and do the required steps.

This is currently connected to the FLOW mainnet. What we want to do is enable Developer Mode on this. Go to settings (cog icon, bottom right), click on `Developer Mode`, and turn on the toggle switch.

Then, select Testnet for the network.

<figure><img src=".gitbook/assets/image (1).png" alt=""><figcaption></figcaption></figure>

### 💰 Getting Testnet Tokens

Once your wallet is all set up, we need to get some testnet FLOW tokens.

1. Visit the [Flow Testnet Faucet](https://testnet-faucet.onflow.org/fund-account)
2. Copy your wallet address from Lilico and paste it in the address input box
3. Click on `Fund your account` and wait to receive the Testnet Tokens

### 🖥️ The Flow CLI

The Flow CLI is a command-line interface that provides useful utilities for building Flow applications. Think of it like Hardhat, but for Flow.

Install the Flow CLI on your computer. Installation instructions vary based on operating system, so [follow the installation steps here](https://docs.onflow.org/flow-cli/install/) to get the up-to-date instructions.

We will use the CLI to create dApps moving forward for Flow.
