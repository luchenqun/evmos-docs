---
sidebar_position: 5
---

# IBC 转账

## Solidity 接口和 ABI

`ICS20.sol` - 之前被称为 `IBCTransfer.sol` - 是一个接口，通过它 Solidity 合约可以与 Evmos 链上的 IBC 协议进行交互。
对于开发者来说，这非常方便，因为他们不需要了解 [IBC-go](https://ibc.cosmos.network/) 中 `transfer` 模块的具体实现细节。
相反，他们可以使用他们熟悉的以太坊接口进行 IBC 转账。
在 [evmos/extensions 仓库](https://github.com/evmos/extensions/tree/main/examples/simple-ibc-transfer) 中可以找到一个简单实现的示例。

### 接口 `ICS20.sol`

在 [evmos/extensions 仓库中找到 Solidity 接口](https://github.com/evmos/extensions/blob/main/precompiles/stateful/ICS20.sol)。

### ABI

在 [evmos/extensions 仓库中找到 ABI](https://github.com/evmos/extensions/blob/main/precompiles/abi/ics20.json)。

## 交易

- `approve`

    ```solidity
    /// @dev 批准特定数量的代币进行 IBC 转账。
    /// @param spender 将要花费资金的地址。
    /// @param allocations 授权的分配。
    function approve(
        address spender,
        Allocation[] calldata allocations
    ) external returns (bool approved);
    ```

- `revoke`

    ```solidity
    /// @dev 撤销特定地址的 IBC 转账授权。
    /// @param spender 将要花费资金的地址。
    function revoke(address spender) external returns (bool revoked);
    ```

- `increaseAllowance`
  
    ```solidity
    /// @dev 增加给定地址的授权金额，并为 IBC 转账方法提供 IBC 连接。
    /// @param spender 将要花费资金的地址。
    /// @param sourcePort IBC 交易的源端口。
    /// @param sourceChannel IBC 交易的源通道。
    /// @param denom 转账代币的单位。
    /// @param amount 要花费的代币数量。
    function increaseAllowance(
        address spender,
        string calldata sourcePort,
        string calldata sourceChannel,
        string calldata denom,
        uint256 amount
    ) external returns (bool approved);
    ```

- `decreaseAllowance`

  ```solidity
  /// @dev 减少给定spender的特定数量的令牌的授权额度，用于IBC转账方法。
  /// @param spender 将花费资金的地址。
  /// @param sourcePort IBC交易的源端口。
  /// @param sourceChannel IBC交易的源通道。
  /// @param denom 转移的令牌的面额。
  /// @param amount 要花费的令牌数量。
  function decreaseAllowance(
      address spender,
      string calldata sourcePort,
      string calldata sourceChannel,
      string calldata denom,
      uint256 amount
  ) external returns (bool approved);
  ```

- `transfer`

  ```solidity
  /// @dev Transfer定义了执行IBC转账的方法。
  /// @param sourcePort 验证者的地址
  /// @param sourceChannel 验证者的地址
  /// @param denom 要转账给接收者的Coin的面额
  /// @param amount 要转账给接收者的Coin的数量
  /// @param sender 发送者的十六进制地址
  /// @param receiver 接收者的bech32地址
  /// @param timeoutHeight 接收者的bech32地址
  /// @param timeoutTimestamp 接收者的bech32地址
  /// @param memo 接收者的bech32地址
  function transfer(
      string memory sourcePort,
      string memory sourceChannel,
      string memory denom,
      uint256 amount,
      address sender,
      string memory receiver,
      Height memory timeoutHeight,
      uint64 timeoutTimestamp,
      string memory memo
  ) external returns (uint64 nextSequence);
  ```

## 查询

- `denomTrace`

  ```solidity
  /// @dev DenomTrace定义了返回denom跟踪的方法。
  function denomTrace(
      string memory hash
  ) external returns (DenomTrace memory denomTrace);
  ```

- `denomTraces`

  ```solidity
  /// @dev DenomTraces定义了返回所有denom跟踪的方法。
  function denomTraces(
      PageRequest memory pageRequest
  )
      external
      returns (
          DenomTrace[] memory denomTraces,
          PageResponse memory pageResponse
      );
  ```

- `denomHash`

    ```solidity
    /// @dev DenomHash定义了一种返回货币追踪信息哈希的方法。
    function denomHash(
        string memory trace
    ) external returns (string memory hash);
    ```

- `allowance`

    ```solidity
    /// @dev 返回spender在IBC转账中被owner允许转账的剩余代币数量。默认情况下，这是一个空数组。
    /// @param owner 拥有代币的账户地址。
    /// @param spender 能够转账代币的账户地址。
    /// @return allocations 对应源端口和通道的剩余可转账金额。
    function allowance(
        address owner,
        address spender
    ) external view returns (Allocation[] memory allocations);
    ```

## Events

每个交易都会触发相应的事件。它们是：

- `IBCTransfer`

    ```solidity
    /// @dev 当执行ICS-20转账时触发。
    /// @param sender 发送者的地址。
    /// @param receiver 接收者的地址。
    /// @param sourcePort IBC交易的源端口。
    /// @param sourceChannel IBC交易的源通道。
    /// @param denom 转账的代币名称。
    /// @param amount 转账的代币数量。
    /// @param memo IBC交易的备注。
    event IBCTransfer(
        address indexed sender,
        string indexed receiver,
        string sourcePort,
        string sourceChannel,
        string denom,
        uint256 amount,
        string memo
    );
    ```

- `IBCTransferAuthorization`

    ```solidity
    /// @dev 当授予ICS-20转账授权时触发。
    /// @param grantee 受让人的地址。
    /// @param granter 授权人的地址。
    /// @param sourcePort IBC交易的源端口。
    /// @param sourceChannel IBC交易的源通道。
    /// @param spendLimit 授权的代币数量。
    event IBCTransferAuthorization(
        address indexed grantee,
        address indexed granter,
        string sourcePort,
        string sourceChannel,
        Coin[] spendLimit
    );
    ```

- `RevokeIBCTransferAuthorization`

    ```solidity
    /// @dev 当所有者撤销授权者的津贴时，触发此事件。
    /// @param owner 代币的所有者。
    /// @param spender 将花费资金的地址。
    event 撤销IBCTransfer授权(
        address indexed owner,
        address indexed spender
    );
    ```

- `AllowanceChange`

    ```solidity
    /// @dev 当通过调用减少或增加津贴方法来更改津贴的授权时，触发此事件。values字段指定新的津贴，methods字段保存了设置授权的方法的信息。
    /// @param owner 代币的所有者。
    /// @param spender 将花费资金的地址。
    /// @param methods 设置授权的方法的消息类型URL。
    /// @param values 批准的代币数量。
    event 津贴更改(
        address indexed owner,
        address indexed spender,
        string[] methods,
        uint256[] values
    );
    ```


---
sidebar_position: 5
---

# IBC Transfer

## Solidity Interface & ABI

`ICS20.sol` - previously known as `IBCTransfer.sol` - is an interface through which Solidity contracts can interact with the IBC protocol on Evmos chain.
This is convenient for developers as they don’t need to know the implementation details behind the `transfer` module in [IBC-go](https://ibc.cosmos.network/).
Instead, they can perform IBC transfers using the Ethereum interface they are familiar with.
An example of a simple implementation can be found in the [evmos/extensions repo](https://github.com/evmos/extensions/tree/main/examples/simple-ibc-transfer).

### Interface `ICS20.sol`

Find the [Solidity interface in the evmos/extensions repo](https://github.com/evmos/extensions/blob/main/precompiles/stateful/ICS20.sol).

### ABI

Find the [ABI in the evmos/extensions repo](https://github.com/evmos/extensions/blob/main/precompiles/abi/ics20.json).

## Transactions

- `approve`

    ```solidity
    /// @dev Approves IBC transfer with a specific amount of tokens.
    /// @param spender spender The address which will spend the funds.
    /// @param allocations The allocations for the authorization.
    function approve(
        address spender,
        Allocation[] calldata allocations
    ) external returns (bool approved);
    ```

- `revoke`

    ```solidity
    /// @dev Revokes IBC transfer authorization for a specific spender
    /// @param spender The address which will spend the funds.
    function revoke(address spender) external returns (bool revoked);
    ```

- `increaseAllowance`
  
    ```solidity
    /// @dev Increase the allowance of a given spender by a specific amount of tokens and IBC connection for IBC transfer methods.
    /// @param spender The address which will spend the funds.
    /// @param sourcePort The source port of the IBC transaction.
    /// @param sourceChannel The source channel of the IBC transaction.
    /// @param denom The denomination of the tokens transferred.
    /// @param amount The amount of tokens to be spent.
    function increaseAllowance(
        address spender,
        string calldata sourcePort,
        string calldata sourceChannel,
        string calldata denom,
        uint256 amount
    ) external returns (bool approved);
    ```

- `decreaseAllowance`
  
    ```solidity
    /// @dev Decreases the allowance of a given spender by a specific amount of tokens for for IBC transfer methods.
    /// @param spender The address which will spend the funds.
    /// @param sourcePort The source port of the IBC transaction.
    /// @param sourceChannel The source channel of the IBC transaction.
    /// @param denom The denomination of the tokens transferred.
    /// @param amount The amount of tokens to be spent.
    function decreaseAllowance(
        address spender,
        string calldata sourcePort,
        string calldata sourceChannel,
        string calldata denom,
        uint256 amount
    ) external returns (bool approved);
    ```

- `transfer`
  
    ```solidity
    /// @dev Transfer defines a method for performing an IBC transfer.
    /// @param sourcePort the address of the validator
    /// @param sourceChannel the address of the validator
    /// @param denom the denomination of the Coin to be transferred to the receiver
    /// @param amount the amount of the Coin to be transferred to the receiver
    /// @param sender the hex address of the sender
    /// @param receiver the bech32 address of the receiver
    /// @param timeoutHeight the bech32 address of the receiver
    /// @param timeoutTimestamp the bech32 address of the receiver
    /// @param memo the bech32 address of the receiver
    function transfer(
        string memory sourcePort,
        string memory sourceChannel,
        string memory denom,
        uint256 amount,
        address sender,
        string memory receiver,
        Height memory timeoutHeight,
        uint64 timeoutTimestamp,
        string memory memo
    ) external returns (uint64 nextSequence);
    ```

## Queries

- `denomTrace`
  
    ```solidity
    /// @dev DenomTrace defines a method for returning a denom trace.
    function denomTrace(
        string memory hash
    ) external returns (DenomTrace memory denomTrace);
    ```

- `denomTraces`
  
    ```solidity
    /// @dev DenomTraces defines a method for returning all denom traces.
    function denomTraces(
        PageRequest memory pageRequest
    )
        external
        returns (
            DenomTrace[] memory denomTraces,
            PageResponse memory pageResponse
        );
    ```

- `denomHash`
  
    ```solidity
    /// @dev DenomHash defines a method for returning a hash of the denomination trace info.
    function denomHash(
        string memory trace
    ) external returns (string memory hash);
    ```

- `allowance`
  
    ```solidity
    /// @dev Returns the remaining number of tokens that spender will be allowed to spend on behalf of owner through
    /// IBC transfers. This is an empty array by default.
    /// @param owner The address of the account owning tokens.
    /// @param spender The address of the account able to transfer the tokens.
    /// @return allocations The remaining amounts allowed to spend for
    /// corresponding source port and channel.
    function allowance(
        address owner,
        address spender
    ) external view returns (Allocation[] memory allocations);
    ```

## Events

Each of the transactions emits its corresponding event. These are:

- `IBCTransfer`

    ```solidity
    /// @dev Emitted when an ICS-20 transfer is executed.
    /// @param sender The address of the sender.
    /// @param receiver The address of the receiver.
    /// @param sourcePort The source port of the IBC transaction.
    /// @param sourceChannel The source channel of the IBC transaction.
    /// @param denom The denomination of the tokens transferred.
    /// @param amount The amount of tokens transferred.
    /// @param memo The IBC transaction memo.
    event IBCTransfer(
        address indexed sender,
        string indexed receiver,
        string sourcePort,
        string sourceChannel,
        string denom,
        uint256 amount,
        string memo
    );
    ```

- `IBCTransferAuthorization`

    ```solidity
    /// @dev Emitted when an ICS-20 transfer authorization is granted.
    /// @param grantee The address of the grantee.
    /// @param granter The address of the granter.
    /// @param sourcePort The source port of the IBC transaction.
    /// @param sourceChannel The source channel of the IBC transaction.
    /// @param spendLimit The coins approved in the allocation
    event IBCTransferAuthorization(
        address indexed grantee,
        address indexed granter,
        string sourcePort,
        string sourceChannel,
        Coin[] spendLimit
    );
    ```

- `RevokeIBCTransferAuthorization`

    ```solidity
    /// @dev This event is emitted when an owner revokes a spender's allowance.
    /// @param owner The owner of the tokens.
    /// @param spender The address which will spend the funds.
    event RevokeIBCTransferAuthorization(
        address indexed owner,
        address indexed spender
    );
    ```

- `AllowanceChange`

    ```solidity
    /// @dev This event is emitted when the allowance of a spender is changed by a call to the decrease or increase
    /// allowance method. The values field specifies the new allowances and the methods field holds the
    /// information for which methods the approval was set.
    /// @param owner The owner of the tokens.
    /// @param spender The address which will spend the funds.
    /// @param methods The message type URLs of the methods for which the approval is set.
    /// @param values The amounts of tokens approved to be spent.
    event AllowanceChange(
        address indexed owner,
        address indexed spender,
        string[] methods,
        uint256[] values
    );
    ```
