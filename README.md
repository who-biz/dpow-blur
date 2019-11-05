# This repository is under contruction

There are files in here from many different projects, including but not limited to BLUR, BTC, KMD, and XMR. Please retain proper licensing if you reuse any files, and be aware that this repo is under heavy development... So files will not yet be in their proper homes.

# Contents:


- <a href="https://github.com/blur-network/dpow-blur#deps">Dependencies</a>
- <a href="https://github.com/blur-network/dpow-blur#create-wallet">Create a notary wallet</a>
- <a href="https://github.com/blur-network/dpow-blur#start-wallet">Launching a notary wallet</a>


### Automatically performed operations:
- <a href="https://github.com/blur-network/dpow-blur#create-tx">Create a notarization tx</a>
- <a href="https://github.com/blur-network/dpow-blur#relay-tx">Relaying a notarization tx (request for futher signatures)</a>
- <a href="https://github.com/blur-network/dpow-blur#append-sig">Appending signatures to a pending notarization</a>


### Manually performed operations
- <a href="https://github.com/blur-network/dpow-blur#view-pending">Viewing pending notarization txs</a>
- <a href="https://github.com/blur-network/dpow-blur#rpc-calls">Other RPC calls</a>


<h2 id="deps"> Dependencies </h2>

Libhydrogen requires CMake 3.14+ to compile.  If your distribution does not include this in the package manager, you can download the latest release's source from here: <a href="https://github.com/Kitware/CMake/releases/download/v3.15.2/cmake-3.15.2.tar.gz">CMake 3.15.2 from Kitware's github</a>

Both libbtc and the native blur files require GCC 8.3 or below to compile.  GCC 9+ will error out.

Once that is extracted from the archive, building and installing is as simple as `./configure && make && sudo make install`

Ubuntu/Debian One-Liner:

`sudo apt install build-essential cmake pkg-config libssl-dev libunwind-dev libevent-dev libsodium-dev binutils-dev libboost-all-dev`

Arch Linux One-Liner: 

`sudo pacman -S base-devel cmake boost openssl libsodium libunwind binutils-devel libevent`

Fedora One-Liner: 

`sudo dnf install cmake boost-devel openssl-devel sodium-devel libunwind-devel binutils-devel libevent-devel`


<h2 id="create-wallet"> Creating a Notarization Wallet for Notarization Tx's on BLUR</h2>


*To create a wallet on BLUR's network, for notary nodes already owning a secp256k1 private key:*


**Step 1:** Compile binaries from source, then create a json-formatted wallet configuration file as follows: 
(named `btc.json` in our example)


```
{
        "version": 1,
        "filename": "test_wallet",
        "scan_from_height": 0,
        "btc_pubkey":"24e31f93eff0cc30eaf0b2389fbc591085c0e122c4d11862c1729d090106c842",
        "password":"password"
}
```

Substitute the hexidecimal representation of your node's public key into the field titled `btc_pubkey`.

**Step 2:** Launch `blurd` with the following options:


```
./blurd --testnet
```


**Step 3:** Launch `blur-notary-server-rpc` with the following options:

*Note: Change the name of the json file if you named yours something other than* `btc.json`


```
./blur-notary-server-rpc --generate-from-btc-pubkey btc.json --rpc-bind-port 12121 --testnet
```

You should see the following:

![nnwallet](https://i.imgur.com/I9cRpQp.png)


Double check that the values in red match each other! If they do not, then the wallet was generated incorrectly, or data was corrupted between generation, and a check on the values afterward. 

Your wallet should now be running.  Skip the `Starting the Notary Server Wallet` heading if you don't plan to shut down your notary node between this point and testing.

<br><br>

<h2 id="start-wallet">Starting the Notary Server Wallet</h2>

**Step 1:**  Start `blurd` with the following options:


```
./blurd --testnet
```


After your wallet is generated, you can reopen this existing file with the following startup flags:


```
./blur-notary-server-rpc --wallet-file test_wallet --rpc-bind-port 12121 --prompt-for-password --testnet --disable-rpc-login
```


The RPC interface will prompt you for the wallet password.  Enter the password you entered into `btc.json` on creation.


<br><br>


<h2 id="create-tx">Creating a Notarization Transaction</h2>


<h4>Please note: Everything after step 3 will be performed automatically (without any action on your part) after launching the notary server.</h4>


<i>What is shown below, excluding steps 1-3, is for informational purposes.  You should never need to call</i> `create_ntz_transfer` or `append_ntz_sig` <i>methods manually.</i>


**Step 1:** Before creating a transaction, you must paste the public spendkey that was generated when creating your wallet, into the 4th column in the `Notaries_elected1` table, located in `src/komodo/komodo_notaries.h`.  If your public spendkey is not located in this table, the wallet will not permit you to create a notarization tx. 

**Step 2:**  Compile source again, and start up the daemon with `./blurd --testnet`.

**Step 3:**  After the daemon is running, launch the notary wallet with `blur-notary-server-rpc --wallet-file=test_wallet.keys --rpc-bind-port 12121 --prompt-for-password --disable-rpc-login --testnet`

**Step 4:** Issue the following `curl` command in a separate terminal:


```
curl -X POST http://127.0.0.1:12121/json_rpc -d '{"method":"create_ntz_transfer"}'
```


*Notes:*

The command above will create a new notarization tx with one signature.

The first two times this command is called, the generated transactions will be added to a cache (`ntz_ptx_cache`). 

Once there are two `pending_tx`s in the cache, the third call will be relayed as a request for more signatures using `NOTIFY_REQUEST_NTZ_SIG`

Destinations for the transaction are automatically populated, using the BTC & CryptoNote pubkeys provided in `komodo_notaries.h`.  Each notary wallet is sent 0.00000001 BLUR.


**TL;DR: The above is performed automatically, upon launching a wallet with pubkeys matching one of the 64 hardcoded keypairs.  The method `create_ntz_sig` is called twice before the wallet will check the notarization pool for pending ntz's.  These two calls cache the tx blob and data, and the cache serves to speed up the addition of new signatures to pending ntz's, when found in the ntzpool. Each call takes about 15 seconds to complete.**


<br><br>


<h2 id="append-sig">Appending signatures to a pending notarization</h2>

**Note: This should be performed automatically, if the cache is full (contains two transactions), and a notarization sequence has already begun (Pending notarization is in pool already)** 

If you wish to call it manually, use the syntax below:

```
curl -X POST http://127.0.0.1:12121/json_rpc -d '{"method":"append_ntz_sig"}'
```

This command will check the notarization pool for pending tx's, and fetch the one with the highest value in the `sig_count` field. This field's value should match the quantity of non-negative elements in the `signers_index` field.

The called function will return false for any keypair that is not hardcoded in `src/komodo/komodo_notaries.h`

This function will automatically pull in all necessary tx data, as well as cleaning all transactions from the ntzpool, except two pending txs.

These two transactions are:

1.) The transaction which has just been signed  
2.) The transaction itself, prior to signing.

#2 is left in pool in the event that after leaving the wallet side, the transaction becomes invalid.  This transaction will be cleaned after the next notary node appends its signatures.

<br><br>

<h2 id="relay-tx">Relaying a request for Notarization Signatures from other Notary Nodes:</h2>

**This will also be performed automatically, upon a successful and non-cacheing call to either of the above RPC methods**

To relay the created `tx_blob` from the previous command `ntz_transfer` issued to the wallet, perform the following:


Using the `tx_blob` from terminal output, issue the following `curl` command to the daemon's port `21111` (not the wallet):


```
curl -X POST http://localhost:21111/json_rpc -d '{"method":"request_ntz_sig","params":{"tx_blob":"long hex here","sig_count":1,"signers_index:[51,-1,-1,-1,-1,-1,-1,-1,-1,-1,-1,-1,-1],"payment_id":"0c9a0f1fc513e0aa5d70cccb0b00e9ff924ce9c4b0e2ec373060fbde1c5cc4e0"}}'
```

*This protocol command will not relay an actual transaction unless `sig_count` is greater than 13.* 

<br><br>


<h2 id="view-pending">Viewing Pending Notarizations</h2>
To view pending notarization txs, which have requested further notarization signatures:

Issue the following command to the running daemon interface: `print_pool`

If a notarization is currently idling in the mempool, awaiting further signatures, they will show up under the heading `Pending Notarization Transactions: `

Example:

User input:

```
print_pool
```

Output of a pending ntz tx with 9 signatures:

```
Pending Notarization Transactions: 
=======================================================
id: e5f38b36e7efd4275e32ad90d578dd36acbeae86301b7033df963f6416937419
ptx_hash: 8c152cb5b997b6f2f299f3d625808feaa6d2aefa0a9478b8842858aa3e53eb37
sig_count: 9
signers_index:  16 36 60 10 21 63 47 51 00 -1 -1 -1 -1 
blob_size: 53602
fee: 0.059570940000
fee/byte: 0.000001111356
receive_time: 1572956332 (33 minutes ago)
relayed: no
do_not_relay: F
kept_by_block: F
double_spend_seen: F
max_used_block_height: 2546
max_used_block_id: b56701d1d774e070aebd8366489cc63ec75e8c4bac4e37475f090f253a01e304
last_failed_height: 0
last_failed_id: 0000000000000000000000000000000000000000000000000000000000000000
```

**Notes on fields not present in a standard BLUR tx:** 

`ptx_hash` is the hash of a corresponding `ptx_blob` binary archive.  This archive contains all transaction data from the previous signer, for the next signer to add to their vectorized txs.  

`sig_count` is a count of signatures currently added to the transaction. 

`signers_index` is an array with 13 values, each of which corresponds to a Notary Node, and their row number in the hardcoded pubkeys.

If the value is not a default `-1`, the number should correspond to a notary node that has already signed and appended their transactions to the pending request, in time-sequential
order.


Because these notarization transactions use a completely separate validation structure (located in `src/cryptonote_basic/verification_context.h`), they will not validate
as normal transactions until `sig_count` reaches a point of being greater than `13`.

At that point, the `ntz_tx_verification_context` will be converted to the standard
`tx_verification_context`, and relayed to the network as a normal transcation.  Prior to this point, the separate context will prevent a transaction from being treated as a normal transaction, or validated as such by other nodes.


<br><br>


<h2 id="rpc-calls"> RPC Calls for KMD-BLUR Data </h2>


To retrieve the current blockchain data, and notarization data (not yet populated):


`curl -X POST http://localhost:52542/json_rpc -d '{"method":"get_notarization_data"}'`


Output:


```
{
  "id": 0,
  "jsonrpc": "2.0",
  "result": {
    "assetchains_symbol": "BLUR",
    "current_chain_hash": "0d6b7b0de6106877972fdafb806932cdd4c30dd007fc9510dfdec4db8b5ca69c",
    "current_chain_height": 155736,
    "current_chain_pow": "73ab840dcb63e247df9d98988a195b2ba17def30daa835073a8377be65010000",
    "notarized_hash": "0000000000000000000000000000000000000000000000000000000000000000",
    "notarized_height": 0,
    "notarized_txid": "0000000000000000000000000000000000000000000000000000000000000000"
  }
```


To retrieve the merkle root of a given block, by block hash or by vector of transaction hashes (`b.miner_tx` + `b.tx_hashes`):


By block hash:
```
$ curl -X POST http://localhost:52542/json_rpc -d '{"method":"get_merkle_root","params":{"block_hash":"fcef71dbd8138c1bb738df9848307bd766af11e763c9125014a023db706877d"}}'
{
  "id": 0,
  "jsonrpc": "2.0",
  "result": {
    "status": "OK",
    "tree_hash": "8f85f91445345bfdc47633fb8f81002781994ace103706568f97249e5f5efee1"
  }
}
```


By transaction hashes:


```
$ curl -X POST http://localhost:52542/json_rpc -d '{"method":"get_merkle_root","params":{"tx_hashes":["6ca1743cb1db1f4f34b132919b7941f766146b4dbf36fd6db88ff9563b7710b","abdbab0a70288fc8106de68715db988e901cf51f77696011c6479822a9236b8b"]}}'
{
  "id": 0,
  "jsonrpc": "2.0",
  "result": {
    "status": "OK",
    "tree_hash": "8f85f91445345bfdc47633fb8f81002781994ace103706568f97249e5f5efee1"
```

