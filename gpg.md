```
gpg --gen-key

# tip: first list your keys in GPG
gpg -K --keyid-format long --with-colons --with-fingerprint

# then export the one you want (look next to `fpr`)
gpg --export -a A4AA3A5BDBD40EA549CABAF9FBC07D6A97016CB3
```
