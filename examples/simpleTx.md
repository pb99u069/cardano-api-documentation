# Building a simple Transaction

the following example is taken from the PPP0303

## Generate keys

First, we need to create a key pair of type `VerificationKey PaymentKey` and `SigningKey PaymentKey`. To get the singing key, the `cardano-cli` command `address key-gen` uses the cardano-api function `generateSigningKey :: Key keyrole => AsType keyrole -> IO (SigningKey keyrole)` under the hood.

`keyrole` can be a `ByronKey`, `PaymentKey` (by default) or a `PaymentExtendedKey`, resulting in `AsByronKey`, `AsPaymentKey` or `AsPaymentExtendedKey` values for `AsType keyrole`.

The functions for the cryptographic key are coming from the cardano-crypto-class package, for PaymentKeys we use the public-key signature system Ed25519, Ed25519DSIGN being an instance of the [class DSIGNAlgorithm](https://github.com/input-output-hk/cardano-base/blob/dd60865a18478aa7f2936693da952cd22d76b080/cardano-crypto-class/src/Cardano/Crypto/DSIGN/Ed25519.hs)

having built a signing key, cardano-cli uses the `getVerificationKey :: SigningKey PaymentKey -> VerificationKey PaymentKey` function which is defined in the `Key` class of which `PaymentKey` is an instance. 

```haskell
-- cardano-cli
generateKeyPair :: Key keyrole => AsType keyrole -> IO (VerificationKey keyrole, SigningKey keyrole)
generateKeyPair asType = do
  skey <- generateSigningKey asType
  return (getVerificationKey skey, skey)
```

Having generated the pair, the cardano-cli serialises each key and writes it to a file, using `writeFileTextEnvelope` from the cardano-api.

The serialisation to CBOR is done inside the `serialiseToTextEnvelope` function, so a type that is an instance of the `HasTextEnvelope` class must also be an instance of the `SerialiseAsCBOR` class, which is the case for both `SigningKey PaymentKey` and `VerificationKey PaymentKey`.

```haskell
serialiseToTextEnvelope :: forall a. HasTextEnvelope a => Maybe TextEnvelopeDescr -> a -> TextEnvelope
serialiseToTextEnvelope mbDescr a =
    TextEnvelope {
      teType    = textEnvelopeType ttoken
    , teDescription   = fromMaybe (textEnvelopeDefaultDescr a) mbDescr
    , teRawCBOR = serialiseToCBOR a
    }
  where
    ttoken :: AsType a
    ttoken = proxyToAsType Proxy
```

## Generate an address

The `cardano-cli` command `address build` works for both key and script addresses.

First, the `cardano-cli` function `runAddressBuild` checks if the argument provided is a `PaymentVerifierKey` or a `PaymentVerifierScript`. In the case of a key, it builds a value of type `AddressAny` with `AddressShelley <$> buildShelleyAddress (castVerificationKey vk) mbStakeVerifier nw` where nw is the networkId and mbStakeVerifier an optional stake key. `buildShelleyAddress` in turn uses `makeShelleyAddress` from the cardano-api under the hood.

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

In case the argument to `runAddressBuild` is a script, the `PaymentCredential` value for `makeShelleyAddress` is built with the `PaymentCredentialByScript` data constructor, which takes the hash of the script as argument. 

## A simple transaction to a key address

To build a simple transaction to a key address (as opposed to a slightly more complex transaction to a script address), the cardano-cli tool needs only a few parameters:
 
```bash
cardano-cli transaction build \
    --alonzo-era \
    --testnet-magic 1 \
    --change-address $(cat 01.addr) \
    --tx-in dfc1a522cd34fe723a0e89f68ed43a520fd218e20d8e5705b120d2cedc7f45ad#0 \
    --tx-out "$(cat 02.addr) 10000000 lovelace" \
    --out-file tx.body
```

with these values, we can build the two fields `txIns` and `txOuts` of the cardano-api `TxBodyContent`. 

The `--tx-in` input gets parsed to a `(TxIn, Maybe (ScriptWitness WitCtxTxIn era))`. As this input is spent from a key address, we don't need a script witness, and the value is thus `(txIn, Nothing)`. 
Of course, a witness is also needed to spend funds from a key address, and so inside the `runTxBuild` function, the `(tx-in, Nothing)` gets converted into a value of `(txin, BuildTxWith $ KeyWitness KeyWitnessForSpending)`. This is the format that the cardano-api type `TxBodyContent` excpects for its `txIns` field.

The `--tx-out` input meanwhile gets parsed to a `TxOut CtxTx era`, which is already the format the `TxBodyContent` excpects for its `tx-outs` field. 

```haskell
data TxOut ctx era = TxOut (AddressInEra    era)
                           (TxOutValue      era)
                           (TxOutDatum ctx  era)
                           (ReferenceScript era)
```

As we don't need a datum for an output to a key address, the `TxOutDatum` is of value TxOutDatumNone. We neither have a `ReferenceScript`, so its value is `ReferenceScriptNone`.

The `cardano-cli` can then proceed to build a `BalancedTxBody era` with `runTxBuild`.

This `runTxBuild` contains basically two big steps. The first: constructing the cardano-api `txBodyContent`. As already described above, we have the two fields `txIns` and `txOuts`. Most of the other fields are of no importance for a simple transaction and get 'none' values, like for example `TxInsCollateralNone :: TxInsCollateral era`
The one field lacking a value at this point is `txFee`. Since the transaction hasn't been balanced yet - which is the second big step inside `runTxBuild` - it is set to 0 at this point.

Balancing the transaction is done with `makeTransactionBodyAutoBalance` (see `Balancing the transaction` in `spendFromScript.md` for details). To use this function we need a running node to query the `nodeEra`, to which we then can apply the queryStateForBalancedTx function. Having thus all the necessary node information, `makeTransactionBodyAutoBalance` finally produces a BalancedTxBody.
     
```haskell
data BalancedTxBody era
  = BalancedTxBody
      (TxBodyContent BuildTx era)
      (TxBody era)
      (TxOut CtxTx era) -- ^ Transaction balance (change output)
      Lovelace          -- ^ Estimated transaction fee
```

## Sign a simple transaction

Signing a transaction means basically just providing the required witnesses. In our case, to execute the `cardano-cli transaction sign` command we need to provide the balanced transaction body and a `SigningKey PaymentKey` (the one associated to the `VerificationKey PaymentKey` that was used to build the address where the funds are spent from) 

The key witness is calculated by applying the `makeShelleyKeyWitness` function to these two arguments. And signing the transaction is then straightforward:

`signedTx = makeSignedTransaction allKeyWits txbody`

where allKeyWits is, in our case, a list of just the one key witness.

under the hood, signing a transaction looks like this:

``` haskell
\wsk ->
  let sk        = toShelleySigningKey wsk
      vk        = getShelleyKeyWitnessVerificationKey sk
      signature = makeShelleySignature txhash sk
   in ShelleyKeyWitness era $ Shelley.WitVKey vk signature
```

where `wsk` is a `WitnessPaymentKey (SigningKey PaymentKey)` of type `ShelleyWitnessSigningKey` and `txhash` is the hash of the transaction body. The hash function used here, `hashAnnotated`, is coming from the [cardano-ledger](https://github.com/input-output-hk/cardano-ledger/blob/master/libs/cardano-ledger-core/src/Cardano/Ledger/SafeHash.hs)

Note: creating a witness can also be done separatly by providing a tx-body-file and a signing-key-file to the command `cardano-cli transaction witness`

Note: the module `Cardano.Api.Convenience.Construction` contains the `constructBalancedTx` function which does the balancing and signing all in one. And to get the required arguments from the node, the `Cardano.Api.Convenience.Query` module offers the `queryStateForBalancedTx` function.

## Submit a simple transaction

The core function to submit a transaction in the cardano-api is `submitTxToNodeLocal`. Instead of the `Tx` obtained from the signing step, this function expects a `TxInMode` though.

```haskell
data TxInMode mode where
     TxInMode :: Tx era -> EraInMode era mode -> TxInMode mode
```

As the `cardano-cli` parser defaults to the cardano consensus mode and as we declared our transaction to be in the alonzo era, our value of type `EraInMode era mode` is AlonzoEraInCardanoMode. 

```bash
cardano-cli transaction build \
    --alonzo-era \
    --testnet-magic 1097911063 \
    --change-address $(cat 01.addr) \
    --tx-in dfc1a522cd34fe723a0e89f68ed43a520fd218e20d8e5705b120d2cedc7f45ad#0 \
    --tx-out "$(cat 02.addr) 10000000 lovelace" \
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
