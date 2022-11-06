## How to fund a channel with utxos external to the CLN wallet using PSBT

- We use cln to initiate the funding with `fundchannel_start`, which will give us the address for the output of the funding transaction;
- We than use a bitcoind wallet to fund the transaction and pass a funded (unsigned) psbt to `cln`


```
$ cln fundchannel_start $nodeid 100000
{
   "funding_address": "bcrt1qfrumv2af0peamcju6p03deywayqjkn635jvy5az8rs35p670uqtsu3v2ma",
   "scriptpubkey": "002048f9b62ba97873dde25cd05f16e48ee9012b4f51a4984a74471c2340ebcfe017",
   "warning_usage": "The funding transaction MUST NOT be broadcast until after channel establishment has been successfully completed by running `fundchannel_complete`"
}
```

- Now we use the rpc `walletcreatefundedpsbt` to create the funding transaction using coins from the bitcoin core wallet

```
$ bitcoin-cli walletcreatefundedpsbt "[]" "[{\"bcrt1qfrumv2af0peamcju6p03deywayqjkn635jvy5az8rs35p670uqtsu3v2ma\": 0.00100000}]"
{
  "psbt": "cHNidP8BAIkCAAAAAeb9rnoFitOsGTQ9aXBZ0NclUohloAgNT1qjJsf3XaTHAQAAAAD+////Ai6JDiQBAAAAIlEgJrOWy/0tGmIT4NHfrBvHRnQev0dEI06+/+K5Cu9U4A6ghgEAAAAAACIAIEj5tiupeHPd4lzQXxbkjukBK09RpJhKdEccI0Drz+AXAAAAAAABAHECAAAAAdm3QbdMwDOZzbrSeYmDN1ceWRRGFQ2tuCaC6Kwnh+nwAAAAAAD+////AgDh9QUAAAAAFgAUeZ336s8AEfuy5ZZIBAzsnJrFZltzEBAkAQAAABYAFP9S1ZrxyLdU04TIHqMfzLrEyK0tZQAAAAEBH3MQECQBAAAAFgAU/1LVmvHIt1TThMgeox/MusTIrS0iBgPpk0/YIkw62cYAuRQ93VhwWw+cYXkUmmxUxrOzcT+btxjpxLCxVAAAgAEAAIAAAACAAQAAAAEAAAAAAAA=",
  "fee": 0.00000165,
  "changepos": 0
}
```

- Now that we have a txid for the funding transaction, we send back the psbt to `cln`:

```
$ cln fundchannel_complete 0253ee7da51677badd5f28a1aabdb6970fc8badbc964d0306680946a463b0938d6 cHNidP8BAIkCAAAAAeb9rnoFitOsGTQ9aXBZ0NclUohloAgNT1qjJsf3XaTHAQAAAAD+////Ai6JDiQBAAAAIlEgJrOWy/0tGmIT4NHfrBvHRnQev0dEI06+/+K5Cu9U4A6ghgEAAAAAACIAIEj5tiupeHPd4lzQXxbkjukBK09RpJhKdEccI0Drz+AXAAAAAAABAHECAAAAAdm3QbdMwDOZzbrSeYmDN1ceWRRGFQ2tuCaC6Kwnh+nwAAAAAAD+////AgDh9QUAAAAAFgAUeZ336s8AEfuy5ZZIBAzsnJrFZltzEBAkAQAAABYAFP9S1ZrxyLdU04TIHqMfzLrEyK0tZQAAAAEBH3MQECQBAAAAFgAU/1LVmvHIt1TThMgeox/MusTIrS0iBgPpk0/YIkw62cYAuRQ93VhwWw+cYXkUmmxUxrOzcT+btxjpxLCxVAAAgAEAAIAAAACAAQAAAAEAAAAAAAA=
{
   "channel_id": "f5e659ec7aaf62ba2dac5d9b9fd34057d36768513bc74c0e91ac91db39010f23",
   "commitments_secured": true
}
```

We can check the channel status using the `listfunds` command:

```
$ cln listfunds
{
   "outputs": [],
   "channels": [
      {
         "peer_id": "0253ee7da51677badd5f28a1aabdb6970fc8badbc964d0306680946a463b0938d6",
         "connected": true,
         "state": "CHANNELD_AWAITING_LOCKIN",
         "channel_sat": 100000,
         "our_amount_msat": "100000000msat",
         "channel_total_sat": 100000,
         "amount_msat": "100000000msat",
         "funding_txid": "220f0139db91ac910e4cc73b516867d35740d39f9b5dac2dba62af7aec59e6f5",
         "funding_output": 1
      }
   ]
}
```

Now we can complete the psbt by signing it, and wait for the required confirmations to start using the channel.
We use the rpc's `walletprocesspsbt`, `finalizepsbt` and `sendrawtransaction`:

```
$ bitcoin-cli walletprocesspsbt cHNidP8BAIkCAAAAAeb9rnoFitOsGTQ9aXBZ0NclUohloAgNT1qjJsf3XaTHAQAAAAD+////Ai6JDiQBAAAAIlEgJrOWy/0tGmIT4NHfrBvHRnQev0dEI06+/+K5Cu9U4A6ghgEAAAAAACIAIEj5tiupeHPd4lzQXxbkjukBK09RpJhKdEccI0Drz+AXAAAAAAABAHECAAAAAdm3QbdMwDOZzbrSeYmDN1ceWRRGFQ2tuCaC6Kwnh+nwAAAAAAD+////AgDh9QUAAAAAFgAUeZ336s8AEfuy5ZZIBAzsnJrFZltzEBAkAQAAABYAFP9S1ZrxyLdU04TIHqMfzLrEyK0tZQAAAAEBH3MQECQBAAAAFgAU/1LVmvHIt1TThMgeox/MusTIrS0iBgPpk0/YIkw62cYAuRQ93VhwWw+cYXkUmmxUxrOzcT+btxjpxLCxVAAAgAEAAIAAAACAAQAAAAEAAAAAAAA=
{
  "psbt": "cHNidP8BAIkCAAAAAeb9rnoFitOsGTQ9aXBZ0NclUohloAgNT1qjJsf3XaTHAQAAAAD+////Ai6JDiQBAAAAIlEgJrOWy/0tGmIT4NHfrBvHRnQev0dEI06+/+K5Cu9U4A6ghgEAAAAAACIAIEj5tiupeHPd4lzQXxbkjukBK09RpJhKdEccI0Drz+AXAAAAAAABAHECAAAAAdm3QbdMwDOZzbrSeYmDN1ceWRRGFQ2tuCaC6Kwnh+nwAAAAAAD+////AgDh9QUAAAAAFgAUeZ336s8AEfuy5ZZIBAzsnJrFZltzEBAkAQAAABYAFP9S1ZrxyLdU04TIHqMfzLrEyK0tZQAAAAEBH3MQECQBAAAAFgAU/1LVmvHIt1TThMgeox/MusTIrS0BCGsCRzBEAiB3tlhaM+/4HRLfGZAIXuAMhRHRUHkIg3eRSsXxh6+3DwIgKIfV6XF2zoubFMbh8ze+s4OzqGSyE+ofERLx1YLQh68BIQPpk0/YIkw62cYAuRQ93VhwWw+cYXkUmmxUxrOzcT+btwAAAA==",
  "complete": true
}
```

```
bitcoin-cli finalizepsbt cHNidP8BAIkCAAAAAeb9rnoFitOsGTQ9aXBZ0NclUohloAgNT1qjJsf3XaTHAQAAAAD+////Ai6JDiQBAAAAIlEgJrOWy/0tGmIT4NHfrBvHRnQev0dEI06+/+K5Cu9U4A6ghgEAAAAAACIAIEj5tiupeHPd4lzQXxbkjukBK09RpJhKdEccI0Drz+AXAAAAAAABAHECAAAAAdm3QbdMwDOZzbrSeYmDN1ceWRRGFQ2tuCaC6Kwnh+nwAAAAAAD+////AgDh9QUAAAAAFgAUeZ336s8AEfuy5ZZIBAzsnJrFZltzEBAkAQAAABYAFP9S1ZrxyLdU04TIHqMfzLrEyK0tZQAAAAEBH3MQECQBAAAAFgAU/1LVmvHIt1TThMgeox/MusTIrS0BCGsCRzBEAiB3tlhaM+/4HRLfGZAIXuAMhRHRUHkIg3eRSsXxh6+3DwIgKIfV6XF2zoubFMbh8ze+s4OzqGSyE+ofERLx1YLQh68BIQPpk0/YIkw62cYAuRQ93VhwWw+cYXkUmmxUxrOzcT+btwAAAA==
{
  "hex": "02000000000101e6fdae7a058ad3ac19343d697059d0d725528865a0080d4f5aa326c7f75da4c70100000000feffffff022e890e240100000022512026b396cbfd2d1a6213e0d1dfac1bc746741ebf4744234ebeffe2b90aef54e00ea08601000000000022002048f9b62ba97873dde25cd05f16e48ee9012b4f51a4984a74471c2340ebcfe01702473044022077b6585a33eff81d12df1990085ee00c8511d15079088377914ac5f187afb70f02202887d5e97176ce8b9b14c6e1f337beb383b3a864b213ea1f1112f1d582d087af012103e9934fd8224c3ad9c600b9143ddd58705b0f9c6179149a6c54c6b3b3713f9bb700000000",
  "complete": true
}
```

```
bitcoin-cli sendrawtransaction 02000000000101e6fdae7a058ad3ac19343d697059d0d725528865a0080d4f5aa326c7f75da4c70100000000feffffff022e890e240100000022512026b396cbfd2d1a6213e0d1dfac1bc746741ebf4744234ebeffe2b90aef54e00ea08601000000000022002048f9b62ba97873dde25cd05f16e48ee9012b4f51a4984a74471c2340ebcfe01702473044022077b6585a33eff81d12df1990085ee00c8511d15079088377914ac5f187afb70f02202887d5e97176ce8b9b14c6e1f337beb383b3a864b213ea1f1112f1d582d087af012103e9934fd8224c3ad9c600b9143ddd58705b0f9c6179149a6c54c6b3b3713f9bb700000000

220f0139db91ac910e4cc73b516867d35740d39f9b5dac2dba62af7aec59e6f5
```

The funding transaction was signed and broadcast. After 6 confirmations the channel is ready to be used for payments:

```
$ cln listfunds
{
   "outputs": [],
   "channels": [
      {
         "peer_id": "0253ee7da51677badd5f28a1aabdb6970fc8badbc964d0306680946a463b0938d6",
         "connected": true,
         "state": "CHANNELD_NORMAL",
         "short_channel_id": "103x2x1",
         "channel_sat": 100000,
         "our_amount_msat": "100000000msat",
         "channel_total_sat": 100000,
         "amount_msat": "100000000msat",
         "funding_txid": "220f0139db91ac910e4cc73b516867d35740d39f9b5dac2dba62af7aec59e6f5",
         "funding_output": 1
      }
   ]
}
```


