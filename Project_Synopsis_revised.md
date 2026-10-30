# Cryptographically Secured Real-Time Network Intrusion Detection System with Merkle Tree-Validated SQLite Forensic Storage

This draft updates the cryptography choices in the existing synopsis. The
remaining objectives are project targets, not claims that they are already
implemented.

## Revised objective 2

Protect sensitive forensic telemetry before SQLite storage using AES-256-GCM
authenticated encryption. Document which fields remain searchable in plaintext
and which are encrypted, and measure encryption and insertion latency. Key
generation, storage, and access assumptions must be stated. DES and AES-CBC are
not used in the production storage path.

## Revised objective 6

Evaluate end-to-end capture and flow classification, AES-256-GCM insert latency
and storage overhead, Merkle proof verification time versus full checkpoint
verification, and classifier precision, recall, F1 score, and confusion matrix
on CICIDS2017. Include modification, deletion, and reordering tamper tests.

## Terminology correction

AES-256-GCM provides confidentiality and authentication for the encrypted
payload. Merkle roots and RSA signatures provide tamper evidence for checkpointed
records. A locally stored checkpoint does not by itself guarantee detection of
rollback by an attacker who can replace both the database and checkpoint files;
independent anchoring is needed for that claim.
