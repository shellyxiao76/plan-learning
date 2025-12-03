# Palantir Foundry Core Concepts: Datasets

## **What is a Foundry Dataset?**

A **dataset** is Foundry's fundamental data abstraction - essentially a versioned, managed wrapper around files in a backing filesystem with rich metadata.

**Key Characteristics:**
- **Multi-format support**: Structured (Parquet/tabular), semi-structured (JSON/XML), unstructured (images, PDFs, videos)
- **Immutable versioning**: Changes create new versions via "transactions" or "builds"
- **Dataset views**: The "latest" pointer to the current state
- **Schema management**: Explicit or inferred schemas with evolution tracking
- **Built-in governance**: Permissions, lineage, and metadata integrated at the dataset level

---

## **Key Differences from Traditional Data Platforms**

| Aspect | Traditional Platforms | Palantir Foundry |
|--------|----------------------|------------------|
| **Data Model** | Tables/files as separate entities | Unified dataset abstraction with metadata |
| **Versioning** | External tools (Git, DVC) or none | Native immutable versioning with full lineage |
| **Schema** | Database-enforced or file-based | Flexible, managed at dataset level across formats |
| **Incremental Updates** | Custom logic in pipelines | Built-in incremental append/update semantics |
| **Metadata** | Separate catalog systems | Tightly coupled with data (permissions, schema, lineage) |
| **Governance** | Bolted-on security layers | Integrated at the dataset primitive level |

**The big shift**: Foundry treats datasets as **versioned, governed artifacts** rather than mutable tables or raw files. Think "Git for data" meets "enterprise data lakehouse."

---

## **Deep Dive Recommendations**

For a senior data engineer, focus on these technical areas:

### **1. Core Foundry Primitives** (Start here)
   - **Datasets**: Versioning model, transaction semantics
   - **Builds**: Foundry's compute abstraction (not just Spark jobs)
   - **Transforms**: Code-first data pipelines
   - **Branches**: Development workflows for data

### **2. Differentiated Capabilities**
   - **Data lineage**: How it's automatically captured end-to-end
   - **Incremental processing**: Native support vs. custom watermarking
   - **Multi-modal compute**: Spark, Python, SQL - how Foundry abstracts compute
   - **Ontology layer**: Semantic layer on top of datasets (unique to Foundry)

### **3. Hands-On Learning Path**
   ```
   1. Create a simple dataset → Understand versioning
   2. Write a Code Repository transform → See how builds work
   3. Test incremental appends → Compare to Delta Lake/Iceberg patterns
   4. Explore lineage graph → Understand metadata propagation
   5. Branch a pipeline → See Git-like data workflows
   ```

### **4. Technical Comparisons to Map Your Knowledge**
   - **Databricks Lakehouse** ↔ Foundry (datasets + ontology + governance)
   - **dbt models** ↔ Foundry transforms (code-first, versioned)
   - **Delta Lake/Iceberg** ↔ Foundry's transaction/versioning layer
   - **Data catalogs (Collibra/Atlan)** ↔ Native Foundry metadata

---

## **Next Steps**

Continue with:
1. **Transforms & Builds** - How data pipelines work
2. **Pipeline scheduling & orchestration** - Job execution model
3. **Integration patterns** - Connecting to external systems
4. **Ontology** - The semantic/business layer (unique differentiator)
