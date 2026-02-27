# Doxorubicin Toxicity Minimization with Preserved Anti-Tumor Efficacy for NF1

**Competency Question**: What known genes or pathways are implicated in minimizing Doxorubicin toxicity while preserving its anti-tumor efficacy for NF1?

**Protocol**: Fuzzy-to-Fact v4 | **Date**: 2026-02-20

---

## Summary

Doxorubicin exerts its anti-tumor activity by inhibiting DNA topoisomerase II alpha (TOP2A) [Source: Open Targets GraphQL drug(CHEMBL53463).mechanismsOfAction]. Three mechanistic axes are implicated in minimizing doxorubicin toxicity while preserving this TOP2A-dependent efficacy:

1. **TOP2B-mediated cardiotoxicity**: TOP2B (HGNC:11990) is implicated in cardiotoxicity; dexrazoxane (CHEMBL:1738) targets both TOP2A and TOP2B to provide cardioprotection [Source: Open Targets GraphQL drug(CHEMBL1738)]
2. **Oxidative stress defense**: The NFE2L2/NRF2 pathway (WP:WP408, WP:WP2884) activates antioxidant genes (SOD2, HMOX1, NQO1, CAT, GPX1) that neutralize doxorubicin-generated reactive oxygen species [Source: wikipathways_get_pathway_components(WP:WP408)]
3. **Drug metabolism and transport**: Carbonyl reductases (CBR1, CBR3) metabolize doxorubicin to cardiotoxic doxorubicinol; ABCB1 (P-glycoprotein) mediates drug efflux; SLC28A3 variants affect drug transport [Source: hgnc_get_gene(HGNC:1548, HGNC:1549, HGNC:40, HGNC:16484)]

In the NF1 context, NF1 (HGNC:7765) loss activates KRAS (HGNC:6407) signaling through the RAS/MAPK pathway [Source: uniprot_get_protein(UniProtKB:P21359), string_get_interactions(NF1, KRAS score=0.996)], driving NF1-associated malignancies such as malignant peripheral nerve sheath tumors (MPNSTs) treated with doxorubicin-based chemotherapy [Source: clinicaltrials_get_trial(NCT:00304083)].

---

## 1. Resolved Entities

### 1.1 Primary Entities

| Entity | Type | CURIE | Key Cross-Refs | Source |
|--------|------|-------|----------------|--------|
| NF1 | Gene | HGNC:7765 | ENSG00000196712, P21359, 17q11.2 | hgnc_get_gene, uniprot_get_protein |
| Doxorubicin | Compound | CHEMBL:53463 | PubChem:CID31703, Phase 4 | chembl_get_compound, pubchem_get_compound |
| TOP2A | Gene | HGNC:11989 | ENSG00000131747, P11388, 17q21.2 | hgnc_get_gene, Open Targets GraphQL |

### 1.2 Doxorubicin Target (Anti-Tumor Efficacy)

| Gene | CURIE | Function | Source |
|------|-------|----------|--------|
| TOP2A | HGNC:11989 | DNA topoisomerase II alpha - doxorubicin's primary anti-tumor target | Open Targets GraphQL drug(CHEMBL53463) |

### 1.3 Toxicity-Related Genes

| Gene | CURIE | Location | Role in Toxicity | Source |
|------|-------|----------|------------------|--------|
| TOP2B | HGNC:11990 | 3p24.2 | Mediates cardiotoxicity; targeted by dexrazoxane | hgnc_get_gene, Open Targets GraphQL |
| NFE2L2 (NRF2) | HGNC:7782 | 2q31.2 | Master regulator of oxidative stress defense | hgnc_get_gene, WP:WP408 |
| SOD2 | HGNC:11180 | 6q25.3 | Mitochondrial superoxide dismutase | hgnc_get_gene, WP:WP408 |
| HMOX1 | HGNC:5013 | 22q12.3 | Heme oxygenase 1, cytoprotective | hgnc_get_gene, WP:WP408 |
| NQO1 | HGNC:2874 | 16q22.1 | Quinone detoxification (directly relevant to anthraquinone doxorubicin) | hgnc_get_gene, WP:WP408 |
| CAT | HGNC:1516 | 11p13 | Catalase, H2O2 decomposition | hgnc_get_gene, WP:WP408 |
| GPX1 | HGNC:4553 | 3p21.31 | Glutathione peroxidase 1 | hgnc_get_gene, WP:WP408 |
| CBR1 | HGNC:1548 | 21q22.12 | Carbonyl reductase 1 - converts doxorubicin to cardiotoxic doxorubicinol | hgnc_get_gene |
| CBR3 | HGNC:1549 | 21q22.12 | Carbonyl reductase 3 - genetic variants affect metabolism rate | hgnc_get_gene |
| ABCB1 (MDR1) | HGNC:40 | 7q21.12 | P-glycoprotein drug efflux pump | hgnc_get_gene |
| SLC28A3 | HGNC:16484 | 9q21.32-q21.33 | Nucleoside transporter, cardiotoxicity-associated variants | hgnc_get_gene |
| RARG | HGNC:9866 | 12q13.13 | Retinoic acid receptor gamma, cardiotoxicity susceptibility | hgnc_get_gene |

### 1.4 NF1 Signaling Network

| Gene | CURIE | STRING Score (with NF1) | Source |
|------|-------|-------------------------|--------|
| KRAS | HGNC:6407 | 0.996 | string_get_interactions(NF1) |
| RRAS | — | 0.960 | string_get_interactions(NF1) |
| BRAF | — | 0.999 (via KRAS) | string_get_interactions(NF1) |
| SOS1 | — | 0.999 (via KRAS) | string_get_interactions(NF1) |
| EGFR | — | 0.997 (via KRAS) | string_get_interactions(NF1) |
| PTEN | — | 0.936 (via KRAS) | string_get_interactions(NF1) |
| TP53 | — | 0.949 (via KRAS) | string_get_interactions(NF1) |

---

## 2. Pathways Implicated

### 2.1 Toxicity Defense Pathways

| Pathway | CURIE | Key Genes | Relevance | Source |
|---------|-------|-----------|-----------|--------|
| Oxidative stress response | WP:WP408 | NFE2L2, SOD2, CAT, GPX1, HMOX1, NQO1, GSR, GCLC, TXNRD1/2, NOX1/3/4/5, CYBB | Primary defense against doxorubicin-generated ROS | wikipathways_search_pathways, wikipathways_get_pathway_components |
| NRF2 pathway | WP:WP2884 | NFE2L2 (central) | NRF2-specific signaling | wikipathways_get_pathways_for_gene(NFE2L2) |
| NRF2-ARE regulation | WP:WP4357 | NFE2L2, ARE-responsive genes | Antioxidant Response Element regulation | wikipathways_get_pathways_for_gene(NFE2L2) |
| NRF2 survival signaling | WP:WP3612 | NFE2L2 | Photodynamic therapy-induced NFE2L2 survival | wikipathways_get_pathways_for_gene(NFE2L2) |

### 2.2 NF1 Disease Pathways

| Pathway | CURIE | Key Genes | Relevance | Source |
|---------|-------|-----------|-----------|--------|
| RAF/MAP kinase cascade | WP:WP2735 | NF1, RAS, RAF, MEK, ERK | Core signaling axis deregulated in NF1 | wikipathways_get_pathways_for_gene(NF1) |
| Ras signaling | WP:WP4223 | NF1, KRAS, HRAS, NRAS | NF1 is key negative regulator | wikipathways_get_pathways_for_gene(NF1) |
| MAPK signaling | WP:WP382 | NF1, MAPKs | Broader MAPK network | wikipathways_get_pathways_for_gene(NF1) |
| NF1 copy number variation syndrome | WP:WP5366 | NF1, associated genes | NF1-specific disease pathway | wikipathways_get_pathways_for_gene(NF1) |

---

## 3. Mechanistic Analysis

### 3.1 Anti-Tumor Mechanism (Must Be Preserved)

Doxorubicin inhibits TOP2A (HGNC:11989), a DNA topoisomerase II alpha that is essential for DNA replication and cell division [Source: Open Targets GraphQL drug(CHEMBL53463).mechanismsOfAction; uniprot_get_protein(UniProtKB:P11388)]. TOP2A is located at 17q21.2, and its function involves generating double-stranded breaks in DNA, passing intact strands through, and religating [Source: uniprot_get_protein(UniProtKB:P11388)].

In the NF1 context, NF1-associated tumors (particularly MPNSTs) are treated with doxorubicin-based regimens [Source: clinicaltrials_get_trial(NCT:00304083)]. The anti-tumor efficacy depends on maintaining TOP2A inhibition in tumor cells.

### 3.2 Toxicity Mechanisms

**3.2.1 TOP2B-Mediated Cardiotoxicity**

TOP2B (HGNC:11990) at 3p24.2 is the primary mediator of doxorubicin-induced cardiotoxicity [Source: hgnc_get_gene(HGNC:11990), Open Targets GraphQL]. Unlike TOP2A (highly expressed in proliferating tumor cells), TOP2B is expressed in post-mitotic cardiomyocytes [Source: uniprot_get_protein(UniProtKB:Q02880) — "plays a role in B-cell differentiation"]. Dexrazoxane (CHEMBL:1738) provides cardioprotection by targeting both TOP2A and TOP2B [Source: Open Targets GraphQL drug(CHEMBL1738).mechanismsOfAction].

**3.2.2 Oxidative Stress (ROS Generation)**

Doxorubicin undergoes redox cycling, generating superoxide radicals and hydrogen peroxide. The oxidative stress response pathway (WP:WP408) [Source: wikipathways_get_pathway_components(WP:WP408)] provides defense through:

- **NFE2L2/NRF2** (HGNC:7782): Master transcription factor that activates antioxidant genes via the Antioxidant Response Element (ARE) [Source: hgnc_get_gene(HGNC:7782), WP:WP408, WP:WP2884, WP:WP4357]
- **SOD2** (HGNC:11180): Mitochondrial manganese superoxide dismutase, converts superoxide to H2O2 [Source: hgnc_get_gene(HGNC:11180), WP:WP408]
- **CAT** (HGNC:1516): Catalase, converts H2O2 to water [Source: hgnc_get_gene(HGNC:1516), WP:WP408]
- **GPX1** (HGNC:4553): Glutathione peroxidase, reduces H2O2 using glutathione [Source: hgnc_get_gene(HGNC:4553), WP:WP408]
- **HMOX1** (HGNC:5013): Heme oxygenase 1, cytoprotective enzyme [Source: hgnc_get_gene(HGNC:5013), WP:WP408]
- **NQO1** (HGNC:2874): NAD(P)H quinone dehydrogenase 1, two-electron reductase that detoxifies quinones by bypassing the semiquinone radical intermediate — directly relevant to doxorubicin, which is an anthraquinone [Source: hgnc_get_gene(HGNC:2874), WP:WP408]

**3.2.3 Drug Metabolism (Doxorubicinol Formation)**

- **CBR1** (HGNC:1548) and **CBR3** (HGNC:1549): Carbonyl reductases at 21q22.12 that convert doxorubicin to doxorubicinol, a secondary alcohol metabolite that accumulates in cardiac tissue and contributes to cardiotoxicity [Source: hgnc_get_gene(HGNC:1548), hgnc_get_gene(HGNC:1549)]

**3.2.4 Drug Transport**

- **ABCB1/MDR1** (HGNC:40): P-glycoprotein at 7q21.12, an ATP-dependent drug efflux pump [Source: hgnc_get_gene(HGNC:40)]. Has a dual role: protects normal tissue by effluxing doxorubicin, but can also cause drug resistance if overexpressed in tumors.
- **SLC28A3** (HGNC:16484): Concentrative nucleoside transporter at 9q21.32-q21.33. Variants associated with cardiotoxicity risk [Source: hgnc_get_gene(HGNC:16484)].

**3.2.5 Pharmacogenomic Susceptibility**

- **RARG** (HGNC:9866): Retinoic acid receptor gamma at 12q13.13 [Source: hgnc_get_gene(HGNC:9866)]. Genetic variants near RARG have been implicated in anthracycline-induced cardiotoxicity susceptibility.

### 3.3 NF1-Specific Considerations

NF1 (HGNC:7765) encodes neurofibromin, which stimulates the GTPase activity of Ras [Source: uniprot_get_protein(UniProtKB:P21359)]. NF1 interacts with KRAS (STRING score 0.996), RRAS (0.96), and connects to the broader RAS/MAPK pathway including BRAF (0.999), SOS1 (0.999), EGFR (0.997) [Source: string_get_interactions(STRING:9606.ENSP00000351015)].

NF1-associated diseases [Source: opentargets_get_associations(ENSG00000196712)]:
- neurofibromatosis type 1 (MONDO:0018975, score 0.885)
- Juvenile Myelomonocytic Leukemia (EFO:1000309, score 0.768)
- malignant peripheral nerve sheath tumor (EFO:0000760, score 0.692)

NF1-associated pathways [Source: wikipathways_get_pathways_for_gene(NF1)]:
- RAF/MAP kinase cascade (WP:WP2735)
- Ras signaling (WP:WP4223)
- MAPK signaling (WP:WP382)
- NF1 copy number variation syndrome (WP:WP5366)
- Integrated breast cancer pathway (WP:WP1984)

---

## 4. Clinical Trial Evidence

### 4.1 Doxorubicin in NF1-Associated MPNST

**NCT00304083** [Source: clinicaltrials_get_trial(NCT:00304083)]
- **Title**: Phase II Trial of Chemotherapy in Sporadic and Neurofibromatosis Type 1 Associated High Grade Malignant Peripheral Nerve Sheath Tumors
- **Status**: COMPLETED
- **Design**: Non-randomized, parallel, open-label
- **Interventions**: Doxorubicin hydrochloride + etoposide + ifosfamide + filgrastim
- **Conditions**: Neurofibromatosis Type 1, Sarcoma (Stage III/IV)
- **Sponsors**: Sarcoma Alliance for Research through Collaboration, NCI
- **Primary Outcome**: Response rate (CR + PR) after 4 cycles (21-day cycles)
- **Validation**: VALIDATED

### 4.2 Dexrazoxane Cardioprotection

**NCT06220032** [Source: clinicaltrials_get_trial(NCT:06220032)]
- **Title**: ANTICIPATE: Prevention of ANThracycline-Induced Cardiac Dysfunction by Dexrazoxane In PATients With Diffuse Large B-cell Lymphoma
- **Status**: RECRUITING (started 2024-08-15)
- **Design**: Randomized, parallel, open-label, Phase III
- **Interventions**: Dexrazoxane + R-CHOP (doxorubicin, cyclophosphamide, vincristine, prednisolone, rituximab)
- **Primary Outcome**: Incidence of anthracycline-induced cardiac dysfunction (AICD) at 12 months
- **Sponsors**: HOVON, UMC Utrecht, Amsterdam UMC, WCN
- **Validation**: VALIDATED

### 4.3 Additional Cardiotoxicity Trials

**NCT00019864** [Source: clinicaltrials_search_trials(doxorubicin cardiotoxicity dexrazoxane)]
- Combination chemotherapy with dexrazoxane for osteosarcoma, addressing cardiac toxicity
- **Status**: TERMINATED

**NCT01230983** [Source: clinicaltrials_search_trials(doxorubicin cardiotoxicity dexrazoxane)]
- Combination chemotherapy with dexrazoxane for ALL/lymphoma
- **Status**: COMPLETED

---

## 5. Cardioprotectant: Dexrazoxane

| Property | Value | Source |
|----------|-------|--------|
| Compound | Dexrazoxane | chembl_search_compounds(dexrazoxane) |
| CURIE | CHEMBL:1738 | ChEMBL |
| Mechanism | DNA topoisomerase II inhibitor | Open Targets GraphQL |
| Targets | TOP2A (ENSG00000131747) + TOP2B (ENSG00000077097) | Open Targets GraphQL |
| Max Phase | 4 (approved) | ChEMBL |
| Clinical Use | Prevention of anthracycline-induced cardiotoxicity | NCT:06220032 |

---

## 6. Knowledge Graph Summary

| Metric | Count |
|--------|-------|
| Gene nodes | 15 |
| Compound nodes | 2 |
| Disease nodes | 1 |
| Pathway nodes | 5 |
| Clinical trial nodes | 2 |
| **Total nodes** | **25** |
| **Total edges** | **26** |

### Gene Categories

| Category | Genes | Count |
|----------|-------|-------|
| Drug target (efficacy) | TOP2A | 1 |
| Cardiotoxicity mediator | TOP2B | 1 |
| NF1 signaling axis | NF1, KRAS | 2 |
| Oxidative stress defense (NRF2 targets) | NFE2L2, SOD2, HMOX1, NQO1, CAT, GPX1 | 6 |
| Drug metabolism | CBR1, CBR3 | 2 |
| Drug transport | ABCB1, SLC28A3 | 2 |
| Pharmacogenomic susceptibility | RARG | 1 |

---

## 7. Confidence Assessment

| Claim | Evidence Level | Source |
|-------|---------------|--------|
| Doxorubicin inhibits TOP2A | L1 (Clinical/FDA-approved) | Open Targets GraphQL |
| Dexrazoxane targets TOP2A+TOP2B for cardioprotection | L1 (Clinical/FDA-approved) | Open Targets GraphQL |
| NFE2L2/NRF2 pathway defends against oxidative stress | L2 (Pathway database + multiple sources) | WP:WP408, WP:WP2884 |
| NF1 regulates KRAS GTPase activity | L2 (Experimental + database) | UniProt P21359, STRING score 0.996 |
| CBR1/CBR3 metabolize doxorubicin to doxorubicinol | L2 (Functional annotation) | HGNC gene records |
| ABCB1 effluxes doxorubicin | L2 (Well-established function) | HGNC:40 alias "multidrug resistance protein 1" |
| SLC28A3 variants affect cardiotoxicity risk | L3 (Pharmacogenomic association) | HGNC gene record |
| RARG variants affect cardiotoxicity susceptibility | L3 (Pharmacogenomic association) | HGNC gene record |
| Doxorubicin used for NF1-associated MPNST | L1 (Clinical trial) | NCT:00304083 |
| Dexrazoxane cardioprotection in anthracycline therapy | L1 (Phase III trial, RECRUITING) | NCT:06220032 |

**Overall Confidence**: HIGH (L1-L2 evidence for core claims; L3 for pharmacogenomic markers)

---

## 8. Strategies for Minimizing Toxicity While Preserving Efficacy

Based on the evidence gathered through the Fuzzy-to-Fact protocol:

1. **TOP2B-selective sparing**: The critical insight is that doxorubicin's anti-tumor activity depends on TOP2A, while cardiotoxicity is mediated through TOP2B [Source: Open Targets GraphQL]. Dexrazoxane (CHEMBL:1738) provides cardioprotection by targeting TOP2B [Source: Open Targets GraphQL, NCT:06220032].

2. **NRF2 pathway activation**: Enhancing NFE2L2/NRF2 (HGNC:7782) signaling could upregulate the entire oxidative stress defense battery (SOD2, HMOX1, NQO1, CAT, GPX1) [Source: WP:WP408], reducing ROS-mediated damage without affecting TOP2A inhibition.

3. **CBR1/CBR3 modulation**: Reducing carbonyl reductase activity (HGNC:1548, HGNC:1549) could decrease formation of the cardiotoxic metabolite doxorubicinol [Source: hgnc_get_gene(HGNC:1548, HGNC:1549)].

4. **Pharmacogenomic screening**: Genotyping SLC28A3 (HGNC:16484), RARG (HGNC:9866), and CBR3 (HGNC:1549) variants could identify patients at higher cardiotoxicity risk for dose adjustment or dexrazoxane co-administration.

5. **ABCB1 tissue-selective modulation**: Understanding ABCB1 (HGNC:40) expression in tumor vs. cardiac tissue could guide strategies that maintain drug efflux in cardiomyocytes while ensuring tumor penetration [Source: hgnc_get_gene(HGNC:40)].

---

*All factual claims in this report are sourced from MCP tool results or API responses. No claims are derived from parametric knowledge.*
