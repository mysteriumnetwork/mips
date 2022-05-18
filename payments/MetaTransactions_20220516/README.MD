# Adding EIP-2771 support into Mysterium meta transactions

## Motivation

When in 2019 we were working on payment system there was no known solutions for “meta transactions”. Even this terminology was not a thing so we invented not only own solution but also terminology -- “transaction maker” or “transactor” (see 4.6 in ["Accountant pattern"](https://github.com/mysteriumnetwork/payments-smart-contracts/blob/master/docs/paper/accountant-pattern.pdf) paper).

Our solution was good enough for MVP phase, however it could be improved by two main properties:
 - enable potential interoperability with other solutions for meta transaction;
 - resolve issues with front running and losses transactor funds due to failing transactions;

Now, in 2022, there are a few widely used meta transaction solutions (e.g. GSN, Biconomy, Infura ITX). There are a few EIPs which are dedicated for meta transactions (e.g. `EIP-2770`, `EIP-2771`, `EIP-3005`, `EIP-3009`, `ERC-1776` or even `ERC-2612`).

Crypto community (building on top of EVM compatible chains) has reached some consensus on minimal interoperability solution for meta transactions support on smart contracts level. [_EIP2771_](https://eips.ethereum.org/EIPS/eip-2771) becomes de facto choice for developers who would like to add meta transactions support into their smart contracts. It is used by many projects and is supported by OpenZeppelin, GSN, Biconomi and Infura. All of them have own nuances and features, however `EIP2771` gives at least some possibility for interoperability and ads possibility to switch between solutions.

It could be beneficial to be in sync with wider crypto community and addapt widely used standarts instead of using home made stuff.

## Key proposal

We could refactor our meta transactions solution and add `EIP-2771` support on smart contract level. In same time we could stay with own protocol for relayers, very similar to current transactor but with minimal changes which would allow to avaid front running (e.g. adding additional fields where transaction could be relayed only by chosen relayer) and with node changes so user could add fallback relayers in config.

We already have configurable and interoperable solution for JSONRPC calls (reading data from blockchain), such implementation would add configurable and interoperable solution for meta transactions (writing data into blockchain).

I'd like our meta transactions to be done in _Web3_ spirit -- give decentralisation and add possibility of more control for node runners.

## Implementation details

### Vocabulary

- __Meta transaction__ - transaction that have been authorized by the _Transaction Signer_ (for example, signed by an externally owned account) and relayed by an untrusted third party (known as _Relay Server_) that pays for the blockchain fees (a.k.a. gas).

- __Relayer__ - also know as _Relay Server_ or _Transactor_. It is a service which receives signed request by client application and relayes it into blockchain by paying ETH or MATIC for transaction fees. In return such service is receiving MYST tokens.

- __Trusted Forwarder__ (or simly _Forwarder_) - a smart contract that is trusted by the _Recipient_ to correctly verify the signature and nonce before forwarding the request from Transaction Signer.

- __Reciepient__ - a contract that can securely accept meta-transactions through a Trusted Forwarder by being compliant with `EIP-2117`.

### Smart contrancts level changes

Instead of using `msg.sender` to define who signed the transaction, `EIP-2117` is suggesting to use `msgSender()` function instead, which will extract sender from last 20 bytes of `msg.data` in case when call was made by _Trusted Forwarder__.

```Solidity
function _msgSender() internal view returns (address payable signer) {
    signer = msg.sender;
    if (msg.data.length>=20 && isTrustedForwarder(signer)) {
        assembly {
            signer := shr(96,calldataload(sub(calldatasize(),20)))
        }
    }
}
```

Contract could have a few _Trusted Forwarders_ with different transaction relaying logic. The simple _Frowarder_ can simply have list of whitelisted relayers or could validate original sender's signature.

```Solidity
import "../utils/cryptography/ECDSA.sol";

contract SimpleForwarder {
    using ECDSA for bytes32;

    mapping(address => uint256) private _nonces;
    struct ForwardRequest {
        address from;
        address to;
        address relayer;
        uint256 gas;
        uint256 nonce;
        bytes data;     // formated payload to call wanted function with params on `to` smart contract address
    }

    function verify(ForwardRequest calldata req, bytes calldata signature) public view returns (bool) {
        address signer = _hashTypedDataV4(
            keccak256(abi.encode(req.from, req.to, req.relayer, req.gas, req.nonce, keccak256(req.data)))
        ).recover(signature);

        return _nonces[req.from] == req.nonce && signer == req.from && msg.sender == req;
    }

    function execute(ForwardRequest calldata req, bytes calldata signature) public payable returns (bool, bytes memory) {
        require(req.relayer == msg.sender, "Only choser relayer can send this tx");
        require(verify(req, signature), "Signature does not match request");
        _nonces[req.from] = req.nonce + 1;

        (bool success, bytes memory returndata) = req.to.call{gas: req.gas}(abi.encodePacked(req.data, req.from));

        return (success, returndata);
    }
}
```

Such implementation requires minimal changes into current `HermesImplementation.sol` code, it also maintains ideally same payment promise structure are we have now (`abi.encodePacked(getChainID(), _channelId, _amount, _transactorFee, _hashlock)`). However `_transactorFee` could be calculated and be exchanged from MYST to MATIC directly during transaction time.

```Solidity
function execute(ForwardRequest calldata req, bytes calldata signature) public payable returns (bool, bytes memory) {
    require(req.relayer == msg.sender, "Only choser relayer can send this tx");
    require(verify(req, signature), "Signature does not match request");
    _nonces[req.from] = req.nonce + 1;

    // Calculate charge
    uint256 gasUsed = req.gas + 21000 + SWAP_GAS_USAGE;

    uint256 chargeableGasPrice = tx.gasprice;
    uint256 txFee = gasUsed * chargeableGasPrice;
    uint256 _reserveIn = IERC20(WETHAddress).balanceOf(_pool);
    uint256 reserveOut = IERC20(MYSTAddress).balanceOf(_pool);
    uint256 transactorFee = UniswapV2Library.getAmountOut(txFee, reserveIn, reserveOut);

    (bool success, bytes memory returndata) = req.to.call{gas: req.gas}(abi.encodePacked(reqData, transactorFee, req.from));

    // Swap into MATIC and refund relayer
    IUniswapRouter02(ROUTER).swapExactTokensForETH(
        transactorFee,
        0,
        [WETHAddress, MYSTAddress],
        msg.sender,
        block.timestamp
    );

    return (success, returndata);
}
```

In such case we would need to delete `_transactorFee` from payment promise and adapt `selltePromise` and other funtions to pay fee if transaction is done from _Forwarder_. This also would a little change transactor's api, node could simply call `​/identity​/settle_and_rebalance` without asking for fee. TransactorFee would be always calculated on smart contract level and rate extracted from DEX.

It is worth to mention that instead of having universal `execute()` function we could have mirror functions of hermes or registry (e.g. `settlePromise`) and simpified process of how we're constructing payload. In any case this is not a complex problem to solve.

![Meta tx call via Forwarder](/payments/MetaTransactions_20220516/MetaTransactions.png)

### Transactor level changes

At the moment we have `​/identity​/settle_and_rebalance` call which is accepting payment promise and some additional metadata:

```
{
  "amount": "string",
  "chainID": 0,
  "channelID": "string",
  "fee": "string",
  "hermesID": "string",
  "preimage": "string",
  "providerID": "string",
  "signature": "string"
}
```

This call would need additionally expect missing forwarder request metadata and validate it's signature:

```
{
  ...
  "gas": "string",
  "nonce": "integer",
  "metaTxSignature": "string"
}
```

Example metaTxSignature creation format:

```go
type ForwardRequest struct {
    From      common.Address  // ProviderId
    To        common.Address  // HermesContractAddress
    Relayer   common.Address  // Chosen relayer identity
    Gas       *big.Int        // Gas needed to execute chosen transaction
    Nonce     *big.Int        // Incrementing counter per relayer<>forwarder pair
    Data      Promise         // PaymentPromise
    Signature []byte
}

func (r ForwardRequest) GetHash() {
    request := []byte{}
    request = append(request, r.From.Bytes()...)
    request = append(request, r.To.Bytes()...)
    request = append(request, r.Relayer.Bytes()...)
    request = append(request, Pad(math.U256(r.Gas).Bytes(), 32)...)
    request = append(request, Pad(math.U256(r.Nonce).Bytes(), 32)...)
    request = append(request, r.Data.GetHash()...)

    return request
}

func (r ForwardRequest) CreateSignature(ks hashSigner, signer common.Address) {
    return ks.SignHash(
        accounts.Account{Address: signer},
        r.GetHash(),
    )
}
```

Finally, after signature verification is done, there is one more change on what transactor will do. Instead of calling hermes smart contract directly, he should call `execute(ForwardRequest calldata req, bytes calldata signature)` function on _Forwarder_.

### Node level changes

There are two potential node level changes. First, node should stop calling fee endpoint and start sending additional metadata and metaTxSignature (see previous section on how `ForwardRequest` may look). Second change is to allow user set relayer's (transactor's) URL in config and having possibility to set secondary relayer. Node should also watch if requested transaction was mined and if not in some period of time (e.g. in 3 minutes) he should repease it with another relayer.

## Benefits

- Enables potential interoperability with other solutions. E.g. Mysterium meta transactions could become GSN compatible in the future or start using Infura’s ITX as fallback option for home made transactors —> improve network stability.
- This way allows us to calculate tx fees or even swap MYST into MATIC (via QuickSwap or UniswapV3) during settlement transaction —> no need to calculate and invalidate fees on transactor level, no risk of loss because of MYST rate etc.
- Done in spirit of web3 (gives decentralisation and possibility of control for node runners).
- It prevents transactor's funds loss due to tx front running which is happening after Mysterium mainnet release.
- In such way we still maintain possibility for users to pay for gas by themself, without using meta transactions.
- Using EIP-2117 give possibility to call any function in our smart contracts via meta transactions, not only currently supported `settlePromise` or `registerIdentity`.