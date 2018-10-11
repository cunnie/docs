Create your key using the 25519 curve — it's cool, and it's short:
```
gpg --expert --full-gen-key
```

Enter the following at each of the prompts:
```
  9 # ECC and ECC
  1 # 25519
  6y # 6 years
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

Configure git to use your key:
```
git config --global user.signingkey 93B3BB2C9F7F5BA4
git config --global commit.gpgsign true
```

To fix `error: gpg failed to sign the data` when committing
```
export GPG_TTY=$(tty)
```
