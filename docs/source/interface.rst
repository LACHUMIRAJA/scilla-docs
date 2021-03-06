Interpreter Interface
=========================

The Scilla interpreter executable provides a calling interface so as to allow
invoking of transitions with specified inputs and obtain outputs. Execution of
a contract with supplied inputs will result in a set of outputs and a change in
the smart contract mutable state. 

Calling Interface
###################

A transition defined in a smart contract can be called either by issuing a
transaction or by another smart contract through message calls. The same
interface will be used to call a contract via a transaction from outside and
also for inter-contract message calls.

The input to the interpreter (``scilla-runner``) consists of four JSON files.
Each of these are described below. Every execution of the interpreter will be
provided with four json inputs: ::

    ./scilla-runner -init init.json -istate input_state.json -iblockchain input_blockchain.json -imessage input_message.json -o output.json -i input.scilla

The interpreter executable can be run either to create a contract (denoted
``CreateContract``) or to invoke a function in a contract (``InvokeContract``).
Depending on the two cases, some of the arguments will be absent. The table
below presents the arguments that should be present in either of the two cases.
A ``CreateContract`` or an ``InvokeContract`` can be distinguished based on the
presence of ``input_message.json``. If the argument is absent, then the
interpreter will treat evaluation as a ``CreateContract`` else it will treat it
as an ``InvokeContract``. Note that for ``CreateContract``, the interpreter
only performs basic type checking to match with contract’s immutable
parameters.


+---------------------------+-----------------------+------------------------------------------+
|                           |                       | Present                                  |
+===========================+=======================+=====================+====================+
| Input                     |    Description        |``CreateContract``   | ``InvokeContract`` |
+---------------------------+-----------------------+---------------------+--------------------+
| ``init.json``             | Mutable params        | Yes                 |  Yes               |
+---------------------------+-----------------------+---------------------+--------------------+
| ``input_state.json``      | Contract state        | No                  |  Yes               |  
+---------------------------+-----------------------+---------------------+--------------------+
| ``input_blockchain.json`` | Blockchain state      | Yes                 |  Yes               |    
+---------------------------+-----------------------+---------------------+--------------------+
| ``input_message.json``    | Transition and params | No                  |  Yes               |
+---------------------------+-----------------------+---------------------+--------------------+
| ``output.json``           | Output                | Yes                 |  Yes               |
+---------------------------+-----------------------+---------------------+--------------------+
| ``input.scilla``          | Input contract        | Yes                 |  Yes               |
+---------------------------+-----------------------+---------------------+--------------------+


Initial Immutable State
#########################

``init.json`` defines the values of the immutable variables in a contract.
`init.json` does not change b/w invocations.  ``init.json`` is an array of
objects, each of which contains the following fields:

=====  ==========================================
Field      Description
=====  ==========================================  
vname  Name of the immutable contract parameter
type   Type of the immutable contract parameter
value  Value of the immutable contract parameter
=====  ==========================================  


Example 1
**********

For the ``HelloWorld.scilla`` contract fragment given below, we have only one
immutable variable ``owner``.

.. code-block:: ocaml

    contract HelloWorld
     (* Immutable parameters *)
     (owner: Address)


A sample ``init.json`` for this contract will look like the following:

.. code-block:: json

    [
        {
            "vname" : "owner",
            "type"  : "Address", 
            "value" : "0x1234567890123456789012345678901234567890"
        }
    ]


Example 2
**********
    
For  ``Crowdfunding.scilla`` contract fragment given below, we have three 
immutable variables ``owner``, ``max_block`` and ``goal``.


.. code-block:: ocaml

    contract Crowdfunding
        (* Immutable parameters *)
        (owner     : Address,
         max_block : BNum,
         goal      : Int)


A sample ``init.json`` for this contract will look like the following:


.. code-block:: json

    [
        {
            "vname" : "owner",
            "type"  : "Address", 
            "value" : "0x1234567890123456789012345678901234567890"
        },
        {
            "vname" : "max_block",
            "type"  : "BNum" ,
            "value" : "199"
        },
        { 
            "vname" : "goal",
            "type"  : "Int",
            "value" : "500"
        }
    ]

Input Blockchain State
########################

``input_blockchain.json`` supplies the current blockchain state to the
interpreter. It is similar to ``init.json``, except that it is a fixed size
array of objects, each object having ``vname`` fields only from a
pre-determined set (which correspond to actual blockchain state variables). 

Permitted JSON example: only JSONs that differ in the ``value`` field from the
below example are permitted for now.

.. code-block:: json

    [
        {
            "vname" : "BLOCKNUMBER",
            "type"  : "BNum", 
            "value" : "3265"
        }
    ]

Input Message
###############

``input_message.json`` contains the required information to invoke a
transition. The json is an array of the following four objects:

=======  ===========================================
Field      Description
=======  ===========================================  
_tag      Transition to be invoked
_amount   Number of ZILs to be transferred
_sender   Address of the invoker
params    An array of parameter objects
=======  ===========================================  


The first three fields namely ``_sender``, ``_amount``, and ``_tag`` are compulsory in the
sense that their value cannot be ``NULL``. 

The ``params`` array is encoded similar to how ``init.json`` is encoded,
specifying the (``vname``, ``type``, ``value``) of each parameter that has to be
passed to the transition being invoked. 

Example 1
**********
For the following transition:

.. code-block:: ocaml

    transition SayHello()

an example ``input_message.json`` is given below:

.. code-block:: json

    {
        "_tag"    : "SayHello",
        "_amount" : "0",
        "_sender" : "0x1234567890123456789012345678901234567890",
        "params"  : []
    }

Example 2
**********
For the following transition:

.. code-block:: ocaml

    transition TransferFrom (from : Address, to : Address, tokens : Int)

an example ``input_message.json`` is given below:

.. code-block:: json

    {
        "_tag"    : "TransferFrom",
        "_amount" : "0",
        "_sender" : "0x64345678901234567890123456789012345678cd",
        "params"  : [
            {
                "vname" : "from",
                "type"  : "Address",
                "value" : "0x1234567890123456789012345678901234567890"
            },
            {
                "vname" : "to",
                "type"  : "Address",
                "value" : "0x78345678901234567890123456789012345678cd"
            },
            {
                "vname" : "tokens",
                "type"  : "Int",
                "value" : "580"
            }
        ]
    }




Interpreter Output
#####################

The interpreter will return a JSON object (``output.json``)  with the following
fields:

=======  ================================================================
Field      Description
=======  ================================================================  
message   The emitted message to another contract/non-contract account. 
states    And array of objects that form the new contract state
=======  ================================================================  

``message`` is a JSON object that will have a similar format as
``input_message.json``, except that instead of ``_sender``, it will have a
``_recipient`` field. The fields in ``message`` are given below:

===========       ===========================================
Field              Description
===========       ===========================================  
_tag               Transition to be invoked
_amount            Number of ZILs to be transferred
_recipient         Address of the recipient
params             An array of parameter objects to be passed
===========       ===========================================  


The ``params`` array is encoded similar to how ``init.json`` is encoded,
specifying the (``vname``, ``type``, ``value``) of each parameter that has to
be passed to the transition being invoked. ``states`` is an array of objects
that represents the mutable state of the contract. Each entry of the ``states``
array also specifies ``vname``, ``type``, ``value``. 


Example 1
*********

The example below is an output generated by ``HelloWorld.scilla``. 

.. code-block:: json

    {
      "message" : {
        "_tag"      : "Main",
        "_amount"   : "0",
        "_recipient": "0x1234567890123456789012345678901234567890",
        "params": [
          { "vname": "welcome_msg", "type": "String", "value": "Hello World" }
        ]
      },
      "states": [
        { "vname": "_balance", "type": "Int", "value": "0" },
        { "vname": "msgstr", "type": "String", "value": "Hello World" }
      ]
    }

Example 2
*********

Another slightly more involved example with ``Map`` in ``states``.

.. code-block:: json

    {
      "message": {
        "_tag"       : "Main",
        "_amount"    : "0",
        "_recipient" : "0x12345678901234567890123456789012345678ab",
        "params": [ { "vname": "code", "type": "Int", "value": "1" } ]
      },
      "states": [
        { "vname": "_balance", "type": "Int", "value": "100" },
        {
          "vname"  : "backers",
          "type"   : "Map",
          "value"  : [
                        { "keyType": "Address", "valType": "Int" },
                        { "key": "0x12345678901234567890123456789012345678ab", "val": "100" }
          ]
        },
        {
          "vname" : "funded",
          "type"  : "ADT",
          "value" : { "constructor": "False", "argtypes": [], "arguments": [] }
        }
      ]
    }




.. note::

    For mutable variables of type ``Map``, the first entry in the ``value``
    field is a type of the ``key`` and ``value``. Also note the ``value``
    field of a variable of type ``ADT`` that has several field namely,
    ``constructor``, ``argtypes`` and ``arguments``.

Input Mutable Contract State
############################

``input_state.json`` contains the current value of mutable state variables. It
is similar to the ``states`` field in ``output.json`` except that there is an
extra field ``_balance`` that contains the balance of the contract in ZILs.
Given below is an example of ``input_state.json`` for ``Crowdfunding.scilla``. 

.. code-block:: json

    [
      {
        "vname" : "backers",
        "type"  : "Map",
        "value" : [
                      { "keyType": "Address", "valType": "Int" },
                      { "key": "0x12345678901234567890123456789012345678ab", "val": "100" }
        ]
      },
      {
        "vname" : "funded",
        "type"  : "ADT",
        "value" : { "constructor": "False", "argtypes": [], "arguments": [] }
      },
      {
        "vname" : "_balance",
        "type"  : "Int",
        "value" : "100"
      }
    ]


