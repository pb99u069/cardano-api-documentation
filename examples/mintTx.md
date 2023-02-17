# Building a minting Transaction

## example taken from PPP 0306

from the cardano-api perspective, there is no difference beween `writeValidator` (see payToScript.md) and `writeMintingPolicy` 

```haskell
writeMintingPolicy :: FilePath -> Plutus.MintingPolicy -> IO (Either (FileError ()) ())
writeMintingPolicy file = writeFileTextEnvelope @(PlutusScript PlutusScriptV1) file Nothing . PlutusScriptSerialised . SBS.toShort . LBS.toStrict . serialise . Plutus.getMintingPolicy
```

### getting the value to mint

first we need to calculate the policyId from the monetary policy script:

cardano-cli transaction policyid --script-file $policyFile

to make addresses from scripts, we use `hashScript`

```haskell
hashScript (PlutusScript PlutusScriptV1 (PlutusScriptSerialised script)) =
    -- For Plutus V1, we convert to the Alonzo-era version specifically and
    -- hash that. Later ledger eras have to be compatible anyway.
    ScriptHash
  . Ledger.hashScript @(ShelleyLedgerEra AlonzoEra)
  $ Alonzo.PlutusScript Alonzo.PlutusV1 script
```

note that this `ScriptHash` is separate from the 'Hash' type to avoid the script
hash type being parametrised by the era. The representation is era
independent, and there are many places where we want to use a script
hash where we don't want things to be era-parametrised.

then this is serialised with `serialiseToRawBytesHexText`, giving us the policyId

The TokenName, called `AssetName` on the cardano-api side, must be in hex form. so coming from the plutus side, we first need to convert it to the cardano-api type and then serialise it to hex bytes (and also unpack it as the cardano-cli expects a string):

```haskell
unsafeTokenNameToHex :: TokenName -> String
unsafeTokenNameToHex = BS8.unpack . serialiseToRawBytesHex . fromJust . deserialiseFromRawBytes AsAssetName . getByteString . unTokenName
  where
    getByteString (BuiltinByteString bs) = bs
```

having the policyId and tnHex, the value defined for the cardano-cli must be in the form of `amt policyId.tnHex`, where amt is the amount of the token we want to mint.

### Building the transaction

again, as for spending from a validator script, minting from a policy script is only possible with a script witness. 

more precisely, the value passed to `runTxBuild` in cardano-cli is of type `(Value, [ScriptWitness WitCtxMint era])`, which is somehow similar to the txIns field of the transaction body content, but instead of a list of pairs (txIn, script witness for that txIn), we have just 1 value containing all the tokens to mint plus a list of witnesses, this time with the context `WitCtxMint` instead of `WitCtxTxIn`

The number of script witnesses depends on the number of the poliyIds contained in the value, of course. In our case it's just 1 token to mint, so there will be 1 script witness, 

As for the datum needed for a script witness, we simply parse it, when in the `WitCtxMint`, to a value of `NoScriptDatumForMint`. The redeemer meanwhile is a type alias for `ScriptData` and thus context independent. Same as in spendFromScript, we use the unit.json redeemer.
the txMintValue of the transaction body content however expects a value of type `TxMintValue`:

```haskell
data TxMintValue build era where
     TxMintValue :: MultiAssetSupportedInEra era
                 -> Value
                 -> BuildTxWith build
                      (Map PolicyId (ScriptWitness WitCtxMint era))
                 -> TxMintValue build era
```

this is done with the `createTxMintValue` function.

Once we have the value for the `txIns` field of the transaction body content, we can proceed as in the `payToScript.md` example. Apart from running the minting script instead of the validator script, the balancing phase of the transaction building is identical, and same goes for signing and submitting the transaction.

```bash
cardano-cli transaction build \
    $MAGIC \
    --tx-in $oref \
    --tx-in-collateral $oref \
    --tx-out "$addr + 1500000 lovelace + $v" \
    --mint "$v" \
    --mint-script-file $policyFile \
    --mint-redeemer-file testnet/unit.json \
    --change-address $addr \
    --protocol-params-file $ppFile \
    --out-file $unsignedFile \

cardano-cli transaction sign \
    --tx-body-file $unsignedFile \
    --signing-key-file $skeyFile \
    $MAGIC \
    --out-file $signedFile

cardano-cli transaction submit \
    $MAGIC \
    --tx-file $signedFile
```