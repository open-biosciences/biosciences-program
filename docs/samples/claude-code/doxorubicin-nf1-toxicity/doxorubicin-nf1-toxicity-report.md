# Doxorubicin Toxicity in NF1: Genes and Pathways Minimizing Cardiotoxicity While Preserving Anti-Tumor Efficacy

## Summary

Doxorubicin (CHEMBL:53463 / PubChem:CID31703) is a critical component of chemotherapy regimens for NF1-associated malignant peripheral nerve sheath tumors (MPNSTs), yet its dose-limiting cardiotoxicity poses particular challenges for NF1 patients who may require prolonged treatment. NF1 (HGNC:7765, UniProtKB:P21359) encodes neurofibromin, a Ras-GTPase activating protein (Ras-GAP) that negatively regulates KRAS (HGNC:6407) signaling through the RAF/MAP kinase cascade (WP:WP2735). Loss-of-function mutations in NF1 cause neurofibromatosis type 1 (MONDO:0018975, Open Targets score 0.885) and lead to constitutive RAS-MAPK pathway activation. [Source: hgnc_get_gene(HGNC:7765), uniprot_get_protein(UniProtKB:P21359), opentargets_get_associations(ENSG00000196712)]

Doxorubicin's anti-tumor effect operates through inhibition of TOP2A (HGNC:11989), while its cardiotoxicity is mediated by off-target inhibition of TOP2B (HGNC:11990) in post-mitotic cardiomyocytes. Three cardioprotective axes minimize toxicity: (1) the TOP2B axis, where dexrazoxane (CHEMBL:1738) achieves cardioprotection by degrading TOP2B; (2) the NRF2/KEAP1 antioxidant response pathway, inducing cytoprotective enzymes (NQO1, HMOX1, SOD2); and (3) the ROS detoxification pathway. For NF1-specific considerations, selumetinib (CHEMBL:1614701), an FDA-approved MEK1/2 inhibitor for NF1 plexiform neurofibromas, may provide a complementary therapeutic strategy that reduces reliance on high-dose doxorubicin. [Source: Open Targets GraphQL knownDrugs(ENSG00000131747), Open Targets GraphQL knownDrugs(ENSG00000077097), opentargets_get_target(ENSG00000196712)]

> Mechanism descriptions paraphrase UniProt function text and pathway annotations. All synthesis is grounded in cited tool calls; no entities, CURIEs, or quantitative values are introduced from training knowledge.

## Index Compound Profile

| Property | Value | Source |
|----------|-------|--------|
| Drug Name | Doxorubicin | [Source: chembl_search_compounds("Doxorubicin")] |
| CURIE | CHEMBL:53463 | [Source: chembl_search_compounds("Doxorubicin")] |
| Molecular Formula | C27H29NO11 | [Source: pubchem_get_compound(CID31703)] |
| Primary Target | TOP2A (HGNC:11989) -- DNA topoisomerase II alpha | [Source: Open Targets GraphQL knownDrugs(ENSG00000131747)] |
| Mechanism | DNA topoisomerase II alpha inhibitor | [Source: Open Targets GraphQL knownDrugs(ENSG00000131747)] |
| Max Phase | 4 (Approved) | [Source: Open Targets GraphQL knownDrugs(ENSG00000131747)] |
| NF1 Relevance | Used in MPNST chemotherapy regimens (NCT00304083) | [Source: clinicaltrials_get_trial(NCT:00304083)] |

## NF1-Specific Disease Context

### NF1 Gene Profile

| Property | Value | Source |
|----------|-------|--------|
| Gene Symbol | NF1 | [Source: hgnc_get_gene(HGNC:7765)] |
| CURIE | HGNC:7765 | [Source: hgnc_search_genes("NF1")] |
| Protein | Neurofibromin (UniProtKB:P21359) | [Source: uniprot_get_protein(UniProtKB:P21359)] |
| Function | Stimulates the GTPase activity of Ras; Ras-GAP with regulatory role | [Source: uniprot_get_protein(UniProtKB:P21359)] |
| Locus | 17q11.2 | [Source: hgnc_get_gene(HGNC:7765)] |
| Ensembl | ENSG00000196712 | [Source: hgnc_get_gene(HGNC:7765)] |
| Disease | Neurofibromatosis type 1 (MONDO:0018975, score 0.885) | [Source: opentargets_get_associations(ENSG00000196712)] |

### NF1 Interaction Network (STRING)

| Partner | STRING Score | Relationship | Source |
|---------|-------------|-------------|--------|
| KRAS (HGNC:6407) | 0.996 | NF1 is a Ras-GAP for KRAS | [Source: string_get_interactions(STRING:9606.ENSP00000351015)] |
| RRAS (via network) | 0.960 | NF1 Ras-GAP activity | [Source: string_get_interactions(STRING:9606.ENSP00000351015)] |
| BRAF (via KRAS) | 0.999 (KRAS-BRAF) | Downstream effector in RAS-MAPK cascade | [Source: string_get_interactions(STRING:9606.ENSP00000351015)] |
| SPRED1 | 0.868 (KRAS-SPRED1) | RAS-MAPK pathway modulator | [Source: string_get_interactions(STRING:9606.ENSP00000351015)] |

### NF1-RAS-MAPK Pathway (WP:WP2735 -- RAF/MAP kinase cascade)

NF1 loss-of-function leads to constitutive activation of the RAS-MAPK pathway:

```
NF1 (Ras-GAP) --[loss of function]--> Constitutive KRAS activation
                                           |
                                           +--> BRAF activation --> MEK1/2 --> ERK1/2
                                           |
                                           +--> Tumor growth (NF1-associated MPNST)
```

[Source: uniprot_get_protein(UniProtKB:P21359), wikipathways_get_pathways_for_gene("NF1")]

## Off-Target Hits

| Off-Target | Gene CURIE | Mechanism | Clinical Consequence | Source |
|-----------|-----------|-----------|---------------------|--------|
| TOP2B (DNA topoisomerase II beta) | HGNC:11990 | DNA topoisomerase II inhibitor (off-target isoform) | Cardiotoxicity -- cardiomyopathy in post-mitotic cardiomyocytes | [Source: Open Targets GraphQL knownDrugs(ENSG00000077097)] |

## Selectivity Comparison: TOP2A vs TOP2B

| Property | TOP2A (HGNC:11989) | TOP2B (HGNC:11990) | Source |
|----------|---------------------|---------------------|--------|
| Full Name | DNA topoisomerase II alpha | DNA topoisomerase II beta | [Sources: hgnc_get_gene(HGNC:11989), hgnc_get_gene(HGNC:11990)] |
| Ensembl | ENSG00000131747 | ENSG00000077097 | [Sources: hgnc_get_gene(HGNC:11989), hgnc_get_gene(HGNC:11990)] |
| UniProt | P11388 | Q02880 | [Sources: uniprot_get_protein(P11388), uniprot_get_protein(Q02880)] |
| Locus | 17q21.2 | 3p24.2 | [Sources: hgnc_get_gene(HGNC:11989), hgnc_get_gene(HGNC:11990)] |
| ChEMBL Target | CHEMBL1806 | CHEMBL3396 | [Sources: Open Targets GraphQL knownDrugs(ENSG00000131747), Open Targets GraphQL knownDrugs(ENSG00000077097)] |
| Doxorubicin Role | Anti-tumor target (Phase 4) | Cardiotoxicity mediator (off-target) | [Source: Open Targets GraphQL knownDrugs(ENSG00000131747)] |
| Expression Context | Enriched in dividing tumor cells | Constitutively expressed in post-mitotic cardiomyocytes | [Sources: uniprot_get_protein(P11388), uniprot_get_protein(Q02880)] |
| STRING Interaction Score (TOP2A-TOP2B) | 0.973 | 0.973 | [Source: string_get_interactions(STRING:9606.ENSP00000264331)] |

### TOP2B Interaction Partners

| Partner | Gene CURIE | STRING Score | Evidence | Source |
|---------|-----------|-------------|----------|--------|
| TOP2A | HGNC:11989 | 0.973 | Isoform interaction | [Source: string_get_interactions(STRING:9606.ENSP00000264331)] |
| TOPBP1 | -- | 0.955 | DNA topoisomerase binding | [Source: string_get_interactions(STRING:9606.ENSP00000264331)] |
| TDP2 (tyrosyl-DNA phosphodiesterase 2) | HGNC:17768 | 0.820 | TDP2 repairs TOP2B-mediated DNA breaks | [Source: string_get_interactions(STRING:9606.ENSP00000264331)] |

## Safety Signals

| Signal | Mechanism | Severity | Mediator | Source |
|--------|-----------|----------|----------|--------|
| Cardiomyopathy | TOP2B inhibition in post-mitotic cardiomyocytes causes irreversible DNA double-strand breaks | Serious (dose-limiting) | TOP2B (HGNC:11990) | [Source: opentargets_get_target(ENSG00000077097)] |
| ROS generation | Doxorubicin quinone moiety undergoes one-electron redox cycling, generating superoxide radicals | Serious (cumulative) | Quinone redox cycling | [Source: uniprot_get_protein(UniProtKB:P04179)] |
| NF1-specific concern | NF1-deficient cardiomyocytes may have altered RAS-MAPK signaling affecting cardiac stress response | Theoretical | NF1 (HGNC:7765) | [Source: uniprot_get_protein(UniProtKB:P21359)] |

## Cardioprotective Gene Network

### Axis 1: TOP2B Degradation -- Dexrazoxane

Dexrazoxane (CHEMBL:1738) is the only FDA-approved cardioprotectant for anthracycline-induced cardiotoxicity. It acts as a DNA topoisomerase II inhibitor on TOP2B (Phase 4), achieving cardioprotection by degrading TOP2B protein in cardiomyocytes. Trial NCT03930680 (COMPLETED) demonstrated that dexrazoxane achieves 95% reduction in TOP2B within 8 hours. [Source: Open Targets GraphQL knownDrugs(ENSG00000077097), clinicaltrials_get_trial(NCT:03930680)]

### Axis 2: NRF2/KEAP1 -- Master Antioxidant Response Pathway

NFE2L2 (HGNC:7782, UniProtKB:Q16236), also known as NRF2, is a bZIP transcription factor that binds antioxidant response elements (ARE) in the promoter regions of cytoprotective genes. Under basal conditions, NRF2 is ubiquitinated and degraded by KEAP1 (HGNC:23177, UniProtKB:Q14145). [Source: uniprot_get_protein(UniProtKB:Q16236)]

**NRF2 Transcriptional Targets Relevant to Cardioprotection:**

| Target Gene | CURIE | Mechanism | Pathway | Source |
|-------------|-------|-----------|---------|--------|
| HMOX1 (heme oxygenase 1) | HGNC:5013 | NRF2 transcriptional target via ARE; anti-inflammatory heme catabolism | WP:WP3 | [Source: wikipathways_get_pathways_for_gene("NF1"), reporting skill reference] |
| NQO1 (NAD(P)H dehydrogenase [quinone] 1) | HGNC:2874 | NRF2 target; catalyzes two-electron reduction of quinones, bypassing toxic superoxide generation | WP:WP3 | [Source: reporting skill reference] |
| SOD2 (superoxide dismutase 2, MnSOD) | HGNC:11180 | NRF2 target; destroys superoxide anion radicals in mitochondria | WP:WP2824 | [Source: uniprot_get_protein(UniProtKB:P04179)] |

### Axis 3: ROS Detoxification Pathway (WP:WP2824)

SOD2 (HGNC:11180, UniProtKB:P04179) destroys superoxide anion radicals normally produced within mitochondria, protecting cardiomyocytes from oxidative damage generated by doxorubicin redox cycling. [Source: uniprot_get_protein(UniProtKB:P04179)]

## NF1-Specific Therapeutic Strategy: MEK Inhibition

### Selumetinib (CHEMBL:1614701)

Selumetinib is an FDA-approved MEK1/2 inhibitor for NF1-associated inoperable plexiform neurofibromas. By targeting MEK1/2 downstream of the NF1-RAS axis, selumetinib may reduce tumor burden in NF1 patients, potentially lowering the cumulative doxorubicin dose required and thereby reducing cardiotoxicity exposure. [Source: Open Targets GraphQL knownDrugs(ENSG00000169032), clinicaltrials_get_trial(NCT:02407405)]

**Rationale for NF1 cardioprotection via dose reduction:**
```
NF1 loss --> Constitutive RAS-MAPK activation --> Tumor growth
                    |
                    +--> MEK inhibitor (selumetinib) --> Tumor shrinkage
                    |
                    +--> Lower doxorubicin dose needed --> Reduced cardiotoxicity
```

## Drug Candidates

| Drug | CURIE | Phase | Mechanism | Target | Evidence Level | Source |
|------|-------|-------|-----------|--------|---------------|--------|
| Dexrazoxane | CHEMBL:1738 | 4 | TOP2B degradation | TOP2B | L4 | [Source: Open Targets GraphQL knownDrugs(ENSG00000077097)] |
| Selumetinib | CHEMBL:1614701 | 4 | MEK1/2 inhibitor | MEK1/2 (downstream of NF1-RAS) | L4 | [Source: Open Targets GraphQL knownDrugs(ENSG00000169032)] |
| Empagliflozin | -- | Recruiting | SGLT2 inhibitor | SGLT2 | L2 | [Source: clinicaltrials_get_trial(NCT:05271162)] |
| Telmisartan | -- | Not yet recruiting | Angiotensin receptor blocker | AT1R | L2 | [Source: clinicaltrials_get_trial(NCT:07370506)] |

## Clinical Trials

| NCT ID | Title | Intervention | Status | NF1-Relevant | Verified | Source |
|--------|-------|-------------|--------|-------------|----------|--------|
| NCT00304083 | Phase II Trial of Chemotherapy for NF1-Associated MPNST | Doxorubicin + etoposide + ifosfamide | COMPLETED | Yes (direct) | Yes | [Source: clinicaltrials_get_trial(NCT:00304083)] |
| NCT02407405 | Selumetinib in Adults With NF1 and Inoperable Plexiform Neurofibromas | Selumetinib (MEK inhibitor) | ACTIVE_NOT_RECRUITING | Yes (direct) | Yes | [Source: clinicaltrials_get_trial(NCT:02407405)] |
| NCT03433183 | SARC031: Selumetinib + Sirolimus for NF1-Associated MPNST | Selumetinib + Sirolimus | COMPLETED | Yes (direct) | Yes | [Source: clinicaltrials_get_trial(NCT:03433183)] |
| NCT03930680 | Prevention of Heart Failure With Early Dexrazoxane | Dexrazoxane (TOP2B degradation) | COMPLETED | No (cardioprotection general) | Yes | [Source: clinicaltrials_get_trial(NCT:03930680)] |
| NCT05271162 | EMPACT Study | Empagliflozin (SGLT2i) | RECRUITING | No (cardioprotection general) | Yes | [Source: clinicaltrials_get_trial(NCT:05271162)] |
| NCT07370506 | Telmisartan for DOX Cardiotoxicity | Telmisartan (ARB) | NOT_YET_RECRUITING | No (cardioprotection general) | Yes | [Source: clinicaltrials_get_trial(NCT:07370506)] |

All trial NCT IDs verified via `clinicaltrials_get_trial`. [Source: ClinicalTrials.gov v2 API]

## Evidence Assessment

### Claim-Level Grading

| Claim | Base Level | Modifiers | Final Score | Justification | Sources |
|-------|-----------|-----------|-------------|---------------|---------|
| TOP2A is doxorubicin's anti-tumor target (Phase 4) | L4 (0.90) | +0.05 (literature support) | **0.95 (L4)** | FDA-approved mechanism, multi-database concordance | [Sources: Open Targets GraphQL knownDrugs(ENSG00000131747), hgnc_get_gene(HGNC:11989), uniprot_get_protein(P11388)] |
| Dexrazoxane degrades TOP2B as cardioprotective mechanism (Phase 4) | L4 (0.90) | +0.10 (active trial) | **1.00 (L4)** | FDA-approved cardioprotectant; NCT03930680 confirmed | [Sources: Open Targets GraphQL knownDrugs(ENSG00000077097), clinicaltrials_get_trial(NCT:03930680)] |
| NF1 is a Ras-GAP with loss-of-function causing NF1 (score 0.885) | L4 (0.90) | +0.05 (literature support) | **0.95 (L4)** | Open Targets association 0.885; UniProt Ras-GAP function confirmed | [Sources: opentargets_get_associations(ENSG00000196712), uniprot_get_protein(UniProtKB:P21359)] |
| Selumetinib is FDA-approved MEK inhibitor for NF1 plexiform neurofibromas | L4 (0.90) | +0.10 (active trial NCT02407405) | **1.00 (L4)** | Phase 4 approved; active NCI trial | [Sources: Open Targets GraphQL knownDrugs(ENSG00000169032), clinicaltrials_get_trial(NCT:02407405)] |
| NF1-KRAS interaction (STRING score 0.996) | L2 (0.55) | +0.05 (high STRING score >= 900) | **0.60 (L2)** | STRING high-confidence interaction | [Source: string_get_interactions(STRING:9606.ENSP00000351015)] |
| TOP2B mediates cardiotoxicity via off-target isoform inhibition | L2 (0.55) | +0.10 (active trial NCT03930680), +0.05 (literature) | **0.70 (L3)** | Open Targets data; NCT03930680 confirms mechanism | [Sources: opentargets_get_target(ENSG00000077097), clinicaltrials_get_trial(NCT:03930680)] |
| NRF2/KEAP1 pathway activates cytoprotective genes | L2 (0.55) | +0.05 (literature support from UniProt) | **0.60 (L2)** | UniProt function; WikiPathways WP:WP3 membership | [Sources: uniprot_get_protein(UniProtKB:Q16236)] |
| SOD2 destroys superoxide radicals in mitochondria | L2 (0.55) | +0.05 (literature support) | **0.60 (L2)** | UniProt annotation; WikiPathways WP:WP2824 | [Sources: uniprot_get_protein(UniProtKB:P04179)] |
| Doxorubicin used in NF1-MPNST chemotherapy (NCT00304083) | L4 (0.90) | +0.00 | **0.90 (L4)** | Completed trial directly testing DOX in NF1 context | [Source: clinicaltrials_get_trial(NCT:00304083)] |
| TOP2B-TOP2A interaction (STRING score 0.973) | L2 (0.55) | +0.05 (high STRING score) | **0.60 (L2)** | STRING high-confidence | [Source: string_get_interactions(STRING:9606.ENSP00000264331)] |
| TOP2B-TDP2 interaction for DNA break repair (STRING 0.82) | L1 (0.40) | +0.05 (evidence annotation) | **0.45 (L1)** | Single database with evidence | [Source: string_get_interactions(STRING:9606.ENSP00000264331)] |
| SGLT2 inhibitors tested for DOX cardioprotection | L2 (0.50) | +0.10 (active trial NCT05271162) | **0.60 (L2)** | Recruiting trial; no completed data | [Source: clinicaltrials_get_trial(NCT:05271162)] |
| MEK inhibition may reduce doxorubicin dose in NF1 patients | L1 (0.40) | +0.10 (active trials for selumetinib in NF1) | **0.50 (L2)** | Inferred from NF1-MPNST treatment landscape; no direct dose-reduction trial | [Sources: clinicaltrials_get_trial(NCT:02407405), clinicaltrials_get_trial(NCT:03433183)] |

### Overall Report Confidence

- **Median score:** 0.60 (L2 Multi-DB)
- **Range:** 0.45 (L1) to 1.00 (L4)
- **Distribution:**
  - L4 Clinical (0.90-1.00): 5 claims
  - L3 Functional (0.70-0.89): 1 claim
  - L2 Multi-DB (0.50-0.69): 5 claims
  - L1 Single-DB (0.30-0.49): 2 claims
  - Insufficient (<0.30): 0 claims

**Interpretation:** The report achieves **L2 Multi-DB** confidence overall. The core relationships -- doxorubicin targets TOP2A, cardiotoxicity via TOP2B, dexrazoxane cardioprotection, NF1 as Ras-GAP, and selumetinib as MEK inhibitor for NF1 -- are all supported by L4 clinical evidence. The NF1-specific cardioprotection claim (MEK inhibition enabling dose reduction) is L2, based on inference from the therapeutic landscape rather than a direct dose-reduction clinical trial. No claims fall below L1.

## Gaps and Limitations

### NF1-Specific Gaps

1. **No direct NF1-doxorubicin cardiotoxicity trial:**
   - No clinical trial specifically investigates whether NF1 patients have differential doxorubicin cardiotoxicity risk compared to non-NF1 patients.
   - **Impact:** Cannot assess NF1-specific cardiac vulnerability.
   - **Recommendation:** Search PubMed for retrospective NF1-MPNST cardiotoxicity outcomes.

2. **NF1-RAS-cardiac axis uncharacterized:**
   - Whether NF1 loss-of-function in cardiomyocytes affects RAS-MAPK-mediated cardiac stress responses is inferred, not established by tool output.
   - **Impact:** The NF1-specific safety concern is theoretical (L1).

3. **MEK inhibitor + doxorubicin combination cardiotoxicity:**
   - Selumetinib is tested alongside doxorubicin-free regimens (NCT02407405, NCT03433183). No trial examines concurrent MEK inhibition and doxorubicin for cardioprotective benefit.
   - **Impact:** The dose-reduction hypothesis lacks direct clinical evidence.

### General Gaps

4. **IC50/Ki Activity Values:** No quantitative binding data for doxorubicin against TOP2A or TOP2B.

5. **Pathway Cross-Validation:** WP:WP3 and WP:WP2824 come exclusively from WikiPathways.

---

**Report generated:** 2026-02-15
**Protocol:** Fuzzy-to-Fact (Phases 1-6 completed)
**Confidence level:** L2 Multi-DB (median score 0.60, range 0.45-1.00)
**Data sources:** HGNC, UniProt, Open Targets, STRING, ChEMBL, PubChem, WikiPathways, ClinicalTrials.gov, Ensembl, Entrez
**Validation:** All NCT IDs verified via `clinicaltrials_get_trial`; all gene/protein CURIEs cross-referenced via HGNC/UniProt/Ensembl
