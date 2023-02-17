# building transactions by hand

to construct a cardano-api `TxBody` we need first the bodyContent.
we will construct a cardano-api `TxBodyContent BuildTx AlonzoEra`, for which we need, in particular, the two fields `txIns` and `txOuts`

## construct the fitting (TxIn, BuildTxWith build (Witness WitCtxTxIn era))

`txIns :: TxIns build era`, which is of type:

type TxIns build era = [(TxIn, BuildTxWith build (Witness WitCtxTxIn era))]

next to the TxIn itself, there is the information of whether the TxIn needs a `KeyWitness` or a `ScriptWitness`

```haskell
-- for spending with a KeyWitness
txInForSpending :: TxIn -> (TxIn, BuildTxWith BuildTx (Witness WitCtxTxIn era))
txInForSpending = (,BuildTxWith (KeyWitness KeyWitnessForSpending))

-- for spending with a ScriptWitness
txInForValidator ::
  (PlutusTx.ToData d, PlutusTx.ToData r) =>
  TxIn ->
  Validator ->
  TxDatum d ->
  TxRedeemer r ->
  ExecutionUnits ->
  Either ToCardanoError (TxIn, BuildTxWith BuildTx (Witness WitCtxTxIn AlonzoEra))
txInForValidator txIn validator (TxDatum datum) (TxRedeemer redeemer) exUnits = do
  scriptInEra <- toCardanoScriptInEra (getValidator validator)
  case scriptInEra of
    ScriptInEra lang (PlutusScript version script) ->
      pure
        ( txIn
        , BuildTxWith $
            ScriptWitness ScriptWitnessForSpending $
              PlutusScriptWitness
                lang
                version
                script
                (ScriptDatumForTxIn (toCardanoData datum))
                (toCardanoData redeemer)
                exUnits
        )
    -- only Plutus scripts are supported
    ScriptInEra _ (SimpleScript _ _) -> Left DeserialisationError
```

important: `toCardanoScriptInEra` and `toCardanoAPIData` (which is used by `toCardanoData`) are functions from the `plutus-apps` module `Ledger.Tx.CardanoApi.Internal`. This module is an interface to the transaction types from 'cardano-api'
and gets re-exported through [Ledger.Tx.CardanoAPI](https://github.com/input-output-hk/plutus-apps/blob/main/plutus-ledger/src/Ledger/Tx/CardanoAPI.hs)

## construct the fitting TxOut ctx era

```haskell
data TxOut ctx era = TxOut (AddressInEra    era)
                           (TxOutValue      era)
                           (TxOutDatum ctx  era)
                           (ReferenceScript era)
```

### TxOut to key address

Constructing a TxOut to a key address is pretty straightforward

for the address, we first we need a `SigningKey PaymentKey` which we get (as `PaymentKey` has an instance of the `Key` class) with

`generateSigningKey :: Key keyrole => AsType keyrole -> IO (SigningKey keyrole)`

the corresponding `VerificationKey PaymentKey` and then its hash is obtained with

`vkeyHash = verificationKeyHash (getVerificationKey skey)`, with both of these functions defined in the `Key` class

defining the networkId as either `Mainnet` or `Testnet NetworkMagic`, we get the address with another cardano-api function:

`keyAddr = makeShelleyAddressInEra networkId (PaymentCredentialByKey vkeyHash) NoStakeAddress`

next to the address, we need a value. For this, cardano-api provides the function `lovelaceToTxOutValue` which we apply to some `Lovelace` (a newtype wrapper around `Integer`).
The datum field of the `TxOut` is `TxOutDatumNone`, as this is a payment to a key address.

putting it all together:

`addressOut = TxOut keyAddr (lovelaceToTxOutValue lovelace) TxOutDatumNone ReferenceScriptNone`

### TxOut to script address

To construct a TxOut for a script address we also need a `TxOutDatumHash`. if the datum d is an instance to the `ToData` class, we can convert it as follows:

`datumHash = TxOutDatumHash ScriptDataInAlonzoEra (hashScriptData $ toCardanoData d)`

where `toCardanoData` again is from Ledger.Tx.CardanoAPI

For the script address, the `PaymentCredential` must be a `PaymentCredentialByScript ScriptHash`, having this we can again construct the address with

`scriptAddr = makeShelleyAddressInEra networkId (PaymentCredentialByScript scriptHash) NoStakeAddress`

And the resulting TxOut is then

`scriptOut = TxOut scriptAddr (lovelaceToTxOutValue lovelace) datumHash ReferenceScriptNone`

so in a real world example:

```haskell
-- simplified, from `buildBetTx`, having a validatorAddress and a changeAddress
-- this tx only spends from key addresses, so the txIns are built with `txInForSpending`
-- this tx pays to a script (scriptOut) and to a key address (addressOut)
buildTxBody :: [TxIn] -> Either String (TxBody AlonzoEra)
buildTxBody inputRefs = do
  let bodyContent =
        baseBodyContent
          { txIns = txInForSpending <$> inputRefs
          , txOuts = scriptOut : addressOut
          }
  first (("bad tx-body: " <>) . show) $ createAndValidateTransactionBody bodyContent
```

`createAndValidateTransactionBody` is from the cardano-api

as we see from the TxBodyContent, we are in the AlonzoEra, this is decisive for `createAndValidateTransactionBody` and particularly for `makeShelleyTransactionBody`, which depending on the era builds a different TxBody
in our case, we have the `ShelleyBasedEraAlonzo`, (must be ShelleyBased as in the Byron era there were no complex transactions). so our baseBodyContent is missing the fields 

- txInsReference =  TxInsReferenceNone        :: TxInsReference build era,
- txTotalCollateral = TxTotalCollateralNone   :: TxTotalCollateral era,
- txReturnCollateral = TxReturnCollateralNone :: TxReturnCollateral CtxTx era,

of course we could add them and build a `TxBody BabbageEra`

as `baseBodyContent` we take:

```haskell
baseBodyContent :: TxBodyContent BuildTx AlonzoEra
baseBodyContent =
  TxBodyContent
    { txIns = []
    , txInsCollateral = TxInsCollateralNone
    , txOuts = []
    , txFee = TxFeeExplicit TxFeesExplicitInAlonzoEra 0
    , txValidityRange = (TxValidityNoLowerBound, TxValidityNoUpperBound ValidityNoUpperBoundInAlonzoEra)
    , txMetadata = TxMetadataNone
    , txAuxScripts = TxAuxScriptsNone
    , txExtraKeyWits = TxExtraKeyWitnessesNone
    , txProtocolParams = BuildTxWith Nothing
    , txWithdrawals = TxWithdrawalsNone
    , txCertificates = TxCertificatesNone
    , txUpdateProposal = TxUpdateProposalNone
    , txMintValue = TxMintNone
    , txScriptValidity = TxScriptValidity TxScriptValiditySupportedInAlonzoEra ScriptValid
    }
```

txFee is set to 0 at this point because this transaction will be sent to a hydra head via websocket.  

to build a transaction that is to be sent to a cardano-node, we would use `makeTransactionBodyAutoBalance` instead of `createAndValidateTransactionBody`. `makeTransactionBodyAutoBalance` needs more information, though, all of which can be queried from a local node. how this is done is shown in the `runTxBuildCmd` function from the [Cardano.CLI.Shelley.Run.Transaction](https://github.com/input-output-hk/cardano-node/blob/master/cardano-cli/src/Cardano/CLI/Shelley/Run/Transaction.hs) module

To build a transaction that spends from a script address, see `buildClaimTx` from the [hydra-demo](https://github.com/mlabs-haskell/hydra-demo/blob/master/src/HydraRPS/App.hs). 

There are a few differences to the example above:

#### build with txInForValidator

the `txIns` field of the TxBodyContent contains elements that are built with `txInForValidator` instead of `txInForSpending`
      
note: as this transaction is not going to be sent to a cardano node but to a hydra head, the execution units are just set to the half of the maxTxExUnits: 

```haskell
ExecutionUnits
  { executionSteps = executionSteps maxTxExUnits `div` 2
  , executionMemory = executionMemory maxTxExUnits `div` 2
  }
```

In the case of a transaction for the cardano blockchain, the evaluation is done with `evaluateTransactionExecutionUnits` in `makeTransactionBodyAutoBalance`, where all the scripts are run to count the execution units

#### Collateral

we need to define a `TxIn` as collateral
txInsCollateral = TxInsCollateral CollateralInAlonzoEra [collateralTxIn]

#### ProtocolParameters

we need the `ProtocolParameters`, as scripts will be executed in the validation of this transcation

## sign and submit tx

so we have a balanced tx of type `TxBody AlonzoEra`.

signing this unsignedTx is straightforward:

`signedTx = signTx userSkey unsignedTx`

```haskell
signTx :: SigningKey PaymentKey -> TxBody AlonzoEra -> Tx AlonzoEra
signTx signingKey body = Tx body [witness]
  where
    witness = makeShelleyKeyWitness body (WitnessPaymentKey signingKey)

-- this is possible because of the pattern
pattern Tx :: TxBody era -> [KeyWitness era] -> Tx era
pattern Tx txbody ws <- (getTxBodyAndWitnesses -> (txbody, ws))
  where
    Tx txbody ws = makeSignedTransaction ws txbody
```

## submit it

done through the websocket, so different. for cardano-node see cardano-cli
