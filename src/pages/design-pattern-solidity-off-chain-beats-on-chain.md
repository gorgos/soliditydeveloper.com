---
title: 'Design Pattern Solidity: Off-chain beats on-chain'
date: 2019-09-14T22:36:30.692Z
description: Off-chain beat on-chain
featuredImage: ../../static/img/broken_chain.png
---
As you might have realized, Ethereum transactions are anything but cheap. In particular, if you are computing complex things or storing a lot of data. That means sometimes we cannot put all logic inside Solidity.

Instead, we can utilize off-chain computations to help us. A very simple example would be:

- - -

**Don't**

```javascript
function storeSum(uint256 a, uint256 b) {
  storedSum = a + b;
}
```

**Do**

```javascript
function storeSum(uint256 _storedSum) {
  storedSum = _storedSum;
}
```

- - -

You compute `a + b` before sending out the transaction. Very simple. Now this simple pattern is valid for almost everything and it can be sometimes quite challenging finding the most efficient way to do more off-chain.

## Sorted Ranking Example with Off-Chain Sorting

Let's look at a more complex example. This was something I actually did recently. The requirement was to have a ranking for users inside the smart contract. At first glance that might sound very easy, but a ranking means having a sorted array. And keeping a sorted array of length `n` means on average for a new insertion an added complexity of `n/2` (for finding the correct insert position).

At best, we have only a small ranking, then this just means it costs quite a bit more gas for every insertion. At worst, we have a large ranking and might even get out-of-gas exceptions, rendering a serious security risk for the contract.

**Solution**

* Using a [circular linked list](https://github.com/modular-network/ethereum-libraries/blob/master/LinkedListLib), we can easily insert and remove users in the list, but we need to modify the library to use a different mapping (`mapping (address => mapping (bool => address)) list`).
* Having a separate mapping for the user points `mapping (address => uint256) public userPoints`.
* Pre-computing the sorted spot for a newly inserted user - you might have guessed it - off-chain.
* Only verifying the correctness of the suggested insert position when inserted a new user.

I won't go into code examples, as this will get quite long and messy. But I will give you the ideas and outline each function:

 **getSortedSpot** 

\`function getSortedSpot(address _user, uint256 _points) public view returns (address)\`

This will be a view function inside your contract. It iterates from bottom to top through your ranking linked list while in each iteration comparing the user points of the current list address to the given `_points`. Once you find the first address in the list that has more points, return it as reference. You will call `getSortedSpot` before inserting a new user to find out the correct insertion position.

Things to consider:

* Don't return a reference address that equals `_user`. As this means an existing user in the ranking will get a new position.
* If there are no addresses with more points, we are dealing with the new user becoming the first rank. Return the current first rank as reference.

**insertUser** 

`function sortedInsertUser(address user, address referenceUser) public`

This will be your actual insertion method. You pass the result from `getSortedSpot` as `referenceUser`. And now we can just verify that the reference is indeed correct:

1. Compute the new points for `user` based on whatever your metrics are.
2. Compare the computed points to those of `referenceUser`. The referenced user must have more points.
3. Compare the computed points to those of one rank below `referenceUser`. One rank below must have less points.
4. Insert the user into the ranking.

Things to consider:

* The first and last rank require special consideration.
* If the passed `user` already exists in the ranking, remove it before newly inserting it or you will get double entries.
