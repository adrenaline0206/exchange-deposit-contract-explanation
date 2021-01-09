## ExchangeDepositContract
This article is a commentary on [ExchangeDepositContract](https://github.com/bitbankinc/exchangeDepositContract).

- Overview of ExchangeDepositContract
    - What is Exchange Deposit Contract
- Details of ExchangeDepositContract
    - Explanation about code of Exchange Deposit Contract
- Payment confirmation of ExchangeDepositContract
    - Actual deposit to Exchange Deposit Contract

## Overview of ExchangeDepositContract
### ExchangeDepositContract configuration
The Exchange Deposit Contract consists of four contracts.
Forwarding Contracts are created from Proxy Factory Contracts. The contract address of the Forwarding Contract is the address to which the user deposits. This deposit address is given to each user, and the user will deposit ETH or ERC20 tokens to this address. The Exchange Deposit Contract acts as an intermediary for each contract, and the found  is finally transferred to the Cold Wallet through this contract. At that time, the event fires and the event history of deposit remains, so deposit detection is performed based on that record. Extra Implementation Contract is used to extend the logic.

![スクリーンショット 2020-12-31 17.00.07.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/268135/07baf480-c53a-b1a1-7016-c4d3e8fdeac4.png)

### Deposit flow
The deposit destination of ETH and ERC20 is the same contract address, but the flow after deposit is different. Here, we will roughly check the flow of deposit.

①ETH

- When depositing from User's EAO address to Forwarding Contract's contract address, calldata will be null because there is no external call

- Forwarding Contract will send the deposited ETH to the contract address of Exchange Deposit Contract by call if call data is null.

- The Exchange Deposit Contract sends the deposited ETH to the Cold Wallet with a call. At that time, emit fires a Deposit event. Payment is detected by collating this data with user information.

![スクリーンショット 2021-01-03 14.39.29.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/268135/a31af04d-4929-c0a4-a7ce-26e5ecfe30e1.png)

②ERC20 token

- Call ERC20 transfer from User's contract address to deposit ERC20 tokens from User's contract address to Forwarding Contract's contract address. The balance information of the ERC20 contract will be updated.

- Check the transfer event history of ERC20 and if you have the contract address of Forwarding Contract, you can see that there was a deposit.

- From admin (which can be executed even if you are not admin) to call gatherErc20 to Forwarding Contract to send ERC20 deposited in Forwarding Contract to Cold Wallet

- Forwarding Contract calls gatherErc20 of Exchange Deposit Contract with delegate call

- By calling ERC20 transfer from Forwarding Contract, the balance information of ERC20 contract will be updated

![スクリーンショット 2021-01-03 14.48.24.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/268135/03be5727-eccf-756a-c37a-981e32a81985.png)

## Details of ExchangeDepositContract

[Exchange Deposit Contract](https://github.com/bitbankinc/exchangeDepositContract) has 5 solidity files. ① and ② are directly related to the deposit detection system, and ③ to ⑤ are test contracts.

①ExchangeDeposit

It will be the main contract. This contract is deployed only once at the beginning.

②ProxyFactory

It is a contract that creates a Forwarding Contract for each user. The user makes a deposit to the Forwarding Contract contract address.

③【Test】SimpleCoin

It is a contract that imitates ERC20. This contract is used when testing ERC20.

④【Test】SimpleBadCoin

It is a contract that imitates ERC20. Used when testing bad cases of SimpleCoin.

⑤【Test】SampleLogic

It is a contract that simulates how it works when a new logic contract is specified from a proxy contract. Used when testing.


## Proxy Factory Contract

```javascript
contract ProxyFactory {
    bytes
        private constant INIT_CODE = hex'604080600a3d393df3fe7300000000000000000000000000000000000000003d366025573d3d3d3d34865af16031565b363d3d373d3d363d855af45b3d82803e603c573d81fd5b3d81f3';

    address payable private immutable mainAddress;

    constructor(address payable addr) public {
        require(addr != address(0), '0x0 is an invalid address');
        mainAddress = addr;
    }

    function deployNewInstance(bytes32 salt) external returns (address dst) {
        bytes memory initCodeMem = INIT_CODE;
        address payable addrStack = mainAddress;
        assembly {
        
            let pos := add(initCodeMem, 0x20)
            let first32 := mload(pos)
            let addrBytesShifted := shl(8, addrStack)
            
            mstore(pos, or(first32, addrBytesShifted))

            dst := create2(
                0,
                pos,
                74,
                salt
            )
            
            if eq(dst, 0) {
                revert(0, 0)
            }
        }
    }
}
```
It is writing the Forwarding Contract in bytecode to `INIT_CODE`.

```javascript
bytes private constant INIT_CODE = hex'604080600a3d393df3fe7300000000000000000000000000000000000000003d366025573d3d3d3d34865af16031565b363d3d373d3d363d855af45b3d82803e603c573d81fd5b3d81f3';
```

The bytecode below.

```
0x604080600a3d393df3fe7300000000000000000000000000000000000000003d366025573d3d3d3d34865af16031565b363d3d373d3d363d855af45b3d82803e603c573d81fd5b3d81f3
```

It is writing bytecode to memory with `bytes memory`. `initCodeMem` contains the byte length of` INIT_CODE`.

```
bytes memory initCodeMem = INIT_CODE;
```

`let pos: = add (initCodeMem, 0x20)` points to the beginning of the Forwarding Contract bytecode.

```javascript
let pos := add(initCodeMem, 0x20)
let first32 := mload(pos)
```

The first 32 bytes of the Forwarding Contract are stored in `first32`.

```
604080600a3d393df3fe7300000000000000000000000000000000000000003d
```

Now shift the addrStack (address) 8 bits to the left.

```javascript
let addrBytesShifted := shl(8, addrStack)
```

Remove 00 from the front and stick it to the back

```
Before shift：
000000000000000000000000AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA

After shift：
0000000000000000000000AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA00
```

`or` the first 32 bytes of the Forwarding Contract read from the shifted address and memory.

```javascript
mstore(pos, or(first32, addrBytesShifted))
```

The bytecode is formatted by overwriting `60408060 ...` in `FF` and` AA` in `00`.

```
60 40 80 60 0a 3d 39 3d f3 fe 73 0000000000000000000000000000000000000000 3d
or
00 00 00 00 00 00 00 00 00 00 00 AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA 00
```

The formatted result is as follows.

```
60 40 80 60 0a 3d 39 3d f3 fe 73 AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA 3d
```

## Forwarding Contract
The following bytecode defined in deployNewInstance of Proxy Factory Contract is Forwarding Contract.

```
60 40 80 60 0a 3d 39 3d f3 fe 73 AA AA AA AA AA AA AA AA AA AA AA AA AA AA AA AA AA AA AA AA 3d 36 60 25 57 3d 3d 3d 3d 34 86 5a f1 60 31 56 5b 36 3d 3d 37 3d 3d 36 3d 85 5a f4 5b 3d 82 80 3e 60 3c 57 3d 81 fd 5b 3d 81 f3
```

It would be like to explain what the Forwarding Contract is doing. At that time, a diagram showing the status of POS (position), OPCODE (opcode), OPCODE TEXT (content of opcode), and STACK (stack) is used (* 3).

※3 Please refer to [Ethereum Virtual Machine Opcodes] (https://ethervm.io/) for OPCODE.

### Forwarding Contract deployment code
This part is the deployment code for deploying the executable code.

```
60 40 80 60 0a 3d 39 3d f3 fe 
```

- POS 00〜06

0x40 (64 bytes) of POS 0a to 3F (execution code) is copied to the 0x0 area of the memory.

- POS 09

The OPCODE `fe` stands for` INVALID` and means the boundary between the deploy code and the execution code.

| POS | OPCODE | OPCODE TEXT | STACK |
| --- | --- | --- | --- |
| 00 | 60 40 | PUSH1 0x40 | 0x40 |
| 02 | 80 | DUP 1 | 0x40 0x40 |
| 03 | 60 0a | PUSH1 0x0a | 0x0a 0x40 0x40 |
| 05 | 3d | RETURNDATASIZE | 0x0 0x0a 0x40 0x40 |
| 06 | 39 | CODECOPY | 0x40 |
| 07 | 3d | RETURNDATASIZE | 0x0 0x40 |
| 08 | f3 | RETURN | |
| 09 | fe | INVALID |  |

### Forwarding Contractの実行コード
In this part, branching is performed depending on whether the length of call data is 0.

```
73 AA AA AA AA AA AA AA AA AA AA AA AA AA AA AA AA AA AA AA AA 3d 36 60 25 57
```

- POS 00〜19

If the length of calldata is 0, POS 1A bytecode is continued without jumping. Jumps to POS 25 if the calldata length is non-zero.

| POS | OPCODE | OPECODE TEXT | STACK |
| --- | --- | --- | --- |
| 00 | 73 ... | PUSH20 ... | {ADDR} |
| 15 | 3d | RETURNDATASIZE | 0x0 {ADDR} |
| 16 | 36 | CALLDATASIZE | CDS 0x0 {ADDR} |
| 17 | 6025 | PUSH1 0x25 | 0x25 CDS 0x0 {ADDR} |
| 19 | 57 | JUMPI | 0x0 {ADDR} |

### Processing when calldata length = 0
Execute the call when the length of calldata = 0 (no external call, which means simple remittance).

```
3d 3d 3d 3d 34 86 5a f1 60 31 56
```

- POS 1A〜21

When executing CALL, you need to specify some arguments. {RES} returns a Boolean value indicating whether the CALL was successful.

```
call(g, a, v, in, insize, out, outsize)
```

g: Gas used for remittance
a: Remittance destination address
v: Amount of ether to send (unit is wei)
in: Memory location where the data to call is held
insize: data size
out: Memory location to hold the return data
outsize: data size

- POS 22〜24

Be sure to jump to POS 31.

| POS | OPCODE | OPECODE TEXT | STACK |
| --- | --- | --- | --- |
| 1A | 3d | RETURNDATASIZE | 0x0 0x0 {ADDR} |
| 1B | 3d | RETURNDATASIZE | 0x0 0x0 0x0 {ADDR} |
| 1C | 3d | RETURNDATASIZE | 0x0 0x0 0x0 0x0 {ADDR} |
| 1D | 3d | RETURNDATASIZE | 0x0 0x0 0x0 0x0 0x0 {ADDR} |
| 1E | 34 | CALLVALUE | VALUE 0x0 0x0 0x0 0x0 0x0 {ADDR} |
| 1F | 86 | DUP7 | {ADDR} VALUE 0x0 0x0 0x0 0x0 0x0 {ADDR} |
| 20 | 5a | GAS | GAS {ADDR} VALUE 0x0 0x0 0x0 0x0 0x0 {ADDR} |
| 21 | f1 | CALL | {RES} 0x0  {ADDR} |
| 22 | 6031 | PUSH1 0x31 | 0x31{RES} 0x0  {ADDR} |
| 24 | 56 | JUMP | {RES} 0x0  {ADDR} |

### Processing when calldata length! = 0
Execute a delegate call when the calldata length! = 0.

```
5b 36 3d 3d 37 3d 3d 36 3d 85 5a f4
```

- POS 25〜30

DELEGATECALL is basically the same as CALL in terms of arguments, but it retains the caller and calldata.

```
delegatecall(g, a, in, insize, out, outsize)
```

| POS | OPCODE | OPECODE TEXT | STACK |
| --- | --- | --- | --- |
| 25 | 5b | JUMPDEST | 0x0 {ADDR} |
| 26 | 36 | CALLDATASIZE | CDS 0x0 {ADDR} |
| 27 | 3d | RETURNDATASIZE | 0x0 CDS 0x0 {ADDR} |
| 28 | 3d | RETURNDATASIZE | 0x0 0x0 CDS 0x0 {ADDR} |
| 29 | 37 | CALLDATACOPY | 0x0 {ADDR} |
| 2A | 3d | RETURNDATASIZE | 0x0 0x0 {ADDR} |
| 2B | 3d | RETURNDATASIZE | 0x0 0x0 0x0 {ADDR} |
| 2C | 36 | CALLDATASIZE | CDS 0x0 0x0 0x0 {ADDR} |
| 2D | 3d | RETURNDATASIZE | 0x0 CDS 0x0 0x0 0x0 {ADDR} |
| 2E | 85 | DUP6 | {ADDR} 0x0 CDS 0x0 0x0 0x0 {ADDR} |
| 2F | 5a | GAS | GAS {ADDR} 0x0 CDS 0x0 0x0 0x0 {ADDR} |
| 30 | f4 | DELEGATECALL | {RES} 0x0  {ADDR} |

### Branch if CALL or DELEGATE CALL succeeds or fails
In this part, processing is performed according to the result of executing CALL or DELEGATE CALL above.

```
5b3d82803e603c573d81fd5b3d81f3
```

- POS 31-38
If RES (result of execution) is 0 (failure), the processing after POS 39 is performed as it is, and if RES is 1 (success), it jumps to POS 3C.

- POS 39 ~ 3B
Restores (reverts) the state and returns data from 0x0 to RDS.

- POS 3C ~ 3F
Returns data from 0x0 to RDS.

| POS | OPCODE | OPECODE TEXT | STACK |
| --- | --- | --- | --- |
| 31 | 5b | JUMPDEST | {RES} 0x0 {ADDR} |
| 32 | 3d | RETURNDATASIZE | RDS {RES} 0x0 {ADDR} |
| 33 | 82 | DUP3 | 0x0 RDS {RES} 0x0 {ADDR} |
| 34 | 80 | DUP1 | 0x0 0x0 RDS {RES} 0x0 {ADDR} |
| 35 | 3e | RETURNDATACOPY | {RES} 0x0 {ADDR} |
| 36 | 60 3c | PUSH1 0x3c | 0x3c RDS {RES} 0x0 {ADDR} |
| 38 | 57 | JUMPI | 0x0 {ADDR} |
| 39 | 3d | RETURNDATASIZE | RDS 0x0 {ADDR} |
| 3A | 81 | DUP2 | 0x0 RDS 0x0 {ADDR} |
| 3B | fd | REVERT | 0x0 {ADDR} |
| 3C | 5b | JUMPDEST | 0x0 {ADDR} |
| 3D | 3d | RETURNDATASIZE | RDS 0x0 {ADDR} |
| 3E | 81 | DUP2 | 0x0 RDS 0x0 {ADDR} |
| 3F | f3 | RETURN | 0x0  {ADDR} |


## Payment confirmation of ExchangeDepositContract

### Import library
Use OpenZeppelin, a library of smart contracts. OpenZeppelin is used to secure the processing commonly used in contracts.

```javascript
import '@openzeppelin/contracts/token/ERC20/IERC20.sol';
import '@openzeppelin/contracts/token/ERC20/SafeERC20.sol';
import '@openzeppelin/contracts/utils/Address.sol';
```

When using the library, write something like `using ... for ...`. SafeERC20 is a basic implementation of the ERC20 interface, for example the safeTransfer function can securely transfer ERC20 tokens. IERC20 is an interface that must be compliant when implementing ERC20. Address deals with functions related to the address type, for example the isContract function returns whether it is a contract account or not as a boolean.

```javascript
using SafeERC20 for IERC20;
using Address for address payable;
```

### Variables
It will be the address of the cold wallet. By adding the `payable` qualifier, you will be able to send money to this contract address.

 ```javascript
 address payable public coldAddress;
 ```

This is the minimum deposit amount for the user to deposit. Since 1ETH = 1e18 (10 to the 18th power), 1e16 means 0.01ETH.

 ```javascript
 uint256 public minimumInput = 1e16;
 ```

The address of the logic contract that the proxy contract calls with DELEGATE CALL.

```javascript
address payable public implementation;
```

With an address that has the authority to control the ExchangeDeposit contract, it is possible to forcibly stop the contract, disable logic transfer, change the cold address and implementation address, and so on.
The `immutable` keyword is a read-only value that stores the value directly in the contract body rather than in storage.

```javascript
address payable public immutable adminAddress;
```

The instance address of the Exchange Deposit Contract. It is used to identify whether this address is a proxy address or an Exchange Deposit address.

```javascript
address payable private immutable thisAddress;
```

### Constructor

Checking if coldAddr and adminAddr are not 0 addresses (invalid addresses).

```javascript
require(coldAddr != address(0), '0x0 is an invalid address');
require(adminAddr != address(0), '0x0 is an invalid address');
```

The address of this contract is set in thisAddress.

```javascript
coldAddress = coldAddr;
adminAddress = adminAddr;
thisAddress = address(this);
```

### Event
Events are used to log transactions. The Deposit event logs deposits sent from the Forwarding contract. In the argument, specify the address of the sender of the funds and the amount sent.

```javascript
event Deposit(address indexed receiver, uint256 amount);
```

### Modifier
Qualifiers are specified when you want to perform a specific operation before executing a function. The contract that executes the function with this qualifier is executed when the following conditions are met.

Executes the internal code if it is adminAddress.

```javascript
modifier onlyAdmin {
    require(msg.sender == adminAddress, 'Unauthorized caller');
    _;
}
```

The internal code is executed when the cold address is not 0 due to the forced stop.

```javascript
modifier onlyAlive {
    require(
        getExchangeDepositor().coldAddress() != address(0),
        'I am dead :-('
    );
    _;
}
```

Executes internal code when ExchangeDeposit is called with a call.

```javascript
modifier onlyExchangeDepositor {
    require(isExchangeDepositor(), 'Calling Wrong Contract');
    _;
}
```
### Function

** isExchangeDepositor function **

Checks if the code context is Exchange Deposit or Proxy and returns a Boolean value.

```javascript
function isExchangeDepositor() internal view returns (bool) {
    return thisAddress == address(this);
}
```

** getExchangeDepositor function **

If the code context is ExchangeDeposit, it returns an instance of the ExchangeDeposit contract, otherwise it returns an instance of the ExchangeDeposit contract using the ExchangeDeposit function with the address of this contract as an argument.

```javascript
function getExchangeDepositor() internal view returns (ExchangeDeposit) {
    return isExchangeDepositor() ? this : ExchangeDeposit(thisAddress);
}
```

** getImplAddress function **

This is a function to get the implementation address. If the code context is ExchangeDeposit, it returns implementation (instance address of the logic contract), otherwise it returns the instance address with `implementation ()` after instantiating with the ExchangeDeposit function.

```javascript
function getImplAddress() internal view returns (address payable) {
    return
        isExchangeDepositor()
            ? implementation
            : ExchangeDeposit(thisAddress).implementation();
}
```

** getSendAddress function **

This is a function to get the sendTo address of the gatherEth and gatherErc20 functions. This is usually the address of the cold wallet, but if the ExchangeDeposit contract has been terminated, it will be the adminAddress.

```javascript
function getSendAddress() internal view returns (address payable) {
    ExchangeDeposit exDepositor = getExchangeDepositor();

    address payable coldAddr = exDepositor.coldAddress();
    address payable toAddr = coldAddr == address(0)
        ? exDepositor.adminAddress()
        : coldAddr;
    return toAddr;
}
```

** gatherErc20 function **

Transfer the ERC20 token to the cold wallet (adminAddress if it has been killed).

```javascript
function gatherErc20(IERC20 instance) external {
    uint256 forwarderBalance = instance.balanceOf(address(this));
    if (forwarderBalance == 0) {
        return;
    }
    instance.safeTransfer(getSendAddress(), forwarderBalance);
}
```

** gatherEth function **

If ETH is given by selfdestruct (* 4) to another contract, the receive function cannot handle it. In that case, it is a function to transfer ETH to the cold wallet (adminAddress if it is forcibly terminated).

```javascript
function gatherEth() external {
    uint256 balance = address(this).balance;
    if (balance == 0) {
        return;
    }
    (bool result, ) = getSendAddress().call{ value: balance }('');
    require(result, 'Could not gather ETH');
}
```

* 4 Function that deletes a contract

** changeColdAddress function **
You can change the cold address to a new address. At that time, the conditions must be as follows.
① The execution context is Exchange Deposit
② Not forcibly terminated
③ adminAddress

```javascript
function changeColdAddress(address payable newAddress)
    external
    onlyExchangeDepositor
    onlyAlive
    onlyAdmin
{
    require(newAddress != address(0), '0x0 is an invalid address');
    coldAddress = newAddress;
}
```

** changeImplAddress function **

You can change the implementation address to a new address. The conditions are the same as for the changeColdAddress function. The address to be changed must be a 0 address (so that it can be returned to the non-existent state) or a contract address.

```javascript
function changeImplAddress(address payable newAddress)
    external
    onlyExchangeDepositor
    onlyAlive
    onlyAdmin
{
    require(
        newAddress == address(0) || newAddress.isContract(),
        'implementation must be contract'
    );
    implementation = newAddress;
}
```


** changeMinInput function **

You can change the minimum deposit amount. The conditions are the same as for the changeColdAddress function.

```javascript
function changeMinInput(uint256 newMinInput)
    external
    onlyExchangeDepositor
    onlyAlive
    onlyAdmin
{
    minimumInput = newMinInput;
}
```

** kill function **

Set the cold address to 0 and kill the contract.

```javascript
function kill() external onlyExchangeDepositor onlyAlive onlyAdmin {
    coldAddress = address(0);
}
```

### receive and fallback
** receive function **

The receive function is similar to the fallback function, but is called when ETH is sent in a transaction without call data. If there is no receive function, the fallback function will be called.

```javascript
receive() external payable {
    require(coldAddress != address(0), 'I am dead :-(');
    require(msg.value >= minimumInput, 'Amount too small');
    (bool success, ) = coldAddress.call{ value: msg.value }('');
    require(success, 'Forwarding funds failed');
    emit Deposit(msg.sender, msg.value);
}
```

Check if the cold address has been killed and is not below the minimum deposit amount.

```javascript
require(coldAddress != address(0), 'I am dead :-(');
require(msg.value >= minimumInput, 'Amount too small');
```

Forwards ETH to the cold address. `emit` is used to output the transaction log recorded in` event`. The address of the source of funds and the amount of money are specified and output.

```javascript
(bool success, ) = coldAddress.call{ value: msg.value }('');
require(success, 'Forwarding funds failed');
emit Deposit(msg.sender, msg.value);
```

** fallback function **

This fallback function is executed when a function that is not in the ExchangeDeposit contract is called.

```javascript
fallback() external payable onlyAlive {
    address payable toAddr = getImplAddress();
    require(toAddr != address(0), 'Fallback contract not set');
    (bool success, ) = toAddr.delegatecall(msg.data);
    require(success, 'Fallback contract failed');
}
```

Get the ImplAddress (address of the logic contract) and call msg.data as call data with a delegate call to that address.

```javascript
address payable toAddr = getImplAddress();
(bool success, ) = toAddr.delegatecall(msg.data);
```

## Payment confirmation of ExchangeDepositContract
Finally, in order to check the deposit flow, I would like to confirm the process of depositing ETH and ERC20 tokens to the deposit address on the test network and reaching the cold address.

### Environment
MetaMask: 8.1.9
solidity: 0.6.11
Remix:
Etherscan:

### Deposit ETH
① Cooperation between MetaMask and Remix

Change the Remix `ENVIRONMENT` to` Injected Web 3`. Log in to MetaMask and select `Ropsten Testnet` to work together.

![スクリーンショット 2021-01-03 22.21.06.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/268135/b05f3f34-4e59-d5e8-15f9-776c2e56e795.png)

②MetaMask settings

Create AdminAccount, ColdAccount, TransferAccount in MetaMask. Deposit ETH from Faucet to Admin Account and Transfer Account.

![スクリーンショット 2021-01-03 22.44.26.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/268135/8ec971a6-92f4-bf0a-2ee0-a443b22fccc2.png)

③ Remix settings

Compile ExchangeDepositContract and ProxyFactoryContract together in one file.


④ Deploy Exchange Deposit Contract

--Deploy Exchange Deposit Contract with Admin Account. Set Contract to Exchange Deposit
--Specify `cold address (Cold Account address)` and `admin address (Admin Account address)` as arguments and press `transact`.
--The MetaMask window will open, so press `Confirm`.

![スクリーンショット 2021-01-04 7.44.25.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/268135/d0fc6306-c862-bc33-dd19-be539239699e.png)

⑤ Deploy ProxyFactory Contract

--Deploy ProxyFactoryContract with AdminAccount. Set Contract to ProxyFactoryContract
--Specify the contract address of ExchangeDepositContract deployed in ④ as an argument and press `transact`.
--The MetaMask window will open, so press the `Confirm` button.

![スクリーンショット 2021-01-04 7.44.25.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/268135/7767f0f9-86eb-2d3b-239a-8563ad3c78d4.png)

⑥ Create deployNewInstance

--Execute the deployNewInstance function of ProxyFactoryContract deployed in ⑤
--Specify a 32-byte salt as an argument (any 32-byte hexadecimal number is OK)

```
Example：
0x0000000000000000000000AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA00
```

![スクリーンショット 2021-01-04 8.07.55.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/268135/8429d8b7-1da3-4ddb-d9eb-92b6dbc32e12.png)

⑦Check the address of deployNewInstance

Click the Etherscan URL to see the details of deployNewInstance

The part surrounded by red is the address of deployNewInstance

![スクリーンショット 2021-01-04 8.13.18.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/268135/544d1bf3-a83f-fa4a-8dd7-3dbec003d035.png)


⑧ Remittance to the deposit address

Try sending 0.5 ETH from the user's account (Transfer Account) to the deposit address (deployNewInstance contract address) that is actually assigned to each user using MetaMask.


⑨ Confirmation of payment

It was confirmed that the amount of 0.5 ETH sent from the user's account (Transfer Address) to the user's deposit address (deploy New Instance Address) has arrived at the cold address (Cold Address).

![スクリーンショット 2021-01-04 8.52.34.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/268135/9d4b5db0-21c2-4212-04b7-12b6a2cfc20b.png)

### Deposit ERC20 Token

Then, make a deposit using MetaMask and Remix. The procedure up to the above `ETH deposit` step ⑦ is the same.

① Deploy SimpleCoinContract

![スクリーンショット 2021-01-04 22.23.37.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/268135/2dbd2eac-1e45-7f1e-c008-4ad8b96a5b7a.png)


② Preparation for remittance
Send a token to TransferAccount with the giveBalance function of SimpleCoinContract.

![スクリーンショット 2021-01-04 22.24.31.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/268135/a06de716-d668-f800-b316-d20b152f8a15.png)


③ Send money to the deposit address

Transfer the token from the Transfer Account to the deposit address.

④ Check the token Transfer event history

If you refer to the Transfer event history with the contract address of the token, you can confirm that there was a deposit at the deposit address.

![スクリーンショット 2021-01-04 22.53.50.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/268135/e973b106-cefc-17d7-c750-e8c9cffce81b.png)


⑤ Instantiate Exchange Deposit Contract
--The ERC20 token is not automatically sent to the call address, so you need to send money from the deposit address to the cold address.
--Instantiate Exchange Deposit Contract with the deposit address (enter the deposit address in the At Address field and press it)

![スクリーンショット 2021-01-04 22.58.42.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/268135/f37711d4-049e-98dd-3c4c-bf53ae1b11b7.png)

⑥ Remittance to cold address

Use the gatherErc20 function of ExchangeDepositContract instantiated with the deposit address to send money to the cold address. Specify the contract address of the token in the argument of the gatherErc20 function.

![スクリーンショット 2021-01-04 23.00.33.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/268135/25794233-514e-63bd-9445-3dd606f82369.png)

⑦ Confirmation of payment

I was able to confirm that the amount sent from the Transfer Address to the user's deposit address has arrived at the cold address.

![スクリーンショット 2021-01-04 23.02.43.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/268135/e0e79a3f-e658-65bd-e739-6c3d13936fc1.png)

## References
- [Ethereum Whitepaper](https://ethereum.org/en/whitepaper/)
- [Ethereum yellow paper](https://ethereum.github.io/yellowpaper/paper.pdf) 
- [Ethereum Virtual Machine Opcodes](https://ethervm.io/)
- [Solidity Document](https://solidity.readthedocs.io/en/v0.6.11/)
