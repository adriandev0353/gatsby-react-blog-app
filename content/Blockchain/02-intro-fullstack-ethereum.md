# Intro to Fullstack Ethereum Development

This article will help you escape writing solidity tutorials in Remix and explain the tools you will need to create a simple full-stack dapp. The smart contract will be very simple itself and that is because we're focusing on all of the other tools you will need.

## Our stack

- [Solidity](https://docs.soliditylang.org) (To write our smart contract)
- [Hardat](https://hardhat.org/) (build, test and deployment framework)
- [React](https://reactjs.org/) (Create our frontend)
- [Metamask](https://metamask.io/) (Web Wallet that will allow us to interact with the ethereum blockchain) 
- [Ethers](https://docs.ethers.io) (web3 library for interacting with the blockchain and our smart contract)

## Installing a web wallet

Before getting started make sure you have a web wallet installed, I recommend [Metamask](https://metamask.io/download/). It is essentially just a browser extension that will allow us to interact with the ethereum blockchain. Just follow the instructions provided in the link to install and make sure not to use the same seed phrase for development for real ethereum/money. 

## Environment

First head over to the hardhat [website](https://hardhat.org/), we're going to be doing most of what is covered in the tutorial section as well as some of the documentation.

Make sure you have nodejs installed, if you don't then follow the setup [here](https://hardhat.org/tutorial/setting-up-the-environment.html) 

## Create a new project

To get your project started:

```
mkdir intro-fullstack-ethereum
cd intro-fullstack-ethereum
npm init --yes
npm i --save-dev hardhat
```

In the same directory where you installed Hardhat run:

```
npx hardhat
```

A menu will appear, for this tutorial we will be selecting `Create an empty hardhat.config.js`

Make sure these packages are installed, you may have been asked to install them when initializing the project.

```
npm install --save-dev @nomiclabs/hardhat-ethers ethers @nomiclabs/hardhat-waffle ethereum-waffle chai
```

## Create our Contract

Let's create our contract, it will be very simple, the contract will only read from and write to the blockchain. I wanted to keep the contract simple since this will be a comprehensive look at all of the tools necessary to create a fullstack dapp.

First we will need to create a directory for our contract, hardhat expects them to be in a directory called `contracts/` so:

```
mkdir contracts
cd contracts
touch SimpleStorage.sol
```

### Our Smart Contract

```
// SPDX-License-Identifier: MIT
pragma solidity >=0.5.0 <0.9.0;

contract SimpleStorage {
    uint256 storedData;

    constructor(uint256 _storedData) {
        storedData = _storedData;
    }

    function set(uint256 x) public {
        storedData = x;
    }

    function get() public view returns (uint256) {
        return storedData;
    }
}
```

TODO: explain contract

We can now use `hardhat` to compile our contract with

```
npx hardhat compile
```

## Shorthand commands

Instead of typing out `npx hardhat <command>` we can install `hardhat-shorthand` and use the `hh` command

```
npm i -g hardhat-shorthand
```

We can also get tab completion by running the command:

```
hardhat-completion install
```

Choose to install the autocompletion for your shell and you should now get tab completion after typing `hh` as long as you are in a hardhat project directory.

Try compiling your contract now with:

```
hh compile
```

## Testing

It is very import since money is often on the line when it comes to smart contracts. I will show you how to create a simple test for our `SimpleStorage` contract.

It is also important to note that hardhat comes with it's own network so when we run our tests hardhat will spin up a local network where we can deploy our contract to and test.

First create a directory called `test/` and create a file called `simple-storage-test.js`

```
mkdir `test`
touch simple-storage-test.js
```

Here are our simple tests:

```
const { expect } = require('chai')

describe('SimpleStorage contract', function () {
  it('test deployment', async function () {
    const SimpleStorage = await ethers.getContractFactory('SimpleStorage')

    const simpleStorage = await SimpleStorage.deploy(123)

    const storedValue = await simpleStorage.get()

    expect(storedValue).to.equal(123)
  })

  it('test set new value', async function () {
    const SimpleStorage = await ethers.getContractFactory('SimpleStorage')

    const simpleStorage = await SimpleStorage.deploy(123)

    await simpleStorage.set(456)

    const storedValue = await simpleStorage.get()

    expect(storedValue).to.equal(456)
  })
})
```

You will always be able to call the `deploy` method on your contract even if you didn't define a method called `deploy` essentially it just calls your constructor.

We have defined two tests here one just deploys the contract with an initial value and checks that it was deployed properly, the other does the same except we set a new value using the `set` method defined in our smart contract.


Before we can run our test we will need to `require` `hardhat-waffle` this will make the `ethers` variable available in global scope

So add the following line to the top of your `hardhat.config.js` file:

```
require("@nomiclabs/hardhat-waffle");
```

Ok now we are ready to run our test:

```
$ hh test # or npx hardhat test

  SimpleStorage contract
    ✓ test deployment (366ms)
    ✓ test set new value (52ms)


  2 passing (420ms)
```

You will notice the text that we added to the test is printed out when the test runs

**NOTE** Another good example test: [link](https://hardhat.org/tutorial/testing-contracts.html) 
### console.log in solidity

When running contracts inside of the hardhat network we can make use of a special logging function provided by hardhat, here is an example of how to add it to your contract:

```
// SPDX-License-Identifier: MIT
pragma solidity >=0.5.0 <0.9.0;

import "hardhat/console.sol";

contract SimpleStorage {
    uint256 storedData;

    constructor(uint256 _storedData) {
        console.log("Deployed by: ", msg.sender);
        console.log("Deployed with value: %s", _storedData);
        storedData = _storedData;
    }

    function set(uint256 x) public {
        console.log("Set value to: %s", x);
        storedData = x;
    }

    function get() public view returns (uint256) {
        console.log("Retrieved value: %s", storedData);
        return storedData;
    }
}
```

Now when we run `hh test` we should see:

```
  SimpleStorage contract
Deployed by:  0xf39fd6e51aad88f6f4ce6ab8827279cfffb92266
Deployed with value: 123
Retrieved value: 123
    ✓ test deployment (465ms)
Deployed by:  0xf39fd6e51aad88f6f4ce6ab8827279cfffb92266
Deployed with value: 123
Set value to: 456
Retrieved value: 456
    ✓ test set new value (74ms)
```

As you can see the logging function is very useful when trying to figure out what is happening with the internal logic of the smart contract.

## Deploying our contract

Now we are ready to deploy our contract, hardhat provides a local blockchain for testing so that we don't need to setup a node or use a third party provider to deploy to a testnet. We will first deploy to the local network and then optionally I will show you how to deploy to a testnet.

### Deployment script

Before we can deploy our contract we will need a deployment script so that `hardhat` knows how to instantiate the contract.

- First create a directory called `scripts/` and a file inside called `deploy.js`:

```
mkdir scripts/
touch scripts/deploy.js
```

- Here is some example code we can use to deploy our contract:

```
async function main() {
  // We get the contract to deploy
  const SimpleStorage = await ethers.getContractFactory("SimpleStorage");
  const greeter = await SimpleStorage.deploy(789);

  // NOTE: All Contracts have an associated address
  console.log("SimpleStorage deployed to:", greeter.address);
}

main()
  .then(() => process.exit(0))
  .catch((error) => {
    console.error(error);
    process.exit(1);
  });
```


### Local deployment

- Start a local node

```
hh node # or npx hardhat node
```

You should see that 20 accounts are created with 10000 ETH each.

- In another terminal deploy the contract:

```
npx hardhat run --network localhost scripts/deploy.js
```

In the terminal where you started your local node you should have noticed the following output:

```
  Contract deployment: SimpleStorage
  Contract address:    0x5fbdb2315678afecb367f032d93f642f64180aa3
  Transaction:         0x024bf3c1eb46ed3f405b7b10ea4c5e6b46c9e9deeed38d0743c6a7ae16b4d5b1
  From:                0xf39fd6e51aad88f6f4ce6ab8827279cfffb92266
  Value:               0 ETH
  Gas used:            290646 of 290646
  Block #1:            0x0eabbc7dff1b963c76b5b077d3df3353f201a450d72361b3de3218454cffe105

  console.log:
    Deployed by:  0xf39fd6e51aad88f6f4ce6ab8827279cfffb92266
    Deployed with value: 789
```

**NOTE** Notice the `console.log` section, we can utilize the builtin logging functionality whenever our contract is deployed on the `hardhat` network, this is not just available in testing.

## Creating our frontend

We will be using [React](https://reactjs.org/) since it is by far the most popular framework used to create frontends for dapps. 

Make sure you are in the root of our project (in the `intro-fullstack-ethereum` directory) and run:

```
npx create-react-app frontend
```

and make sure you can start your frontend by entering the `frontend` directory and running:

```
npm start
```

## Interacting with the blockchain (using ethers.js)

## Styling

## Testnet deployment (Optional)

