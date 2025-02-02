---
title: Hooks
description: >-
  The "Lock" contract includes hooks that let developers customize their behavior.
---

# Hooks

Hooks have been added to strategic places of the Lock contract to allow lock managers to modify behaviour by calling a 3rd party contract.

We currently (version 11) support 5 hooks:

- `onKeyPurchaseHook`: called when a key purchase is triggered
- `onKeyCancelHook`: called when a key is canceled
- `onTokenUriHook`: called when the tokenURI is fetched
- `onValidKeyHook`: called when checking if a user has a valid key
- `onKeyTransferHook`: called when a key is transfered from an address to another.

## OnKeyPurchase Hook

The `onKeyPurchaseHook` allows you to create custom purchase logic, for instance dynamic pricing, etc.

It contains 2 main functions

1. `keyPurchasePrice` which is used to determine the purchase price before issuing a transaction.
2. `onKeyPurchase` which is called every time a key is sold

The `ILockKeyPurchaseHook` contract interface describes the parameters of each function (from, recipient, original price, price paid, etc), so the hook can be properly implemented.

For instance, you can find out how to implement discount codes or invite-only purchases using contract extensions we developed in the [Key Purchase Hook doc](../../Tutorials/the-key-purchase-hook/).

## OnKeyCancel Hook

Called when a key is cancelled, it can be useful to use with `onKeyPurchaseHook` to track lock ownership on a 3rd party contract.

The `ILockKeyCancelHook` interface is quite straightforward:

```solidity
interface ILockKeyCancelHook
{
  function onKeyCancel(
    address operator, // msg.sender issuing the cancel
    address to, // account which had the key canceled
    uint256 refund // amount to refund in ETH OR ERC-20
  ) external;
```

## OnTokenUri Hook

This hook is called every time the `tokenURI()` is called. This allows customization of the metadata for each token.

Want each key owner to have his/her own profile pic? Change description based on your own NFT? Just hook a contract compatible with the `ILockTokenURIHook` interface and return your own tokenURI.

```solidity
interface ILockTokenURIHook
{
  function tokenURI(
    address lockAddress, // the address of the lock
    address operator,    // the msg.sender issuing the call
    address owner,    // the owner of the key
    uint256 keyId,    // the id (tokenId) of the key (if applicable)
    uint expirationTimestamp    // the key expiration timestamp
  ) external view returns(string memory);
}
```

## OnValidKey Hook

This hook is called every time the (ERC721) `balanceOf` method is called. This allows you to override the balance of the actual contract and define if an account owns a valid key or not based using your own logic.

That way you could whitelist your own NFT holders or DAO members, and provide them access without having them to register. Just use a connector contract compatible with `ILockValidKeyHook` that checks if the account is allowed or not, and register it as a hook.

```solidity
interface ILockValidKeyHook
{
  function hasValidKey(
    address lockAddress, // the address of the current lock
    address keyOwner, // the potential owner of a key
    uint256 expirationTimestamp, // the key expiration timestamp
    bool isValidKey // the validity in the lock contrat
  )
  external view
  returns (bool);
}
```

## onKeyTransferHook Hook

Called when a key is transfered, it can be useful to use with `onKeyPurchaseHook` to track key ownership on a 3rd party contract.

The `ILockKeyTransferHook` interface is quite straightforward:

```solidity

interface ILockKeyTransferHook
{

  /**
   * @notice If the lock owner has registered an implementer then this hook
   * is called every time balanceOf is called
   * @param lockAddress the address of the current lock
   * @param tokenId the Id of the transferred key
   * @param operator who initiated the transfer
   * @param from the previous owner of transferred key
   * @param from the previous owner of transferred key
   * @param to the new owner of the key
   * @param expirationTimestamp the key expiration timestamp (after transfer)
   */
  function onKeyTransfer(
    address lockAddress,
    uint tokenId,
    address operator,
    address from,
    address to,
    uint expirationTimestamp
  ) external;
}
```

## Configuration

### Register a hook

To register a hook, call the `setEventHooks` method with the contract address(es) containing the hook logic :

```solidity
function setEventHooks(
  address _onKeyPurchaseHook,
  address _onKeyCancelHook,
  address _onValidKeyHook,
  address _onTokenURIHook,
  address _onKeyTransferHook
) external
```

Once a hook address is registered, the function at the address will be executed as an additional step to the original logic.

If you just want to set a single hook, just use the address zero for the others `0x0000000000000000000000000000000000000000`. Additionally, you can de-register any hook anytime by setting it back to 0.

Note that you could create a single contract containing multiple hooks logic, but you will still have to pass the contract address for each hook you want to register.
