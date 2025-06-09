---
title: "LSM Tree based key value store part 1"
tags: Database Algorithms
---
As I was swimming around in the vast cosmos known as youtube, I had come across this absolutely amazing playlist of lectures on database internals (check [this](https://www.youtube.com/watch?v=otE2WvX3XdQ&list=PLSE8ODhjZXjYDBpQnSymaectKjxCy6BYq) out, it is really cool), I had come across the section on LSM or **Log Structured Merge Tree**. This a storge mechanism used in storage engines as part of database management systems which is an append only log of PUT or DELETE operations, and it got me wondering. What if I were to create a key value store that is backed by a LSM based storage engine ? how would that work ? how would that perform ?

Now ofcourse I could answer that question by simply just asking that question on reddit or ChatGPT, but wher eis the fun in that ? So no, I shall be embarking on the journey on my own and answer that question by myself. I am going to be deep diving into this rabbit hole guns blazing. It's going to be gritty, it's going to messy, there will be times I might regret, but it won't be boring. So I'm going to be jumping into this hellhole like I am the [doom slayer](https://doom.fandom.com/wiki/Doom_Slayer) and I am going to be making my own barely acceptable and hopefully functioning key value store which persists to disk using a LSM Tree based storage engine in Golang.

## What is a LSM Tree ?

The LSM Tree can be seen as a storage mechanism with 2 components, One that resisdes on the memory and the other that resides on disk. Any PUT or DELETE Operations are written to memory first. They are are written into something called a **memtable**. **Memtables** are in memory disk data structures, these could be skiplists, avl tres, red-black trees or any other self balancing data structure. The Operations are also logged into something known as a **WAL (Write Ahead Logs)**, these write ahead logs are sequential logs that help ensure durability and maintain write order, they are helpful for recovering data in case of crashes or failures. 

Once the memtables have reched a certain capacity, they are flushed to the disk. there is where things get more interesting. They are flushed in the form knwon as **SS Tables (String Sorted Tables)**. The SS Tables are in disk data structure that store store key-value paris in sorted order which are stored in immutable files, hence append only. 

As more and more data is flushed to disk the SS Tables are stored in levels where each level contains ss tables that are compact than the levels above. 

![LSM Tree](/assets/images/Untitled%20Diagram.drawio.png){: .align-center}

## The Key-Value Store

The LSM Tree is going to be the storage engine for the key value store that is going to help in persisting the key-value tuples disk. The storage engine is going to simple in its desing, i.e its going to only expose api to put and delete records from the storage engine, no fancy-shmansy stuff, just simple Put(), Delete() and Get()

The layer above is where the key value store will sit, this is what is going to responsible for actually exposing the key-value store facilities to the outside user. 

![LSM Tree](/assets/images/key-value-store-architecture.png){: .align-center}

In the Next part we shall be bringing this thing to life. Stay tuned for that, its going to exciting!