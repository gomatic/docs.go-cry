---
title: go-cry
---
**Stream encryption with [age](https://age-encryption.org), keyed by SSH keys.** The `sshage` package seals a byte stream to a set of age recipients (an SSH public key becomes a recipient) and unseals it with age identities loaded from an SSH private key — every failure carries a sentinel error matchable with `errors.Is`.

- **Source:** [gomatic/go-cry](https://github.com/gomatic/go-cry)
- **API reference:** [pkg.go.dev/github.com/gomatic/go-cry](https://pkg.go.dev/github.com/gomatic/go-cry)

## Install

```sh
go get github.com/gomatic/go-cry
```

The package is named `sshage`:

```go
import sshage "github.com/gomatic/go-cry"
```

## Usage

The API is three functions and two slice types. `Encrypt` seals `r` to `w` for a set of `Recipients`; `Decrypt` reverses it with a set of `Identities`; `ParseIdentities` loads age identities from an SSH private-key file.

Recipients and identities come from age's SSH support (`filippo.io/age/agessh`): `agessh.ParseRecipient` turns an `authorized_keys` line into an `age.Recipient`, and `ParseIdentities` reads an `age.Identity` from a private-key file.

```go
package main

import (
	"bytes"
	"log"

	"filippo.io/age/agessh"
	sshage "github.com/gomatic/go-cry"
)

func main() {
	// An SSH public key (authorized_keys form) becomes an age recipient.
	rcpt, err := agessh.ParseRecipient("ssh-ed25519 AAAAC3Nza... user@host")
	if err != nil {
		log.Fatal(err)
	}

	// Seal plaintext to the recipient.
	var sealed bytes.Buffer
	plaintext := bytes.NewReader([]byte("secret data"))
	if err := sshage.Encrypt(&sealed, plaintext, sshage.Recipients{rcpt}); err != nil {
		log.Fatal(err)
	}

	// Load identities from the matching SSH private key and unseal.
	ids, err := sshage.ParseIdentities("/home/user/.ssh/id_ed25519")
	if err != nil {
		log.Fatal(err)
	}

	var opened bytes.Buffer
	if err := sshage.Decrypt(&opened, &sealed, ids); err != nil {
		log.Fatal(err)
	}
	// opened.Bytes() == []byte("secret data")
}
```

`Encrypt` accepts multiple recipients — any one of the corresponding identities can then `Decrypt` the stream. age is authenticated encryption, so a tampered, truncated, or wrong-key ciphertext fails to decrypt rather than yielding altered plaintext.

## Errors

Every failure wraps one of four sentinel `errs.Const` values from [gomatic/go-error](https://github.com/gomatic/go-error), each recoverable with `errors.Is`:

- `ErrEncrypt` — age encryption failed.
- `ErrDecrypt` — age decryption failed (wrong key, tampered, truncated, or non-age input).
- `ErrOpenFile` — the identity file could not be read.
- `ErrParseIdentity` — the SSH private key could not be parsed into an age identity.

```go
if err := sshage.Decrypt(&opened, &sealed, ids); errors.Is(err, sshage.ErrDecrypt) {
	// wrong key, tampered, or truncated ciphertext
}
```

## Design

The package is a thin, stream-oriented seam over `filippo.io/age` and `filippo.io/age/agessh`. `Recipients` (`[]age.Recipient`) and `Identities` (`[]age.Identity`) are named slice types over age's own interfaces, and `IdentityFile` names the private-key path — the caller constructs recipients and identities through age's SSH helpers, keeping this package's surface to encrypt, decrypt, and load. All work is `io.Reader`/`io.Writer` based, so nothing is buffered whole in memory. It was extracted from `gomatic/ssh-tgzx`'s `internal/crypt`.
