---
description: Ship your own name service on FLOW - Part 2 - Setting up for Deployment
---

# Name Space - Part 2

<figure><img src=".gitbook/assets/image.png" alt=""><figcaption></figcaption></figure>

Great! You've gone through a LONG tutorial on how to build the Domains contract. You learnt a shit ton of things. Now, it's time to deploy to the Flow testnet, and set up your environment to be able to do that.

Since Flow is relatively earlier stage, compared to more mature ecosystems like Ethereum, the developer tooling is a bit lower-level than Ethereum. It takes a bit of setup to get everything working, however you get used to it very quickly. This is why we have decided to keep the environment setup as a separate level entirely to make sure we can explain each step properly.

### 🌊 flow.json

When you created your app initially in the last level using `flow app create flow-name-service` it set up a project structure for you to work off of.

In the folder `flow-name-service`, the most important file we will be working with is `flow.json`. This is the configuration file for the Flow CLI, and defines the configuration for actions that the Flow CLI can perform for you.

Think of this as roughly equivalent to `hardhat.config.js` on Ethereum.

Open this file up in your code editor, and the default should look something like this:

```json
{
  "emulators": {
    "default": {
      "port": 3569,
      "serviceAccount": "emulator-account"
    }
  },
  "contracts": {},
  "networks": {
    "emulator": "127.0.0.1:3569",
    "mainnet": "access.mainnet.nodes.onflow.org:9000",
    "testnet": "access.devnet.nodes.onflow.org:9000"
  },
  "accounts": {
    "emulator-account": {
      "address": "f8d6e0586b0a20c7",
      "key": "2619878f0e2ff438d17835c2a4561cb87b4d24d72d12ec34569acd0dd4af7c21"
    }
  },
  "deployments": {}
}

```

You see it comes pre-configured with one `account` - the `emulator-account`. This account does not work on the Flow Testnet or Mainnet, and is only to be used with the local Flow Emulator - which is similar to running a local Hardhat node.

In our case, we will not be touching the Emulator and will deploy to the Flow testnet.

### 🧾 Create a Testnet Account

One of the first things we want to do is generate a new account for use on the Flow Testnet. Since the Emulator Account is the same for everyone, we should really not be using it for anything on Testnet or Mainnet, as it is accessible to thousands of people.

Let's walk through the process of creating a new Testnet account, and adding it to the configuration.

In your terminal, `cd` into the `flow-name-service` directory on your computer, and execute the following command:

```sh
flow keys generate
```

