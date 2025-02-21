---
sidebar_position: 25
---

# Producing a Plutus contract blueprint

Plutus contract blueprints ([CIP-0057](https://cips.cardano.org/cip/CIP-0057)) are used to document the binary interface of a Plutus contract in a machine-readable format (JSON schema).

A contract blueprint can be produced by using the `writeBlueprint` function exported by the `PlutusTx.Blueprint` module:

``` haskell
writeBlueprint
  :: FilePath
  -- ^ The file path where the blueprint will be written to,
  --  e.g. '/tmp/plutus.json'
  -> ContractBlueprint
  -- ^ Contains all the necessary information to generate 
  -- a blueprint for a Plutus contract.
  -> IO ()
```

## Demonstrating the usage of the `writeBlueprint` function

In order to demonstrate the usage of the `writeBlueprint` function, let's consider the following example validator function and its interface:

<LiteralInclude file="Example/Cip57/Blueprint/Main.hs" language="haskell" title="interface types" start="-- BEGIN interface types" end="-- END interface types" />

<LiteralInclude file="Example/Cip57/Blueprint/Main.hs" language="haskell" title="validator" start="-- BEGIN validator" end="-- END validator" />

## Required pragmas and imports

First of all, we need to specify required pragmas and import necessary modules:

<LiteralInclude file="Example/Cip57/Blueprint/Main.hs" language="haskell" title="pragmas" start="-- BEGIN pragmas" end="-- END pragmas" />

<LiteralInclude file="Example/Cip57/Blueprint/Main.hs" language="haskell" title="imports" start="-- BEGIN imports" end="-- END imports" />

## Defining a contract blueprint value

Next, we define a contract blueprint value of the following type:

``` haskell
data ContractBlueprint where
  MkContractBlueprint
    :: forall referencedTypes
    . { contractId :: Maybe Text
        -- ^ An optional identifier for the contract.
      , contractPreamble :: Preamble
        -- ^ An object with meta-information about the contract.
      , contractValidators :: Set (ValidatorBlueprint referencedTypes)
        -- ^ A set of validator blueprints that are part of the contract.
      , contractDefinitions :: Definitions referencedTypes
        -- ^ A registry of schema definitions used across the blueprint.
      }
    -> ContractBlueprint
```

> :pushpin: **NOTE** 
> 
> The `referencedTypes` type parameter is used to track the types used in the contract making sure their schemas are included in the blueprint and that they are referenced in a type-safe way.
> 
> The blueprint will contain JSON schema definitions for all the types used in the contract, including the types **nested** within the top-level types (`MyParams`, `MyDatum`, `MyRedeemer`) recursively.

We can construct a value of this type in the following way:

<LiteralInclude file="Example/Cip57/Blueprint/Main.hs" language="haskell" title="contract blueprint declaration" start="-- BEGIN contract blueprint declaration" end="-- END contract blueprint declaration" />

The `contractId` field is optional and can be used to give a unique identifier to the contract.

The `contractPreamble` field is a value of type `PlutusTx.Blueprint.Preamble` and contains a meta-information
about the contract:

``` haskell
data Preamble = MkPreamble
  { preambleTitle         :: Text
  -- ^ A short and descriptive title of the contract application
  , preambleDescription   :: Maybe Text
  -- ^ A more elaborate description
  , preambleVersion       :: Text
  -- ^ A version number for the project.
  , preamblePlutusVersion :: PlutusVersion
  -- ^ The Plutus version assumed for all validators
  , preambleLicense       :: Maybe Text
  -- ^ A license under which the specification
  -- and contract code is distributed
  }
```

## Example construction

Here is an example construction:

<LiteralInclude file="Example/Cip57/Blueprint/Main.hs" language="haskell" title="preamble declaration" start="-- BEGIN preamble declaration" end="-- END preamble declaration" />

The `contractDefinitions` field is a registry of schema definitions used across the blueprint. 
It can be constructed using the `deriveDefinitions` function which automatically constructs schema definitions for all the types it is applied to including the types nested within them.

Since every type in the `referencedTypes` list is going to have its derived JSON-schema in the `contractDefinitions` registry under a certain unique `DefinitionId` key, we need to make sure that it has the following instances:

- `instance HasBlueprintDefinition MyType` allows to add a `MyType` schema definition to the `contractDefinitions` registry by providing a `DefinitionId` value for `MyType`, e.g. `"my-type"`.

  The good news is that most of the time these instances either already exist (for types defined in the Plutus libraries) or can be derived automatically:

  <LiteralInclude file="Example/Cip57/Blueprint/Main.hs" language="haskell" title="Generically-derived HasBlueprintDefinition instances" start="-- BEGIN derived instances" end="-- END derived instances" />

- `instance HasBlueprintSchema MyType` allows to derive a JSON schema for `MyType`.

  Types that are covered by a blueprint are exposed via the validator arguments and therefore must be convertible to/from `Data` representation. The conversion is done using instances of `ToData`, `FromData` and optionally `UnsafeFromData`. 
  
  You probably already have these instances derived with Template Haskell functions `makeIsDataIndexed` or `unstableMakeIsData` which are located in the `PlutusTx.IsData.TH` module. You can add derivation of the 
  `HasBlueprintSchema` instance very easily: 
    - Replace usages of `PlutusTx.IsData.TH.makeIsDataIndexed` with `PlutusTx.Blueprint.TH.makeIsDataSchemaIndexed`.
    - Replace usages of `PlutusTx.IsData.TH.unstableMakeIsData` with `PlutusTx.Blueprint.TH.unstableMakeIsDataSchema`.
   
  (This way `HasBlueprintSchema` instance is guaranteed to correspond to the `ToData` and `FromData` instances which are derived with it.)

  Here is an example:
  <LiteralInclude file="Example/Cip57/Blueprint/Main.hs" language="haskell" title="TH-derived HasBlueprintSchema instances" start="-- BEGIN makeIsDataSchemaIndexed" end="-- END makeIsDataSchemaIndexed" />

## Defining a validator blueprint

Finally, we need to define a validator blueprint for each validator used in the contract.

Our contract can contain one or more validators. For each one we need to provide a description as a value of the following type:

> ``` haskell
> data ValidatorBlueprint (referencedTypes :: [Type]) = MkValidatorBlueprint
>   { validatorTitle       :: Text
>   -- ^ A short and descriptive name for the validator.
>   , validatorDescription :: Maybe Text
>   -- ^ An informative description of the validator.
>   , validatorRedeemer    :: ArgumentBlueprint referencedTypes
>   -- ^ A description of the redeemer format expected by this validator.
>   , validatorDatum       :: Maybe (ArgumentBlueprint referencedTypes)
>   -- ^ A description of the datum format expected by this validator.
>   , validatorParameters  :: [ParameterBlueprint referencedTypes]
>   -- ^ A list of parameters required by the script.
>   , validatorCompiled    :: Maybe CompiledValidator
>   -- ^ A full compiled and CBOR-encoded serialized flat script together with its hash.
>   }
>   deriving stock (Show, Eq, Ord)
> ```

In our example, this would be:

<LiteralInclude file="Example/Cip57/Blueprint/Main.hs" language="haskell" title="validator blueprint declaration" start="-- BEGIN validator blueprint declaration" end="-- END validator blueprint declaration" />

The `definitionRef` function is used to reference a schema definition of a given type. 
It is smart enough to discover the schema definition from the `referencedType` list and fails to compile if the referenced type is not included.

If you want to provide validator code with its hash, you can use the `compiledValidator` function:

``` haskell
compiledValidator
  :: PlutusVersion
  -- ^ Plutus version (e.g. `PlutusV3`) to calculate the hash of the validator code.
  -> ByteString
  -- ^ The compiled validator code.
  -> CompiledValidator
```

## Writing the blueprint to a file

With all the pieces in place, we can now write the blueprint to a file:

<LiteralInclude file="Example/Cip57/Blueprint/Main.hs" language="haskell" title="write blueprint to file" start="-- BEGIN write blueprint to file" end="-- END write blueprint to file" />

## Annotations

Any [CIP-0057](https://cips.cardano.org/cip/CIP-0057) blueprint type definition may include [optional keywords](https://cips.cardano.org/cip/CIP-0057#for-any-data-type) to provide additional information:

- title
- description
- $comment

It's possible to add these keywords to a Blueprint type definition by annotating the Haskell type from which it's derived with a corresponding annotation:

- `SchemaTitle`
- `SchemaDescription`
- `SchemaComment`

For example, to add a title and description to the `MyParams` type, we can use the `SchemaTitle` and `SchemaDescription` annotations:

<LiteralInclude file="Example/Cip57/Blueprint/Main.hs" language="haskell" title="MyParams annotations" start="-- BEGIN MyParams annotations" end="-- END MyParams annotations" />

These annotations result in the following JSON schema definition:

``` json
{
  "title": "Title for the MyParams definition",
  "description": "Description for the MyParams definition",
  "dataType": "constructor",
  "fields": [
    { "$ref": "#/definitions/Bool" },
    { "$ref": "#/definitions/Integer" }
  ],
  "index": 0
}
```

For sum-types, it's possible to annotate constructors:

<LiteralInclude file="Example/Cip57/Blueprint/Main.hs" language="haskell" title="MyRedeemer annotations" start="-- BEGIN MyRedeemer annotations" end="-- END MyRedeemer annotations" />

These annotations result in the following JSON schema definition:

``` json
{
  "oneOf": [
    {
      "$comment": "Left redeemer",
      "dataType": "constructor",
      "fields": [],
      "index": 0
    },
    {
      "$comment": "Right redeemer",
      "dataType": "constructor",
      "fields": [],
      "index": 1
    }
  ]
}
```

It is also possible to annotate a validator's parameter or argument **type** (as opposed to annotating *constructors*):

``` haskell
{-# ANN type MyParams (SchemaTitle "Example parameter title") #-}
{-# ANN type MyRedeemer (SchemaTitle "Example redeemer title") #-}
```

Then, instead of providing them literally:

``` haskell
myValidator =
  MkValidatorBlueprint
    { ... elided
    , validatorParameters =
        [ MkParameterBlueprint
            { parameterTitle = Just "My Validator Parameters"
            , parameterDescription = Just "Compile-time validator parameters"
            , parameterPurpose = Set.singleton Spend
            , parameterSchema = definitionRef @MyParams
            }
        ]
    , validatorRedeemer =
        MkArgumentBlueprint
          { argumentTitle = Just "My Redeemer"
          , argumentDescription = Just "A redeemer that does something awesome"
          , argumentPurpose = Set.fromList [Spend, Mint]
          , argumentSchema = definitionRef @MyRedeemer
          }
    , ... elided
    }
```

Use TH to have a more concise version:

``` haskell
myValidator =
  MkValidatorBlueprint
    { ... elided
    , validatorParameters =
        [ $(deriveParameterBlueprint ''MyParams (Set.singleton Purpose.Spend)) ]
    , validatorRedeemer =
        $(deriveArgumentBlueprint ''MyRedeemer (Set.fromList [Purpose.Spend, Purpose.Mint]))
    , ... elided
    }
```

## Resulting full blueprint example

Here is the full [CIP-0057](https://cips.cardano.org/cip/CIP-0057) blueprint produced by this example: [plutus.json](/plutus.json)

> :pushpin: **NOTE** 
> 
> You can find a more elaborate example of a contract blueprint in the `Blueprint.Tests` module of the Plutus repository.

