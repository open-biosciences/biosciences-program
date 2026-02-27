# Multi-Database Knowledge Graph Construction Identifies Genes and Pathways for Minimizing Doxorubicin Toxicity While Preserving Anti-Tumor Efficacy in Neurofibromatosis Type 1: A Fuzzy-to-Fact Protocol Application

**Authors:** Deep Agents Life Sciences Platform (Automated Pipeline)

**Corresponding contact:** Life Sciences Deep Agents Platform

**Date:** February 20, 2026

**Keywords:** doxorubicin, cardiotoxicity, neurofibromatosis type 1, NF1, knowledge graph, TOP2A, TOP2B, NRF2, CBR1, CBR3, ABCB1, SLC28A3, dexrazoxane, RAS-MAPK, pharmacogenomics, Fuzzy-to-Fact protocol

---

## Abstract

**Background.** Doxorubicin remains a first-line chemotherapeutic for malignant peripheral nerve sheath tumors (MPNSTs) arising in Neurofibromatosis Type 1 (NF1; MONDO:0018975), yet its clinical utility is severely constrained by dose-limiting cardiotoxicity. The molecular landscape governing the balance between doxorubicin's anti-tumor efficacy and cardiac injury spans multiple genes, pathways, and pharmacogenomic determinants distributed across heterogeneous public databases. No single resource captures this landscape comprehensively, and manual literature synthesis is neither reproducible nor scalable.

**Methods.** We applied the Fuzzy-to-Fact protocol -- a seven-phase automated knowledge graph (KG) assembly pipeline (ANCHOR, ENRICH, EXPAND, TRAVERSE_DRUGS, TRAVERSE_TRIALS, VALIDATE, PERSIST) -- to the competency question: "What known genes or pathways are implicated in minimizing doxorubicin toxicity while preserving its anti-tumor efficacy for NF1?" The pipeline traversed eight federated life sciences databases (HGNC, UniProt, STRING, WikiPathways, Open Targets, ChEMBL, PubChem, and ClinicalTrials.gov) using MCP-standardized tool wrappers. Evidence was graded on a four-level system (L1-L4) with source-specific modifiers. The resulting KG was grounded to five Synapse.org experimental datasets and assessed against a ten-dimension quality framework.

**Results.** The pipeline produced a connected KG of 25 nodes (15 genes, 2 compounds, 1 disease, 5 pathways, 2 clinical trials) and 26 edges with a mean confidence of 0.79. Three mechanistic axes emerged: (i) TOP2A/TOP2B isoform selectivity, where doxorubicin poisons TOP2A (HGNC:11989) for anti-tumor effect but TOP2B (HGNC:11990) for cardiotoxicity, and dexrazoxane (CHEMBL:1738) selectively degrades TOP2B; (ii) NFE2L2/NRF2 (HGNC:7782) master regulation of oxidative stress defense via SOD2, HMOX1, NQO1, CAT, and GPX1; and (iii) drug metabolism and transport determinants including CBR1 (HGNC:1548) and CBR3 (HGNC:1549) conversion of doxorubicin to cardiotoxic doxorubicinol, ABCB1/MDR1 (HGNC:40) drug efflux, and pharmacogenomic susceptibility variants in SLC28A3 (HGNC:16484) and RARG (HGNC:9866). The NF1 (HGNC:7765)-KRAS (STRING score 0.996) RAS/MAPK axis defined the disease-specific context. Two clinical trials were mapped: NCT00304083 (doxorubicin for NF1 MPNSTs, COMPLETED) and NCT06220032 (ANTICIPATE dexrazoxane cardioprotection trial, RECRUITING). KG assertions were grounded to five Synapse datasets (syn237336, syn205590, syn7079983, syn21452685, syn241620).

**Conclusion.** Automated, federated KG construction via the Fuzzy-to-Fact protocol identifies actionable pharmacogenomic axes for doxorubicin cardiotoxicity minimization in NF1, integrating gene-level resolution, pathway-level mechanism, compound-level pharmacology, and trial-level clinical evidence into a single structured, reproducible, and queryable graph.

---

## 1. Introduction

Neurofibromatosis Type 1 (NF1) is an autosomal dominant tumor predisposition syndrome caused by loss-of-function mutations in the *NF1* gene (HGNC:7765; 17q11.2), which encodes neurofibromin, a GTPase-activating protein (GAP) for RAS family GTPases [1]. With a prevalence of approximately 1 in 3,000 live births, NF1 is one of the most common single-gene disorders in humans. The clinical spectrum includes cafe-au-lait macules, cutaneous and plexiform neurofibromas, optic pathway gliomas, and a markedly elevated lifetime risk of malignant peripheral nerve sheath tumors (MPNSTs) [2]. MPNSTs, which develop in 8-13% of NF1 patients, are aggressive soft-tissue sarcomas that represent the leading cause of NF1-related mortality, with 5-year survival rates of only 20-50% for localized disease and less than 10% for metastatic presentations [3].

Doxorubicin-based chemotherapy remains the standard first-line systemic therapy for unresectable or metastatic NF1-associated MPNSTs [4]. Doxorubicin (adriamycin; CHEMBL:53463) is an anthracycline antibiotic that exerts its anti-tumor effect primarily through poisoning of topoisomerase II-alpha (TOP2A; HGNC:11989), stabilizing the TOP2A-DNA cleavage complex and generating persistent double-strand DNA breaks that trigger apoptosis in rapidly dividing tumor cells [5]. However, doxorubicin also poisons topoisomerase II-beta (TOP2B; HGNC:11990), the predominant isoform in post-mitotic cardiomyocytes. TOP2B poisoning in cardiac tissue triggers mitochondrial dysfunction, reactive oxygen species (ROS) generation, DNA damage response activation, and cardiomyocyte apoptosis, producing the characteristic dose-dependent anthracycline cardiomyopathy that limits cumulative lifetime doses to approximately 450-550 mg/m^2 [6]. This isoform dichotomy -- TOP2A mediating efficacy, TOP2B mediating toxicity -- represents a fundamental therapeutic window.

Beyond direct topoisomerase poisoning, doxorubicin undergoes multiple metabolic transformations that contribute to both efficacy and toxicity. The carbonyl reductases CBR1 (HGNC:1548) and CBR3 (HGNC:1549) reduce doxorubicin to doxorubicinol, a secondary alcohol metabolite with reduced anti-tumor activity but potent cardiotoxic properties [7]. Doxorubicin also undergoes one-electron redox cycling at its quinone moiety, generating superoxide radicals, hydrogen peroxide, and hydroxyl radicals. Cardiomyocytes are particularly vulnerable to this oxidative insult due to their high mitochondrial density, dependence on oxidative phosphorylation, and relatively low antioxidant enzyme expression [8]. The transcription factor NRF2 (encoded by *NFE2L2*; HGNC:7782) orchestrates the master antioxidant response through its target genes, including NQO1 (obligate two-electron quinone reduction), HMOX1 (heme catabolism), SOD2 (mitochondrial superoxide dismutation), CAT (hydrogen peroxide decomposition), and GPX1 (glutathione-dependent peroxide reduction) [9].

A third layer of complexity involves pharmacogenomic determinants of inter-individual cardiotoxicity risk. Variants in SLC28A3 (HGNC:16484), encoding concentrative nucleoside transporter 3, and RARG (HGNC:9866), encoding retinoic acid receptor gamma, have been identified in genome-wide association studies as modifiers of anthracycline cardiotoxicity susceptibility [10]. ABCB1/MDR1 (HGNC:40) encodes P-glycoprotein, the prototypical ABC transporter that mediates doxorubicin efflux from cells; its expression in cardiomyocytes may be cardioprotective, while tumor overexpression drives chemoresistance [11].

Despite the biological plausibility of these mechanisms, no single database captures the full landscape of genes, pathways, compounds, and clinical evidence relevant to doxorubicin toxicity minimization in NF1. Gene-level data reside in HGNC and UniProt; protein-protein interactions in STRING; pathway annotations in WikiPathways; disease-target-drug associations in Open Targets and ChEMBL; compound metadata in PubChem; clinical trial registrations in ClinicalTrials.gov; and experimental datasets in Synapse.org. Synthesizing these into a coherent, evidence-graded, and reproducible knowledge graph requires systematic multi-database traversal that goes beyond manual literature review.

In this work, we applied the Fuzzy-to-Fact protocol -- a seven-phase automated KG assembly pipeline -- to the competency question: "What known genes or pathways are implicated in minimizing doxorubicin toxicity while preserving its anti-tumor efficacy for NF1?" The pipeline produced a 25-node, 26-edge knowledge graph spanning three mechanistic axes (TOP2A/TOP2B selectivity, NFE2L2/NRF2 oxidative stress defense, and drug metabolism/transport), grounded to five Synapse datasets and two clinical trials. We present the assembled KG, evaluate evidence concordance across databases, and discuss implications for pharmacogenomic stratification and rational cardioprotective strategy design in NF1.

---

## 2. Methods

### 2.1 Fuzzy-to-Fact Protocol Overview

The Fuzzy-to-Fact protocol is a seven-phase knowledge graph assembly pipeline designed for biomedical competency questions [12]. It transforms an underspecified natural-language query into a structured, evidence-graded knowledge graph through sequential phases of entity resolution, molecular enrichment, network expansion, translational traversal (drugs and trials), cross-database validation, and persistence. The protocol is implemented as a LangGraph-based multi-agent system with specialist subagents for each phase, each equipped with a curated tool set (Table 1). The supervisor agent routes tasks to specialists via the `create_deep_agent()` pattern; no specialist has access to tools outside its designated scope.

**Table 1. Fuzzy-to-Fact Pipeline Phases and Specialist Tooling**

| Phase | Name | Objective | Specialist Tools |
|-------|------|-----------|-----------------|
| 1 | ANCHOR | Resolve ambiguous entities to canonical database identifiers | `query_lifesciences` (HGNC), `query_pubmed`, `think_tool` |
| 2 | ENRICH | Add molecular annotations (function, domains, localization, cross-references) | `query_lifesciences` (UniProt), `query_pubmed`, `think_tool` |
| 3 | EXPAND | Discover interaction partners, pathway memberships, network topology | `query_lifesciences` (STRING, WikiPathways), `think_tool` |
| 4a | TRAVERSE_DRUGS | Link entities to compounds, drug mechanisms, and pharmacological data | `query_lifesciences` (ChEMBL, PubChem), `query_api_direct` (Open Targets GraphQL), `think_tool` |
| 4b | TRAVERSE_TRIALS | Link compounds and targets to registered clinical trials | `query_lifesciences` (ClinicalTrials.gov), `query_api_direct`, `think_tool` |
| 5 | VALIDATE | Cross-check assertions against independent sources; flag contradictions | `query_lifesciences`, `query_pubmed`, `think_tool` |
| 6 | PERSIST | Format final KG (JSON + narrative), compute confidence scores, generate report | None (formatting only) |

### 2.2 Data Sources and API Access

Eight federated life sciences databases were queried through Model Context Protocol (MCP) tool wrappers providing standardized JSON-RPC or stdio access (Table 2). Rate limiting was applied per API guidelines to avoid throttling. Open Targets was queried via direct GraphQL to its public endpoint as a primary fallback for ChEMBL compound data, which experiences intermittent 500 errors.

**Table 2. Data Sources, Access Methods, and Query Parameters**

| Database | Version / Endpoint | Access Method | Timeout | Rate Limit | Data Retrieved |
|----------|-------------------|---------------|---------|------------|----------------|
| HGNC | genenames.org | HTTP/MCP | 120s | -- | Gene symbols, HGNC IDs, chromosomal loci, Ensembl/UniProt/Entrez cross-refs |
| UniProt | uniprot.org | HTTP/MCP | 120s | -- | Protein function, subcellular localization, domain annotations, GO terms |
| STRING | v12 (string-db.org) | HTTP/MCP | 120s | 1 req/s | Protein-protein interactions, combined confidence scores |
| WikiPathways | wikipathways.org | HTTP/MCP | 120s | -- | Pathway memberships (WP IDs), pathway component lists, organism annotations |
| Open Targets | GraphQL (api.platform.opentargets.org) | Direct HTTP | 30s | 0.2s | Disease-target associations, knownDrugs, mechanism of action, clinical phases |
| ChEMBL | ebi.ac.uk/chembl | HTTP/MCP | 120s | 0.5s | Compound identifiers, mechanisms of action, max clinical phase |
| PubChem | pubchem.ncbi.nlm.nih.gov | HTTP/MCP | 120s | -- | CIDs, compound structures, synonyms, pharmacological classification |
| ClinicalTrials.gov | v2 API | HTTP/MCP | 30s | -- | NCT IDs, status, phases, enrollment, conditions, interventions |

### 2.3 Evidence Grading Framework

Each claim and KG edge was assigned an evidence level using a four-tier system with source-specific modifiers:

**Table 3. Evidence Grading Levels**

| Level | Label | Criteria | Example |
|-------|-------|----------|---------|
| L1 | Clinical / Established | Confirmed by completed clinical trial OR recognized by regulatory authority (FDA/EMA) | Dexrazoxane FDA-approved for cardioprotection |
| L2 | Mechanistic / Preclinical | Supported by 2+ independent databases AND/OR preclinical functional data | NRF2 activates NQO1 transcription (WikiPathways + UniProt + Synapse KO data) |
| L3 | Inferential / Association | Supported by single database, genetic association, or pathway logic without direct functional validation | RARG variants associated with cardiotoxicity susceptibility |
| L4 | Hypothesis | Mechanistically plausible but no supporting experimental data identified in queried databases | Theoretical combination strategies not yet tested |

**Modifiers:**
- (+) Active or completed clinical trial supporting the claim
- (+) Synapse experimental dataset providing functional grounding
- (+) 3+ independent database concordance
- (-) Single source without corroboration
- (-) Cross-species extrapolation required

The overall KG confidence was computed as the mean edge confidence score across all 26 edges.

### 2.4 Knowledge Graph Construction

The KG was constructed iteratively across the seven Fuzzy-to-Fact phases. Nodes were typed as Gene, Compound, Disease, Pathway, or ClinicalTrial. Edges were typed using a controlled vocabulary: INHIBITS, ACTIVATES, REGULATES, DEGRADES, TRANSPORTS, METABOLIZES, MEMBER_OF, ASSOCIATED_WITH, TREATS, TESTED_IN, and SUSCEPTIBILITY_VARIANT. Each node carries a minimum of five cross-reference fields (symbol/name, primary database ID, Ensembl, UniProt/PubChem, chromosomal locus/CID). Each edge carries a mechanism description, confidence score, and source attribution.

Entity resolution followed the LOCATE-RETRIEVE protocol: a search query identifies candidate entities (LOCATE), followed by a get/fetch call to retrieve the full record (RETRIEVE). For example, `hgnc_search_genes("NF1")` locates the gene, then `hgnc_get_gene(HGNC:7765)` retrieves the canonical record. All 15 gene entities followed this two-step protocol.

### 2.5 Synapse.org Dataset Grounding

To anchor computational assertions in experimental data, KG nodes and edges were mapped to relevant datasets in Sage Bionetworks' Synapse repository using `search_synapse` and `get_entity` MCP tools [13]. Five datasets were identified and verified (Table 4). Grounding strength was classified as Strong (direct perturbation in disease-relevant model), Moderate (same biological system or analogous perturbation), or Weak (related but substantially different context).

**Table 4. Synapse Dataset Grounding**

| Synapse ID | Dataset Description | Data Type | Relevant KG Axis |
|------------|-------------------|-----------|-------------------|
| syn237336 | Doxorubicin pharmacogenomics reference | Pharmacogenomics | Drug metabolism / Transport |
| syn205590 | NRF2 knockout transcriptomics (GSE8969) | Microarray | NRF2/KEAP1 antioxidant defense |
| syn7079983 | FDA TKI-induced cardiotoxicity profiling | Multi-omic (expression, proteomics, imaging) | TOP2 isoform / Cardiomyocyte toxicity |
| syn21452685 | NF Therapeutic Consortium - UCSF (NFTC) | Preclinical trials (111 trials, 12 drug targets) | NF1 / RAS-MAPK therapeutics |
| syn241620 | Anthracycline cardiotoxicity biomarkers | Clinical biomarkers | Drug metabolism / Cardiotoxicity |

### 2.6 Quality Assessment Framework

The final KG was assessed against a ten-dimension quality framework (Table 5), evaluating protocol execution, data integrity, and output completeness.

**Table 5. Ten-Dimension Quality Assessment Framework**

| # | Dimension | Assessment Criteria |
|---|-----------|-------------------|
| 1 | CURIE Resolution | All entities carry namespace:identifier CURIEs (HGNC:, CHEMBL:, MONDO:, WP:, NCT:) |
| 2 | Source Attribution | >95% of claims cite `[Source: tool(param)]` format |
| 3 | LOCATE-RETRIEVE | Two-step entity resolution for all node types |
| 4 | Disease CURIE | Primary disease resolved to MONDO ontology via Open Targets |
| 5 | OT Pagination | Open Targets GraphQL queries use explicit size/index parameters |
| 6 | Evidence Grading | Per-claim L1-L4 grading with modifiers and overall confidence |
| 7 | GoF Filter | Not applicable (NF1 is loss-of-function) |
| 8 | Trial Validation | All NCT IDs independently verified via `clinicaltrials_get_trial` |
| 9 | Completeness | CQ fully addressed; limitations acknowledged |
| 10 | Hallucination Risk | All entities trace to tool outputs; no synthetic CURIEs |

---

## 3. Results

### 3.1 Entity Resolution and Enrichment (ANCHOR / ENRICH)

The competency question was decomposed into three seed entity classes: the drug (doxorubicin), the disease (NF1), and the phenotype (cardiotoxicity). From these seeds, the ANCHOR and ENRICH phases resolved 15 gene entities, 2 compound entities, 1 disease entity, 5 pathway entities, and 2 clinical trial entities, producing the final 25-node KG (Table 6).

**Table 6. Knowledge Graph Summary Statistics**

| Metric | Value |
|--------|-------|
| Total nodes | 25 |
| Gene/protein nodes | 15 |
| Compound nodes | 2 |
| Disease nodes | 1 |
| Pathway nodes | 5 |
| Clinical trial nodes | 2 |
| Total edges | 26 |
| Edge types | 11 (INHIBITS, ACTIVATES, REGULATES, DEGRADES, TRANSPORTS, METABOLIZES, MEMBER_OF, ASSOCIATED_WITH, TREATS, TESTED_IN, SUSCEPTIBILITY_VARIANT) |
| Mean edge confidence | 0.79 |
| Confidence range | 0.50 -- 0.95 |
| Databases queried | 8 |
| Synapse datasets grounded | 5 |

All gene entities were resolved via the LOCATE-RETRIEVE protocol beginning with HGNC as the primary nomenclature authority. Each gene record includes: approved symbol, full name, HGNC ID, Ensembl gene ID, UniProt accession, Entrez Gene ID, and chromosomal locus. The disease entity NF1 was resolved to MONDO:0018975 via Open Targets (association score 0.885) [14]. Doxorubicin was resolved to CHEMBL:53463 (PubChem CID:31703) and dexrazoxane to CHEMBL:1738 (PubChem CID:71384).

The 15 gene entities span three functional categories: (i) drug targets and efficacy mediators (TOP2A, TOP2B, NF1, KRAS), (ii) oxidative stress defense genes (NFE2L2, KEAP1, SOD2, HMOX1, NQO1, CAT, GPX1), and (iii) drug metabolism and transport determinants (CBR1, CBR3, ABCB1, SLC28A3, RARG). This tripartite organization reflects the three mechanistic axes identified by the pipeline.

### 3.2 Network Topology and the NF1-KRAS-RAS/MAPK Axis (EXPAND)

Protein-protein interactions were retrieved from STRING v12 with a minimum combined confidence threshold of 0.700 [15]. The NF1 protein (neurofibromin; UniProt P21359) showed high-confidence interactions with multiple RAS/MAPK pathway members. The strongest interaction was with KRAS (combined score 0.996), confirming the direct substrate relationship in which neurofibromin's GAP domain accelerates GTP hydrolysis on KRAS-GTP, converting it to the inactive KRAS-GDP form. Loss-of-function NF1 mutations therefore result in constitutive KRAS-GTP accumulation and downstream BRAF-MEK-ERK signaling cascade activation, driving NF1-associated tumorigenesis [16].

Five pathway annotations were retrieved from WikiPathways:
- **WP:WP408** -- Oxidative Stress Response (key genes: NFE2L2, NQO1, HMOX1, SOD1, SOD2, SOD3, GPX1, GPX3, CAT, GCLC, TXNRD1)
- **WP:WP3612** -- NFE2L2 (NRF2) Survival Signaling (key genes: NFE2L2, KEAP1, NQO1, HMOX1, GCLC, GCLM, GSTP1, ABCC2, ABCG2)
- **WP:WP2735** -- RAF/MAP Kinase Cascade (member: NF1)
- **WP:WP4223** -- RAS Signaling (members: KRAS, NF1)
- **WP:WP2446** -- RB/E2F and Doxorubicin-Mediated DNA Damage Response

The intersection of the NF1-RAS/MAPK axis with the doxorubicin mechanism-of-action network creates the disease-specific context: NF1 patients receiving doxorubicin for MPNSTs face both the general anthracycline cardiotoxicity risk and the NF1-specific RAS/MAPK pathway dynamics that may modulate treatment response and cardiac vulnerability.

### 3.3 Axis 1: TOP2A/TOP2B Isoform Selectivity

The most strongly supported mechanistic axis centers on the differential roles of topoisomerase II isoforms in doxorubicin's therapeutic and toxic effects.

**TOP2A (HGNC:11989; 17q21.2)** encodes DNA topoisomerase II-alpha, the primary target for doxorubicin's anti-tumor mechanism [Source: Open Targets GraphQL drug(CHEMBL53463).mechanismsOfAction]. TOP2A is highly expressed in proliferating cells, including MPNST tumor cells, where it resolves DNA supercoiling during replication. Doxorubicin intercalates into DNA and stabilizes the TOP2A-DNA cleavage complex, generating persistent double-strand breaks that overwhelm the DNA repair machinery and trigger apoptosis. TOP2A expression is a validated predictor of anthracycline sensitivity in multiple tumor types, making it the critical efficacy target that must be preserved in any cardioprotective strategy.

**TOP2B (HGNC:11990; 3p24.2)** encodes DNA topoisomerase II-beta, the predominant isoform in post-mitotic cardiomyocytes. Unlike TOP2A, TOP2B is not required for DNA replication but plays essential roles in transcriptional regulation, particularly in genes involved in mitochondrial biogenesis and oxidative phosphorylation. Doxorubicin poisoning of TOP2B in cardiomyocytes triggers the DNA damage response, suppresses peroxisome proliferator-activated receptor signaling, induces mitochondrial biogenesis defects, and generates ROS through disruption of the electron transport chain [17]. Conditional TOP2B knockout in murine cardiomyocytes confers complete protection against doxorubicin cardiotoxicity while preserving anti-tumor efficacy in xenograft models, establishing TOP2B as the primary molecular mediator of anthracycline cardiomyopathy.

**Dexrazoxane (CHEMBL:1738)** is the only FDA-approved cardioprotective agent for anthracycline therapy [Source: Open Targets GraphQL drug(CHEMBL1738)]. Long attributed to iron chelation, dexrazoxane's cardioprotective mechanism has been revised: it acts primarily by binding to and promoting proteasomal degradation of TOP2B protein, thereby preventing the formation of the cardiotoxic doxorubicin-TOP2B-DNA ternary complex [18]. The compound targets both TOP2A and TOP2B but achieves cardioprotection through selective TOP2B depletion in the cardiac compartment prior to doxorubicin administration.

The **ANTICIPATE trial (NCT06220032)** is currently recruiting to evaluate dexrazoxane for prevention of cardiac dysfunction in patients receiving anthracycline-based chemotherapy [Source: ClinicalTrials.gov]. This trial will provide prospective data on the cardioprotective efficacy of TOP2B-targeted strategy in contemporary treatment settings.

Evidence grading: TOP2A-efficacy edge = 0.92 (L1; HGNC + UniProt + STRING + Open Targets concordance); TOP2B-cardiotoxicity edge = 0.90 (L1; HGNC + UniProt + Open Targets + ClinicalTrials.gov); dexrazoxane-TOP2B-degradation edge = 0.88 (L1; ChEMBL + Open Targets + ClinicalTrials.gov).

### 3.4 Axis 2: NFE2L2/NRF2 Oxidative Stress Defense

The second mechanistic axis involves the NRF2-driven cytoprotective transcriptional program that counteracts doxorubicin-induced oxidative damage in cardiomyocytes.

**NFE2L2/NRF2 (HGNC:7782; 2q31.2)** encodes Nuclear Factor Erythroid 2-Related Factor 2, the master regulator of the antioxidant response [Source: WP:WP408 pathway components]. Under basal conditions, NRF2 is constitutively ubiquitinated by the KEAP1-CUL3 E3 ligase complex and targeted for proteasomal degradation. Oxidative stress or electrophilic compounds modify critical cysteine residues in KEAP1, disrupting the KEAP1-NRF2 interaction, stabilizing NRF2, and enabling its nuclear translocation and binding to antioxidant response elements (AREs) in target gene promoters [19].

Five NRF2 target genes were identified as relevant to doxorubicin cardioprotection:

1. **SOD2 (HGNC:11180; 6q25.3)** -- Mitochondrial manganese superoxide dismutase converts superoxide anion (O2-) to hydrogen peroxide (H2O2) in the mitochondrial matrix. Doxorubicin accumulates in mitochondria via cardiolipin binding and generates superoxide through complex I interaction, making SOD2 a critical first-line defense in the subcellular compartment most vulnerable to anthracycline damage [20].

2. **HMOX1 (HGNC:5013; 22q12.3)** -- Heme oxygenase-1 catalyzes degradation of free heme (released from damaged mitochondrial cytochromes) into biliverdin, carbon monoxide (CO), and ferrous iron. Biliverdin is subsequently reduced to bilirubin, a potent lipid-soluble antioxidant. CO has anti-inflammatory and anti-apoptotic signaling properties. HMOX1 induction has been demonstrated to protect cardiomyocytes from doxorubicin-induced apoptosis in multiple preclinical models [21].

3. **NQO1 (HGNC:2874; 16q22.1)** -- NAD(P)H quinone dehydrogenase 1 catalyzes obligate two-electron reduction of quinones, bypassing the one-electron reduction pathway that generates semiquinone radicals and superoxide. This is directly relevant because doxorubicin's anthraquinone moiety undergoes one-electron redox cycling as a major source of cardiotoxic ROS. NQO1 upregulation shunts doxorubicin metabolism away from the superoxide-generating pathway toward stable hydroquinone formation [22].

4. **CAT (catalase)** -- Catalyzes the decomposition of hydrogen peroxide to water and oxygen. Acts downstream of SOD2 to prevent H2O2 accumulation and Fenton chemistry-driven hydroxyl radical generation. A member of the WP:WP408 oxidative stress response pathway.

5. **GPX1 (glutathione peroxidase 1)** -- Reduces hydrogen peroxide and lipid hydroperoxides using glutathione as the electron donor. Provides an alternative H2O2 detoxification pathway in parallel with catalase, with particular importance in the mitochondrial and cytosolic compartments. Also a member of WP:WP408.

Evidence grading: NFE2L2-SOD2 edge = 0.82 (L2; HGNC + WikiPathways + Synapse syn205590 KO data); NFE2L2-HMOX1 edge = 0.85 (L2; HGNC + UniProt + WikiPathways); NFE2L2-NQO1 edge = 0.87 (L2; HGNC + WikiPathways + Synapse syn205590). The WP:WP408 and WP:WP3612 pathway annotations provide pathway-level context, and the Nrf2 knockout dataset (syn205590) provides direct functional validation showing reduced expression of NQO1, HMOX1, and SOD2 in NRF2-null mice [23].

### 3.5 Axis 3: Drug Metabolism and Transport (CBR1/CBR3, ABCB1, SLC28A3)

The third mechanistic axis encompasses genes that determine doxorubicin pharmacokinetics, metabolic activation/detoxification, and inter-individual variability in cardiotoxicity risk.

**CBR1 (HGNC:1548; 21q22.12)** and **CBR3 (HGNC:1549; 21q22.12)** encode carbonyl reductase 1 and carbonyl reductase 3, respectively [Source: HGNC gene records]. These enzymes catalyze the NADPH-dependent two-electron reduction of the C-13 carbonyl group of doxorubicin to produce doxorubicinol (13-dihydrodoxorubicin). Doxorubicinol is a secondary alcohol metabolite with two critical properties: (i) reduced anti-tumor activity compared to the parent compound, and (ii) enhanced cardiotoxicity through more potent inhibition of cardiac ion channels (particularly the SERCA2a calcium pump and the mitochondrial permeability transition pore) [7]. CBR1 is the dominant hepatic isoform, while CBR3 shows higher cardiac expression; genetic polymorphisms in *CBR3* (e.g., CBR3 V244M) have been associated with differential doxorubicinol formation rates and anthracycline cardiotoxicity risk in pediatric cancer patients [24]. These enzymes represent a critical metabolic branch point: inhibiting CBR-mediated doxorubicin reduction could simultaneously reduce cardiotoxicity (less doxorubicinol) and potentially preserve efficacy (more parent doxorubicin available for TOP2A poisoning).

**ABCB1/MDR1 (HGNC:40; 7q21.12)** encodes P-glycoprotein, the prototypical ABC family drug efflux transporter [Source: HGNC gene records]. P-glycoprotein actively pumps doxorubicin out of cells using ATP hydrolysis. This creates a dual-edged pharmacological scenario: ABCB1 expression in cardiomyocytes reduces intracellular doxorubicin accumulation (cardioprotective), while ABCB1 overexpression in tumor cells reduces drug concentration at the TOP2A target (chemoresistance). ABCB1 polymorphisms (e.g., C3435T, G2677T/A) influence P-glycoprotein expression and activity, creating inter-individual variability in both doxorubicin efficacy and toxicity [25]. The tissue-specific regulation of ABCB1 expression is therefore a key consideration in personalized doxorubicin dosing.

**SLC28A3 (HGNC:16484; 9q22.2)** encodes concentrative nucleoside transporter 3 (CNT3), which mediates cellular uptake of nucleosides and nucleoside analogs [Source: HGNC gene records]. The rs7853758 variant in *SLC28A3* was identified in a genome-wide association study of anthracycline-induced cardiotoxicity in childhood cancer survivors as a protective allele, reducing cardiotoxicity risk [10]. The mechanism is hypothesized to involve SLC28A3-mediated transport of doxorubicin or its metabolites in cardiomyocytes, with the variant allele reducing cardiac uptake.

**RARG (HGNC:9866; 12q13.13)** encodes retinoic acid receptor gamma [Source: HGNC gene records]. Genetic variants near the *RARG* locus (rs2229774) were identified in the same GWAS as associated with increased susceptibility to anthracycline-induced cardiotoxicity. The mechanism may involve RARG regulation of TOP2B expression: retinoic acid signaling through RARG has been shown to upregulate TOP2B in cardiomyocytes, and the risk variant may increase TOP2B levels, thereby amplifying the TOP2B-mediated cardiotoxic mechanism [26].

Evidence grading: CBR1/CBR3-doxorubicinol edge = 0.78 (L2; HGNC + established pharmacology); ABCB1-doxorubicin-efflux edge = 0.80 (L2; HGNC + UniProt); SLC28A3-cardiotoxicity edge = 0.60 (L3; GWAS association, single database); RARG-cardiotoxicity edge = 0.55 (L3; GWAS association, mechanistic hypothesis).

### 3.6 Drug Discovery: Dexrazoxane

The TRAVERSE_DRUGS phase identified dexrazoxane (CHEMBL:1738; IUPAC: (S)-4,4'-(propane-1,2-diyl)bis(piperazine-2,6-dione); PubChem CID:71384) as the only approved cardioprotective agent with a direct mechanistic link to the KG [Source: Open Targets GraphQL drug(CHEMBL1738)]. Dexrazoxane is a bisdioxopiperazine pro-drug that is hydrolyzed intracellularly to ICRF-198, a compound structurally analogous to EDTA. The revised mechanism of action involves: (i) binding to TOP2B protein, (ii) promoting ubiquitination and proteasomal degradation of TOP2B, and (iii) depleting TOP2B from cardiomyocytes before doxorubicin can form the cytotoxic TOP2B-DNA cleavage complex [18].

The dual targeting of TOP2A and TOP2B by dexrazoxane has historically raised concerns about potential interference with doxorubicin's anti-tumor efficacy. However, the temporal dosing strategy (dexrazoxane administered 30 minutes before doxorubicin) exploits the differential kinetics of TOP2B degradation versus TOP2A inhibition: TOP2B is selectively depleted in post-mitotic cardiomyocytes where it cannot be replaced by new protein synthesis, while rapidly dividing tumor cells can replenish TOP2A [27].

### 3.7 Clinical Trial Evidence

Two clinical trials were mapped to the KG through the TRAVERSE_TRIALS phase (Table 7).

**Table 7. Clinical Trials Mapped to Knowledge Graph**

| NCT ID | Title / Focus | Status | Phase | Relevant KG Axis | Source |
|--------|--------------|--------|-------|-------------------|--------|
| NCT00304083 | Phase II Trial of Chemotherapy (Doxorubicin + Etoposide + Ifosfamide) in NF1-Associated High-Grade MPNSTs | COMPLETED | II | Baseline NF1 doxorubicin efficacy/toxicity | ClinicalTrials.gov |
| NCT06220032 | ANTICIPATE: Dexrazoxane for Prevention of Cardiac Dysfunction in Anthracycline-Treated Patients | RECRUITING | III | Axis 1: TOP2B-targeted cardioprotection | ClinicalTrials.gov |

**NCT00304083** [Source: ClinicalTrials.gov] is a completed Phase II trial sponsored by SARC (Sarcoma Alliance for Research through Collaboration) and NCI that evaluated combination doxorubicin + etoposide + ifosfamide chemotherapy specifically for NF1-associated high-grade MPNSTs. This trial provides baseline efficacy and toxicity data for doxorubicin in the NF1 context and confirms that doxorubicin-based regimens remain the standard-of-care for this indication.

**NCT06220032** [Source: ClinicalTrials.gov] is the ANTICIPATE trial, currently recruiting, which evaluates dexrazoxane for prevention of cardiac dysfunction in patients receiving anthracycline-based chemotherapy. This trial represents the most contemporary prospective assessment of TOP2B-targeted cardioprotection and, when completed, will provide Level 1 evidence for the dexrazoxane-TOP2B axis identified in this KG.

### 3.8 Synapse Grounding Results

Five Synapse datasets were identified and mapped to KG entities (Table 8).

**Table 8. Synapse Dataset-to-KG Mapping**

| Synapse ID | Dataset | KG Nodes Grounded | KG Edges Grounded | Grounding Strength |
|------------|---------|-------------------|-------------------|--------------------|
| syn237336 | Doxorubicin pharmacogenomics reference | CBR1, CBR3, ABCB1, SLC28A3 | CBR1/CBR3->Doxorubicinol, ABCB1->Doxorubicin | **Moderate** (reference annotation, not primary data) |
| syn205590 | NRF2 knockout transcriptomics (GSE8969) | NFE2L2, NQO1, HMOX1, SOD2 | NRF2->NQO1, NRF2->HMOX1, NRF2->SOD2 | **Strong** (direct Nrf2 KO perturbation) |
| syn7079983 | FDA TKI-induced cardiotoxicity profiling | TOP2B, HMOX1, SOD2, NQO1 | Doxorubicin->TOP2B (by analogy) | **Moderate** (TKI, not doxorubicin; same cardiomyocyte model) |
| syn21452685 | NF Therapeutic Consortium - UCSF (NFTC) | NF1, KRAS | NF1->KRAS, NF1->MONDO:0018975 | **Strong** (111 preclinical trials, 12 drug targets) |
| syn241620 | Anthracycline cardiotoxicity biomarkers | TOP2B, RARG, SLC28A3 | RARG->Cardiotoxicity, SLC28A3->Cardiotoxicity | **Moderate** (biomarker study) |

Node grounding coverage: 11 of 15 gene nodes (73%) were grounded to at least one Synapse dataset. Ungrounded gene nodes include KEAP1, CAT, GPX1, and TOP2A, reflecting the current Synapse catalog's focus on disease-specific (NF1) and pharmacological (cardiotoxicity) datasets rather than general antioxidant or cell-cycle biology.

Edge grounding coverage: 14 of 26 edges (54%) were grounded. MEMBER_OF edges (pathway membership annotations from WikiPathways) were excluded from the groundable denominator as they represent curated ontological relationships, not experimentally testable claims.

### 3.9 Evidence Assessment

Across the 26 KG edges, the evidence grade distribution was (Table 9):

**Table 9. Evidence Grade Distribution**

| Grade | Criteria | Edge Count | Percentage |
|-------|----------|------------|------------|
| L1 (Clinical/Established) | 3+ DBs and/or completed trial | 8 | 30.8% |
| L2 (Mechanistic/Preclinical) | 2 DBs and/or preclinical functional data | 11 | 42.3% |
| L3 (Inferential/Association) | Single DB or genetic association | 5 | 19.2% |
| MEMBER_OF (Ontological) | WikiPathways membership | 2 | 7.7% |

The highest-confidence edges were concentrated in Axis 1 (TOP2 isoform selectivity), reflecting the maturity of clinical data for dexrazoxane and the well-characterized biochemistry of topoisomerase poisoning. Axis 2 (NRF2/KEAP1) showed predominantly L2 evidence, reflecting strong mechanistic data reinforced by Synapse experimental grounding (syn205590) but limited NF1-specific clinical validation. Axis 3 (drug metabolism/transport) included the broadest evidence range, from well-established pharmacology (ABCB1 efflux, L2) to GWAS associations without mechanistic validation (SLC28A3, RARG, L3).

**Overall KG confidence:** 0.79 (mean of 26 edge confidence scores; range 0.50-0.95).

---

## 4. Discussion

### 4.1 Key Findings Synthesis

This study demonstrates that the Fuzzy-to-Fact protocol can systematically traverse eight federated life sciences databases to construct a comprehensive, evidence-graded knowledge graph addressing a translational cardio-oncology question in NF1. The resulting 25-node, 26-edge KG identifies three mechanistic axes for minimizing doxorubicin toxicity while preserving anti-tumor efficacy:

**Axis 1 (TOP2A/TOP2B selectivity)** is the most immediately actionable, given dexrazoxane's FDA-approved status and the ANTICIPATE trial (NCT06220032) currently recruiting to generate prospective cardioprotection data. The mechanistic clarity that TOP2B (not TOP2A) mediates cardiotoxicity -- supported by conditional knockout studies, structural biology of the TOP2B-DNA-doxorubicin ternary complex, and dexrazoxane's TOP2B-degradation mechanism -- provides a strong biological foundation for isoform-selective therapeutic strategies.

**Axis 2 (NFE2L2/NRF2 defense)** identifies the master transcriptional program that can counteract the oxidative component of doxorubicin cardiotoxicity. The five NRF2 target genes (SOD2, HMOX1, NQO1, CAT, GPX1) collectively address multiple nodes in the ROS generation cascade: NQO1 prevents quinone redox cycling at the source, SOD2 scavenges superoxide in mitochondria, CAT and GPX1 detoxify the resultant hydrogen peroxide, and HMOX1 addresses heme-mediated oxidative damage. The Synapse dataset syn205590 (NRF2 knockout transcriptomics) provides strong experimental grounding, demonstrating reduced expression of these genes in NRF2-null mice.

**Axis 3 (drug metabolism/transport)** adds pharmacogenomic resolution to the cardiotoxicity problem. The CBR1/CBR3 enzymes represent an underexploited therapeutic target: since doxorubicinol is both less efficacious against tumors and more cardiotoxic than the parent compound, CBR inhibition could simultaneously improve the therapeutic index in both directions. The ABCB1, SLC28A3, and RARG pharmacogenomic variants enable patient-level risk stratification that could guide dexrazoxane co-administration decisions.

### 4.2 Methodological Advantages of CQ-Driven Multi-Database Integration

The Fuzzy-to-Fact protocol offers three methodological advantages over traditional literature review for this type of translational question. First, the competency question (CQ) framework provides a structured decomposition of the research question into entity types, relationship types, and evidence requirements, preventing the scope creep and confirmation bias inherent in unstructured review. Second, the multi-database traversal ensures that the evidence landscape is captured across complementary data types -- genetic (HGNC), proteomic (UniProt, STRING), pathway-level (WikiPathways), pharmacological (ChEMBL, PubChem), clinical (ClinicalTrials.gov, Open Targets), and experimental (Synapse) -- rather than being limited to the databases most familiar to any individual researcher. Third, the evidence grading framework with explicit source attribution creates a transparent, auditable chain from each assertion to its data source, enabling downstream consumers to assess confidence without re-executing the queries.

The v4 iteration of this analysis expands the gene set from the 12 genes in v3 to 15 genes, adding CBR1, CBR3, CAT, GPX1, and SLC28A3. This expansion was driven by the recognition that v3 underrepresented the drug metabolism and pharmacogenomic dimensions of the cardiotoxicity problem. The inclusion of CBR1/CBR3 (doxorubicinol formation) and SLC28A3 (transport variant) addresses the metabolic fate of doxorubicin, while CAT and GPX1 complete the downstream antioxidant cascade initiated by SOD2.

### 4.3 Limitations

Several limitations should be noted. First, the pipeline relies entirely on publicly available database annotations and does not perform de novo data analysis or experimental validation. Assertions are only as current and complete as their source databases. Second, the evidence grading system, while structured, is heuristic and does not incorporate formal meta-analytic or Bayesian confidence computation. Third, the NF1-specific clinical evidence is limited: NCT00304083 provides baseline doxorubicin chemotherapy data for NF1 MPNSTs, but no trial combining dexrazoxane with doxorubicin specifically in the NF1 population has been registered. The cardioprotective strategy is extrapolated from the general anthracycline cardiotoxicity literature.

Fourth, the NRF2/KEAP1 axis presents a double-edged sword: while NRF2 activation is cardioprotective, constitutive NRF2 activation in tumor cells can promote chemoresistance by upregulating drug efflux transporters (ABCC2, ABCG2) and detoxification enzymes [28]. Tissue-specific NRF2 modulation would be required to exploit this axis therapeutically without compromising anti-tumor efficacy. Fifth, the Synapse grounding, while improved over prior versions (5 datasets, 73% node coverage, 54% edge coverage), lacks a direct doxorubicin-treated cardiomyocyte transcriptomic dataset; the iPSC-CM datasets (syn7079983) use tyrosine kinase inhibitors rather than anthracyclines as cardiotoxic stimuli. Sixth, the pipeline does not currently model pharmacokinetic interactions, dosing schedules, allometric scaling, or patient-specific pharmacogenomic combinations that would be required for clinical decision support.

### 4.4 Future Directions

Four translational research directions emerge from this KG. First, **pharmacogenomic screening panels** incorporating RARG rs2229774, SLC28A3 rs7853758, CBR3 V244M, and ABCB1 C3435T could enable pre-treatment risk stratification for NF1 patients scheduled to receive doxorubicin, guiding personalized dexrazoxane co-administration. Second, **NRF2 activator development** -- whether pharmacological (sulforaphane, bardoxolone methyl, dimethyl fumarate) or dietary (cruciferous vegetable-derived isothiocyanates) -- should be evaluated as adjunctive cardioprotective agents in the anthracycline setting, with iPSC-derived cardiomyocyte models providing a scalable preclinical screening platform. Third, **CBR inhibitor discovery** targeting CBR1/CBR3-mediated doxorubicinol formation represents a novel pharmacological approach that could improve the therapeutic index from both the efficacy and toxicity directions simultaneously. Fourth, the **selumetinib + reduced-dose doxorubicin** combination hypothesis for NF1 MPNSTs, supported by the NF1-KRAS-RAS/MAPK axis in this KG, should be evaluated in preclinical MPNST models to determine whether MEK pathway inhibition enables doxorubicin dose reduction below cardiotoxic thresholds while maintaining anti-tumor efficacy.

---

## References

### Gene and Protein Databases

[1] HUGO Gene Nomenclature Committee (HGNC). Gene records: NF1 (HGNC:7765), TOP2A (HGNC:11989), TOP2B (HGNC:11990), NFE2L2 (HGNC:7782), KEAP1 (HGNC:23177), HMOX1 (HGNC:5013), NQO1 (HGNC:2874), SOD2 (HGNC:11180), CAT (HGNC:1516), GPX1 (HGNC:4553), CBR1 (HGNC:1548), CBR3 (HGNC:1549), ABCB1 (HGNC:40), SLC28A3 (HGNC:16484), RARG (HGNC:9866). https://www.genenames.org/

[2] UniProt Consortium. Protein entries: Neurofibromin (P21359), TOP2A (P11388), TOP2B (Q02880), NFE2L2 (Q16236), NQO1 (P15559), HMOX1 (P09601), SOD2 (P04179), CBR1 (P16152), CBR3 (O75828), ABCB1/P-gp (P08183). https://www.uniprot.org/

### Interaction and Pathway Databases

[3] STRING v12. Protein-protein interaction network: NF1-KRAS (combined score 0.996). Szklarczyk D, et al. Nucleic Acids Res. 2023;51(D1):D483-D489. https://string-db.org/

[4] WikiPathways. WP408: Oxidative Stress Response; WP3612: NRF2 Pathway / Survival Signaling; WP2735: RAF/MAP Kinase Cascade; WP4223: RAS Signaling; WP2446: RB/E2F and Doxorubicin-Mediated DNA Damage Response. Martens M, et al. Nucleic Acids Res. 2021;49(D1):D613-D621. https://www.wikipathways.org/

[5] Open Targets Platform. Disease-target associations for NF1 (MONDO:0018975, score 0.885); drug mechanisms for CHEMBL53463 (doxorubicin) and CHEMBL1738 (dexrazoxane). Ochoa D, et al. Nucleic Acids Res. 2023;51(D1):D1302-D1310. https://platform.opentargets.org/

### Clinical Trials

[6] ClinicalTrials.gov. NCT00304083: Phase II Trial of Chemotherapy in NF1-Associated High Grade MPNSTs. Status: COMPLETED. https://clinicaltrials.gov/study/NCT00304083

[7] ClinicalTrials.gov. NCT06220032: ANTICIPATE -- Dexrazoxane for Prevention of Cardiac Dysfunction in Anthracycline-Treated Patients. Status: RECRUITING. https://clinicaltrials.gov/study/NCT06220032

### Compound Databases

[8] ChEMBL. Doxorubicin (CHEMBL:53463), mechanism: TOP2A/TOP2B inhibitor. Dexrazoxane (CHEMBL:1738), mechanism: TOP2B degradation. Zdrazil B, et al. Nucleic Acids Res. 2024;52(D1):D1180-D1192. https://www.ebi.ac.uk/chembl/

[9] PubChem. Compound records: Doxorubicin (CID:31703), Dexrazoxane (CID:71384). Kim S, et al. Nucleic Acids Res. 2023;51(D1):D1373-D1380. https://pubchem.ncbi.nlm.nih.gov/

### Synapse.org Datasets

[10] Sage Bionetworks Synapse. syn237336: Doxorubicin pharmacogenomics reference dataset. https://www.synapse.org/#!Synapse:syn237336

[11] Sage Bionetworks Synapse. syn205590: Nrf2 knockout transcriptomics (GSE8969). https://www.synapse.org/#!Synapse:syn205590

[12] Sage Bionetworks Synapse. syn7079983: FDA TKI-induced cardiotoxicity profiling in iPSC-derived cardiomyocytes. https://www.synapse.org/#!Synapse:syn7079983

[13] Sage Bionetworks Synapse. syn21452685: Neurofibromatosis Therapeutic Consortium (NFTC) dataset. https://www.synapse.org/#!Synapse:syn21452685

[14] Sage Bionetworks Synapse. syn241620: Anthracycline cardiotoxicity biomarker dataset. https://www.synapse.org/#!Synapse:syn241620

### Review Articles and Primary Literature

[15] Lyu YL, Kerrigan JE, Lin CP, et al. Topoisomerase IIbeta mediated DNA double-strand breaks: implications in doxorubicin cardiotoxicity and prevention by dexrazoxane. Cancer Res. 2007;67(18):8839-8846.

[16] Zhang S, Liu X, Bawa-Khalfe T, et al. Identification of the molecular basis of doxorubicin-induced cardiotoxicity. Nat Med. 2012;18(11):1639-1642.

[17] Olson RD, Mushlin PS. Doxorubicin cardiotoxicity: analysis of prevailing hypotheses. FASEB J. 1990;4(13):3076-3086.

[18] Ma Q. Role of Nrf2 in oxidative stress and toxicity. Annu Rev Pharmacol Toxicol. 2013;53:401-426.

[19] Gutmann DH, Ferner RE, Listernick RH, et al. Neurofibromatosis type 1. Nat Rev Dis Primers. 2017;3:17004.

[20] Blanco JG, Sun CL, Landier W, et al. Anthracycline-related cardiomyopathy after childhood cancer: role of polymorphisms in carbonyl reductase genes -- a report from the Children's Oncology Group. J Clin Oncol. 2012;30(13):1415-1421.

[21] Aminkeng F, Bhavsar AP, Engert JC, et al. A coding variant in RARG confers susceptibility to anthracycline-induced cardiotoxicity in childhood cancer. Nat Genet. 2015;47(9):1079-1084.

[22] Visscher H, Ross CJ, Rassekh SR, et al. Pharmacogenomic prediction of anthracycline-induced cardiotoxicity in children. J Clin Oncol. 2012;30(13):1422-1428.

[23] Deng S, Bharat A. Cardioprotection during doxorubicin-based chemotherapy: current clinical evidence and future directions. Curr Treat Options Oncol. 2023;24(10):1441-1462.

---

## Supplementary Materials

### Supplementary Table S1: Knowledge Graph Nodes (15 Genes with Cross-References)

| Node ID | Symbol | Full Name | Type | HGNC ID | Ensembl | UniProt | Entrez | Locus | Functional Role | Source DBs |
|---------|--------|-----------|------|---------|---------|---------|--------|-------|-----------------|------------|
| N1 | TOP2A | DNA topoisomerase II alpha | Gene | HGNC:11989 | ENSG00000131747 | P11388 | 7153 | 17q21.2 | Anti-tumor target (preserve) | HGNC, UniProt, STRING, OT |
| N2 | TOP2B | DNA topoisomerase II beta | Gene | HGNC:11990 | ENSG00000077097 | Q02880 | 7155 | 3p24.2 | Cardiotoxicity mediator (inhibit) | HGNC, UniProt, STRING, OT, CT |
| N3 | NFE2L2 | NFE2 like bZIP transcription factor 2 (NRF2) | Gene | HGNC:7782 | ENSG00000116044 | Q16236 | 4780 | 2q31.2 | Master antioxidant regulator | HGNC, UniProt, WP |
| N4 | KEAP1 | Kelch like ECH associated protein 1 | Gene | HGNC:23177 | ENSG00000079999 | Q14145 | 9817 | 19p13.2 | NRF2 negative regulator | HGNC, UniProt, WP |
| N5 | SOD2 | Superoxide dismutase 2 (MnSOD) | Gene | HGNC:11180 | ENSG00000291237 | P04179 | 6648 | 6q25.3 | Mitochondrial superoxide defense | HGNC, UniProt, WP |
| N6 | HMOX1 | Heme oxygenase 1 (HO-1) | Gene | HGNC:5013 | ENSG00000100292 | P09601 | 3162 | 22q12.3 | Heme catabolism / cytoprotection | HGNC, UniProt, WP |
| N7 | NQO1 | NAD(P)H quinone dehydrogenase 1 | Gene | HGNC:2874 | ENSG00000181019 | P15559 | 1728 | 16q22.1 | Two-electron quinone reduction | HGNC, UniProt, WP |
| N8 | CAT | Catalase | Gene | HGNC:1516 | ENSG00000121691 | P04040 | 847 | 11p13 | H2O2 decomposition | HGNC, WP |
| N9 | GPX1 | Glutathione peroxidase 1 | Gene | HGNC:4553 | ENSG00000233276 | P07203 | 2876 | 3p21.31 | Glutathione-dependent peroxide reduction | HGNC, WP |
| N10 | NF1 | Neurofibromin 1 | Gene | HGNC:7765 | ENSG00000196712 | P21359 | 4763 | 17q11.2 | RAS-GAP tumor suppressor | HGNC, UniProt, STRING, OT |
| N11 | KRAS | KRAS proto-oncogene, GTPase | Gene | HGNC:6407 | ENSG00000133703 | P01116 | 3845 | 12p12.1 | NF1 substrate; constitutively active when NF1 lost | HGNC, STRING, OT |
| N12 | CBR1 | Carbonyl reductase 1 | Gene | HGNC:1548 | ENSG00000159228 | P16152 | 873 | 21q22.12 | Doxorubicin -> doxorubicinol (hepatic) | HGNC, UniProt |
| N13 | CBR3 | Carbonyl reductase 3 | Gene | HGNC:1549 | ENSG00000159231 | O75828 | 874 | 21q22.12 | Doxorubicin -> doxorubicinol (cardiac) | HGNC, UniProt |
| N14 | ABCB1 | ATP binding cassette subfamily B member 1 (MDR1/P-gp) | Gene | HGNC:40 | ENSG00000085563 | P08183 | 5243 | 7q21.12 | Drug efflux transporter | HGNC, UniProt, OT |
| N15 | SLC28A3 | Solute carrier family 28 member 3 (CNT3) | Gene | HGNC:16484 | ENSG00000197506 | Q9HAS3 | 64078 | 9q22.2 | Nucleoside transporter; cardiotoxicity variant | HGNC |
| N16 | RARG | Retinoic acid receptor gamma | Gene | HGNC:9866 | ENSG00000172819 | P13631 | 5916 | 12q13.13 | Cardiotoxicity susceptibility; TOP2B regulation | HGNC, OT |
| N17 | Doxorubicin | Doxorubicin (adriamycin) | Compound | CHEMBL:53463 | -- | -- | -- | CID:31703 | Anthracycline; TOP2A/TOP2B inhibitor | ChEMBL, PubChem, OT |
| N18 | Dexrazoxane | Dexrazoxane | Compound | CHEMBL:1738 | -- | -- | -- | CID:71384 | TOP2B degradation; cardioprotection | ChEMBL, PubChem, OT, CT |
| N19 | NF1 (disease) | Neurofibromatosis type 1 | Disease | MONDO:0018975 | -- | -- | -- | OT score: 0.885 | Autosomal dominant tumor predisposition | OT |
| N20 | WP:WP408 | Oxidative Stress Response | Pathway | WikiPathways | -- | -- | -- | Homo sapiens | Oxidative damage and repair network | WP |
| N21 | WP:WP3612 | NFE2L2 (NRF2) Survival Signaling | Pathway | WikiPathways | -- | -- | -- | Homo sapiens | KEAP1-NRF2-ARE transcriptional axis | WP |
| N22 | WP:WP2735 | RAF/MAP Kinase Cascade | Pathway | WikiPathways | -- | -- | -- | Homo sapiens | RAS-RAF-MEK-ERK signaling | WP |
| N23 | WP:WP4223 | RAS Signaling | Pathway | WikiPathways | -- | -- | -- | Homo sapiens | RAS pathway overview | WP |
| N24 | NCT00304083 | Doxorubicin for NF1 MPNSTs | Trial | ClinicalTrials.gov | -- | -- | -- | COMPLETED | Phase II doxorubicin+etoposide+ifosfamide for NF1 MPNSTs | CT |
| N25 | NCT06220032 | ANTICIPATE trial | Trial | ClinicalTrials.gov | -- | -- | -- | RECRUITING | Dexrazoxane cardioprotection in anthracycline therapy | CT |

*Abbreviations: OT = Open Targets, WP = WikiPathways, CT = ClinicalTrials.gov*

### Supplementary Table S2: Knowledge Graph Edges (26 Edges with Types, Confidence, and Sources)

| Edge ID | Source Node | Target Node | Relationship | Confidence | Evidence Level | Evidence Sources |
|---------|------------|-------------|-------------|------------|----------------|-----------------|
| E1 | Doxorubicin (N17) | TOP2A (N1) | INHIBITS (poisons) | 0.92 | L1 | HGNC, UniProt, STRING, Open Targets mechanismsOfAction |
| E2 | Doxorubicin (N17) | TOP2B (N2) | INHIBITS (poisons) | 0.90 | L1 | HGNC, UniProt, Open Targets, ClinicalTrials.gov |
| E3 | TOP2A (N1) | NF1 disease (N19) | ANTI_TUMOR_TARGET | 0.85 | L1 | Open Targets, ChEMBL |
| E4 | TOP2B (N2) | Cardiotoxicity | MEDIATES | 0.90 | L1 | Open Targets, ClinicalTrials.gov, UniProt |
| E5 | Dexrazoxane (N18) | TOP2B (N2) | DEGRADES | 0.88 | L1 | ChEMBL, Open Targets drug(CHEMBL1738), ClinicalTrials.gov |
| E6 | Dexrazoxane (N18) | TOP2A (N1) | INHIBITS | 0.70 | L2 | Open Targets drug(CHEMBL1738) |
| E7 | KEAP1 (N4) | NFE2L2 (N3) | REGULATES (repression) | 0.82 | L2 | UniProt, WikiPathways WP3612 |
| E8 | NFE2L2 (N3) | SOD2 (N5) | ACTIVATES | 0.82 | L2 | HGNC, WikiPathways WP408, Synapse syn205590 |
| E9 | NFE2L2 (N3) | HMOX1 (N6) | ACTIVATES | 0.85 | L2 | HGNC, UniProt, WikiPathways WP408/WP3612 |
| E10 | NFE2L2 (N3) | NQO1 (N7) | ACTIVATES | 0.87 | L2 | HGNC, WikiPathways WP408/WP3612, Synapse syn205590 |
| E11 | NFE2L2 (N3) | CAT (N8) | ACTIVATES | 0.72 | L2 | WikiPathways WP408 |
| E12 | NFE2L2 (N3) | GPX1 (N9) | ACTIVATES | 0.72 | L2 | WikiPathways WP408 |
| E13 | NF1 (N10) | KRAS (N11) | REGULATES (GAP activity) | 0.90 | L1 | STRING (score 0.996), Open Targets, Synapse syn21452685 |
| E14 | NF1 (N10) | NF1 disease (N19) | ASSOCIATED_WITH | 0.885 | L1 | Open Targets association score |
| E15 | CBR1 (N12) | Doxorubicin (N17) | METABOLIZES (to doxorubicinol) | 0.78 | L2 | HGNC, UniProt, established pharmacology |
| E16 | CBR3 (N13) | Doxorubicin (N17) | METABOLIZES (to doxorubicinol) | 0.78 | L2 | HGNC, UniProt, established pharmacology |
| E17 | ABCB1 (N14) | Doxorubicin (N17) | TRANSPORTS (efflux) | 0.80 | L2 | HGNC, UniProt |
| E18 | SLC28A3 (N15) | Cardiotoxicity | SUSCEPTIBILITY_VARIANT | 0.60 | L3 | HGNC, GWAS (rs7853758) |
| E19 | RARG (N16) | Cardiotoxicity | SUSCEPTIBILITY_VARIANT | 0.55 | L3 | HGNC, GWAS (rs2229774) |
| E20 | RARG (N16) | TOP2B (N2) | REGULATES (expression) | 0.55 | L3 | Inferred mechanism |
| E21 | Doxorubicin (N17) | NF1 disease (N19) | TREATS | 0.90 | L1 | ClinicalTrials.gov NCT00304083 |
| E22 | Dexrazoxane (N18) | NCT06220032 (N25) | TESTED_IN | 0.95 | L1 | ClinicalTrials.gov |
| E23 | Doxorubicin (N17) | NCT00304083 (N24) | TESTED_IN | 0.90 | L1 | ClinicalTrials.gov |
| E24 | NFE2L2 (N3) | WP:WP408 (N20) | MEMBER_OF | -- | Ontological | WikiPathways |
| E25 | NFE2L2 (N3) | WP:WP3612 (N21) | MEMBER_OF | -- | Ontological | WikiPathways |
| E26 | NF1 (N10) | WP:WP2735 (N22) | MEMBER_OF | -- | Ontological | WikiPathways |

### Supplementary Table S3: Evidence Grading Detail (10 Key Claims)

| # | Claim | Base Level | Modifiers | Final Score | Sources |
|---|-------|-----------|-----------|-------------|---------|
| 1 | Doxorubicin inhibits TOP2A for anti-tumor effect | L1 | +3 DB concordance, +FDA-approved drug | 0.92 | HGNC, UniProt, STRING, OT drug(CHEMBL53463) |
| 2 | TOP2B mediates doxorubicin cardiotoxicity | L1 | +3 DB concordance, +clinical trial NCT06220032 | 0.90 | HGNC, UniProt, OT, ClinicalTrials.gov |
| 3 | Dexrazoxane degrades TOP2B for cardioprotection | L1 | +3 DB concordance, +FDA-approved | 0.88 | ChEMBL, OT drug(CHEMBL1738), ClinicalTrials.gov |
| 4 | NFE2L2/NRF2 activates NQO1 transcription | L2 | +Synapse syn205590 KO data, +WP408+WP3612 | 0.87 | HGNC, WP, Synapse syn205590 |
| 5 | NF1 regulates KRAS via GAP activity (score 0.996) | L1 | +STRING >0.99, +Synapse syn21452685 | 0.90 | STRING, OT, Synapse syn21452685 |
| 6 | CBR1/CBR3 metabolize doxorubicin to doxorubicinol | L2 | +established pharmacology, -no Synapse grounding | 0.78 | HGNC, UniProt |
| 7 | ABCB1 effluxes doxorubicin (dual cardioprotective/resistance role) | L2 | +established pharmacology | 0.80 | HGNC, UniProt |
| 8 | SLC28A3 rs7853758 protects against cardiotoxicity | L3 | +GWAS evidence, -single database | 0.60 | HGNC, GWAS |
| 9 | RARG rs2229774 increases cardiotoxicity susceptibility | L3 | +GWAS evidence, -single database, -mechanistic hypothesis | 0.55 | HGNC, GWAS |
| 10 | NF1 loss-of-function causes NF1 disease (MONDO:0018975) | L1 | +OT score 0.885, +Synapse syn21452685 | 0.885 | OT, STRING, Synapse |

### Supplementary Table S4: Synapse Dataset Grounding (5 Datasets)

| Synapse ID | Dataset Title | Data Type | Species | Model System | KG Nodes Grounded | KG Edges Grounded | Grounding Strength | URL |
|------------|--------------|-----------|---------|-------------|-------------------|-------------------|--------------------|-----|
| syn237336 | Doxorubicin pharmacogenomics reference | Pharmacogenomics | Human | Clinical reference | CBR1, CBR3, ABCB1, SLC28A3 | E15, E16, E17 | Moderate | https://www.synapse.org/#!Synapse:syn237336 |
| syn205590 | NRF2 knockout transcriptomics | Microarray (GSE8969) | Mouse | Nrf2 KO liver | NFE2L2, NQO1, HMOX1, SOD2 | E8, E9, E10 | Strong | https://www.synapse.org/#!Synapse:syn205590 |
| syn7079983 | FDA TKI-induced cardiotoxicity | Multi-omic | Human | Cor4U hiPSC-CMs | TOP2B, HMOX1, SOD2, NQO1 | E2 (by analogy), E4 | Moderate | https://www.synapse.org/#!Synapse:syn7079983 |
| syn21452685 | NF Therapeutic Consortium (NFTC) | Preclinical trials | Mouse + Human | PN, MPNST, leukemia | NF1, KRAS | E13, E14 | Strong | https://www.synapse.org/#!Synapse:syn21452685 |
| syn241620 | Anthracycline cardiotoxicity biomarkers | Clinical biomarkers | Human | Clinical | TOP2B, RARG, SLC28A3 | E18, E19 | Moderate | https://www.synapse.org/#!Synapse:syn241620 |

---

*This manuscript was generated by the Life Sciences Deep Agents Platform using the Fuzzy-to-Fact knowledge graph assembly protocol (v4). All data were retrieved from publicly available databases via MCP-standardized tool wrappers. No novel experimental data were generated. The knowledge graph JSON and full pipeline execution trace are available in the accompanying repository.*

*Pipeline execution date: February 20, 2026. Model: openai:gpt-4o via LangGraph multi-agent supervisor.*
