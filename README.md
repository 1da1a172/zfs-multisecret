# ZFS Multisecret

A script that allows multiple keys to decrypt a zfs filesystem.

## Big picture

Each user key is used to encrypt the zfs key.
The encrypted zfs key (and associated metadata) is stored in zfs user
properties (see `ZFSPROPS(7)`).
Each set of user properties is called a "slot".

Each slot uses the following properties:
- `wrappedkey:slotname`
- `pbkdf2iters:slotname`
- `keylocation:slotname`

## Getting started

A `zfsms` filesystem is created by converting an existing encrypted ZFS
filesystem.

```bash
truncate test.zpool --size 1G
mkdir ztest
zpool create \
    -R $PWD/ztest \
    -O encryption=on \
    -O keyformat=passphrase \
    ztest \
    $PWD/test.zpool \
    <<< some_initial_passphrase
zfsms convert ztest <<< a_passphrase_for_the_default_slot
```

|-|-|
|**NOTE**| After converting a filesystem, the old key will no longer work. |
```
# zfs load-key -n ztest <<< some_initial_passphrase
Key load error: Hex key too short (expected 64).
0 / 1 key(s) successfully verified
```

Now, when you look at the properties for the filesystem, you will see the zfsms
properties mentioned above:
```
$ zfs get all ztest | grep :default
ztest  keylocation:default   prompt                                                            local
ztest  pbkdf2iters:default   350000                                                            local
ztest  wrappedkey:default    U2FsdGVkX1/Gw5y4hja2mmxoLUQNxvOjuV97VqpeaW8CPGW6Evfmoks6A1FgkQ1N  local
```

Add a second slot with the `add-key` subcommand:
```
$ sudo zfsms add-key -s slotdeux -u default ztest
Getting the user key for slot "default"
Enter passphrase:
Enter passphrase:
$ zfs get all ztest | grep :slotdeux
ztest  wrappedkey:slotdeux   U2FsdGVkX19p2J0GEDgDbB8KkKOfB/NlV34g9n/btHCo1Q0pJFtfFU18gyrfjNIz  local
ztest  pbkdf2iters:slotdeux  350000                                                            local
ztest  keylocation:slotdeux  prompt                                                            local
$ zfs get all ztest | grep :default
ztest  keylocation:default   prompt                                                            local
ztest  pbkdf2iters:default   350000                                                            local
ztest  wrappedkey:default    U2FsdGVkX1/Gw5y4hja2mmxoLUQNxvOjuV97VqpeaW8CPGW6Evfmoks6A1FgkQ1N  local
```

Note that the first prompt is for the `default` slot, to unwrap the zfs key, and
the second prompt is to set the passphrase for the new slot.
The prompts are not good and need some work.

To see all subcommands, use `zfsms help`.
To see help for a subcommand, use `zfsms help <subcommand>`.

## Keys and locations

Due to limitations of shell scripting, all secrets should be a single line of
ASCII printable characters.
The null (`0x00`) and new line (`0x0a`) characters in particular are known
to be problematic, and may result in undefined behavior.

The key location can be `prompt` or a URI.

### `prompt`

If `stdin` is a terminal, the script will interactively prompt for the
passphrase.
Otherwise, the key will be read from stdin automatically.

### URI

The secret can also be read from a file.
The key location must start with `file://`, `https://`, or `http://`.

## security notes

### Access to zfs key

If a user knows a secret for one of the slots, they can trivially access the zfs
secret, which can be used directly, bypassing zfsms.

```
$ zfs get wrappedkey:default -Ho value ztest \
    | base64 -d \
    | openssl aes-256-ctr -d \
        -iter $(zfs get pbkdf2iters:default) \
        -pass pass:a_passphrase_for_the_default_slot \
    | xxd -p -c 0 \
2277a268d0d286f6c0d47157cf981c4adcb6665125811dbf953f15159ad56638
$ !! | sudo zfs load-key -n ztest
1 / 1 key(s) successfully verified
```

As such, this should be considered a tool for convenience and ease of
management, not for additional security, as meaningfully revoking access is
difficult.

### aes-256-ctr

`zfsms` uses aes-256-ctr, where zfs's builtin encryption uses aes-256-gcm.
This means the cleartext of the zfs key is not verified until after it has been
decrypted, and in some cases, attempted to be used.

An attacker that can modify the zfs user properties could alter the secret, but
it would become unusable, and the attacker would not know what it was changed
to.

The current situation is not ideal, but is unlikely to pose an actual threat.

### Key rotation

Be aware of the limitations of changing a zfs key, even without zfsms.
See `ZFS-CHANGE-KEY(8)`'s description of the `change-key` subcommand for
details.

Also note that using `zfs change-key` instead of `zfsms change-key` will break
the multisecret scheme, as the zfs key will have been changed out from
underneath it.
The user properties will still exist, but the slots will no longer work.

## Testing harness

The script `zfsmstest` can be used to test the functionality of the `zfsms`
script.
It covers most of the code, so it can be leaned on to verify changes.
It also makes for a good reference on how to use `zfsms`.
