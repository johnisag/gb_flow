---
description: Ship your own name service on FLOW - Part 3 - The Website
---

# Name Space - dApp

<figure><img src=".gitbook/assets/image (2).png" alt=""><figcaption></figcaption></figure>

We'll build out the full website where users can do the following things:

1. Connect with their Flow Wallet
2. Look at all the registered FNS Domains
3. Purchase a new FNS domain
4. Manage an FNS domain they own
   1. Edit the Bio
   2. Edit the Linked Address
   3. Renew it for longer duration

As always, we will use Next.js to build this out.&#x20;

### 🧹 Pre-Work

Before we set up a new Next.js app, there's one tiny thing we have to do.

You see, GitHub does not push empty folders into a repository. If you have an empty folder inside a Git repo, it will not be pushed to GitHub when you make a commit and push.

When we set up our Flow App using the Flow CLI, it contained an empty folder called `web`. To make sure that GitHub keeps track of the folder, even though it is an empty, the Flow CLI auto-generated a file called `.gitkeep` within the `web` folder. This file tells GitHub to ignore its rules, and let us push an empty folder to GitHub anyway when we want to.

Why am I talking about this? Well, we want to create a Next.js app inside this `web` folder and maintain the project structure the Flow CLI generated for us. Unfortunately, the `create-next-app` tool has problems creating an app in a directory that already has files in it - even if it's literally an empty `.gitkeep`.

So, before we proceed, make sure you delete the `.gitkeep` file in `flow-name-service/web`.

### 👨‍🔬 Setting Up

Open up your terminal and enter the `flow-name-service` directory. Then, run the following command:

```
npx create-next-app@latest ./web
```

This will setup a new Next.js project for you within the `web` folder that the Flow CLI set up for us. We now have a fresh web app ready to go!

### 😎 Git Good

The `create-next-app` tool also initializes a Git repo when it sets up the project. However, since we would like to make our parent directory `flow-name-service` a Git repo, we don't want to keep the `web` folder as a separate Git repo to avoid having one Git repo inside another Git repo (Git submodules).

Run the following command in your terminal

```
cd web
# Linux / macOS
rm -rf .git
# Windows
rmdir /s /q .git
```

### ⛩ File Structure

The `pages` directory within the `frontend` folder is where we will be doing most of our work. Right now, the `pages` directory should look something like this

```sh
pages/
├─ api/
│  ├─ hello.js
├─ _app.js
├─ index.js
```

We won't be doing any backend here, so we can get rid of the `api` folder. So go ahead and delete that.

`index.js` is our homepage, and we will use that to display all the registered FNS domains.

Apart from that, create a new file `purchase.js` under `pages`, which will be the Purchase page.

Then, create a directory called `manage` under `pages`, and within it create two files - `index.js` and `[nameHash].js`.

`manage/index.js` will show all the domains owned by the currently logged in user, and they can click on any of them to go to `manage/[nameHash].js` where we will let them update the Bio, Address, or Renew the Domain.

Now, we will also be creating some React components to increase reusability across pages, so we don't write the same code multiple times.e

Create a directory named `components` under `web`, and we will add some components here as we go.

Also, we will store all the Flow configuration, Transactions, and Scripts that we write within its own folder. Create a directory named `flow` under `web` and we will start adding things there shortly.

Lastly, create a directory named `contexts` under `web` - this is where we will create a React Context (a way to share state variables and other code across pages and components) to store data about our currently logged in user.

By the end, you should have a structure that looks like this:

```
components/contexts/flow/pages/
├─ [manage]/
│  ├─ index.js
│  ├─ [nameHash].js
├─ _app.js
├─ purchase.js
├─ index.js
```

### 💰 Flow Client Library (FCL)

We will use the [Flow Client Library](https://docs.onflow.org/fcl/) for handling wallet connection, running scripts, sending transactions, etc across the entire application.

Run the following command in your terminal to install the dependency required:

```
npm install @onflow/fcl
```

### ⚙️ Configuring the FCL

Create a file named **`config.js`** under **`web/flow`** directory. Here we will specify configuration for the Flow Client Library to use for a few different things.

Add the following code to it:

```javascript
import { config } from "@onflow/fcl";

config({
  // The name of our dApp to show when connecting to a wallet
  "app.detail.title": "Flow Name Service",
  // An image to use as the icon for our dApp when connecting to a wallet
  "app.detail.icon": "https://placekitten.com/g/200/200",
  // RPC URL for the Flow Testnet
  "accessNode.api": "https://rest-testnet.onflow.org",
  // A URL to discover the various wallets compatible with this network
  // FCL automatically adds support for all wallets which support Testnet
  "discovery.wallet": "https://fcl-discovery.onflow.org/testnet/authn",
  // Alias for the Domains Contract
  // UPDATE THIS to be the address of YOUR contract account address
  "0xDomains": "UPDATE_ME",
  // Testnet aliases for NonFungibleToken and FungibleToken contracts
  "0xNonFungibleToken": "0x631e88ae7f1d7c20",
  "0xFungibleToken": "0x9a0766d93b6608b7",
});
```

**MAKE SURE** you update the contract alias for `0xDomains` otherwise your website will not work. We will see how this alias works in a bit.

### &#x20;Account Initialization

If you remember from Part 1, I mentioned that the `createEmptyCollection` global function on our smart contract will be used to initialize user accounts who wish to purchase FNS domains.

We already initialized the smart contract / admin account during the contract initializer. However, all other users who want to buy FNS domains, must first initialize an empty collection in the requisite storage paths in their own accounts.

Therefore, we need two things:

1. A script that can check whether or not a user's account is already initialized
2. A transaction that can initialize their account for them, if necessary

Create a file **`scripts.js`** under **`web/flow`** directory. Add the following code to it:

```javascript
import * as fcl from "@onflow/fcl";

export async function checkIsInitialized(addr) {
  return fcl.query({
    cadence: IS_INITIALIZED,
    // addr: address as string ; t.Address 
    // (encode it as address) (t contains all Flow data types)
    args: (arg, t) => [arg(addr, t.Address)],
  });
}

const IS_INITIALIZED = `
import Domains from 0xDomains
import NonFungibleToken from 0xNonFungibleToken

pub fun main(account: Address): Bool {
    let capability = getAccount(account).getCapability<&Domains.Collection{NonFungibleToken.CollectionPublic, Domains.CollectionPublic}>(Domains.DomainsPublicPath)
    return capability.check()
}
`;

```

There's a few different things to learn from this snippet of code.

Let's first look at the `IS_INITIALIZED` Cadence Script. Notice the imports. Instead of importing from addresses, we are importing contracts from `0xDomains` and `0xNonFungibleToken` respectively. This is done not only for readability purposes, but also so that you can write the scripts once and have them work across multiple networks that your dApp might support - for example Testnet and Mainnet versions.

The aliases for these were defined earlier when we configured the `config.js` file for FCL.

As for the script itself, it just attempts to borrow a public capability from the given account address for `Domains.Collection`. `capability.check()` returns `true` or `false` depending on whether or not that resource exists at the given public path. If it does, that means the user's account has already been initialized. If it does not, then we will ask them to initialize it through a transaction.

Lastly, let's look at the function we are exporting. Two things to notice there:

1. The `fcl.query` syntax
2. The `args: (arg, t) => [arg(addr, t.Address)],` line

FCL offers two main methods of interacting with the blockchain. `fcl.query` and `fcl.mutate`. Since Scripts are akin to `view` functions in Solidity and don't require any gas fees to run, we are essentially just querying the blockchain. So we use `fcl.query` to run Scripts. `fcl.mutate`, as we will see right after this, is used to make transactions to the blockchain that modify the state.

As for the arguments, since our script requires an `account` argument to be passed for it, we need to encode our string version of the address into something the Flow network and Cadence can understand. We do this using the `(arg, t)` helper values given to us.

`arg` is a function that takes a string value representing the argument, in this case the address. `t` is an object that contains all the different data types that Cadence has, so we can tell `arg` how to encode/decode the argument we are giving.

In this case, we are giving the string `addr` and telling FCL to encode it as the type `Address` by using `t.Address`

All of this will become second-nature as we write more and more scripts and transactions for our app.
