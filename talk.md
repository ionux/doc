 The Script Interpreter

Gavin Andresen once pointed out there are no bitcoins, only transactions.

He's referring to the fact that all transfers of bitcoin are atomic.
An old, unspent coin is destroyed, and new outputs are created.
The reallocation is dictated by something called bitcoin script.

Every time a coin is transferred, requirements for redemption are 
expressed as Bitcoin Script. It's purpose is to:

 - prevent anyone from stealing the coins
 - allows the coin to be retrieved as terms dictate

### Supplemental
 
 Documentation of opcodes https://en.bitcoin.it/wiki/Script
 Source code for the interpreter: https://github.com/bitcoin/bitcoin/blob/master/src/script/interpreter.cpp

### Script

Script is a simple stack based, byte-code language. There are three types of opcodes:

  * those which push data to the stack
  * flow control
  * those which operate on the stack (arithmetic/cryptographic functions)
    

Usually, they operate on consumed values and push new data back onto the stack.
Each opcode carries out some sanity checking before proceeding, ie, checks required values are present on the stack.

All bitcoin scripts are pure, in that they are self contained, and cannot talk to the outside world. 

### Some Op Codes
   - OP_ADD / OP_SUB / OP_MUL / OP_DIV - Consume two values from the stack, and push the single result
   - OP_DUP - Duplicates the value at the top of the stack
   - OP_EQUAL - Compare the first two values on the stack, pushing a boolean indicating the result.
   - OP_EQUALVERIFY -  The same as OP_EQUAL, but causes an error if the values are not equal
   - OP_SHA256 / OP_RIPEMD160 / OP_HASH256 / OP_HASH160 / OP_SHA1 - Consume the first value of the stack, and push the resulting hash.
   - OP_IF / OP_ELSE / OP_ENDIF - Flow control, based on values on the stack
   - OP_CHECKSIG / OP_CHECKMULTISIG - Verify one or several ECDSA signature
   - OP_CHECKLOCKTIMEVERIFY / OP_CHECKSEQUENCEVERIFY (locktime, relative locktime) 

### Pushing data? 

 There are some simple rules for encoding data to be pushed. 
 On the byte level, the string is prefixed by it's length in bytes, allowing the parser to handle dynamic length strings. 
 In human readable representations this is implicitly happening all the time, but will be omitted mostly for clarity.
 
 * If the length of the string is <= 75, simply prefix the string with it's length.
 * If the length of the string is <= 255, the indicator is: "OP_PUSHDATA1 [length; uint8]"
 * If the length of the string is <= 65535, the indicator is: "OP_PUSHDATA2 [length; uint16]"
 * If the length of the string is <= 2^64-1, the indicator is: "OP_PUSHDATA4 [length; uint64]"
   
  The last is forbidden by network rules at this time, it's just not useful right now!
  
  The trick here is to use the opcode that lets you encode the least amount of data, but can still fit your string.  
  
### Simple examples 1
 1 1 OP_ADD 2 OP_EQUAL
  
 The above is a simple script to check that 1 + 1 is indeed 2.
 
 By running this script through the interpreter, we expect to be left with just one value on the stack. In this
 case, the value will always be true. 
 
<pre>
   Stack     Exec      Remaining
  |      |  |     |  | 1 1 OP_ADD 2 OP_EQUAL     |
  |      |  | 1   |  | 1 OP_ADD 2 OP_EQUAL       | _Push_ the number 1 onto the stack
  | 1    |  | 1   |  | OP_ADD 2 OP_EQUAL         | _Push_ the number 1 onto the stack
  | 1 1  |  | ADD |  | 2 OP_EQUAL                | _Pop_ the two numbers, add them, and _push_ the result
  | 2    |  | 2   |  | OP_EQUAL                  | _Push_ the number 2 onto the stack
  | 2 2  |  |EQUAL|  |                           | _Pop_ two values, compare them, and _push_ a boolean for this comparison
  | true |  |     |  |                           |
</pre>

### And in the real world?

 Bitcoin uses the cryptographic functionality in Script. 
 
 Coins can be reallocated by saying they can only be redeemed by the person with this private key. 
 This is where ECDSA is involved. The most common transaction type is 'pay-to-pubkey-hash'. You've
 used this if you've paid to a bitcoin address starting with '1'. 
 
 But in the beginning, there was only pay-to-public key. We'll look at this first. 
 
 The following is a snippet of a pay-to-pubkey transaction. Only the outputs are shown.
 
<pre>
{
    "txid": "f4184fc596403b9d638783cf57adfe4c75c605f6356fbc91338530e9831e9e16",
    "nVersion": 1,
    "nLockTime": 0,
    "inputs": [
      ...
    ],
    "outputs": [
      {
        "value": 1000000000,
        "scriptPubKey": "04ae1a62fe09c5f51b13905f07f06b99a2f7159b2225f374cd378d71302fa28414e7aab37397f554a7df5f142c21c1b7303b8a0626f1baded5c72a704f7e6cd84c OP_CHECKSIG"
      }
    ]
}
</pre>
 
### Our first transaction

Lets pretend that the above transaction belongs to us. If you look at the output, there is only one argument before OP_CHECKSIG. This is our public key. 

OP_CHECKSIG is an opcode that takes two values from the stack, and attempts to validate them using ECDSA. 

Those arguments are: [sig] [public key]
 
By setting the output script in this way, the funds are now locked to whomever can produce a signature 
by the private key, wh'os public key is 0x03fc5e46fe3e4ac0cff896be960e4a651166c4ea5038ac3a15d9425557f293c93b.
Because it's impossible to reverse a public key[*], only the bearer of the private key can spend these coins. 
 
When you lock coins using an output script, the bitcoin network expects that when spent, a script will be
provided that satisfies the requirements of the output script, whatever they may be. 

Whenever a transaction is being verified, the input script is run through the interpreter _followed by the 
output script_. Since the output script checks that it's requirements are being met, any necessary parameters
must be provided in the input script. 

### Spending our pay-to-pubkey transaction 
Lets look at what a transaction that spends this output might look like:

<pre>
{
    "txid": "ea44e97271691990157559d0bdd9959e02790c34db6c006d779e82fa5aee708e",
    "nVersion": 1,
    "nLockTime": 0,
    "inputs": [
      {
        "hashPrevOut": "f4184fc596403b9d638783cf57adfe4c75c605f6356fbc91338530e9831e9e16",
        "nPrevOut": 0,
        "scriptSig": "304402204e45e16932b8af514961a1d3a1a25fdf3f4f7732e9d624c6c61548ab5fb8cd410220181522ec8eca07de4860a4acdd12909d831cc56cbbac4622082221a8768d1d0901"
        "nSequence": 0xffffffff
      }
    ],
    "outputs": [
      ... 
    ]
}
</pre>

Since input scripts are run before the output script, lets see how this might look:

<pre>
Run the input script:

   Stack              Exec        Remaining
  |            |     |        |  | |sig|                |
  |            |     | |sig|  |  |                      | _Push_ the signature onto the stack
  | |sig|      |     |        |  |                      | 
  
Now run the output script:  
  
  | |sig|      |     |        |  | |pub| OP_CHECKSIG    | 
  | |sig|      |     ||pub|   |  | OP_CHECKSIG          | _Push_ the public key onto the stack
  | |sig| |pub||     |CHECKSIG|  |                      | _Pop_ two values, and perform ECDSA. Push a boolean with the result.
  | true       |     |        |  |                      |
  
</pre>

### A word on compatibility

If anyone tried to produce a signature with the wrong private key, we know the network will
verify it against the public key in the output anyway, meaning validation will fail. 
  
This is the type of work a full node does. Full nodes verify all transaction on the network, 
and use their set of rules to decide if blocks are valid, etc. 

Since transaction validity affects which blocks a node is going to accept, the implementation
affects the software's ability to sync with the network.

In fact, reimplementing the parsing and interpretation of Bitcoin scripts is no simple task!

At various points in history, bitcoin's consensus code has depended on external libraries. 

Satoshi used OpenSSL for ECDSA. This had unintended side effects, since if openssl suddenly decided
certain encodings weren't valid, this might cause nodes to break. 

This has happened before, and had the potential to cause a schism hard fork in bitcoin if someone
 happened to use one of these uncommon encodings. 
 
### Authentication today

We no longer use raw pay-to-pubkey, because public keys are harder to transfer. Instead we use addresses.

Bitcoin addresses are simply hashes of the corresponding public key encoded in base58. They include a checksum for integrity.  

Instead of directly specifying a public key in our script, we specify the hash. Our locking script will verify
that the redeeming party provides the correct public key first, before allowing expensive ECDSA validation. 

As before, the input script will also provide the signature, making it look like this: [signature] [public key]

Our new output script: OP_DUP OP_HASH160 [hash] OP_EQUALVERIFY OP_CHECKSIG

<pre>
   Run the input script:

   Stack            Exec      Remaining
  |             |  |            |   | [s] [p]            |
  |             |  |[s]         |   | [p]                | 
  |[s]          |  |[p]         |   |                    | 
  
    Now run the output script:  
  
  |[s][p]       |  |            |   | OP_DUP OP_HASH160 [hash] OP_EQUALVERIFY OP_CHECKSIG | 
  |[s][p]       |  | DUP        |   | OP_HASH160 [hash] OP_EQUALVERIFY OP_CHECKSIG        | 
  |[s][p][p]    |  | HASH160    |   | [hash] OP_EQUALVERIFY OP_CHECKSIG                   | 
  |[s][p][h1]   |  | [hash]     |   | OP_EQUALVERIFY OP_CHECKSIG                          |
  |[s][p][h1][h]|  | EQUALVERIFY|   | OP_CHECKSIG                                         |
  |[s][p]       |  | CHECKSIG   |   |                                                     |
  | true        |  |            |   |                                                     |
</pre>  

### Why did we change?

Pure public keys are cumbersome to communicate, whereas addresses are roughly fixed size. 

Two protection layers exist when addresses are used. Should ECDSA be broken, two different hashing algorithms must also be broken to learn the private key. 

### What's next?

The bitcoin script language is full of opportunities for interesting contracts. 

One of the first interesting contract was multi-signature accounts, made possible by OP_CHECKMULTISIG. 

This opcode requires at least 3 parameters:

 * the number of required signatures (must be less than the total number of keys)
 * a list of public keys
 * the number of public keys
 * a list of signatures
 * a null byte, due to a bug. 
 
 OP_CHECKMULTISIG allows up to 16 public keys.
 
A 'bare' multisignature script looks like this:

<pre>
{
    "txid": "f4184fc596403b9d638783cf57adfe4c75c605f6356fbc91338530e9831e9e16",
    "nVersion": 1,
    "nLockTime": 0,
    "inputs": [
      ...
    ],
    "outputs": [
      {
        "value": 1000000000,
        "scriptPubKey": "OP_2 [pubkey1] [pubkey2] [pubkey3] OP_3 OP_CHECKMULTISIG"
      }
    ]
}
</pre>

A transaction redeeming one of these outputs might look like: 

<pre>
{
    "txid": "ea44e97271691990157559d0bdd9959e02790c34db6c006d779e82fa5aee708e",
    "nVersion": 1,
    "nLockTime": 0,
    "inputs": [
      {
        "hashPrevOut": "f4184fc596403b9d638783cf57adfe4c75c605f6356fbc91338530e9831e9e16",
        "nPrevOut": 0,
        "scriptSig": "OP_0 [sig1] [sig2]"
        "nSequence": 0xffffffff
      }
    ]
    ...outputs trimmed...
}
</pre>

This contract allows _any_ 2 of the _3_ participants to produce signatures for a transaction,
without the participation of the third party. This is analogous to escrow, and has been implemented as such. 

### Pay to script hash

Pay to script hash is a development that has lead to a greater proliferation in complex bitcoin scripts. 

The issue was essentially: it's hard to ask someone to pay toa multi-signature script if the script is really long.
 
Instead of asking the funding party to pay to the multi-signature script, they can instead use _pay-to-script-hash_. This allows for the hashes of scripts to be embedded in an address, for the same benefits as not exchanging raw public keys. 

The virtues are: 
 * the funding party doesn't have to pay the fee for the large output script
 * the funding party doesn't have to know the details of the script
 * pay to script-hash masks some of the complexities of bitcoin scripts. 
 * only the redeeming party needs to know about the script, meaning there is some privacy (only the hash is revealed)
 
NB: Pay-to-script-hash is a soft-fork feature. OP_HASH160 [hash] OP_EQUAL implicitly means P2SH by soft-fork. 
 
### The Output Script

<pre>
 OP_2 [pubkey1] [pubkey2] [pubkey3] OP_3 OP_CHECKMULTISIG
   becomes
 OP_HASH160 [hash] OP_EQUAL
</pre>
 
 The output script checks the requirement that the HASH160 of some data equals the specified hash.
 
### The Input Script

<pre>
 OP_0 [sig1] [sig2] 
   becomes
 OP_0 [sig1] [sig2] [serialized script]
</pre>
  
  The input script needs to provide the same information as before (the signatures, and the dummy byte) but it 
  also needs to provide the raw script containing our public keys. This is similar to specifying the public key 
  in a pay-to-pubkey-hash redemption script, since only it's hash in the output script.
  
  The only requirements for P2SH are that the script is provided. P2SH does not mandate multisignature scripts, it's
  just an example!
  
### Altogether:

 * First the input scripts are run (no change)

<pre>
 Run: OP_0 [sig1] [sig2] [serialized script]
 Stack: [00] [sig1] [sig2] [serialized script]
</pre>
 
 * Now the output script (no change)

<pre>
 Run: OP_HASH160 [hash] OP_EQUAL
 Stack: [00] [sig1] [sig2] 
</pre>
 
 * If still executing, rerun the input scripts in a clean stack:

<pre>
 Stack: [00] [sig1] [sig2]
</pre>
 
 * Before finally running the serialized script:

<pre>
 Stack: true
</pre> 

### What else is possible?

The interpreter doesn't change much, but it's still under development. 

New opcodes are being added by soft-fork, adding interesting functionality. 

These new opcodes are repurposed NOP opcodes. For compatibility, OP_DROP needs to be called because NOP's never changed the value on the stack.

### OP_CHECKLOCKTIMEVERIFY (OP_HODL)

This allows bitcoins to be marked unspendable _until some point in the future_.

 A pseudo example:

<pre>
   IF [now + 3 months] CHECKLOCKTIMEVERIFY
      do something
   ELSE
      do something else
</pre>
      
 Previously, it was not possible to provably lock funds into a contract, meaning collusion against a party is possible.
  
<pre>
  IF
      [now + 3 months] CHECKLOCKTIMEVERIFY DROP
      [Lenny's pubkey] CHECKSIGVERIFY
      1
  ELSE
      2
  ENDIF
  [Alice's pubkey] [Bob's pubkey] 2 CHECKMULTISIG
</pre>

 This is an escrow contract between Alice and Bob, with Lenny as arbitrator:
 
 * Alice and Bob can release the coins at any time
 * Lenny, and either Alice or Bob access only after 3 months (should a dispute prevent the contract from being resolved)
 * Depending on which case is desired, a boolean needs to be pushed at the end of the scriptSig, to trigger the IF / ELSE code.
  
### OP_CHECKSEQUENCEVERIFY

This opcode is a _relative_ locktime, so the transaction is locked until a certain time has passed since
it confirms in the blockchain.

<pre>
   IF
      2 [Alice's pubkey] [Bob's pubkey] [Escrow's pubkey] 3 CHECKMULTISIGVERIFY
   ELSE
      "30d" CHECKSEQUENCEVERIFY DROP
      [Alice's pubkey] CHECKSIGVERIFY
   ENDIF
</pre>

This is a 2-of-3 escrow, with an automatic refund to Alice should 30 days occur. Any other time, any two parties can sign to releasea the funds. 

Packaging this into a pay-to-script-hash address, Alice can make a payment, automatically starting the refund countdown.

