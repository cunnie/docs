Create your key using the 25519 curve — it's cool, and it's short:
```
gpg --expert --full-gen-key
```

Enter the following at each of the prompts:
```
  9 # ECC and ECC
  1 # 25519
  40y # 40 years
```

Integrate gpg with GitHub: <https://help.github.com/articles/signing-commits-with-gpg/>

List your keys
```
gpg -K --keyid-format long --with-colons --with-fingerprint
```

Export the one you want (look next to `sec:u:256:22`)
```
gpg --export -a 93B3BB2C9F7F5BA4
```

Export a public key in a human-readable format:
```
gpg --output bcunnie.gpg --export --armor bcunnie@vmware.com
```

Configure git to use your key:
```
git config --global user.signingkey 93B3BB2C9F7F5BA4
git config --global commit.gpgsign true
```

To fix `error: gpg failed to sign the data` when committing
```
export GPG_TTY=$(tty)
```

To delete a key:
```
gpg --list-keys
...
pub   ed25519 2018-10-11 [SC] [expires: 2024-10-09]
      FB79E69D115334D081A56A032BBE9A01CFC8E752
uid           [ unknown] Brian Cunnie <bcunnie@pivotal.io>
uid           [ unknown] Rowan Jacobs <rojacobs@pivotal.io>
sub   cv25519 2018-10-11 [E] [expires: 2024-10-09]

gpg --delete-secret-keys  FB79E69D115334D081A56A032BBE9A01CFC8E752
  # MANY confirmations
gpg --delete-key FB79E69D115334D081A56A032BBE9A01CFC8E752
```

To edit a key:

```
gpg --list-keys
...
pub   ed25519 2018-07-17 [SC] [expires: 2024-07-15]
      08A8C8E8C9A1D3350413FA2E93B3BB2C9F7F5BA4
uid           [ultimate] Brian Cunnie (Work email) <bcunnie@pivotal.io>
uid           [ultimate] Brian Cunnie <brian.cunnie@gmail.com>
sub   cv25519 2018-07-17 [E] [expires: 2024-07-15]

gpg --edit-key 08A8C8E8C9A1D3350413FA2E93B3BB2C9F7F5BA4
  uid
    sec  ed25519/93B3BB2C9F7F5BA4
	 created: 2018-07-17  expires: 2024-07-15  usage: SC
	 trust: ultimate      validity: ultimate
    ssb  cv25519/B56458A7AE187951
	 created: 2018-07-17  expires: 2024-07-15  usage: E
    [ultimate] (1). Brian Cunnie (Work email) <bcunnie@pivotal.io>
    [ultimate] (2)  Brian Cunnie <brian.cunnie@gmail.com>
  uid 1
  deluid
  adduid
    Real name: Brian Cunnie
    Email address: bcunnie@vmware.com
  save
```
