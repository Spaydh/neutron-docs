# Messages
> NOTE: Messages are similar to vanilla cosmos gov module
## Proposal Submission

Proposals can be submitted by any account via a `MsgSubmitProposal`
transaction.

All `sdk.Msgs` passed into the `messages` field of a `MsgSubmitProposal` message
must be registered in the app's `MsgServiceRouter`. Each of these messages must
have one signer, namely the gov module account. And finally, the metadata length
must not be larger than the `maxMetadataLen` config passed into the gov keeper.

**State modifications:**

* Generate new `proposalID`
* Create new `Proposal`
* Initialise `Proposal`'s attributes
* Decrease balance of sender by `InitialDeposit`
* If `MinDeposit` is reached:
    * Push `proposalID` in `ProposalProcessingQueue`
* Transfer `InitialDeposit` from the `Proposer` to the governance `ModuleAccount`

A `MsgSubmitProposal` transaction can be handled according to the following
pseudocode.

```go
// PSEUDOCODE //
// Check if MsgSubmitProposal is valid. If it is, create proposal //

upon receiving txGovSubmitProposal from sender do

  if !correctlyFormatted(txGovSubmitProposal)
    // check if proposal is correctly formatted and the messages have routes to other modules. Includes fee payment.
    // check if all messages' unique Signer is the gov acct.
    // check if the metadata is not too long.
    throw

  initialDeposit = txGovSubmitProposal.InitialDeposit
  if (initialDeposit.Atoms <= 0) OR (sender.AtomBalance < initialDeposit.Atoms)
    // InitialDeposit is negative or null OR sender has insufficient funds
    throw

  if (txGovSubmitProposal.Type != ProposalTypePlainText) OR (txGovSubmitProposal.Type != ProposalTypeSoftwareUpgrade)

  sender.AtomBalance -= initialDeposit.Atoms

  depositParam = load(GlobalParams, 'DepositParam')

  proposalID = generate new proposalID
  proposal = NewProposal()

  proposal.Messages = txGovSubmitProposal.Messages
  proposal.Metadata = txGovSubmitProposal.Metadata
  proposal.TotalDeposit = initialDeposit
  proposal.SubmitTime = <CurrentTime>
  proposal.DepositEndTime = <CurrentTime>.Add(depositParam.MaxDepositPeriod)
  proposal.Deposits.append({initialDeposit, sender})
  proposal.Submitter = sender
  proposal.YesVotes = 0
  proposal.NoVotes = 0
  proposal.NoWithVetoVotes = 0
  proposal.AbstainVotes = 0
  proposal.CurrentStatus = ProposalStatusOpen

  store(Proposals, <proposalID|'proposal'>, proposal) // Store proposal in Proposals mapping
  return proposalID
```

## Deposit

Once a proposal is submitted, if
`Proposal.TotalDeposit < ActiveParam.MinDeposit`, Atom holders can send
`MsgDeposit` transactions to increase the proposal's deposit.

```protobuf
// MsgDeposit defines a message to submit a deposit to an existing proposal.
message MsgDeposit {
option (gogoproto.equal)            = false;
option (gogoproto.goproto_stringer) = false;
option (gogoproto.stringer)         = false;
option (gogoproto.goproto_getters)  = false;

uint64   proposal_id = 1 [(gogoproto.jsontag) = "proposal_id", (gogoproto.moretags) = "yaml:\"proposal_id\""];
string   depositor   = 2;
repeated cosmos.base.v1beta1.Coin amount = 3
[(gogoproto.nullable) = false, (gogoproto.castrepeated) = "github.com/cosmos/cosmos-sdk/types.Coins"];
}
```
**State modifications:**

* Decrease balance of sender by `deposit`
* Add `deposit` of sender in `proposal.Deposits`
* Increase `proposal.TotalDeposit` by sender's `deposit`
* If `MinDeposit` is reached:
    * Push `proposalID` in `ProposalProcessingQueueEnd`
* Transfer `Deposit` from the `proposer` to the governance `ModuleAccount`

A `MsgDeposit` transaction has to go through a number of checks to be valid.
These checks are outlined in the following pseudocode.

```go
// PSEUDOCODE //
// Check if MsgDeposit is valid. If it is, increase deposit and check if MinDeposit is reached

upon receiving txGovDeposit from sender do
  // check if proposal is correctly formatted. Includes fee payment.

  if !correctlyFormatted(txGovDeposit)
    throw

  proposal = load(Proposals, <txGovDeposit.ProposalID|'proposal'>) // proposal is a const key, proposalID is variable

  if (proposal == nil)
    // There is no proposal for this proposalID
    throw

  if (txGovDeposit.Deposit.Atoms <= 0) OR (sender.AtomBalance < txGovDeposit.Deposit.Atoms) OR (proposal.CurrentStatus != ProposalStatusOpen)

    // deposit is negative or null
    // OR sender has insufficient funds
    // OR proposal is not open for deposit anymore

    throw

  depositParam = load(GlobalParams, 'DepositParam')

  if (CurrentBlock >= proposal.SubmitBlock + depositParam.MaxDepositPeriod)
    proposal.CurrentStatus = ProposalStatusClosed

  else
    // sender can deposit
    sender.AtomBalance -= txGovDeposit.Deposit.Atoms

    proposal.Deposits.append({txGovVote.Deposit, sender})
    proposal.TotalDeposit.Plus(txGovDeposit.Deposit)

    if (proposal.TotalDeposit >= depositParam.MinDeposit)
      // MinDeposit is reached, vote opens

      proposal.VotingStartBlock = CurrentBlock
      proposal.CurrentStatus = ProposalStatusActive
      ProposalProcessingQueue.push(txGovDeposit.ProposalID)

  store(Proposals, <txGovVote.ProposalID|'proposal'>, proposal)
```

## Vote

Once `ActiveParam.MinDeposit` is reached, voting period starts. From there,
bonded Atom holders are able to send `MsgVote` transactions to cast their
vote on the proposal.

```protobuf
// MsgVote defines a message to cast a vote.
message MsgVote {
  option (gogoproto.equal)            = false;
  option (gogoproto.goproto_stringer) = false;
  option (gogoproto.stringer)         = false;
  option (gogoproto.goproto_getters)  = false;

  uint64     proposal_id = 1 [(gogoproto.jsontag) = "proposal_id", (gogoproto.moretags) = "yaml:\"proposal_id\""];
  string     voter       = 2;
  VoteOption option      = 3;
}
```
**State modifications:**

* Record `Vote` of sender

*Note: Gas cost for this message has to take into account the future tallying of the vote in EndBlocker.*

Next is a pseudocode outline of the way `MsgVote` transactions are
handled:

```go
  // PSEUDOCODE //
  // Check if MsgVote is valid. If it is, count vote//

  upon receiving txGovVote from sender do
    // check if proposal is correctly formatted. Includes fee payment.

    if !correctlyFormatted(txGovDeposit)
      throw

    proposal = load(Proposals, <txGovDeposit.ProposalID|'proposal'>)

    if (proposal == nil)
      // There is no proposal for this proposalID
      throw


    if  (proposal.CurrentStatus == ProposalStatusActive)


        // Sender can vote if
        // Proposal is active
        // Sender has some bonds

        store(Governance, <txGovVote.ProposalID|'addresses'|sender>, txGovVote.Vote)   // Voters can vote multiple times. Re-voting overrides previous vote. This is ok because tallying is done once at the end.
```
