# Metadata

> NOTE: Metadata are similar to the original `gov` module. The gov module has two locations for metadata where users can
> provide further context about the on-chain actions they are taking. By default all metadata fields have a 255 character
> length field where metadata can be stored in json format, either on-chain or off-chain depending on the amount of data
> required. Here we provide a recommendation for the json structure and where the data should be stored. There are two
> important factors in making these recommendations. First, that the gov and group modules are consistent with one
> another, note the number of proposals made by all groups may be quite large. Second, that client applications such as
> block explorers and governance interfaces have confidence in the consistency of metadata structure accross chains.

## Proposal

Location: off-chain as json object stored on IPFS (mirrors group proposal):

```json
{
  "title": "",
  "authors": "",
  "summary": "",
  "details": "",
  "proposalForumURL": "",
  "voteOptionContext": ""
}
```

## Vote

Location: on-chain as json within 255 character limit (mirrors group vote):

```json
{
  "justification": ""
}
```