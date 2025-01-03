# Frequently Asked Questions

### Transaction Types and Processing

**Q: There are two types of transactions: ones that modify on-chain state (like withdrawals) and ones that only modify the Merkle tree database. Is there any difference in how the rollup processes these? How do we distinguish which transactions need to be settled on-chain?**

A: The off-chain processing is actually the same for both types. You can think of the transactions that modify on-chain state as having "side effects". The rollup processes them identically, but some transactions will trigger additional on-chain state changes.

### zkWasm Application Migration and Upgrades

**Q: Why don't we need to modify the Verifier contract when upgrading a zkWasm application?**

A: This is because zkWasm uses image as input to the circuit, and there's an image commitment that checks which image the proof is for. When you upgrade the logic, you don't need to change the circuit because the entire zkWasm virtual machine is written in circuits. This makes the verifier universal and applicable across different versions of your application logic.

The image is part of the input, and the commitment mechanism ensures verification of which specific image (application version) the proof corresponds to. This design allows for logic upgrades without requiring changes to the underlying verification infrastructure.


**Q: In MIGRATE, why do we only get the Merkle root from the contract? How is this sufficient for an upgrade?**

A: The upgrade mechanism only allows changes to the business logic, not the data structure. Since the data structure remains unchanged, updating the Merkle root is sufficient. The mini-rollup doesn't need to be modified because the underlying data structure remains the same.