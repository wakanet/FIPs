---
fip: "0024"
title: BatchBalancer & BatchDiscount Post-HyperDrive Adjustment
author: zx, jbenet, zenground0, momack2
discussions-to: https://github.com/filecoin-project/FIPs/issues/173
status: Final
type: Technical
category: Core
created: 2021-09-21
spec-sections: 
  - specs-actors
review-period-end: 2021-10-11
requires (*optional*): <FIP number(s)>
replaces (*optional*): <FIP number(s)>
---

 `fip-balancer_post_hyperdrive.md`.

## Simple Summary 简述
Adjust BatchBalancer and BatchDiscount to match observed network growth rate post-HyperDrive & apply mechanism consistently to both ProveCommitAggregate & PreCommitBatch.

调整BatchBalancer和BatchDiscount，以匹配HyperDrive后观察到的网络增长率，并将机制一致地应用于ProveCommitgGate和PreCommitBatch。

## Abstract 摘要
BatchBalancer and BatchDiscount were introduced in FIP13 HyperDrive to [align participants' incentives with the long-term health and success of the network](https://github.com/filecoin-project/FIPs/blob/master/FIPS/fip-0013.md#incentive-considerations). At that time, the parameter values were set to accommodate an onboarding rate of up to 1 EiB/day. However, given that the network is growing at ~60PiB/day nearly 3 months after HyperDrive (up roughly 2x from ~30-35PiB/day prior to the upgrade), these parameters must be re-calibrated for the long-term success of the network and its participants. This FIP proposes to increase BatchBalancer to 5 nanoFIL. 

FIP13 HyperDrive中引入了BatchBalancer和BatchDiscount，以[使参与者的激励与网络的长期健康和成功保持一致](https://github.com/filecoin-project/FIPs/blob/master/FIPS/fip-0013.md#incentive-considerations)。当时，参数值被设置为适应高达1 EiB/天的到达率。然而，鉴于HyperDrive推出近3个月后，网络以每天约60PiB的速度增长（从升级前的每天约30-35PiB增长了约2倍），为了网络及其参与者的长期成功，必须重新校准这些参数。该FIP建议将BatchBalancer增加至5 nanoFIL。

In addition, BatchBalancer and BatchDiscount were only applied at ProveCommitAggregate, not at PreCommitBatch. The protocol should also apply the same mechanism at PreCommitBatch to be in line with the spirit and considerations in [FIP13](https://github.com/filecoin-project/FIPs/blob/master/FIPS/fip-0013.md).

此外，BatchBalancer和BatchDiscount仅适用于ProveCommitAggregate，而不适用于预提交批次。协议还应在预提交批次应用相同的机制，以符合[FIP13](https://github.com/filecoin-project/FIPs/blob/master/FIPS/fip-0013.md)中的精神和考虑.

## Change Motivation

**BatchBalancer.** The initial parameter values of BatchBalancer and BatchDiscount were set after considering storage onboarding expectations, equilibrium network BaseFee, return on providing storage on Filecoin, cost of PublishStorageDeals, and protocol revenue. HyperDrive unlocked 10-25x storage onboarding capacity and the parameters were provisioning for growth on the order of onboarding 1 EiB/day. However, the network is not growing at that level, resulting in a loss in protocol revenue that hurts all participants long-term. Hence, the protocol needs to adjust its parameters accordingly based on current network growth rate (~60PiB/day) and current growth projections to >150PiB/day. The protocol may re-calibrate this parameter once the network significantly surpasses that growth rate. Work is being done to design mechanisms to set these protocol parameters algorithmically based on on-chain states in the future, so that the protocol re-calibrates automatically based on participants’ behavior.

**PreCommitBatch.** During HyperDrive, BatchBalancer and BatchDiscount were initially only applied at ProveCommitAggregate to simplify implementation. However, storage onboarding is a two-step process and the same mechanism should be applied at PreCommitBatch.

## Specification
- Decompose **SingleProofGasUsage** into **SinglePreCommitGasUsage** and **SingleProveCommitGasUsage**.
- Replace **SingleProofGasUsage** with **SingleProveCommitGasUsage** at **ProveCommitAggregate**. Apply the same **PayBatchGasCharge** function at **PreCommitBatch** and replace **SingleProveCommitGasUsage** with **SinglePreCommitGasUsage** at **PreCommitBatch**. 
- Increase **BatchBalancer** value to 5 nanoFIL.
- Currently, the following charge is calculated for each **ProveCommitAggregate** message.

```
func PayBatchGasCharge(numProofsBatched, BaseFee) {
    // Cryptoecon Params (need to be updated if verification benchmarks change)
    BatchDiscount = 1/20 unitless
    BatchBalancer = 5 nanoFIL
    SinglePreCommitGasUsage = 16433324.1825
    SingleProveCommitGasUsage = 49299972.5475

    // Calculating BatchGasCharge at ProveCommitAggregate
    numProofsBatched = <# of proofs in this batched operation>
    BatchGasFee = Max(BatchBalancer, BaseFee)
    BatchGasCharge = BatchGasFee * SingleProveCommitGasUsage *  numProofsBatched *     BatchDiscount

    // Pay for the batch
    PayNetFee(BatchGasCharge) // this can be a msg.Send to f99. Does not affect BaseFee
    // normal gas for the verification computation is paid as usual (using & affecting BaseFee)
}
```

## Design Rationale
Reuse modular function PayBatchGasCharge.

## Backwards Compatibility
This FIP changes actors behavior so it requires a new filecoin network version.

## Test Cases

- Measure total batch gas charge when only PreCommitBatch is used. Confirm it is 25% of both PreCommitBatch and ProveCommitAggregate batch gas charge
- Measure total batch gas charge when only ProveCommitBatch is used. Confirm it is 75% of both combined.
- Test that with the new BatchBalancer parameter value the cross over BaseFee and batch gas charges across a range of BaseFee are as expected.

## Security Considerations
This FIP does not touch underlying proofs or security.

## Incentive Considerations

Including BatchBalancer and BatchDiscount at PreCommitBatch follows from the [incentive considerations in FIP13](https://github.com/filecoin-project/FIPs/blob/master/_fips/fip-0013.md#incentive-considerations) and protects the interest of the protocol which then benefits all network participants.

There were many tradeoffs and considerations in setting the parameter values for BatchBalancer and BatchDiscount given the wide range of stakeholders and interests. Adjusting BatchBalancer and BatchDiscount first changes the equilibrium crossover BaseFee where the unit cost of not aggregating exceeds that of aggregation and hence creating an incentive to aggregate to free up chain capacity. The crossover BaseFee and be computed in close form with the following formula.

```
CrossoverNetworkBaseFee = BatchBalancer * BatchDiscount * SingleProofGasUsage / (SingleProofGasUsage - BatchProofGasUsage / NumProofsBatched)
```

This equilibrium crossover BaseFee in turn impacts other metrics such as the cost of PublishStorageDeals, protocol revenue for a particular growth rate, and the return of storage provision. You can find the table below for a comparison between existing and proposed Balancer values. Note that the estimate can be off and network participants can choose to deviate from the equilibrium.

| BatchBalancer | Estimated Crossover Network BaseFee | Estimated PublishStorageDeals Network Fee | Estimated 32 GiB Sector Network Fee | Estimated 32 GiB Sector Annual Return | Estimated Daily Protocol Revenue at Current Growth Rate |
| ----------- | ----------- | ----------- | ----------- | ----------- | ----------- |
| 2 nanoFIL   | ~0.12 nanoFIL       | ~0.0064 FIL       | ~0.0085 FIL       | ~0.165 FIL       | ~13k FIL       |
| 5 nanoFIL   | ~0.32 nanoFIL        | ~0.016 FIL       | ~0.021 FIL       | ~0.165 FIL         | ~34k FIL       |
| 7 nanoFIL   | ~0.45 nanoFIL        | ~0.023 FIL       | ~0.030 FIL       | ~0.165 FIL       | ~47k FIL       |
| 10 nanoFIL   | ~0.64 nanoFIL        | ~0.032 FIL       | ~0.042 FIL       | ~0.165 FIL       | ~68k FIL       |

Note that Network Fee is halved for 64 GiB sectors and the above unit cost numbers go down further as the number of aggregation increases. This FIP currently proposes 5 nanoFIL for BatchBalancer. However, if the community, especially storage providers, believe that 7 nanoFIL is a better value, we would support that change too.

## Product Considerations
All FIPs should optimize for long-term network health. This FIP balances many difficult tradeoffs to achieve that goal - aiming to align all network participants with the network’s long term health and growth. Recalibrating network parameters to observed network growth rates as this FIP proposes is critical to that aim. However, one negative side-effect is a roughly-estimated ~.009 FIL increase in the gas costs of PublishStorageDeal messages, which has negative product considerations for storage providers accepting storage deals / participating in the deal-making market. However, the significant overall demand and return for storage providers accepting FIL+ deals after this increase is still strongly rational for storage providers as amortized over the length of the deal.

## Implementation
Specs-actors PR: TODO

## Copyright
Copyright and related rights waived via [CC0](https://creativecommons.org/publicdomain/zero/1.0/).

