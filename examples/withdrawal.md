-- very much work in progress still..

# withdraw from staking address

instead of using a public/private key to create a stake address, we can instead use a plutus script, and then the hash will give a stake address, a script stake address

in the cardano-api, we can build this address with:

```haskell
makeStakeAddress :: NetworkId -> StakeCredential -> StakeAddress
makeStakeAddress nw sc =
    StakeAddress
      (toShelleyNetwork nw)
      (toShelleyStakeCredential sc)
```

where the StakeCredential is, in this case, a `StakeCredentialByScript`

```haskell
data StakeCredential
       = StakeCredentialByKey    (Hash StakeKey)
       | StakeCredentialByScript  ScriptHash
  deriving (Eq, Ord, Show)
```

so the first step, as always, is to get the script.

In the example, we use 

writeStakeValidator :: FilePath -> Plutus.Address -> IO (Either (FileError ()) ())
writeStakeValidator file = writeFileTextEnvelope @(PlutusScript PlutusScriptV1) file Nothing . PlutusScriptSerialised . SBS.toShort . LBS.toStrict . serialise . Plutus.getStakeValidator . stakeValidator


### the body content

very much later on, to build the txBodyContent, we will need to fill the following field:

`txWithdrawals :: TxWithdrawals  build era`

```haskell
data TxWithdrawals build era where

     TxWithdrawalsNone :: TxWithdrawals build era

     TxWithdrawals     :: WithdrawalsSupportedInEra era
                       -> [(StakeAddress, Lovelace,
                            BuildTxWith build (Witness WitCtxStake era))]
                       -> TxWithdrawals build era
```

### the witness

our witness is a `ScriptWitness ScriptWitnessForStakeAddr scriptWitness`, where

```haskell
data ScriptWitness witctx era where

     SimpleScriptWitness :: ScriptLanguageInEra lang era
                         -> SimpleScriptVersion lang
                         -> SimpleScriptOrReferenceInput lang
                         -> ScriptWitness witctx era

     PlutusScriptWitness :: ScriptLanguageInEra  lang era
                         -> PlutusScriptVersion  lang
                         -> PlutusScriptOrReferenceInput lang
                         -> ScriptDatum witctx
                         -> ScriptRedeemer
                         -> ExecutionUnits
                         -> ScriptWitness witctx era
```

where

```haskell
data PlutusScriptOrReferenceInput lang
  = PScript (PlutusScript lang)
  | PReferenceScript TxIn (Maybe ScriptHash)
  deriving (Eq, Show)
```
