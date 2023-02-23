# withdraw from staking address

Instead of using signing/verifcation keys to create a stake address, we can also use a plutus script and hash it to get a stake address.

In PPP0310 we have such a script which is parameterised by an address and which we can serialise, convert to its cardano-api type and write to disc (as seen almost identically in other examples).

```haskell
writeStakeValidator :: FilePath -> Plutus.Address -> IO (Either (FileError ()) ())
writeStakeValidator file = writeFileTextEnvelope @(PlutusScript PlutusScriptV1) file Nothing . PlutusScriptSerialised . SBS.toShort . LBS.toStrict . serialise . Plutus.getStakeValidator . stakeValidator
```
where `stakeValidator` is of type `Address -> ValidatorScript`

the logic of the script is explained in the lecture, basically it just demands that in order to withdraw the rewards, half of the withdrawl has to go to a specified address (the one used as argument for the typed script)

so, once the cardano-cli has the script, it can produce a `StakeCredential` with value `StakeCredentialByScript (ScriptHash sh)`, where `sh` is the script hash obtained with `scriptHash`.
finally, the cardano-api provides the `makeStakeAddress` function to build the address.

```haskell
makeStakeAddress :: NetworkId -> StakeCredential -> StakeAddress
makeStakeAddress nw sc =
    StakeAddress
      (toShelleyNetwork nw)
      (toShelleyStakeCredential sc)
```

the corresponding cardano-cli command is `cardano-cli stake-address build`.

## registering and delegating

The stake address itself will only be used for withdrawals. To register and delegate it to a pool, we only need the underlying `StakeCredential`.

the cardano-cli offers the commands `stake-address registration-certificate` and `stake-address delegation-certificate`, both taking the script as argument. For the delegation we also need a `PoolId` which in the example is obtained with the `query stake-pools` command (there is only one pool on the testnet).
As shown below, in the cardano-api such a certificates are just wrapped `StakeCredential`s (plus the poolId info in case of the delegation).

```haskell
data Certificate =
     StakeAddressRegistrationCertificate   StakeCredential
   | StakeAddressDeregistrationCertificate StakeCredential
   | StakeAddressDelegationCertificate     StakeCredential PoolId
   ...
```

having the required certificates (`$registration` and `$delegation` below), we can build the transaction to register and delegate onchain.

registration does not require witnessing, but delegation (and also deregistration) does, so we need the `--certificate-script-file` and `--certificate-redeemer-file` as arguments to be able to build the script witness.

```bash
cardano-cli transaction build \
    --testnet-magic 42 \
    --change-address $(cat $script_payment_addr) \
    --out-file $raw \
    --tx-in $txin \
    --tx-in-collateral $txin \
    --certificate-file $registration \
    --certificate-file $delegation \
    --certificate-script-file $script \
    --certificate-redeemer-file unit.json \
    --protocol-params-file $pp
```

the cardano-cli converts the info above into the cardano-api `TxCertificates` type, needed for the `txCertificates` field of the transaction body content.

```haskell
data TxCertificates build era where

     TxCertificatesNone :: TxCertificates build era

     TxCertificates     :: CertificatesSupportedInEra era
                        -> [Certificate]
                        -> BuildTxWith build
                             (Map StakeCredential (Witness WitCtxStake era))
                        -> TxCertificates build era
```

The `[Certificate]` list will contain both the registration and delegation certificate. And, as delegation requires witnessing, the `Map StakeCredential (Witness WitCtxStake era)` above contains our stake credential mapped with the appropriate witness, which is a `ScriptWitness ScriptWitnessForStakeAddr sWit`, where the `sWit` is a `PlutusScriptWitness` containing the usual information (as the script itself and the redeemer).

The execution units for the stake script will, as in the other examples, again be calculated and subsituted into the witness by `makeTransactionBodyAutoBalance`

finally the transaction has to signed with the signing key associated with the `--tx-in` value (which is not only needed for the transaction fees, but also for the registraion deposit). 

## withdrawing

the `--change-address` given above is a key address, which has not only a payment component, but also a staking component, which is of course our staking script.

this is done by adding to the already known `cardano-cli address build` the `--stake-script-file` argument.

we funded this address in the above transaction. and now we use it to withdraw


one of our inputs for the cardano-cli `runTxBuild` function is:
 
[(StakeAddress, Lovelace, Maybe (ScriptWitness WitCtxStake era))]

having given the address and amount, the script and the redeemer like so:

    --withdrawal "$(cat tmp/user1-script-stake.addr)+$amt1" \
    --withdrawal-script-file tmp/stake-validator.script \
    --withdrawal-redeemer-file unit.json \

we can build that input, and in the process also convert the `Just scriptWitness` value into the witness type needed for our transaction body content, which is of type `Witness WitCtxStake era`: 

```haskell
TxWithdrawals :: WithdrawalsSupportedInEra era
              -> [(StakeAddress, Lovelace,
                   BuildTxWith build (Witness WitCtxStake era))]
              -> TxWithdrawals build era
```

this conversion is done with the cardano-cli function `validateTxWithdrawals`. What this function is doing is basically transforming `Nothing`s into key witnesses (tagged with `KeyWitnessForStakeAddr`) and `Just`s into script witnesses (tagged with `ScriptWitnessForStakeAddr`), both of cardano-api type `Witness`.

so, having built the value for the `txWithdrawals` field of the transaction body content, the cardano-cli again uses the `makeTransactionBodyAutoBalance` function, as seen in the other examples, to balance the transaction.

so again, this function calculates the execution units, this time of the stake validator script, and replaces the default value of 0 execution units in the script witness with the new value. Then, after calculating the fee and the change going to the change address, the transaction body is built by applying the transaction body content to `createAndValidateTransactionBody` 

what about the -tx-out? we must pay at least half of the rewards to user 2, so we create that txout in the transaction

### signing

signing then is nothing special, we just need to provide the signing key to spend the `--tx-in` and the `--collateral`, which in this case are the same input.

```bash
#!/bin/bash

txin=$1
amt1=$(scripts/query-stake-address-info-user1-script.sh | jq .[0].rewardAccountBalance)
amt2=$(expr $amt1 / 2 + 1)
pp=tmp/protocol-params.json
raw=tmp/tx.raw
signed=tmp/tx.signed

echo "txin = $1"
echo "amt1 = $amt1"
echo "amt2 = $amt2"

export CARDANO_NODE_SOCKET_PATH=cardano-private-testnet-setup/private-testnet/node-bft1/node.sock

cardano-cli query protocol-parameters \
    --testnet-magic 42 \
    --out-file $pp

cardano-cli transaction build \
    --testnet-magic 42 \
    --change-address $(cat tmp/user1-script.addr) \
    --out-file $raw \
    --tx-in $txin \
    --tx-in-collateral $txin \
    --tx-out "$(cat tmp/user2.addr)+$amt2 lovelace" \
    --withdrawal "$(cat tmp/user1-script-stake.addr)+$amt1" \
    --withdrawal-script-file tmp/stake-validator.script \
    --withdrawal-redeemer-file unit.json \
    --protocol-params-file $pp

cardano-cli transaction sign \
    --testnet-magic 42 \
    --tx-body-file $raw \
    --out-file $signed \
    --signing-key-file cardano-private-testnet-setup/private-testnet/addresses/user1.skey

cardano-cli transaction submit \
    --testnet-magic 42 \
    --tx-file $signed
```
