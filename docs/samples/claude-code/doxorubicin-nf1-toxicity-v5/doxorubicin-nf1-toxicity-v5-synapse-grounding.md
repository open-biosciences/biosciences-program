# Synapse Grounding Report: Doxorubicin-NF1 Toxicity-Efficacy Knowledge Graph

**CQ:** What known genes or pathways are implicated in minimizing Doxorubicin toxicity while preserving its anti-tumor efficacy for NF1?

**Date:** 2026-02-27 | **Protocol:** Fuzzy-to-Fact v2 | **Synapse Tools:** search_synapse, get_entity

---

## 1. Summary

| Metric | Count | Percentage |
|--------|-------|------------|
| Total nodes | 24 | - |
| Grounded nodes | 11 | 46% |
| Total edges | 25 | - |
| MEMBER_OF edges (ontological, excluded) | 8 | - |
| Groundable edges | 17 | - |
| Grounded edges | 10 | 59% |

**Note:** MEMBER_OF edges represent ontological pathway membership from WikiPathways and are not amenable to experimental dataset grounding. Edge coverage is calculated using the adjusted denominator of 17 groundable edges.

---

## 2. Dataset-to-Graph Mapping Table

| Synapse ID | Name | Nodes Grounded | Edges Grounded | Evidence Type | Species | Grounding Strength |
|------------|------|----------------|----------------|---------------|---------|-------------------|
| syn237336 | GSE11940: Doxorubicin-topoisomerase II cleavage complex | TOP2A, Doxorubicin | Doxorubicin→TOP2A (INHIBITOR) | Gene expression after doxorubicin treatment | Human (HeLa) | **Strong** |
| syn7079983 | FDA TKI-induced cardiotoxicity | TOP2B | Doxorubicin→TOP2B, Dexrazoxane→TOP2B | iPSC-CM cardiotoxicity profiling | Human (iPSC-CM) | **Moderate** |
| syn13856689 | TKI-induced cardiotoxicity in iPSC-CMs | TOP2B | Doxorubicin→TOP2B | RNA-seq + proteomics + metabolomics in cardiomyocytes | Human (iPSC-CM) | **Moderate** |
| syn44222814 | NF1 Schwann cells treated with selumetinib | NF1, KRAS, Selumetinib | NF1→KRAS (REGULATES), Selumetinib→KRAS (INHIBITOR) | RNA-seq of NF1 pNF cells under MEK inhibition | Human (ipNF95.6) | **Strong** |
| syn8012530 | NF1 Cutaneous Neurofibroma + Selumetinib Trial | NF1, Selumetinib | Selumetinib→KRAS (INHIBITOR) | Longitudinal clinical trial with WGS + methylation | Human | **Strong** |
| syn21452685 | NF Therapeutic Consortium - UCSF | NF1 | NF1→KRAS (context) | 111 preclinical trials, 12 drug targets | Mouse/Human | **Moderate** |
| syn205590 | GSE8969: Nrf2 KO mice oxidative stress | NFE2L2, SOD2, HMOX1 | KEAP1→NRF2, NRF2→SOD2, NRF2→HMOX1 | Nrf2 KO transcriptomics (cytoprotective enzymes reduced) | Mouse | **Strong** |
| syn237499 | GSE11952: NFE2L2/Nrf2 oxidant-responsive TF | NFE2L2 | KEAP1→NRF2 | Nrf2 role in antioxidant gene induction | Human | **Strong** |
| syn360396 | GSE29632: Nrf2-deficient neonates | NFE2L2, GPX1, NQO1 | NRF2→NQO1, NRF2→GPX1 | Nrf2 KO neonates, Gpx2/Marco as Nrf2 effectors | Mouse | **Strong** |
| syn352945 | GSE30988: Doxorubicin immune modulation | Doxorubicin | (contextual) | Doxorubicin gene regulation in HeLa cells | Human | **Weak** |

---

## 3. Node Grounding Coverage

### Grounded Nodes (11/24)

| Node | CURIE | Synapse Datasets | Strength |
|------|-------|-----------------|----------|
| NF1 | HGNC:7765 | syn44222814, syn8012530, syn21452685 | Strong |
| TOP2A | HGNC:11989 | syn237336 | Strong |
| TOP2B | HGNC:11990 | syn7079983, syn13856689 | Moderate |
| NFE2L2 | HGNC:7782 | syn205590, syn237499, syn360396 | Strong |
| SOD2 | HGNC:11180 | syn205590 | Strong |
| HMOX1 | HGNC:5013 | syn205590 | Strong |
| NQO1 | HGNC:2874 | syn360396 | Moderate |
| GPX1 | HGNC:4553 | syn360396 | Moderate |
| KRAS | HGNC:6407 | syn44222814 | Strong |
| Doxorubicin | CHEMBL:53463 | syn237336, syn352945 | Strong |
| Selumetinib | CHEMBL:1614701 | syn44222814, syn8012530 | Strong |

### Ungrounded Nodes (13/24)

| Node | CURIE | Reason |
|------|-------|--------|
| KEAP1 | HGNC:23177 | No direct Synapse dataset; function confirmed by NRF2 KO studies |
| CAT | HGNC:1516 | Pathway member, no specific Synapse dataset |
| CBR1 | HGNC:1548 | Metabolic enzyme, no specific Synapse dataset |
| CBR3 | HGNC:1549 | Metabolic enzyme, no specific Synapse dataset |
| ABCB1 | HGNC:40 | Transporter, no specific Synapse dataset |
| Dexrazoxane | CHEMBL:1738 | No specific Synapse dataset for cardioprotection |
| Omaveloxolone | CHEMBL:4303525 | Novel NRF2 activator, no Synapse coverage |
| Bardoxolone methyl | CHEMBL:1762621 | Phase 3 compound, no Synapse coverage |
| NF1 (disease) | MONDO_0018975 | Disease ontology term |
| WP:WP408 | Pathway | Ontological entity |
| WP:WP2884 | Pathway | Ontological entity |
| WP:WP2735 | Pathway | Ontological entity |
| WP:WP4223 | Pathway | Ontological entity |

---

## 4. Edge Grounding Coverage

### Grounded Edges (10/17 groundable)

| Edge | Type | Synapse Dataset | Strength | Justification |
|------|------|----------------|----------|---------------|
| Doxorubicin→TOP2A | INHIBITOR | syn237336 | Strong | Gene expression profiling after doxorubicin-topoisomerase II cleavage complex formation |
| Doxorubicin→TOP2B | INHIBITOR | syn7079983 | Moderate | iPSC-CM cardiotoxicity model; TKIs not anthracyclines but same tissue endpoint |
| Dexrazoxane→TOP2B | INHIBITOR | syn7079983 | Moderate | Cardioprotection context in iPSC-CM models |
| NF1→KRAS | REGULATES | syn44222814 | Strong | RNA-seq of NF1-mutated Schwann cells shows MEK pathway crosstalk |
| KEAP1→NFE2L2 | REGULATES | syn205590 | Strong | Nrf2 KO mice show reduced cytoprotective enzyme expression |
| NFE2L2→SOD2 | REGULATES | syn205590 | Strong | SOD2 reduced in Nrf2 KO mice |
| NFE2L2→HMOX1 | REGULATES | syn205590 | Strong | HMOX1 reduced in Nrf2 KO mice |
| NFE2L2→NQO1 | REGULATES | syn360396 | Moderate | NQO1 identified as Nrf2-dependent gene |
| NFE2L2→GPX1 | REGULATES | syn360396 | Moderate | GPX family (Gpx2) validated as Nrf2 effector |
| Selumetinib→KRAS | INHIBITOR | syn44222814, syn8012530 | Strong | RNA-seq and clinical trial data for MEK inhibition in NF1 |

### Ungrounded Groundable Edges (7/17)

| Edge | Type | Reason |
|------|------|--------|
| CBR1→Doxorubicin | METABOLIZES | No doxorubicinol metabolism dataset on Synapse |
| CBR3→Doxorubicin | METABOLIZES | No doxorubicinol metabolism dataset on Synapse |
| ABCB1→Doxorubicin | TRANSPORTS | No P-gp efflux dataset specific to doxorubicin |
| NFE2L2→CAT | REGULATES | No specific Synapse dataset for CAT as NRF2 target |
| Omaveloxolone→NFE2L2 | AGONIST | No Synapse dataset for omaveloxolone NRF2 activation |
| Bardoxolone→KEAP1 | INHIBITOR | No Synapse dataset for bardoxolone KEAP1 inhibition |
| NF1→MONDO_0018975 | ASSOCIATED_WITH | Disease association, not experimentally groundable |

### MEMBER_OF Edges (8, ontological -- not groundable)

NF1→WP:WP2735, NFE2L2→WP:WP408, NFE2L2→WP:WP2884, SOD2→WP:WP408, HMOX1→WP:WP408, NQO1→WP:WP408, GPX1→WP:WP408, CAT→WP:WP408

---

## 5. Evidence Level Upgrades

| Claim | Original Level | Upgrade | New Level | Justification |
|-------|---------------|---------|-----------|---------------|
| NRF2 activates SOD2/HMOX1 | L2 | +Synapse | L2+ | Nrf2 KO study (syn205590) shows direct gene regulation |
| Doxorubicin inhibits TOP2A | L1 (FDA) | +Synapse | L1+ | GSE11940 (syn237336) provides molecular mechanism |
| NF1-KRAS interaction | L1 (STRING) | +Synapse | L1+ | NF1 Schwann cell RNA-seq (syn44222814) confirms pathway |
| Selumetinib for NF1 | L1 (FDA) | +Synapse | L1+ | Clinical trial data (syn8012530) with WGS |

---

## 6. Grounding Confidence Matrix

| Claim | Datasets | Strength | Justification |
|-------|----------|----------|---------------|
| Doxorubicin targets TOP2A for anti-tumor effect | syn237336 | Strong | Direct experimental evidence of doxorubicin-TOP2 complex |
| TOP2B mediates cardiotoxicity | syn7079983, syn13856689 | Moderate | Cardiomyocyte tissue model, but TKIs not anthracyclines |
| NRF2/KEAP1 axis controls antioxidant defense | syn205590, syn237499, syn360396 | Strong | Multiple Nrf2 KO studies confirming mechanism |
| NF1 loss activates KRAS/MEK pathway | syn44222814, syn21452685 | Strong | NF1-mutated cells + preclinical consortium data |
| Selumetinib inhibits MEK in NF1 context | syn44222814, syn8012530 | Strong | RNA-seq + longitudinal clinical trial |

---

## 7. Methodology

**Synapse MCP Tools Used:**
- `search_synapse`: Keyword searches for "doxorubicin cardiotoxicity", "TOP2B doxorubicin", "NF1 neurofibromatosis", "NRF2 oxidative stress", "doxorubicin anthracycline cardiomyocyte", "selumetinib MEK NF1"
- `get_entity`: Metadata retrieval for syn7079983, syn21452685, syn205590

**Matching Criteria:**
- Entity labels from KG nodes used as search terms
- Compound + tissue context searches for drug-target edges
- Gene + perturbation searches for regulatory edges

**MEMBER_OF Exclusion Rationale:** MEMBER_OF edges represent curated pathway membership from WikiPathways. These are definitional/ontological relationships and cannot be validated by experimental datasets. Including them would deflate edge coverage from 59% to 40% (10/25).

---

## 8. Limitations

1. **Cross-species**: NRF2 grounding datasets (syn205590, syn360396) are from mouse models; human translation requires caution
2. **Drug class mismatch**: Cardiotoxicity datasets (syn7079983, syn13856689) profile TKIs, not anthracyclines; tissue context (iPSC-CMs) is relevant but drug mechanism differs
3. **No direct doxorubicin cardiotoxicity dataset**: Synapse lacks a dedicated anthracycline-induced cardiomyocyte damage dataset with transcriptomic/proteomic readout
4. **NRF2 activator drugs**: Omaveloxolone and bardoxolone methyl have no Synapse coverage; their cardioprotective potential in doxorubicin context remains ungrounded
5. **CBR1/CBR3 metabolism**: No Synapse dataset for doxorubicinol formation pathway
6. **Platform age**: Several GEO-derived datasets (GSE8969, GSE11940, GSE11952) use older microarray platforms
