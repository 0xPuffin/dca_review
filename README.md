# DCA Review

Summary: A great concept and execution that seems like it could use a little bit of refinement and some additional safeguard around the math usage. My biggest recommendation is to pick either structs or ERC721 tokens to store the position data in, but not both. The number 1 most important thing I recommend is to replace the math with SafeMath, especially when using non even int values.

## General Comments

Since this contract already uses a few openzeppelin dependencies I recommend switching the math operations to SafeMath. For a few like the +/- 1 changes to length its probably overkill but for others the standard solidity arithmetic may lead to unexpected behavior (like silent overflow and underflow errors) that would be caught by using the SafeMath library. This could be especially unexpected for users in places like line 57 if their `sellDurationInDays` and `dailySellAmount` exceed the max value and overflow back around to a much smaller number then expected.

there are instances of using uint in multiple places in this contract. If you decide to implement SafeMath and even if you don't it may be advisable to explicitly declare the type of uint used like uint256 so others building on this contract are sure what types are expected/returned from different functions.

While there is a `exit` function on line 113 it contains a lot of logic that looks right to me but may contain some unknown bug we don't see. For this reason I recommend implementing an emergencyExit function that simply transfers all tokens of both types in the pair to the owner incase everything else fails for some reason. The contract's balance of each of the two tokens can be gotten using the ERC20 balanceOf function like `tokenToBuy.balanceOf(address(this))`.

A number of the contract level variables seem like they are intended to be used on a position specific basis but since they are implemented on the contract level and not in the struct or token for a specific position they will be updated when a function changing any position is changed and may cause unexpected behaviors. For example if position 1 is updated and `lastDaySold` changes then that may mean that for position 2 the `lastDaySold` is no longer accurate. This is also related to the comment about line 66. My overall recommendation for it is to perhaps create a DCAFactory contract that allows the minting of new DCA (or inherited from DCA) ERC721 tokens so that the important variables can be stored in each specific position and have no overlap. Alternatively every position specific variable could be moved into the struct and the token minting could be dropped. I recommend the ERC721 approach instead of the mapping of structs because then the positions could be made tradeable by users if they choose to sell them. 

## Line Specific Comments

DCA.sol line 26: Since the `userPositions` are called using a incremental ID there is no reason I see why `userPositions` can't be an array instead of a mapping. That way the gas costs of creating new positions would be reduced slightly as the new instance of the `UserPosition` struct would be able to be pushed to the array and there would be no need for an addition operation calculating the length to determine the next id. If this idea is adopted then it would be possible to also drop the `length` variable implemented on line 20.


DCA.sol line 61-65: the way that a new `UserPosition` is initialized can be shorted to look like
 ```
    userPositions[id] = UserPosition(
      dayStart,
      dailySellAmount,
      lastDay
    );
    
```
It is a slight change and may be a minus for readability but I is a plus for gas efficency of both the function call and the initial contract deployment.

DCA.sol line 66: Since these positons are stored in a mapping already it seems to be a extra gas cost to also issue them as a new token. I recommend choosing to either store the data all in the token and drop the mapping or drop the token and keep all the data in the mapping. Doing both raises the overall gas cost to call the function. Choosing one or the other could mean also reducing the number of total lines in the contract for example if you go with just the mapping then you could remove the burn statement on line 117.

DCA.sol line 81: since this contract is meant to be inherited by other contracts for different trading pairs it my be more flexible to allow the user to set the base decimal value when inheriting this contract. i.e. instead of using 1e18 let the user input the number represending decimals as a constructor variable so this contract can work with other ERC20s that may have more or less then 18 decimal places.

WETHtoDAI.sol line 39: this function is implemented as a internal pure but is never used anywhere else in either contract. If its meant for users it should be marked public and if its not intended to be used in the contracts or by the end users then it would lower deployment costs to remove it and store the url in a comment so contract readers can find it. 
