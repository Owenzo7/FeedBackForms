# Summary.

Crestal Network is building a platform for anyone to create a productive AI agent with The Nation, an ecosystem where agents autonomously generate revenue, manage wallets, launch tokens, and access powerful skills to create value both on-chain and off-chain. The set of contracts in scope are the core "agent creation and update" management contracts. They are upgradeable contracts supporting gasless transactions (erc-4337), plus a few simple functions to accept on-chain ERC20-based payments. The set of contracts in scope are the core "agent creation and update" management contracts. They are upgradeable contracts supporting gasless transactions (erc-4337), plus a few simple functions to accept on-chain ERC20-based payments.

### Findings.

### [M1]("https://github.com/sherlock-audit/2025-03-crestal-network-judging/issues/205") createCommonProjectIDAndDeploymentRequest() hardcodes request id index to 0, leading to lost requests for users.

`createCommonProjectIDAndDeploymentRequest` is called by the `createAgent` function, in which `user` pays fees to create an agent.

The index is suppose to protect the user from overwriting a requestId with the same requestId but different serverURL. However, it is hardcoded to 0.


## What went wrong and how to fix it next time.

* Didn't review part of this codebase
* Didnt look through the internal functions well enough.
* Need to look at checks keenly.


## Take aways.

* If any function is using indexes or anything to do with ID's. Try and see whether you can reenter the function more than 2 times and overwrite something.
* Look at internal functions deeply because they are too important.

## Rewards -> `$1,993`.

### [M-2]("https://github.com/sherlock-audit/2025-03-crestal-network-judging/issues/225") Signatures missing some parameters being vulnerable to attackers using them coupled with malicious parameters.

`createAgentWithSigWithNFT()` for example signs projectId, base64RecParam, serverURL. However, it does not sign privateWorkerAddress or tokenId. This is an issue because although Base has a private mempool, the protocol integrates with Biconomy, which leverages ERC4337 and has a mempool for bundlers. Hence, the signatures will be available in the mempool and anyone can fetch them and submit it directly to base with other malicious tokenId or privateWorkerAddress.

Thus, users can be forced to create agents with token ids they didn't intend to use or use invalid worker addresses, DoSing them. Workers have incentive to do this as they can censor other workers this way from using other workers and only they will be able to make the deployments, censoring other workers. The protocol intends to benefit workers from their work, so they have incentive to do so.

If [createAgentWithTokenWithSig()](https://github.com/sherlock-audit/2025-03-crestal-network/blob/main/crestal-omni-contracts/src/BlueprintCore.sol#L491), the token address used can be another one that has a bigger cost and users end up paying more.

## What went wrong and how to fix it next time.

* Didn't review like all of the codebase.
* the sig was missing crucial params like `tokenId` and `privateWorkeraddr`.

## Take aways.

* Always make sure when dealing with sigs that things relating to Ownership of any identity i.e assets or tokens have been hashed i.e `tokenId` and `privateWorkerAddr`.
* Need more resilience in reviewing the whole codebase.
  

## Rewards -> `$1,993`.


### [M-3]("https://github.com/sherlock-audit/2025-03-crestal-network-judging/issues/467") Lack of access control in `setWorkerPublicKey` in Blueprint.sol which results to users to lose funds.

The lack of access control in the `setWorkerPublicKey` function will cause a significant loss of funds for users as a malicious actor will register a fake Worker public key to intercept and disrupt deployment payments.

```solidity

@> function setWorkerPublicKey(bytes calldata publicKey) public {
    if (workersPublicKey[msg.sender].length == 0) {
        workerAddressesMp[WORKER_ADDRESS_KEY].push(msg.sender);
    }
    workersPublicKey[msg.sender] = publicKey;
}
```

## What went wrong and how to fix it next time.

* No access control for the function.
* Didn't review the whole codebase.

## Take aways.

* Take a keen look at setter functions and see whether they can be exploited. (Make sure to have a full understanding of the codebase to be able to think like this).
  

## Rewards -> `$897`.

### [M-4] ("https://github.com/sherlock-audit/2025-03-crestal-network-judging/issues/46") Incomplete whitelist implementation in Blueprintcore allows bypass of agent creation restrictions.

Inconsistent whitelist validation across agent creation functions in BlueprintCore causes a complete bypass of whitelist restrictions as non-whitelisted users can create agents through alternative functions.

## What went wrong and how to fix it next time.

* Whitelist implementation wasnt implemented in functions that seemed that they needed it.
* Didnt review the wholecodebase.

## Take aways.
Make sure that whitelist implementation is in functions that require it.

## Rewards -> `$77`.