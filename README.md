# 🏗 scaffold-eth | 🏰 BuidlGuidl

## 🚩 Challenge 5: Minimum Viable Exchange

This challenge will provide a tutorial to help you build/understand a simple decentralized exchange, with one token-pair (ERC20 BALLOONS ($BAL) and ETH). This readme is an upated version of the [original tutorial](https://medium.com/@austin_48503/%EF%B8%8F-minimum-viable-exchange-d84f30bd0c90). Please read the intro for a background on what we are building! There is also a Youtube video that may help you understand the concepts covered within this challenge too: https://www.youtube.com/watch?v=eP5w6Ger1EQ&t=364s&ab_channel=SimplyExplained.

This branch was heavily based off of this branch: https://github.com/scaffold-eth/scaffold-eth-challenges/tree/challenge-5-dex.

---

### Checkpoint 0: 📦 install 📚
# 🏄‍♂️ Quick Start

Prerequisites: [Node (v16 LTS)](https://nodejs.org/en/download/) plus [Yarn](https://classic.yarnpkg.com/en/docs/install/) and [Git](https://git-scm.com/downloads)

_TODO: Update this with appropriate links_

```bash
git clone https://github.com/scaffold-eth/scaffold-eth-challenges.git challenge-5-dex
cd challenge-5-dex
git checkout challenge-5-dex
yarn install
```

### Checkpoint 1: 🔭 Environment 📺

You'll have three terminals up for:

`yarn start` (react app frontend)

`yarn chain` (hardhat backend)

`yarn deploy` (to compile, deploy, and publish your contracts to the frontend)

Navigate to the Debug Contracts tab and you should see two smart contracts displayed called `DEX` and `Balloons`.

> 👩‍💻 Rerun `yarn deploy` whenever you want to deploy new contracts to the frontend (run `yarn deploy --reset` for a completely fresh deploy if you have made no contract changes).

`Balloons.sol` is just an example ERC20 contract that mints 1000 to whatever address deploys it.
`DEX.sol` is what we will build in this tutorial and you can see it starts with a SafeMath library to help us prevent overflows and underflows and also tracks a token (ERC20 interface) that we set in the constructor (on deploy).

---

### Checkpoint 2: Reserves

**Feature Details:** We want to create an automatic market where our contract will hold reserves of both ETH and 🎈 Balloons. These reserves will provide liquidity that allows anyone to swap between the assets.

Let’s add a couple new variables to `DEX.sol`:

```
uint256 public totalLiquidity;
mapping (address => uint256) public liquidity;
```

These variables track the total liquidity, but also by individual addresses too.
Then, let’s create an init() function in `DEX.sol` that is payable and then we can define an amount of tokens that it will transfer to itself:

```
function init(uint256 tokens) public payable returns (uint256) {
  require(totalLiquidity==0,"DEX:init - already has liquidity");
  totalLiquidity = address(this).balance;
  liquidity[msg.sender] = totalLiquidity;
  require(token.transferFrom(msg.sender, address(this), tokens));
  return totalLiquidity;
}
```

Calling init() will load our contract up with both ETH and 🎈 Balloons.

We can see that the DEX starts empty. We want to be able to call init() to start it off with liquidity, but we don’t have any funds or tokens yet. Add some ETH to your local account using the faucet and then find the `00_deploy_your_contract.js` file. Uncomment the line below and add your address:

```
  // // paste in your address here to get 10 balloons on deploy:
  // await balloons.transfer("YOUR_ADDRESS","" + (10 * 10 ** 18));
```

Run `yarn deploy`. The front end should show you that you have balloon tokens. We can’t just call init() yet because the DEX contract isn’t allowed to transfer tokens from our account. We need to approve() the DEX contract with the Balloons UI. Copy and paste the DEX address and then set the amount to 5000000000000000000 (5 _ 10¹⁸). You can confirm this worked using the allowance() function. Now we are ready to call init() on the DEX. We will tell it to take (5 _ 10¹⁸) of our tokens and we will also send 0.01 ETH with the transaction. You can see the DEX contract's value update and you can check the DEX token balance using the balanceOf function on the Balloons UI.

This works pretty well, but it will be a lot easier if we just call the init() function as we deploy the contract. In the `00_deploy_your_contract.js` script try uncommenting the init section so our DEX will start with 3 ETH and 3 Balloons of liquidity:

```
  // // uncomment to init DEX on deploy:
  // console.log("Approving DEX ("+dex.address+") to take Balloons from main account...")
  // // If you are going to the testnet make sure your deployer account has enough ETH
  // await balloons.approve(dex.address, ethers.utils.parseEther('100'));
  // console.log("INIT exchange...")
  // await dex.init("" + (3 * 10 ** 18), {value:ethers.utils.parseEther('3'), gasLimit:200000})
```

Now when we `yarn deploy --reset` then our contract should be initialized as soon as it deploys and we should have equal reserves of ETH and tokens.

---

### Checkpoint 3: Price

**Feature Details:** Follow along with the [original tutorial](https://medium.com/@austin_48503/%EF%B8%8F-minimum-viable-exchange-d84f30bd0c90) Price section for an understanding of the DEX's pricing model and for a price function to add to your contract. You may need to update the Solidity syntax (e.g. use + instead of .add, \* instead of .mul, etc). Deploy when you are done.

### 🥅 Goals TODO

- [ ] Do you understand how the x\*y=k price curve actually works? Write down a clear explanation for yourself and derive the formula for price. You might have to shake off some old algebra skills!
- [ ] You should be able to go through the price section of this tutorial with the sample numbers and generate the same outputChange variable.

> Hints: See this link (https://hackernoon.com/formulas-of-uniswap-a-deep-dive), solve for the change in the Output Reserve. Also, don't forget to think about how to implement the trading fee. Solidity doesn't allow for decimals, so one way that contracts are written to implement percentage is using whole uints (997 and 1000) as numerator and denominator factors, respectively.

---

### Checkpoint 4: Trading

**Feature Details:** Let’s edit the DEX.sol smart contract and add two new functions for swapping from each asset to the other.

<details markdown='1'><summary>**Solution Code (don't look until you have tried!)** </summary>

```
function ethToToken() public payable returns (uint256) {
  uint256 token_reserve = token.balanceOf(address(this));
  uint256 tokens_bought = price(msg.value, address(this).balance - msg.value, token_reserve);
  require(token.transfer(msg.sender, tokens_bought));
  return tokens_bought;
}

function tokenToEth(uint256 tokens) public returns (uint256) {
  uint256 token_reserve = token.balanceOf(address(this));
  uint256 eth_bought = price(tokens, token_reserve, address(this).balance);
  (bool sent, ) = msg.sender.call{value: eth_bought}("");
  require(sent, "Failed to send user eth.");
  require(token.transferFrom(msg.sender, address(this), tokens));
  return eth_bought;
}
```

</details>

Each of these functions calculate the resulting amount of output asset using our price function that looks at the ratio of the reserves vs the input asset. We can call tokenToEth and it will take our tokens and send us ETH or we can call ethToToken with some ETH in the transaction and it will send us tokens. Let’s deploy our contract then move over to the frontend. Exchange some ETH for tokens and some tokens for ETH!

---

### Checkpoint 5: Liquidity

**Feature Details:** So far, only the init() function controls liquidity. To make this more decentralized, it would be better if anyone could add to the liquidity pool by sending the DEX both ETH and tokens at the correct ratio.
Let’s create two new functions that let us deposit and withdraw liquidity. How would you write this function out? Try before taking a peak!

<details markdown='1'><summary>**Solution Code (don't look until you have tried!)** </summary>
📣 Make sure you update the `InfuraID` before you go to production. Huge thanks to [Infura](https://infura.io/) for our special account that fields 7m req/day!

# 🏃💨 Speedrun Ethereum
Register as a builder [here](https://speedrunethereum.com) and start on some of the challenges and build a portfolio.

# 💬 Support Chat

```
function deposit() public payable returns (uint256) {
  uint256 eth_reserve = address(this).balance - msg.value;
  uint256 token_reserve = token.balanceOf(address(this));
  uint256 token_amount = ((msg.value * token_reserve) / eth_reserve) + 1;
  uint256 liquidity_minted = (msg.value * totalLiquidity) / eth_reserve;
  liquidity[msg.sender] += liquidity_minted;
  totalLiquidity += liquidity_minted;
  require(token.transferFrom(msg.sender, address(this), token_amount));
  return liquidity_minted;
}

function withdraw(uint256 liq*amount) public returns (uint256, uint256) {
uint256 token_reserve = token.balanceOf(address(this));
uint256 eth_amount = (liq_amount * address(this).balance) / totalLiquidity;
uint256 token*amount = (liq_amount * token_reserve) / totalLiquidity;
liquidity[msg.sender] -= liq_amount;
totalLiquidity -= liq_amount;
(bool sent, ) = msg.sender.call{value: eth_amount}("");
require(sent, "Failed to send user eth.");
require(token.transfer(msg.sender, token_amount));
return (eth_amount, token_amount);
}

```

Take a second to understand what these functions are doing after you paste them into DEX.sol in packages/hardhat/contracts:
The deposit() function receives ETH and also transfers tokens from the caller to the contract at the right ratio. The contract also tracks the amount of liquidity the depositing address owns vs the totalLiquidity.
The withdraw() function lets a user take both ETH and tokens out at the correct ratio. The actual amount of ETH and tokens a liquidity provider withdraws will be higher than what they deposited because of the 0.3% fees collected from each trade. This incentivizes third parties to provide liquidity. </details>

### 🥅 Goals TODO

- [ ] Deposit liquidity, and then check your liquidity amount through the mapping. Has it changed properly? Did the right amount of assets get deposited?
- [ ] What happens if you deposit(), then another user starts swapping out for most of the balloons, and then you try to withdraw your position as a liquidity provider? Answer: you should get the amount of liquidity proportional to the ratio of assets within the isolated liquidity pool.

---

### Checkpoint 6: UI

TODO: <details markdown='1'><summary>Context for people newer to ReactJS</summary> For those that are really new to anything front-end development, there are many resources out there to possibly use. This one was particularly helpful from minutes 15 to 20 describing the typical folder structure within a ReactJS project.
https://www.youtube.com/watch?v=w7ejDZ8SWv8&ab_channel=TraversyMedia

 </details>

TODO:
\*From a fresh master branch off of scaffold-eth repo, we found the following was needed to get things hooked up with the front-end:

1. Update index.js file within components sub-directory to include some things from the OG challenge repos:

```
export { default as Dex } from "./DEX";
export { default as Curve } from "./Curve";
```

2. Other files you'll need from OG repo: DEX.jsx, Curve.jsx
   TODO: INSERT LINKS TO OG REPO

3. You will likely run into errors from your front-end assuming you've ran `yarn start` already. Let's fix those!

Find useEventListener, useContractLoader, useContractReader, useBlockNumber, useBlanace, useTokenBalance within DEX.jsx file. You will see them calling for .hooks --> delete those and replace with the following:

```
import { useEventListener } from "eth-hooks/events/useEventListener";
import { useContractLoader } from "eth-hooks";
import { useContractReader } from "eth-hooks";
import { useBlockNumber } from "eth-hooks";
import { useBalance } from "eth-hooks";
import { useTokenBalance } from "eth-hooks/erc/erc-20/useTokenBalance";
```

These replacements are needed because the pointers within the `DEX.jsx` and `Curve.jsx` files from the OG repo are not accurate with the `master branch` off of `scaffold-eth` repo.

In 00_deploy_your_contracts.js, you'll have to write in the necessary code to get your contracts, using Ethers.js.

As well, make sure that the tags are updated to your contract names, it should be something like `module.exports.tags = ["YourContract"];` and you'll want to change it to:

[![Open in Gitpod](https://gitpod.io/button/open-in-gitpod.svg)](https://gitpod.io/#github.com/scaffold-eth/scaffold-eth)
