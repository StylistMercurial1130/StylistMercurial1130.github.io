---
title: "LSM Tree based key value store part 1"
tags: Database Algorithms
---
As I was swimming around in the vast cosmos known as YouTube, I came across this absolutely amazing playlist of lectures on database internals (check [this](https://www.youtube.com/watch?v=otE2WvX3XdQ&list=PLSE8ODhjZXjYDBpQnSymaectKjxCy6BYq) out, it is really cool). I came across the section on LSM or **Log Structured Merge Tree**. This is a storage mechanism used in storage engines as part of database management systems, which is an append-only log of PUT or DELETE operations, and it got me wondering: What if I were to create a key-value store that is backed by an LSM-based storage engine? How would that work? How would that perform?

Now, of course, I could answer that question by simply just asking on Reddit or ChatGPT, but where is the fun in that? So no, I shall be embarking on the journey on my own and answer that question by myself. I am going to be deep-diving into this rabbit hole, guns blazing. It's going to be gritty, it's going to be messy, there will be times I might regret it, but it won't be boring. So I'm going to be jumping into this hellhole like I am the [Doom Slayer](https://doom.fandom.com/wiki/Doom_Slayer), and I am going to be making my own barely acceptable and hopefully functioning key-value store which persists to disk using an LSM Tree-based storage engine in Golang.

## What is an LSM Tree?

The LSM Tree can be seen as a storage mechanism with two components: one that resides in memory and the other that resides on disk. Any PUT or DELETE operations are written to memory first. They are written into something called a **memtable**. **Memtables** are in-memory data structures; these could be skiplists, AVL trees, red-black trees, or any other self-balancing data structure. The operations are also logged into something known as a **WAL (Write Ahead Log)**. These write-ahead logs are sequential logs that help ensure durability and maintain write order; they are helpful for recovering data in case of crashes or failures.

Once the memtables have reached a certain capacity, they are flushed to disk. This is where things get more interesting. They are flushed in the form known as **SS Tables (Sorted String Tables)**. The SS Tables are on-disk data structures that store key-value pairs in sorted order, which are stored in immutable files—hence, append-only.

As more and more data is flushed to disk, the SS Tables are stored in levels where each level contains SS Tables that are more compact than the levels above.

![LSM Tree](/assets/images/Untitled%20Diagram.drawio.png){: .align-center}

## The Key-Value Store

The LSM Tree is going to be the storage engine for the key-value store that is going to help in persisting the key-value tuples to disk. The storage engine is going to be simple in its design, i.e., it's going to only expose APIs to put and delete records from the storage engine—no fancy-shmancy stuff, just simple Put(), Delete(), and Get().

The layer above is where the key-value store will sit; this is what is going to be responsible for actually exposing the key-value store facilities to the outside user.

![LSM Tree](/assets/images/key-value-store-architecture.png){: .align-center}

In the next part, we shall be bringing this thing to life. Stay tuned for that, it's going to be exciting!