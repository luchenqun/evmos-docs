---
sidebar_position: 1
---

# 授权

用户应授予授权以允许智能合约代表用户账户发送消息。
这通过 `Authorization.sol` 实现，
该合约提供了授予权限和允许额度所需的函数。
预编译合约使用 `AuthorizationI` 接口，
允许用户批准相应的消息和金额。

## Solidity 接口

### `Authorization.sol`

在 evmos/extensions 仓库中找到 [Solidity 接口](https://github.com/evmos/extensions/blob/main/precompiles/common/Authorization.sol)。

## 交易

- `approve`

    批准一组 Cosmos 或 IBC 交易，并指定一定数量的代币

    ```solidity
    function approve(
            address spender,
            uint256 amount,
            string[] calldata methods
        ) external returns (bool approved);
    ```

- `revoke`
  
    撤销 Cosmos 交易的授权。

    ```solidity
    function revoke(
        address spender,
        string[] calldata methods
    ) external returns (bool revoked);
    ```

- `increaseAllowance`

    增加给定 spender 的允许额度，指定一定数量的代币，用于 IBC 转账方法或质押

    ```solidity
    function increaseAllowance(
            address spender,
            uint256 amount,
            string[] calldata methods
        ) external returns (bool approved);
    ```

- `decreaseAllowance`

    减少给定 spender 的允许额度，指定一定数量的代币，用于 IBC 转账方法或质押

    ```solidity
    function decreaseAllowance(
            address spender,
            uint256 amount,
            string[] calldata methods
        ) external returns (bool approved);
    ```

## 查询

- `allowance`

    返回 spender 可以代表 owner 通过 IBC 转账方法或质押花费的剩余代币数量。
    默认情况下为零

    ```solidity
    function allowance(
            address owner,
            address spender,
            string calldata method
    ) external view returns (uint256 remaining);
    ```

## 事件

- `Approval`

    当调用`approve`方法设置spender的allowance时，会触发此事件。
    `value`字段指定新的allowance，`methods`字段保存设置approval的方法的信息。

    ```solidity
    event Approval(
            address indexed owner,
            address indexed spender,
            string[] methods,
            uint256 value
        );
    ```

- `Revocation`

    当所有者撤销spender的allowance时，会触发此事件。

    ```solidity
    event Revocation(
        address indexed owner,
        address indexed spender,
        string[] methods
    );
    ```

- `AllowanceChange`

    当调用减少或增加allowance的方法改变spender的allowance时，会触发此事件。
    `values`字段指定新的allowances，`methods`字段保存设置approval的方法的信息。

    ```solidity
    event AllowanceChange(
            address indexed owner,
            address indexed spender,
            string[] methods,
            uint256[] values
        );
    ```


---
sidebar_position: 1
---

# Authorization

The user should grant authorization to allow smart contracts
to send messages on behalf of a user account.
This is achieved by the `Authorization.sol`
that provides the necessary functions to grant approvals and allowances.
The precompiled contracts use the `AuthorizationI` interface,
to allow users to approve the corresponding messages and amounts.

## Solidity Interfaces

### `Authorization.sol`

Find the [Solidity interface in the evmos/extensions repo](https://github.com/evmos/extensions/blob/main/precompiles/common/Authorization.sol).

## Transactions

- `approve`

    Approves a list of Cosmos or IBC transactions with a specific amount of tokens

    ```solidity
    function approve(
            address spender,
            uint256 amount,
            string[] calldata methods
        ) external returns (bool approved);
    ```

- `revoke`
  
    Revokes authorizations of Cosmos transactions.

    ```solidity
    function revoke(
        address spender,
        string[] calldata methods
    ) external returns (bool revoked);
    ```

- `increaseAllowance`

    Increase the allowance of a given spender by a specific amount of tokens for IBC transfer methods or staking

    ```solidity
    function increaseAllowance(
            address spender,
            uint256 amount,
            string[] calldata methods
        ) external returns (bool approved);
    ```

- `decreaseAllowance`

    Decreases the allowance of a given spender by a specific amount of tokens for IBC transfer methods or staking

    ```solidity
    function decreaseAllowance(
            address spender,
            uint256 amount,
            string[] calldata methods
        ) external returns (bool approved);
    ```

## Queries

- `allowance`

    Returns the remaining number of tokens that the spender will be allowed to
    spend on behalf of the owner through IBC transfer methods or staking.
    This is zero by default

    ```solidity
    function allowance(
            address owner,
            address spender,
            string calldata method
    ) external view returns (uint256 remaining);
    ```

## Events

- `Approval`

    This event is emitted when the allowance of a spender is set by a call to the `approve` method.
    The `value` field specifies the new allowance and the `methods`
    field holds the information for which methods the approval was set.

    ```solidity
    event Approval(
            address indexed owner,
            address indexed spender,
            string[] methods,
            uint256 value
        );
    ```

- `Revocation`

    This event is emitted when an owner revokes a spender's allowance.

    ```solidity
    event Revocation(
        address indexed owner,
        address indexed spender,
        string[] methods
    );
    ```

- `AllowanceChange`

    This event is emitted when the allowance of a spender is changed by a call to the decrease or increase allowance method.
    The `values` field specifies the new allowances and the `methods`
    field holds the information for which methods the approval was set.

    ```solidity
    event AllowanceChange(
            address indexed owner,
            address indexed spender,
            string[] methods,
            uint256[] values
        );
    ```
