{% set short_lang = 'python' %}
{% if language == 'JavaScript' %}
    {% set short_lang = 'js' %}
{% endif %}

{% set lowercase_lang = 'python' %}
{% if language == 'JavaScript' %}
    {% set lowercase_lang = 'javascript' %}
{% endif %}

***********************************************
Transaction Processor Tutorial  {{ language }}
***********************************************

Overview
========

This tutorial covers the creation of a new Sawtooth transaction family in
{{ language }}, based on the Sawtooth SDK. We will construct a transaction
handler which implements a distributed version of the multi-player game tic-
tac-toe.

.. note::

    The SDK contains a fully-implemented version of tic-tac-toe. This tutorial
    is meant to demonstrate the relevant concepts, rather than to create a
    complete implementation. See the SDK_ for full implemenations in
    multiple languages.

.. _SDK: https://github.com/hyperledger/sawtooth-core/tree/master/sdk/examples

A general description of tic-tac-toe, including the rules, can be found on
Wikipedia at:

    https://en.wikipedia.org/wiki/Tic-tac-toe

A full implementation of the tic-tac-toe transaction family can be found in

``/project/sawtooth-core/sdk/examples/xo_{{ lowercase_lang }}/``.

Prerequisites
=============

This tutorial assumes that you have gone through :doc:`/app_developers_guide/getting_started` and are
familiar with the concepts introduced there.

You should be familiar with the concepts introduced in the
:doc:`/app_developers_guide/getting_started` guide and have a working Sawtooth
environment prior to completing this tutorial.

The Transaction Processor
=========================

There are two top-level components of a transaction processor: a processor
class and a handler class. The SDK provides a general-purpose processor class.
The handler class is application-dependent and contains the business logic for
a particular family of transactions. Multiple handlers can be connected to an
instance of the processor class.

Handlers get called in two ways:

#. An ``apply`` method
#. Various "metadata" methods

The metadata is used to connect the handler to the processor, and
we'll discuss it at the end of this tutorial. The bulk of the handler, however,
is made up of ``apply`` and its helper functions, so that's where we'll start.

The ``apply`` Method
====================

{% if language == 'JavaScript' %}
``apply`` gets called with two arguments, ``transactionProcessRequest`` and ``stateStore``.
``transactionProcessRequest`` holds the command that is to be executed (e.g. taking a space or
creating a game), while ``stateStore`` stores information about the current
state of the game (e.g. the board layout and whose turn it is).

The transaction contains payload bytes that are opaque to the validator core,
and transaction family specific. When implementing a transaction handler the
binary serialization protocol is up to the implementer.

{% else %} 
``apply`` gets called with two arguments, ``transaction`` and
``state_store``. The argument ``transaction`` is an instance of the class
Transaction that is created from the  protobuf definition. Also,
``state_store`` is an instance of the class State from the  python SDK.

``transaction`` holds the command that is to be executed (e.g. taking a space or
creating a game), while ``state_store`` stores information about the current
state of the game (e.g. the board layout and whose turn it is).

The transaction contains payload bytes that are opaque to the validator core,
and transaction family specific. When implementing a transaction handler the
binary serialization protocol is up to the implementer.
{% endif %}

Without yet getting into the details of how this information is encoded, we can
start to think about what ``apply`` needs to do. ``apply`` needs to

1) unpack the command data from the transaction,
2) retrieve the game data from the state store,
3) play the game, and
4) save the updated game data.

Accordingly, a top-down approach to ``apply`` might look like this:

{% if language == 'JavaScript' %}

.. code-block:: javascript

      apply (transactionProcessRequest, stateStore) {
        return _unpackTransaction(transactionProcessRequest)
        .then((transactionData) => {

        let stateData = _getStateData(transactionData.gameName, stateStore)

        let updatedGameData = _playXO(
          stateData.board,
          stateData.state,
          stateData.player1,
          stateData.player2,
          transactionData.signer,
          transactionData.action,
          transactionData.space
        )
        _storeGameData(transactionData.gameName, updatedGameData, stateStore)
        })
      }
    }

{% else %}

{# Python code is the default #}

.. code-block:: python

    def apply(self, transaction, state_store):
        signer, game_name, action, space = \
            self._unpack_transaction(transaction)

        board, state, player1, player2 = \
            self._get_state_data(game_name, state_store)

        updated_game_data = self._play_xo(
            board, state,
            player1, player2,
            signer, action, space
        )

        self._store_game_data(game_name, updated_game_data, state_store)

{% endif %}

Note that the third step is the only one that actually concerns tic-tac-toe;
the other three steps all concern the coordination of data.

Data
====

.. note::

    :doc:`/architecture/transactions_and_batches` contains a detailed
    description of how transactions are structured and used. Please read
    this document before proceeding, if you have not reviewed it.

So how do we get data out of the transaction? The transaction consists of a
header and a payload. The header contains the "signer", which is used to
identify the current player. The payload will contain an encoding of the game
name, the action ('create' a game, 'take' a space), and the space (which will be
an empty string if the action isn't 'take'). So our {% if language ==
'JavaScript' %}``_unpackTransaction``{% else %}``_unpack_transaction``{% endif
%} function will look like this:

{% if language == 'JavaScript' %}

.. code-block:: javascript

    const _unpackTransaction = (transaction) =>
      new Promise((resolve, reject) => {
        let header = TransactionHeader.decode(transaction.header)
        let signer = header.signerPubkey
        try {
          let payload = _decodeData(transaction.payload)
          resolve({gameName: payload[0],
                  action: payload[1],
                  space: payload[2],
                  signer: signer})
        } catch (err) {
          let reason =  new InvalidTransaction("Invalid payload serialization")
          reject(reason)
        }
      })


{% else %}

.. code-block:: python

    def _unpack_transaction(self, transaction):
        header = TransactionHeader()
        header.ParseFromString(transaction.header)
        signer = header.signer

        try:
            game_name, action, space = self._decode_data(transaction.payload)
        except:
            raise InvalidTransaction("Invalid payload serialization")

        return signer, game_name, action, space

{% endif %}


Before we say how exactly the transaction payload will be decoded, let's look at
{% if language == 'JavaScript' %}``_getStateData``{% else
%}``_get_state_data``{% endif %}. Now, as far as the handler is concerned, it
doesn't matter how the game data is stored. The only thing that matters is that
given a game name, the state store is able to give back the correct game data.
(In our full XO implementation, the game data is stored in a Merkle-radix tree.)


{% if language == 'JavaScript' %}

.. code-block:: javascript

    const _getStateData = (gameName, stateStore) => {
        let address = _makeGameAddress(gameName)

        return stateStore.get([address])
        .then((stateEntries) => {
        try {
          let data =  _decodeData(stateEntries[address])
          if (data.length < 5){
            while (data.length < 5){
              data.push("")
            }
          }
          return {board: data[0],
                  gameState: data[1],
                  player1: data[2],
                  player2: data[3],
                  storedName: data[4]}
        } catch(err) {
          throw new InternalError("Failed to deserialize game data." + err)
        }
      })
      .catch(_toInternalError)
    }

    const _toInternalError = (err) => {
      let message = (err.message) ? err.message : err
      throw new InternalError(message)
    }

{% else %}

.. code-block:: python

    def _get_state_data(self, game_name, state_store):
        game_address = self._make_game_address(game_name)

        state_entries = state_store.get([game_address])

        try:
            return self._decode_data(state_entries[0].data)
        except IndexError:
            return None, None, None, None
        except:
            raise InternalError("Failed to deserialize game data.")


{% endif %}

By convention, we'll store game data at an address obtained from hashing the
game name prepended with some constant:

{% if language == 'JavaScript' %}

.. code-block:: javascript

    const _makeGameAddress = (gameName) => {
       let prefix = XO_NAMESPACE
       let gameHash = crypto.createHash('sha512').update(gameName).digest('hex').toLowerCase()
       return prefix + gameHash
    }
        
{% else %}

.. code-block:: python

    def _make_game_address(self, game_name):
        prefix = self._namespace_prefix
        game_name_utf8 = game_name.encode('utf-8')
        return prefix + hashlib.sha512(game_name_utf8).hexdigest()


{% endif %}

Finally, we'll store the game data. To do this, we simply need to encode the
updated state of the game and store it back at the address from which it came.

{% if language == 'JavaScript' %}

.. code-block:: javascript

    const _storeGameData = (gameName, gameData, stateStore) => {
      let gameAddress = _makeGameAddress(gameName)

      let encodedGameData = _encodeData(gameData)

      let entries = {[gameAddress]: gameData}
      stateStore.set(entries)
      .then((gameAddresses) => {
        if (gameAddresses.length < 1) {
          throw new InternalError('State Error!')
        }
        console.log(`Set ${gameAddress} to ${gameData}`)
      })
    }

{% else %}

.. code-block:: python

    def _store_game_data(self, game_name, game_data, state_store):
        game_address = self._make_game_address(game_name)

        encoded_game_data = self._encode_data(game_data)

        addresses = state_store.set([
            StateEntry(
                address=game_address,
                data=encoded_game_data
            )
        ])

        if len(addresses) < 1:
            raise InternalError("State Error")

{% endif %}

So, how should we encode and decode the data? We have many options in binary
encoding schemes; the binary data stored in the validator state is up to the
implementer of the handler. In this case, we'll encode the data as a simple
UTF-8 comma-separated value string, but we could use something more
sophisticated, `BSON <http://bsonspec.org/>`_.


{% if language == 'JavaScript' %}

.. code-block:: javascript

    const _decodeData = (data) => {
      return data.toString().split(",")
    }

    const _encodeData = (data) => {
      return Buffer.from(data.join())
    }

{% else %}

.. code-block:: python

    def _decode_data(self, data):
        return data.decode().split(',')

    def _encode_data(self, data):
        return ','.join(data).encode()

{% endif %}

Implementing Game Play
======================


All that's left to do is describe how to play tic-tac-toe. The details here are
fairly straighforward, and the {% if language == 'JavaScript' %}``_playXO``{%
else %}``_play_xo``{% endif %} function could certainly be implemented in
different ways. To see our implementation, go to ``/project/sawtooth-
core/sdk/examples/xo_{{ lowercase_lang }}``. We choose to represent the board as
a string of length 9, with each character in the string representing a space
taken by X, a space taken by O, or a free space. Updating the board
configuration and the current state of the game proceeds straightforwardly.


The {% if language == 'JavaScript' %}``XOHandler``{% else %}``XoTransactionHandler``{% endif %} Class
===================================

{% if language == 'JavaScript' %}

And that's all there is to ``apply``! All that's left to do is set up the
``XOHandler`` class and its metadata. The metadata is used to
*register* the transaction processor with a validator by sending it information
about what kinds of transactions it can handle.

.. code-block:: javascript

    class XOHandler extends TransactionHandler {
      constructor () {
        super(XO_FAMILY, '1.0', 'csv-utf8', [XO_NAMESPACE])
      }

      apply (transactionProcessRequest, stateStore) {
        // 

Note that the XOHandler class extends the TransactionHandler class defined in the 
JavaScript SDK. 

{% else %}

And that's all there is to ``apply``! All that's left to do is set up the
``XoTransactionHandler`` class and its metadata. The metadata is used to
*register* the transaction processor with a validator by sending it information
about what kinds of transactions it can handle.

.. code-block:: python

    class XoTransactionHandler:
        def __init__(self, namespace_prefix):
            self._namespace_prefix = namespace_prefix

        @property
        def family_name(self):
            return 'xo'

        @property
        def family_versions(self):
            return ['1.0']

        @property
        def encodings(self):
            return ['csv-utf8']

        @property
        def namespaces(self):
            return [self._namespace_prefix]

        def apply(self, transaction, state_store):
            # ...


{% endif %}



Playing XO With The XO Client
=============================

You can now use the included XO client to test your new XO transaction
processor. Sawtooth includes an XO CLI-based client written in Python, which
can interact with any correctly written XO transaction processor, regardless
of language.

The steps below show you how to set up and play a game.


Start The Necessary Components
------------------------------

To play tic-tac-toe, you need the following components to be running and
connected:

#. At least one validator
#. The config family transaction processor, tp_config
#. An XO family transaction processor (the one you have written, or one
   of the included XO processors)
#. The REST API


For more information on configuring and runing Sawtooth components, see
:doc:`/app_developers_guide/getting_started`.


Create Players
--------------


Create two players to play the game:

.. code-block:: console

    $ xo init --username jack
    $ xo init --username jill


The command produces output similar to the following for both players:

.. code-block:: console

    set username: jack
    set url: http://127.0.0.1:8080
    writing file: /home/ubuntu/.sawtooth/keys/jack.priv
    writing file: /home/ubuntu/.sawtooth/keys/jack.addr
    set username: jill
    set url: http://127.0.0.1:8080
    writing file: /home/ubuntu/.sawtooth/keys/jill.priv
    writing file: /home/ubuntu/.sawtooth/keys/jill.addr


When interacting with the XO cli, `xo --init` can also be used to set the
active player. If the above commands are run in the order shown, both players
will be created, but "jill" will be the active player. To switch the active
player, simply run the `xo init` command again with the name of the
desired player.

Set "jack" as the active player now with:

.. code-block:: console

    $ xo init --username jack


Create A Game
-------------

Create a game with the following command:

.. code-block:: console

    $ xo create game1

To see list of the created games, enter the following command:

.. code-block:: console

    $ xo list

The command outputs a list of the games that have been created:

.. code-block:: console

    GAME            PLAYER 1        PLAYER 2        BOARD     STATE
    game1                                           --------- P1-NEXT


Take A Space As Player One
--------------------------

Start playing by taking a space as the first player, "jack":

.. code-block:: console

    $ xo take game1 4

.. note::

    The board spaces are numbered from one to nine. The upper-left corner is
    number one, and the lower right corner is number nine.


Take A Space As Player Two
--------------------------

To take a space on the board as player two, "jill" needs to be set as the
active player. Run the following command:

.. code-block:: console

    $ xo init --username jill


Now take a space on the board as player two:

.. code-block:: console

    $ xo take game1 3


Show The Current State Of The Game Board
----------------------------------------

Whenever you want to see the current state of the game board, enter the
following command:

.. code-block:: console

    $ xo show game1

You will see the current state of the board displayed:

.. code-block:: console

    GAME:     : game1
    PLAYER 1  : 024c8f
    PLAYER 2  : 03f8f2
    STATE     : P1-NEXT

        |   | O
     ---|---|---
      X |   |
     ---|---|---
        |   |


Continue Game
-------------

You can continue the game until one of the players wins, or
the game ends in a draw:

.. warning::

  Be sure to switch users before taking each move to simulate two distinct
  users playing.


.. code-block:: console

    $ xo show game1
    GAME:     : game1
    PLAYER 1  : 024c8f
    PLAYER 2  : 03f8f2
    STATE     : P2-WIN

      X |   | O
     ---|---|---
      X | O |
     ---|---|---
      O |   | X


