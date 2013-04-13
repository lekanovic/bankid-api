# BankID API documentation

BankID is a standard authorization protocol used in Norway. It allows users to
use the same password and two-factor authentication to log in to different
merchants.

This is an attempt at documenting the (proprietary) protocol.

## General overview

BankID is used when a web site (**merchant**) needs to securely identify its
**users**. Users will have to enter their social security number
("fødselsnummer"), a password, and a one-time password (either generated on a
custom device or sent to their phone number). A **central server** handles the
actual authentication, allowing the user to use the same authentication method
across distinct merchants.

A BankID **client** (e.g. the Java applet) communicates with the merchant and
the central server over HTTPS. Requests are sent as HTTP POST using standard
form-data encoding. Binary data is encoded using base64. In addtion to HTTPS,
every request is encrypted using AES-256-CBC and RSA-2048. The client
includes a set of known public keys belonging to the central server; the
merchant's public key is requested from the central server.

The Java applet is instantiated with *parameters* generated by the merchant.
These parameters includes (but is not limited to): URLs to the central- and merchant's
server, valid IPs for domains (to prevent MITM attacks), timeout variables,
languge settings, authentication type. The parameters are signed by the
merchant. The client itself doesn't validate the signature, but rather just
passes it on to the central server.

Every HTTP request consists of two "levels": an envelope that contains a RPC
request. Example:

```ruby
# GetMerchantKey-request:
req = {
  "bh" => "GetMerchantKey",
  "ao" => "java",
  "tf" => "5.3.2",
  "xx" => "3.7",
  "ij" => "...",
  "dg" => "...",
  "df" => "...",
}

# Encode as form-data
data = encode(req)

# Envelope
env = {
  "ao" => "java",
  "tf" => "5.3.2",
  "bu" => "3.7",
  "bt" => "...",
  "edo" => base64(encrypt_data(data)),
  "eko" => base64(encrypt_key(key)),
  "eao" => base64(checksum(key, data)),
}

# This is sent as HTTP POST
body = encode(env)
```

Yes, the protocol full of 2-letter key names. And the keys are different when
talking to the merchant's server (as opposed to the central server).

## Crypto

For every request there's generated a new **secret key** (32 bytes). This
secret key is never directly used, but derivations are created using
HMAC-SHA256.

Given `HMAC_SHA256(key, message)`, the derived keys can be computed as:

```
request_encryption_key =       HMAC_SHA256(secret_key, 'requestEncryptionKey')
request_iv = first 16 bytes of HMAC_SHA256(secret_key, 'requestInitVector')
request_auth_key =             HMAC_SHA256(secret_key, 'requestAuthenticationKey')

response_encryption_key =       HMAC_SHA256(secret_key, 'responseEncryptionKey')
response_iv = first 16 bytes of HMAC_SHA256(secret_key, 'responseInitVector')
response_auth_key =             HMAC_SHA256(secret_key, 'responseAuthenticationKey')
```

### Public keys

The Java applet includes five pairs of keys (included in this repo as
`keys/1a.pub` to `keys/5b.pub`). I haven't quite figured out what the different
keys mean.

### Secret key encryption

The secret key is encrypted using the server's public key (RSA). At the moment
it seems that it uses `keys/5a.pub`.

### Data encryption

General data is encrypted using AES-256-CBC. See the derived keys above for the
key and initialization vector.


