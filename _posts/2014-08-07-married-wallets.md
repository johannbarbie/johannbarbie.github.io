---
layout: post
title: Introduction to Married Wallets
excerpt: "Married Wallets, asking the significant other’s permission to spend."
modified: 2014-08-07
tags: [java, bitcoin, bitcoinj, multi-sig, bip32]
comments: true
image:
  feature: texture-feature-05.jpg
  credit: Texture Lovers
  creditlink: http://texturelovers.com
---

<section id="table-of-contents" class="toc">
  <header>
    <h3>Overview</h3>
  </header>
<div id="drawer" markdown="1">
*  Auto generated table of contents
{:toc}
</div>
</section><!-- /#table-of-contents -->

# 1. Introduction

When Satoshi Nakamoto introduced the idea of "A Peer-to-Peer Electronic Cash System" in 2008 he intended to obviate the need for "financial institutions serving as trusted third parties to process electronic payments"[^1]. Every user can protect their coins using public key cryptography and initiate transactions by signing with private keys only they control. 

Adoption of Bitcoin beyond a tech-savvy user community has led to tensions between its design for individual control and the inexperience of new users. Services that host private keys make users believe that they are in control and that their funds are safe. These services are preposterous by nature, and simply re-introduce counterparty risk, departing from the very vision of Bitcoin.

Distributed contracts[^2] created between a service provider and a user can solve this problem and provide increased security or advanced services for users without the need for trust. Unfortunately, the Bitcoin block chain is limited in it’s functionality. It can not have a notion of external states and does not allow the execution of arbitrary logic that would be required for many contracts.

Married Wallets can be described as setups of joint control between a user's wallet and an oracle, a third party service that evaluates logic and real-world conditions. Mike Hearn, lead developer of the bitcoinj library, coined the term Married Wallet[^3] and developed the design. Konstantin Korenkov[^4] and Johann Barbie have started the first implementation.

This article will first introduce the Bitcoin Improvement Proposals[^5] that enable the setup of joint control before describing the design choices that enable Married Wallets. The article will close with a simple example of implementing a remote signer and give an outlook on further development.

# 2. Enabling Technologies

Advancements in wallet technology and Bitcoin protocol improvements have enabled schemes to create joint control on the block chain. Joint control for funds is created by interlinking the keychains of multiple distinct instances of wallets in a way that requires a majority of instances to cooperate in order to spend. perhaps discuss the specifics/dynamics of M-of-M and M-of-N, use a workable example?

The Pay-To-Script-Hash transaction type allows an efficient representation of multi-signature scripts while Hierarchical Deterministic wallets allow for one-time setup of such joint control. These technologies will be described further in this section.

## BIP 11 - M-of-N Standard Transactions

OP_CHECKMULTISIG is one of the opcodes from the Bitcoin Script language for cryptographic operations. A Script, a set of opcodes, is included with every output of transactions on the block chain and is evaluated against a spending transaction’s input.

<figure>
  <img src="/images/mw1.png">
  <figcaption>Basic transaction structure.</figcaption>
</figure>

The CHECKMULTISIG opcode will take all signatures that are provided by the scriptSig of the redeeming input and verify their validity with the public keys in the script of the redeemed output. In the given example, the script will only return success if at least 2 of the 3 public keys match the signatures. 

{% highlight bash %}
OP_0 {signature} {signature}
{% endhighlight %}

{% highlight bash %}
OP_2 {pubkey} {pubkey} {pubkey} OP_3 OP_CHECKMULTISIG
{% endhighlight %}

A script output like this has multiple disadvantages. Due to its size an address can not be generated from it. A multi-sig output also makes the funding of such a multi-signature script expensive for the payee.

## BIP 16 - Pay-to-Script Hash

Bitcoin Improvement Proposal 16 solves the issues with the previously stated OP_CHECKMULTISIG outputs by "mov[ing] the responsibility for supplying the conditions to redeem a transaction from the sender of the funds to the redeemer."[^6]

It does so by creating a new standard transaction type called Pay-to-Script-Hash (P2SH). An input with a scriptSig containing signatures and a serialized script hashing to the given script hash in the output can claim this output. 

{% highlight bash %}
...signatures ... <serialized OP_CHECKMULTISIG>
{% endhighlight %}

{% highlight bash %}
OP_HASH160 <scriptHash> OP_EQUAL
{% endhighlight %}

The redeeming script has to evaluate successfully with the signatures. Because the serialized script is hashed to a 20-byte hash, now the size of the P2SH input is always the same, thus making the creation of addresses as simple as previous Pay-to-PubkeyHash addresses.

## BIP 32 - Hierarchical Deterministic Wallets

The original Satoshi Bitcoin client features a wallet with a “bag of keys” implementation. It randomly pre-generates 100 keys on creation. Once a key is used by receiving or sending a transaction the key pool is refilled with new, randomly generated keys. A wallet design that periodically introduces new entropy is problematic for use cases like backups or wallet synchronization and prohibits the existence of more than one instance of the same wallet at a time, as they would run out of sync on key creation.

The Hierarchical Deterministic Wallet proposal introduces a clever design that combines a single seed of entropy with standard key derivation to create a deterministic set of key pairs that can be re-generated given the initial seed of entropy.

<figure>
  <img src="/images/mw2.png">
  <figcaption><a href="https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki" title="Hirarchical Deterministic Wallet layout">Hirarchical Deterministic Wallet layout</a>.</figcaption>
</figure>



Hierarchical Deterministic Wallets also utilize schemes in elliptic curve mathematics to calculate public keys without revealing the private keys. This design not only overcomes the drawbacks of the original wallet implementation, it also allows for separate accounts and selective levels of sharing within the hierarchy.

Selective sharing would be impossible if keys within the chain of derivation would merely depend on each other. Instead, additional entropy, the chain code, is introduced within the seed that can be combined with a private or public key to assemble extended keys. Extended public and private keys can then be shared with other systems and allow for watch-only or spending wallets. By using a lookahead of pre-generated unspent keys, multiple instances of wallets can stay in sync with each other using just the Bitcoin block chain.

BIP 32 is an extensive proposal and is complemented by BIP 43 and 44 to introduce additional structure to make wallets interoperable.  tbd -> read more somewhere else if you are interested.

# 3. Married Wallets

Married Wallets combine the improvement proposals introduced in the previous section. During the setup of joint control, entropy seeds are initialize in every instance of the wallet and extended public keys are shared. 
Throughout the lifetime of a Married Wallet every transaction will require a majority consensus. The implementation provides an interface for application developers to plug in their specific implementation of signature exchange.

## Following Key Chains

During the setup, each instance exports an extended public key of the account it wants to use to vote on spending decisions in the wallet. We call the keychain of this account the followed keychain. Then, all extended public keys are shared with all instances. 

<figure>
  <img src="/images/mw3.png">
  <figcaption>Married Wallet - setup.</figcaption>
</figure>

Each instance then imports all but its own extended public keys and creates watch-only keychains that track its followed keychain. We call the tracking chains that have no private keys, “following key chains”. From the setup on, the followed keychain and following keychains provide us with tuples of public keys at every position in a chain.

<figure>
  <img src="/images/mw4.png">
  <figcaption>Married Wallet - key derivation and relationship.</figcaption>
</figure>

When the lookahead for the wallet is generated each of the tuples will be combined into a redeem script using OP_CHECKMULTISIG and with the P2SH transaction type we can create addresses to represent each of the tuples. With this functionality in place we are able to fund a married wallet and all instances will be able to track the wallet balance independently using only the block chain.


## Pluggable Signers

Once married the different instances of a married wallet can derive new public keys and create new addresses without intermediate communication. Through the block chain all instances synchronize balances automatically. But what the instances can not do is spending without requiring a vote from the voting pool.

To implement custom signature collection schemes Married Wallets provide a TransactionSigner interface. The transaction signer is given an unsigned transaction object and a set of transaction outputs with its related redeem script and keys. In the mentioned set of keys there is one key that is from the followed keychain and one or many public keys from the following keychains. For each following key a signature has to be gathered from a remote instance of the wallet. 

Each service provides their own implementation of a TransactionSigner that implements the details of signature collection. An example for a simple TransactionSigner could be implemented like this:

{% highlight java %}
/**
 * This signer tries to sign inputs with keys it has.
 * If key has no private bytes, signer asks mock server to provide signature. 
 * Copyright 2014 Kosta Korenkov
 */
public class TestP2SHTransactionSigner implements TransactionSigner {

  @Override
  public TransactionSignature[][] signInputs(Transaction tx,
 Map<TransactionOutput, RedeemData> redeemData) {
    int numInputs = tx.getInputs().size();
    int numSigs = redeemData.values().iterator().next().getKeys().size();
    TransactionSignature[][] signatures = 
      new TransactionSignature[numInputs][numSigs];
    for (int i = 0; i < numInputs; i++) {
      TransactionInput txIn = tx.getInput(i);
      TransactionOutput txOut = txIn.getOutpoint().getConnectedOutput();
      if (!redeemData.containsKey(txOut))
        continue;
      checkArgument(txOut.getScriptPubKey().isPayToScriptHash(),
 "TestP2SHTransactionSigner works only with P2SH transactions");
      Script redeemScript = redeemData.get(txOut).getRedeemScript();
      Sha256Hash sighash = tx.hashForSignature(i, redeemScript, 
        Transaction.SigHash.ALL, false);
      List<ECKey> keys = redeemData.get(txOut).getKeys();
      // no need to calculate all signatures for N of M transaction,
      //we need only minimum number required to spend
      int treshold = redeemScript.getNumberOfSignaturesRequiredToSpend();
      for (int j = 0; j < treshold; j++) {
        ECKey key = keys.get(j);
        try {
          signatures[i][j] = new TransactionSignature(key.sign(sighash), 
     Transaction.SigHash.ALL, false);
        } catch (ECKey.KeyIsEncryptedException e) {
          throw e;
        } catch (ECKey.MissingPrivateKeyException e) {
           // if key has no private key bytes, 
           // we asking signing server to provide signature for it
           signatures[i][j] = new TransactionSignature(
            getTheirSignature(sighash, key), Transaction.SigHash.ALL, false);
        }
      }
    }
    return signatures;
  }
}
{% endhighlight %}

An example for a simple remote signer could be implemented like this:

{% highlight java %}
@Path("/signer")
public class Signer {
    private static final byte[] ENTROPY = 
      Sha256Hash.create("don't use a seed like this".getBytes()).getBytes();
    private static DeterministicKeyChain keyChain;

    static {
        keyChain = new DeterministicKeyChain(ENTROPY, "", 1389353062L);
        keyChain.setLookaheadSize(2);
    }

    @POST
    @Path("/sign")
    @Produces(MediaType.TEXT_PLAIN)
    public byte[] sign(@FormParam("sighash") String hexSighash, 
                       @FormParam("keypath") String keyPath) {
        Sha256Hash sighash = new Sha256Hash(hexSighash);
        ImmutableList<ChildNumber> path = 
            ImmutableList.copyOf(HDUtils.parsePath(keyPath));

        DeterministicKey key = keyChain.getKeyByPath(path, true);
        ECKey.ECDSASignature signature = key.sign(sighash);
        byte[] sig = signature.encodeToDER();
        return sig;
    }
}
{% endhighlight %}

The depicted code represents a mock server that just always signs any transaction and hands back the signatures. It is a simple template for implementing a hosted service in joint control for wallet spending. A wallet protection service would typically monitor spending volume and employ multi-factor authentication to protect its customers’ funds.

# 4. Conclusion

Married Wallets propose a new aptronym to Bitcoin wallets that combine Hierarchical Deterministic features with multi-signature. The term is more expressive, as it "conjures the image of the wallet being required to get the significant others permission"[^3]. The use case of multi-signature is narrowed down to an always-on remote signer.

Married Wallets implement best practices in wallet design and multi-signature to create a simple and extensible platform for application developers and users. The architecture makes creating new secure wallet services as simple as implementing an interface and using them as simple as installing a plugin.

The bitcoinj library that Married Wallets are built on is one of few SPV implementations in the bitcoin space. An SPV client with pluggable oracle support will allow for a multitude of new mobile applications in the consumer space.

## Open Tasks

Some aspects of the implementation have yet to be delivered. After a TransactionSigner is initialized it will need to store data for a user’s account in the wallet file. This data needs to be handed back to the signer during construction so it is ready to go. To avoid unstable states after wallet deserialization an isReady() interface has to be available.

Mobile wallets pose specific requirements due to limited system resources. A signing process can be interrupted at any time due to external events like an incoming phone call. The event might force the bitcoin wallet out of memory, raising the need for an interruptible, serializable and asynchronous signing process.[^3]

## An Aside

Vitalik Buterin talks in his recent article “Multisig: A Revolution Incomplete“[^7] about the state of multi-signature wallets in the Bitcoin industry. He points that businesses should avoid “building a ‘moat’” but rather embrace “principles of commoditization, generalization and separation of concerns that are so important for a robust decentralized ecosystem”. 

I want to expand this statement and stress the importance of collaboration, standardization and education. A few companies in the space claim to have custom bitcoinj implementations of HD-Multisig wallets. They have not contributed their implementations and by this hinder the adoption and standardization of multi-sig technologies.


[^1]: Satoshi Nakamoto,  A Peer-to-Peer Electronic Cash System ([https://bitcoin.org/bitcoin.pdf](https://bitcoin.org/bitcoin.pdf), 2008)
[^2]: [https://en.bitcoin.it/wiki/Contracts](https://en.bitcoin.it/wiki/Contracts)
[^3]: Mike Hearn, ([Married Wallets Design Document](https://groups.google.com/forum/#!msg/bitcoinj/Uxl-z40OLuQ/e2m4mEWR6gMJ), 2014)
[^4]: [https://www.linkedin.com/profile/view?id=229577067](https://www.linkedin.com/profile/view?id=229577067)
[^5]: [https://en.bitcoin.it/wiki/Bitcoin_Improvement_Proposals](https://en.bitcoin.it/wiki/Bitcoin_Improvement_Proposals)
[^6]: [https://github.com/bitcoin/bips/blob/master/bip-0016.mediawiki](https://github.com/bitcoin/bips/blob/master/bip-0016.mediawiki)
[^7]: [http://bitcoinmagazine.com/15290/multisig-revolution-incomplete/](http://bitcoinmagazine.com/15290/multisig-revolution-incomplete/)