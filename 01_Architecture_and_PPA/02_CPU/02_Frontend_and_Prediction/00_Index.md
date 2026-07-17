# CPU Frontend and Prediction

This subdomain owns the machine that converts a byte-addressed instruction stream into a sustained stream of correctly steered operations.

| Chapter | Owns |
|---|---|
| [Branch Prediction Deep Dive](01_Branch_Prediction_Deep_Dive.md) | BTB, TAGE, indirect prediction, RAS, FTQ, accuracy/latency trade-offs |
| [Fetch, Decode, and µop Delivery](02_Fetch_Decode_and_Uop_Delivery.md) | I-cache/ITLB delivery, alignment, decode, µop caches, queues, bandwidth accounting |

Treat prediction accuracy and delivery bandwidth as one coupled frontend: either can starve a wide backend.

**Up:** [CPU](../00_Index.md) · [Architecture + PPA](../../00_Index.md)
