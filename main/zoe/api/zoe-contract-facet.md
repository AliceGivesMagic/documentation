# Zoe Contract Facet

<Zoe-Version/>

A Zoe Contract Facet is an API object for a running contract instance to access the Zoe state
for that instance. A Zoe Contract Facet is accessed synchronously from within the contract,
and usually is referred to in code as `zcf`.

The contract instance is launched by `E(zoe).startInstance()`, and is given access to
the `zcf` object during that launch. In the operations below, `instance` is
the handle for the running contract instance.

## start

This section covers the code you need to have at the start of your contract code.

To warn if the correct return values for your contract are not being returned,
add this right before the start of your contract code. It also lets TypeScript-aware
tools (IDEs like vsCode and WebStorm) inform the developer about required parameters
and return values and warn when methods are misused.

```js
/**
 * @type {ContractStartFn}
 */
```
Your contract code must export a function `start` as a non-default export.
`zcf` is the Zoe Contract Facet and is the only argument provided to the contract

```js
const start = zcf => {
  ...
  // your code here
  return harden({ creatorFacet, creatorInvitation, publicFacet });
}
harden(start);
export { start };
```
The contract must return a record with any (or none) of the following:

- `creatorFacet` - an object, usually with admin authority. It is only given to the entity that
calls `E(zoe).startInstance()`; i.e. the party that was the creator of the current contract `instance`.
It creates `invitations` for other parties, and takes actions that are unrelated to making offers.
- `creatorInvitation` - a Zoe `invitation` only given to the entity that calls `E(zoe).startInstance()`; i.e.
the party that was the creator of the current contract `instance`.
This is usually used when a party has to make an offer first, such as escrowing the underlying good
for sale in an auction or covered call.
- `publicFacet` - an object available through Zoe to anyone who knows the contract instance. Use the `publicFacet` for general queries and actions, such as getting the current price or creating public `invitations`.

## zcf.makeZCFMint(keyword, amountMathKind)
- `keyword` `{String}`
- `amountMathKind` `{AmountMathKind}` (defaults to `MathKind.NAT`)
- Returns: `{Promise<ZCFMint>}`

Creates a synchronous Zoe mint, allowing users to mint and reallocate digital assets synchronously
instead of relying on an asynchronous ERTP `mint`.

**Important**: `ZCFMints` do **not** have the same methods as an ERTP `mint`. Do not try to use
ERTP methods on a `ZCFMint` or vice versa.

**Important**: On the other hand, the `issuer`, `brand`, and `amountMath` associated with a `zcfMint`
do have the same methods as their ERTP-derived counterparts. Assets created by a `zcfMint` are treated
the same as ERTP `mint`-created assets by ERTP methods.

`ZCFMints` have three methods:
- `getIssuerRecord()`
  - Returns: `{IssuerRecord}`
  - Returns an `issuerRecord` containing the `issuer`, `brand`, or `amountMath` associated with the `zcfMint`.
- `mintGains`
  - `gains`: `AmountKeywordRecord`
  - `zcfSeat`: `{ZCFSeat}` - optional
  - Returns: `{ZCFSeat}`
  - All `amounts` in `gains` must be of this `ZCFMint`'s `brand`.
    The `gains`' keywords are in that `seat`'s namespace.
    Mint the `gains` `amount` of assets and add them to
    that `seat`'s `allocation`. If a `seat` is provided,
    it is returned. Otherwise a new `seat` is returned.
  - `zcfMint.mintGains({ Token: amount }, seat);`
- `burnLosses`
  - `losses`: `AmountKeywordRecord`
  - `zcfSeat` : `{ZCFSeat}`
  - Returns: void
  - All `amounts` in `losses` must be of this `ZCFMint`'s `brand`.
    The `losses`' keywords are in that `seat`'s namespace.
    Subtract `losses` from that `seat`'s `allocation`, then
    burn that `amount` of assets from the pooled `purse`.
  - `zcfMint.burnLosses({ Token: amount }, seat);`

`AmountKeywordRecord` is a record in which the keys are keywords, and
the values are `amounts`. Keywords are unique identifiers per contract,
that tie together the `proposal`, `payments` to be escrowed, and `payouts`
to the user. In the below example, `Asset` and `Price` are keywords.

Users should submit their `payments` using keywords:
```js
const payments = { Asset: quatloosPayment };
```

And, users will receive their `payouts` with keywords as the keys of a `payout`:
```js
quatloosPurse.deposit(payout.Asset);
```

For example:
```js
const quatloos5 = quatloosAmountMath.make(5n);
const quatloos9 = quatloosAmountMath.make(9n);
const myAmountKeywordRecord =
{
  Asset: quatloos5,
  Price: quatloos9
}
```
The following demonstrates `zcf.makeZCFMint`:

**Note**: The call to make the `ZCFMint` is asynchronous, but
calls to the resulting `ZCFMint` are synchronous.
```js
const mySynchronousMint = await zcf.makeZCFMint('MySyncMint', 'set');
const { amountMath, brand, issuer } = mySynchronousMint.getIssuerRecord();
mySynchronousMint.mintGains({ MyKeyword: amount }, seat);
```

## zcf.getInvitationIssuer()
- Returns: [`{Issuer}`](/ertp/api/issuer.md)

Zoe has a single `invitationIssuer` for the entirety of its
lifetime. This method returns the Zoe `InvitationIssuer`, which
validates user-received `invitations` to participate
in contract instances. 

"All `invitations` come from this single `invitation` `issuer` and its `mint`, which 
mint `invitations` and validate their `amounts`."

```js
const invitationIssuer = await zcf.getInvitationIssuer();
```

## zcf.saveIssuer(issuer, keyword)
- `issuer` `{Issuer}`
- `keyword` `{String}`
- Returns: `{Promise<IssuerRecord>}`

Informs Zoe about an `issuer` and returns a promise for acknowledging
when the `issuer` is added and ready. The `keyword` is the one associated
with the new `issuer`. It returns a promise for `issuerRecord` of the new `issuer`

This saves an `issuer` in Zoe's records for this contract `instance`.
It also has saved the `issuer` information such that Zoe can handle offers involving
this `issuer` and ZCF can provide the `issuerRecord` synchronously on request.

An `IssuerRecord` has three fields, each of which holds the namesake object
associated with the `issuer` value of the record:
`issuerRecord.amountMath`, `issuerRecord.brand`, and `issuerRecord.issuer`)

```js
await zcf.saveIssuer(secondaryIssuer, keyword);
```
## zcf.makeInvitation(offerHandler, description, customProperties)
- `offerHandler` `{ZCFSeat => Object}`
- `description` `{String}`
- `customProperties` `{Object}`
- Returns: [`{Promise<Invitation>}`](/ertp/api/payment.md#payment)

Make a credible Zoe `invitation` for a smart contract. Note that `invitations` are a special case
of an ERTP `payment`. They are associated with the `invitationIssuer` and its `mint`, which 
validate and mint `invitations`. `zcf.makeInvitation()` serves as an interface to
the `invitation` `mint`.

The `invitation`'s
`value` specifies:
- The specific contract `instance`.
- The Zoe `installation`.
- A unique `handle`

The second argument is a required `description` for the `invitation`,
and should include whatever information is needed for a potential recipient of the `invitation`
to know what they are getting in the optional `customProperties` argument, which is
put in the `invitation`'s `value`.

```js
const creatorInvitation = zcf.makeInvitation(makeCallOption, 'makeCallOption')
```

## zcf.makeEmptySeatKit()
- Returns: `{ZCFSeat, Promise<UserSeat>}`

Returns an empty `ZCFSeat` and a promise for a `UserSeat`

Zoe uses `seats` to represent offers, and has two seat facets (a
particular view or API of an object;
there may be multiple such APIs per object) a `ZCFSeat` and a `UserSeat`.
```js
const { zcfSeat: mySeat } = zcf.makeEmptySeatKit();
```
## ZCFSeat object

Zoe uses `seats` to access or manipulate offers. Seats represent active
offers and let contracts and users interact with them. Zoe has two kinds
of seats. `ZCFSeats` are used within contracts and with `zcf` methods.
`UserSeats` represent offers external to Zoe and the contract.

A `ZCFSeat` includes synchronous queries for the current state of the
associated offer, such as the amounts of assets that are currently
allocated to the offer. It also includes synchronous operations
to manipulate the offer. The queries and operations are as follows:

- `hasExited()`
  - Returns: `{ Boolean }`
  - Returns `true` if the `seat` has exited, `false` if it is still active.
- `getNotifier()`
  - Returns: `{ Notifier<Allocation> }`
  - Returns a `notifier` associated with the `seat`'s `allocation`. You use a `notifier`
    wherever some piece of code has changing state that other code wants updates on.
    This `notifier` provides updates on changing `allocations` for this `seat`, and tells
    when the `seat` has been exited. For more on `notifiers`, see the [Distributed Programming Guide](/distributed-programming.md#notifiers).
- `getProposal()`
  - Returns: `{ ProposalRecord }`
  - A `Proposal` is represented by a `ProposalRecord`. It is the rules
    accompanying the escrow of `payments` dictating what the user expects
    to get back from Zoe. It has keys `give`, `want`, and
    `exit`. `give` and `want` are records with keywords as keys and
    `amounts` as values. The `proposal` is a user's understanding of the
    contract that they are entering when they make an offer. See
    [`E(zoe).offer()`](./zoe.md#e-zoe-offer-invitation-proposal-paymentkeywordrecord) for full details.
  - Example:
    ```js
    const { want, give, exit } = sellerSeat.getProposal();
    ```
- `getAmountAllocated(keyword, brand)`
  - Returns: `{ Amount }`
  - Returns the `amount` from the part of the `allocation` that matches the
    `keyword` and `brand`. If the `keyword` is not in the `allocation`, it
    returns an empty `amount` of the `brand` argument.

    This is similar to the next method, `getCurrentAllocation()`. `getAmountAllocated()`
    gets the `allocation` of one keyword at a time, while `getCurrentAllocation()` returns
    all the current `allocations` at once.
- `getCurrentAllocation()`
  - Returns: `{ <Allocation> }`
  - An `Allocation` is an `AmountKeywordRecord` of key-value pairs where
    the key is a keyword such as `Asset` or `Price` applicable to the
    contract. The value is an `amount` with its `value` and `brand`.

    `Allocations` represent the `amounts` to be paid out to each `seat` on exit. (After
    `exit()` has been called, the final allocation balances, which were transferred to
	payouts continue to be reported.)
    Normal reasons for exiting are the user requesting to exit or the contract
    explicitly choosing to close out the `seat`. The guarantees also hold if the contract
    encounters an error or misbehaves. There are several methods for finding out
    what `amount` a current `allocation` is.

    This is similar to the previous method, `getAmountAllocated()`. `getAmountAllocated()`
    gets the `allocation` of one keyword at a time, while `getCurrentAllocation()` returns
    all the current `allocations` at once.

    An `Allocation` example:
    - ```js
      {
        Asset: quatloosAmountMath.make(5n),
        Price: moolaAmountMath.make(9n)
      }
      ```
 - `exit(completion)`
   - Returns: `void`
   - Causes the `seat` to exit, concluding its existence. All `payouts`, if any,
     are made, and the `seat` object can no longer interact with the contract.
     The `completion` argument is usually a string, but this is not required. Its
     only use is for the notification sent to the contract instance's `done()` function. 
     Any other still open seats or outstanding promises and the contract instance continue.
 - `fail(msg)`
   - Returns: `void`
   - The `seat` exits, displaying the optional `msg` string, if there is one, on the console.
     This is equivalent to exiting, except that `exit` is successful while
     `fail()` signals an error occured while processing the offer. The contract
     still gets its current `allocation` and the `seat` can no longer interact with the contract.
     Any other still open seats or outstanding promises and the contract instance continue.
 - `kickOut(msg)` **Renamed fail(msg) as of 4-OCT-2020. DO NOT USE**
   - Returns: `void`
   - The `seat` exits, displaying the optional `msg` string, if there is one, on the console.
     This is equivalent to exiting, except that `exit` is for a successful transaction while
     `kickOut()` aborts the transaction attempt and signals an error. The contract
     still gets its current `allocation` and the `seat` can no longer interact with the contract.
 - `stage(newAllocation)`
   - Returns: `{ SeatStaging }`
   - A `seatStaging` is an association of a `seat` with reallocations. `reallocate()` takes
     at least two `seatStagings` as arguments and does its reallocation based on them.

     You can create multiple independent `seatStagings` for a `seat`. None of them has
     any effect until submitted to `reallocate()`. Each call to `stage()` starts from the
     `seat`'s current `allocation` and uses the `newAllocation` as a replacement for the
     current state. Any keywords not mentioned in `newAllocation` retain the
     same `amounts`. All keywords mentioned in the `newAllocation` have their `amounts`
     replaced with the corresponding `amount` from `newAllocation`.

     Note that ZoeHelpers [`trade()`](./zoe-helpers.md#trade-zcf-left-right-lefthasexitedmsg-righthasexitedmsg) and [`swap()`](./zoe-helpers.md#swap-zcf-leftseat-rightseat-lefthasexitedmsg-righthasexitedmsg) might
     be easier to use for simple cases.
 - `isOfferSafe(newAllocation)`
   - Returns `{ Boolean }`
   - Takes an `allocation` as an argument and returns `true` if that `allocation`
     satisfies offer safety, `false` if is doesn't. Essentially, it checks
     `newAllocation` for offer safety, against the `seat`'s `proposal`.
     It checks whether `newAllocation` fully satisfies
     `proposal.give` (giving a refund) or whether it fully satisfies
     `proposal.want`. Both can be fully satisfied. See the ZoeHelper
     [`satisfies()`](./zoe-helpers.md#satisfies-zcf-seat-update) method for more details.

## zcf.getBrandForIssuer(issuer)
- `issuer` `{Issuer}`
- Returns: `{Brand}`

Returns the `brand` associated with the `issuer`.

## zcf.getIssuerForBrand(brand)
- `brand` `{Brand}`
- Returns: `{Issuer}`

Returns the `issuer` of the `brand` argument.

## zcf.getAmountMath(brand)
- `brand` `{String}`
- Returns: `{amountMath}`

Returns the `amountMath` object associated with the `brand` argument.
```js
const assetAmountMath = zcf.getAmountMath(assetAmount.brand);
```
## zcf.stopAcceptingOffers()
- The contract requests Zoe to not accept offers for this contract instance. 
It can't be called from outside the contract unless the contract explicitly makes it accessible.

## zcf.shutdown(completion)
     
Shuts down the entire vat and contract instance and gives payouts.

All open `seats` associated with the current `instance` have `fail()`
called on them.

Call when:
- You want nothing more to happen in the contract, and
- You don't want to take any more offers

The `completion` argument is usually a string, but this 
is not required. It is used for the notification sent to the
contract instance's `done()` function. Any still open seats or
other outstanding promises are closed with a generic 'vat terminated' 
message.
```js
zcf.shutdown();
```
## zcf.getTerms()
- Returns: `{Object}`

Returns the `issuers`, `brands`, `maths`, and custom `terms` the current contract `instance` was instantiated with.

The returned values look like:
```js
{ brands, issuers, maths, customTermA, customTermB ... }
// where brands, issuers, and maths are all keywordRecords, like:

{
    brands: { A: moolaKit.brand, B: simoleanKit.brand },
    issuers: { A: moolaKit.issuer, B: simoleanKit.issuer },
    maths: { A: moolaKit.amountMath, B: simoleanKit.amountMath },
    customTermA: 'something',
    customTermB: 'something else',
 };
 ```

Note that there is also an `E(zoe).getTerms(instance)`. Often the choice of which to use is not which method
to use, but which of Zoe Service or ZCF you have access to. On the contract side, you more easily have access
to `zcf`, and `zcf` already knows what instance is running. So in contract code, you use `zcf.getTerms()`. From
a user side, with access to Zoe Service, you use `E(zoe).getTerms()`.
```js
const { brands, issuers, maths, terms } = zcf.getTerms()
```

## zcf.getZoeService()
- Returns: [ZoeService](./zoe.md)

This is the only way to get the user-facing [Zoe Service API](/zoe/api/zoe.md#zoe) to
the contract code as well.
```js
// Making an offer to another contract instance in the contract.
const zoeService = zcf.getZoeService();
E(zoeService).offer(creatorInvitation, proposal, paymentKeywordRecord);
```

## zcf.assertUniqueKeyword(keyword)
- `keyword` `{String}`
- Returns: Undefined

Checks if a keyword is valid and not already used as a `brand` in this `instance` (i.e. unique)
and could be used as a new `brand` to make an `issuer`. Throws an appropriate error if it's not
a valid keyword, or is not unique.
```js
zcf.assertUniqueKeyword(keyword);
```
## zcf.reallocate(seatStagings)
- `seatStagings` `{SeatStaging[]}` (at least two)
- Returns: `{void}`

The contract reallocates over `seatStagings`, which are
associations of `seats` with new `allocations` to be used in reallocation.
There must be at least two `seatStagings` in the array argument.

The reallocation only succeeds if it:
1. Conserves rights (the specified `amounts` have the same total value as the
  current total amount)
2. Is 'offer-safe' for all parties
  involved. Offer safety is checked at the staging step.

The reallocation is partial, only applying to `seats` associated
with the `seatStagings`. By induction, if rights conservation and
offer safety hold before, they hold after a safe reallocation.

This is true even though we only re-validate for `seats` whose
`allocations` change. A reallocation can only effect offer safety for
those `seats`, and since rights are conserved for the change, overall
rights are unchanged.

`reallocate()` throws these errors:
- `reallocating must be done over two or more seats`
- `The seatStaging was not recognized`
- `The seatStaging was not recognized`
- `keyword must be unique`
- If the total `amount` per `brand` is not equal to the total `amount` per `brand`
  in the proposed reallocation. (no message)
```js
zcf.reallocate(
    seat.stage(seatAAllocation),
    seat.stage(seatBAllocation),
  );
```
