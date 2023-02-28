# Building a spending from validator tx

This example is taken from PPP0303

```bash
cardano-cli transaction build \
    --alonzo-era \
    --testnet-magic 1097911063 \
    --change-address $(cat 02.addr) \
    --tx-in 18cbe6cadecd3f89b60e08e68e5e6c7d72d730aaa1ad21431590f7e6643438ef#1 \
    --tx-in-script-file vesting.plutus \
    --tx-in-datum-file unit.json \
    --tx-in-redeemer-file unit.json \
    --tx-in-collateral 18e93407ea137b6be63039fd3c564d4c5233e7eb7ce4ee845bc7df12c80e4df7#1 \
    --required-signer-hash c2ff616e11299d9094ce0a7eb5b7284b705147a822f4ffbd471f971a \
    --invalid-before 48866954 \
    --protocol-params-file protocol.json \
    --out-file tx.body
```

While, to spend from a key address, we simply 'tag' the the txIn with `KeyWitness KeyWitnessForSpending`, tagging a `TxIn` that is to be spent from a script address means building a witness that contains the script itself.

The resulting value is a (`txIn, ScriptWitness ScriptWitnessForSpending plutusScriptWitness`), where `plutusScriptWitness` contains the script itself, as well as the datum, redeemer and the script language and script version information.

When using the `cardano-cli`, these values are provided with `tx-in-script-file`, `tx-in-datum-file` and `tx-in-redeemer-file`. The script gets parsed by the `cardano-cli` into a type `ScriptInAnyLang` which contains the script (of type `PlutusScript PlutusScriptV1` in our case).
The `PlutusScriptOrReferenceInput lang` type, needed for the script witness below, can then be built with the data constructor `PScript` and the sript value.

The ScriptDatum gets built with `ScriptData` and the `ScriptDatumForTxIn` data constructor while `ScriptRedeemer` is simply a type alias for `ScriptData`.

```haskell
-- building a script witness
data ScriptWitness witctx era where
     PlutusScriptWitness :: ScriptLanguageInEra  lang era
                         -> PlutusScriptVersion  lang
                         -> PlutusScriptOrReferenceInput lang
                         -> ScriptDatum witctx
                         -> ScriptRedeemer
                         -> ExecutionUnits
                         -> ScriptWitness witctx era

-- building the witness with the sript witness
data Witness witctx era where
     ScriptWitness :: ScriptWitnessInCtx witctx
                   -> ScriptWitness      witctx era
                   -> Witness            witctx era
```

Building the txOut is straightforward in this example, as the output goes to a key address.

The `--requiredSignerHash` input gets parsed into a a value for the `txExtraKeyWits` field of the transaction body content. Extra key witnesses visible to scripts are supported from the Alonzo era onwards.
In our case we need this because the script checks that the transaction is signed by a certain party.

As this transaction requires the running of a script to spend the txIn, the `makeTransactionAutoBalance` has to do some heavier work as in the two previous examples. 

first, as before, make a TxBody, then run `evaluateTransactionExecutionUnits`. Under the hood, this function runs the script with `evaluateScriptRestricting` from the [plutus-ledger-api](https://github.com/input-output-hk/plutus/blob/master/plutus-ledger-api/src/PlutusLedgerApi/Common/Eval.hs)

finally, the new value is inserted into the witness with `substituteExecutionUnits`, which lets us then calculate the txFee with the `evaluateTransactionFee` function.

so what is left to do is calculating the balance going to the change address. This is done with the `evaluateTransactionBalance` function, and with the resulting value we can then create the corresponding TxOut for the transaction body content: 

`TxOut changeaddr balance TxOutDatumNone ReferenceScriptNone`

having done all this, we can build the final transaction body content by updating the `txFee` and `txOuts` fields and apply one last time the `createAndValidateTransactionBody` function. 

the result is:

`BalancedTxBody finalTxBodyContent txbody3 (TxOut changeaddr balance TxOutDatumNone ReferenceScriptNone) fee`

### signing and submitting

we need the skey for the collateral, the rest is identical to signing and submitting the simple transaction.

```bash
cardano-cli transaction build \
    --alonzo-era \
    --testnet-magic 1097911063 \
    --change-address $(cat 02.addr) \
    --tx-in 18cbe6cadecd3f89b60e08e68e5e6c7d72d730aaa1ad21431590f7e6643438ef#1 \
    --tx-in-script-file vesting.plutus \
    --tx-in-datum-file unit.json \
    --tx-in-redeemer-file unit.json \
    --tx-in-collateral 18e93407ea137b6be63039fd3c564d4c5233e7eb7ce4ee845bc7df12c80e4df7#1 \
    --required-signer-hash c2ff616e11299d9094ce0a7eb5b7284b705147a822f4ffbd471f971a \
    --invalid-before 48866954 \
    --protocol-params-file protocol.json \
    --out-file tx.body

cardano-cli transaction sign \
    --tx-body-file tx.body \
    --signing-key-file 02.skey \
    --testnet-magic 1097911063 \
    --out-file tx.signed

cardano-cli transaction submit \
    --testnet-magic 1097911063 \
    --tx-file tx.signed
```
