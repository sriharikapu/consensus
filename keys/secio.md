https://forum.web3.foundation/t/transport-layer-authentication-libp2ps-secio/69



## Transport layer authentication - libp2p's SECIO

There are numerous transports for libp2p, but only QUIC was actually designed to be secure.  Instead, one routes traffic through [libp2p's secio protocol](https://github.com/libp2p/specs/pull/106).  We trust QUIC's cryptographic layer which is TLS 1.3, but secio itself is a home brew job with no serious security analysis, which usually [goes](https://github.com/tendermint/tendermint/issues/3010) [poorly](https://github.com/tendermint/kms/issues/111).  

There has been minimal discussion of secio's security but [Dominic Tarr](https://github.com/auditdrivencrypto/secure-channel/blob/master/prior-art.md#ipfss-secure-channel) raised some concerns in the [original pull request](https://github.com/ipfs/go-ipfs/pull/34).  I'll reraise several concerns from that discussion: 

First, there is no effort made to hide secio node keys because "IPFS has no interest in [metadata privacy]" according to Brendan McMillion, so nodes leak their identities onto the raw internet.  We think identifying nodes sounds easy anyways, but at minimum this invites attacks.  There is an asymmetry to key exchanges that leaks less by first establishing a secure channel and only then authenticating.  We might reasonably break this asymmetry by deciding that specific roles require more privacy.  We might for example help protect validator operators or improve censorship resistance in some cases, like fishermen. 

Second, there is cipher suit agility in secio, at minimum in their use of multihash, but maybe even in the underlying key exchange itself.  We've seen numerous attacks on TLS <= 1.2 due to cipher suit agility, especially the downgrade attacks.  I therefore strongly recommend using TLS 1.3 *if* cipher suit agility is required.  We could likely version the entire protocol though, thus avoiding any cipher suit agility.  In any case, constructs like multihash should be considered hazardous in even basic key exchanges, but certainly in more advanced protocols involving signatures or associated data.

Third, there are [no ACKs in secio](https://github.com/libp2p/go-libp2p-secio/issues/12) which might yield interesting attacks when depending upon the underlying insecure transport's own ACKs.  ([related](https://github.com/OpenBazaar/openbazaar-go/issues/483))

As QUIC uses UDP only, we could add TCP based transport that uses TLS 1.3, perhaps by extending libp2p's existing transport with support for TLS 1.3, or perhaps adding a more flexible TLS 1.3 layer.  We might prefer a flexible TLS 1.3 layer over conventional TLS integration into libp2p extending transports because our authentication privacy demands might work differently from TLS's server oriented model.  

We could identify some reasonable [Noise](https://noiseprotocol.org/noise.html) variant, if avoiding the complexity of TLS sounds like a priority.  I believe Noise XX fits the blockchain context well, due to Alice and Bob roles being easily reversible, improved modularity, and more asynchronous key certification from on-chain data.  At the extreme, we could imagine identifing particular handshakes for particular interactions though, like GRANDPA using KK and fishermen using NK.

In short, our two simplest routes consist of replacing secio with either TLS 1.3 or Noise XX.
