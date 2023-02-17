# Paying to a script validator

## example taken from PPP 0303

### Validator

first, we need to convert the plutus validator into a cardano-api type and write it to a file. The function below does this, using the `PlutusScriptSerialised` data constructor which takes as argument a `ShortByteString`

```haskell
writeValidator :: FilePath -> Ledger.Validator -> IO (Either (FileError()) ())
writeValidator file = writeFileTextEnvelope @(PlutusScript PlutusScriptV1) file Nothing . PlutusScriptSerialised . SBS.toShort . LBS.toStrict . serialise . Ledger.unValidatorScript
```

### ScriptData

cardano-api has its own `data` type. The conversion can be done with the cardano-api function `fromPlutusData :: Plutus.Data -> ScriptData`.
To use this cardano-api data in the cardano-cli, it is serialised to json and written to a file. `scriptDataToJson` works either with `ScriptDataJsonDetailedSchema` (used to write Json into a file) or with `ScriptDataJsonNoSchema` (if inlined in the command directly).
the following code shows how to convert and serialise the unit value.

```haskell
writeUnit :: IO ()
writeUnit = writeJSON "unit.json" ()

writeJSON :: PlutusTx.ToData a => FilePath -> a -> IO ()
writeJSON file = LBS.writeFile file . encode . scriptDataToJson ScriptDataJsonDetailedSchema . fromPlutusData . PlutusTx.toData
```

alternatively, we can use this [library](https://github.com/input-output-hk/plutus-apps/blob/main/plutus-ledger/src/Ledger/Tx/CardanoAPI.hs) as an interface to the transaction types from `cardano-api`, see hydra-demo.md example.

### building the transaction

Building a transaction with a payment to a script address brings two major changes. First, instead of a payment key hash, we need a script hash for our address. and second, we need to provide a datum hash.

so how do we get the address of a script? As for a key address, we can use `cardano-cli address build`, but this time instead of a `VerificationKey PaymentKey`, we must provide a `Script`

under the hood, `makeShelleyAddress` must be applied to a `PaymentCredential`, though. So cardano-api provides the `hashScript` function which converts our `PlutusScript PlutusScriptV1 (PlutusScriptSerialised script)` into a value of type `ScriptHash`.
applying the `PaymentCredentialByScript` data constructor to this script finally gives us the correct PaymentCredential

```haskell
data Script lang where

     SimpleScript :: !(SimpleScriptVersion lang)
                  -> !(SimpleScript lang)
                  -> Script lang

     PlutusScript :: !(PlutusScriptVersion lang)
                  -> !(PlutusScript lang)
                  -> Script lang
```

```haskell
makeShelleyAddress :: NetworkId
                   -> PaymentCredential
                   -> StakeAddressReference
                   -> Address ShelleyAddr
makeShelleyAddress nw pc scr =
    ShelleyAddress
      (toShelleyNetwork nw)
      (toShelleyPaymentCredential pc)
      (toShelleyStakeReference scr)
```

To get a `AddressInEra`, needed to build the txOut, we can use the `shelleyAddressInEra` function

To build a transaction for a payment to a script, we additionally need the datum, provided to `cardano-cli` via --tx-out-datum-hash-file in this case.

this file is read and parsed into a `TxOutDatumHash ScriptDataInAlonzoEra hash`, where hash is of type `Hash ScriptData`. As mentioned above, a value of type `ScriptData` can also be converted directly from a plutus type with `fromPlutusData` 

so, having the script payment credential and datum, building the TxOut is straightforward (using `ReferenceScriptNone` as `ReferenceScript era`):

```haskell
data TxOut ctx era = TxOut (AddressInEra    era)
                           (TxOutValue      era)
                           (TxOutDatum ctx  era)
                           (ReferenceScript era)
```

Other then the new TxOut, the transaction body content is built exactly in the same way as in `simple transaction` 

```bash
cardano-cli transaction build \
    --alonzo-era \
    --testnet-magic 1097911063 \
    --change-address $(cat 01.addr) \
    --tx-in abae0d0e19f75938537dc5e33252567ae3b1df1f35aafedd1402b6b9ccb7685a#0 \
    --tx-out "$(cat vesting.addr) 200000000 lovelace" \
    --tx-out-datum-hash-file unit.json \
    --out-file tx.body
```

once we have the TxBodyContent, we can again apply `makeTransactionBodyAutoBalance`, giving us a BalancedTxBody AlonzoEra  

and signing and submitting works the same way as in simpleTx.

```bash
cardano-cli transaction build \
    --alonzo-era \
    --testnet-magic 1097911063 \
    --change-address $(cat 01.addr) \
    --tx-in abae0d0e19f75938537dc5e33252567ae3b1df1f35aafedd1402b6b9ccb7685a#0 \
    --tx-out "$(cat vesting.addr) 200000000 lovelace" \
    --tx-out-datum-hash-file unit.json \
    --out-file tx.body

cardano-cli transaction sign \
    --tx-body-file tx.body \
    --signing-key-file 01.skey \
    --testnet-magic 1097911063 \
    --out-file tx.signed

cardano-cli transaction submit \
    --testnet-magic 1097911063 \
    --tx-file tx.signed
```
