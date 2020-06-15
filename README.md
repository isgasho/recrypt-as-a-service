# Use Recrypt-as-a-Service for Cryptographic Orthogonal Access Control

<p>
    <a href="https://online.visualstudio.com/environments/new?name=Recrypt%20as%20a%20Service&repo=second-state/recrypt-as-a-service">
        <img src="https://img.shields.io/endpoint?style=social&url=https%3A%2F%2Faka.ms%2Fvso-badge">
    </a>
</p>

This is a [Mozilla Open Labs project](https://builders.mozilla.community/springlab/index.html).

[Cryptographic Orthogonal Access Control](https://dl.acm.org/doi/10.1145/3201595.3201602) is a scalable approach for individuals to control who can access their data *without* shared secrets or storing secrets (e.g., private keys) on a centralized service. Applications like Dropbox and Google Drive have shown that centralized repositories and permission management systems could dramatically improve the user experience for file sharing. Yet, storing plain text documents and/or encryption keys on those centralized services have profound privacy implications.

For example, in the age of COVID-19, there is great need for people to share their medical records remotely. Cryptographic Orthogonal Access Control empowers people to share with confidence that their privacy will be protected. While there are central services and repositories to faciliate sharing, it is mathematically impossible for those services to violate your privacy. As a user, you alone decides who and when sees your data.

Read more about this [scalable privacy computing service](https://hackernoon.com/second-state-releases-scalable-privacy-service-at-mozilla-open-labs-b15u3wh7).

## Requirements

1 All parties, including individuals, doctors, hospitals, employers etc., can create public identities on the service.

2 An individual, Alice, can grant access to her data to a list of other identities in the service. Let's say that Bob is granted access.

3 Alice can now publish encrypted data files. She can upload the encrypted files to any public service.

4 Only the identities that are granted access in #2 (e.g., Bob) can generate decryption keys to decrypt and read those files.

5 Alice can grant access to more identities, say Charlie, at any time. Charlie, like Bob, will have access to all encrypted files from Alice.

6 Alice can revoke access to any identity. Let's say Bob's access s revoked. Bob will not be able to generate decryption keys for any Alice's files.

> It is still possible for Bob to retain some decryption keys before his access was revoked. It is best for Alice to use a new encryption key for each file.

> Traditional PKI requires Alice to create a different key for each person on her access list. It is a huge amount of work for Alice if she has a million files and a million people on the list. The proxy re-encryption scheme only requires Alice to encrypt and upload her data files once. Bob and Charlie can create their own decryption keys as long as their access is not revoked.

## Use case scenario

Alice creates an identity on the service via a `create_identity` request. She needs to save the returned UUID and private key values.

```
$ curl http://127.0.0.1:3000/create_identity
{"uuid":"f9d2ec5a-5f48-4e1b-a0b3-d52b0e0f521f","pk":[8,209,114,217,157,178,185,53,154,42,198,226,108,120,241,24,27,22,244,136,57,40,145,224,234,97,138,58,70,33,116,21]}

# {"uuid":"<Alice UUID>", "pk":"<Alice private key>"}
```

Bob, Charlie, and others each creates an identity on the service via `create_identity` requests. They need to save the returned ID and private key values. Here is Bob's UUID and private key.

```
$ curl http://127.0.0.1:3000/create_identity
{"uuid":"e7a2fc4c-7220-4617-8721-4e67120c63a9","pk":[5,251,27,236,155,82,189,136,54,48,225,95,30,66,96,94,121,155,63,6,207,72,79,9,168,119,2,23,106,36,46,200]}

# {"uuid":"<Bob UUID>", "pk":"<Bob private key>"}
```

Now, Alice can grant Bob access to all her data via a `grant_access` request. The request URL takes two parameters. The first is Alice's UUID and the second is Bob's UUID. The request body must be Alice's private key since she is authorizing access.

```
# curl -d "<Alice private key>" -H 'Content-Type: application/json' -X POST http://127.0.0.1:3000/grant_access/<Alice UUID>/<Bob UUID>
# { ... "signature":"<access signature>" ... }

$ curl -d "[8,209,114,217,157,178,185,53,154,42,198,226,108,120,241,24,27,22,244,136,57,40,145,224,234,97,138,58,70,33,116,21]" -H 'Content-Type: application/json' -X POST http://127.0.0.1:3000/grant_access/f9d2ec5a-5f48-4e1b-a0b3-d52b0e0f521f/e7a2fc4c-7220-4617-8721-4e67120c63a9
{...,"signature":[27,157,189,125,255,133,207,85,122,65,1,4,213,170,221,62,32,34,183,33,83,185,149,84,81,27,219,244,206,31,134,236,177,209,53,190,229,89,31,229,141,198,64,109,81,70,149,229,94,117,49,73,187,220,148,9,135,205,225,107,88,148,37,10]}
```

When Alice creates a confidential document, she can create a new AES encryption key to encrypt it. She generates the AES key via a `create_sym_key` request. Alice's UUID is in the URL since this AES key is generated by her and will be shared to people Alice granted access to. Alice notes the key's UUID, and publishes the UUID with her encrypted document.

```
# curl http://127.0.0.1:3000/create_sym_key/<Alice UUID>
# {"uuid":"<Key UUID>","sk":"<AES key in HEX>"}

$ curl http://127.0.0.1:3000/create_sym_key/f9d2ec5a-5f48-4e1b-a0b3-d52b0e0f521f
{"uuid":"9ebc843e-c82a-4c72-88b5-d8401f640cc7","sk":"83177a48b4c97ed4b0d8f0fd0f34f549b4d2987582d59397e67294566c481ad2"}
```

When Bob wants to decrypt the document, he asks for Alice's AES key via a `get_sym_key` request. The URL has Bob's UUID, Alice's UUID, and the AES key's UUID. The request body is Bob's private key to prove this is really Bob. The return value is the AES key that can decrypt Alice's documents.

```
# curl -d "<Bob private key>" -H 'Content-Type: application/json' -X POST http://127.0.0.1:3000/get_sym_key/<Bob UUID>/<Alice UUID>/<Key UUID>
# <AES key in HEX>

$ curl -d "[5,251,27,236,155,82,189,136,54,48,225,95,30,66,96,94,121,155,63,6,207,72,79,9,168,119,2,23,106,36,46,200]" -H 'Content-Type: application/json' -X POST http://127.0.0.1:3000/get_sym_key/e7a2fc4c-7220-4617-8721-4e67120c63a9/f9d2ec5a-5f48-4e1b-a0b3-d52b0e0f521f/9ebc843e-c82a-4c72-88b5-d8401f640cc7
83177a48b4c97ed4b0d8f0fd0f34f549b4d2987582d59397e67294566c481ad2
```

Now Bob received Alice's AES key and decrypted her documents. Alice can create more documents and more AES keys. She shares the key UUID to everyone she `grant_access` to and they will all be able to retrieve the key themselves. If Alice does not `grant_access` to Charlie, the above `get_sym_key` operaton with Charlie's UUID and private key will fail. 

Finally, Alice can revoke access for Bob. She does so by posting the `signature` value returned from `grant_access` to a `revoke_access` request.

```
# curl -d "<access signature>" -H 'Content-Type: application/json' -X POST http://127.0.0.1:3000/revoke_access/<Alice UUID>/<Bob UUID>

$ curl -d "[27,157,189,125,255,133,207,85,122,65,1,4,213,170,221,62,32,34,183,33,83,185,149,84,81,27,219,244,206,31,134,236,177,209,53,190,229,89,31,229,141,198,64,109,81,70,149,229,94,117,49,73,187,220,148,9,135,205,225,107,88,148,37,10]" -H 'Content-Type: application/json' -X POST http://127.0.0.1:3000/revoke_access/f9d2ec5a-5f48-4e1b-a0b3-d52b0e0f521f/e7a2fc4c-7220-4617-8721-4e67120c63a9
[Success]
```

Bob cannot get any of Alice's published AES key anymore.

```
# curl -d "<Bob private key>" -H 'Content-Type: application/json' -X POST http://127.0.0.1:3000/get_sym_key/<Bob UUID>/<Alice UUID>/<Key UUID>

$ curl -d "[5,251,27,236,155,82,189,136,54,48,225,95,30,66,96,94,121,155,63,6,207,72,79,9,168,119,2,23,106,36,46,200]" -H 'Content-Type: application/json' -X POST http://127.0.0.1:3000/get_sym_key/e7a2fc4c-7220-4617-8721-4e67120c63a9/f9d2ec5a-5f48-4e1b-a0b3-d52b0e0f521f/9ebc843e-c82a-4c72-88b5-d8401f640cc7
[Error]
```

## Building and running the service

Our service incorporates [Ironcore Labs'](https://ironcorelabs.com/) Rust-based proxy re-encryption library [Recrypt.rs](https://docs.rs/recrypt/0.11.1/recrypt/).

### Prerequisite

```
$ sudo apt-get update
$ sudo apt-get -y upgrade
$ sudo apt install build-essential curl wget git vim libboost-all-dev

$ curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
$ source $HOME/.cargo/env

$ curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.35.3/install.sh | bash
$ export NVM_DIR="$HOME/.nvm"
$ [ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"
$ [ -s "$NVM_DIR/bash_completion" ] && \. "$NVM_DIR/bash_completion"

$ nvm install v10.19.0
$ nvm use v10.19.0

$ npm install -g ssvmup # Append --unsafe-perm if permission denied
$ npm install ssvm
$ npm install uuid
$ npm install express
```

### Build the application

```
$ ssvmup build
```

### Start the service

```
$ cd node
$ node app.js
```

