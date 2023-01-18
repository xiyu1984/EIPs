---
eip: <to be assigned>
title: Omniverse Distributed Ledger Technology(O-DLT for short)
description: The O-DLT is a new application-level token features built over multiple existing L1 public chains, enabling asset-related operations such as transfers and receptions running over different consensus spaces synchronously and equivalently.
author: Shawn Zheng(@xiyu1984), Jason Cheng(chengjingxx@gmail.com), George Huang(@virgil2019), Kay Lin(@kay404)
discussions-to: <URL>
status: Draft
type: Standards Track
category: ERC
created: 2023-01-17
requires (*optional): <EIP number(s)>
---

## Abstract

The **Omniverse DLT**(O-DLT for short) is a new application-level token features built **over** multiple existing L1 public chains, enabling asset-related operations such as transfers and receptions running over different consensus spaces **synchronously** and **equivalently**.  
The core meaning of Omniverse is that the ***legitimacy of all on-chain states and operations can be equivalently verified and consistently recorded over different consensus spaces, regardless of where they were initiated.***  
O-DLT works at an application level, which means everything related is processed in smart contracts or similar mechanisms, just as the ERC20/ERC721 did.  

## Motivation

For projects serving multiple chains, it might be useful that the token is able to be accessed anywhere. Although assets-bridges can more or less make it, we don't think it is enough. And in the process of R&D, we found that the fragmentation of tokens is a common and disturbing problem among chains and L2s.  
- We want our token to be treated as a whole instead of being divided into fragmented parts on different public chains. O-DLT can get it.
- When one chain breaks down, we don't want to lose our assets along with it. Assets-bridge paradigm cannot provide a guarantee for this. O-DLT can provide this guarantee even if there's only one chain that works, for example, we can rely on the stability and robustness of Ethereum.  
- Not just for a certain token, we think the Omniverse Token might be useful for other projects on Ethereum and other chains. O-DLT is actually a new kind of open-source asset paradigm at the application level. 


## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119 and RFC 8174.

### Omniverse Account
The Omniverse account is expressed as a public key created by the elliptic curve `secp256k1`, which has already been supported by Ethereum tech stacks. For those who don’t support secp256k1 or have a different address system, a mapping mechanism is needed.  

### Data Structure
The definations of omniverse transaction data is as follows:  
```solidity
/**
 * @dev Omniverse transaction data structure
 * @Member nonce: The serial number of an o-transactions sent from an Omniverse Account. If the current nonce of an o-account is k, the valid nonce in the next o-transaction is k+1. 
 * @Member chainId: The chain where the o-transaction is initiated
 * @Member initiateSC: The contract address from which the o-transaction is first initiated
 * @Member from: The Omniverse account which signs the o-transaction
 * @Member payload: The encoded bussiness logic data, which is maintained by the developer
 * @Member signature: The signature of the above informations. 
 */
struct OmniverseTransactionData {
    uint128 nonce;
    uint32 chainId;
    bytes initiateSC;
    bytes from;
    bytes payload;
    bytes signature;
}
```
- The data structure `OmniverseTransactionData` MUST be defined as above.
- The member `nonce` MUST be defined as `uint128` due to better compatibility for more tech stacks of blockchains.
- The member `chainId` MUST be defined as `uint32`.
- The member `initiateSC` MUST be defined as `bytes`.
- The member `from` MUST be defined as `bytes`.
- The member `payload` MUST be defined as `bytes`. It is encoded from a user-defined data related to the o-transaction. For example:  
    - For fungible tokens it is RECOMMENDED as follows:  
        ```solidity
        /**
        * @dev Fungible token data structure, which will be encoded to or decoded from
        * the field `payload` of `OmniverseTransactionData`
        *
        * op: The operation type
        * NOTE op: 0-31 are reserved values, 32-255 are custom values
        *             op: 0 Transfers omniverse token `amount` from user `from` to user `exData`, `from` MUST have at least `amount` token
        *             op: 1 User `from` mints token `amount` to user `exData`
        *             op: 2 User `from` burns token `amount` from user `exData`
        * exData: The operation data. This sector could be empty and is determined by `op`
        * amount: The amount of token which is operated
        */
        struct Fungible {
            uint8 op;
            bytes exData;
            uint256 amount;
        }
        ```
        - The related raw data for `signature` in o-transaction is the concatenation of the raw bytes of `op`, `exData`, and `amount`.  
    - For non-fungible tokens it is RECOMMENDED as follows:  
        ```solidity
        /**
        * @dev Non-Fungible token data structure, which will be encoded to or decoded from
        * the field `payload` of `OmniverseTransactionData`
        *
        * op: The operation type
        * NOTE op: 0-31 are reserved values, 32-255 are custom values
        *             op: 0 Transfers omniverse token `tokenId` from user `from` to user `exData`, `from` MUST have the token with `tokenId`
        *             op: 1 User `from` mints token with `tokenId` to user `exData`
        *             op: 2 User `from` burns token with `tokenId` from user `exData`
        * exData: The operation data. This sector could be empty and is determined by `op`
        * tokenId: The tokenId of the non-fungible token which is operated
        */
        struct NonFungible {
            uint8 op;
            bytes exData;
            uint256 tokenId;
        }
        ```
        - The related raw data for `signature` in o-transaction is the concatenation of the raw bytes of `op`, `exData`, and `tokenId`. 
- The member `signature` MUST be defined as `bytes`. It is RECOMMENDED to be created as follows, which is determined by certain omniverse token developers according to their situations:  
    - Concat the sectors in `OmniverseTransactionData` as below (take Fungible token for example) and calculate the hash with `keccak256`: 
        ```solidity
        function getTransactionHash(OmniverseTransactionData memory _data) public pure returns (bytes32) {
            Fungible memory fungible = decodeData(_data.payload);
            bytes memory payload = abi.encodePacked(fungible.op, fungible.exData, uint128(fungible.amount));
            bytes memory rawData = abi.encodePacked(_data.nonce, _data.chainId, _data.initiateSC, _data.from, payload);
            return keccak256(rawData);
        }
        ```
    - The signature is about the hash value.

### Smart Contract Interface
- Omniverse Protocol  
    ```solidity
    import "../OmniverseTransactionData.sol";

    /**
    * @dev Interface of the omniverse DLT
    */
    interface IOmniverseTransaction {
        /**
        * @dev Emitted when a transaction which has nonce `nonce` and was signed by user `pk` is executed
        */
        event TransactionSent(bytes pk, uint256 nonce);

        /**
        * @dev Sends an omniverse transaction with omniverse transaction data `_data`
        * NOTE The transaction MUST be deferred executed, and the developer should implement a trigger mechanism
        * @param _data Omniverse transaction data
        * See more information in OmniverseTransactionData.sol
        *
        * Emit a {TransactionSent} event
        */
        function sendOmniverseTransaction(OmniverseTransactionData calldata _data) external;

        /**
        * @dev Returns the count of transactions sent by user `_pk`
        * @param _pk Omniverse account to be queried
        */
        function getTransactionCount(bytes memory _pk) external view returns (uint256);

        /**
        * @dev Returns the transaction data `txData` and timestamp `timestamp` of the user `_use` at a specified nonce `_nonce`
        * @param _user Omniverse account to be queried
        * @param _nonce The nonce to be queried
        */
        function getTransactionData(bytes calldata _user, uint256 _nonce) external view returns (OmniverseTransactionData memory, uint256);

        /**
        * @dev Returns the chain ID
        */
        function getChainId() external view returns (uint32);
    }
    ```
- Omniverse Fungible  
    ```solidity
    import "./IOmniverseTransaction.sol";

    /**
    * @dev Interface of the omniverse fungible token, which inherits {IOmniverseTransaction}
    */
    interface IOmniverseFungible is IOmniverseTransaction {
        /**
        * @dev Returns the omniverse balance of a user `_pk`
        * @param _pk Omniverse account to be queried
        */
        function omniverseBalanceOf(bytes calldata _pk) external view returns (uint256);
    }
    ```
- Omniverse Non-Fungible
    ```solidity
    import "./IOmniverseTransaction.sol";

    /**
    * @dev Interface of the omniverse non fungible token, which inherits {IOmniverseTransaction}
    */
    interface IOmniverseNonFungible is IOmniverseTransaction {
        /**
        * @dev Returns the omniverse balance of a user `_pk`
        * @param _pk Omniverse account to be queried
        */
        function omniverseBalanceOf(bytes calldata _pk) external view returns (uint256);

        /**
        * @dev Returns the owner of a token `tokenId`
        * @param _tokenId Omniverse token id to be queried
        */
        function omniverseOwnerOf(uint256 _tokenId) external view returns (bytes memory);
    }
    ```

## Rationale
### Architecture
![image](https://user-images.githubusercontent.com/83746881/213079540-2159e0f1-d74c-495f-87b1-fa3334193069.png)
  
- The implementation of the Omniverse Account is not very hard, and we temporarily choose a common elliptic curve secp256k1 to make it out, which has already been supported by Ethereum tech stacks. For those who don’t support secp256k1 or have a different address system, we can adapt them with a simple mapping mechanism ([Flow for example](https://github.com/Omniverse-Web3-Labs/omniverse-flow)).  
- The Omniverse Transaction guarantees the ultimate consistency of omniverse transactions(o-transaction for short) across all chains. The related data structure is `OmniverseTransactionData` mentioned [above](#data-structure).
    - The `nonce` is very important, which is the key point to synchronize the states globally.
    - The `nonce` appears in two places, the one is `nonce in o-transaction` data as above, and the other is `account nonce` maintained by on-chain O-DLT smart contracts. The example codes about the `account nonce` can be found [here](https://github.com/Omniverse-Web3-Labs/omniverse-evm/blob/main/contracts/contracts/SkywalkerFungible.sol#L50) 
    - The `nonce in o-transaction` data will be verified according to the `account nonce` managed by on-chain O-DLT smart contracts. Some example codes can be found [here](https://github.com/Omniverse-Web3-Labs/omniverse-evm/blob/main/contracts/contracts/libraries/OmniverseProtocol.sol#L64).
- The Omniverse Token could be implemented with the [interfaces mentioned above](#smart-contract). It can also be used with the combination of ERC20/ERC721. The prototype of the code can be found [here](https://github.com/Omniverse-Web3-Labs/omniverse-evm/blob/main/contracts/contracts/interfaces/IOmniverseFungible.sol)  
    - The first thing is verifying the signature of the o-transaction data. 
    - Then the operation will be added to a pre-execution cache, and wait for a fixed time until is executed. The waiting time will be able to be settled by the deployer, for example, 5 minutes. 
    - The off-chain synchronizer will deliver the o-transaction data to other chains. If another o-transaction data with the same nonce and the same sender account is received within the waiting time, and if there's any content in `OmniverseTransactionData` difference, a malicious attack happens and the related sender account will be punished. 
    - The example code of `sendOmniverseTransaction` is [here](https://github.com/Omniverse-Web3-Labs/omniverse-evm/blob/main/contracts/contracts/SkywalkerFungible.sol#L103)
    - and the example code of executing is [here](https://github.com/Omniverse-Web3-Labs/omniverse-evm/blob/main/contracts/contracts/SkywalkerFungible.sol#L110). 
    - The implementation for Omniverse Non-Fungible Token is almost the same and the defination of the interface can be found [here](https://github.com/Omniverse-Web3-Labs/omniverse-evm/blob/main/contracts/contracts/interfaces/IOmniverseNonFungible.sol) 
- The Omniverse Verification is mainly about the verification of the signature implemented in different tech stacks according to the blockchain. As the signature is unfakeable and non-deniable, malicious attacks could be found deterministicly.
- The bottom is the off-chain synchronizer. The synchronizer is a very simple off-chain procedure, and it just listens to the Omniverse events happening on-chain and delivers the latest o-transaction events. As everything in the Omniverse paradigm is along with a signature and is verified cryptographically, there's no need to worry about synchronizers doing malicious things, and I will explain it later. The off-chain part of O-DLT is indeed trust-free. Everyone can launch a synchronizer to get rewards by helping synchronize information.  

### Features
The O-DLT has the following features:
- The omniverse token(o-token for short) based on O-DLT  is not fragmented into separated parts by the boundary of blockchains but as a whole. If someone has one o-token on Ethereum, he will have an equivalent one on other chains at the same time.
- The state of the tokens based on O-DLT is synchronous on different chains. If someone sends/receives one token on Ethereum, he will send/receive one token on other chains at the same time.

### Workflow
![image](https://user-images.githubusercontent.com/83746881/212859794-13a0ba68-f89f-45cf-8fb4-e5fd09970166.png)

- Suppose a common user `A` and the related operation `account nonce` is $k$.
- `A` initiate an omniverse transfer operation on Ethereum by calling `omniverse_transfer`. The current `account nonce` of `A` in the O-DLT smart contracts deployed on Ethereum is $k$ so the valid value of `nonce in o-transaction` needs to be $k+1$.  
- The O-DLT smart contracts on Ethereum verify the signature of the o-transaction data at an **application level**. If the verification for the signature and data succeeds, the o-transaction data will be published on the O-DLT smart contracts of the Ethereum side. The verification for the data includes:
    - whether the amount is valid
    - and whether the `nonce in o-transaction` is 1 larger than the `account nonce` maintained by the on-chain O-DLT
- Now, `A`'s newest submitted `nonce in o-transaction` on Ethereum is $k+1$, but still $k$ on other chains.
- The off-chain synchronizers find the newly published o-transaction, and they will find the `nonce in o-transaction` is larger than the related `account nonce` on other chains.
- These synchronizers will rush to deliver this message because whoever submits to the destination chain first will get a reward. There's no will for independent synchronizers to do evil because they just deliver `A`'s o-transaction data. (The reward is coming from the service fee or a mining mechanism according to the average number of o-transactions within a fixed time. The strategy of the reward may not be just for the first one but for the first three with a gradual decrease.) 
- Finally, the O-DLT smart contracts deployed on other chains will all receive the o-transaction data, verify the signature and execute it when the **waiting time is up**. After execution, the underlying `account nonce` will add 1. Now all the `account nonce` of account `A` will be $k+1$, and the state of the balances of the related account will be the same too.  

We have provided an intuitive but non-rigorous [proof for the **ultimate consistency**](https://github.com/Omniverse-Web3-Labs/o-amm/blob/main/docs/Proof-of-ultimate-consistency.md) for a better understanding of the **synchronization** mechanisms.

## Reference Implementation
- [Omniverse Fungible Token](https://github.com/Omniverse-Web3-Labs/omniverse-evm/tree/main/contracts/contracts)
    - Common Tools
        ```solidity
        // SPDX-License-Identifier: MIT
        pragma solidity >=0.8.0 <0.9.0;

        import "../OmniverseTransactionData.sol";

        /**
        * @dev Fungible token data structure, which will be encoded from or decoded from
        * the field `payload` of `OmniverseTransactionData`
        *
        * op: The operation type
        * NOTE op: 0-31 are reserved values, 32-255 are custom values
        *             op: 0 Transfers omniverse token `amount` from user `from` to user `exData`, `from` MUST have at least `amount` token
        *             op: 1 User `from` mints token `amount` to user `exData`
        *             op: 2 User `from` burns token `amount` from user `exData`
        * exData: The operation data. This sector could be empty and is determined by `op`
        * amount: The amount of token which is operated
        */
        struct Fungible {
            uint8 op;
            bytes exData;
            uint256 amount;
        }

        /**
        * @dev Used to record one omniverse transaction data
        * txData: The original omniverse transaction data committed to the contract
        * timestamp: When the omniverse transaction data is committed
        */
        struct OmniverseTx {
            OmniverseTransactionData txData;
            uint256 timestamp;
        }

        /**
        * @dev An malicious omniverse transaction data
        * oData: The recorded omniverse transaction data
        * hisNonce: The nonce of the historical transaction which it conflicts with
        */
        struct EvilTxData {
            OmniverseTx oData;
            uint256 hisNonce;
        }

        /**
        * @dev Used to record the historical omniverse transactions of a user
        * txList: Successful historical omniverse transaction list
        * evilTxList: Malicious historical omniverse transaction list
        */
        struct RecordedCertificate {
            OmniverseTx[] txList;
            EvilTxData[] evilTxList;
        }

        // Result of verification of an omniverse transaction
        enum VerifyResult {
            Success,
            Malicious
        }

        /**
        * @dev The library is mainly responsible for omniverse transaction verification and
        * provides some basic methods.
        * NOTE The verification method is for reference only, and developers can design appropriate
        * verification mechanism based on their bussiness logic.
        */
        library SkywalkerFungibleHelper {
            /**
            * @dev Encode `_fungible` into bytes
            */
            function encodeData(Fungible memory _fungible) internal pure returns (bytes memory) {
                return abi.encode(_fungible.op, _fungible.exData, _fungible.amount);
            }

            /**
            * @dev Decode `_data` from bytes to Fungible
            */
            function decodeData(bytes memory _data) internal pure returns (Fungible memory) {
                (uint8 op, bytes memory exData, uint256 amount) = abi.decode(_data, (uint8, bytes, uint256));
                return Fungible(op, exData, amount);
            }
            
            /**
            * @dev Get the hash of a transaction
            */
            function getTransactionHash(OmniverseTransactionData memory _data) public pure returns (bytes32) {
                Fungible memory fungible = decodeData(_data.payload);
                bytes memory payload = abi.encodePacked(fungible.op, fungible.exData, uint128(fungible.amount));
                bytes memory rawData = abi.encodePacked(_data.nonce, _data.chainId, _data.initiateSC, _data.from, payload);
                return keccak256(rawData);
            }

            /**
            * @dev Recover the address
            */
            function recoverAddress(bytes32 _hash, bytes memory _signature) public pure returns (address) {
                uint8 v;
                bytes32 r;
                bytes32 s;
                assembly {
                    r := mload(add(_signature, 32))
                    s := mload(add(_signature, 64))
                    v := mload(add(_signature, 65))
                }
                address recovered = ecrecover(_hash, v, r, s);
                require(recovered != address(0), "Verify failed");
                return recovered;
            }

            /**
            * @dev Check if the public key matches the recovered address
            */
            function checkPkMatched(bytes memory _pk, address _address) public pure {
                bytes32 hash = keccak256(_pk);
                address pkAddress = address(uint160(uint256(hash)));
                require(_address == pkAddress, "Signer not sender");
            }

            /**
            * @dev Verify an omniverse transaction
            */
            function verifyTransaction(RecordedCertificate storage rc, OmniverseTransactionData memory _data) public returns (VerifyResult) {
                uint256 nonce = rc.txList.length;
                
                bytes32 txHash = getTransactionHash(_data);
                address recoveredAddress = recoverAddress(txHash, _data.signature);
                // Signature verified failed
                checkPkMatched(_data.from, recoveredAddress);

                // Check nonce
                if (nonce == _data.nonce) {
                    return VerifyResult.Success;
                }
                else if (nonce > _data.nonce) {
                    // The message has been received, check conflicts
                    OmniverseTx storage hisTx = rc.txList[_data.nonce];
                    bytes32 hisTxHash = getTransactionHash(hisTx.txData);
                    if (hisTxHash != txHash) {
                        // to be continued, add to evil list, but can not be duplicated
                        EvilTxData storage evilTx = rc.evilTxList.push();
                        evilTx.hisNonce = nonce;
                        evilTx.oData.txData = _data;
                        evilTx.oData.timestamp = block.timestamp;
                        return VerifyResult.Malicious;
                    }
                    else {
                        revert("Duplicated");
                    }
                }
                else {
                    revert("Nonce error");
                }
            }
        }
        ```
    - Fungible token smart contract
        ```solidity
        // SPDX-License-Identifier: MIT
        pragma solidity >=0.8.0 <0.9.0;

        import "@openzeppelin/contracts/access/Ownable.sol";
        import "./ERC20.sol";
        import "./libraries/SkywalkerFungibleHelper.sol";
        import "./interfaces/IOmniverseFungible.sol";

        /**
        * @dev Implementation of the {IOmniverseFungible} interface
        */
        contract SkywalkerFungible is ERC20, Ownable, IOmniverseFungible {
            uint8 constant TRANSFER = 0;
            uint8 constant MINT = 1;
            uint8 constant BURN = 2;
            uint8 constant DEPOSIT = 3;
            uint8 constant WITHDRAW = 4;

            /**
            * @dev Deposit request information
            * receiver: The target of deposit
            * amount: The amount of deposit
            */
            struct DepositRequest {
                bytes receiver;
                uint256 amount;
            }

            /** @dev Used to index a delayed transaction
            * sender: The account which sent the transaction
            * nonce: The nonce of the delayed transaction
            */
            struct DelayedTx {
                bytes sender;
                uint256 nonce;
            }

            /**
            * @dev The member information
            * chainId: The chain which the member belongs to
            * contractAddr: The contract address on the member chain
            */
            struct Member {
                uint32 chainId;
                bytes contractAddr;
            }

            // Chain id used to distinguish different chains
            uint32 chainId;
            // O-transaction cooling down time
            uint256 public cdTime;
            // Omniverse accounts record
            mapping(bytes => RecordedCertificate) transactionRecorder;
            // Transactions to be executed
            mapping(bytes => OmniverseTx) public transactionCache;

            // All information of chains on which the token is deployed
            Member[] members;
            // Omniverse balances
            mapping(bytes => uint256) omniverseBalances;
            // Delay-executing transactions
            DelayedTx[] delayedTxs;
            // MPC address who has the permission to deposit
            bytes public committee;
            // Deposit request list to be reviewed
            DepositRequest[] depositRequests;
            // Current dealing deposit request index
            uint256 public depositDealingIndex;
            // Account map from evm address to public key
            mapping(address => bytes) accountsMap;

            event OmniverseTokenTransfer(bytes from, bytes to, uint256 value);
            event OmniverseTokenWithdraw(bytes from, uint256 value);
            event OmniverseTokenDeposit(bytes to, uint256 value);

            /**
            * @dev Throws if called by any account other than the committe
            */
            modifier onlyCommittee() {
                address committeeAddr = _pkToAddress(committee);
                require(msg.sender == committeeAddr, "Not committee");
                _;
            }

            /**
            * @dev Initiates the contract
            * @param _chainId The chain which the contract is deployed on
            * @param _name The name of the token
            * @param _symbol The symbol of the token
            */
            constructor(uint8 _chainId, string memory _name, string memory _symbol) ERC20(_name, _symbol) {
                chainId = _chainId;
            }

            /**
            * @dev Set the address of committee
            */
            function setCommitteeAddress(bytes calldata _address) public onlyOwner {
                committee = _address;
            }

            /**
            * @dev See {IOmniverseFungible-sendOmniverseTransaction}
            * Send an omniverse transaction
            */
            function sendOmniverseTransaction(OmniverseTransactionData calldata _data) external override {
                _omniverseTransaction(_data);
            }

            /**
            * @dev See {IOmniverseFungible-triggerExecution}
            */
            function triggerExecution() external {
                require(delayedTxs.length > 0, "No delayed tx");

                OmniverseTx storage cache = transactionCache[delayedTxs[0].sender];
                require(cache.timestamp != 0, "Not cached");
                require(cache.txData.nonce == delayedTxs[0].nonce, "Nonce error");
                (OmniverseTransactionData storage txData, uint256 timestamp) = (cache.txData, cache.timestamp);
                require(block.timestamp >= timestamp + cdTime, "Not executable");
                delayedTxs[0] = delayedTxs[delayedTxs.length - 1];
                delayedTxs.pop();
                cache.timestamp = 0;
                // Add to transaction recorder
                RecordedCertificate storage rc = transactionRecorder[txData.from];
                rc.txList.push(cache);

                Fungible memory fungible = SkywalkerFungibleHelper.decodeData(txData.payload);
                if (fungible.op == WITHDRAW) {
                    _omniverseWithdraw(txData.from, fungible.amount, txData.chainId == chainId);
                }
                else if (fungible.op == TRANSFER) {
                    _omniverseTransfer(txData.from, fungible.exData, fungible.amount);
                }
                else if (fungible.op == DEPOSIT) {
                    _omniverseDeposit(txData.from, fungible.exData, fungible.amount);
                }
                else if (fungible.op == MINT) {
                    _checkOwner(txData.from);
                    _omniverseMint(fungible.exData, fungible.amount);
                }
            }
            
            /**
            * @dev Check if the transaction can be executed successfully
            */
            function _checkExecution(OmniverseTransactionData memory txData) internal view {
                Fungible memory fungible = SkywalkerFungibleHelper.decodeData(txData.payload);
                if (fungible.op == WITHDRAW) {
                    _checkOmniverseWithdraw(txData.from, fungible.amount);
                }
                else if (fungible.op == TRANSFER) {
                    _checkOmniverseTransfer(txData.from, fungible.amount);
                }
                else if (fungible.op == DEPOSIT) {
                }
                else if (fungible.op == MINT) {
                    _checkOwner(txData.from);
                }
                else {
                    revert("OP code error");
                }
            }

            /**
            * @dev Returns the nearest exexutable delayed transaction info
            * or returns default if not found
            */
            function getExecutableDelayedTx() external view returns (DelayedTx memory ret) {
                if (delayedTxs.length > 0) {
                    OmniverseTx storage cache = transactionCache[delayedTxs[0].sender];
                    if (block.timestamp >= cache.timestamp + cdTime) {
                        ret = delayedTxs[0];
                    }
                }
            }

            /**
            * @dev Returns the count of delayed transactions
            */
            function getDelayedTxCount() external view returns (uint256) {
                return delayedTxs.length;
            }

            /**
            * @dev See {IOmniverseFungible-omniverseBalanceOf}
            * Returns the omniverse balance of a user
            */
            function omniverseBalanceOf(bytes calldata _pk) external view override returns (uint256) {
                return omniverseBalances[_pk];
            }

            /**
            * @dev See {IERC20-balanceOf}.
            */
            function balanceOf(address account) public view virtual override returns (uint256) {
                bytes storage pk = accountsMap[account];
                if (pk.length == 0) {
                    return 0;
                }
                else {
                    return omniverseBalances[pk];
                }
            }

            /**
            * @dev See {IERC20-balanceOf}.
            */
            function nativeBalanceOf(address account) public view returns (uint256) {
                return _balances[account];
            }

            /**
            * @dev Receive and check an omniverse transaction
            */
            function _omniverseTransaction(OmniverseTransactionData memory _data) internal {
                // Check if the tx initiateSC is correct
                bool found = false;
                for (uint256 i = 0; i < members.length; i++) {
                    if (members[i].chainId == _data.chainId) {
                        require(keccak256(members[i].contractAddr) == keccak256(_data.initiateSC), "Wrong initiateSC");
                        found = true;
                    }
                }
                require(found, "Wrong initiateSC");

                // Check if the sender is honest
                // to be continued, we can use block list instead of `isMalicious`
                require(!isMalicious(_data.from), "User malicious");

                // Verify the signature
                VerifyResult verifyRet = SkywalkerFungibleHelper.verifyTransaction(transactionRecorder[_data.from], _data);

                if (verifyRet == VerifyResult.Success) {
                    // Check cache
                    OmniverseTx storage cache = transactionCache[_data.from];
                    require(cache.timestamp == 0, "Transaction cached");
                    // Logic verification
                    _checkExecution(_data);
                    // Delays in executing
                    cache.txData = _data;
                    cache.timestamp = block.timestamp;
                    delayedTxs.push(DelayedTx(_data.from, _data.nonce));
                    if (_data.chainId == chainId) {
                        emit TransactionSent(_data.from, _data.nonce);
                    }
                }
                else if (verifyRet == VerifyResult.Malicious) {
                    // Slash
                }
            }

            /**
            * @dev Check if an omniverse transfer operation can be executed successfully
            */
            function _checkOmniverseTransfer(bytes memory _from, uint256 _amount) internal view {
                uint256 fromBalance = omniverseBalances[_from];
                require(fromBalance >= _amount, "Exceed Balance");
            }

            /**
            * @dev Exucute an omniverse transfer operation
            */
            function _omniverseTransfer(bytes memory _from, bytes memory _to, uint256 _amount) internal {
                _checkOmniverseTransfer(_from, _amount);
                
                uint256 fromBalance = omniverseBalances[_from];
                
                unchecked {
                    omniverseBalances[_from] = fromBalance - _amount;
                }
                omniverseBalances[_to] += _amount;

                emit OmniverseTokenTransfer(_from, _to, _amount);

                address toAddr = _pkToAddress(_to);
                accountsMap[toAddr] = _to;
            }

            /**
            * @dev Check if an omniverse withdraw operation can be executed successfully
            */
            function _checkOmniverseWithdraw(bytes memory _from, uint256 _amount) internal view {
                uint256 fromBalance = omniverseBalances[_from];
                require(fromBalance >= _amount, "Exceed Balance");
            }

            /**
            * @dev Execute an omniverse withdraw operation
            */
            function _omniverseWithdraw(bytes memory _from, uint256 _amount, bool _thisChain) internal {
                _checkOmniverseWithdraw(_from, _amount);

                uint256 fromBalance = omniverseBalances[_from];
                
                unchecked {
                    omniverseBalances[_from] = fromBalance - _amount;
                }
                
                if (_thisChain) {
                    address ownerAddr = _pkToAddress(_from);

                    // mint
                    _totalSupply += _amount;
                    _balances[ownerAddr] += _amount;
                }

                emit OmniverseTokenWithdraw(_from, _amount);
            }

            /**
            * @dev Execute an omniverse deposit operation
            */
            function _omniverseDeposit(bytes memory _from, bytes memory _to, uint256 _amount) internal {
                require(keccak256(_from) == keccak256(committee), "Not committee");

                unchecked {
                    omniverseBalances[_to] += _amount;
                }

                emit OmniverseTokenDeposit(_to, _amount);
            }
            
            /**
            * @dev Check if the public key is the owner
            */
            function _checkOwner(bytes memory _pk) internal view {
                address fromAddr = _pkToAddress(_pk);
                require(fromAddr == owner(), "Not owner");
            }

            /**
            * @dev Execute an omniverse mint operation
            */
            function _omniverseMint(bytes memory _to, uint256 _amount) internal {
                omniverseBalances[_to] += _amount;
                emit OmniverseTokenTransfer("", _to, _amount);

                address toAddr = _pkToAddress(_to);
                accountsMap[toAddr] = _to;
            }

            /**
            * @dev Convert the public key to evm address
            */
            function _pkToAddress(bytes memory _pk) internal pure returns (address) {
                bytes32 hash = keccak256(_pk);
                return address(uint160(uint256(hash)));
            }

            /**
            * @dev Add new chain members to the token
            */
            function setMembers(Member[] calldata _members) external onlyOwner {
                for (uint256 i = 0; i < _members.length; i++) {
                    if (i < members.length) {
                        members[i] = _members[i];
                    }
                    else {
                        members.push(_members[i]);
                    }
                }

                for (uint256 i = _members.length; i < members.length; i++) {
                    delete members[i];
                }
            }

            /**
            * @dev Returns chain members of the token
            */
            function getMembers() external view returns (Member[] memory) {
                return members;
            }

            /**
            * @dev Users request to convert native token to omniverse token
            */
            function requestDeposit(bytes calldata from, uint256 amount) external {
                address fromAddr = _pkToAddress(from);
                require(fromAddr == msg.sender, "Signer not sender");

                uint256 fromBalance = _balances[fromAddr];
                require(fromBalance >= amount, "Exceed balance");

                // Update
                unchecked {
                    _balances[fromAddr] = fromBalance - amount;
                }
                _totalSupply -= amount;

                depositRequests.push(DepositRequest(from, amount));
            }

            /**
            * @dev The committee approves a user's request
            */
            function approveDeposit(uint256 index, uint128 nonce, bytes calldata signature) external onlyCommittee {
                require(index == depositDealingIndex, "Index error");

                DepositRequest storage request = depositRequests[index];
                depositDealingIndex++;

                OmniverseTransactionData memory p;
                p.nonce = nonce;
                p.chainId = chainId;
                p.from = committee;
                p.initiateSC = abi.encodePacked(address(this));
                p.signature = signature;
                p.payload = SkywalkerFungibleHelper.encodeData(Fungible(DEPOSIT, request.receiver, request.amount));
                _omniverseTransaction(p);
            }

            /**
            @dev Returns the deposit request at `index`
            @param index: The index of requests
            */
            function getDepositRequest(uint256 index) external view returns (DepositRequest memory ret) {
                if (depositRequests.length > index) {
                    ret = depositRequests[index];
                }
            }
            
            /**
            @dev See {IERC20-decimals}.
            */
            function decimals() public view virtual override returns (uint8) {
                return 12;
            }

            /**
            * @dev See IOmniverseFungible
            */
            function getTransactionCount(bytes memory _pk) external override view returns (uint256) {
                return transactionRecorder[_pk].txList.length;
            }

            /**
            * @dev See IOmniverseFungible
            */
            function getTransactionData(bytes calldata _user, uint256 _nonce) external override view returns (OmniverseTransactionData memory txData, uint256 timestamp) {
                RecordedCertificate storage rc = transactionRecorder[_user];
                OmniverseTx storage omniTx = rc.txList[_nonce];
                txData = omniTx.txData;
                timestamp = omniTx.timestamp;
            }

            /**
            * @dev Set the cooling down time of an omniverse transaction
            */
            function setCooingDownTime(uint256 _time) external {
                cdTime = _time;
            }

            /**
            * @dev Index the user is malicious or not
            */
            function isMalicious(bytes memory _pk) public view returns (bool) {
                RecordedCertificate storage rc = transactionRecorder[_pk];
                return (rc.evilTxList.length > 0);
            }

            /**
            * @dev See IOmniverseFungible
            */
            function getChainId() external view returns (uint32) {
                return chainId;
            }
        }
        ```
- The implememtation of Omniverse Non-Fungible Token is similiar with the Omniverse Fungible Token.  

## Security Considerations
### Attack Vector Analysis
According to the above, there are two roles:
**common users** who initiate a o-transaction (at the application level)
and **synchronizers** who just carry the o-transaction data if they find differences between different chains.  

The two roles might be where the attack happens:  
#### **Will the *synchronizers* cheat?**  
- Simply speaking, it's none of the **synchronizer**'s business as **they cannot create other users' signatures** unless some **common users** tell him, but at this point, we think it's a problem with the role **common user**.  
- The **synchronizer** has no will and cannot do evil because the transaction data that they deliver is verified by the related **signature** of others(a **common user**).  
- The **synchronizers** will be rewarded as long as they submit valid o-transaction data, and *valid* only means that the signature and the amount are both valid. This will be detailed and explained later when analyzing the role of **common user**.  
- The **synchronizers** will do the delivery once they find differences between different chains:
    - If the current `account nonce` on one chain is smaller than a published `nonce in o-transaction` on another chain
    - If the transaction data related to a specific `nonce in o-transaction` on one chain is different from another published o-transaction data with the same `nonce in o-transaction` on another chain

- **Conclusion: The *synchronizers* won't cheat because there are no benefits and no way for them to do so.**

#### **Will the *common user* cheat?**
- Simply speaking, **maybe they will**, but fortunately, **they can't succeed**.  
- Suppose the current `account nonce` of a **common user** `A` is $k$ on all chains.  
- Common user `A` initiates an o-transaction on a Parachain of Polkadot first, in which `A` transfers `10` o-tokens to an o-account of a **common user** `B`. The `nonce in o-transaction` needs to be $k+1$. After signature and data verification, the o-transaction data(`ot-P-ab` for short) will be published on Polkadot.
- At the same time, `A` initiates an o-transaction with the same nonce $k+1$ but different data(transfer `10` o-tokens to another o-account `C`) on Ethereum. This o-transaction(named `ot-E-ac`) will pass the verification on Ethereum first, and be published.  
- At this point, it seems `A` finished a ***double spend attack*** and the O-DLT states on Polkadot and Ethereum are different.  
- **Response strategy**:
    - As we mentioned above, the synchronizers will deliver `ot-P-ab` to the O-DLT on Ethereum and deliver `ot-E-ac` to the O-DLT on Polkadot because they are different although with the same nonce. The synchronizer who submits the o-transaction first will be rewarded as the signature is valid.
    - Both the O-DLTs on Polkadot and Ethereum will find that `A` did cheating after they received `ot-E-ac` and `ot-P-ab` respectively as the signature of `A` is non-deniable.  
    - We mentioned above that the execution of an o-transaction will not be done immediately and instead there needs to be a fixed waiting time. So the `double spend attack` caused by `A` won't succeed.
    - There will be many synchronizers waiting for delivering o-transactions to get rewards. So although it's almost impossible that a **common user** can submit two o-transactions to two chains, none of the synchronizers deliver the o-transactions successfully because of a network problem or something else, we still provide a solution:  
        - The synchronizers will connect to several native nodes of every public chain to avoid the malicious native nodes.
        - If it indeed happened that all synchronizers' network break, the o-transaction will be synchronized when the network recovered. If the waiting time is up and the cheating o-transaction has been executed, we will revert it from where the cheating happens according to the `nonce in o-transaction` and `account nonce`.
- `A` will be punished(lock his account or something else, and this is about the certain tokenomics determined by developers according to their own situation).  

- **Conclusion: The *common user* maybe cheat but won't succeed.**

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).