# PKS Module — ClusterCAD Database Tools

**Author:** Dennis Wu
**Course:** BioE 234, Spring 2026

---

## What these tools do

This set of six tools forms the **natural parts discovery** stage of the PKS design pipeline. Once a PKS module architecture has been proposed by RetroTide or TridentSynth, these tools search the ClusterCAD database of 531 experimentally characterized PKS clusters to find real biological parts — amino acid sequences and DNA sequences — that implement each proposed module.

The tools also support standalone exploration: a user can browse all PKS clusters, inspect domain architectures module by module, trace biosynthetic intermediates step by step, and search across all 531 clusters simultaneously for modules matching any domain type or substrate.

PKS design (from RetroTide / TridentSynth)
│
▼
clustercad_search_domains   →  find natural modules matching each domain profile
│
▼
clustercad_list_clusters    →  browse clusters by name to find MIBiG accessions
│
▼
clustercad_cluster_details  →  get subunit and module count for a cluster
│
▼
clustercad_get_subunits     →  get full domain architecture and intermediate SMILES
│
▼
clustercad_domain_lookup    →  get amino acid sequence for a specific domain
│
▼
clustercad_subunit_lookup   →  get full AA + DNA sequence for an entire subunit
│
▼
reverse_translate + submit_antismash → codon-optimize and validate the construct

## Novelty Aspect

While ClusterCAD itself is an existing database, the six tools built here make it programmatically accessible to an AI agent for the first time as part of an automated PKS design pipeline. Each tool is independently useful for a PKS scientist, but together they form a complete parts-discovery workflow that previously required manual navigation of the ClusterCAD web interface.

The most novel tool is `clustercad_search_domains`, which builds a local cache of all 531 PKS clusters and enables instant cross-database search by domain type, substrate specificity, activity status, and module architecture — capabilities not available through the ClusterCAD web interface. The synonym translation layer (e.g. "butyryl" → "butmal") makes the tool robust to natural language inputs without requiring the user or AI to know ClusterCAD's internal vocabulary. The cache builds once (~70 seconds) and is committed to the repository so all teammates benefit immediately with no startup delay.

---

## Files

| File | Description |
|------|-------------|
| `tools/clustercad_list_clusters.py` | Lists all PKS clusters with MIBiG accessions |
| `tools/clustercad_list_clusters.json` | C9 JSON wrapper |
| `tools/clustercad_cluster_details.py` | Returns subunit/module summary for a cluster |
| `tools/clustercad_cluster_details.json` | C9 JSON wrapper |
| `tools/clustercad_get_subunits.py` | Returns full domain architecture and intermediate SMILES |
| `tools/clustercad_get_subunits.json` | C9 JSON wrapper |
| `tools/clustercad_domain_lookup.py` | Returns amino acid sequence for a specific domain |
| `tools/clustercad_domain_lookup.json` | C9 JSON wrapper |
| `tools/clustercad_subunit_lookup.py` | Returns AA + DNA sequence for a full subunit |
| `tools/clustercad_subunit_lookup.json` | C9 JSON wrapper |
| `tools/clustercad_search_domains.py` | Cross-database search by domain type and substrate |
| `tools/clustercad_search_domains.json` | C9 JSON wrapper |
| `tools/clustercad_cache.json` | Local cache of all 531 clusters (auto-generated, committed to repo) |
| `tests/test_clustercad_tools.py` | 60+ pytest tests covering all six tools |

---

## Tool 1 — `clustercad_list_clusters`

Returns a browsable list of PKS clusters from ClusterCAD with their MIBiG accession numbers and descriptions. Used as the entry point when the user references a cluster by name rather than accession number. Gemini should always call this tool first rather than asking the user for an accession number.

**Input:** `reviewed_only` (bool, default true) — if true, only return manually reviewed clusters. `max_results` (int, default 20) — maximum number of clusters to return.

**Output:** A list of accession numbers and cluster descriptions, e.g. `[{"accession": "BGC0001492.1", "description": "Abyssomicin"}]`.

**Example:**

User: What PKS clusters are available for Abyssomicin?
Gemini calls: clustercad_list_clusters(reviewed_only=false, max_results=20)
→ finds BGC0001492.1

---

## Tool 2 — `clustercad_cluster_details`

Returns a summary of a specific PKS cluster given its MIBiG accession number, including subunit count, module count, and a link to the ClusterCAD web page. Used to get a quick overview before retrieving the full domain architecture.

**Input:** `mibig_accession` (string, required) — MIBiG accession number e.g. `"BGC0001492.1"`.

**Output:** A dict with `accession`, `description`, `subunit_count`, `module_count`, and `url`, e.g. `{"accession": "BGC0001492.1", "description": "Abyssomicin", "subunit_count": 3, "module_count": 7, "url": "https://clustercad.jbei.org/pks/BGC0001492.1"}`.

**Example:**
User: How many modules does the Abyssomicin PKS have?
Gemini calls: clustercad_list_clusters() → BGC0001492.1
Gemini calls: clustercad_cluster_details("BGC0001492.1") → 3 subunits, 7 modules

---

## Tool 3 — `clustercad_get_subunits`

Returns the complete domain architecture for every subunit and module in a PKS cluster. For each module it provides all domain types present (KS, AT, KR, DH, ER, ACP, TE), domain IDs for downstream sequence retrieval, substrate annotations, and the predicted intermediate SMILES — the growing chain state after that module has completed its condensation and reductive cycle.

This is the key tool for tracing how a natural PKS builds its product step by step, and for identifying which module produces an intermediate matching an engineering target. Domain IDs returned here are required input for `clustercad_domain_lookup`, and subunit IDs are required input for `clustercad_subunit_lookup`.

**Input:** `mibig_accession` (string, required) — MIBiG accession number.

**Output:** A list of subunits, each with `subunit_name`, `subunit_id`, and a list of `modules`. Each module has `module` (e.g. "module 0"), `product_smiles`, and a list of `domains` each with `domain_type`, `domain_id`, and `annotation`.

**Example:**
User: Show me the full domain architecture of the Abyssomicin PKS.
Gemini calls: clustercad_list_clusters() → BGC0001492.1
Gemini calls: clustercad_get_subunits("BGC0001492.1") → full module-by-module architecture

---

## Tool 4 — `clustercad_domain_lookup`

Returns the amino acid sequence and positional coordinates of a single domain identified by its domain ID. Domain IDs are obtained from `clustercad_get_subunits`. This tool provides the sequence-level information needed to design domain-swapping constructs for chimeric PKS engineering.

**Input:** `domain_id` (integer, required) — domain ID from `clustercad_get_subunits`.

**Output:** A dict with `domain_id`, `name`, `start`, `stop`, `annotations`, and `AAsequence`.

**Example:**
User: Give me the amino acid sequence of the AT domain in module 0 of Abyssomicin.
Gemini calls: clustercad_list_clusters() → BGC0001492.1
Gemini calls: clustercad_get_subunits("BGC0001492.1") → domain_id 27717
Gemini calls: clustercad_domain_lookup(27717) → amino acid sequence

---

## Tool 5 — `clustercad_subunit_lookup`

Returns both the amino acid sequence and the full nucleotide (DNA) sequence for an entire PKS subunit, along with its GenBank accession number. Subunit IDs are obtained from `clustercad_get_subunits`. This is the final step before gene synthesis — providing the exact DNA sequence to order or pass to `reverse_translate` for codon optimization.

**Input:** `subunit_id` (integer, required) — subunit ID from `clustercad_get_subunits`.

**Output:** A dict with `subunit_id`, `name`, `start`, `stop`, `genbank_accession`, `AAsequence`, and `DNAsequence`.

**Example:**
User: Get the DNA sequence of the AbsB1 subunit in Abyssomicin.
Gemini calls: clustercad_list_clusters() → BGC0001492.1
Gemini calls: clustercad_get_subunits("BGC0001492.1") → subunit_id 24119
Gemini calls: clustercad_subunit_lookup(24119) → AA + DNA sequence

---

## Tool 6 — `clustercad_search_domains`

Searches all 531 PKS clusters simultaneously for modules matching specified domain criteria. Uses a locally cached JSON index built on first run for instant subsequent queries. This is the primary tool for finding real natural examples of a module proposed by RetroTide or TridentSynth — given a domain profile, it returns every natural cluster that contains a matching module along with its intermediate SMILES and all domain IDs.

**Inputs:**

- `domain_type` (string, default `""`) — primary domain type to search for, e.g. `"AT"`, `"KR"`, `"DH"`
- `annotation_contains` (string, default `""`) — substrate or activity text to match in annotations, auto-translated from common names (see below)
- `domain_types` (list, default `[]`) — list of domain types that must ALL be present in the module, e.g. `["KR","DH","ER"]`
- `exclude_annotation` (string, default `""`) — exclude domains containing this annotation text, e.g. `"inactive"`
- `active_only` (bool, default `false`) — only return domains not annotated as inactive
- `loading_module_only` (bool, default `false`) — only return loading modules (module 0)
- `cluster_description_contains` (string, default `""`) — filter results by cluster name, e.g. `"erythromycin"`
- `min_modules` (int, default `0`) — only return clusters with at least this many total modules
- `max_modules` (int, default `9999`) — only return clusters with at most this many total modules
- `reviewed_only` (bool, default `false`) — only search manually reviewed clusters
- `max_results` (int, default `10`) — maximum number of results to return
- `force_refresh` (bool, default `false`) — rebuild the local cache from scratch

**Substrate name auto-translation:** ClusterCAD uses abbreviated substrate names internally. `clustercad_search_domains` automatically translates common chemical names so the user and AI never need to know ClusterCAD's internal vocabulary: malonyl-CoA → mal, methylmalonyl-CoA → mmal, ethylmalonyl-CoA → emal, butyryl/butylmalonyl-CoA → butmal, hexylmalonyl-CoA → hxmal, hydroxymalonyl-CoA → hmal, methoxymalonyl-CoA → mxmal, isobutyryl-CoA → isobut, propionyl-CoA → prop, acetyl-CoA → Acetyl-CoA, pyruvate → pyr.

**Output:** A list of matching results each with `accession`, `description`, `subunit_name`, `subunit_id`, `module`, `total_modules`, `product_smiles`, `all_domains`, and `matching_domains`.

**Examples:**
User: Find PKS modules that load butyryl-CoA.
Gemini calls: clustercad_search_domains(domain_type="AT", annotation_contains="butyryl", loading_module_only=true)
→ auto-translates butyryl → butmal
→ instantly searches 531 clusters from local cache
→ returns matching modules with domain IDs and intermediate SMILES

User: Find modules with a fully active reducing loop.
Gemini calls: clustercad_search_domains(domain_types=["KR","DH","ER"], active_only=true, max_results=10)
→ returns modules where all three reductive domains are present and active

---

## Local Cache

`clustercad_search_domains` builds a local cache of all 531 PKS clusters the first time it runs. This takes approximately 70 seconds and downloads every cluster's domain architecture from ClusterCAD. The cache is saved to `modules/pks/tools/clustercad_cache.json` and committed to the repository so all teammates benefit immediately without rebuilding. All subsequent searches are instant. To rebuild the cache if the ClusterCAD database has been updated, call `clustercad_search_domains` with `force_refresh=True`.

---

## Running the tests

```bash
pytest tests/test_clustercad_tools.py -v

To run tests for a single tool:
pytest tests/test_clustercad_tools.py::TestSearchDomains -v

The test suite covers 60+ cases across all six tools including return type and required key validation, correct values for known clusters (Abyssomicin BGC0001492.1), synonym translation (butyryl → butmal), filter correctness (loading_module_only, active_only, min_modules, reviewed_only), error handling for invalid inputs, and string-to-integer coercion for Gemini compatibility.

Individual Contribution Scope
I built all six ClusterCAD tools, their JSON descriptors, and the full test suite. I started by scraping the ClusterCAD web interface to understand its structure, then built the tools incrementally — first list, details, and subunits for basic cluster exploration, then domain and subunit lookup for sequence retrieval, and finally the search tool with local caching and synonym translation as the most novel contribution. I also contributed Section 11 of THEORY.md, prompt test entries in prompts.json, and SKILL.md updates documenting each tool, the substrate vocabulary, and the overall tool-chaining workflow. I tested the full pipeline end-to-end through the Gemini client with queries including "find modules that load butyryl-CoA", "give me the DNA sequence of the AbsB1 subunit in Abyssomicin", and "find all modules with active KR, DH, and ER domains".