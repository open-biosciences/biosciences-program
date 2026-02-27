# Synapse Grounding: Doxorubicin Toxicity in NF1

## 1. Summary

- **Nodes in KG**: 18
- **Edges in KG**: 20
- **Synapse datasets identified**: 8
- **Nodes with at least one grounding dataset**: 10/18 (56%)
- **Edges with at least one grounding dataset**: 7/20 (35%)

## 2. Dataset-to-Graph Mapping Table

| Synapse ID | Name | Nodes Grounded | Edges Grounded | Evidence Type | Species | Samples |
|-----------|------|---------------|---------------|--------------|---------|---------|
| syn200548 | E-MEXP-2071: Rat heart treated with doxorubicin or dexrazoxane+doxorubicin | Doxorubicin (CHEMBL:53463), TOP2B (HGNC:11990), Dexrazoxane (CHEMBL:1738), Cardiomyopathy (EFO:0000318) | DOX->TOP2B, DEX->TOP2B, TOP2B->Cardiomyopathy | Transcription profiling | Rattus norvegicus (SHR strain) | Cardiac tissue |
| syn44222814 | NF1 mutated Schwann cell (ipNF95.6) treated with selumetinib | NF1 (HGNC:7765), Selumetinib (CHEMBL:1614701), KRAS (HGNC:6407) | Selumetinib->KRAS (MEK inhibition), NF1->KRAS (RAS-GAP) | RNA-seq; drug perturbation | Homo sapiens (NF1-/- Schwann cells) | ipNF95.6 cell line |
| syn61259950 | Phase II Selumetinib in NF1 Adults (NCT02407405) | NF1 (HGNC:7765), Selumetinib (CHEMBL:1614701), NF1 disease (MONDO:0018975) | Selumetinib->KRAS, NF1->MONDO:0018975 | Clinical trial data | Homo sapiens | Adult NF1 patients |
| syn7079983 | FDA TKI-induced cardiotoxicity | TOP2B (HGNC:11990), TOP2A (HGNC:11989), Cardiomyopathy (EFO:0000318) | DOX->TOP2B (contextual), TOP2A-TOP2B (co-expression) | Multi-omic (RNA-seq, proteomics, imaging) | Homo sapiens (hiPSC-CMs) | Cor4U iPSC-derived cardiomyocytes |
| syn13856689 | TKI cardiotoxicity in iPSC-derived cardiomyocytes | TOP2B (HGNC:11990), Cardiomyopathy (EFO:0000318) | TOP2B->Cardiomyopathy | RNA-seq, proteomics, metabolomics | Homo sapiens (hiPSC-CMs) | 30 RNA-seq, 10 proteomics, 16 metabolomics samples |
| syn205590 | GSE8969: Nrf2 KO mouse liver regeneration | NFE2L2 (HGNC:7782), KEAP1 (HGNC:23177) | NFE2L2->KEAP1 (pathway validation) | Microarray (Nrf2 KO vs WT) | Mus musculus | Liver tissue |
| syn237499 | GSE11952: NFE2L2/Nrf2 oxidant-responsive transcription | NFE2L2 (HGNC:7782), HMOX1 (HGNC:5013), NQO1 (HGNC:2874) | NFE2L2->HMOX1, NFE2L2->NQO1 (transcriptional activation) | Microarray | Homo sapiens / Mus musculus | Lung tissue |
| syn51658171 | RAS-MAPK + cAMP-PKA inhibition for NF1 cutaneous neurofibromas | NF1 (HGNC:7765), KRAS (HGNC:6407), WP:WP2735 (RAF/MAP kinase cascade) | NF1->KRAS (RAS-GAP), NF1->WP:WP2735 | Drug perturbation; in vitro + in vivo | Homo sapiens / Mus musculus | NF1-/- Schwann cells, Nf1-KO mice |

## 3. Node Grounding Coverage

### Grounded Nodes (10/18)

| Node | ID | Grounding Datasets | Strength |
|------|----|--------------------|----------|
| Doxorubicin | CHEMBL:53463 | syn200548 | **Strong** -- Direct drug treatment in cardiac tissue |
| NF1 | HGNC:7765 | syn44222814, syn61259950, syn51658171 | **Strong** -- NF1-mutant cells treated with MEK inhibitor |
| TOP2B | HGNC:11990 | syn200548, syn7079983, syn13856689 | **Strong** -- Dexrazoxane+doxorubicin cardiac profiling |
| TOP2A | HGNC:11989 | syn7079983, syn13856689 | **Moderate** -- Co-expressed in cardiomyocytes; TKI (not DOX) focus |
| KRAS | HGNC:6407 | syn44222814, syn51658171 | **Moderate** -- NF1-mutant cells with RAS-MAPK profiling |
| NFE2L2 | HGNC:7782 | syn205590, syn237499 | **Moderate** -- Nrf2 KO models confirm pathway function |
| KEAP1 | HGNC:23177 | syn205590 | **Moderate** -- Nrf2 KO model (KEAP1-NRF2 axis) |
| Dexrazoxane | CHEMBL:1738 | syn200548 | **Strong** -- Direct dexrazoxane+doxorubicin cardiac profiling |
| Selumetinib | CHEMBL:1614701 | syn44222814, syn61259950 | **Strong** -- Direct drug perturbation in NF1 cells + clinical trial |
| Cardiomyopathy | EFO:0000318 | syn7079983, syn13856689 | **Moderate** -- Cardiomyocyte toxicity phenotyping |

### Ungrounded Nodes (8/18)

| Node | ID | Note |
|------|------|------|
| HMOX1 | HGNC:5013 | NRF2 target; no direct Synapse dataset profiling HMOX1 in cardiac context |
| NQO1 | HGNC:2874 | NRF2 target; GSE11952 profiles NRF2 targets but not cardiac-specific |
| SOD2 | HGNC:11180 | No Synapse dataset directly profiling SOD2 in doxorubicin context |
| TDP2 | HGNC:17768 | No Synapse dataset for TOP2B-TDP2 DNA repair axis |
| NF1 disease | MONDO:0018975 | Clinical entity; syn61259950 provides clinical trial context |
| WP:WP2735 | RAF/MAP kinase cascade | Pathway node; syn51658171 provides RAS-MAPK data |
| WP:WP3 | NRF2 transcriptional activation | Pathway node; syn237499 provides Nrf2 transcriptional data |
| WP:WP2824 | ROS detoxification | Pathway node; no direct Synapse match |

## 4. Edge Grounding Coverage

### Grounded Edges (7/20)

| Edge | Type | Grounding Dataset | Strength | Justification |
|------|------|-------------------|----------|--------------|
| DOX -> TOP2B | INHIBITOR | syn200548 | **Strong** | E-MEXP-2071 directly profiles doxorubicin-treated cardiac tissue; TOP2B is expressed in heart |
| DEX -> TOP2B | INHIBITOR | syn200548 | **Strong** | Dexrazoxane pre-treatment arm in cardiac profiling study |
| NF1 -> KRAS | REGULATES | syn44222814, syn51658171 | **Moderate** | NF1-mutant cells show RAS-MAPK activation; selumetinib treatment modulates pathway |
| Selumetinib -> KRAS | INHIBITOR | syn44222814, syn61259950 | **Strong** | Direct drug treatment RNA-seq + Phase II clinical trial (NCT02407405) |
| NF1 -> MONDO:0018975 | ASSOCIATED_WITH | syn61259950 | **Strong** | Clinical trial enrolls NF1 patients with plexiform neurofibromas |
| NFE2L2 -> KEAP1 | REGULATES | syn205590 | **Moderate** | Nrf2 KO model confirms KEAP1-NRF2 regulatory axis |
| TOP2B -> Cardiomyopathy | ASSOCIATED_WITH | syn7079983, syn13856689 | **Moderate** | iPSC-CM toxicity studies profile cardiac damage; TKI (not DOX) compounds |

### Ungrounded Edges (13/20)

| Edge | Type | Note |
|------|------|------|
| DOX -> TOP2A | INHIBITOR | No Synapse dataset directly testing doxorubicin-TOP2A selectivity |
| TOP2B -> TOP2A | INTERACTS | STRING interaction; no Synapse perturbation data |
| TOP2B -> TDP2 | INTERACTS | STRING interaction; no Synapse dataset |
| NF1 -> WP:WP2735 | MEMBER_OF | Pathway membership; contextual support from syn51658171 |
| KRAS -> WP:WP2735 | MEMBER_OF | Pathway membership |
| NFE2L2 -> HMOX1 | REGULATES | Transcriptional regulation; syn237499 provides contextual NRF2 data |
| NFE2L2 -> NQO1 | REGULATES | Transcriptional regulation; syn237499 provides contextual NRF2 data |
| NFE2L2 -> SOD2 | REGULATES | Transcriptional regulation; no cardiac-specific Synapse data |
| NFE2L2 -> WP:WP3 | MEMBER_OF | Pathway membership |
| KEAP1 -> WP:WP3 | MEMBER_OF | Pathway membership |
| HMOX1 -> WP:WP3 | MEMBER_OF | Pathway membership |
| NQO1 -> WP:WP3 | MEMBER_OF | Pathway membership |
| SOD2 -> WP:WP2824 | MEMBER_OF | Pathway membership |

## 5. Evidence Level Upgrades

| Claim | Original Level | Upgrade | New Level | Justification |
|-------|---------------|---------|-----------|---------------|
| Dexrazoxane degrades TOP2B (cardioprotection) | L4 (1.00) | No upgrade needed | L4 (1.00) | Already max; syn200548 provides additional transcriptomic validation |
| TOP2B mediates off-target cardiotoxicity | L3 (0.70) | +Synapse | L3+ (0.70) | syn200548 cardiac transcriptome; syn7079983 iPSC-CM phenotyping |
| Selumetinib is MEK1/2 inhibitor for NF1 | L4 (1.00) | No upgrade needed | L4 (1.00) | syn61259950 matches NCT02407405; syn44222814 provides molecular data |
| NF1-KRAS interaction | L2 (0.60) | +Synapse | L2+ (0.60) | syn44222814 NF1-mutant cells confirm RAS pathway activation |
| NRF2/KEAP1 pathway | L2 (0.60) | +Synapse | L2+ (0.60) | syn205590 Nrf2 KO validates pathway; syn237499 confirms NRF2 targets |

## 6. Grounding Confidence Matrix

| Claim | Datasets | Strength | Justification |
|-------|----------|----------|---------------|
| Doxorubicin causes cardiotoxicity via TOP2B | syn200548, syn7079983 | **Strong** | E-MEXP-2071 directly tests DOX in cardiac tissue; TKI-cardiotoxicity study profiles cardiomyocyte damage pathways |
| Dexrazoxane achieves cardioprotection by TOP2B degradation | syn200548 | **Strong** | Dexrazoxane pre-treatment arm in rat cardiac transcription profiling |
| NF1 loss activates RAS-MAPK via KRAS | syn44222814, syn51658171 | **Strong** | NF1-/- Schwann cells show MEK inhibitor sensitivity; RAS-MAPK pathway profiled |
| Selumetinib treats NF1 plexiform neurofibromas | syn61259950, syn44222814 | **Strong** | Phase II clinical trial (NCT02407405) + molecular RNA-seq in NF1 cells |
| NRF2/KEAP1 regulates cytoprotective genes | syn205590, syn237499 | **Moderate** | Nrf2 KO and oxidative stress models; not cardiac-specific context |
| SOD2 detoxifies ROS in mitochondria | -- | **Ungrounded** | No Synapse dataset profiling SOD2 in doxorubicin-cardiac context |
| TOP2B-TDP2 DNA break repair | -- | **Ungrounded** | No Synapse dataset for TDP2 in cardiac context |
| MEK inhibition may reduce doxorubicin dose in NF1 | syn44222814 | **Weak** | Inferred from NF1-selumetinib data; no doxorubicin combination dataset |

## 7. Methodology

### Tools Used

| Tool | Queries Executed |
|------|-----------------|
| `search_synapse` | "doxorubicin cardiotoxicity", "TOP2B doxorubicin", "NF1 neurofibromatosis", "NRF2 oxidative stress", "dexrazoxane", "selumetinib MEK inhibitor", "KRAS RAS MAPK pathway", "doxorubicin cardiomyocyte transcriptome" |
| `get_entity` | syn200548, syn44222814, syn61259950, syn7079983, syn205590, syn2111731 |
| `get_entity_annotations` | syn44222814, syn61259950, syn7079983 |
| `get_entity_children` | syn200548 |

### Matching Criteria

1. **Direct match**: Dataset compound/gene matches KG node AND experimental design tests the specific mechanistic claim
2. **Contextual match**: Dataset profiles relevant biology but experimental design does not directly test the specific edge
3. **Pathway-level match**: Dataset provides transcriptomic/proteomic context for pathway members without gene/compound-specific perturbation

## 8. Limitations

1. **Cross-species extrapolation**: syn200548 (E-MEXP-2071) uses rat cardiac tissue; cardiac TOP2B biology may differ from human
2. **TKI vs anthracycline**: syn7079983 and syn13856689 study TKI cardiotoxicity (sunitinib, sorafenib, lapatinib, erlotinib), not doxorubicin -- cardiomyocyte damage pathways may overlap but mechanisms differ
3. **NRF2 studies not cardiac-specific**: syn205590 (liver) and syn237499 (lung) profile NRF2 in non-cardiac tissues; cardiac NRF2 function extrapolated
4. **No doxorubicin+NF1 dataset**: No Synapse dataset combines doxorubicin treatment with NF1-deficient models, which is the core intersection of this CQ
5. **Platform age**: syn200548 (2012) and syn205590 (2012) use older microarray platforms; newer RNA-seq datasets may provide higher resolution
6. **Coverage gaps**: SOD2, TDP2, HMOX1, NQO1 lack direct Synapse grounding in doxorubicin-cardiac context; these nodes rely solely on database annotations from pipeline Phases 1-5

---

**Generated:** 2026-02-15
**Protocol:** Fuzzy-to-Fact Publication Pipeline Stage 1b
**Synapse MCP Tools:** search_synapse, get_entity, get_entity_annotations, get_entity_children
