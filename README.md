<pre>
  BIP: ???
  Layer: Applications
  Title: Signatures of Messages using Bitcoin Private Keys
  Author: Christopher Gilliard &ltchristopher.gilliard@gmail.com&gt
  Comments-Summary: No comments yet.
  Comments-URI: TBD
  Status: Proposal
  Type: Informational
  Created: 2019-02-16
  License: BSD-2-Clause
</pre>

==Abstract==

This document describes a signature format for signing messages with Bitcoin private keys.

The specification is intended to set a standard for signatures of messages that can be interchanged between different clients.

One of the key problems in this area is that there are several different types of Bitcoin addresses and without introducing specific standards it is unclear which type of address format is being used. See [1]. This BIP will attempt to address these issues.

==Copyright==

This BIP is licensed under the 2-clause BSD license.

==Motivation==

Since Bitcoin private keys can not only be used to sign Bitcoin transactions it has become customary to use them to sign various messages for differing purposes. Some applications of signing messages with a Bitcoin private key are as follows: proof of funds for collateral, credit worthiness, enterence to events, airdrops, audits as well as other applications. While there was no BIP written for how to digitally sign messages with Bitcoin private keys with P2PKH addresses it is a fairly well understood process, however with the introduction of Segwit (both in the form of P2SH and bech32) addresses, it is unclear how to distinguish a P2PKH, P2SH, or bech32 address from one another. This BIP proposes a standard signature format that will allow clients to distinguish between the different address formats.

==Specification==

===Background on ECDSA Signatures===

In the rest of this text we will assume the public key cryptography used in Bitcoin, namely elliptic curve cryptography using the field and curve parameters defined by secp256k1 (http://www.secg.org/sec2-v2.pdf). Variables below are either:
* Integers modulo the order of the curve (referred to as n).
* Coordinates of points on the curve.
* Byte sequences.

Addition (+) of two coordinate pair is defined as application of the EC group operation.
Concatenation (||) is the operation of appending one byte sequence onto another.

As standard conversion functions, we assume:
* point(p): returns the coordinate pair resulting from EC point multiplication (repeated application of the EC group operation) of the secp256k1 base point with the integer p.
* ser<sub>32</sub>(i): serialize a 32-bit unsigned integer i as a 4-byte sequence, most significant byte first.
* ser<sub>256</sub>(p): serializes the integer p as a 32-byte sequence, most significant byte first.
* ser<sub>P</sub>(P): serializes the coordinate pair P = (x,y) as a byte sequence using SEC1's compressed form: (0x02 or 0x03) || ser<sub>256</sub>(x), where the header byte depends on the parity of the omitted y coordinate.
* parse<sub>256</sub>(p): interprets a 32-byte sequence as a 256-bit number, most significant byte first.


===Extended keys===

In what follows, we will define a function that derives a number of child keys from a parent key. In order to prevent these from depending solely on the key itself, we extend both private and public keys first with an extra 256 bits of entropy. This extension, called the chain code, is identical for corresponding private and public keys, and consists of 32 bytes.

We represent an extended private key as (k, c), with k the normal private key, and c the chain code. An extended public key is represented as (K, c), with K = point(k) and c the chain code.

Each extended key has 2<sup>31</sup> normal child keys, and 2<sup>31</sup> hardened child keys. Each of these child keys has an index. The normal child keys use indices 0 through 2<sup>31</sup>-1. The hardened child keys use indices 2<sup>31</sup> through 2<sup>32</sup>-1. To ease notation for hardened key indices, a number i<sub>H</sub> represents i+2<sup>31</sup>.

===Child key derivation (CKD) functions===

Given a parent extended key and an index i, it is possible to compute the corresponding child extended key. The algorithm to do so depends on whether the child is a hardened key or not (or, equivalently, whether i ≥ 2<sup>31</sup>), and whether we're talking about private or public keys.

====Private parent key &rarr; private child key====

The function CKDpriv((k<sub>par</sub>, c<sub>par</sub>), i) &rarr; (k<sub>i</sub>, c<sub>i</sub>) computes a child extended private key from the parent extended private key:
* Check whether i ≥ 2<sup>31</sup> (whether the child is a hardened key).
** If so (hardened child): let I = HMAC-SHA512(Key = c<sub>par</sub>, Data = 0x00 || ser<sub>256</sub>(k<sub>par</sub>) || ser<sub>32</sub>(i)). (Note: The 0x00 pads the private key to make it 33 bytes long.)
** If not (normal child): let I = HMAC-SHA512(Key = c<sub>par</sub>, Data = ser<sub>P</sub>(point(k<sub>par</sub>)) || ser<sub>32</sub>(i)).
* Split I into two 32-byte sequences, I<sub>L</sub> and I<sub>R</sub>.
* The returned child key k<sub>i</sub> is parse<sub>256</sub>(I<sub>L</sub>) + k<sub>par</sub> (mod n).
* The returned chain code c<sub>i</sub> is I<sub>R</sub>.
* In case parse<sub>256</sub>(I<sub>L</sub>) ≥ n or k<sub>i</sub> = 0, the resulting key is invalid, and one should proceed with the next value for i. (Note: this has probability lower than 1 in 2<sup>127</sup>.)

The HMAC-SHA512 function is specified in [http://tools.ietf.org/html/rfc4231 RFC 4231].

====Public parent key &rarr; public child key====

The function CKDpub((K<sub>par</sub>, c<sub>par</sub>), i) &rarr; (K<sub>i</sub>, c<sub>i</sub>) computes a child extended public key from the parent extended public key. It is only defined for non-hardened child keys.
* Check whether i ≥ 2<sup>31</sup> (whether the child is a hardened key).
** If so (hardened child): return failure
** If not (normal child): let I = HMAC-SHA512(Key = c<sub>par</sub>, Data = ser<sub>P</sub>(K<sub>par</sub>) || ser<sub>32</sub>(i)).
* Split I into two 32-byte sequences, I<sub>L</sub> and I<sub>R</sub>.
* The returned child key K<sub>i</sub> is point(parse<sub>256</sub>(I<sub>L</sub>)) + K<sub>par</sub>.
* The returned chain code c<sub>i</sub> is I<sub>R</sub>.
* In case parse<sub>256</sub>(I<sub>L</sub>) ≥ n or K<sub>i</sub> is the point at infinity, the resulting key is invalid, and one should proceed with the next value for i.

====Private parent key &rarr; public child key====

The function N((k, c)) &rarr; (K, c) computes the extended public key corresponding to an extended private key (the "neutered" version, as it removes the ability to sign transactions).
* The returned key K is point(k).
* The returned chain code c is just the passed chain code.

To compute the public child key of a parent private key:
* N(CKDpriv((k<sub>par</sub>, c<sub>par</sub>), i)) (works always).
* CKDpub(N(k<sub>par</sub>, c<sub>par</sub>), i) (works only for non-hardened child keys).
The fact that they are equivalent is what makes non-hardened keys useful (one can derive child public keys of a given parent key without knowing any private key), and also what distinguishes them from hardened keys. The reason for not always using non-hardened keys (which are more useful) is security; see further for more information.

====Public parent key &rarr; private child key====

This is not possible.

===The key tree===

The next step is cascading several CKD constructions to build a tree. We start with one root, the master extended key m. By evaluating CKDpriv(m,i) for several values of i, we get a number of level-1 derived nodes. As each of these is again an extended key, CKDpriv can be applied to those as well.

To shorten notation, we will write CKDpriv(CKDpriv(CKDpriv(m,3<sub>H</sub>),2),5) as m/3<sub>H</sub>/2/5. Equivalently for public keys, we write CKDpub(CKDpub(CKDpub(M,3),2),5) as M/3/2/5. This results in the following identities:
* N(m/a/b/c) = N(m/a/b)/c = N(m/a)/b/c = N(m)/a/b/c = M/a/b/c.
* N(m/a<sub>H</sub>/b/c) = N(m/a<sub>H</sub>/b)/c = N(m/a<sub>H</sub>)/b/c.
However, N(m/a<sub>H</sub>) cannot be rewritten as N(m)/a<sub>H</sub>, as the latter is not possible.

Each leaf node in the tree corresponds to an actual key, while the internal nodes correspond to the collections of keys that descend from them. The chain codes of the leaf nodes are ignored, and only their embedded private or public key is relevant. Because of this construction, knowing an extended private key allows reconstruction of all descendant private keys and public keys, and knowing an extended public keys allows reconstruction of all descendant non-hardened public keys.

===Key identifiers===

Extended keys can be identified by the Hash160 (RIPEMD160 after SHA256) of the serialized ECDSA public key K, ignoring the chain code. This corresponds exactly to the data used in traditional Bitcoin addresses. It is not advised to represent this data in base58 format though, as it may be interpreted as an address that way (and wallet software is not required to accept payment to the chain key itself).

The first 32 bits of the identifier are called the key fingerprint.

===Serialization format===

Extended public and private keys are serialized as follows:
* 4 byte: version bytes (mainnet: 0x0488B21E public, 0x0488ADE4 private; testnet: 0x043587CF public, 0x04358394 private)
* 1 byte: depth: 0x00 for master nodes, 0x01 for level-1 derived keys, ....
* 4 bytes: the fingerprint of the parent's key (0x00000000 if master key)
* 4 bytes: child number. This is ser<sub>32</sub>(i) for i in x<sub>i</sub> = x<sub>par</sub>/i, with x<sub>i</sub> the key being serialized. (0x00000000 if master key)
* 32 bytes: the chain code
* 33 bytes: the public key or private key data (ser<sub>P</sub>(K) for public keys, 0x00 || ser<sub>256</sub>(k) for private keys)

This 78 byte structure can be encoded like other Bitcoin data in Base58, by first adding 32 checksum bits (derived from the double SHA-256 checksum), and then converting to the Base58 representation. This results in a Base58-encoded string of up to 112 characters. Because of the choice of the version bytes, the Base58 representation will start with "xprv" or "xpub" on mainnet, "tprv" or "tpub" on testnet.

Note that the fingerprint of the parent only serves as a fast way to detect parent and child nodes in software, and software must be willing to deal with collisions. Internally, the full 160-bit identifier could be used.

When importing a serialized extended public key, implementations must verify whether the X coordinate in the public key data corresponds to a point on the curve. If not, the extended public key is invalid.

===Master key generation===

The total number of possible extended keypairs is almost 2<sup>512</sup>, but the produced keys are only 256 bits long, and offer about half of that in terms of security. Therefore, master keys are not generated directly, but instead from a potentially short seed value.

* Generate a seed byte sequence S of a chosen length (between 128 and 512 bits; 256 bits is advised) from a (P)RNG.
* Calculate I = HMAC-SHA512(Key = "Bitcoin seed", Data = S)
* Split I into two 32-byte sequences, I<sub>L</sub> and I<sub>R</sub>.
* Use parse<sub>256</sub>(I<sub>L</sub>) as master secret key, and I<sub>R</sub> as master chain code.
In case I<sub>L</sub> is 0 or ≥n, the master key is invalid.

<img src=bip-0032/derivation.png></img>

==Specification: Wallet structure==

The previous sections specified key trees and their nodes. The next step is imposing a wallet structure on this tree. The layout defined in this section is a default only, though clients are encouraged to mimic it for compatibility, even if not all features are supported.

===The default wallet layout===

An HDW is organized as several 'accounts'. Accounts are numbered, the default account ("") being number 0. Clients are not required to support more than one account - if not, they only use the default account.

Each account is composed of two keypair chains: an internal and an external one. The external keychain is used to generate new public addresses, while the internal keychain is used for all other operations (change addresses, generation addresses, ..., anything that doesn't need to be communicated). Clients that do not support separate keychains for these should use the external one for everything.
* m/i<sub>H</sub>/0/k corresponds to the k'th keypair of the external chain of account number i of the HDW derived from master m.
* m/i<sub>H</sub>/1/k corresponds to the k'th keypair of the internal chain of account number i of the HDW derived from master m.

===Use cases===

====Full wallet sharing: m====

In cases where two systems need to access a single shared wallet, and both need to be able to perform spendings, one needs to share the master private extended key. Nodes can keep a pool of N look-ahead keys cached for external chains, to watch for incoming payments. The look-ahead for internal chains can be very small, as no gaps are to be expected here. An extra look-ahead could be active for the first unused account's chains - triggering the creation of a new account when used. Note that the name of the account will still need to be entered manually and cannot be synchronized via the block chain.

====Audits: N(m/*)====

In case an auditor needs full access to the list of incoming and outgoing payments, one can share all account public extended keys. This will allow the auditor to see all transactions from and to the wallet, in all accounts, but not a single secret key.

====Per-office balances: m/i<sub>H</sub>====

When a business has several independent offices, they can all use wallets derived from a single master. This will allow the headquarters to maintain a super-wallet that sees all incoming and outgoing transactions of all offices, and even permit moving money between the offices.

====Recurrent business-to-business transactions: N(m/i<sub>H</sub>/0)====

In case two business partners often transfer money, one can use the extended public key for the external chain of a specific account (M/i h/0) as a sort of "super address", allowing frequent transactions that cannot (easily) be associated, but without needing to request a new address for each payment.
Such a mechanism could also be used by mining pool operators as variable payout address.

====Unsecure money receiver: N(m/i<sub>H</sub>/0)====

When an unsecure webserver is used to run an e-commerce site, it needs to know public addresses that are used to receive payments. The webserver only needs to know the public extended key of the external chain of a single account. This means someone illegally obtaining access to the webserver can at most see all incoming payments but will not be able to steal the money, will not (trivially) be able to distinguish outgoing transactions, nor be able to see payments received by other webservers if there are several.

==Compatibility==

To comply with this standard, a client must at least be able to import an extended public or private key, to give access to its direct descendants as wallet keys. The wallet structure (master/account/chain/subchain) presented in the second part of the specification is advisory only, but is suggested as a minimal structure for easy compatibility - even when no separate accounts or distinction between internal and external chains is made. However, implementations may deviate from it for specific needs; more complex applications may call for a more complex tree structure.

==Security==

In addition to the expectations from the EC public-key cryptography itself:
* Given a public key K, an attacker cannot find the corresponding private key more efficiently than by solving the EC discrete logarithm problem (assumed to require 2<sup>128</sup> group operations).
the intended security properties of this standard are:
* Given a child extended private key (k<sub>i</sub>,c<sub>i</sub>) and the integer i, an attacker cannot find the parent private key k<sub>par</sub> more efficiently than a 2<sup>256</sup> brute force of HMAC-SHA512.
* Given any number (2 ≤ N ≤ 2<sup>32</sup>-1) of (index, extended private key) tuples (i<sub>j</sub>,(k<sub>i<sub>j</sub></sub>,c<sub>i<sub>j</sub></sub>)), with distinct i<sub>j</sub>'s, determining whether they are derived from a common parent extended private key (i.e., whether there exists a (k<sub>par</sub>,c<sub>par</sub>) such that for each j in (0..N-1) CKDpriv((k<sub>par</sub>,c<sub>par</sub>),i<sub>j</sub>)=(k<sub>i<sub>j</sub></sub>,c<sub>i<sub>j</sub></sub>)), cannot be done more efficiently than a 2<sup>256</sup> brute force of HMAC-SHA512.
Note however that the following properties does not exist:
* Given a parent extended public key (K<sub>par</sub>,c<sub>par</sub>) and a child public key (K<sub>i</sub>), it is hard to find i.
* Given a parent extended public key (K<sub>par</sub>,c<sub>par</sub>) and a non-hardened child private key (k<sub>i</sub>), it is hard to find k<sub>par</sub>.

===Implications===

Private and public keys must be kept safe as usual. Leaking a private key means access to coins - leaking a public key can mean loss of privacy.

Somewhat more care must be taken regarding extended keys, as these correspond to an entire (sub)tree of keys.

One weakness that may not be immediately obvious, is that knowledge of a parent extended public key plus any non-hardened private key descending from it is equivalent to knowing the parent extended private key (and thus every private and public key descending from it). This means that extended public keys must be treated more carefully than regular public keys.
It is also the reason for the existence of hardened keys, and why they are used for the account level in the tree. This way, a leak of account-specific (or below) private key never risks compromising the master or other accounts.


==Test Vectors==

===Test vector 1===

Seed (hex): 000102030405060708090a0b0c0d0e0f
* Chain m
** ext pub: xpub661MyMwAqRbcFtXgS5sYJABqqG9YLmC4Q1Rdap9gSE8NqtwybGhePY2gZ29ESFjqJoCu1Rupje8YtGqsefD265TMg7usUDFdp6W1EGMcet8
** ext prv: xprv9s21ZrQH143K3QTDL4LXw2F7HEK3wJUD2nW2nRk4stbPy6cq3jPPqjiChkVvvNKmPGJxWUtg6LnF5kejMRNNU3TGtRBeJgk33yuGBxrMPHi
* Chain m/0<sub>H</sub>
** ext pub: xpub68Gmy5EdvgibQVfPdqkBBCHxA5htiqg55crXYuXoQRKfDBFA1WEjWgP6LHhwBZeNK1VTsfTFUHCdrfp1bgwQ9xv5ski8PX9rL2dZXvgGDnw
** ext prv: xprv9uHRZZhk6KAJC1avXpDAp4MDc3sQKNxDiPvvkX8Br5ngLNv1TxvUxt4cV1rGL5hj6KCesnDYUhd7oWgT11eZG7XnxHrnYeSvkzY7d2bhkJ7
* Chain m/0<sub>H</sub>/1
** ext pub: xpub6ASuArnXKPbfEwhqN6e3mwBcDTgzisQN1wXN9BJcM47sSikHjJf3UFHKkNAWbWMiGj7Wf5uMash7SyYq527Hqck2AxYysAA7xmALppuCkwQ
** ext prv: xprv9wTYmMFdV23N2TdNG573QoEsfRrWKQgWeibmLntzniatZvR9BmLnvSxqu53Kw1UmYPxLgboyZQaXwTCg8MSY3H2EU4pWcQDnRnrVA1xe8fs
* Chain m/0<sub>H</sub>/1/2<sub>H</sub>
** ext pub: xpub6D4BDPcP2GT577Vvch3R8wDkScZWzQzMMUm3PWbmWvVJrZwQY4VUNgqFJPMM3No2dFDFGTsxxpG5uJh7n7epu4trkrX7x7DogT5Uv6fcLW5
** ext prv: xprv9z4pot5VBttmtdRTWfWQmoH1taj2axGVzFqSb8C9xaxKymcFzXBDptWmT7FwuEzG3ryjH4ktypQSAewRiNMjANTtpgP4mLTj34bhnZX7UiM
* Chain m/0<sub>H</sub>/1/2<sub>H</sub>/2
** ext pub: xpub6FHa3pjLCk84BayeJxFW2SP4XRrFd1JYnxeLeU8EqN3vDfZmbqBqaGJAyiLjTAwm6ZLRQUMv1ZACTj37sR62cfN7fe5JnJ7dh8zL4fiyLHV
** ext prv: xprvA2JDeKCSNNZky6uBCviVfJSKyQ1mDYahRjijr5idH2WwLsEd4Hsb2Tyh8RfQMuPh7f7RtyzTtdrbdqqsunu5Mm3wDvUAKRHSC34sJ7in334
* Chain m/0<sub>H</sub>/1/2<sub>H</sub>/2/1000000000
** ext pub: xpub6H1LXWLaKsWFhvm6RVpEL9P4KfRZSW7abD2ttkWP3SSQvnyA8FSVqNTEcYFgJS2UaFcxupHiYkro49S8yGasTvXEYBVPamhGW6cFJodrTHy
** ext prv: xprvA41z7zogVVwxVSgdKUHDy1SKmdb533PjDz7J6N6mV6uS3ze1ai8FHa8kmHScGpWmj4WggLyQjgPie1rFSruoUihUZREPSL39UNdE3BBDu76

===Test vector 2===

Seed (hex): fffcf9f6f3f0edeae7e4e1dedbd8d5d2cfccc9c6c3c0bdbab7b4b1aeaba8a5a29f9c999693908d8a8784817e7b7875726f6c696663605d5a5754514e4b484542
* Chain m
** ext pub: xpub661MyMwAqRbcFW31YEwpkMuc5THy2PSt5bDMsktWQcFF8syAmRUapSCGu8ED9W6oDMSgv6Zz8idoc4a6mr8BDzTJY47LJhkJ8UB7WEGuduB
** ext prv: xprv9s21ZrQH143K31xYSDQpPDxsXRTUcvj2iNHm5NUtrGiGG5e2DtALGdso3pGz6ssrdK4PFmM8NSpSBHNqPqm55Qn3LqFtT2emdEXVYsCzC2U
* Chain m/0
** ext pub: xpub69H7F5d8KSRgmmdJg2KhpAK8SR3DjMwAdkxj3ZuxV27CprR9LgpeyGmXUbC6wb7ERfvrnKZjXoUmmDznezpbZb7ap6r1D3tgFxHmwMkQTPH
** ext prv: xprv9vHkqa6EV4sPZHYqZznhT2NPtPCjKuDKGY38FBWLvgaDx45zo9WQRUT3dKYnjwih2yJD9mkrocEZXo1ex8G81dwSM1fwqWpWkeS3v86pgKt
* Chain m/0/2147483647<sub>H</sub>
** ext pub: xpub6ASAVgeehLbnwdqV6UKMHVzgqAG8Gr6riv3Fxxpj8ksbH9ebxaEyBLZ85ySDhKiLDBrQSARLq1uNRts8RuJiHjaDMBU4Zn9h8LZNnBC5y4a
** ext prv: xprv9wSp6B7kry3Vj9m1zSnLvN3xH8RdsPP1Mh7fAaR7aRLcQMKTR2vidYEeEg2mUCTAwCd6vnxVrcjfy2kRgVsFawNzmjuHc2YmYRmagcEPdU9
* Chain m/0/2147483647<sub>H</sub>/1
** ext pub: xpub6DF8uhdarytz3FWdA8TvFSvvAh8dP3283MY7p2V4SeE2wyWmG5mg5EwVvmdMVCQcoNJxGoWaU9DCWh89LojfZ537wTfunKau47EL2dhHKon
** ext prv: xprv9zFnWC6h2cLgpmSA46vutJzBcfJ8yaJGg8cX1e5StJh45BBciYTRXSd25UEPVuesF9yog62tGAQtHjXajPPdbRCHuWS6T8XA2ECKADdw4Ef
* Chain m/0/2147483647<sub>H</sub>/1/2147483646<sub>H</sub>
** ext pub: xpub6ERApfZwUNrhLCkDtcHTcxd75RbzS1ed54G1LkBUHQVHQKqhMkhgbmJbZRkrgZw4koxb5JaHWkY4ALHY2grBGRjaDMzQLcgJvLJuZZvRcEL
** ext prv: xprvA1RpRA33e1JQ7ifknakTFpgNXPmW2YvmhqLQYMmrj4xJXXWYpDPS3xz7iAxn8L39njGVyuoseXzU6rcxFLJ8HFsTjSyQbLYnMpCqE2VbFWc
* Chain m/0/2147483647<sub>H</sub>/1/2147483646<sub>H</sub>/2
** ext pub: xpub6FnCn6nSzZAw5Tw7cgR9bi15UV96gLZhjDstkXXxvCLsUXBGXPdSnLFbdpq8p9HmGsApME5hQTZ3emM2rnY5agb9rXpVGyy3bdW6EEgAtqt
** ext prv: xprvA2nrNbFZABcdryreWet9Ea4LvTJcGsqrMzxHx98MMrotbir7yrKCEXw7nadnHM8Dq38EGfSh6dqA9QWTyefMLEcBYJUuekgW4BYPJcr9E7j

===Test vector 3===

These vectors test for the retention of leading zeros. See [https://github.com/bitpay/bitcore-lib/issues/47 bitpay/bitcore-lib#47] and [https://github.com/iancoleman/bip39/issues/58 iancoleman/bip39#58] for more information.

Seed (hex): 4b381541583be4423346c643850da4b320e46a87ae3d2a4e6da11eba819cd4acba45d239319ac14f863b8d5ab5a0d0c64d2e8a1e7d1457df2e5a3c51c73235be
* Chain m
** ext pub: xpub661MyMwAqRbcEZVB4dScxMAdx6d4nFc9nvyvH3v4gJL378CSRZiYmhRoP7mBy6gSPSCYk6SzXPTf3ND1cZAceL7SfJ1Z3GC8vBgp2epUt13
** ext prv: xprv9s21ZrQH143K25QhxbucbDDuQ4naNntJRi4KUfWT7xo4EKsHt2QJDu7KXp1A3u7Bi1j8ph3EGsZ9Xvz9dGuVrtHHs7pXeTzjuxBrCmmhgC6
* Chain m/0<sub>H</sub>
** ext pub: xpub68NZiKmJWnxxS6aaHmn81bvJeTESw724CRDs6HbuccFQN9Ku14VQrADWgqbhhTHBaohPX4CjNLf9fq9MYo6oDaPPLPxSb7gwQN3ih19Zm4Y
** ext prv: xprv9uPDJpEQgRQfDcW7BkF7eTya6RPxXeJCqCJGHuCJ4GiRVLzkTXBAJMu2qaMWPrS7AANYqdq6vcBcBUdJCVVFceUvJFjaPdGZ2y9WACViL4L

==Implementations==

Two Python implementations exist:

PyCoin (https://github.com/richardkiss/pycoin) is a suite of utilities for dealing with Bitcoin that includes BIP0032 wallet features.  BIP32Utils (https://pypi.org/project/bip32utils/) is a library and command line interface specifically focused on BIP0032 wallets and scripting.

2 Java implementations exist: https://github.com/bitsofproof/supernode/blob/1.1/api/src/main/java/com/bitsofproof/supernode/api/ExtendedKey.java and https://github.com/bushidowallet/bushido-java-core/tree/master/src/main/java/com/bushidowallet/core/bitcoin/bip32

A C++ implementation is available at https://github.com/ciphrex/mSIGNA/blob/master/deps/CoinCore/src/hdkeys.h

An Objective-C implementation is available at https://github.com/oleganza/CoreBitcoin/blob/master/CoreBitcoin/BTCKeychain.h

A Ruby implementation is available at https://github.com/GemHQ/money-tree

Two Go implementations exist:

hdkeychain (https://github.com/conformal/btcutil/tree/master/hdkeychain) provides an API for bitcoin hierarchical deterministic extended keys (BIP0032).  Go HD Wallet (https://github.com/WeMeetAgain/go-hdwallet).

Two JavaScript implementations exist: available at https://github.com/sarchar/brainwallet.github.com/tree/bip32 and https://github.com/bitpay/bitcore

A PHP implementation is available at https://github.com/Bit-Wasp/bitcoin-lib-php

A C# implementation is available at https://github.com/NicolasDorier/NBitcoin (ExtKey, ExtPubKey)

A Haskell implementation is available at https://github.com/haskoin/haskoin together with a CLI interface at https://github.com/np/hx

==Acknowledgements==

* Konstantin Bay - review
* Holly Casaletto - review
* James Bryrer - review


Reference:
[1] - https://github.com/bitcoin/bitcoin/issues/10542

