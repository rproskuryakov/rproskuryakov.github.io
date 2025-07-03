+++
title = "Security Strategies in Machine Learning"
date = "2019-03-09"
draft = true
[taxonomies]
tags=["homomorphic-encryption", "security", "differencial-privacy"]
[extra]
comment = false
+++


# Homomorphic Encryption

Inference on data encrypted with Homomorphic Encryption


https://github.com/OpenMined/PySyft
https://github.com/microsoft/SEAL
https://github.com/FederatedAI/FATE
https://github.com/homenc/HElib
https://github.com/Microsoft/CryptoNets
https://crypten.ai/


Homomorphic Encryption (HE) and Differential Privacy (DP) are two powerful tools for safeguarding sensitive data, and they often complement each other in various applications. While they serve distinct purposes, their combination can offer robust privacy protection.
Homomorphic Encryption
HE allows computations to be performed directly on encrypted data without decrypting it first.

1 This ensures data confidentiality as it remains protected throughout the computation process. 2

Differential Privacy
DP adds noise to data to prevent inference about specific individuals from the released data.

It ensures that the presence or absence of an individual's data in the dataset has a minimal impact on the result.

Federated Learning: In federated learning, HE can protect the privacy of individual models, while DP can be used to add noise to model updates before sharing them with a central server.



Security & Privacy

Encoding of data. access control

https://www.llamaindex.ai/blog/retrieving-privacy-safe-documents-over-a-network

https://www.alexanderjunge.net/blog/short-diff-privacy-rag/