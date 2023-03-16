---
description: Ship your own name service on FLOW - Part 1 - Smart Contracts
---

# Name Space - Contract

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

<figure><img src=".gitbook/assets/image (6) (1).png" alt=""><figcaption></figcaption></figure>

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

```solidity
import NonFungibleToken from "./interfaces/NonFungibleToken.cdc"
import FungibleToken from "./interfaces/FungibleToken.cdc"
import FlowToken from "./tokens/FlowToken.cdc"

// The Domains contract defines the Domains NFT Collection
// to be used by flow-name-service
pub contract Domains: NonFungibleToken {

}
```

In the above code, we just defined our contract. The `: NonFungibleToken` syntax means that the `Domains` contract will be implementing the `NonFungibleToken` standard interface. The rest of our code will be going inside our contract block.

I will try my best to not introduce any variables or functions beforehand that don't make sense until we have gotten to the point we actually have to use them and I have an opportunity to explain why we need them in the first place. Occasionally, this might still happen, but generally we will be jumping back and forth between writing specific functions, introducing new global variables, modifying our functions, etc.

### The DomainInfo Struct

Let's create a simple struct called `DomainInfo` that contains all the data pertaining to a single FNS domain NFT. We can use this to return all data associated with an NFT is someone wants to fetch that data.

Add the following code to your smart contract:

```solidity
import NonFungibleToken from "./interfaces/NonFungibleToken.cdc"
import FungibleToken from "./interfaces/FungibleToken.cdc"
import FlowToken from "./tokens/FlowToken.cdc"

// The Domains contract defines the Domains NFT Collection
// to be used by flow-name-service
pub contract Domains: NonFungibleToken {

    // Struct that represents information about an FNS domain
    pub struct DomainInfo {
        // Public Variables of the Struct
        pub let id: UInt64
        pub let owner: Address
        pub let name: String
        pub let nameHash: String
        pub let expiresAt: UFix64
        pub let address: Address?
        pub let bio: String
        pub let createdAt: UFix64

        // Struct initializer
        init(
            id: UInt64,
            owner: Address,
            name: String,
            nameHash: String,
            expiresAt: UFix64,
            address: Address?,
            bio: String,
            createdAt: UFix64
        ) {
            self.id = id
            self.owner = owner
            self.name = name
            self.nameHash = nameHash
            self.expiresAt = expiresAt
            self.address = address
            self.bio = bio
            self.createdAt = createdAt
        }
    }
}
```

This is a good opportunity to explain some of the variables we will be using, and what they mean, and why they're relevant. Some of them are quite obvious.

* `id`: The Token ID of the FNS Domain NFT
* `owner`: The Address of the NFT owner
* `name`: The domain name (learnweb3.fns)
* `nameHash`: The SHA-256 hash of the domain name. We do this because names themselves can be arbitrarily long, and we would like to work with fixed-length identifiers for each domain name to have predictable computation costs. We will primarily use the `nameHash` and the `id` as unique identifiers for each domain.
* `expiresAt`: The timestamp at which the domain is going to expire. Represented in seconds.
* `address`: The linked address to this domain (need not be same as `owner`). The `?` in the data type `Address?` represents this is an optional field, and may be `nil` while the owner has not linked an address.
* `bio`: A short bio written by the owner for this FNS domain
* `createdAt`: The timestamp at which this domain was registered. Represented in seconds.

### The NFT Resource

**The Resource Interfaces**

Similar to how our contract implements the `NonFungibleToken` standard interface, represented by `pub contract Domains: NonFungibleToken`, resources can also implement various resource interfaces.

We will use this feature to define separate resource interfaces for the Public and Private portions of the `NFT` resource, so we can interact with them easily later on. Additionally, the `NFT` resource must also implement the `NonFungibleToken.INFT` interface, which is a super simple interface that just mandates the existence of a public property called `id` within the resource.

Add the following code for the `DomainPublic` resource interface (Public portion of the NFT) to your code:

```solidity
pub resource interface DomainPublic {
    pub let id: UInt64
    pub let name: String
    pub let nameHash: String
    pub let createdAt: UFix64
    
    pub fun getBio(): String
    pub fun getAddress(): Address?
    pub fun getDomainName(): String
    pub fun getInfo(): DomainInfo
}
```

Note that the interface only defines the variables and functions that must exist, not the values or the code for those variables and functions. That is still the job of the resource which will implement this interface.

The definitions themselves are quite self explanatory, these are all the variables and functions we want to make available to third-parties.

Now for the private part, add the following code for the `DomainPrivate` resource interface (Private portion of the NFT) to your code:

```solidity
pub resource interface DomainPrivate {
    pub fun setBio(bio: String)
    pub fun setAddress(addr: Address)
}
```

**The Resource Itself**

Now for the main part, the actual `NFT` resource itself. This is going to be long, so I am going to break it into chunks.

Start by defining the resource itself, and add the following code:

```solidity
pub resource NFT: DomainPublic, DomainPrivate, NonFungibleToken.INFT {
// Rest of the code under this subheading goes here
}
```

**The Resource Itself**

Now for the main part, the actual `NFT` resource itself. This is going to be long, so I am going to break it into chunks.

Start by defining the resource itself, and add the following code:

```solidity
    pub resource NFT: DomainPublic, DomainPrivate, NonFungibleToken.INFT {
        pub let id: UInt64
        pub let name: String
        pub let nameHash: String
        pub let createdAt: UFix64

        // access(self) implies that only the code within this resource
        // can read/modify this variable directly
        // This is similar to `private` in Solidity
        access(self) var address: Address?
        access(self) var bio: String

        init(id: UInt64, name: String, nameHash: String) {
            self.id = id
            self.name = name
            self.nameHash = nameHash
            self.createdAt = getCurrentBlock().timestamp
            self.address = nil
            self.bio = ""
        }

        pub fun getBio(): String {
            return self.bio
        }

        pub fun getAddress(): Address? {
            return self.address
        }

        pub fun getDomainName(): String {
            return self.name.concat(".fns")
        }

        pub fun setBio(bio: String) {
            // This is like a `require` statement in Solidity
            // A 'pre'-check to running this function
            // If the condition is not valid, it will throw the given error
            pre {
                Domains.isExpired(nameHash: self.nameHash) == false : "Domain is expired"
            }
            self.bio = bio
            emit DomainBioChanged(nameHash: self.nameHash, bio: bio)
        }

        pub fun setAddress(addr: Address) {
            pre {
                Domains.isExpired(nameHash: self.nameHash) == false : "Domain is expired"
            }

            self.address = addr
            emit DomainAddressChanged(nameHash: self.nameHash, address: addr)
        }

        pub fun getInfo(): DomainInfo {
            let owner = Domains.owners[self.nameHash]!

            return DomainInfo(
                id: self.id,
                owner: owner,
                name: self.getDomainName(),
                nameHash: self.nameHash,
                expiresAt: Domains.expirationTimes[self.nameHash]!,
                address: self.address,
                bio: self.bio,
                createdAt: self.createdAt
            )
        }

    }
```

Note how we are using a similar syntax as the contract definition. We are implementing three resource interfaces for our NFT resource, so we need to add all the variables and functions these resource interfaces expect to be present.

Let's first define the variables, and a simple initializer. Add the following code within your `NFT` resource.

```solidity
pub let id: UInt64pub let name: Stringpub let nameHash: Stringpub let createdAt: UFix64
    di
}
```

The initializer for this resource takes a few parameters, and assigns default values for the rest - `createdAt`, `address`, and `bio`. Since the creation of this resource will only happen when a new NFT is registered, we can just assign `createdAt` to the timestamp of the current block on the network.

Now, let's define the functions that the resource interfaces expect. We need to implement all the functions expected by all the three interfaces we are implementing.

Let's do the simple ones first:

```solidity
pub fun getBio(): String {    return self.bio}
pub fun getAddress(): Address? {    return self.address}
pub fun getDomainName(): String {    return self.name.concat(".fns")}
```

So far so good, pretty basic. But here we have hit two roadblocks.

1. For the `getInfo()` function that will return a `DomainInfo` struct, we don't currently have a way to fetch the `owner` of a given NFT. We also do not have a way to get the `expiresAt` property.
2. For the `setBio()` and `setAddress()` function, we must ensure that the domain has not crossed its expiry date, in which case the owner should not be allowed to modify anything.

You must be thinking why didn't we just add those two properties directly into the `NFT` resource. Well, a couple of reasons.

1. We want to have global track of the owners of all domains, and the expiry dates of all domains, in existence, and tying them to the NFT resource means we will have to fetch it from the account storage of potentially a lot of different accounts.
2. We want to have the ability to modify the expiration date of a domain when it is renewed, for example, and it is not wise to put that function in the Public portion of the NFT, and if we put it in the Private portion we lose access to that function as that's only accessible to the NFT owner.

***

**So, let's zoom out a bit,** and step outside the NFT resource for a second. Go back to the top-level of your code, and add a couple of global variables.

```
pub let owners: {String: Address}pub let expirationTimes: {String: UFix64}
```

We will use these dictionaries (mappings) to store information about all domain owners and the expiry times. The key (String) will be the domain's `nameHash`, and the values will represent the owner address and expiry time respectively.

Also, let's add a couple of events to the contract we will emit as certain things happen.

```
pub event DomainBioChanged(nameHash: String, bio: String)pub event DomainAddressChanged(nameHash: String, address: Address)
```

While we are here, let's also add a few helper functions to our smart contract (again at the global level).

Note that the last two functions had the access control modifier of `access(account)`. This is different from `access(self)` we saw earlier.

`access(self)` allows the _code_ within the smart contract to access that function/variable - whereas `access(account)` allows the _account_ to access the function/variable - which includes the code itself as well.

On Flow, smart contracts are deployed to regular accounts on the network. So you can have an account you do your everyday things on, which also runs a smart contract. Therefore, `access(account)` here means you, the owner, have access to this function even if calling the smart contract externally. It also means that other smart contracts you have deployed to the same account can also access this function.

***

Now, let's head back to our NFT resource, and fill in the missing pieces. Add the following functions **inside** the `NFT` resource.

```solidity
        pub fun setBio(bio: String) {
            // This is like a `require` statement in Solidity
            // A 'pre'-check to running this function
            // If the condition is not valid, it will throw the given error
            pre {
                Domains.isExpired(nameHash: self.nameHash) == false : "Domain is expired"
            }
            self.bio = bio
            emit DomainBioChanged(nameHash: self.nameHash, bio: bio)
        }

        pub fun setAddress(addr: Address) {
            pre {
                Domains.isExpired(nameHash: self.nameHash) == false : "Domain is expired"
            }

            self.address = addr
            emit DomainAddressChanged(nameHash: self.nameHash, address: addr)
        }

        pub fun getInfo(): DomainInfo {
            let owner = Domains.owners[self.nameHash]!

            return DomainInfo(
                id: self.id,
                owner: owner,
                name: self.getDomainName(),
                nameHash: self.nameHash,
                expiresAt: Domains.expirationTimes[self.nameHash]!,
                address: self.address,
                bio: self.bio,
                createdAt: self.createdAt
            )
        }
```

This code should be mostly self-explanatory, so we will move on. Having added these functions, we have finished the implementation of the `NFT` resource.

By this point, your NFT resource should overall look something like this:

```solidity
    pub resource NFT: DomainPublic, DomainPrivate, NonFungibleToken.INFT {
        pub let id: UInt64
        pub let name: String
        pub let nameHash: String
        pub let createdAt: UFix64

        // access(self) implies that only the code within this resource
        // can read/modify this variable directly
        // This is similar to `private` in Solidity
        access(self) var address: Address?
        access(self) var bio: String

        init(id: UInt64, name: String, nameHash: String) {
            self.id = id
            self.name = name
            self.nameHash = nameHash
            self.createdAt = getCurrentBlock().timestamp
            self.address = nil
            self.bio = ""
        }

        pub fun getBio(): String {
            return self.bio
        }

        pub fun getAddress(): Address? {
            return self.address
        }

        pub fun getDomainName(): String {
            return self.name.concat(".fns")
        }

        pub fun setBio(bio: String) {
            // This is like a `require` statement in Solidity
            // A 'pre'-check to running this function
            // If the condition is not valid, it will throw the given error
            pre {
                Domains.isExpired(nameHash: self.nameHash) == false : "Domain is expired"
            }
            self.bio = bio
            emit DomainBioChanged(nameHash: self.nameHash, bio: bio)
        }

        pub fun setAddress(addr: Address) {
            pre {
                Domains.isExpired(nameHash: self.nameHash) == false : "Domain is expired"
            }

            self.address = addr
            emit DomainAddressChanged(nameHash: self.nameHash, address: addr)
        }

        pub fun getInfo(): DomainInfo {
            let owner = Domains.owners[self.nameHash]!

            return DomainInfo(
                id: self.id,
                owner: owner,
                name: self.getDomainName(),
                nameHash: self.nameHash,
                expiresAt: Domains.expirationTimes[self.nameHash]!,
                address: self.address,
                bio: self.bio,
                createdAt: self.createdAt
            )
        }

    }
```

#### The Collection Resource

1/3 things done! Time for the `Collection` resource now. Similar to earlier, we will define two interfaces first - `CollectionPublic` and `CollectionPrivate`. Since we are also following the `NonFungibleToken` standard, the `Collection` resource must also implement a few interfaces present within the `NonFungibleToken` standard as we will see.

**The Resource Interfaces**

Make sure you are no longer writing code inside the `NFT` resource, and have stepped out from it.

Define the following `CollectionPublic` resource interface:

This interface only defines one function, `borrowDomain`. Remember the concept of `borrow`ing things from an account storage from the TaskTracker level? This function will use that knowledge.

We will enable third-parties to inspect someone's NFT Collection, and borrow a reference to a given NFT within the Collection. However, we will only let them borrow a reference to the `DomainPublic` interface so they don't have access to functions present within `DomainPrivate`.

Now, define the following `CollectionPrivate` resource interface:

Let's first look at `borrowDomainPrivate`. This is pretty similar to `borrowDomain` as in the public collection, except it returns a reference to the _full_ NFT i.e. both its public and private parts. This is expected to be used by the owner of the NFT to borrow a reference to the NFT's private functions, to update the `bio` and `address` attached to the NFT.

The more interesting one here is `mintDomain`. It's an `access(account)` function. Which means, even though it's part of `CollectionPrivate`, it can only be used by the account of the smart contract, which essentially makes it an admin/owner function.

`mintDomain` will be used by the `Registrar` resource we have yet to define, and is used to mint a new NFT domain. It then transfers the domain into the `receiver` which is passed as an argument.

**The Resource Itself**

Now let's start working on writing code for the actual `Collection` resource. In total, we will be implementing five resource interfaces. Two of which we defined, and three that come from the `NonFungibleToken` standard. You can open the code for the `NonFungibleToken` interface to look at what each of those interfaces require.

We'll need to add a few events that are required by the `NonFungibleToken` standard before we proceed. Add the following events to your smart contract:

Create the resource block as follows:

The three interfaces coming from `NonFungibleToken` - `Provider`, `Receiver`, and `CollectionPublic` define functions like `deposit`, `withdraw`, `borrowNFT`, and `getIDs`. I will explain what each of them do as we write them.

Let's create a public variable to store references to the `NFT` resources that belong to this Collection, and create a simple initializer. Add the following code within the `Collection` resource block:

Now, let's create the functions required by each interface. Starting with `NonFungibleToken.Provider`

This function first tries to move the `NFT` resource (the `domain`) out of the dictionary. If it fails to remove it, i.e. the given ID was not found, it panics and throws an error. If it does find it, it emits a withdraw event, and returns the resource to the caller. The caller can then use this resource and save it within their account storage, or further send it to someone else, as part of their transaction.

You have to think of this as a user interacting with their personal NFT collection stored in their account storage. They get access to their collection, call the `withdraw` function on it, get access to a specific `NFT` resource, and then do whatever they want with it.

***

Now, time for the `deposit` function required by `NonFungibleToken.Receiver`

The mental model for this function is that a third-party is calling the `deposit` function on a user's public NFT collection, thereby 'depositing' an NFT in that user's collection. To do so, they provide a `NonFungibleToken.NFT` resource, which in our case would be the `Domains.NFT` resource. They can do this by, for example, first calling `withdraw` on their own collection to get access to the NFT resource.

We typecast the provided resource as a `Domains.NFT` resource to get access to its `id` and `nameHash` properties. Then we check it hasn't expired yet, because if it has, we shouldn't let it be transferred. We then update the owner of the NFT.

The interesting part is this line:

```
let oldToken <- self.ownedNFTs[id] <- domain
```

Can you guess what's happening here?

We're simultaneously performing two operations here.

First, we move the existing NFT resource out of our dictionary. In practice, this is likely to just be `nil` as our code will not allow for two NFT's with the same Token ID, but Cadence does not know that. Cadence thinks that an `NFT` resource must exist in the dictionary, so it wants the resource to be properly moved elsewhere or destroyed.

So, we first move the existing resource (likely `nil`) out of the dictionary, and simultaneously move the new resource `domain` into the dictionary at that `id`. Then, after emitting an event, we `destroy` the old resource, so Cadence does not complain about a resource that no longer lives anywhere.

***

Amazing, now let's do the two functions required by `NonFungibleToken.CollectionPublic`. These are both quite simple.

`getIDs` here is pretty straightforward, just returns all the Token IDs present in `ownedNFTs`.

`borrowNFT` is a generic borrow function required by the NFT standard. It basically returns a reference to the NFT as represented in the `NFT` standard. Don't worry, the NFT standard does not expose any private functions that we defined, it's quite basic, but it's a required function needed for things such as NFT Marketplaces which implement one smart contract to work with all NFT contracts, and they use this function to arbitrarily fetch an NFT for any collection without having to conform to their specific resource types.

***

Now, for the good shit, time to implement the two interfaces we defined. Let's start off with `Domains.CollectionPublic`.

This is again basically the same thing as `borrowNFT`, except it returns it typecasted to `DomainPublic` which exposes public functions we have defined on our Domain NFT. `borrowNFT` does not do this, as it only exposes the functions defined by the `NonFungibleToken.NFT` resource interface, which has no idea about our custom functions.

Note: We added a `pre` check here to make sure the Domain NFT exists, but did not do this in `borrowNFT`. This is because if you look at the NFT standard contract, you will see that the `pre` check is already implemented there for us, whereas here we have to do it ourselves.

***

Finally, time for `Domains.CollectionPrivate`

We will need to define an event and a couple more global variables before we do this. Zoom out a bit, and go to the top of your smart contract. Add the following two variables there, and an event:

Also, define a couple helper functions on the global level as well:

These are both very similar to what we had done before.

The `borrowDomainPrivate` is pretty similar to what we did earlier. It's the same as `borrowDomain`, except that instead of returning just the public portion of the resource, it returns the full resource. Therefore, this function itself is part of the private portion of `Collection`. This will be used by the Collection owner to get full access to an NFT they own, so they can update the `bio` or `address` etc.

The interesting part is `mintDomain`. Hoo, boy! There's a lot to uncover in this function!

The code structure is not too bad up until the last line. We first check if the domain is available. If so, we create a new `Domains.NFT` resource. We set the owner, expiration time, and the nameHash -> ID mapping for it. We increment the totalSupply by 1, and emit an event.

The last line is where it gets interesting, and before I explain that to you, here's a quick lesson on what Capabilities are in Cadence.

***

**Capabilities - Short Lesson**

We have learned that you can split a resource into Public and Privately exposed parts so that third-parties can only access the public parts of a resource you own. We did not actually cover how that process happens. Here's where Capabilities come in.

Users will often want to make it so that specific other users or even anyone else can access certain fields and functions of a stored resource. This can be done by creating a capability.

A capability basically gives some third-party the capability (permission) to access certain parts of a resource you own, without actually transferring the resource to them.

We will use this a lot as we build our frontend, to gain access to the public portions of the `Collection` and `NFT` resources from users so we can load and display public information about those resources on our website. The syntax for how to do this will be covered later.

🤔 What are capabilities used for?

To give a third-party the capability to make a transaction on your behalfTo give a third-party access to certain parts of a resource without moving the resource

🤔 Why should you use capabilities instead of just transferring a resource?

Because it is a cleaner and more specific approachBecause there is no guarauntee you will get your resource back if you transfer itBecause capabilities cost less gas

***

**Back on track**

Now, going back to the `mintDomain` function. Let's see what this last line is doing.

```
receiver.borrow()!.deposit(token: <- domain)
```

Note the type of the `receiver` in the function arguments. It has the type `Capability<&{NonFungibleToken.Receiver}`

This type specifies that it is a capability which conforms to the resource interface `NonFungibleToken.Receiver`. If you scroll above, you will notice that `NonFungibleToken.Receiver` is the resource interface that defines the `deposit` function on the `Collection`.

I mentioned earlier that the `deposit` function is used to deposit an NFT resource in a user's collection.

So basically, the user is giving us access to the public portion of their `Collection`, thereby giving us permission to call the `deposit` function on their collection, thereby letting us transfer the newly minted NFT into their Collection. Makes sense?

The `receiver.borrow()` part is what uses the Capability and 'borrows' the resource using the capability. Remember in the TaskTracker dApp we also used `borrow` to gain access to the `TaskList` resource to then add/remove items from it? Similarly, the `borrow` function here gives us access to the `NonFungibleToken.Receiver` interface of the `Collection` resource - which is basically just access to the `deposit` function.

Then, we call the `deposit` function and move our newly created `domain` resource into the user's collection.

***

There is one last thing we need to do for the `Collection` resource. That is, we need to specify a destructor.

A destructor is basically the opposite of a constructor (or in Cadence's case, an initalizer). Similar to how a constructor is run once when the object is created, a destructor is run once when the object is destroyed.

This is necessary for the `Collection` resource because within it it contains other resources. Specifically, the `ownedNFTs` dictionary contains NFT resources. Since Flow and Cadence will not allow resources to end up 'homeless' - they must either be moved somewhere else, or destroyed, in case their parent resource gets destroyed.

We did not have to do this for the `NFT` resource because the `NFT` resource contains no other resources stored within it, so a destructor is not necessary.

Add the following code to the `Collection` resource, and then we are done with this:

```
destroy() {    destroy self.ownedNFTs}
```

This will destroy all the NFT resources contained within the collection as well, if the `Collection` is destroyed by someone - perhaps the collection owner.

🤔 Why do you never need a destructor for a contract?

Because contracts can handle destruction of contained resources automaticallyBecause contracts cannot be destroyed

🤔 When do you need a destructor in a resource?

All the timeIt is optional and up to the developerWhen the resource has references to other resources within it

***

Anyway, we are now DONE with the `Collection` resource. That's 2/3 done!!!

Again, coming from Solidity, I understand some of this may seem overwhelming. You're learning a LOT of new things in one go. Please ask questions on the Discord whenever you need help. There is no such thing as a bad or a stupid question, so don't be afraid! We're all just avatars in the metaverse living inside a simulation anyway.

At this point, your overall `Collection` resource should look something like this:

```solidity
    pub resource Collection: CollectionPublic, CollectionPrivate, NonFungibleToken.Provider, NonFungibleToken.Receiver, NonFungibleToken.CollectionPublic {
        // Dictionary (mapping) of Token ID -> NFT Resource 
        pub var ownedNFTs: @{UInt64: NonFungibleToken.NFT}

        init() {
            // Initialize as an empty resource
            self.ownedNFTs <- {}
        }

        // NonFungibleToken.Provider
        pub fun withdraw(withdrawID: UInt64): @NonFungibleToken.NFT {
            let domain <- self.ownedNFTs.remove(key: withdrawID)
                ?? panic("NFT not found in collection")
            emit Withdraw(id: domain.id, from: self.owner?.address)
            return <- domain
        }

        // NonFungibleToken.Receiver
        pub fun deposit(token: @NonFungibleToken.NFT) {
            // Typecast the generic NFT resource as a Domains.NFT resource
            let domain <- token as! @Domains.NFT
            let id = domain.id
            let nameHash = domain.nameHash
            
            if Domains.isExpired(nameHash: nameHash) {
                panic("Domain is expired")
            }
            
            Domains.updateOwner(nameHash: nameHash, address: self.owner?.address)
            
            let oldToken <- self.ownedNFTs[id] <- domain
            emit Deposit(id: id, to: self.owner?.address)
            
            destroy oldToken
        }

        // NonFungibleToken.CollectionPublic
        pub fun getIDs(): [UInt64] {
            // Ah, the joy of being able to return keys which are set
            // in a mapping. Solidity made me forget this was doable.
            return self.ownedNFTs.keys
        }

        pub fun borrowNFT(id: UInt64): &NonFungibleToken.NFT {
            return (&self.ownedNFTs[id] as &NonFungibleToken.NFT?)!
        }

        // Domains.CollectionPublic
        pub fun borrowDomain(id: UInt64): &{Domains.DomainPublic} {
            pre {
                self.ownedNFTs[id] != nil : "Domain does not exist"
            }
            
            let token = (&self.ownedNFTs[id] as auth &NonFungibleToken.NFT?)!
            return token as! &Domains.NFT
        }

        // Domains.CollectionPrivate
        access(account) fun mintDomain(name: String, nameHash: String, expiresAt: UFix64, receiver: Capability<&{NonFungibleToken.Receiver}>) {
            pre {
                Domains.isAvailable(nameHash: nameHash) : "Domain not available"
            }

            let domain <- create Domains.NFT(
                id: Domains.totalSupply,
                name: name,
                nameHash: nameHash
            )

            Domains.updateOwner(nameHash: nameHash, address: receiver.address)
            Domains.updateExpirationTime(nameHash: nameHash, expTime: expiresAt)
            
            // We haven't defined the next two lines yet
            Domains.updateNameHashToID(nameHash: nameHash, id: domain.id)
            Domains.totalSupply = Domains.totalSupply + 1

            emit DomainMinted(id: domain.id, name: name, nameHash: nameHash, expiresAt: expiresAt, receiver: receiver.address)

            receiver.borrow()!.deposit(token: <- domain)
        }

        pub fun borrowDomainPrivate(id: UInt64): &Domains.NFT {
            pre {
                self.ownedNFTs[id] != nil: "Domain does not exist"
            }
            let ref = (&self.ownedNFTs[id] as auth &NonFungibleToken.NFT?)!
            return ref as! &Domains.NFT
        }

        destroy() {
            destroy self.ownedNFTs
        }
    }
```

### The Registrar Resource

Time for the final challenge. This is what will complete our smart contract, pretty much.

Before we proceed with defining the resource, let's add a few more variables and events to our code that we will need.

Make sure you are no longer writing code inside the `Collection` resource, it's time to zoom out again.

Add the following variables and events to your contract:

#### The Resource Interfaces

As usual, we will split the Registrar into Public and Private portions. Let's define the two resource interfaces, starting with `RegistrarPublic`

A new variable we have defined here is `prices`. `prices` will be a mapping which defines the rental cost of domain names based on their length. A 1 letter domain name, for example, should cost more than a 30 letter domain name. The private portion of `Registrar` will allow the admin to set various prices depending on domain length.

`renewDomain` and `registerDomain` are self-explanatory functions. `getPrices` just returns the `prices` dictionary. `getVaultBalance` will be used to return the balance of the `Registrar` i.e. how many Flow tokens it has that it earned by selling domain names. More on this shortly.

Now, let's define `RegistrarPrivate`.

`setPrices` is again fairly straightforward. Admin functions to update the prices. We will explain `updateRentVault` in more detail shortly, but note it's argument `vault: @FungibleToken.Vault` - it's our FIRST use of the `FungibleToken` standard we imported at the beginning of this lesson! `withdrawVault` does what it sounds like, moves the Flow tokens earned by the `Registrar` into a different account.

Let's break for a quick lesson on `FungibleToken` Vaults.

***

**FungibleToken.Vault**

In Ethereum, we are used to ERC-20 token balances being stored as part of the global account state on the blockchain. Any contract or account can receive ERC-20 tokens, and then spend them or move them as they want. A `Vault` in the `FungibleToken` standard behaves somewhat similarly, but with resources in mind.

Similar to how the `NonFungibleToken` standard defines `Collection` - where each user has a `Collection` of their own stored in their storage, and that collection contains the NFTs they own from that smart contract, the `FungibleToken` has `Vault`s.

A `Vault` is nothing more than a collection of tokens. You could define a `Vault` to be empty, to have some portion of the total tokens you own of that type, or to have all the tokens you own of that type. It is completely up to the user.

In context of the Registrar, we will create an empty `Vault` for the `FlowToken` - which is the token we will accept for payment in exchange for these domains. As users buy new domains, or renew existing ones, they will be making deposits to this `Vault`.

We can then withdraw the `Vault` using the `withdrawVault` function, and move tokens out of the `Registrar`'s vault, into a separate Vault, perhaps on a different address.

We can also update the `Vault` being used, by having it refer directly to a `Vault` created on a different address perhaps, just by providing a Capability for `FungibleToken.Receiver` - similar to how we asked the user for a Capability for `NonFungibleToken.Receiver` to mint a domain to their account. The `FungibleToken.Receiver` capability allows us to deposit tokens to someone's Vault. Therefore, instead of the `Registrar` using the empty vault that will be initially set up, we can have it accept payment directly to a `Vault` on a different address.

🤔 What is a FungibleToken.Vault resource?

A resource that stores all your tokensA resource that can store a certain amount of token for a given token (FUSDUSDCFlowetc)A resource that lets people view your token balance

***

**The Resource Itself**

Let's build out the `Registrar` resource, and we will almost be done!

Start by defining the resource block as always:

Let's define the variables this resource will contain, and the initializer.

The comments in the code above should reasonably explain what is happening. As for the initializer, we are passing a reference to a `FungibleToken.Vault`, and a Capability for `Domains.Collection`. These are set to the resource, as well as the other values for which we have chosen reasonable defaults.

Now, time to implement the requisite functions. Let's start off with `RegistrarPublic` functions. These are the main meat of the code, so I will go function-by-function.

First, `renewDomain`

The comments in the code should sufficiently describe what is going on. While this is a relatively longer function, it's mostly quite simple for what it is doing.

Let's move on to `registerDomain`.

Again, the comments in the code should sufficiently describe what is going on. However, if you were to have any doubts about any part of `Registrar`, the above two functions are likely to be it. Just message on Discord and we'd be happy to help.

Before moving on, let's implement the `Domains.getDomainNameHash` function that we are using above in `registerDomain` but did not define until now.

**Zoom out** a bit, and add the following helper function to your smart contract **outside** the `Registrar` resource on the global level.

Now, back to the `Registrar` resource.

Let's move on to the other simpler functions, I'll do them all in one go as they're much simpler than `renewDomain` or `registerDomain`

Great! Now, finally, since the `Registrar` resource contains other resources within it - specifically the `FungibleToken.Vault` - we have to create a destructor to destroy the vault if the Registrar is destroyed.

```
destroy() {    destroy self.rentVault}
```

#### Global Functions

We are done with ALL resources!!!!

Now it's just time to add a few global functions that are publicly accessible for anyone to use, create an initializer for the smart contract overall, etc.

Here's a list of things left to do:

1. Create a public function called `createEmptyCollection` which initializes an empty `Domains.Collection` into someone's account storage. This is required by the `NonFungibleToken` standard the contract is implementing.
2. Create a public helper function `getRentCost` that our frontend can later use to predict cost of a given domain name for a given duration
3. Create an initializer for the overall smart contract, which defines values for all our global variables, initializes the Collection for the admin account, and initializes the Registrar for the admin account
4. Create a public helper function `getVaultBalance` so we can see the Flow tokens balance of the Registrar
5. Create public functions for `registerDomain` and `renewDomain`, which call into the admin account owned `Registrar`

**createEmptyCollection**

Since the `Domains.Collection` resource will be used both by the admin account and all of our users, we need to have a public function they can use to create an empty resource (one that starts off with no `ownedNFTs`) in their account storage.

Define a global public function for this as follows:

Super simple, it just creates a new resource and returns it to the caller. They can then store it in their account storage, or do whatever else they might want to do.

**getRentCost**

Add the following public helper function so users can predict the cost of the domain they want to purchase so they know how much funds to send along with the `registerDomain` transaction.

🤔 A third-party user can call:

pub functions in a contract as well as public portions of resources stored in the contract accountOnly the pub functions in a contractOnly the public portion of resources stored in a contract account

**Initializer**

This is where a lot of the magic happens, so understand this carefully. Add the following variable declarations and one event to the global smart contract, first of all, which we will set in the Initializer and will be very helpful moving forward.

```
// Storage, Public, and Private paths for Domains.Collection resourcepub let DomainsStoragePath: StoragePathpub let DomainsPrivatePath: PrivatePathpub let DomainsPublicPath: PublicPath
// Storage, Public, and Private paths for Domains.Registrar resourcepub let RegistrarStoragePath: StoragePathpub let RegistrarPrivatePath: PrivatePathpub let RegistrarPublicPath: PublicPath
// Event that implies a contract has been initialized// Required by the NonFungibleToken standardpub event ContractInitialized()
```

Now, add the following Initializer, and make sure to read the comments in the code to understand what is going on.

**getVaultBalance**

I intentionally added code for this function _after_ we had gone through the Initializer. Now you know where the `Registrar` resource is stored, and how we plan on accessing it. Through the Public and Private capabilities we linked to the admin account's storage paths.

This function will need to borrow those capabilities in order to call the `getVaultBalance` function inside the Registrar resource from the admin account.

Add the following helper function to your smart contract:

Notice how we first get a capability from `self.account` for the `RegistrarPublic` resource interface. We then borrow the resource using that capability, and call the public `getVaultBalance` function we gain access to from the public portion of the resource.

**registerDomain**

This will work pretty much similarly to `getVaultBalance`, where we get access to the `RegistrarPublic` capability, and then just call `registerDomain` on the public capability.

Since these capabilities are public, we could have chosen not to define these functions in our contract at all, and let the users borrow those capabilities and call those functions directly as part of their transaction - but this is just to make the user-sided work (website-side work, really) a little easier for us.

**renewDomain**

Very similar to the above two:

### 🎁 Wrapping Up

And now, FINALLY, with everything in place, we are DONE writing our smart contract.

This level was **HUGE!** I am sure you learned A LOT, and got a LOT more experience with Cadence and Flow's concepts through this level.

I have said this multiple times throughout the level, but I will say it again. Learning a new language, especially one with so many new things (resources, capabilities) and building the mental models around it can take time, and it's okay. I don't expect you to become an expert in Cadence just by doing this one level, but hopefully at least 50% of the things I said made sense to you. The other 50%, **ask me doubts on Discord**, go through the level again, read the code you have written, and it will eventually come to you.

Overall, just for reference, here is what your final smart contract code should look like:







Thank you for sticking with me throughout this long tutorial, have an amazing day, and I'll see you in the next one!

Cheers 🥂



