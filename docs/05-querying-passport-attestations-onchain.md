# How to Decode Passport Attestations


## Intro

The purpose of this document is to provide instructions on how to load and decode Passport attestions onchain for a given ETH address. It also outlines the implementation considerations for the `GitcoinPassportDecoder` smart contract.

For details on the EAS schema used to store Gitcoin Passports onchain, please see: [Onchain Passport Attestation](./01-onchain-passport-attestation.md)

## Explanation of the Smart Contracts Involved

In order to understand how Passports and Passports Scores exist onchain, take a read of [Bringing Passport Data Onchain](./00-onchain-data.md#bringing-passport-data-onchain).

In order to load the latest Passport or Score attestations from EAS, we need to perform the following steps:

### Step 1: Get the attestation UID

In order to find the attestation UID that is owned by a given ETH address for a given schema (like the passport schema) we will need to use the  `GitcoinResolver` as this plays the role of an 'indexer' and will store the latest attestation UID for a given schema and for a given recipient.

This means that knowing the schema UID (which will be attained internally) and an ETH address we can get the attestation UID for a users.

### Step 2: Get the attestation

Having the attestaion UID from Step 1, we can just use the `getAttestation` method of the EAS smart contract to load a user's attestation.

### Step 3: Decode the attestation data

The schema of the passport attestation is documented in [Onchain Passport Attestation](./01-onchain-passport-attestation.md). In order to decode it one will need to use the `abi.decode` function as shown in the snippet below:

```sol
// Decode the attestion output
(providers, hashes, issuanceDates, expirationDates, providerMapVersion) = abi.decode(
  attestation.data,
  (uint256[], bytes32[], uint64[], uint64[], uint16)
);
```

### Step 4: Load the stamp / VC data from the attestation

The format of the passport attestation (what each individual field contains) is described in the document [Onchain Passport Attestation](./01-onchain-passport-attestation.md).

In order to decode the stamps that are saved in a Passport attestation, one needs to understand and keep track of all the stamp providers.
To optimise for space and gas costs, the `providers` field has been used as a bit array, with each bit being assigned to a given provider.

But this bit map is not fixed, and can potentially change over time as some providers are removed and others are added to the providers of Gitcoin Passport.
This is why we need to track the versions of the provider map. This can be achieved in a simple mapping like:

```sol
mapping(uint32 => string[]) public providerVersions;
```

This is how the `providerVersions` shall be used:

- keep an array of strings (provider names) for each version
- each position in the array coresponds to 1 bit in the `providers` field in the attestation
- new providers can be added to any array in this map

This how the `providerVersions` is meant to be used:

- the current version used for pushing stamps onchain is typically the latest version present in this map
- adding new providers for the current version of the providers list can be done by simply appending new elements to the array
- removing providers from an array is not possible. When providers are removed from the Passport Application, then there are 2 ways to deal with this in the `providerVersions`:
  - keep the current version of the providers array, and the deprecated providers will simply not be written onchain any more (1 bit from the `providers` field of the attestation will be unused). This typically makes sense when there is only a small number of unused field
  - create a new `providers` list version. This makes sense if the number of unused bits in the `providers` field is higher and we want to reset this

_[← Back to README](..#other-topics)_