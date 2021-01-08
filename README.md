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
Forwarding Contracts are created from Proxy Factory Contracts. The contract address of the Forwarding Contract is the address to which the user deposits. This deposit address is given to each user, and the user will deposit ETH or ERC20 tokens to this address. The Exchange Deposit Contract acts as an intermediary for each contract, and the money is finally transferred to the Cold Wallet through this contract. At that time, the event fires and the event history of deposit remains, so deposit detection is performed based on that record. Extra Implementation Contract is used to extend the logic.

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

## ExchangeDepositContractの説明
それではここからは入金システムの具体的な説明に移りたいと思います。
[Exchange Deposit Contract](https://github.com/bitbankinc/exchangeDepositContract)には５つのsolidityファイルがあります。入金検知システムに直接関わるのは①と②で③〜⑤はテストのためのコントラクトになります。

①ExchangeDeposit
メインのコントラクトになります。このコントラクトは、はじめに1度だけデプロイされます。

②ProxyFactory
ユーザーごとにForwarding Contractを作成するコントラクトになります。ユーザーはForwarding Contractのコントラクトアドレスに対して入金を行います。

③【テスト用】SimpleCoin
ERC20を模擬したコントラクトになります。このコントラクトはERC20のテストする際に使用します。

④【テスト用】SimpleBadCoin
ERC20を模擬したコントラクトになります。SimpleCoinの悪い事例をテストする際に使用します。

⑤【テスト用】SampleLogic
プロキシコントラクトから新しいロジックのコントラクトを指定した際にどの様に機能するのか模擬したコントラクトです。テストする際に使用します。


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
Forwarding Contractをバイトコードで`INIT_CODE`に書き込んでいます。

```javascript
bytes private constant INIT_CODE = hex'604080600a3d393df3fe7300000000000000000000000000000000000000003d366025573d3d3d3d34865af16031565b363d3d373d3d363d855af45b3d82803e603c573d81fd5b3d81f3';
```
具体的には下記のバイトコードです。

```
0x604080600a3d393df3fe7300000000000000000000000000000000000000003d366025573d3d3d3d34865af16031565b363d3d373d3d363d855af45b3d82803e603c573d81fd5b3d81f3
```

`bytes memory`でメモリにバイトコードを書き込んでいます。`initCodeMem`には`INIT_CODE`のバイト長が含まれます。

```
bytes memory initCodeMem = INIT_CODE;
```

`let pos := add(initCodeMem, 0x20)`はForwarding Contractのバイトコードの先頭を指しています。

```javascript
let pos := add(initCodeMem, 0x20)
let first32 := mload(pos)
```

`first32`にはForwarding Contractの先頭の32byteが格納されます。

```
604080600a3d393df3fe7300000000000000000000000000000000000000003d
```

ここでaddrStack(アドレス)を8bit左にシフトします。

```javascript
let addrBytesShifted := shl(8, addrStack)
```

00を前から取り除き、後ろにくっつけます

```
シフト前：
000000000000000000000000AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA

シフト後：
0000000000000000000000AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA00
```

シフトしたアドレスとメモリから読み込んだForwarding Contractの先頭の32byteを`or`します。

```javascript
mstore(pos, or(first32, addrBytesShifted))
```

`FF`に`60408060...`を、`00`に`AA`を上書きすることで、バイトコードを整形しています。

```
60 40 80 60 0a 3d 39 3d f3 fe 73 0000000000000000000000000000000000000000 3d
or
00 00 00 00 00 00 00 00 00 00 00 AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA 00
```

整形した結果は以下の通りになります。

```
60 40 80 60 0a 3d 39 3d f3 fe 73 AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA 3d
```

## Forwarding Contract
Proxy Factory ContractのdeployNewInstanceに定義されている下記のバイトコードがForwarding Contractになります。

```
60 40 80 60 0a 3d 39 3d f3 fe 73 AA AA AA AA AA AA AA AA AA AA AA AA AA AA AA AA AA AA AA AA 3d 36 60 25 57 3d 3d 3d 3d 34 86 5a f1 60 31 56 5b 36 3d 3d 37 3d 3d 36 3d 85 5a f4 5b 3d 82 80 3e 60 3c 57 3d 81 fd 5b 3d 81 f3
```

Forwarding Contractがどの様な処理を行っているのか説明したいと思います。その際にPOS(ポジション)、OPCODE(オペコード)、OPCODE TEXT(オペコードの内容)、STACK(スタック)の状態を表した図を使用します(※3)。

※3 OPCODEについては [Ethereum Virtual Machine Opcodes](https://ethervm.io/)をご参照して下さい
### Forwarding Contractのデプロイコード
この部分は実行コードをデプロイする為のデプロイコードになります。

```
60 40 80 60 0a 3d 39 3d f3 fe 
```

- POS 00〜06

メモリの0x0領域にPOSの0a〜3F(実行コード)の0x40(64byte)をコピーしています。

- POS 09

OPCODEの`fe`は`INVALID`を表しデプロイコードと実行コードの境目を意味します。

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
この部分ではcalldataの長さが0かどうかで分岐を行っています。

```
73 AA AA AA AA AA AA AA AA AA AA AA AA AA AA AA AA AA AA AA AA 3d 36 60 25 57
```

- POS 00〜19

calldataの長さが0だった場合はジャンプせずに引き続きPOS 1Aのバイトコードを実行します。calldataの長さが0以外の場合はPOS 25にジャンプします。


| POS | OPCODE | OPECODE TEXT | STACK |
| --- | --- | --- | --- |
| 00 | 73 ... | PUSH20 ... | {ADDR} |
| 15 | 3d | RETURNDATASIZE | 0x0 {ADDR} |
| 16 | 36 | CALLDATASIZE | CDS 0x0 {ADDR} |
| 17 | 6025 | PUSH1 0x25 | 0x25 CDS 0x0 {ADDR} |
| 19 | 57 | JUMPI | 0x0 {ADDR} |

### calldataの長さ = 0の場合の処理
calldataの長さ = 0(外部呼び出しがない、つまり単純送金を意味しています)の場合にcallを実行します。

```
3d 3d 3d 3d 34 86 5a f1 60 31 56
```

- POS 1A〜21

CALLを実行する際には、いくつかの引数を指定する必要があります。{RES}はCALLが成功したかをブール値で返します。

```
call(g, a, v, in, insize, out, outsize)
```

g: 送金する際に使用するガス
a: 送金先のアドレス
v: 送金するetherの量(単位はwei)
in: callするデータが保持されるメモリの位置
insize: データのサイズ
out: 戻り値のデータを保持するメモリの位置
outsize: データサイズ

- POS 22〜24

必ずPOS 31にジャンプします。

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

### calldataの長さ != 0 の場合の処理
calldataの長さ != 0の場合にdelegatecallを実行します。

```
5b 36 3d 3d 37 3d 3d 36 3d 85 5a f4
```

- POS 25〜30

DELEGATECALLは基本的に引数などCALLと同じですが、呼び出し元とcalldataは保持します。

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

### CALLもしくはDELEGATECALLが成功か失敗した場合の分岐
この部分では前記でCALLもしくはDELEGATECALLを実行した結果に応じた処理を行っています。

```
5b3d82803e603c573d81fd5b3d81f3
```

- POS 31〜38
RES(実行した結果)が0(失敗)の場合はそのままPOS 39以降の処理を行い、RESが1(成功)の場合はPOS 3Cにジャンプします。

- POS 39〜3B
状態を元に戻し(リバートして)0x0からRDSまでのデータを返します。

- POS 3C〜3F
0x0からRDSまでのデータを返します。

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


## ExchangeDeposit Contract

### ライブラリのインポート
スマートコントラクトのライブラリであるOpenZeppelinを使用します。OpenZeppelinはコントラクトでよく使われる処理をセキュアに行う目的で使用します。

```javascript
import '@openzeppelin/contracts/token/ERC20/IERC20.sol';
import '@openzeppelin/contracts/token/ERC20/SafeERC20.sol';
import '@openzeppelin/contracts/utils/Address.sol';
```
ライブラリを使用する際は`using...for...`の様な書き方をします。SafeERC20はERC20インターフェースの基本実装で、例えばsafeTransfer関数はERC20のトークンを安全に転送することができます。IERC20はERC20を実装する際に準拠する必要があるインターフェイスです。Addressはaddressタイプに関連する関数を扱うもので、例えばisContract関数はコントラクトアカウントかどうかをbooleanで返します。

```javascript
using SafeERC20 for IERC20;
using Address for address payable;
```

### 変数
コールドウォレットのアドレスになります。`payable`修飾子をつけることで、このコントラクトアドレスに対して送金することが出来る様になります。

 ```javascript
 address payable public coldAddress;
 ```

 ユーザーが入金する際の最低入金金額です。1ETH=1e18(10の18乗)なので1e16は0.01ETHと言う意味です。

 ```javascript
 uint256 public minimumInput = 1e16;
 ```

プロキシコントラクトがDELEGATECALLで呼び出す、ロジックコントラクトのアドレスです。

```javascript
address payable public implementation;
```

ExchangeDepositコントラクトをコントロールする権限を持ったアドレスで、コントラクトの強制停止、ロジックの転送の無効化、コールドアドレスやimplementationアドレスの変更などをすることが可能です。
`immutable`キーワードは読み取り専用の値で、ストレージではなくコントラクト本体に直接値を格納するものになります。

```javascript
address payable public immutable adminAddress;
```

ExchangeDeposit Contractのインスタンスアドレスです。このアドレスがプロキシアドレスかExchangeDepositアドレスかを識別するために使用します。

```javascript
address payable private immutable thisAddress;
```

### コンストラクタ

coldAddrとadminAddrが0アドレスでないか (無効なアドレスでないか)チェックしています。

```javascript
require(coldAddr != address(0), '0x0 is an invalid address');
require(adminAddr != address(0), '0x0 is an invalid address');
```

thisAddressには、このコントラクトのアドレスを設定しています。

```javascript
coldAddress = coldAddr;
adminAddress = adminAddr;
thisAddress = address(this);
```

### イベント
イベントはトランザクションのログ記録する為に使用します。DepositイベントはForwardingコントラクトから送金された入金のログを記録します。引数には資金の送り元のアドレスと送った金額を指定します。

```javascript
event Deposit(address indexed receiver, uint256 amount);
```

### 修飾子
修飾子は関数を実行する前に特定の処理を実行したい場合に指定します。この修飾子をつけた関数を実行するコントラクトが、以下の条件だった場合に実行されます。

adminAddressである場合に内部のコードを実行します。

```javascript
modifier onlyAdmin {
    require(msg.sender == adminAddress, 'Unauthorized caller');
    _;
}
```

強制停止によりコールドアドレスが０アドレスになっていない場合に内部のコードを実行します。

```javascript
modifier onlyAlive {
    require(
        getExchangeDepositor().coldAddress() != address(0),
        'I am dead :-('
    );
    _;
}
```

ExchangeDepositがcallで呼び出された場合に内部のコードを実行します。

```javascript
modifier onlyExchangeDepositor {
    require(isExchangeDepositor(), 'Calling Wrong Contract');
    _;
}
```
### 関数

**isExchangeDepositor関数**

コードの文脈がExchangeDepositなのかProxyなのかチェックしブール値で返します。

```javascript
function isExchangeDepositor() internal view returns (bool) {
    return thisAddress == address(this);
}
```

**getExchangeDepositor関数**

コードの文脈がExchangeDepositの場合ExchangeDepositコントラクトのインスタンスを返し、そうでない場合はExchangeDeposit関数を使い引数にこのコントラクトのアドレスを指定しExchangeDepositコントラクトのインスタンスを返します。

```javascript
function getExchangeDepositor() internal view returns (ExchangeDeposit) {
    return isExchangeDepositor() ? this : ExchangeDeposit(thisAddress);
}
```

**getImplAddress関数**

implementationアドレスを取得する為の関数です。コードの文脈がExchangeDepositの場合はimplementation(ロジックコントラクトのインスタンスアドレス)を返し、そうでない場合はExchangeDeposit関数でインスタンス化した後に`implementation()`でインスタンスアドレスを返します。

```javascript
function getImplAddress() internal view returns (address payable) {
    return
        isExchangeDepositor()
            ? implementation
            : ExchangeDeposit(thisAddress).implementation();
}
```

**getSendAddress関数**

gatherEthやgatherErc20の関数のsendToアドレスを取得するための関数です。通常はコールドウォレットのアドレスですが、ExchangeDepositコントラクトが強制終了している場合はadminAddressになります。

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

**gatherErc20関数**

ERC20のトークンをコールドウォレット(強制終了している場合はadminAddress)に転送します。

```javascript
function gatherErc20(IERC20 instance) external {
    uint256 forwarderBalance = instance.balanceOf(address(this));
    if (forwarderBalance == 0) {
        return;
    }
    instance.safeTransfer(getSendAddress(), forwarderBalance);
}
```

**gatherEth関数**

他のコントラクトがselfdestruct(※4)によってETHを付与された場合にreceive関数では対応できません。その場合にETHをコールドウォレット(強制終了している場合はadminAddress)に転送する為の関数です。

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

※4 コントラクトを削除する関数

**changeColdAddress関数**
コールドアドレスを新しいアドレスに変更することが出来ます。その際に条件として以下である必要があります。
①実行する文脈がExchangeDeposit
②強制終了されていない
③adminAddressである

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

**changeImplAddress関数**

implementationアドレスを新しいアドレスに変更することが出来ます。条件はchangeColdAddress関数と同様です。変更するアドレスは0アドレス(ない状態に戻せる様にする為)もしくはコントラクトアドレスである必要があります。

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


**changeMinInput関数**

最低入金額を変更することが出来ます。条件はchangeColdAddress関数と同様です。

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
**kill関数**

コールドアドレスを0アドレスにして、コントラクトを強制終了します。

```javascript
function kill() external onlyExchangeDepositor onlyAlive onlyAdmin {
    coldAddress = address(0);
}
```

### receiveとfallback
**receive関数**

receive関数はfallback関数に似ていますが、calldataがない取引にてETHが送金された場合に呼び出されます。receive関数がない場合はfallback関数が呼ばれます。

```javascript
receive() external payable {
    require(coldAddress != address(0), 'I am dead :-(');
    require(msg.value >= minimumInput, 'Amount too small');
    (bool success, ) = coldAddress.call{ value: msg.value }('');
    require(success, 'Forwarding funds failed');
    emit Deposit(msg.sender, msg.value);
}
```

コールドアドレスがkillされていないか、最低入金金額を下回っていないかをチェックします。

```javascript
require(coldAddress != address(0), 'I am dead :-(');
require(msg.value >= minimumInput, 'Amount too small');
```

コールドアドレスにETHを転送します。`emit`は`event`で記録したトランザクションのログを出力する際に使用します。資金の送り元のアドレスとその金額を指定して出力しています。

```javascript
(bool success, ) = coldAddress.call{ value: msg.value }('');
require(success, 'Forwarding funds failed');
emit Deposit(msg.sender, msg.value);
```

**fallback関数**

ExchangeDepositコントラクトにない関数を呼ばれた場合に、このfallback関数が実行されます。

```javascript
fallback() external payable onlyAlive {
    address payable toAddr = getImplAddress();
    require(toAddr != address(0), 'Fallback contract not set');
    (bool success, ) = toAddr.delegatecall(msg.data);
    require(success, 'Fallback contract failed');
}
```

ImplAddress(ロジックコントラクトのアドレス)を取得しそのアドレスに対してdelegatecallでmsg.dataをcalldataとして呼び出します。

```javascript
address payable toAddr = getImplAddress();
(bool success, ) = toAddr.delegatecall(msg.data);
```

## ExchangeDepositContractの入金確認
最後に入金の流れを確認する為、テストネットワーク上で入金アドレスにETHとERC20のトークンを入金し、コールドアドレスに届く過程を確認してみたいと思います。

### 環境
MetaMask 8.1.9
solidity version 0.6.11
Remix
Etherscan

### ETHの入金
①MetaMaskとRemixの連携

Remixの`ENVIRONMENT`を`Injected Web3`に変更します。MetaMaskにログインし`Ropstenテストネット`を選択して連携を行います。

![スクリーンショット 2021-01-03 22.21.06.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/268135/b05f3f34-4e59-d5e8-15f9-776c2e56e795.png)

②MetaMaskの設定

MetaMaskでAdminAccount, ColdAccount, TransferAccountを作成します。AdminAccountとTransferAccountにFaucetからETHを入金しておきます。

![スクリーンショット 2021-01-03 22.44.26.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/268135/8ec971a6-92f4-bf0a-2ee0-a443b22fccc2.png)

③Remixの設定

ExchangeDepositContractとProxyFactoryContractを１つのファイルにまとめてコンパイルします。


④ExchangeDepositContractをデプロイ

- AdminAccountでExchange Deposit Contractをデプロイします。ContractをExchangeDepositに設定して下さい
- 引数に`cold address(ColdAccountのアドレス)`と`admin address(AdminAccountのアドレス)`を指定して`transact`を押下します
- MetaMaskのウィンドウが開きますので`確認`を押下します

![スクリーンショット 2021-01-04 7.44.25.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/268135/d0fc6306-c862-bc33-dd19-be539239699e.png)

⑤ProxyFactoryContractをデプロイ

- AdminAccountでProxyFactoryContractをデプロイします。ContractをProxyFactoryContractに設定して下さい
- 引数に④でデプロイしたExchangeDepositContractのコントラクトアドレスを指定して`transact`を押下します
- MetaMaskのウィンドウが開きますので`確認`ボタンを押します

![スクリーンショット 2021-01-04 7.44.25.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/268135/7767f0f9-86eb-2d3b-239a-8563ad3c78d4.png)

⑥deployNewInstanceの作成

- ⑤でデプロイしたProxyFactoryContractのdeployNewInstance関数を実行します
- 引数に32byteのsaltを指定します(16進数32byteであればなんでもOK)

```
例：
0x0000000000000000000000AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA00
```

![スクリーンショット 2021-01-04 8.07.55.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/268135/8429d8b7-1da3-4ddb-d9eb-92b6dbc32e12.png)

⑦deployNewInstanceのアドレスを確認

deployNewInstanceの詳細を確認する為にEtherscanのURLをクリックします

赤で囲った部分がdeployNewInstanceのアドレスになります

![スクリーンショット 2021-01-04 8.13.18.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/268135/544d1bf3-a83f-fa4a-8dd7-3dbec003d035.png)


⑧入金用のアドレスへ送金

MetaMaskを使って実際にユーザ毎に割り当てられる入金用のアドレス(deployNewInstanceのコントラクトアドレス)に対して、ユーザーのアカウント(TransferAccount)から0.5ETH送金してみます。


⑨入金の確認

ユーザーのアカウント(Transfer Address)から0.5ETHをユーザーの入金用アドレス(deployNewInstanceAddress)に送金した金額がコールドアドレス(ColdAddress)に届いているのが確認出来ました。

![スクリーンショット 2021-01-04 8.52.34.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/268135/9d4b5db0-21c2-4212-04b7-12b6a2cfc20b.png)

### ERC20トークンの入金

それではMetaMaskとRemixを使って入金します上記`ETHの入金`の工程⑦までは同じ作業になります。

①SimpleCoinContractをデプロイ

![スクリーンショット 2021-01-04 22.23.37.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/268135/2dbd2eac-1e45-7f1e-c008-4ad8b96a5b7a.png)


②送金の準備
SimpleCoinContractのgiveBalance関数でTransferAccountにトークンを送ります。
![スクリーンショット 2021-01-04 22.24.31.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/268135/a06de716-d668-f800-b316-d20b152f8a15.png)


③入金用のアドレスへ送金

TransferAccountから入金用のアドレスにトークンを送金します。

④トークンのTransferイベント履歴を調べる

トークンのコントラクトアドレスでTransferイベント履歴を参照すると入金アドレスに入金があったことが確認できます。

![スクリーンショット 2021-01-04 22.53.50.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/268135/e973b106-cefc-17d7-c750-e8c9cffce81b.png)


⑤ExchangeDepositContractをインスタンス化
- ERC20トークンは自動でコールアドレスに送金されないので、入金アドレスからコールドアドレスに送金する操作が必要になります
- 入金用のアドレスでExchangeDepositContractをインスタンス化(At Addressの欄に入金アドレスを入力し押下)します

![スクリーンショット 2021-01-04 22.58.42.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/268135/f37711d4-049e-98dd-3c4c-bf53ae1b11b7.png)



⑥コールドアドレスへの送金

入金用アドレスでインスタンス化したExchangeDepositContractのgatherErc20関数を使ってコールドアドレスへの送金を行います。gatherErc20関数の引数にはトークンのコントラクトアドレスを指定します。

![スクリーンショット 2021-01-04 23.00.33.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/268135/25794233-514e-63bd-9445-3dd606f82369.png)

⑦入金の確認

Transfer Addressからユーザーの入金用アドレスに送金した金額がコールドアドレスに届いているのが確認出来ました。

![スクリーンショット 2021-01-04 23.02.43.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/268135/e0e79a3f-e658-65bd-e739-6c3d13936fc1.png)

## 参考文献
- [Ethereum Whitepaper](https://ethereum.org/en/whitepaper/)
- [Ethereum yellow paper](https://ethereum.github.io/yellowpaper/paper.pdf) 
- [Ethereum Virtual Machine Opcodes](https://ethervm.io/)
- [Solidity Document](https://solidity.readthedocs.io/en/v0.6.11/)
