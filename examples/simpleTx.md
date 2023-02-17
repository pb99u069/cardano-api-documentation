# Building a simple Transaction

## example taken from PPP 0303

the cardano-api is what the cardano-cli uses under the hood.

### generate keys

cardano-cli address key-gen

the command `cardano-cli address key-gen` uses `generateSigningKey :: Key keyrole => AsType keyrole -> IO (SigningKey keyrole)` and `getVerificationKey :: SigningKey PaymentKey -> VerificationKey PaymentKey` from the cardano-api under the hood

`keyrole` can be a `ByronKey`, `PaymentKey` or a `PaymentExtendedKey`, resulting in AsByronKey, AsPaymentKey or AsPaymentExtendedKey as argument
the functions for the cryptographic key are coming from the cardano-crypto-class package, for PaymentKeys we use the public-key signature system Ed25519, Ed25519DSIGN being an instance of the [class DSIGNAlgorithm](https://github.com/input-output-hk/cardano-base/blob/dd60865a18478aa7f2936693da952cd22d76b080/cardano-crypto-class/src/Cardano/Crypto/DSIGN/Ed25519.hs)

having a signing key, cardano-cli uses the `getVerificationKey` function which is defined in the `Key` class of which `PaymentKey` is an instance 

```haskell
-- cardano-cli
generateKeyPair :: Key keyrole => AsType keyrole -> IO (VerificationKey keyrole, SigningKey keyrole)
generateKeyPair asType = do
  skey <- generateSigningKey asType
  return (getVerificationKey skey, skey)
```
so this results in a (`VerificationKey PaymentKey`, `SigningKey PaymentKey`) pair

finally, cardano-cli serialises this and writes it to a file, whith `writeFileTextEnvelope` from the cardano-api

serialisation is done internally, that is why a type that is an instance of the `HasTextEnvelope` class must also be an instance of the `serialiseAsCBOR` class 

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

### generate address

the command `cardano-cli address build` works for both key and script addresses

the cardano-cli `runAddressBuild` function first checks if the argument provided is a `PaymentVerifierKey` or a `PaymentVerifierScript`
in the case of a PaymentKey, it builds a cardano-api value of type `AddressAny` with `AddressShelley <$> buildShelleyAddress (castVerificationKey vk) mbStakeVerifier nw` where nw is the networkId and mbStakeVerifier an optional stake key.
`buildShelleyAddress` runs `makeShelleyAddress` from cardano-api under the hood.
to serialise this address, cardano-api uses the underlying `SerialiseAddress` instance of the `Address ShelleyAddr`.

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

in case the argument to `runAddressBuild` is a script, then the script is first hashed and then used as argument for PaymentCredentialByScript data constructor

### build simple Tx to PubKeyAddress

To build a simple transaction to a PubKeyAddress, the cardano-cli tool needs only very few parameters:
 
```bash
cardano-cli transaction build \
    --alonzo-era \
    --testnet-magic 1 \
    --change-address $(cat 01.addr) \
    --tx-in dfc1a522cd34fe723a0e89f68ed43a520fd218e20d8e5705b120d2cedc7f45ad#0 \
    --tx-out "$(cat 02.addr) 10000000 lovelace" \
    --out-file tx.body
```

The tx-in gets parsed to a (TxIn, Maybe (ScriptWitness WitCtxTxIn era)). As this input is spent from a key address, it doesn't need a script witness, and its value is thus of (txIn, Nothing) 
of course a witness is also needed to spend funds from a key address, and so inside the `runTxBuild` function, the tx-in gets converted to a value of (txin, BuildTxWith $ KeyWitness KeyWitnessForSpending). This is the format that the cardano-api type `TxBodyContent` excpects for its tx-ins field.

the tx-out meanwhile gets parsed to a `TxOut CtxTx era`, which is already the format the `TxBodyContent` excpects for its tx-out field 

the change-address isn't needed at this point, but it will be later in the balancing process.

so, having the txBodyContent we can it builds a `BalancedTxBody era` with `runTxBuild`

this `runTxBuild` contains basically two big steps. The first: constructing the txBodyContent. As already described above, we have the two fields tx-ins and tx-out for this. Most of the other fields are of no importance for a simple transaction and get 'none' values, like for example `TxInsCollateralNone :: TxInsCollateral era`
The one field lacking a value at this point is tx-fee. Since the transaction hasn't been balanced yet - which is the second big step inside `runTxBuild` - it is set to 0 for now.

balancing the transaction is achieved with `makeTransactionBodyAutoBalance`

The second big step is balancing the transaction, which is done with `makeTransactionBodyAutoBalance`. to use this function we need a running node to query the `nodeEra`, to which we then can apply the queryStateForBalancedTx function. 
having all the necessary node information, makeTransactionBodyAutoBalance finally produces a BalancedTxBody.
     
```haskell
data BalancedTxBody era
  = BalancedTxBody
      (TxBodyContent BuildTx era)
      (TxBody era)
      (TxOut CtxTx era) -- ^ Transaction balance (change output)
      Lovelace    -- ^ Estimated transaction fee
```

### sign simple transaction

signing a transaction means basically just providing the required witnesses. In our case, to execute the `cardano-cli transaction sign` command we need to provide the balanced transaction body and a `SigningKey PaymentKey`

The key witness is calculated by applying the `makeShelleyKeyWitness` function to these two arguments.

signing the transaction is then straightforward:

`signedTx = makeSignedTransaction allKeyWits txbody`

where allKeyWits is, in our case, a list of just one key witness.

under the hood, signing a transaction looks like this:

``` haskell
\wsk ->
  let sk        = toShelleySigningKey wsk
      vk        = getShelleyKeyWitnessVerificationKey sk
      signature = makeShelleySignature txhash sk
   in ShelleyKeyWitness era $
        Shelley.WitVKey vk signature
```

where `wsk` is a `WitnessPaymentKey (SigningKey PaymentKey)` of type `ShelleyWitnessSigningKey` and `txhash` is the hash of the transaction body. The hash function used here, `hashAnnotated`, is coming from the [cardano-ledger](https://github.com/input-output-hk/cardano-ledger/blob/master/libs/cardano-ledger-core/src/Cardano/Ledger/SafeHash.hs)

note: creating a witness can also be done separatly by providing a tx-body-file and a signing-key-file to the command `cardano-cli transaction witness`

### submit simple transaction

the parser defaults to the cardano consensus mode
so, as we are in the alonzo era, our value of type `EraInMode` is AlonzoEraInCardanoMode

this eraInMode, or more precisely a `TxInMode tx eraInMode` is needed by the `submitTxToNodeLocal`. So the Tx is converted to a TxInMode before submitting it

```haskell
-- | A 'Tx' in one of the eras supported by a given protocol mode.
--
-- For multi-era modes such as the 'CardanoMode' this type is a sum of the
-- different transaction types for all the eras. It is used in the
-- LocalTxSubmission protocol.
--
data TxInMode mode where

     -- | Everything we consider a normal transaction.
     --
     TxInMode :: Tx era -> EraInMode era mode -> TxInMode mode
```

`localTxSubmissionClientSingle` finally does submit the message as intended. 

i have a InAnyCardanoEra Tx :: InAnyCardanoEra era Tx

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
