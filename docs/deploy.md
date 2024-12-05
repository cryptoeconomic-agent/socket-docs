---
id: deploy-contracts
title: Deploying onchain smart contracts
---

# How to deploy contracts to chains?

## 1. Deploy

Deployments of onchain contracts from Offchain VM is done using a Deployer contract. Lets look at the deployer of `MyToken` example to better understand.

![image.png](../static/img/deploy1.png)

```solidity
contract MyTokenDeployer is AppDeployerBase {
    address public myToken;

    constructor(
        address addressResolver_,
        FeesData memory feesData_,
        string calldata name_,
        string calldata symbol_,
        uint8 decimals_
    ) AppDeployerBase(addressResolver_, feesData_) {
        myToken = address(new MyToken(name_, symbol_, decimals_));
        creationCodeWithArgs[myToken] = abi.encodePacked(
            type(MyToken).creationCode,
            abi.encode(name_, symbol_, decimals_)
        );
    }

    function deployContracts(
        uint32 chainSlug
    ) external async {
        _deploy(myToken, chainSlug);
    }

    function initialize(uint32 chainSlug) public override async {}
}
```

Key things to note here are -

It extends the `AppDeployerBase` to manage the deployment process.

In the `constructor`, a copy of `MyToken` is deployed to the offchainVM, and its `creationCode` along with constructor parameters is stored in a mapping. This stored code is used for deploying the token to the underlying chains. The constructor also takes in `addressResolver` and `feesData`, we will talk more on these at a later stage. Or you can read more about them [here](https://www.notion.so/How-to-call-smart-contracts-14d818fd2858808281cce0ca530b2e66?pvs=21).

The `deployContracts` function takes a `chainSlug` as an argument, specifying the chain where the contract should be deployed. It calls the inherited `_deploy` function and uses the `async` modifier for interacting with underlying chains. When you call \_deploy, both the onchain contract and its [forwarder](https://www.notion.so/How-to-call-smart-contracts-14d818fd2858808281cce0ca530b2e66?pvs=21) are deployed.

The `initialize` function is empty in this example, lets see when that can be used next.

## 2. Initialize

Since we store the `creationCode` along with `constructor parameters`, they essentially become constants. But there can be use cases where the contract need dynamic or chain specific values while setting up. For such cases, the initialize flow has to be used. Lets extend the `MyToken` example to set mint limits.

![image.png](../static/img/deploy2.png)

```solidity
contract MyToken is ERC20 {
    ...

    uint256 mintLimit;

    function mint(address to_, uint256 amount_) external onlySOCKET {
        require(amount_ <= mintLimit, "more than mint limit");
        _mint(to_, amount_);
    }

    function setMintLimit(uint256 newLimit) external onlySOCKET {
        mintLimit = newLimit;
    }
}
```

We will set this limit using `initialize` function, and to make things a bit more dynamic, we will set a higher limit for Ethereum compared to chains.

```solidity
contract MyTokenDeployer is AppDeployerBase {
    ...

    function initialize(uint32 chainSlug) public override async {
        uint256 mintLimit;
        if (chainSlug == 1) {
            mintLimit = 10 ether;
        } else {
            mintLimit = 1 ether;
        }

        MyToken(forwarderAddresses[myToken][chainSlug]).setMintLimit(
            mintLimit
        );
    }
}
```

The initialize function follows similar flow to how you make on [chain calls](https://www.notion.so/How-to-call-smart-contracts-14d818fd2858808281cce0ca530b2e66?pvs=21) using the `async` modifier and `forwarderAddress`.

You can also note that the forwarder addresses of deployed contracts are stored in `forwarderAddresses` mapping in the `AppDeployerBase` and can be accessed easily here.

## 3. Deploy multiple contracts

So far we have been working with a single `MyToken` contract on chain. But the deployer also supports working with multiple contracts. Lets create `MyTokenVault` to lock tokens on chain and extend the deployer to deploy both contracts.

```solidity
contract MyTokenVault {
    address public myToken;
    address public _SOCKET;

    // user => amount
    mapping(address => uint256) public lockedAmount;

    constructor() {
        _SOCKET = msg.sender;
    }

    error NotSOCKET();

    modifier onlySOCKET() {
        if (msg.sender != _SOCKET) revert NotSOCKET();
        _;
    }

    function setMyToken(address myToken_) external onlySOCKET {
        myToken = myToken_;
    }

    function lock(uint256 amount) external {
        MyToken(myToken).transferFrom(msg.sender, address(this), amount);
        lockedAmount[msg.sender] += amount;
    }

    function unlock(uint256 amount) external {
        lockedAmount[msg.sender] -= amount;
        MyToken(myToken).transfer(msg.sender, amount);
    }
}
```

This contract needs to be on chain, therefore lets change `MyTokenDeployer` to include it as well.

```solidity
contract MyTokenDeployer is AppDeployerBase {
    address public myToken;
    address public myTokenVault;

    constructor(
        address addressResolver_,
        FeesData memory feesData_,
        string calldata name_,
        string calldata symbol_,
        uint8 decimals_
    ) AppDeployerBase(addressResolver_, feesData_) {
        myToken = address(new MyToken(name_, symbol_, decimals_));
        creationCodeWithArgs[myToken] = abi.encodePacked(
            type(MyToken).creationCode,
            abi.encode(name_, symbol_, decimals_)
        );

        myTokenVault = address(new MyTokenVault());
        creationCodeWithArgs[myTokenVault] = type(MyTokenVault).creationCode;
    }

    function deployContracts(uint32 chainSlug) external async {
        _deploy(myToken, chainSlug);
        _deploy(myTokenVault, chainSlug);
    }

    function initialize(uint32 chainSlug) public override async {
        uint256 mintLimit;
        if (chainSlug == 1) {
            mintLimit = 10 ether;
        } else {
            mintLimit = 1 ether;
        }

        address myTokenForwarder = forwarderAddresses[myToken][chainSlug];
        address myTokenVaultForwarder = forwarderAddresses[myTokenVault][chainSlug];
        address myTokenOnChainAddress = getOnChainAddress(myToken, chainSlug);

        MyToken(myTokenForwarder).setMintLimit(
            mintLimit
        );
        MyTokenVault(myTokenVaultForwarder).setMyToken(
            myTokenOnChainAddress
        );
    }
}
```

This `MyTokenDeployer` deploys both contracts, sets limit on `MyToken` and sets `MyToken’`s onchain address on `MyTokenVault`. Key things to note in this contract -

- `MyTokenVault` doesnt have any constructor arguments. Therefore we can directly store its `creationCode` without encoding anything along with it.
- We can get the forwarder addresses of both these from `forwarderAddresses` mapping.
- Since `MyTokenVault` locks `MyToken`, its needs the token’s on chain address. This address can be fetched using `getOnChainAddress` function.