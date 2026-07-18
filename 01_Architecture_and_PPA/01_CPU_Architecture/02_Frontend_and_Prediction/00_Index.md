# Central Processing Unit (CPU) Architecture › Frontend and Prediction

> **Abbreviation key — skim now and return as needed:** branch target buffer (BTB); program counter (PC).

**Plain-language purpose:** Explain how a CPU chooses the next instruction address and turns instruction bytes into a steady stream of internal operations before it knows which control path is correct.

## Terms introduced here

| Term | Meaning |
|---|---|
| program counter (PC) | address of the instruction being fetched |
| branch predictor | guesses whether and where control flow changes |
| branch target buffer (BTB) | remembers target addresses of previously seen branches |
| micro-operation (µop) | internal operation produced by decoding an instruction |
| redirect | replacement fetch address after a prediction or exception changes |

## Reading order

1. [Branch Prediction](01_Branch_Prediction_Deep_Dive.md) — direction, target, return, confidence, and recovery cost.
2. [Fetch, Decode, and µop Delivery](02_Fetch_Decode_and_Uop_Delivery.md) — byte supply, alignment, decode width, queues, and sustained delivery.

**Comes from:** [Core Foundations](../01_Core_Foundations/00_Index.md).
**Hands off to:** [Out-of-Order Backend](../03_Out_of_Order_Backend/00_Index.md).

---

[CPU Architecture](../00_Index.md) · [Architecture book](../../00_Index.md)
