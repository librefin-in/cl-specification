# Common Library Specification

This is a technical specification for the NPCI/UPI Common Library. The primary role of the cryptography library (called the CL henceforth) is to encrypt secrets (OTP/PIN) for two-factor authentication before they are sent over the wire.

Note that the first factor (device binding) is in the hand of the PSP, and the common library doesn't do anything there. A summary of the CL public interface (old version) can be found at <https://www.irjet.net/archives/V4/i6/IRJET-V4I6509.pdf>.

## Overview

Two random keys are generated on the device: A refresh key, and an data encryption key (DEK). The DEK is encrypted further by a Key-Encryption-Key(KEK1), and sent across.

To perform transactions, the DEK is used to encrypt a message digest to validate the transaction to generate a integrity-check-message. This message is further encrypted by another signing key.

## Initialization

The initialization stage generates two keys. There is a `token`, and a key `K0`. These are both 256-bit keys, to be used for AES Encryption.
The keys are stored as a hex-encoded 64 character long ascii string. Once authorized (by going through a challenge process), the `K0` key will be shared and authorized for signing future transactions.

## Initial Challenge

To register a device, you need to submit the encryption keys by wrapping them in a challenge that is forwarded to the server over a `GetToken` UPI call (See Section [2.2 of the review](https://www.irjet.net/archives/V4/i6/IRJET-V4I6509.pdf))

The Challenge string is generated by joining the `token`, `K0`, and the device Identifier using pipes (`|`), encrypting the resulting string by simple RSA (using the NPCI Signing Key) and then base64 encoding the ciphertext. RSA encryption is used with `ECB` mode, and `PKCS1Padding`.

```java
msg = token // 64 characters hex ascii string
   + "|"
   + K0     // 64 characters hex ascii string
   + "|"
   + deviceID // Alphanumeric Device Identifier

challenge = Base64.encode(RSAEncrypt(msg, getSignerKey())
```

The signing certificate can be found at <https://ipfs.io/ipfs/QmPaJ4BxWgbzm99mTsjxpeodPgnYyLfEAJCWyU4SLuA985?filename=QmPaJ4BxWgbzm99mTsjxpeodPgnYyLfEAJCWyU4SLuA985>.

The response to a `GetToken` API Request is a validation checksum (Called `NPCIGetTokenResponse`). You can generate this by concatenating the `appId`, `mobileNumber`, and the `deviceId` (once again using pipe as separator), doing a sha256 hash, and then base64 encoding the result. If these two match, the device has been successfully registered. The additional parameters are sent outside of the Common-Library and that flow is dependent upon the PSP Application.

## Payment Challenge

To make payments, a special kind of challenge is required, which is generated by the CL, by specifying what kind of authentication mechanism is being used. The most common one is UPI PIN. Given a PIN `P`, and the NPCI RSA Public Key (received as a `RespListKeys` response from the server). The key hasn't changed since 2015 and can be found at <https://ipfs.io/ipfs/QmVexKvdu6ZZV3621ZriACBbwKjZTkxQ5cRShG8whAixXE?filename=key.pem>, an implementation of the CL can then generate a challenge by the following procedure:

1. Generate a `refId` as a 35 character string. The easiest way is to generate a UUID and remove the hyphens.
2. Join the amount, refId, payer, payee, appId, mobileNo, deviceId by using pipes as the separator to form the checksum message
3. Generate the `sha256` hash of the checksum message, and encrypt it directly (as raw bytes) using the `K0` key.
4. Encode the encrypted result in Base64 format.
5. Generate the challenge message by joining the PIN, refId, and result from (4) using pipe as the separator.
6. Encrypt the challenge message using the NPCI Public Key and save it as E

Return the final challenge as `2.0|E`. Or, in pseudo-code:

```java
message = amount
  + "|" + refId
  + "|" + payer
  + "|" + payee
  + "|" + appId
  + "|" + mobileNo
  + "|" + deviceId

encrypted = Base64.encode(
  AESEncrypt(sha256(message), K0)
);

return "2.0|" + Base64.encode(RSAEncrypt(PIN  + "|" + refId + "|" + encrypted, getPublicKey()));
```

## Test Vectors

TODO

## Refresh Mechanism

TODO
