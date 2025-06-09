---
title: "LSM Tree based key value store part 2"
tags: Database Algorithms
---
We have the plan ready, we know what we have to do, so let's get into the meat of things. To continue where we left off in [LSM Tree Based KeyValue Store 1](./2025-06-09-LSM-Tree-Key-Value-Store-1.md), we had established what the key-value store is going to look like. So now let's put that into implementation. We are going to develop this from the bottom up, so let's start with the storage engine.

## The Storage Engine

As we have seen, the LSM tree storage mechanism consists of 2 components: the in-memory one (Memtables) and the disk. So let's look at the memtable.

## Memtable

The memtable consists of a self-balancing data structure that is flushed to disk in the form of an SS Table upon reaching a certain capacity. Our self-balancing data structure is going to be an **AVL tree**. Why AVL trees? They're easy to implement and have an average time complexity of insertion, deletion, and searching of log(n). Sounds like a solid explanation to me, if you were to ask why.

### AVL Trees

An **AVL tree** is a self-balancing **BST (Binary Search Tree)** where the difference between the height of the left and the right subtree cannot be more than 1.

The balance factor (which is the difference between the height of the left and right subtree) is what helps determine the balancing procedure. The difference should be in the range of -1 to 1, i.e., should be either -1, 0, or 1.

Our node structure is going to be defined something like

``` go
type node [V any] struct {
    key int
    value V
    height int
    leftNode *node[V]
    rightNode *node[V]
}
```

and our root node is encapsulated in

``` go
type AvlTree[V any] struct {
    rootNode *node[V]
    height   int
}
```

At the moment, we are going to treat the key as an int for simplifying the implementation, but in the next part, we are going to change this up to handle the keys in a more generic fashion.

#### AVL Tree Insertion

Whenever a node is inserted into the tree, there are 2 scenarios that could occur.

1. The tree is balanced upon inserting, we don't have to balance, yay!
2. The tree is unbalanced. Oh, no.

In case we hit the "Oh, no" scenario, we don't worry at all. There is a way to get through this tough time.

The first "Oh, no" case. Let's say we insert the elements 1, 2, 3 in the following orders:

1. {3, 2, 1}
2. {3, 1, 2}

![left rotation scenario](/assets/images/rotation_senarion_one.png){: .align-center}

The balance factor at node 3 in (1) and (2) is going to be 3, which is greater than 1, so here we have what I like to call a **left bias** scenario.

In the case of a left bias scenario, we need to apply a few rotations at the child level and grandchild level. The first rotation we are going to see is the **left rotation**.

Let's look at (2) over here. To balance the tree, we need to apply a **left rotation** with 1 as the pivot. This is required so that we can apply a **right rotation** at 3 to balance the tree. This won't make sense as of now, but hang on, it will all come together quite nicely.

The **left rotation** looks something like this.

``` go
func leftRotation[V any](current *node[V]) *node[V] {
    n := current.rightNode
    current.rightNode = n.leftNode
    n.leftNode = current

    current.height = 1 + max(current.leftNode.getHeight(), current.rightNode.getHeight())
    n.height = 1 + max(n.leftNode.getHeight(), n.rightNode.getHeight())

    return n
}
```

1. Take the right side of the subtree and move that one level up.
2. Take the parent to the right side and move that down one level to the left.

It's almost like you are attaching a rope to the root node and pulling it down to the left side. The tree is going to look something like this.

![left rotation result](/assets/images/left_rotation_result.png){: .align-center}

Look at that, we have gotten (1) from (2). Now here we apply the right rotation at 3.

The **right rotation** looks something like this

``` go
func rightRotation[V any](current *node[V]) *node[V] {
    n := current.leftNode
    current.leftNode = n.rightNode
    n.rightNode = current

    current.height = 1 + max(current.leftNode.getHeight(), current.rightNode.getHeight())
    n.height = 1 + max(n.leftNode.getHeight(), n.rightNode.getHeight())

    return n
}
```
1. Take the left side of the subtree and move that one level up.
2. Take the parent to the right side and move that down one level to the right.

![right rotation result](/assets/images/right_rotation_result.png){: .align-center}

And here we go, tree balanced, danger averted.

Now let's take the scenario in which the balance factor is negative, or what I like to call **right bias**.

Let's say we insert the elements 10, 20, and 30 in the order {10, 30, 20}. We would have a tree like this:

![RL scenario](/assets/images/RL_Scenario.png){: .align-center}

Now we apply first a **right rotation** at 30 and then a **left rotation** at 10.

![RL Result](/assets/images/RL_Rotation_Result.png){: .align-center}

Overall, the rotation should go something like this

``` go
current.height = 1 + max(current.leftNode.getHeight(), current.rightNode.getHeight())
balanceFactor := current.leftNode.getHeight() - current.rightNode.getHeight()

// left bias
if balanceFactor > 1 {
    if key > current.leftNode.key {
        current.leftNode = leftRotation(current.leftNode)
        return rightRotation(current)
    } else {
        return rightRotation(current)
    }
}

// right bias
if balanceFactor < -1 {
    if key < current.rightNode.key {
        current.rightNode = rightRotation(current.rightNode)
        return leftRotation(current)
    } else {
        return leftRotation(current)
    }
}
```

#### AVL Tree Deletion

Whenever a node is deleted from a tree we come across 3 scenarios.

1. The node being deleted is a leaf node.
2. The node being deleted has either only a left child or only a right child.
3. The node being deleted has both the left node and right node.

In the case of scenario (1) it's quite simple, just delete the node regularly with no extra checks. In case of scenario (2), we replace the deleted node with whichever child is not null. Scenario (3) involves a bit of extra work. We need to find the smallest element in the right subtree that is greater than the root node and replace the root node with the given node. We should be left with 2 instances of the node; at this point, we call the deletion procedure on the right subtree to remove the duplicate.

``` go
switch {
case key < current.key:
    {
        current.leftNode = t.delete(key, current.leftNode)
    }
case key > current.key:
    {
        current.rightNode = t.delete(key, current.rightNode)
    }
case key == current.key:
    {
        if current.isLeaf() {
            return nil
        }

        if current.leftNode == nil {
            current = current.rightNode
        } else if current.rightNode == nil {
            current = current.leftNode
        } else {
            temp := current.rightNode.getInOrder()
            current.key = temp.key
            current.value = temp.value

            current.rightNode = t.delete(temp.key, current.rightNode)
        }
    }
}
```

where the ``getInOrder`` function is defined like this

``` go
func (n *node[V]) getInOrder() *node[V] {
    if n.leftNode == nil {
        return n
    }

    return n.leftNode.getInOrder()
}
```

After we have deleted the node, we have to check the balance factor to see what kind of rotations we need to apply, as seen in the insertion procedure.

#### AVL Tree Search

The search procedure is quite simple: we start at the root node and compare keys. Once we find a key that matches, we return that key.

``` go
func (t *AvlTree[V]) search(key int,root *node[V]) *V {
    if root.key == key || root == nil {
        return &root.value
    }

    if key > root.key {
        return t.search(key,root.rightNode)
    }

    return t.search(key,root.leftNode)
}
```

This is going to form the basis on which we shall build our memtable. In the next part, we are going to be looking at how we interface this AVL tree with an iterator, how we can keep track of the size of the memtable, and creation of SS tables out of the memtable.