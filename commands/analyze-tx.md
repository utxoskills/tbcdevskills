---
name: analyze-tx
description: Analyze a TBC transaction from raw decode data and classify its business meaning.
---

# Analyze TBC Transaction

When the user gives a TBC `txid`, do this:

1. Use the `transaction-analysis` skill.
2. If the transaction involves Pool, FTLP, or similar-looking outputs, also use `contract-patterns`.
3. If the transaction appears mixed or multi-party, switch to atomic analysis and asset-flow reasoning.
4. Return:
   - transaction type
   - roles and addresses
   - asset and amount movement
   - payment vs change reasoning
   - why the classification is correct

Also consult `tbc-dev/SKILL.md` if you need the complete canonical reference.
