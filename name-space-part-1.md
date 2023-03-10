---
description: Ship your own name service on FLOW - Part 1 - Smart Contracts
---

# Name Space - Part 1

You've likely heard of Ethereum Name Service (ENS).&#x20;

Maybe you've even heard about Unstoppable Domains.&#x20;

These platforms allow you to buy and sell domain names tied to blockchains where a human readable domain name can represent your crypto address, and a few other things.&#x20;

**These domain names, at least for ENS, are also represented as NFTs so they can be traded on the secondary market.**&#x20;

Some rare domain names, such as three digit ENS names, have occasionally gone for hundreds of thousands of dollars on the secondary market.

In this level, we will be building **OUR OWN** name service on Flow, completely from scratch!

We will use and learn about a bunch of amazing Flow developer tools to make this possible

* **Flow CLI**
* **Cadence** (a LOT of it)
  * DEEEP dive into **Resources**
  * DEEEP dive into **Capabilities**
* **NextJS** (React)
* **Flow Blockchain**

This level will also help you understand deeply the process of deploying contracts to Flow's network, how to write Cadence code, and dig deep into advanced Cadence concepts like resources and account storage.

We will divide this lesson into two parts - Smart Contracts and Frontend.

### 🤩 Final Output

This is what we will be building by the end of this lesson series:

<figure><img src=".gitbook/assets/image (6).png" alt=""><figcaption></figcaption></figure>

### 🔬 Setting Up

Make sure you have installed the [Flow CLI](https://docs.onflow.org/flow-cli/install/) to proceed.&#x20;

**Install Cadence Extension for Visual Studio Code from** [**here**](https://marketplace.visualstudio.com/items?itemName=onflow.cadence)**.**

Run the following command in your terminal

```
flow setup flow-name-service
```

Choose `Template` and then `Basic` when it prompts you.

This will setup a basic Flow project for us where we will write our code, inside the `flow-name-service` directory.

### 💡 Understanding

We will use NFTs to represent every domain name on the name service. For this, we will use Flow's NFT Standard - `NonFungible`. For the most part, it is quite similar to ERC-721 on Ethereum, but of course has some differences specific to Cadence and uses Resources to represent individual NFTs and NFT Collections. The documentation for the NFT Standard can be found [here](https://github.com/onflow/flow-nft).

Users will be able to purchase a new domain name by minting a new NFT, which can then be traded on a marketplace as well. This NFT will contain information that associates the user's crypto address to the domain name as part of its metadata.

### 🪵 Using the Standards

Since we are going to use the `NonFungibleToken` token standard, let's start by creating a Cadence file for the NFT interface (similar to Interfaces in Solidity) which will allow us to implement our contract.

Create a new directory under **`flow-name-service/cadence/contracts`** called **`interfaces`**, and then create a file named **`NonFungibleToken.cdc`** there.

> Cadence code files end with the extension .cdc

Copy over the NFT standard interface into that file from [NonFungibleToken.cdc](https://github.com/onflow/flow-nft/blob/master/contracts/NonFungibleToken.cdc).

This file defines all the functions, resources, events, etc that are required for an NFT contract following the standard on Flow. When we write our contract, we will be implementing this interface.

Additionally, since the NFT domain names are going to be purchased using the Flow Token, we will also need its required standards. Unlike Ethereum, the Flow token itself follows the same `FungibleToken` standard as all other tokens built on Flow. This is again quite similar to `ERC-20` on Ethereum, except, of course, has Cadence and Flow specific things we will understand.

Create a file named **`FungibleToken.cdc`** in

**`flow-name-service/cadence/contracts/interfaces`**,&#x20;

and copy over the Fungible Token standard interface into that file from **** [**FungibleToken.cdc**](https://github.com/onflow/flow-ft/blob/master/contracts/FungibleToken.cdc)****

Lastly, we also need the contract for the Flow Token itself as we will be doing some things specific to that token itself. Again, because the Flow token itself is implemented using the `FungibleToken` standard, it itself is implemented as a smart contract on Flow - this is very unlike ETH on Ethereum which is just built into the network and lacks most ERC-20 features.

**Create** a directory within **`flow-name-service/cadence/contracts`** called **`tokens` ** and create a file named **`FlowToken.cdc`** there with the code copied from [**FlowToken.cdc**](https://github.com/onflow/flow-core-contracts/blob/master/contracts/FlowToken.cdc)****

**IMPORTANT NOTE:** Notice that `FlowToken.cdc` has an import at the top of the file that looks like this:

```
import FungibleToken from 0xFUNGIBLETOKENADDRESS
```

**REPLACE that with the following line:**

```
import FungibleToken from "../interfaces/FungibleToken.cdc"
```

Don't worry about what was going on with the original line there, we will get to it shortly, but for now - we should import the `FungibleToken` standard from our local file where we defined the interface for it.

### 🧱 Code Structure

Let's build a mental model of the code we are going to be writing so the following steps make sense before we proceed. As a developer, it is important to have a somewhat vague idea of how you are going to structure your code so that it makes sense. It also helps you think about the requirements of your project.

At the highest level, there are two major parts of our codebase.&#x20;

* An NFT collection, which we will just call **`Collection`**, which represents the NFT collection for all **FNS domains in our dApp**.&#x20;
* Secondly, **a manager of sort** which will provide us, the Admin, functions to set the price of the domains, enforce minimum rent duration periods, etc. We will call this the **`Registrar`**.

Since Cadence is all based around the concept of Resources, let's think about how this will work. It will also be helpful to read the following section with the code for the **`NonFungibleToken` ** standard open in your code editor, so you can draw some connections between what I'm talking about and what the standard enforces.

### 💎 Everything as a Resource

This part is likely going to be one of the trickiest parts to properly understand in this level. Make sure to spend some time here and have a decent understand before proceeding.

Coming from Solidity, we are used to all Smart Contract related data being stored in the smart contract itself. As we have alluded to multiple times at this point, **data on Flow is usually stored directly with the user itself - in the user's account storage.**&#x20;

Therefore, we have to think about Resources very carefully. Cadence makes it really hard to write bad code, but that also means it has a learning curve when just starting out. I will try my best to explain it the best I can.

There are a few different things that will be represented as Resources in our contract:

* **Registrar**
  * Registrar Public Functions
  * Registrar Private Functions
* **Collection**
  * Collection Public Functions
  * Collection Private Functions
* **NFT**
  * NFT Public Functions
  * NFT Private Functions
* **Global Public Functions**

Let's go with a bottom-up approach.

#### The NFT

At the lowest level, **we have a single NFT that represents a single Domain registered through FNS.** The **`NonFungibleToken` ** standard dictates that **any contract following the standard **_**MUST**_** define a resource named `NFT`** that, at the very least, **has a public variable called `id`** - the Token ID of the NFT.

A resource is nothing more than some piece of data bundled together, except that it 'lives' somewhere.

Keeping that in mind, **think of the NFT as some piece of data** which ****&#x20;

* **defines the domain name**,&#x20;
* the **linked crypto address to the domain,**&#x20;
* **a small bio written by the owner**,&#x20;
* a **registration date**,&#x20;
* an **expiration date**,&#x20;
* etc.

**This piece of data must eventually end up 'living' in the owner's account storage.**&#x20;

* i.e. this data will be stored with the person who owns this NFT.&#x20;

_This further implies that a third-party will need to dig into the owner's account storage to read information about this NFT, and cannot do so directly from the smart contract as the contract itself does not 'own' this NFT._

**Since Flow has the concept of public and private storage** **paths** within an account,&#x20;

* not all data about an NFT must be made public.&#x20;

_For example, a third-party must not have access to a function that would let them change the linked crypto address to the domain name. However, a third party should have access to functions that let them view the currently linked crypto address, and other information that is okay to make public._

You can then roughly divide the `NFT` resource into two categories

* &#x20;a Public and a Private part of it.&#x20;
* Certain functions and variables will be stored in the user's public storage, and others will in the user's private storage thereby making them accessible only by the owner of the NFT.

We will dig deeper into how that is exactly done as we start writing code, but just keep in mind the **NFT resource has a public portion and a private portion**.

#### The Collection

There is a very important terminology difference when we talk about the Collection in terms of Flow vs Ethereum. On Ethereum, an NFT collection typically refers to a smart contract - and all the NFTs minted through that smart contract.&#x20;

**On Flow, however, a collection refers to any group of NFTs minted through a specific smart contract owned by a specific user.** i.e. it does not refer to _all_ NFTs in the smart contract, but rather a subset of them that are owned by a specific user.

As such, for our `Domains` contract, every user who purchases one (or more) FNS domains has their own Collection which can contain one (or more) FNS domains.

Quite similar to the `NFT` resource, the `Collection` resource is also split into Public and Private portions. If we think about it in terms of data being stored in a user's account, the user's account will never store `NFT` data directly. Rather, it will store a `Collection` resource which inside it contains one (or more) `NFT` resources.

As such, to access information about a specific NFT, a third-party must first reference the Public portion of the Collection, load a specific NFT from within that collection, and then use the Public portion of the NFT resource to get its information.

Therefore, the Public portion of the Collection implements functions that let third-party users look at Public portions of the NFTs contained within the collection.

Similarly, the Private portion of the Collection implements functions that let the owner have access to Private portions of the NFTs contained within the collection.

#### The Registrar

Going a similar path as the above two, the Registrar also has a Public portion and a Private portion. **However, the Registar will be a special resource that is only ever stored within the owner's account storage, and never within a third-party's storage**.

<mark style="color:purple;">The Private portion of the Registrar resource gives access to the contract owner to update prices for domains, and other similar things.</mark>

The Public portion exposes functions such as `registerDomain` and `renewDomain` that allow the public to purchase or renew FNS domains.

#### Global Public Functions

These are functions and variables that are defined on the global level for the smart contract, **i.e. are not part of any resource.** These are what will be accessible and called by the public, and expose the core functionalities of the smart contract to the public.

I'm sure all of this seems a little overwhelming to look at right away. Spend some time to try to just build a mental diagram of everything I described above, and black box specific things you don't understand right now as you will understand them as we write code.

### Domains Contract

Create a file within **`flow-name-service/cadence/contracts`** and name it **`Domains.cdc`**. This is where we will write the code for our NFT Collection.





