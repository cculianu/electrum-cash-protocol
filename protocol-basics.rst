Protocol Basics
===============

Message Stream
--------------

Clients and servers communicate using **JSON RPC** over an unspecified underlying stream
transport.  Examples include TCP, SSL, WS and WSS.

Two standards `JSON RPC 1.0
<http://www.jsonrpc.org/specification_v1>`_ and `JSON RPC 2.0
<http://www.jsonrpc.org/specification>`_ are specified; use of version
2.0 is encouraged but not required.  Server support for batch requests
*is* implemented in Fulcrum as of version 1.6.0.

.. note::
  A client or server should only indicate JSON RPC 2.0 by
  setting the `jsonrpc <http://www.jsonrpc.org/specification#request_object>`_ member of
  its messages to ``"2.0"`` if it supports the version 2.0 protocol in
  its entirety.  Fulcrum does and will expect clients advertizing so
  to function correctly.  Those that do not will be disconnected and
  possibly blacklisted.

  Additionally, Fulcrum imposes the following constraint on JSON RPC messages: JSON-RPC `id` fields must not be JSON
  numerics with fractional parts. In other words they should be a string, integer, or `null`. JSON-RPC messages
  containing a floating point number in the `id` field will be rejected.

For the TCP and SSL transports: Each RPC call MUST be delimited by a single newline.
The JSON specification does not permit control characters within strings, so no
confusion is possible there.  However it does permit newlines as extraneous
whitespace between elements; client and server MUST NOT use newlines in such a
way.

For the WS and WSS transports: Newline delimiters between requests are not required,
since this transport has message-level framing.

A server advertising support for a particular protocol version MUST
support each method documented for that protocol version, unless the
method is explicitly marked optional.  It may support other methods or
additional parameters with unspecified behavior.  Use of additional
parameters is discouraged as it may conflict with future versions of
the protocol.

Notifications
-------------

Some RPC calls are subscriptions which, after the initial response,
will send a JSON RPC :dfn:`notification` each time the thing
subscribed to changes.  The `method` of the notification is the same
as the method of the subscription, and the `params` of the
notification (and their names) are given in the documentation of the
method.


.. _version negotiation:

Version Negotiation
-------------------

It is desirable to have a way to enhance and improve the protocol
without forcing servers and clients to upgrade at the same time.

Protocol versions are denoted by dotted number strings with at least
one dot.  Examples: "1.5", "1.4.1", "2.0".  In "a.b.c" *a* is the
major version number, *b* the minor version number, and *c* the
revision number.

A party to a connection will speak all protocol versions in a range,
say from `protocol_min` to `protocol_max`, which may be the same.
When a connection is made, both client and server must initially
assume the protocol to use is their own `protocol_min`.

The client should send a :func:`server.version` RPC call as early as
possible in order to negotiate the precise protocol version; see its
description for more detail.  All responses received in the stream
from and including the server's response to this call will use its
negotiated protocol version.


.. _script hashes:

Script Hashes
-------------

A :dfn:`script hash` is the hash of the binary bytes of the locking
script (ScriptPubKey), expressed as a hexadecimal string.  The hash
function to use is given by the "hash_function" member of
:func:`server.features` (currently :func:`sha256` only).  Like for
block and transaction hashes, when converting the big-endian binary
hash to a hexadecimal string the least-significant byte appears first,
and the most-significant byte last.

For example, the legacy Bitcoin address from the genesis block::

    1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa

has P2PKH script::

    76a91462e907b15cbf27d5425399ebf6f0fb50ebb88f1888ac

with SHA256 hash::

    6191c3b590bfcfa0475e877c302da1e323497acf3b42c08d8fa28e364edf018b

which is sent to the server reversed as::

    8b01df4e368ea28f8dc0423bcf7a4923e3a12d307c875e47a0cfbf90b5c39161

By subscribing to this hash you can find P2PKH payments to that address.

One public key, the genesis block public key, among the trillions for
that address is::

    04678afdb0fe5548271967f1a67130b7105cd6a828e03909a67962e0ea1f61deb
    649f6bc3f4cef38c4f35504e51ec112de5c384df7ba0b8d578a4c702b6bf11d5f

which has P2PK script::

    4104678afdb0fe5548271967f1a67130b7105cd6a828e03909a67962e0ea1f61deb
    649f6bc3f4cef38c4f35504e51ec112de5c384df7ba0b8d578a4c702b6bf11d5fac

with SHA256 hash::

    3318537dfb3135df9f3d950dbdf8a7ae68dd7c7dfef61ed17963ff80f3850474

which is sent to the server reversed as::

    740485f380ff6379d11ef6fe7d7cdd68aea7f8bd0d953d9fdf3531fb7d531833

By subscribing to this hash you can find P2PK payments to the genesis
block public key.

.. note:: The Genesis block coinbase is uniquely unspendable and
   therefore not indexed.  It will not show with the above P2PK script
   hash subscription.


.. _status:

Status
------

To calculate the `status` of a :ref:`script hash <script hashes>` (or
address):

1. order confirmed transactions to the script hash by increasing
height (and position in the block if there are more than one in a
block)

2. form a string that is the concatenation of strings
``"tx_hash:height:"`` for each transaction in order, where:

  * ``tx_hash`` is the transaction hash in hexadecimal

  * ``height`` is the height of the block it is in.

3. Next, with mempool transactions in a :ref:`specified order <mempoolorder>`, append a similar
string for those transactions, but where **height** is ``-1`` if the
transaction has at least one unconfirmed input, and ``0`` if all
inputs are confirmed.

4. The :dfn:`status` of the script hash is the :func:`sha256` hash of the
full string expressed as a hexadecimal string, or :const:`null` if the
string is empty because there are no transactions.

Note that as of Fulcrum 1.8.0 the transaction history for a script hash
also includes transactions that involve sending/receiving :ref:`CashTokens <cashtokens>`
to/from that script hash.


Block Headers
-------------

Originally Electrum clients would download all block headers and
verify the chain of hashes and header difficulty in order to confirm
the merkle roots with which to check transaction inclusion.

With the BTC and BCH chains now past height 500,000, the headers form
over 40MB of raw data which becomes 80MB if downloaded as text from
Electrum servers.  The situation is worse for testnet and coins with
more frequent blocks.  Downloading and verifying all this data on
initial use would take several minutes, during which Electrum was
non-responsive.

To facilitate a better experience for SPV clients, particularly on
mobile, protocol :ref:`version 1.4 <version 1.4>` introduces an
optional *cp_height* argument to the :func:`blockchain.block.header`
and :func:`blockchain.block.headers` RPC calls.

This requests the server provide a merkle proof, to a single 32-byte
checkpoint hard-coded in the client, that the header(s) provided are
valid in the same way the server proves a transaction is included in a
block.  If several consecutive headers are requested, the proof is
provided for the final header - the *prev_hash* links in the headers
are sufficient to prove the others valid.

Using this feature client software only needs to download the headers
it is interested in up to the checkpoint.  Headers after the
checkpoint must all be downloaded and validated as before.  The RPC
calls return the merkle root, so to embed a checkpoint in a client
simply make an RPC request to a couple of trusted servers for the
greatest height to which a reorganisation of the chain is infeasible,
and confirm the returned roots match.

.. note:: with 500,000 headers of 80 bytes each, a naïve server
  implementation would require hashing approximately 88MB of data to
  provide a single merkle proof.  Fulcrum implements an optimization
  such that it hashes only approximately 180KB of data per proof.


.. _dsproofs:

Double Spend Proofs (dsproofs)
------------------------------

A double spend proof is information collected by the Bitcoin Cash peer-to-peer
network on transaction inputs for transactions in the mempool that are seen to
have been attempted at being double-spent. Double-spend proofs only apply to
mempool transactions. Once a transaction is confirmed, the double-spend attempt
is no longer relevant (since the transaction cannot be double-spent anymore
unless there is a reorg). Double-spend proofs indicate that a transaction may
not confirm as expected, and that instead there is a risk that its conflicting
transaction will confirm instead.

`The specification for dsproofs can be found here <https://documentation.cash/protocol/network/messages/dsproof-beta>`_.

In Fulcrum, the dsproofs are returned as JSON objects with the following keys:

  * **dspid**

    This is the hexadecimal hash of the :const:`dsproof` as would
    be returned by querying the BCHN dsproof RPC :const:`getdsproof`.

  * **hex**

    The raw serialized double-spend proof itself.

  * **outpoint**

    A JSON object containing the following keys:

    * **txid**

      The transaction hash of the transaction that generated this outpoint.

    * **vout**

      The integer output number for this outpoint.

  * **txid**

    The primary transaction that is associated with this :const:`dsproof`.

  * **descendants**

    A JSON array of *txid*'s of all the transactions that are potentially
    affected by this double-spend attempt. This list will include `txid` above
    plus all of its descendant transactions.

An example `dsproof` object as might be returned by Fulcrum::

    {
      "dspid": "587d18bf8a64ede9c7450fdaeab27b9b3c46cfa8948f4c145f889601153c56b0",
      "txid": "5b59ce35093fbd13549cd6f203d4b5b01762d70e75b8e9733dfc463e0ff8cc13",
      "hex": "410c56078977120e828e4aacdd813a818d17c47d94183aa176d62c805d47697dddddf46c2ab68ee1e46a3e17aa7da548c38ec43416422d433b1782eb3298356df441",
      "outpoint": {
        "txid": "f6e2a16ba665d5402dad147fe35872961bc6961da62345a2171ee001cfcf7600",
        "vout": 0
      },
      "descendants": [
        "36fbb099e6de59d23477727e3199c65caae35ded957660f56fc681a6d81d5570",
        "5b59ce35093fbd13549cd6f203d4b5b01762d70e75b8e9733dfc463e0ff8cc13"
      ]
    }

Note that as of March 2021, only servers running Bitcoin Cash Node v22.3.0 or later
are capable of reporting double-spend proofs via RPC, and thus only such servers
will provide double-spend proofs to clients via the Electrum Cash protocol.
Servers that support `dsproof` will have the key :const:`"dsproof"` set to
:const:`true` in their :func:`server.features` map.


.. _cashtokens:

CashToken Support
-----------------

As of Fulcrum 1.8.0, transactions containing CashToken inputs and output are
understood and indexed correctly.  This means that the token data is correctly
parsed out of the transaction locking scripts and is separated out from the
actual destination for the output.  So calls like :func:`blockchain.scripthash.get_history`
may return results including transactions that touch token data for a particular
address.

Additionally, as of Fulcrum 1.9.0 (protocol version 1.5.0), CashToken data may
be returned from some RPCs as an optional key :const:`token_data`. The following RPCs may
return results containing this key:  :func:`blockchain.address.listunspent`,
:func:`blockchain.scripthash.listunspent`, and :func:`blockchain.utxo.get_info`.

Servers that return :const:`token_data` in their results will have the key
:const:`"cashtokens"` set to :const:`true` in their :func:`server.features` map.

.. _token_data:

The :const:`token_data` JSON object has the following keys:

  * **amount**

    A JSON string representing the fungible token quantity for this token output.
    The reason a string is used here is because tokens have a maximum amount of 2^63
    whereas JSON numerics support up to 2^53. Thus, the amount cannot be a JSON numeric
    and is instead written as a decimal string in the JSON output.

  * **category**

    A JSON hex string representing 32-byte token category (or token id), in big endian
    (reversed) byte order. Note that on the blockchain token categories are serialized
    in little endian byte order (this is similar to how transaction ids work).

  * **nft** (optional)

    A JSON object representing a single non-fungible token.  This key may be missing if
    this CashToken only contains fungible tokens and no NFTs.  The keys inside this object
    are as follows:

    * **capability**

      A JSON string, one of: :const:`"none"`, :const:`"mutable"`, or :const:`"minting"`.

    * **commitment**

      A JSON hex string representing the NFT's commitment data. Unlike :const:`category` above
      this is not in any reversed order and is in the same order as it would appear on
      the blockchain.

An example of a :func:`blockchain.scripthash.listunspent` result containing :const:`token_data`::

    [
      {
        "height": 126184,
        "token_data": {
          "amount": "1000000",
          "category": "8fd6a2f713beaa5907a776b8b3060cddd1c6ff0588554c2364698ae271321ce9",
          "nft": {
            "capability": "minting",
            "commitment": "f00fd00fb33f"
           }
        },
        "tx_hash": "87489c43bae69c297bbaf65276573b0001c20c647a3d54d2842a4425ff87bacc",
        "tx_pos": 1,
        "value": 1000000
        }
    ]


.. _rpa prefix:

RPA Prefix Search Support
-------------------------

Fulcrum supports the RPA (reusable payment address) indexing scheme. Some RPC methods are provided to search for
transactions matching a certain RPA prefix value. For a discussion of what RPA prefixes are, please consult the
`Reusable Payment Address Specification <https://github.com/imaginaryusername/Reusable_specs/blob/master/reusable_addresses.md>`_
and in particular the section discussing `receiving on-chain via prefix searching <https://github.com/imaginaryusername/Reusable_specs/blob/master/reusable_addresses.md#receiving-onchain-direct>`_

In Fulcrum, the RPA-related APIs such as :func:`blockchain.rpa.get_history` take an :const:`rpa_prefix` argument, which
is a 1-4 character hexadecimal string that represents the first two bytes of the double sha256 hash of a transaction's
serialized input. So for example if the transaction's input serialized and hashed to this value::

    abcd740485f380ff6379d11ef6fe7d7cdd68aea7f8bd0d953d9fdf3531fb7d53

Then possible prefixes would be either: :const:`"a"` (4 bit prefix of the above hash), :const:`"ab"` (8 bit prefix of the above hash),
:const:`"abc"` (12 bit prefix of the above hash), or :const:`"abcd"` (16 bit prefix of the above hash).


.. _mempoolorder:

Mempool Transaction Ordering
----------------------------

The protocol specifies a specific ordering for mempool transactions. This ordering is not necessarily topological
ordering, but is something simpler. The mempool ordering in this protocol is specified as follows: mempool transactions
are sorted in ascending order using the following pseudo-code comparator function (where :const:`a` and :const:`b` are
transactions to be compared)::

    (a.hasUnconfirmedParents, a.txHash) < (b.hasUnconfirmedParents, b.txHash)

That is, transactions are sorted such that all transactions with no unconfirmed parents appear as a grouping before
all transactions with unconfirmed parents, and within each grouping, they are sorted by their transaction hash (as a
string).

Certain concepts in the protocol such as the :ref:`status hash <status>` make use of this ordering. Additionally,
certain methods, such as :func:`blockchain.scripthash.get_mempool` and :func:`blockchain.scripthash.get_history` may
return a list of transactions from the mempool in this order.
