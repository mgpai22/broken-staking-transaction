# Broken Staking Transaction

## Update 2

Pi from Sundae confirmed that input sorting is the issue:

> ok so my guess was right 😅  That can get your evaluation to work, but the network won't accept it.
>
> So, when you submit a transaction, the node constructs the script context (also called "TxInfo") to pass into your smart contract.
>
> For some *UNHOLY UNGODLY FRUSTRATING HELLFIRE DAMNED* reason, it sorts the transaction inputs lexographically.
>
> So, while you have these inputs in your transaction:
> ```
> 0c687f064f4409d7ee16852f9b02b0c9feddd226907fc1d55c006836f3d8d3c5#0
> f17d3477595091dc56dbc16dc710f1edb10a02914b7184b8c0a161bb416bc9d9#1
> 1a497316e1bf39f53cdf6b2b4f7a3ea423ecddfe93db975046b4d37edb8b1aa5#2
> ```
>
> But, when it gets passed to your smart contract, it looks like
> ```
> 0c687f064f4409d7ee16852f9b02b0c9feddd226907fc1d55c006836f3d8d3c5#0
> 1a497316e1bf39f53cdf6b2b4f7a3ea423ecddfe93db975046b4d37edb8b1aa5#2
> f17d3477595091dc56dbc16dc710f1edb10a02914b7184b8c0a161bb416bc9d9#1
> ```
>
> now, the weird thing is that the redeemers in your transaction uses *this* order, rather than the order they are in the bytes of your transaction.
>
> So, in your case, you have 
> ```
>             [0, 0, 121_0([]), [0, 0]],
>             [0, 1, 121_0([_ ...]), [0, 0]],
>             [1, 0, 122_0([]), [0, 0]],
> ```
>
> This means
> ```
> The script on the 0th input in ledger order has redeemer 121_0([]), and a budget of [0,0]
>
> The script on the 1st input in ledger order has redeemer 121_0[(_ ...]), and a budget of [0,0]
>
> The script for the first token minted in ledger order has redeemer 122_0([]) and a budget of [0,0]
> ```
>
> In particular, that means your spend redeemers are pointing at these two inputs:
> ```
> 0c687f064f4409d7ee16852f9b02b0c9feddd226907fc1d55c006836f3d8d3c5#0
> 1a497316e1bf39f53cdf6b2b4f7a3ea423ecddfe93db975046b4d37edb8b1aa5#2
> ```
>
> and not these two
> ```
> 0c687f064f4409d7ee16852f9b02b0c9feddd226907fc1d55c006836f3d8d3c5#0
> f17d3477595091dc56dbc16dc710f1edb10a02914b7184b8c0a161bb416bc9d9#1
> ```
>
> So when you run the code here:
> ```
>  expect Some(stake_redeemer) =
>     pairs.get_first(tx.redeemers, Spend(stake_input.output_reference))
> ```
>
> It says "find me the first redeemer that is for `f17d3477595091dc56dbc16dc710f1edb10a02914b7184b8c0a161bb416bc9d9#1`... and according to the ledger, there *is* no such redeemer.
>
> if you instead change your redeemers to
> ```
>             [0, 0, 121_0([]), [0, 0]],
>             [0, 2, 121_0([_ ...]), [0, 0]], <--- N.B.!!!
>             [1, 0, 122_0([]), [0, 0]],
> ```
>
> I believe it should work.

## Update 1

It seems that how Apollo lib handles sorting could be an issue.

When the uplc eval source is modifed to remove sorting, evaluation is sucessful.

```rs
pub fn get_tx_in_info_v2(
    inputs: &[TransactionInput],
    utxos: &[ResolvedInput],
) -> Result<Vec<TxInInfo>, Error> {
    inputs
        .iter()
        .sorted()
        .map(|input| {
            let utxo = match utxos.iter().find(|utxo| utxo.input == *input) {
                Some(resolved) => resolved,
                None => return Err(Error::ResolvedInputNotFound(input.clone())),
            };
            let address = Address::from_bytes(match &utxo.output {
                TransactionOutput::Legacy(output) => output.address.as_ref(),
                TransactionOutput::PostAlonzo(output) => output.address.as_ref(),
            })
            .unwrap();

            match address {
                Address::Byron(_) => {
                    return Err(Error::ByronAddressNotAllowed);
                }
                Address::Stake(_) => {
                    return Err(Error::NoPaymentCredential);
                }
                _ => {}
            };

            Ok(TxInInfo {
                out_ref: utxo.input.clone(),
                resolved: sort_tx_out_value(&utxo.output),
            })
        })
        .collect()
}
```
[Context](https://github.com/aiken-lang/aiken/blob/a8c032935dbaf4a1140e9d8be5c270acd32c9e8c/crates/uplc/src/tx/script_context.rs#L566)




## Original Content
The broken transaction has the same structure as the working transaction. 

However, the broken transaction has an extra input and a change output. This SHOULD NOT affect evaluation or script execution this has been verified. 

The datum and redeemer data are the same!! The only difference is that the working transaction has the ex and mem units since it sucessfully evaluated.


Using a custom script debugger which allows me to execute the transaction with verbose traces found that the issue lies in `Spend[0]`.

This corresponds to input `0c687f064f4409d7ee16852f9b02b0c9feddd226907fc1d55c006836f3d8d3c5#0`.

this can be verifed by looking at the redeemer section of the [broken transaction](./broken_transaction/broken_transaction.json)

```
"ArrLegacyRedeemer": {
"arr_legacy_redeemer": [
    {
    "tag": "Spend",
    "index": 0,
    "data": {
        "constructor": 0,
        "fields": []
    },
    "ex_units": {
        "mem": 0,
        "steps": 0
    }
    },
    ...
```

Using tracing it outputed the issue to here
```
  trace @"stake_input_reference"
  trace stake_input.output_reference
  trace @"tx.redeemers"
  trace tx.redeemers
  expect Some(stake_redeemer) =
    pairs.get_first(tx.redeemers, Spend(stake_input.output_reference))
```

Output:
```
error: failed script execution
     Spend[0] the validator crashed / exited prematurely
        Trace proxy spend
        Trace D87980
        Trace {_ h'': {_ h'': 2200000 }, h'3918067C0E69E13F8163C64825EBD429B0BCE8177654F49F912368B2': {_ h'70616C6D5F7374616B696E675F70726F78795F67756172645F746F6B656E': 1 }, h'B7C5CD554F3E83C8AA0900A0C9053284A5348244D23D0406C28EAF4D': {_ h'50414C4D0A': 204109775022 } }
        Trace refund ? False
        Trace stake_input_reference
        Trace 121([_ h'F17D3477595091DC56DBC16DC710F1EDB10A02914B7184B8C0A161BB416BC9D9', 1])
        Trace tx.redeemers
        Trace {_ 122([_ 121([_ h'0C687F064F4409D7EE16852F9B02B0C9FEDDD226907FC1D55C006836F3D8D3C5', 0])]): 121([]), 122([_ 121([_ h'1A497316E1BF39F53CDF6B2B4F7A3EA423ECDDFE93DB975046B4D37EDB8B1AA5', 2])]): 121([_ 121([]), {}, {_ h'971D1DCDF9E5E000B6BEA063578BEDB7710E448D3E71D3552DFA75E4': 204109775022 }, {_ h'971D1DCDF9E5E000B6BEA063578BEDB7710E448D3E71D3552DFA75E4': [_ 121([_ 0, h'69D5D0D348B2CF45387989A6086329B8AFD0815A9C54302F8543087ED51F673B916C06933F487F645C8F5CBECB9A94B8597BE67A82E44203D5CEF92795B4F963469A69FE10CEF6788B7D3F3296E1920D6D74157C5616ACCB22137DFE8BCA443515F60B5B8361A5013C11593D644FCF9FCA644B7B9AD71CA3EC81A7397A6FD381']), 121([_ 0, h'65932991D5AB936AA616F976D60BBCA37804C21EFFF2ACF1E1BC6E7F6A6862969ABC83E9CC434211CAA29F95CC96A551F48974D5B6CA49854E58C530CCA4C90535EADBF2CF23F0D9F92049B66BD7F91EDF171FFD5CC88E81EA73A933C4B75CF0B31EC0B369EB51024B0B32D5C3D47D0B3012387F7A26D09B36973981A5BD56C3']), 123([_ 0, h'0068CF79E5E033B5AADCECC7A58A14EE7F9A1A2D91471286A39D8CF5EFC5F7D8', h'0C324E92777E527C5B1A7A7DC487733EA99BA25EF0CAF8CB934B89DB06B379AB'])] }, {}]), 121([_ h'3918067C0E69E13F8163C64825EBD429B0BCE8177654F49F912368B2']): 122([]) }
        Trace expect Some(stake_redeemer) =
                  pairs.get_first(tx.redeemers, Spend(stake_input.output_reference))
```

`stake_redeemer` corresponds to the redeemer of `f17d3477595091dc56dbc16dc710f1edb10a02914b7184b8c0a161bb416bc9d9#1`

looking at the redeemer of the [tx cbor](./broken_transaction/broken_transaction.cbor) in the [redeemer logs](./broken_transaction/broken_transaction_redeemer.txt)

that redeemer DOES EXIST. 

But for some reason, from the trace logs it seems to be non existant. 

The [working transaction](./working_transaction/working_transaction.cbor) has the same redeemer yet it throws no error. 

The working transaction was built with the library of an [apollo fork](https://github.com/zenGate-Global/apollo/tree/d169a805c82db182a76446295805cae7551c4b5d).

The broken transaction was built with the same fork but with a few [commits ahead](https://github.com/zenGate-Global/apollo/tree/37f3a9174ddd67aed203b799a3b8194b71e6b39f)

The diff can be found [here](https://github.com/zenGate-Global/apollo/compare/d169a80...37f3a91), however, the modifications should not be an issue. 

The two offchain engines are different, however, the transaction building logic has not change, so its unclear what the problem is. 
