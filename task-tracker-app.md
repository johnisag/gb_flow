---
description: Cadence - Build a task tracker
---

# Task Tracker App

In this level, we will dig a bit deeper into Cadence, and learn about Arrays, Resources, and Account Storage. Resources are probably the most important feature of Cadence, and we will see what unique things they allow, and also how to use resources properly.

We will be creating a contract where every user can manage their own list of tasks they need to do, and can only add/update tasks within their own list - and not in someone else's list.

### 👩‍🔧 Flow Playground

We'll continue to use the [Flow Playground](https://play.onflow.org/) to write this contract while we are still learning Cadence. Open up the Playground, and go to the `0x01` account, and delete all the default generated code - we'll start from scratch this time.

### 📚 Resources

**Resources are a little bit similar to structs, but not exactly.**&#x20;

**Cadence does have support for structs** as well, but resources allow for certain different things.

If we talk about the differences, structs are really just groupings of data together. They can be created, copied, overwritten, etc. Resources have certain limitations (which allow for certain features) compared to that.

The analogy to think about is that **a Resource is like a finite resource** in the real world.&#x20;

* they are always 'owned' by someone,&#x20;
* they cannot be copied,&#x20;
* they cannot be overwritten,&#x20;
* one resource can only be in one 'position' at a time.&#x20;

We will see what this all means as we begin coding as well.

There is also a **loose similarity between Resources in Cadence, and the Ownership model of data in Rust**. If you're familiar with Rust, think of resources as similar to owned pieces of data to help you understand.

### 📗 The TaskList Resource

In the `0x01` tab on the playground, add the following starter code and let's understand what is going on.

```solidity
pub contract TaskTracker {
    pub resource TaskList {
        pub var tasks: [String]

        init() {
            self.tasks = []
        }
    }
}
```

The first thing to notice here is the `pub resource TaskList` declaration. **A resource declaration in Cadence is very similar to how you would define structs as well.**

The resource itself contains `tasks` - an array of strings (p.s. now you also know the syntax for defining arrays).

Lastly, to note, is that resources need their own `init()` function to initialize values of member variables. In this case, `tasks`. We initialize it to an empty array to begin with.

### 🔨 Resource Creation

**Resources need to be created and used very carefully.**&#x20;

**Resources always live or exist in one 'position' only i.e. you cannot create copies of resources.**&#x20;

If you want to move a resource from one 'position' to another, this must be done explicitly, let's see how.

Add a function inside your contract above as follows
