# Grin - Basic Wallet

## Wallet Files

A Grin wallet maintains its state in an LMDB database, with the master seed stored in a separate file.
When creating a new wallet, the file structure should be:

```sh
~/[Wallet Directory]
   -wallet_data/
      -db/
         -/lmdb
      wallet.seed
   grin-wallet.toml
   grin-wallet.log
```

* `grin-wallet.toml` contains configuration information for the wallet. You can modify values within
  to change ports, the address of your grin node, or logging values.

* `wallet_data/wallet.seed` is your master seed file. You must back this file up somewhere in order to
  be able to recover or restore your wallet (along with its password, if given).

### Data Directory

By default grin will create all wallet files in the hidden directory `.grin` under your home directory (i.e. `~/.grin`).
You can also create and use a wallet with data files in the current directory, as explained in the `grin wallet init`
command below.

#### Logging + Output

Logging configuration for the wallet is read from `grin-wallet.toml`.

#### Switches common to all wallet commands

### Wallet Account

The wallet supports multiple accounts. To set the active account for a wallet command, use the '-a' switch, e.g:

```
[host]$ grin wallet -a account_1 info
```

All output creation, transaction building, and querying is done against a particular account in the wallet.
If the '-a' switch is not provided for a command, the account named 'default' is used.

##### Grin Node Address

The wallet generally needs to talk to a running grin node in order to remain up-to-date and verify its contents. By default, the wallet
tries to contact a node at `127.0.0.1:13413`. To change this, modify the value in the wallet's `grin_wallet.toml` file. Alternatively,
you can provide the `-r` (seRver) switch to the wallet command, e.g.:

```sh
[host]$ grin wallet -a "http://192.168.0.2:1341" info
```

If commands that need to update from a grin node can't find one, they will generally inform you that the node couldn't be reached
and the results verified against the latest chain information.

##### Password

All keys generated by your wallet are combinations of the master seed + a password. If no password is provided, it's assumed this
password is blank. If you do provide a password, all operations will use the seed+password and you will need the password to view or
spend any generated outputs. The password is specified with `-p`

```sh
[host]$ grin wallet -p mypassword info
```

## Basic Wallet Commands

`grin wallet --help` will display usage info and all flags.
`grin wallet help [command]` will display flags specific to the command, e.g `grin wallet help listen`

### init

Before using a wallet a new `grin-wallet.toml` configuration file, seed file `wallet.seed` and storage database need
to be generated via the init command as follows:

By default this will place your wallet files into `~/.grin`. It is VERY IMPORTANT that you back up the `~/.grin/wallet_data/wallet.seed`
file somewhere safe and private, and ensure you somehow remember the password used to generate the wallet.

Alternatively, if you'd like to run a wallet in a directory other than the default, you can run:

```sh
[host]$ grin wallet [-p password] init -h
```

This will create a `grin-wallet.toml` file in the current directory configured to use the data files in the current directory,
as well as all needed data files. When running any `grin wallet` command, grin will check the current directory to see if
a `grin-wallet.toml` file exists. If not it will use the default in `~/.grin`

### account

To create a new account, use the 'grin wallet account' command with the argument '-c', e.g.:

```
[host]$ grin wallet account -c my_account
```

This will create a new account called 'my_account'. To use this account in subsequent commands, provide the '-a' flag to
all wallet commands:

```
[host]$ grin wallet -a my_account info
```

To display a list of created accounts in the wallet, use the 'account' command with no flags:

```
[host]$ grin wallet -a my_account info
```

### info

A summary of the wallet's contents can be retrieved from the wallet using the `info` command. Note that the `Total` sum may appear
inflated if you have a lot of unconfirmed outputs in your wallet (especially ones where a transaction is initiated by other parties
who then never it by posting to the chain). `Currently Spendable` is the most accurate field to look at here.

```sh
____ Wallet Summary Info - Account 'default' as of 49 ____

 Total                            | 3000.000000000
 Awaiting Confirmation            | 60.000000000
 Immature Coinbase                | 180.000000000
 Currently Spendable              | 2760.000000000
 ---------                        | ---------
 (Locked by previous transaction) | 0.000000000

```

### listen

This opens a listener on the specified port, which will listen for:

* Coinbase Transaction from a mining server
* Transactions initiated by other parties

By default the `listen` commands runs in a manner that only allows access from the local machine. To open this port up
to other machines, use the `-e` switch:

```sh
[host]$ grin wallet -e listen
```

To change the port on which the wallet is listening, either configure `grin-wallet.toml` or use the `-l` flag, e.g:

```sh
[host]$ grin wallet -l 14000 listen
```

The wallet will listen for requests until the process is cancelled with `<Ctrl-C>`. Note that external ports/firewalls need to be configured
properly if you're expecting requests from outside your local network (well out of the scope of this document).

### send

This builds a transaction interactively with another running wallet, then posts the final transaction to the chain. As the name suggests,
this is how you send Grins to another party.

The most important fields here are the destination (`-d`)  and the amount itself. To send an amount to another listening wallet:

```sh
[host]$ grin wallet send -d "http://192.168.0.10:13415" 60.00
```

This will create a transaction with the other wallet listening at 192.168.0.10, port 13415 which credits the other wallet 60 grins
while debiting the 60 Grin + fees from your wallet.

It's important to understand exactly what happens during a send command, so at a very basic level the `send` interaction goes as follows:

1) Your wallet selects a number of unspent inputs from your wallet, enough to cover the 60 grins + fees.
2) Your wallet locks these inputs so as not to select them for other transactions, and creates a change output in your wallet for the difference.
3) Your wallet adds these inputs and outputs to a transaction, and sends the transaction to the recipient.
4) The recipient adds their output for 60 grins to the transaction, and returns it to the sender.
5) The sender completes signing of the transaction.
6) The sender posts the transaction to the chain.

Outputs in your wallet will appear as unconfirmed or locked until the transaction hits the chain and is mined and validated.

Other flags here are:

* `-m` 'Method', which can be 'http' or 'file'. In the first case, the transaction will be sent to the IP address which follows the `-d` flag. In the second case, Grin wallet will generate a partial transaction file under the file name specified in the `-d` flag. This file needs to be signed by the recipient using the `grin wallet receive -i filename` command and finalize by the sender using the `grin wallet finalize -i filename.response` command. To create a partial transaction file, use:

 ```sh
[host]$ grin wallet send -d "transaction" -m file 60.00
```

* `-s` 'Selection strategy', which can be 'all' or 'smallest'. Since it's advantageous for outputs to be removed from the Grin chain,
  the default strategy for selecting inputs in Step 1 above is to use as many outputs as possible to consolidate your balance into a
  couple of outputs. This also drastically reduces your wallet size, so everyone wins. The downside is that the entire contents of
  your wallet remains locked until the transaction is  mined validated on the chain. To instead only select just enough inputs to
  cover the amount you want to send + fees, use:

 ```sh
[host]$ grin wallet send -d "http://192.168.0.10:13415" -s smallest 60.00
```

* `-f` 'Fluff' Grin uses a protocol called 'Dandelion' which bounces your transaction directly through several listening nodes in a
  'Stem Phase' before randomly 'Fluffing', i.e. broadcasting it to the entire network. This reduces traceability at the cost of lengthening
  the time before your transaction appears on the chain. To ignore the stem phase and broadcast immediately:

 ```sh
[host]$ grin wallet send -f -d "http://192.168.0.10:13415" 60.00
```

### outputs

Simply displays all the the outputs in your wallet: e.g:

```sh
[host]$ grin wallet outputs
Wallet Outputs - Account 'default' - Block Height: 49                                                                                                               
------------------------------------------------------------------------------------------------------------------------------------------------
 Key Id                Child Key Index  Block Height  Locked Until  Status       Is Coinbase?  Num. of Confirmations  Value         Transaction
================================================================================================================================================
 13aea76c742ec6298360  2                1             4             Unspent      true          49                     60.000000000  37
------------------------------------------------------------------------------------------------------------------------------------------------
 ef619c4cdda170f9a4eb  3                2             5             Unspent      true          48                     60.000000000  38
------------------------------------------------------------------------------------------------------------------------------------------------
 be5a6f68db3ff4b88786  4                3             6             Unspent      true          47                     60.000000000  1
------------------------------------------------------------------------------------------------------------------------------------------------
 753a4086bf73246f8206  5                4             7             Unspent      true          46                     60.000000000  2
------------------------------------------------------------------------------------------------------------------------------------------------
 b2bf4c3e64a67158989f  6                5             8             Unspent      true          45                     60.000000000  4
------------------------------------------------------------------------------------------------------------------------------------------------
 db427d890fe59824ee64  7                6             9             Unspent      true          44                     60.000000000  11
```

Spent outputs are not shown by default. To show them, provide the `-s` flag.

 ```sh
[host]$ grin wallet -s outputs
```

### txs

Every time an operation is performed in your wallet (receive coinbase, send, receive), an entry is added to an internal transaction log
containing vital information about the transaction. Because the Mimblewimble chain contains no identifying information whatsoever,
this transaction log is necessary in order to allow your wallet to keep track of what was sent and received. To view the contents of the
transaction log, use the `txs`

```sh
[host]$ grin wallet txs
Transaction Log - Account 'default' - Block Height: 49                                                                                                                                                                                                                                                                                            
------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Id  Type                 Shared Transaction Id                 Creation Time                      Confirmed?  Confirmation Time                  Num. Inputs  Num. Outputs  Amount Credited  Amount Debited  Fee          Net Difference
==========================================================================================================================================================================================================================================
 1   Confirmed Coinbase   None                                  2018-07-20 19:46:45.658263284 UTC  true        2018-07-20 19:46:45.658264768 UTC  0            1             60.000000000     0.000000000     None         60.000000000
------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 2   Confirmed Coinbase   None                                  2018-07-20 19:46:45.658424352 UTC  true        2018-07-20 19:46:45.658425102 UTC  0            1             60.000000000     0.000000000     None         60.000000000
------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 3   Confirmed Coinbase   None                                  2018-07-20 19:46:45.658541297 UTC  true        2018-07-20 19:46:45.658542029 UTC  0            1             60.000000000     0.000000000     None         60.000000000
------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 4   Confirmed Coinbase   None                                  2018-07-20 19:46:45.658657246 UTC  true        2018-07-20 19:46:45.658657970 UTC  0            1             60.000000000     0.000000000     None         60.000000000
------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 5   Confirmed Coinbase   None                                  2018-07-20 19:46:45.658864074 UTC  true        2018-07-20 19:46:45.658864821 UTC  0            1             60.000000000     0.000000000     None         60.000000000
------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 6   Received Tx          03715cf6-f29b-4a3a-bda5-b02cba6bf0d9  2018-07-20 19:46:46.120244904 UTC  false       None                               0            1             60.000000000     0.000000000     None         60.000000000
------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
>>>>>>> master

To see the inputs/outputs associated with a particular transaction, use the `-i` switch providing the Id of the given transaction, e.g:

```sh
[host]$ grin wallet txs -i 6
Transaction Log - Account 'default' - Block Height: 49
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Id  Type         Shared Transaction Id                 Creation Time                      Confirmed?  Confirmation Time  Num. Inputs  Num. Outputs  Amount Credited  Amount Debited  Fee   Net Difference
===========================================================================================================================================================================================================
 6   Received Tx  03715cf6-f29b-4a3a-bda5-b02cba6bf0d9  2018-07-20 19:46:46.120244904 UTC  false       None               0            1             60.000000000     0.000000000     None  60.000000000
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------


Wallet Outputs - Block Height: 49
------------------------------------------------------------------------------------------------------------------------------------------------
 Key Id                Child Key Index  Block Height  Locked Until  Status       Is Coinbase?  Num. of Confirmations  Value         Transaction
================================================================================================================================================
 a7aebee71fdd78396ae6  9                5             0             Unconfirmed  false         0                      60.000000000  6
------------------------------------------------------------------------------------------------------------------------------------------------

```

#### cancel

Everything before Step 6 in the send phase above  happens completely locally in the wallets' data storage and separately from the chain.
Since it's very easy for a sender, (through error or malice,) to fail to post a transaction to the chain, it's very possible for the contents
of a wallet to become locked, with all outputs unable to be selected because the wallet is waiting for a transaction that will never hit
the chain to complete. For example, in the output from `grin wallet txs -i 6` above, the transaction is showing as `confirmed == false`
meaning the wallet has not seen any of the associated outputs on the chain. If it's evident that this transaction will never be posted, locked
outputs can be unlocked and associate unconfirmed outputs removed with the `cancel` command.

Running against the data above:

```sh
[host]$ grin wallet cancel -i 6
[host]$ grin wallet txs -i 6
Transaction Log - Account 'default' - Block Height: 49
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Id  Type                     Shared Transaction Id                 Creation Time                      Confirmed?  Confirmation Time  Num. Inputs  Num. Outputs  Amount Credited  Amount Debited  Fee   Net Difference
=======================================================================================================================================================================================================================
 6   Received Tx - Cancelled  03715cf6-f29b-4a3a-bda5-b02cba6bf0d9  2018-07-20 19:46:46.120244904 UTC  false       None               0            1             60.000000000     0.000000000     None  60.000000000
-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

```

Note that the Receive transaction has been cancelled, and the corresponding output was removed from the wallet. If I were the sender, my change
output would have been deleted, and any outputs that were locked for the transaction would again be available for use in another transaction.

Be sure to use this command with caution, as there are many edge cases and possible attacks that still need to be dealt with, particularly if you're
the recipient of a transaction. For the time being please be 100% certain that the relevant transaction is never, ever going to be posted before
running `grin wallet cancel`

##### repost

If you're the sender of a posted transaction that doesn't confirm on the chain (due to a fork or full transaction pool), you can repost the copy of it that grin automatically stores in your wallet data whenever a transaction is finalized. This doesn't need to communicate with the recipient again, it just re-posts a transaction created during a previous `send` attempt.  

To do this, look up the transaction id using the `grin wallet txs` command, and using the id (say 3 in this example,) enter:

`grin wallet repost -i 3`

This will attempt to repost the transaction to the chain. Note this won't attempt to send if the transaction is already marked as 'confirmed' within the wallet.

You can also use the `repost` command to dump the transaction in a raw json format with the `-m` (duMp) switch, e.g:

`grin wallet repost -i 3 -m tx_3.json`

This will create a file called tx_3.json containing your raw transaction data. Note that this formatting in the file isn't yet very user-readable.

##### restore

If for some reason the wallet cancel commands above don't work, or you need to restore from a backed up `wallet.seed` file and password, you can perform a full wallet restore.

To do this, generate an empty wallet somewhere with:

```sh
grin wallet init -h
```

Delete the newly generated wallet data directory and seed file:

```sh
[host@new_wallet_dir]# rm -rf wallet_data/db
[host@new_wallet_dir]# rm wallet_data/wallet.seed
```

Then copy your backed up `wallet.seed` file into the new `wallet_data` directory, ensuring it's called `wallet.seed`

```sh
[host@new_wallet_dir]# cp OLD_WALLET.seed wallet_data/wallet.seed
```

Then ensure that you're running a grin node, and makes sure nothing is attempting to mine into your wallet. Then, in the
wallet directory:

```sh
grin wallet restore
```

Note this operation can potentially take a long time. Once it's done, your wallet outputs should be restored, and you can
transact with your restored wallet as before the backup.
