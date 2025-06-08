# ECEN 5598 Final project :Synergy-Implementation-in-Gem-5

Abstract—Modern memory systems must account
for both reliability and security, especially in high-
performance and cloud environments where data integrity
and confidentiality are critical. In this project, we first eval-
uated a baseline secure memory implementation in Gem5,
where integrity metadata such as Message Authentication
Codes (MACs) are stored separately and require additional
memory accesses for verification. Building on this, we im-
plemented and integrated Synergy a secure memory archi-
tecture that co-locates data and MACs in the same memory
block within the gem5 simulation framework. Synergy
reduces overhead by eliminating separate metadata fetches
and takes advantage of ECC encoded blocks for embedding
authentication. Our implementation modifies the secure
memory controller to support Synergy’s data layout by co
location of MAC. We simulate few benchmarks to compare
performance between the original and Synergy-enhanced
secure memory. The results show that Synergy improves
IPC and reduces MAC related access latency, validating its
effectiveness. We also propose our dynamic synergy model
and an extensive evaluation plan to evaluate synergy under
memory intensive workloads.



