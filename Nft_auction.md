<p align="center">
  <a href="" rel="noopener">
 <img src="https://docs.reach.sh/assets/logo.png" alt="Project logo"></a>
</p>
<h3 align="center">Nft Auction</h3>

<div align="center">


</div>

---

<p align="center"> Workshop : Nft Auction
    <br> 
</p>

## Nft Auction
The decentralized application we built is an nft auction. The is a dapp were a user can place their nft up for auction and bidders place their bids, the highest bidder wins the nft and pays the price. The nft seller has to to specify the minimum bid that the bidders can start bidding from and also they have to specify how long the auction will last before it comes to an end.
We use a smart contract to ensure the auction is secure and safe, we use Reach to write this smart contract because its so secure and safe. This dapp was deployed on algorand.
The workshop assumes you have some experience with reach eg: completed the wisdom for sale tutorial or the Rock,paper,scissors tutorial [link](https://www.reach.sh).

# Code along
Ensure you follow the steps in the [Docs](https://www.reach.sh) to install reach on your system.
Run `./reach init` to generate two boiler plate files index.rsh and index.mjs

```js
1 'reach 0.1';

2  export const main = Reach.App(() => {
3    const Alice = Participant('Alice', {
4        getSale_details: Fun([], Object({
5            nftId: Token,
6            minBid: UInt,
7            Time: UInt,
8        })),
9        seeBid: Fun([Address, UInt], Null),
10        showOutcome: Fun([Address, UInt], Null),
11    });
12    const Bobs = API('Bobs', {
13        bid: Fun([UInt], Tuple(Address, UInt)),
14    });
15    const Bob = ParticipantClass('Bob', {
16        seeBid: Fun([Address, UInt], Null),
17        showOutcome: Fun([Address, UInt], Null),
18        optIn: Fun([Token], Null),
19    });
20    init();
21 })
```
* On line 1 we write the reach version, this is very important to ensure smooth compilation.
* From line 2 to line 19 we define the participants of the smart contract and their functions
* initaliziation of the reach application on line 20

```js
22 Alice.only(() => {
23     const { nftId, minBid, Time } = declassify(interact.getSale_details());
24    });
25 Alice.publish(nftId, minBid, Time);
26 const amt = 1;
27 commit();
28 Alice.pay([[amt, nftId]]);
29
30 Bob.interact.optIn(nftId);
31 Bob.interact.seeBid(Alice, minBid);
```
* On line 22 to 24 we access the values from the getSale_details() and publish them on line 25
* Alice pays the nft she wants to auction to the contract on line 28
* Bob opts into the nftid to be auctions and sees the miniumum bid published by Alice

```js
32 const end = lastConsensusTime() + Time;
33    const [
34        highestBidder,
35        lastPrice,
36        isFirstBid,
37    ] = parallelReduce([Alice, minBid, true])
38        .invariant(balance(nftId) == amt && balance() == (isFirstBid ? 0 : lastPrice))
39        .while(lastConsensusTime() <= end)
40        .api_(Bobs.bid,
41            (bid) => {
42                check(bid > lastPrice, "bid is too low,go higher");
43                return [bid, (notify) => {
44                    notify([highestBidder, lastPrice]);
45                    if (!isFirstBid) {
46                        transfer(lastPrice).to(highestBidder);
47                    }
48                    const who = this;
49                    Alice.interact.seeBid(who, bid);
50                    Bob.interact.seeBid(who, bid);
51                    return [who, bid, false];
52                }];
53            }
54        )
55        .timeout(absoluteTime(end), () => {
56            Alice.publish();
57            return [highestBidder, lastPrice, isFirstBid];
58        });
```
An Api call is written in the code block above, Bidders connect to the smart contract using this api to place their bids for the nft
* One line 32 we use the lastConsensusTime() to define how long the auction will last to gether with the deadline alice published
* From line 33 to line 54 we use the parallelReduce to write the api call. In the api we use the check() function that the next bid should be always higher than the previous bid, this is an easy was to end up with the highest bidder at the end of the auction.
* On line 55 we use the timeout function to end the reach application and transfer any token in the contract to the auctioner Alice, in a case where no one places a bid
You can read more about lastConsensusTime() function and paraReduce() in the reach [Docs](https://docs.reach.sh/rsh/)

```js
59 transfer(amt, nftId).to(highestBidder);
60 if (!isFirstBid) {
61        transfer(lastPrice).to(Alice);
62    }
63 Alice.only(() => {
64        interact.showOutcome(highestBidder, lastPrice);
65    });
66 Bob.only(() => {
67        interact.showOutcome(highestBidder, lastPrice);
68    });
69 commit();
70 exit();
```
* On line 59 we transfer the nft to the winner using the transfer function
* We then transfer the winning bid to Alice, display the outcome of the auction to both Alice and bob and exit the application