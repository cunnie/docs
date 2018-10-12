First step is to [create a key for your project](https://github.com/cunnie/docs/blob/master/gpg.md). The initial email address
and name can belong to any developer on your project.

To add new users to the key, run
```
gpg --edit-key $KEY_ID
```
where `$KEY_ID` is something like `2BBE9A01CFC8E752`.

For each additional developer, run `adduid` and enter their name and Pivotal email address.
You should also run `trust` and set some trust level greater than 2.

Every developer should add this key to their GPG keys in GitHub.

Then import the key to each development workstation. On the workstation containing the key, run
```
gpg --export-secret-keys $KEY_ID > project-key.asc
```

Then use `scp` or similar to transfer this file to each workstation. From each new workstation, run
```
gpg --import project-key.asc
git config --global user.signingkey $KEY_ID
git config --global commit.gpgsign true
```

### References

- Exporting and importing a key: <https://makandracards.com/makandra/37763-gpg-extract-private-key-and-import-on-different-machine>
- Adding a second user to a key: <https://www.katescomment.com/how-to-add-additional-email-addresses-to-your-gpg-identity/>
