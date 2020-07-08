# 7.2: Using a Partially Signed Bitcoin Transaction

> :information_source: **NOTE:** This is a draft in progress, so that I can get some feedback from early reviewers. It is not yet ready for learning.

So you've learned the basic workflow of generating a PSBT, but now you want to do something with it. What can PSBTs do that multi-sigs (and normal raw transactions) can't? To start with, you've got the ease of use of a standardized format, which means that you can use your `bitcoin-cli` transactions and meld them with transactions generated by people (or programs) on other platforms. Beyond that, you can do some things that just aren't possible using other mechanics.

Following are three examples, using PSBTs for: multi-sigs, pooling money, and joining coins.

> :warning: **VERSION WARNING:** This is an innovation from Bitcoin Core v 0.17.0. Earlier versions of Bitcoin Core will not be able to work with the PSBT while it is in process (though they will still be able to recognize the final transaction).

## Use a PSBT to MultiSig

Assume you've created a new multi-sig, just like you did in [§6.3](06_3_Sending_an_Automated_Multisig.md). 
```
$ bitcoin-cli -named addmultisigaddress nrequired=2 keys='''["'$address3'","$address4"]'''
{
  "address": "tb1q92a46jww0e6ne066nlagekhczkl3qa84czl5jk5nm78t2yf0vk4qte328m",
  "redeemScript": "5221033055ec2da9bbb34c2acb343692bfbecdef8fab8d114f0036eba01baec3888aa02103f52980d322acaf084bcef3216f3d84bfb672d1db26ce2861de3ec047bede140d52ae",
  "descriptor": "wsh(multi(2,[d6043800/0'/0'/23']033055ec2da9bbb34c2acb343692bfbecdef8fab8d114f0036eba01baec3888aa0,[ff1bf3b1]03f52980d322acaf084bcef3216f3d84bfb672d1db26ce2861de3ec047bede140d))#njkklz9u"
}
bitcoin-cli -named importaddress address="tb1q92a46jww0e6ne066nlagekhczkl3qa84czl5jk5nm78t2yf0vk4qte328m" rescan="false"
```
And, you've got some money in it:
```
$ bitcoin-cli listunspent
[
  {
    "txid": "25e8a26f60cf485768a1e6953b983675c867b7ab126b02e753c47b7db0c4be5e",
    "vout": 0,
    "address": "tb1q92a46jww0e6ne066nlagekhczkl3qa84czl5jk5nm78t2yf0vk4qte328m",
    "label": "",
    "witnessScript": "5221033055ec2da9bbb34c2acb343692bfbecdef8fab8d114f0036eba01baec3888aa02103f52980d322acaf084bcef3216f3d84bfb672d1db26ce2861de3ec047bede140d52ae",
    "scriptPubKey": "00202abb5d49ce7e753cbf5a9ffa8cdaf815bf1074f5c0bf495a93df8eb5112f65aa",
    "amount": 0.01000000,
    "confirmations": 5,
    "spendable": false,
    "solvable": true,
    "desc": "wsh(multi(2,[c1fdfe64]033055ec2da9bbb34c2acb343692bfbecdef8fab8d114f0036eba01baec3888aa0,[d3ed8825/0'/0'/2']03f52980d322acaf084bcef3216f3d84bfb672d1db26ce2861de3ec047bede140d))#5070hfm8",
    "safe": true
  }
]
```
You could use the same mechanisms you did in [§6.2](06_2_Spending_a_Transaction_to_a_Multisig.md), where you serially signed a transaction, but instead we're going to show the advantage of PSBTs for multi-sigs: you can generate a single PSBT, allow everyone to sign that, and then combine the results! There's no more laboriously passing an ever-expanding hex from person to person, which speeds things up and reduces the chances of errors.

TO demonstrate this methodology, we're going to pull that 0.01 BTC out of the multi-sig and divide it between the two signers, who each generated a new address for that purpose:
```
machine1$ bitcoin-cli getnewaddress
tb1qem5l3q5g5h6fsqv352xh4cy07kzq2rd8gphqma
machine2$ bitcoin-cli getnewaddress
tb1q3krplahg4ncu523m8h2eephjazs2hf6ur8r6zp
```
The first thing we do is create a PSBT on either machine. We need to use `createpsbt` from [§7.1](07_1_Creating_a_Partially_Signed_Bitcoin_Transaction.md), not the simpler `walletcreatefundedpsbt`, because we need the extra control of selecting our money protected by the multi-sig
```
machine2$ utxo_txid=25e8a26f60cf485768a1e6953b983675c867b7ab126b02e753c47b7db0c4be5e
machine2$ utxo_vout=0
machine2$ split1=tb1qem5l3q5g5h6fsqv352xh4cy07kzq2rd8gphqma
machine2$ split2=tb1q3krplahg4ncu523m8h2eephjazs2hf6ur8r6zp
machine2$ psbt=$(bitcoin-cli -named createpsbt inputs='''[ { "txid": "'$utxo_txid'", "vout": '$utxo_vout' } ]''' outputs='''{ "'$split1'": 0.004999,"'$split2'": 0.0004999 }''')
```
You then need to send that $psbt to everyone signing:
```
machine2$ echo $psbt
cHNidP8BAHECAAAAAV6+xLB9e8RT5wJrEqu3Z8h1Npg7leahaFdIz2BvouglAAAAAAD/////ArygBwAAAAAAFgAUzun4goil9JgBkaKNeuCP9YQFDadGwwAAAAAAABYAFI2GH/borPHKKjs91ZyG8uigq6dcAAAAAAAAAAA=
```
Just once! And you do it simulataneously.

Here's the result on the second machine, where I generated the PSBT:
```
machine2$ psbt_p1=$(bitcoin-cli walletprocesspsbt $psbt | jq -r '.psbt')
machine2$ bitcoin-cli decodepsbt $psbt_p1
{
  "tx": {
    "txid": "1687e89fcb9dd3067f75495b4884dc1d4d1cf05a6c272b783cfe29eb5d22e985",
    "hash": "1687e89fcb9dd3067f75495b4884dc1d4d1cf05a6c272b783cfe29eb5d22e985",
    "version": 2,
    "size": 113,
    "vsize": 113,
    "weight": 452,
    "locktime": 0,
    "vin": [
      {
        "txid": "25e8a26f60cf485768a1e6953b983675c867b7ab126b02e753c47b7db0c4be5e",
        "vout": 0,
        "scriptSig": {
          "asm": "",
          "hex": ""
        },
        "sequence": 4294967295
      }
    ],
    "vout": [
      {
        "value": 0.00499900,
        "n": 0,
        "scriptPubKey": {
          "asm": "0 cee9f88288a5f4980191a28d7ae08ff584050da7",
          "hex": "0014cee9f88288a5f4980191a28d7ae08ff584050da7",
          "reqSigs": 1,
          "type": "witness_v0_keyhash",
          "addresses": [
            "tb1qem5l3q5g5h6fsqv352xh4cy07kzq2rd8gphqma"
          ]
        }
      },
      {
        "value": 0.00049990,
        "n": 1,
        "scriptPubKey": {
          "asm": "0 8d861ff6e8acf1ca2a3b3dd59c86f2e8a0aba75c",
          "hex": "00148d861ff6e8acf1ca2a3b3dd59c86f2e8a0aba75c",
          "reqSigs": 1,
          "type": "witness_v0_keyhash",
          "addresses": [
            "tb1q3krplahg4ncu523m8h2eephjazs2hf6ur8r6zp"
          ]
        }
      }
    ]
  },
  "unknown": {
  },
  "inputs": [
    {
      "witness_utxo": {
        "amount": 0.01000000,
        "scriptPubKey": {
          "asm": "0 2abb5d49ce7e753cbf5a9ffa8cdaf815bf1074f5c0bf495a93df8eb5112f65aa",
          "hex": "00202abb5d49ce7e753cbf5a9ffa8cdaf815bf1074f5c0bf495a93df8eb5112f65aa",
          "type": "witness_v0_scripthash",
          "address": "tb1q92a46jww0e6ne066nlagekhczkl3qa84czl5jk5nm78t2yf0vk4qte328m"
        }
      },
      "partial_signatures": {
        "03f52980d322acaf084bcef3216f3d84bfb672d1db26ce2861de3ec047bede140d": "304402203abb95d1965e4cea630a8b4890456d56698ff2dd5544cb79303cc28cb011cbb40220701faa927f8a19ca79b09d35c78d8d0a2187872117d9308805f7a896b07733f901"
      },
      "witness_script": {
        "asm": "2 033055ec2da9bbb34c2acb343692bfbecdef8fab8d114f0036eba01baec3888aa0 03f52980d322acaf084bcef3216f3d84bfb672d1db26ce2861de3ec047bede140d 2 OP_CHECKMULTISIG",
        "hex": "5221033055ec2da9bbb34c2acb343692bfbecdef8fab8d114f0036eba01baec3888aa02103f52980d322acaf084bcef3216f3d84bfb672d1db26ce2861de3ec047bede140d52ae",
        "type": "multisig"
      },
      "bip32_derivs": [
        {
          "pubkey": "033055ec2da9bbb34c2acb343692bfbecdef8fab8d114f0036eba01baec3888aa0",
          "master_fingerprint": "c1fdfe64",
          "path": "m"
        },
        {
          "pubkey": "03f52980d322acaf084bcef3216f3d84bfb672d1db26ce2861de3ec047bede140d",
          "master_fingerprint": "d3ed8825",
          "path": "m/0'/0'/2'"
        }
      ]
    }
  ],
  "outputs": [
    {
    },
    {
      "bip32_derivs": [
        {
          "pubkey": "0349cc43324f7ad94bb407a9bf12bc50afd9e7b430a472572f1b63cb555034f52a",
          "master_fingerprint": "d3ed8825",
          "path": "m/0'/0'/3'"
        }
      ]
    }
  ],
  "fee": 0.00450110
}
$ bitcoin-cli analyzepsbt $psbt_p1
{
  "inputs": [
    {
      "has_utxo": true,
      "is_final": false,
      "next": "signer",
      "missing": {
        "signatures": [
          "c1fdfe6454aade5ffea5539a02e51b3418cf5416"
        ]
      }
    }
  ],
  "estimated_vsize": 168,
  "estimated_feerate": 0.02679226,
  "fee": 0.00450110,
  "next": "signer"
}
```
We can see that even though the UTXO information has been imported, and even though we have a _partial signature_, the signing of the single input is still not complete.

Here's the same thing on the other machine:
```
machine1$ psbt=cHNidP8BAHECAAAAAV6+xLB9e8RT5wJrEqu3Z8h1Npg7leahaFdIz2BvouglAAAAAAD/////ArygBwAAAAAAFgAUzun4goil9JgBkaKNeuCP9YQFDadGwwAAAAAAABYAFI2GH/borPHKKjs91ZyG8uigq6dcAAAAAAAAAAA=
machine1$ psbt_p2=$(bitcoin-cli walletprocesspsbt $psbt | jq -r '.psbt')
machine1$ echo $psbt_p2
cHNidP8BAHECAAAAAV6+xLB9e8RT5wJrEqu3Z8h1Npg7leahaFdIz2BvouglAAAAAAD/////ArygBwAAAAAAFgAUzun4goil9JgBkaKNeuCP9YQFDadGwwAAAAAAABYAFI2GH/borPHKKjs91ZyG8uigq6dcAAAAAAABAStAQg8AAAAAACIAICq7XUnOfnU8v1qf+oza+BW/EHT1wL9JWpPfjrURL2WqIgIDMFXsLam7s0wqyzQ2kr++ze+Pq40RTwA266AbrsOIiqBHMEQCIHpLvl8fmC+Qc6sjuHKjnmpZpIAqXnqrd/ZiG/WKMC7ZAiBI9tptilK6G6+uc3ZFMAeeXoIhwxw7mjfHVdrKdkUaogEBBUdSIQMwVewtqbuzTCrLNDaSv77N74+rjRFPADbroBuuw4iKoCED9SmA0yKsrwhLzvMhbz2Ev7Zy0dsmzihh3j7AR77eFA1SriIGAzBV7C2pu7NMKss0NpK/vs3vj6uNEU8ANuugG67DiIqgENYEOAAAAACAAAAAgBcAAIAiBgP1KYDTIqyvCEvO8yFvPYS/tnLR2ybOKGHePsBHvt4UDQT/G/OxACICAvziYIVFLQerxjvTict9uphx55u+zQgDkpEia+gjKpAAENYEOAAAAACAAAAAgBgAAIAAAA==
```
Now note that the way we managed this multi-sig was that we generated a PSBT with the correct UTXOs, where data was held in the wallets of both systems, then we allowed both of the users to process that PSBT on their own, adding UTXOs and signatures. As a result, we have two PSBTs each of which contain one signature and not the other. That wouldn't work in the classic multi-sig scenario, because all the signatures have to be sequential. Here, instead, we can make use of the Combiner role to mush those together.

We again go to either machine, and make sure we have both PSBTs in variables, then we combine them:
```
machine2$ psbt_p2="cHNidP8BAHECAAAAAV6+xLB9e8RT5wJrEqu3Z8h1Npg7leahaFdIz2BvouglAAAAAAD/////ArygBwAAAAAAFgAUzun4goil9JgBkaKNeuCP9YQFDadGwwAAAAAAABYAFI2GH/borPHKKjs91ZyG8uigq6dcAAAAAAABAStAQg8AAAAAACIAICq7XUnOfnU8v1qf+oza+BW/EHT1wL9JWpPfjrURL2WqIgIDMFXsLam7s0wqyzQ2kr++ze+Pq40RTwA266AbrsOIiqBHMEQCIHpLvl8fmC+Qc6sjuHKjnmpZpIAqXnqrd/ZiG/WKMC7ZAiBI9tptilK6G6+uc3ZFMAeeXoIhwxw7mjfHVdrKdkUaogEBBUdSIQMwVewtqbuzTCrLNDaSv77N74+rjRFPADbroBuuw4iKoCED9SmA0yKsrwhLzvMhbz2Ev7Zy0dsmzihh3j7AR77eFA1SriIGAzBV7C2pu7NMKss0NpK/vs3vj6uNEU8ANuugG67DiIqgENYEOAAAAACAAAAAgBcAAIAiBgP1KYDTIqyvCEvO8yFvPYS/tnLR2ybOKGHeP
machine2$ psbt_c=$(bitcoin-cli combinepsbt '''["'$psbt_p1'", "'$psbt_p2'"]''')
machine2$ bitcoin-cli decodepsbt $psbt_c
{
  "tx": {
    "txid": "1687e89fcb9dd3067f75495b4884dc1d4d1cf05a6c272b783cfe29eb5d22e985",
    "hash": "1687e89fcb9dd3067f75495b4884dc1d4d1cf05a6c272b783cfe29eb5d22e985",
    "version": 2,
    "size": 113,
    "vsize": 113,
    "weight": 452,
    "locktime": 0,
    "vin": [
      {
        "txid": "25e8a26f60cf485768a1e6953b983675c867b7ab126b02e753c47b7db0c4be5e",
        "vout": 0,
        "scriptSig": {
          "asm": "",
          "hex": ""
        },
        "sequence": 4294967295
      }
    ],
    "vout": [
      {
        "value": 0.00499900,
        "n": 0,
        "scriptPubKey": {
          "asm": "0 cee9f88288a5f4980191a28d7ae08ff584050da7",
          "hex": "0014cee9f88288a5f4980191a28d7ae08ff584050da7",
          "reqSigs": 1,
          "type": "witness_v0_keyhash",
          "addresses": [
            "tb1qem5l3q5g5h6fsqv352xh4cy07kzq2rd8gphqma"
          ]
        }
      },
      {
        "value": 0.00049990,
        "n": 1,
        "scriptPubKey": {
          "asm": "0 8d861ff6e8acf1ca2a3b3dd59c86f2e8a0aba75c",
          "hex": "00148d861ff6e8acf1ca2a3b3dd59c86f2e8a0aba75c",
          "reqSigs": 1,
          "type": "witness_v0_keyhash",
          "addresses": [
            "tb1q3krplahg4ncu523m8h2eephjazs2hf6ur8r6zp"
          ]
        }
      }
    ]
  },
  "unknown": {
  },
  "inputs": [
    {
      "witness_utxo": {
        "amount": 0.01000000,
        "scriptPubKey": {
          "asm": "0 2abb5d49ce7e753cbf5a9ffa8cdaf815bf1074f5c0bf495a93df8eb5112f65aa",
          "hex": "00202abb5d49ce7e753cbf5a9ffa8cdaf815bf1074f5c0bf495a93df8eb5112f65aa",
          "type": "witness_v0_scripthash",
          "address": "tb1q92a46jww0e6ne066nlagekhczkl3qa84czl5jk5nm78t2yf0vk4qte328m"
        }
      },
      "partial_signatures": {
        "033055ec2da9bbb34c2acb343692bfbecdef8fab8d114f0036eba01baec3888aa0": "304402207a4bbe5f1f982f9073ab23b872a39e6a59a4802a5e7aab77f6621bf58a302ed9022048f6da6d8a52ba1bafae73764530079e5e8221c31c3b9a37c755daca76451aa201",
        "03f52980d322acaf084bcef3216f3d84bfb672d1db26ce2861de3ec047bede140d": "304402203abb95d1965e4cea630a8b4890456d56698ff2dd5544cb79303cc28cb011cbb40220701faa927f8a19ca79b09d35c78d8d0a2187872117d9308805f7a896b07733f901"
      },
      "witness_script": {
        "asm": "2 033055ec2da9bbb34c2acb343692bfbecdef8fab8d114f0036eba01baec3888aa0 03f52980d322acaf084bcef3216f3d84bfb672d1db26ce2861de3ec047bede140d 2 OP_CHECKMULTISIG",
        "hex": "5221033055ec2da9bbb34c2acb343692bfbecdef8fab8d114f0036eba01baec3888aa02103f52980d322acaf084bcef3216f3d84bfb672d1db26ce2861de3ec047bede140d52ae",
        "type": "multisig"
      },
      "bip32_derivs": [
        {
          "pubkey": "033055ec2da9bbb34c2acb343692bfbecdef8fab8d114f0036eba01baec3888aa0",
          "master_fingerprint": "c1fdfe64",
          "path": "m"
        },
        {
          "pubkey": "03f52980d322acaf084bcef3216f3d84bfb672d1db26ce2861de3ec047bede140d",
          "master_fingerprint": "d3ed8825",
          "path": "m/0'/0'/2'"
        }
      ]
    }
  ],
  "outputs": [
    {
      "bip32_derivs": [
        {
          "pubkey": "02fce26085452d07abc63bd389cb7dba9871e79bbecd08039291226be8232a9000",
          "master_fingerprint": "d6043800",
          "path": "m/0'/0'/24'"
        }
      ]
    },
    {
      "bip32_derivs": [
        {
          "pubkey": "0349cc43324f7ad94bb407a9bf12bc50afd9e7b430a472572f1b63cb555034f52a",
          "master_fingerprint": "d3ed8825",
          "path": "m/0'/0'/3'"
        }
      ]
    }
  ],
  "fee": 0.00450110
}
machine2$ bitcoin-cli analyzepsbt $psbt_c
{
  "inputs": [
    {
      "has_utxo": true,
      "is_final": false,
      "next": "finalizer"
    }
  ],
  "estimated_vsize": 168,
  "estimated_feerate": 0.02679226,
  "fee": 0.00450110,
  "next": "finalizer"
}
```
It worked! We just finalize and send and we're done:
```
machine2$ psbt_c_hex=$(bitcoin-cli finalizepsbt $psbt_c | jq -r '.hex')
standup@btctest2:~$ bitcoin-cli -named sendrawtransaction hexstring=$psbt_c_hex
1687e89fcb9dd3067f75495b4884dc1d4d1cf05a6c272b783cfe29eb5d22e985
```
Obviously, there wasn't a big improvement in using this method over multiply signing a transaction for a 2-of-2 multisig when everyone was using `bitcoin-cli`. We could have passed a raw transaction with partial signatures from one user to the other just as easily as that PSBT. But there nonetheless are big advantages to this methodology. First of all, it's platform independent. As long as everyone is using a service that supports Bitcoin Core 0.17, they'll all be able to sign this transaction, which isn't true when multi-sigs are being passed around. But more notably, it's a lot more scalable. Consider a 3-of-5 multisig. Under the old methodology it would have to passed from person to person, greatly increasing the problems if any single link in the chain breaks. Here, other users just have to send the PSBTs back to the Creator, and as soon as she has enough, she can generate the final transaction.

## Use a PSBT to Pool Money

## Use a PSBT to CoinJoin

## Summary: Using a Partially Signed Bitcoin Transaction

> :fire: ***What's the power of a PSBT?***

## What's Next?

Move on to "Bitcoin Scripting" with [Chapter Seven: Introducing Bitcoin Scripts](07_0_Introducing_Bitcoin_Scripts.md).