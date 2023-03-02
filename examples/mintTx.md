# Building a minting Transaction

This example taken from PPP0306

From the cardano-api perspective, there is no difference beween `writeValidator` (see payToScript.md) and `writeMintingPolicy` 

```haskell
writeMintingPolicy :: FilePath -> Plutus.MintingPolicy -> IO (Either (FileError ()) ())
writeMintingPolicy file = writeFileTextEnvelope @(PlutusScript PlutusScriptV1) file Nothing . PlutusScriptSerialised . SBS.toShort . LBS.toStrict . serialise . Plutus.getMintingPolicy
```

## getting the value to mint

First, we need to calculate the policyId from the monetary policy script. This can be done by providing the script file from above to `cardano-cli transaction policyid`. Under the hood, the script is hashed and serialised with the following two cardano-api functions:

`serialiseToRawBytesHexText $ hashScript script`

note: the `ScriptHash` obtained by the `hashScript` function is separate from the `Hash` type to avoid the script hash type being parametrised by the era. The representation is era independent, and there are many places where we want to use a script hash where we don't want things to be era-parametrised.

Next to the policy id, we need a token name to construct a value. The TokenName, called `AssetName` on the cardano-api side, must be in hex form. so coming from the plutus side, we first need to convert it to the cardano-api type and then serialise it to hex bytes (and also unpack it as the cardano-cli expects a string):

```haskell
unsafeTokenNameToHex :: TokenName -> String
unsafeTokenNameToHex = BS8.unpack . serialiseToRawBytesHex . fromJust . deserialiseFromRawBytes AsAssetName . getByteString . unTokenName
  where
    getByteString (BuiltinByteString bs) = bs
```

Having the policyId and tnHex, the value defined for the cardano-cli (see `--tx-out` and `--mint` below) must be in the form of `amt policyId.tnHex`, where amt is the amount of the token we want to mint.

## Building the transaction

Again, as for spending from a validator script, minting from a policy script is only possible with a script witness. 

More precisely, the mint value passed to `runTxBuild` in cardano-cli is of type `(Value, [ScriptWitness WitCtxMint era])`, which is somehow similar to the `txIns` field of the transaction body content, but instead of a list of pairs (txIn, witness), we have just 1 value containing all the tokens to mint plus a list of script witnesses, this time with the context `WitCtxMint` instead of `WitCtxTxIn`

The number of script witnesses depends on the number of the poliyIds contained in the value, of course. In our case it's just 1 token to mint, so there will be 1 script witness. 

As for the datum needed for a script witness, we simply parse it, when in the `WitCtxMint`, to a value of `NoScriptDatumForMint`. The redeemer meanwhile is a type alias for `ScriptData` and thus context independent. As in the `spendFromScript` example, we use the unit.json redeemer.

The cardano-cli function `createTxMintValue` finally produces a value of the correct type (see below) which we need for the `txMintValue` field of the transaction body content.

```haskell
data TxMintValue build era where
     TxMintValue :: MultiAssetSupportedInEra era
                 -> Value
                 -> BuildTxWith build
                      (Map PolicyId (ScriptWitness WitCtxMint era))
                 -> TxMintValue build era
```

Once we have the value for the `txMintValue` field of the transaction body content, we can proceed as in the `payToScript.md` example. Apart from running the minting script instead of the validator script, the balancing phase of the transaction building is identical, and same goes for signing and submitting the transaction.

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