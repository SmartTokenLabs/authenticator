# Dauth, an Oauth-based login protocol

## Overview

Dauth (Delegated-key Authentication, or decentralised authentication) provides Web2 developers with a login and authentication method that is based on Ethereum and is password-less. It will use the same Oauth API formats and similar communication flow that Web2 developers are already familiar with. This method is superior to logging in through signing a message with an Ethereum key because of the following reasons.

**1. Privacy**: Users can *either* a) login **anonymously** *or* b) bind the login session with an Ethereum Address / ENS name. If the user logs in anonymously, the website does not learn the user's Ethereum address. And during the session, if the user chooses to identify themselves with Eth Address or ENS name, they can do so without changing the session or authentication token.

**2. Security**: The user's Ethereum key is not directly used. Instead, a delegated key is generated for each website and not shared between websites. This prevents man-in-the-middle attack, where the attacking website forwards the login challenge from another site to itself. Furthermore, by using one key per site, if delegated auth keys are compromised, its originating Ethereum key is not and safety of crypto assets are not affected.

**3. User Experience**: The user, having logged in, can decide how long the session remains active (e.g. at home, longer sessions are allowed). The user doesn't need to do anything on the UI level, even if the website session times out and requires re-authentication; the authentication happens silently. This prevents modern-day awkwardness such as "session expired, please log in again".

**4. Decentralization**: No other servers are involved when a webserver allows users to log in using ETH Address. However if ENS is used, a node is involved.

## Basic Mechanism

![Communication Flow Chart](compared_with_oauth.svg "Compare DAuth with OAuth")

When a user logs in to a website:
The user's wallet derives a delegated signing key pair at the login by deriving it from the user's Ethereum address, using the website's domain name along with some high entropy randomness, both from the website and the user, as the derivation factors[^1].

[^1]: The Ethereum address might already be the result of a BIP-32 derivation from a master key, but derived further. The derivation factor to be used is a topic of its own, which will be covered later.

The website generates a challenge which the user answers by signing. At this point, the website acknowledges a user identified by the delegated public key, and it doesn't know the user's Ethereum address. Therefore it doesn't know the user's net-worth represented by the Ethereum balance (Privacy concern).
At a later point of time during the session, the user can choose to submit his or her Ethereum address along with the necessary derivation factors, thus proving that they are the address owner. The website can verify the relationship between the delegated signing key and the user's Ethereum address.

This provides an "adaptive" identification method. For example, a user who visits an e-commerce website to buy some products would, at first, not want to be identified but still desires convenient login based on their Ethereum key. Later, when the user starts to be interested in the community discussion, they can submit their Ethereum address, and prove, through a personal Ethereum signature, that they have always been the owner of that Ethereum address and that the delegated signing keys were constructed based on this, for exactly this website.

Through such a mechanism, the role of each key is fixed. Therefore, an Ethereum key will be solely for linking a delegated key to an Ethereum address retroactively, not for authentication against a website. Thus reducing the attack surface on the user's Ethereum signing key.

This also prevents a man-in-the-middle attack where a website asks a user to authenticate while itself act as the user on another website. This is because each website will have its own delegated key.

Since the auth key stays with the browser, the browser can determine how safe the session is. For example, a mobile phone constantly being used by a user will keep any session lasting. If any login session expires, the webserver will require re-authentication through a challenge. Knowing the user is active, the browser can silently sign that challenge with the delegated auth key without bothering the user.

## Specification

Assume the user's Ethereum key of their address is denoted by as `sk` and hence that its corresponding public key is `pk=G*sk`. That is, `G` is the generator for secp256k1 ECDSA. We assume the address of this public key is computed through a function `addr`, i.e. the user's address is `addr(pk)`.

### User trying to log in for the first time

The first time the user wants to log in to a website with the domain `domain`, derive a delegated Ethereum key using `domain` as the derivation factor based on their private Ethereum key `sk`. This can for example be done using BIP-32. Denote the private part of this delegated key by `r`, denote the public part of they key by `R=G*r`. This public key will uniquely identify the user across sessions and devices.

Using `r` the user derives yet another key pair, which will be the *refresh key pair*.
To do so they receive a challenge `d` from the website and derive a random number `e=Keccak(r)`. 
 They then compute the private *refresh key* as `sk_r=r+Keccak(addr(pk),domain,d,e)` and a corresponding public *refresh key* `pk_r=G*(r+Keccak(addr(pk),domain,d,e))`.

The user then signs the public delegated key, the public *refresh key* and the server challenge with both the private delegated key and the private *refresh key*. That is, the user computes `a=sign(r, (R,pk_r, c))` and `b=sign(sk_r, (R,pk_r, c))`. Finally the user sends `a,b,R,pk_r` to the website.
The website verifies the signatures and creates an account linked to `R` with a *refresh key* `pk_r` and furthermore stores `d`.
The user stores `R,e,sk_r,pk_r`.

Note that now the secret key `r` can be deleted from memory, since it will only be required again when a new refresh key needs to be issued or old refresh keys need to be revoked. Both which are actions that require the user to access their wallet and hence their Ethereum account key, and thus they can just generate `r` again when needed.

### User logs in to a website
The user wants to log in to a website with domain `domain` for which it has previously constructed a *refresh key pair*.
In this case the user simply asks the website for a new unique challenge, `c`, and signs this using `sk_r`. That is, `a=sign(sk_r, c)`. The user returns `a` to the website, which then verifies the signature on this with `pk_r` and that the challenge `c` is as expected.  

### User shares ENS 
If the user wants to share their Ethereum identity with the website, they again ask the website for a new unique challenge `c`. The user then computes a personal signature on this challenge and the value `R` using its private Ethereum key, that is `f=sign(sk, (c, R))`. The user then shares `e`, `f` and `addr(pk)` with the website. The site then derives the user's candidate public key from `f` and checks that `pk=R+G*Keccak(addr(pk),domain,d,e)`. If so, it accepts the delegated key `R` is associated with the user's true Ethereum address `addr(pk)`.

Note that instead of `addr(pk)` the user's ENS can simply be used instead.

### User revokes refresh key
If a user's private *refresh key* has been compromised they issue a request to revoke _all_ previous *refresh keys* using their delegated Ethereum key. 
Concretely the user reconstructs the private delegated key `r` using `domain` and their private Ethereum key `sk`. Then they simply sign a message containing a command `revoke`, a time stamp and the public delegated key `R`, using `r` and shares this with the webserver. That is, they share the signature `sign(r, (revoke, timestamp, R))`.
The webserver will then discard any `pk_r` it has associated with `R`. Afterwards the user can construct a new *refresh key* pair.

Note that the reason _all_ previous refresh keys are revoked is because we don't assume any way of synchronising the different refresh keys (and the random numbers used to construct them) between different devices.

## Security observation
The crux of the specification is the following:

1. The user both proves ownership of the *refresh key* by signing the challenge `c` provided by the website.

2. The third party site cannot brute-force the user's address due to the unpredictability of the value `e`.

3. The *refresh key* has been linked to their Ethereum address from the get-go. The latter is achieved since the user "commits" to the random part of its site-specific key by disclosing the value `R`, and the rest of the *refresh key*, `Keccak(addr(pk),domain,d,e)` is uniquely determined in a way that is publicly disclosable but also unpredictable (before disclosure).

## Further work
