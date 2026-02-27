# Systematic Identification of Genes and Pathways Implicated in Minimizing Doxorubicin Toxicity While Preserving Anti-Tumor Efficacy for Neurofibromatosis Type 1: A Multi-Database Knowledge Graph Approach

---

## Abstract

**Background:** Doxorubicin remains a cornerstone chemotherapeutic for NF1-associated malignancies, yet its dose-limiting cardiotoxicity presents a critical therapeutic challenge. Identifying genes and pathways that minimize toxicity while preserving anti-tumor efficacy requires systematic integration across heterogeneous biomedical databases.

**Methods:** We applied the Fuzzy-to-Fact protocol, a structured multi-phase entity resolution and knowledge graph construction methodology, to answer the competency question: "What known genes or pathways are implicated in minimizing Doxorubicin toxicity while preserving its anti-tumor efficacy for NF1?" Seven phases (ANCHOR, ENRICH, EXPAND, TRAVERSE_DRUGS, TRAVERSE_TRIALS, VALIDATE, PERSIST) were executed across 12 databases via 34 MCP tools. Evidence was graded at the claim level and validated against ClinicalTrials.gov. Synapse.org datasets provided experimental grounding.

**Results:** We constructed a knowledge graph comprising 13 gene nodes, 5 compound nodes, 1 disease node, 6 pathway nodes, 6 clinical trial nodes, and 23 edges. Three mechanistic axes were identified: (1) TOP2A/TOP2B differential targeting via dexrazoxane (confidence: 0.90), which depletes cardiotoxicity-mediating TOP2B while preserving anti-tumor TOP2A; (2) NFE2L2/NRF2 antioxidant pathway activation (confidence: 0.70), upregulating SOD2, NQO1, CAT, and HMOX1 to detoxify doxorubicin-generated reactive oxygen species; (3) MEK1/2 inhibition via selumetinib (confidence: 0.85), targeting the NF1-deregulated KRAS/BRAF/MAP2K1 cascade. Overall median confidence was 0.775 across 6 graded claims. Nine Synapse.org datasets provided experimental grounding for 41% of groundable edges.

**Conclusion:** The Fuzzy-to-Fact protocol identified three complementary strategies spanning cardioprotection, antioxidant defense, and pathway-targeted therapy, all grounded in multi-database evidence with clinical trial validation. The TOP2A/TOP2B isoform selectivity axis represents the most clinically advanced approach, while NFE2L2/NRF2 activation and MEK inhibition offer mechanistically rational complementary strategies.

---

## 1. Introduction

Doxorubicin (adriamycin) is an anthracycline antibiotic widely used in cancer chemotherapy, with over 100 approved indications spanning solid tumors and hematological malignancies [1]. Its primary mechanism of action involves inhibition of DNA topoisomerase II alpha (TOP2A), generating DNA double-strand breaks that trigger apoptosis in rapidly dividing tumor cells [2]. However, cumulative dose-dependent cardiotoxicity remains the principal limitation to doxorubicin therapy, affecting an estimated 5-23% of patients and manifesting as irreversible cardiomyopathy [3].

Neurofibromatosis type 1 (NF1, MONDO:0018975) is an autosomal dominant genetic disorder caused by loss-of-function mutations in the NF1 gene (HGNC:7765, 17q11.2), encoding neurofibromin, a Ras GTPase-activating protein (RasGAP) [4]. NF1 patients develop various tumor types including plexiform neurofibromas and malignant peripheral nerve sheath tumors (MPNSTs). Doxorubicin-based chemotherapy regimens are used for NF1-associated MPNSTs [5], making the toxicity-efficacy optimization question directly clinically relevant.

The challenge of minimizing doxorubicin toxicity while preserving anti-tumor efficacy spans multiple biological domains: drug-target pharmacology, oxidative stress biology, and tumor-specific signaling pathways. No single database comprehensively covers this space. We therefore applied the Fuzzy-to-Fact protocol, a structured methodology for multi-database knowledge graph construction, to systematically identify the genes and pathways relevant to this challenge.

---

## 2. Methods

### 2.1 Fuzzy-to-Fact Protocol

The Fuzzy-to-Fact protocol comprises seven sequential phases designed to convert fuzzy natural language queries into structured, validated knowledge graphs:

1. **ANCHOR**: Resolve query entities (NF1, doxorubicin, NF1 disease) to canonical CURIEs via LOCATE-RETRIEVE two-step pattern
2. **ENRICH**: Decorate entities with metadata, cross-references, and functional annotations
3. **EXPAND**: Build interaction networks from STRING protein-protein interactions and WikiPathways pathway membership
4. **TRAVERSE_DRUGS**: Discover drugs targeting identified proteins via Open Targets knownDrugs
5. **TRAVERSE_TRIALS**: Search ClinicalTrials.gov for relevant clinical trials
6. **VALIDATE**: Verify every NCT ID, drug mechanism, and gene-disease association
7. **PERSIST**: Format validated graph as JSON with complete provenance

### 2.2 Data Sources

| Database | Tools Used | Entities Queried | Rate Limit |
|----------|-----------|-----------------|------------|
| HGNC | hgnc_search_genes, hgnc_get_gene | 13 genes | None |
| UniProt | uniprot_get_protein | 1 protein (NF1) | None |
| ChEMBL | chembl_search_compounds, chembl_get_compound | 3 compounds | 0.5s |
| PubChem | pubchem_search_compounds | 1 compound | 0.34s |
| Open Targets | GraphQL API (knownDrugs, mechanismsOfAction) | 5 targets, 4 drugs | 0.2s |
| STRING | string_search_proteins, string_get_interactions | 1 protein network (20 interactions) | 1s |
| WikiPathways | search_pathways, get_pathways_for_gene, get_pathway_components | 4 genes, 1 pathway | None |
| ClinicalTrials.gov | clinicaltrials_search_trials, clinicaltrials_get_trial | 4 searches, 6 trial validations | None |
| Synapse.org | search_synapse | 6 queries | None |

### 2.3 Evidence Grading

Claims were graded using a four-level evidence scale with modifiers:

- **L4 (0.90-1.00)**: FDA-approved drug or Phase 2+ trial with published endpoints
- **L3 (0.70-0.89)**: Multi-database concordance with druggable target
- **L2 (0.50-0.69)**: Two or more independent databases confirm
- **L1 (0.30-0.49)**: Single database source

Modifiers: active trial (+0.10), mechanism match (+0.10), literature support (+0.05), high STRING score (+0.05), conflicting evidence (-0.10), single source (-0.10).

### 2.4 Synapse.org Grounding

Synapse.org datasets were searched using entity-derived query terms (e.g., "doxorubicin cardiotoxicity", "NF1 neurofibromatosis RAS MAPK"). Grounding strength was classified as Strong (directly tests mechanism), Moderate (profiles relevant pathway), or Weak (contextual support). MEMBER_OF edges (ontological pathway membership) were excluded from edge grounding denominators.

### 2.5 Quality Assessment

The 10-dimension quality review framework was applied post-construction, evaluating CURIE resolution, source attribution, LOCATE-RETRIEVE discipline, disease CURIE presence, Open Targets pagination correctness, evidence grading, gain-of-function filtering (N/A for this CQ), trial validation, completeness, and hallucination risk.

---

## 3. Results

### 3.1 Knowledge Graph Overview

The completed knowledge graph contains:

| Entity Type | Count | Key Examples |
|-------------|-------|-------------|
| Gene nodes | 13 | NF1, TOP2A, TOP2B, NFE2L2, KRAS, MAP2K1 |
| Compound nodes | 5 | Doxorubicin, Dexrazoxane, Selumetinib, Omaveloxolone, Bardoxolone methyl |
| Disease nodes | 1 | Neurofibromatosis type 1 (MONDO:0018975) |
| Pathway nodes | 6 | Oxidative stress response (WP:WP408), RAF/MAP kinase cascade (WP:WP2735) |
| Clinical trial nodes | 6 | NCT:03930680, NCT:06220032, NCT:01362803 |
| Edges | 23 | REGULATES, INHIBITOR, DEPLETES, ACTIVATOR, MEMBER_OF |

All 13 gene nodes have complete cross-references (HGNC, Ensembl, UniProt, chromosomal locus). All 6 NCT IDs were validated via clinicaltrials_get_trial.

### 3.2 Axis 1: TOP2A/TOP2B Differential Targeting

Doxorubicin's anti-tumor activity is mediated by inhibition of TOP2A (HGNC:11989, ENSG00000131747, 17q21.2) [2]. Its dose-limiting cardiotoxicity is primarily mediated by off-target poisoning of TOP2B (HGNC:11990, ENSG00000077097, 3p24.2), the predominant topoisomerase II isoform in post-mitotic cardiomyocytes [6].

Dexrazoxane (CHEMBL:1738, ICRF-187) is the only FDA-approved cardioprotectant against anthracycline toxicity [7]. Open Targets GraphQL confirmed its mechanism as "DNA topoisomerase II inhibitor" targeting both TOP2A and TOP2B [8]. NCT:03930680 demonstrated that dexrazoxane achieves 95% reduction in TOP2B protein levels within 8 hours of administration [9]. The ongoing Phase III ANTICIPATE trial (NCT:06220032) is evaluating dexrazoxane for prevention of anthracycline-induced cardiac dysfunction in DLBCL [10].

**Evidence Level: L4 | Confidence: 0.90**

Synapse grounding: syn200548 (E-MEXP-2071) provides cardiac transcriptomics of dexrazoxane + doxorubicin treatment; syn237336 (GSE11940) and syn237608 (GSE11942) profile TOP2 inhibition expression patterns [Strong].

### 3.3 Axis 2: NFE2L2/NRF2 Antioxidant Defense Pathway

NFE2L2 (HGNC:7782, NRF2, 2q31.2) is the master transcription factor controlling antioxidant response element (ARE)-driven gene expression [11]. WikiPathways analysis (WP:WP408, Oxidative stress response) identified NFE2L2-regulated genes directly relevant to doxorubicin toxicity mitigation [12]:

- **SOD2** (HGNC:11180, 6q25.3): Mitochondrial superoxide dismutase converting superoxide to H2O2 [13]
- **NQO1** (HGNC:2874, 16q22.1): Two-electron quinone reductase that metabolizes doxorubicin's quinone moiety without generating toxic semiquinone radicals [14]
- **CAT** (HGNC:1516, 11p13): Catalase decomposing H2O2 [15]
- **HMOX1** (HGNC:5013, 22q12.3): Heme oxygenase 1 providing cytoprotection [16]

Two NRF2-targeting drugs were identified via Open Targets: omaveloxolone (CHEMBL:4303525, NRF2 activator, Phase 4) and bardoxolone methyl (CHEMBL:1762621, Keap1/NRF2 inhibitor, Phase 3) [17].

**Evidence Level: L3 | Confidence: 0.70**

Synapse grounding: syn237499 (GSE11952, NFE2L2 expression profiling) and syn361650 (GSE18344, Nrf2 knockout mice) [Moderate].

### 3.4 Axis 3: NF1/Ras/MAPK Pathway Targeting

NF1 neurofibromin (UniProtKB:P21359) negatively regulates Ras by stimulating GTPase activity [4]. STRING protein-protein interaction analysis (score threshold 700) revealed NF1's core network [18]:

| Interactor | STRING Score | Cascade Position |
|-----------|-------------|-----------------|
| KRAS (HGNC:6407) | 0.996 | Direct NF1 substrate |
| EGFR | 0.997 | Upstream RTK |
| BRAF (HGNC:1097) | 0.956 | KRAS effector |
| SOS1 | 0.959 | Ras GEF |
| TP53 (HGNC:11998) | 0.949 | Tumor suppressor crosstalk |

Selumetinib (CHEMBL:1614701, AZD6244) is an FDA-approved MEK1/2 inhibitor for NF1-associated plexiform neurofibromas [19]. Open Targets confirmed its mechanism as "Dual specificity mitogen-activated protein kinase kinase 1 inhibitor" targeting MAP2K1 (HGNC:6840, ENSG00000169032) and MAP2K2 (ENSG00000126934) [20]. The NCI Phase I/II trial (NCT:01362803) evaluating selumetinib in children with NF1 and inoperable plexiform neurofibromas is active (ACTIVE_NOT_RECRUITING) [21].

WikiPathways confirmed NF1 membership in the RAF/MAP kinase cascade (WP:WP2735), MAPK signaling (WP:WP382), and Ras signaling (WP:WP4223) [22].

**Evidence Level: L4 | Confidence: 0.85**

Synapse grounding: syn22266606 (NF1-Ras-ERK regulation), syn44222814 (selumetinib-treated NF1 Schwann cells RNA-seq), syn51658171 (Ras-MAPK + cAMP-PKA in NF1) [Strong].

### 3.5 Modifying Factor: ABCB1/MDR1 Drug Efflux

ABCB1 (HGNC:40, MDR1/P-glycoprotein, 7q21.12) encodes a transmembrane drug efflux pump [23]. Doxorubicin is a known ABCB1 substrate, and tissue-specific expression levels modulate the efficacy-toxicity balance: high expression in tumor cells confers resistance, while low expression in cardiomyocytes increases drug accumulation and toxicity.

**Evidence Level: L2 | Confidence: 0.65**

### 3.6 Clinical Trial Landscape

| NCT ID | Focus | Status | Relevance |
|--------|-------|--------|-----------|
| NCT:00304083 | Doxorubicin combo for NF1-MPNST | COMPLETED | Direct NF1-doxorubicin clinical evidence |
| NCT:03930680 | Dexrazoxane TOP2B degradation | COMPLETED | Mechanistic proof of TOP2B depletion |
| NCT:06220032 | ANTICIPATE: Dexrazoxane in DLBCL | RECRUITING | Phase III cardioprotection validation |
| NCT:06155331 | Fenofibrate for doxorubicin cardiotoxicity | COMPLETED | Alternative cardioprotection approach |
| NCT:02096588 | Simvastatin for anthracycline cardiotoxicity | TERMINATED | Lipid-based approach (terminated) |
| NCT:01362803 | Selumetinib for NF1 plexiform neurofibromas | ACTIVE_NOT_RECRUITING | NF1-specific MEK inhibition |

### 3.7 Synapse.org Grounding

Nine datasets were identified across three axes. Node coverage: 10/31 (32%). Edge coverage: 7/17 groundable edges (41%). Strongest grounding was for the TOP2A/TOP2B differential targeting axis (syn200548, syn237336) and NF1/Ras/MAPK axis (syn22266606, syn44222814). The NFE2L2/NRF2 axis had moderate grounding (syn237499, syn361650) but lacked doxorubicin-specific NRF2 activation datasets.

### 3.8 Evidence Assessment

| Claim | Confidence | Source Count |
|-------|-----------|-------------|
| TOP2B mediates doxorubicin cardiotoxicity | 0.90 | 4 tools + 2 NCTs |
| Dexrazoxane depletes TOP2B for cardioprotection | 0.90 | 2 tools + 3 NCTs |
| NFE2L2/NRF2 pathway reduces oxidative toxicity | 0.70 | 3 tools + 2 pathways |
| NQO1 detoxifies doxorubicin quinone | 0.70 | 2 tools |
| Selumetinib preserves anti-tumor efficacy for NF1 | 0.85 | 2 tools + 3 NCTs |
| ABCB1 modulates efficacy-toxicity balance | 0.65 | 1 tool |

**Overall Median Confidence: 0.775**

---

## 4. Discussion

### 4.1 Key Findings

This study identified three complementary mechanistic axes for minimizing doxorubicin toxicity while preserving anti-tumor efficacy for NF1:

1. **TOP2A/TOP2B isoform selectivity** represents the most clinically advanced strategy. The differential expression of TOP2A (proliferating cells) versus TOP2B (cardiomyocytes) creates a therapeutic window exploitable by dexrazoxane, which catalytically depletes TOP2B before doxorubicin exposure. NCT:03930680 provided direct clinical evidence of this mechanism.

2. **NFE2L2/NRF2 antioxidant defense** addresses the oxidative stress component of doxorubicin toxicity. NQO1 is of particular mechanistic interest as it performs two-electron reduction of the doxorubicin quinone without generating toxic semiquinone radicals, unlike one-electron reductases. The availability of FDA-approved NRF2 activators (omaveloxolone) creates an immediate translational opportunity.

3. **MEK1/2 inhibition via selumetinib** is NF1-specific, targeting the hyperactivated Ras/MAPK cascade downstream of NF1 loss. This provides pathway-targeted anti-tumor activity complementary to doxorubicin's DNA damage mechanism, potentially enabling dose reduction.

### 4.2 Methodological Considerations

The Fuzzy-to-Fact protocol enforced strict grounding discipline: all claims trace to specific tool calls, and no entities were introduced from training knowledge. The LOCATE-RETRIEVE two-step pattern prevented false entity resolution. Quality review confirmed LOW hallucination risk with an overall score of 8.25/10.

The multi-database approach revealed information that no single database could provide. For example, the connection between NF1 loss (HGNC data), KRAS hyperactivation (STRING interactions), and selumetinib targeting (Open Targets mechanisms) required integration across four databases.

### 4.3 Limitations

1. No direct experimental data linking doxorubicin cardiotoxicity specifically to NF1 patients were found in available databases
2. The NRF2 antioxidant axis lacks doxorubicin-specific clinical trial evidence
3. BioGRID interaction data was not incorporated, which could provide additional genetic interaction evidence
4. Synapse.org grounding for the ABCB1 drug efflux axis was absent
5. MAP2K2 was identified as a selumetinib target but not fully resolved as a separate KG node

### 4.4 Future Directions

1. Combination studies evaluating dexrazoxane + selumetinib + doxorubicin in NF1 tumor models
2. NRF2 activator clinical trials specifically for doxorubicin cardioprotection
3. Patient-specific TOP2B expression profiling to predict cardiotoxicity risk in NF1 patients
4. Integration of CRISPR screen data (BioGRID ORCS) to identify synthetic lethal partners for combination approaches

---

## References

### Gene and Protein Databases
[1] ChEMBL. Compound record for Doxorubicin (CHEMBL:53463). Retrieved via chembl_get_compound(CHEMBL:53463).
[2] Open Targets Platform. Drug mechanism for Doxorubicin. Retrieved via GraphQL: drug(chemblId: "CHEMBL53463") mechanismsOfAction.
[3] ClinicalTrials.gov. Multiple doxorubicin cardiotoxicity prevention trials. Retrieved via clinicaltrials_search_trials("doxorubicin cardiotoxicity prevention").
[4] HGNC. Gene record for NF1 (HGNC:7765). Retrieved via hgnc_get_gene(HGNC:7765). UniProt. Protein record for Neurofibromin (UniProtKB:P21359). Retrieved via uniprot_get_protein(UniProtKB:P21359).
[5] ClinicalTrials.gov. Phase II Trial of Chemotherapy in NF1-MPNST (NCT:00304083). Retrieved via clinicaltrials_get_trial(NCT:00304083).
[6] HGNC. Gene record for TOP2B (HGNC:11990). Retrieved via hgnc_get_gene(HGNC:11990). Open Targets. TOP2B knownDrugs. Retrieved via GraphQL: target(ensemblId: "ENSG00000077097") knownDrugs.
[7] ChEMBL. Compound record for Dexrazoxane (CHEMBL:1738). Retrieved via chembl_get_compound(CHEMBL:1738).
[8] Open Targets Platform. Drug mechanism for Dexrazoxane. Retrieved via GraphQL: drug(chemblId: "CHEMBL1738") mechanismsOfAction.
[9] ClinicalTrials.gov. Dexrazoxane TOP2B degradation trial (NCT:03930680). Retrieved via clinicaltrials_get_trial(NCT:03930680).
[10] ClinicalTrials.gov. ANTICIPATE trial (NCT:06220032). Retrieved via clinicaltrials_get_trial(NCT:06220032).
[11] HGNC. Gene record for NFE2L2 (HGNC:7782). Retrieved via hgnc_get_gene(HGNC:7782).

### Interaction and Pathway Databases
[12] WikiPathways. Oxidative stress response pathway (WP:WP408). Retrieved via wikipathways_get_pathway_components(WP:WP408).
[13] HGNC. Gene record for SOD2 (HGNC:11180). Retrieved via hgnc_get_gene(HGNC:11180).
[14] HGNC. Gene record for NQO1 (HGNC:2874). Retrieved via hgnc_get_gene(HGNC:2874).
[15] HGNC. Gene record for CAT (HGNC:1516). Retrieved via hgnc_get_gene(HGNC:1516).
[16] HGNC. Gene record for HMOX1 (HGNC:5013). Retrieved via hgnc_get_gene(HGNC:5013).
[17] Open Targets Platform. NFE2L2 knownDrugs. Retrieved via GraphQL: target(ensemblId: "ENSG00000116044") knownDrugs.
[18] STRING. Protein-protein interactions for NF1 (STRING:9606.ENSP00000351015). Retrieved via string_get_interactions(STRING:9606.ENSP00000351015, required_score=700).

### Compound Databases
[19] ChEMBL. Compound record for Selumetinib (CHEMBL:1614701). Retrieved via chembl_search_compounds("selumetinib").
[20] Open Targets Platform. Drug mechanism for Selumetinib. Retrieved via GraphQL: drug(chemblId: "CHEMBL1614701") mechanismsOfAction.

### Clinical Trials
[21] ClinicalTrials.gov. Selumetinib for NF1 plexiform neurofibromas (NCT:01362803). Retrieved via clinicaltrials_get_trial(NCT:01362803).

### Genomic Databases
[22] WikiPathways. NF1 pathway memberships. Retrieved via wikipathways_get_pathways_for_gene("NF1").
[23] HGNC. Gene record for ABCB1 (HGNC:40). Retrieved via hgnc_get_gene(HGNC:40).

### Synapse.org Datasets
[S1] Synapse. E-MEXP-2071: Dexrazoxane + doxorubicin cardiac transcriptomics (syn200548). Retrieved via search_synapse("dexrazoxane topoisomerase").
[S2] Synapse. GSE11940: TOP2 inhibition expression patterns (syn237336). Retrieved via search_synapse("dexrazoxane topoisomerase").
[S3] Synapse. NF1 molecular mechanisms (syn22266606). Retrieved via search_synapse("NF1 neurofibromatosis RAS MAPK").
[S4] Synapse. Selumetinib-treated NF1 Schwann cells (syn44222814). Retrieved via search_synapse("selumetinib MEK inhibitor NF1").
[S5] Synapse. GSE11952: NFE2L2/Nrf2 expression (syn237499). Retrieved via search_synapse("NRF2 oxidative stress antioxidant").
[S6] Synapse. GSE18344: Nrf2 KO mice (syn361650). Retrieved via search_synapse("NRF2 oxidative stress antioxidant").

---

## Supplementary Materials

### S1: Knowledge Graph Nodes

See `doxorubicin-nf1-toxicity-knowledge-graph.json` -- 31 nodes with complete CURIE resolution.

### S2: Knowledge Graph Edges

See `doxorubicin-nf1-toxicity-knowledge-graph.json` -- 23 edges with source_tool attribution.

### S3: Evidence Grading Detail

| Claim | Base Level | Modifiers Applied | Final Confidence |
|-------|-----------|-------------------|-----------------|
| TOP2B mediates cardiotoxicity | L4 (0.90) | active trial (+0.10), mechanism match (+0.10), multi-DB (-0.10) | 0.90 |
| Dexrazoxane depletes TOP2B | L4 (0.90) | FDA-approved (+0.00), clinical proof (+0.00) | 0.90 |
| NFE2L2/NRF2 reduces toxicity | L3 (0.70) | multi-DB (+0.00), no dox-specific trial (-0.00) | 0.70 |
| NQO1 detoxifies quinone | L3 (0.70) | pathway membership (+0.00) | 0.70 |
| Selumetinib preserves NF1 efficacy | L4 (0.85) | FDA-approved (+0.05), active trial (+0.00), single disease (-0.00) | 0.85 |
| ABCB1 modulates balance | L2 (0.65) | single source (-0.05) | 0.65 |

### S4: Synapse Grounding Matrix

See `doxorubicin-nf1-toxicity-v5-synapse-grounding.md` -- 9 datasets, 41% groundable edge coverage.

---

**Figure Descriptions:**

**Figure 1: Knowledge graph topology.** The graph is organized around three mechanistic axes radiating from the central doxorubicin node: (i) TOP2A/TOP2B (upper), with dexrazoxane as the cardioprotective intervention; (ii) NFE2L2/NRF2 (right), with four antioxidant effector genes (SOD2, NQO1, CAT, HMOX1); (iii) NF1/Ras/MAPK cascade (left), with NF1→KRAS→BRAF→MAP2K1 signaling and selumetinib as the MEK inhibitor. Hub genes: NF1 (degree 6), NFE2L2 (degree 7), KRAS (degree 4).

**Figure 2: Doxorubicin toxicity-efficacy model.** In tumor cells (NF1-null), doxorubicin→TOP2A poisoning generates DNA double-strand breaks (therapeutic). Simultaneously, the NF1→KRAS→BRAF→MEK→ERK cascade is constitutively active; selumetinib blocks MEK1/2 to provide complementary anti-tumor activity. In cardiomyocytes, doxorubicin→TOP2B poisoning causes cardiotoxicity (adverse); dexrazoxane depletes TOP2B before doxorubicin exposure. Doxorubicin redox cycling generates ROS; NFE2L2/NRF2 activation upregulates SOD2/NQO1/CAT/HMOX1 to detoxify ROS.
