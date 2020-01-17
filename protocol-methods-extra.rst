=======================
 Protocol Methods Extra
=======================

This attempts to document implementation specific extensions that are intended
to be widely useful. These are not part of the main specification, but may be
suitable for eventual standardization.

If the server supports any of these extensions, these must be discoverable via
the RPC call `server.features`.

blockchain.scripthash.get_first_use
===================================

Returns the hash and height of the first block scripthash occured in on the
block chain. It also includes a list of all transaction in that block that
contain the scripthash in its output.

If there are no matching transactions in the blockchain, but there are in the
mempool, these are returned instead. In this case, the block hash returned is
the hex representation of `0` and block height the numeric value `0`

If the server supports this method if it has the key `firstuse` with
value `["1.0"]` in its `server.features` response.

**Signature**

  .. function:: blockchain.block.get_first_use(scripthash)
  .. versionadded:: ElectrsCash 1.1

  *scripthash*

    The script hash as a hexadecimal string.

**Result**

  A dictionary with the following keys.

  * *block_height*

    The block height of the block where the first occurance of scripthash exists.

    Block height is 0 if the first occurance is in the mempool, or on empty result.

  * *block_hash*

    The block hash of the block where the first occurance of scripthash exists.

    Block hash is hex of 0 if the first occurance is in the mempool, or on
    empty result.

  * *tx_hash*

   An array of all transactions containing the scripthash as output in this
   block.

   This array is empty if there are no matches.


**Example Result**

With *scripthash* `98cab349d97ed2e9f115c59757b0547373db25df5c41c267760077949c2dd774` on the Bitcoin Cash testnet chain:

::

   {
      'block_hash': '000000000000007f6c0e53f4560e15e1a805470144c00e1d81b78543e69c59c4',
      'block_height': 1258363,
      'tx_hash': [
         '491aa3f7fd1f23882c3437898125259d2a0c044bea0f6aa659b8d7d6ab75632a'
      ]
   }

**Standardization status**

Improvement proposal not yet written.
