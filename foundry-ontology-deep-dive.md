# Palantir Foundry Ontology: Comprehensive Deep Dive

## Executive Summary

The **Ontology** is Foundry's most distinctive architectural component - it's not just a semantic layer or data catalog, but an **operational, kinetic layer** that creates a "digital twin" of your organization's entities, relationships, and processes. Think of it as a combination of:
- **Knowledge Graph** (entities + relationships)
- **Type-Safe API Layer** (generated SDKs)
- **Business Logic Container** (functions + actions)
- **Access Control System** (granular permissions)
- **Application Framework** (powers Workshop apps)

All sitting on top of your datasets with full lineage, versioning, and governance.

---

## 1. Core Architecture: Dataset → Ontology Mapping

### The Foundational Concept

```
Raw Datasets (Files/Tables)
         ↓
    [Ontology Layer]
         ↓
Objects + Properties + Links (Business View)
         ↓
Applications (Workshop, APIs, Object Explorer)
```

**Key Architectural Principle:**
- Datasets remain the source of truth (immutable, versioned files/Parquet)
- Ontology provides a **semantic projection** on top of datasets
- Each **Object Type** is backed by one or more datasets
- Objects are **rows** in datasets, properties are **columns**, links are **join relationships**

### Example: From Dataset to Object

```
Dataset: employees.parquet
┌─────────┬──────────┬───────────┬─────────────┐
│ emp_id  │ name     │ dept_id   │ start_date  │
├─────────┼──────────┼───────────┼─────────────┤
│ E12345  │ Jane Doe │ D001      │ 2023-01-15  │
└─────────┴──────────┴───────────┴─────────────┘

Ontology Object Type: Employee
- Primary Key: emp_id
- Properties:
  - name (String)
  - startDate (Timestamp) ← mapped from start_date
  - department (Link to Department) ← dept_id becomes link
```

**What Makes This Different:**
- **Not just metadata**: The ontology actively powers applications
- **Bidirectional sync**: Actions can write back to datasets through the ontology
- **Type safety**: Generated SDKs know your exact schema
- **Graph semantics**: Links enable traversal (employee → department → manager)

---

## 2. Core Components Deep Dive

### 2.1 Object Types

**Definition:** An object type represents a class of real-world entities (Employee, Product, Transaction, etc.)

**Technical Characteristics:**
- **Backed by datasets**: Each object type maps to one+ backing datasets
- **Primary key**: Unique identifier field(s) - can be composite
- **Properties**: Typed attributes (primitives, structs, arrays)
- **Versioned**: Object definitions evolve with schema changes
- **Permissioned**: Row-level security based on property values

**Mapping Mechanics:**
```yaml
ObjectType: Flight
Backing Dataset: flights_silver
Primary Key: flight_id
Properties:
  - flight_number: String (from col: flight_num)
  - departure_time: Timestamp (from col: dep_ts)
  - aircraft: Link<Aircraft> (from col: aircraft_reg)
  - status: Enum (from col: status_code)
```

**Unique Features vs Traditional Models:**
- **Multi-format support**: Objects can wrap structured, unstructured, or semi-structured data
- **Computed properties**: Define properties via functions, not just direct column mappings
- **Polymorphism via interfaces** (more below)

---

### 2.2 Properties

**Property Types:**
1. **Primitive**: String, Integer, Double, Boolean, Timestamp, Date, GeoPoint
2. **Structured**: Structs (nested objects), Arrays, Maps
3. **Computed**: Derived from functions (not stored in dataset)
4. **Shared**: Reusable across object types via interfaces

**Advanced Property Features:**
```typescript
// Example: Computed Property
Property: fullAddress (computed)
Implementation: Function that concatenates street, city, state, zip
Benefit: Consistent formatting across applications

// Example: Shared Property (via Interface)
Interface: Auditable
Properties:
  - createdAt: Timestamp
  - modifiedAt: Timestamp
  - createdBy: Link<User>

Employee implements Auditable
Transaction implements Auditable
// Both inherit auditable properties
```

**Key Difference from Traditional Schemas:**
- Properties carry **business semantics**, not just data types
- Metadata includes descriptions, validation rules, lineage
- Changes are **versioned** and **tracked** in ontology history

---

### 2.3 Link Types

**Definition:** Links represent relationships between object types - the "edges" in your knowledge graph.

**Link Characteristics:**
- **Directionality**: Unidirectional or bidirectional
- **Cardinality**: One-to-one, one-to-many, many-to-many
- **Backed by data**: Links are derived from foreign keys or link datasets
- **Traversable**: Enable graph-style queries

**Implementation Patterns:**

**Pattern 1: Foreign Key Link**
```
Dataset: employees (has dept_id column)
Link: Employee.department → Department
Type: Many-to-one
Backing: Foreign key column dept_id
```

**Pattern 2: Link Dataset (Many-to-Many)**
```
Dataset: project_assignments
┌────────────┬────────────┐
│ emp_id     │ project_id │
└────────────┴────────────┘

Link: Employee.projects ↔ Project.assignedEmployees
Type: Many-to-many
Backing: project_assignments link dataset
```

**Graph Traversal Example:**
```typescript
// Get all projects for an employee's department head's reports
employee
  .department
  .manager
  .directReports
  .flatMap(e => e.projects)
```

**Comparison to Traditional Approaches:**
| Traditional | Foundry Links |
|------------|---------------|
| Manual JOINs in SQL | Declarative graph traversal |
| Foreign keys in schemas | Links with metadata + lineage |
| N+1 query problems | Optimized batch loading |
| Static relationships | Versionable, traceable links |

---

### 2.4 Interfaces

**Purpose:** Enable polymorphism and shared behavior across object types.

**Key Capabilities:**
- **Shared properties**: Common attributes across types
- **Shared links**: Relationships that apply to multiple types
- **Flexible queries**: Query across implementing types
- **Extensibility**: Add capabilities without modifying base types

**Example: Interface-Based Design**
```typescript
Interface: Asset
Properties:
  - assetId: String
  - location: GeoPoint
  - lastInspection: Timestamp
Links:
  - assignedTo: User

// Object Types implementing Asset
Vehicle implements Asset
Equipment implements Asset
Facility implements Asset

// Query across all assets
allAssets: Asset[] = ontology.objects
  .where(asset => asset.lastInspection < 90.daysAgo())
  .limit(100)
```

**Use Cases:**
- **Common audit trails** across entities
- **Unified search** across heterogeneous types
- **Shared workflows** (approval, versioning, tagging)
- **Polymorphic collections** in applications

**Comparison:**
- **OOP analogy**: Like interfaces in Java/TypeScript
- **vs dbt**: No equivalent - dbt models are flat tables
- **vs GraphQL**: Similar union types, but with storage backing
- **vs data vault**: Business vault patterns, but semantic-first

---

## 3. Kinetic Elements: Actions & Functions

### 3.1 Action Types

**Definition:** Structured, validated operations that modify objects, properties, and links.

**Key Characteristics:**
- **Parameterized**: Define typed inputs for users/apps
- **Validated**: Rules determine who can execute and when
- **Transactional**: Atomically modify multiple objects/links
- **Audited**: Full history of action executions
- **Side effects**: Trigger notifications, downstream builds, etc.

**Anatomy of an Action:**
```yaml
Action: AssignEmployeeToProject
Parameters:
  - employee: Employee (required)
  - project: Project (required)
  - role: ProjectRole (enum)
  - startDate: Date (default: today)

Validation Rules:
  - User must have "Project Manager" role
  - Project status must be "Active"
  - Employee availability > 0 hours/week

Effects:
  1. Create link: employee → project
  2. Update project.assignedEmployees
  3. Set property: assignment.role = role
  4. Trigger notification to employee's manager
  5. Rebuild downstream: project_capacity_report
```

**Technical Implementation:**
- Actions are **Python/TypeScript functions** under the hood
- Access to full ontology context via SDK
- Can read from and write to multiple object types
- Execute in transactional context (all-or-nothing)

**Comparison to Traditional Patterns:**

| Pattern | Traditional | Foundry Actions |
|---------|------------|-----------------|
| **REST APIs** | Manual endpoint development | Auto-generated from action definition |
| **Stored Procedures** | Database-specific, limited | Language-agnostic, full ontology access |
| **Webhooks** | External orchestration | Native side effects + lineage |
| **Business Rules** | Scattered in app code | Centralized, versioned, auditable |

**Real-World Use Cases:**
- **Data quality**: "Quarantine Suspect Records" action
- **Workflow**: "Approve Purchase Order" with multi-step validation
- **Ops**: "Scale Infrastructure" action triggering cloud APIs
- **Compliance**: "Mark PII for Deletion" with audit trail

---

### 3.2 Functions

**Definition:** Custom server-side logic that reads/computes/transforms ontology data without direct write access.

**Key Characteristics:**
- **Read-optimized**: Query objects, traverse links, aggregate
- **Isolated execution**: Sandboxed, scalable runtime
- **Multi-language**: TypeScript (v1/v2), Python
- **Composable**: Call from actions, Workshop, pipelines, other functions

**Function Use Cases:**

**1. Computed Properties**
```typescript
// Function: calculateEmployeeTenure
Input: Employee
Output: Number (years)
Logic:
  const today = new Date();
  const start = employee.startDate;
  return (today - start) / (365.25 * 24 * 60 * 60 * 1000);

// Used as computed property on Employee
Property: tenure (computed via function)
```

**2. Derived Datasets**
```python
# Function: getHighRiskProjects
Output: Set<Project>
Logic:
  projects = ontology.objects(Project).all()
  return [p for p in projects
          if p.budget > 1_000_000
          and p.percentComplete < 0.5
          and p.daysUntilDeadline < 30]
```

**3. External System Integration**
```typescript
// Function: enrichCustomerWithCRM
Input: Customer
Output: Customer (with CRM data)
Logic:
  const crmData = await fetch(`https://crm.api/customers/${customer.id}`);
  return {
    ...customer,
    lifetimeValue: crmData.ltv,
    healthScore: crmData.score
  };
```

**4. Action Logic**
```typescript
// Function used within Action
Action: ReassignTicket
Logic (uses function):
  const availableAgents = getAvailableAgents(ticket.priority);
  const bestAgent = selectBySkillMatch(availableAgents, ticket.category);
  ticket.assignedTo = bestAgent;
```

**Function vs Action Decision Matrix:**

| Scenario | Use Function | Use Action |
|----------|-------------|-----------|
| Read-only computation | ✓ | |
| Modify objects/links | | ✓ |
| Display in UI | ✓ | |
| User-triggered edit | | ✓ |
| External API call (read) | ✓ | |
| External API call (write) | | ✓ |
| Complex aggregation | ✓ | |
| Multi-step transaction | | ✓ |

---

## 4. Ontology SDK (OSDK)

**Purpose:** Type-safe, language-native SDKs for programmatic ontology access.

**Supported Languages:**
- **TypeScript** (NPM) - Frontend + Backend
- **Python** (Pip/Conda) - Data science + Backend
- **Java** (Maven) - Enterprise integration
- **OpenAPI** - Any language via generated specs

### Key Features

**1. Type Safety**
```typescript
// Generated types from your ontology
import { Employee, Project, Department } from '@my-org/ontology-sdk';

const employee: Employee = await client.objects.Employee.get('E12345');
// TypeScript knows:
// - employee.name is a string
// - employee.department is a Department
// - employee.projects is a Set<Project>

// Compile-time errors:
employee.foo; // ERROR: Property 'foo' does not exist on type 'Employee'
employee.department.totalRevenue; // ERROR: Department has no 'totalRevenue'
```

**2. Graph Traversal**
```typescript
// Fluent API for link traversal
const departmentProjects = await employee.department.projects.fetch();

// Batch loading (no N+1 queries)
const allManagerNames = await employees
  .map(e => e.manager)
  .fetchPage()
  .then(managers => managers.map(m => m.name));
```

**3. Action Execution**
```python
from my_ontology import actions, Employee, Project

# Execute action with parameters
result = actions.AssignEmployeeToProject.apply(
    employee=Employee.get("E12345"),
    project=Project.get("PRJ-2024-001"),
    role="Lead Engineer",
    startDate=date.today()
)

# SDK handles:
# - Validation (user permissions, business rules)
# - Transaction semantics
# - Error handling
# - Audit logging
```

**4. Querying**
```typescript
// Type-safe queries
const recentHires = await client.objects.Employee
  .where(e => e.startDate > new Date('2024-01-01'))
  .and(e => e.department.name === 'Engineering')
  .orderBy(e => e.startDate, 'desc')
  .fetchPage();

// Aggregations
const deptCounts = await client.objects.Employee
  .groupBy(e => e.department)
  .aggregate({ count: e => e.count() });
```

**Architectural Benefits:**
- **No ORM impedance mismatch**: SDK reflects ontology 1:1
- **Centralized schema**: Change ontology, regenerate SDKs
- **Governance enforcement**: Permissions enforced at SDK level
- **Developer velocity**: Autocomplete, type checking, refactoring

---

## 5. How Ontology Differs from Traditional Approaches

### 5.1 vs Semantic Layers (Looker, dbt Semantic Layer, Cube)

| Aspect | Traditional Semantic Layer | Foundry Ontology |
|--------|---------------------------|------------------|
| **Primary Use Case** | BI/Analytics (read-only) | Operational + Analytical (read-write) |
| **Data Model** | Metrics, dimensions, measures | Objects, properties, links, actions |
| **Relationships** | Star/snowflake schemas | Knowledge graph (arbitrary traversal) |
| **Write Operations** | None (read-only) | Actions with validation + transactions |
| **Type Safety** | SQL-level (limited) | Full SDK generation (compile-time) |
| **Business Logic** | Calculated fields | Functions + Actions (full programming) |
| **Application Layer** | Separate (BI tools) | Integrated (Workshop, APIs) |
| **Versioning** | Schema migrations | Full ontology versioning + lineage |

**Key Distinction:** Traditional semantic layers are **passive metadata** for queries. Foundry's ontology is an **active operational layer** that powers applications.

---

### 5.2 vs Knowledge Graphs (Neo4j, GraphQL)

| Aspect | Neo4j / GraphQL | Foundry Ontology |
|--------|-----------------|------------------|
| **Storage** | Native graph DB | Files (Parquet, etc.) + graph projection |
| **Query Language** | Cypher / GraphQL | SDK methods (type-safe) |
| **Schema** | Flexible/schema-less | Strongly typed, versioned |
| **Data Lineage** | External tooling | Native, automatic |
| **Permissions** | Role-based (coarse) | Property-level + computed |
| **Integration** | APIs | Native dataset backing + pipelines |
| **Versioning** | Manual | Built-in (inherited from datasets) |
| **Business Logic** | Stored procedures | Functions + Actions (versioned, tested) |

**Key Distinction:** Traditional graphs separate storage from analytics. Foundry's ontology is **projection-based** - datasets remain source of truth, ontology provides semantic view.

---

### 5.3 vs Data Catalogs (Collibra, Alation, Atlan)

| Aspect | Data Catalogs | Foundry Ontology |
|--------|--------------|------------------|
| **Purpose** | Discovery + Documentation | Operational + Governance |
| **Metadata** | Passive (tags, descriptions) | Active (powers applications) |
| **Lineage** | Post-hoc discovery | Real-time, enforced |
| **Access Control** | Separate from data | Integrated at object/property level |
| **APIs** | Read metadata | Read + Write objects |
| **Business Logic** | None | Functions, Actions |
| **Application Dev** | Not supported | Native (Workshop, SDKs) |

**Key Distinction:** Catalogs describe data. Ontology **operationalizes** data for applications.

---

### 5.4 vs ORMs (Hibernate, Django ORM, TypeORM)

| Aspect | Traditional ORMs | Foundry Ontology |
|--------|-----------------|------------------|
| **Scope** | Single app's data layer | Enterprise-wide semantic layer |
| **Schema Source** | Database | Datasets (versioned, governed) |
| **Generated Code** | Per-app models | Centralized SDK (reused) |
| **Relationships** | Foreign keys | Links (with lineage + metadata) |
| **Validation** | App-level | Ontology-level (enforced everywhere) |
| **Versioning** | DB migrations | Ontology versions + dataset versions |
| **Multi-App** | Schema drift risk | Single source of truth |

**Key Distinction:** ORMs map DB → App. Ontology maps Datasets → Enterprise semantic layer → All apps.

---

## 6. Technical Deep Dive: How It Works Under the Hood

### Data Flow: Read Path

```
User Request (Workshop App / API)
         ↓
  OSDK Query (type-safe)
         ↓
  Ontology Service
         ↓
  Permission Check (object + property level)
         ↓
  Dataset Resolution (which backing dataset?)
         ↓
  Spark Query (optimized pushdown)
         ↓
  Parquet Files (versioned datasets)
         ↓
  Object Materialization (rows → objects)
         ↓
  Response (typed objects)
```

**Optimizations:**
- **Projection pushdown**: Only read required properties
- **Predicate pushdown**: Filter in Spark, not in memory
- **Batch link resolution**: Avoid N+1 queries
- **Caching**: Ontology metadata + frequently accessed objects

---

### Data Flow: Write Path (Action Execution)

```
User Action Trigger (Workshop / API)
         ↓
  Action Validation
    - Permission check (user role)
    - Business rule validation (custom logic)
    - Input parameter validation (types, constraints)
         ↓
  Action Function Execution
    - Read current object state(s)
    - Execute custom logic (function calls, external APIs)
    - Compute new state
         ↓
  Transaction Begin
    - Lock object(s)
    - Generate dataset transaction
         ↓
  Dataset Write
    - Append/update backing dataset(s)
    - Create new dataset version
         ↓
  Ontology Update
    - Update object materialization
    - Propagate link changes
    - Trigger downstream builds (if configured)
         ↓
  Audit Log
    - Record action execution
    - Capture lineage (who, what, when, why)
         ↓
  Transaction Commit
         ↓
  Side Effects (optional)
    - Notifications
    - External webhooks
    - Pipeline triggers
```

**Key Properties:**
- **ACID semantics**: Actions are transactional
- **Versioned writes**: Every change creates new dataset version
- **Lineage capture**: Automatic tracking of data provenance
- **Rollback capability**: Revert to previous ontology/dataset versions

---

## 7. Architectural Implications for Data Engineers

### 7.1 Design Patterns

**Pattern 1: Ontology-First Modeling**
```
Traditional: Start with tables/schemas
Foundry: Start with business entities (objects)

Example:
Instead of: customers, orders, order_items tables
Think: Customer object, Order object, OrderItem object
Then: Map to datasets (can be same schemas, but semantics-first)
```

**Pattern 2: Semantic Joins via Links**
```
Traditional: Manual JOINs in every query
Foundry: Define links once, traverse anywhere

# Bad (traditional)
SELECT o.*, c.name
FROM orders o
JOIN customers c ON o.customer_id = c.id

# Good (ontology)
order.customer.name  // Link traversal
```

**Pattern 3: Centralized Business Logic**
```
Traditional: Business rules in app code (duplicated, drift)
Foundry: Functions + Actions in ontology (single source of truth)

Example: "Valid order" logic
- Traditional: Copied across microservices, dashboards, APIs
- Foundry: Function isValidOrder(order) called everywhere
```

**Pattern 4: Polymorphic Queries via Interfaces**
```
Traditional: Union queries across multiple tables
Foundry: Query interface, get all implementing types

# Find all "expiring" items
expirables = ontology.objects(Expirable)  // Interface
  .where(e => e.expirationDate < 30.daysFromNow())

# Returns: Contracts, Licenses, Certifications, etc.
```

---

### 7.2 Migration Strategies

**Scenario: Existing Data Platform → Foundry**

**Phase 1: Dataset Migration (Foundry as Data Lake)**
- Lift-and-shift existing tables → Foundry datasets
- No ontology yet (just versioned Parquet/files)
- Benefit: Lineage, versioning, governance on raw data

**Phase 2: Ontology Bootstrapping (Semantic Layer)**
- Create object types for core entities
- Map properties to existing dataset columns
- Define key links (foreign key relationships)
- Benefit: Type-safe APIs, graph traversal

**Phase 3: Function Enrichment (Business Logic)**
- Migrate calculated fields → Functions
- Add computed properties
- Implement complex aggregations
- Benefit: Consistent metrics across tools

**Phase 4: Action Implementation (Write Operations)**
- Replace app write logic with Actions
- Centralize validation, business rules
- Enable low-code app development (Workshop)
- Benefit: Governance, auditability, velocity

**Phase 5: Interface Abstraction (Advanced Modeling)**
- Identify common patterns → Interfaces
- Enable polymorphic workflows
- Retire legacy abstractions
- Benefit: Flexibility, extensibility

---

### 7.3 When to Use Ontology vs Raw Datasets

| Use Case | Use Raw Datasets | Use Ontology Layer |
|----------|-----------------|-------------------|
| **Ad-hoc exploration** | ✓ (SQL/Spark) | |
| **ML feature engineering** | ✓ (Python/Spark) | |
| **Production pipelines** | ✓ (Code Repos) | |
| **Business user apps** | | ✓ (Workshop) |
| **External APIs** | | ✓ (OSDK) |
| **Complex relationships** | | ✓ (Links) |
| **Write operations** | | ✓ (Actions) |
| **Governance enforcement** | | ✓ (Permissions) |
| **Cross-team consistency** | | ✓ (Single ontology) |

**Rule of Thumb:**
- **Analytics workloads**: Start with datasets, use ontology for final delivery layer
- **Application workloads**: Always use ontology (type safety, governance)
- **Data engineering**: Datasets for transforms, ontology for orchestration/metadata

---

## 8. Advanced Topics

### 8.1 Ontology Versioning & Branching

**Key Concept:** Ontologies can be versioned and branched like code/data.

**Use Cases:**
- **Development workflow**: Branch ontology, test schema changes, merge
- **A/B testing**: Multiple ontology versions for experimentation
- **Rollback**: Revert to previous ontology version if issues arise
- **Audit**: Track who changed what in ontology structure

**Example Workflow:**
```
main (production ontology)
  ↓
feature/add-supplier-object (dev branch)
  - Add Supplier object type
  - Add links: Product → Supplier
  - Test in dev Workshop apps
  - Merge to main after validation
```

---

### 8.2 Ontology Performance Optimization

**1. Dataset Partitioning**
```
Object Type: Transaction (millions of rows)
Backing Dataset: transactions.parquet (partitioned by date)
Benefit: Queries filter by date → only read relevant partitions
```

**2. Link Dataset Indexing**
```
Link: User.transactions (1:many, user has 1000s of transactions)
Optimization: Ensure link dataset has index on user_id
```

**3. Computed Property Caching**
```
Function: calculateCustomerLifetimeValue (expensive)
Strategy: Materialize as property, update daily via pipeline
Tradeoff: Staleness vs performance
```

**4. Selective Property Loading**
```typescript
// Bad: Load all properties
const employees = await client.objects.Employee.fetchPage();

// Good: Project only needed properties
const employees = await client.objects.Employee
  .select(['name', 'department'])
  .fetchPage();
```

---

### 8.3 Security Model

**Layered Permissions:**
1. **Dataset-level**: Who can read/write backing datasets
2. **Object-level**: Row-level security based on property values
3. **Property-level**: Hide/mask specific properties per user
4. **Action-level**: Who can execute which actions

**Example: Row-Level Security**
```
Object Type: PatientRecord
Rule: User can only see records where
  - user.role == "Doctor" AND
  - patientRecord.assignedDoctor == user
OR
  - user.role == "Admin"

Implementation: Automatic filter in all queries
```

---

### 8.4 Ontology + ML Integration

**Pattern:** Use ontology for feature engineering + model serving

```python
# Feature Engineering (via functions)
def generate_customer_features(customer: Customer):
    return {
        'tenure_days': customer.tenure,
        'total_spend': sum(t.amount for t in customer.transactions),
        'avg_transaction': mean(t.amount for t in customer.transactions),
        'days_since_last_purchase': (today - customer.lastPurchaseDate).days,
        'product_diversity': len(set(t.product for t in customer.transactions))
    }

# Model Prediction (as ontology function)
Function: predictChurnRisk
Input: Customer
Output: Number (0-1 risk score)
Logic:
  features = generate_customer_features(customer)
  return ml_model.predict(features)

# Use in applications
high_risk_customers = ontology.objects(Customer)
  .where(c => c.churnRisk > 0.7)
  .fetchPage()
```

---

## 9. Learning Roadmap for Data Engineers

### Phase 1: Foundations (Week 1-2)
1. **Read**: Ontology overview docs
2. **Hands-on**: Create simple object type (e.g., Employee) backed by CSV
3. **Experiment**: Add properties, links to related object (Department)
4. **Explore**: Use Object Explorer to browse your ontology

**Key Learning:** Understand dataset → object mapping

---

### Phase 2: Graph & Queries (Week 2-3)
1. **Hands-on**: Create complex link structures (many-to-many)
2. **Code**: Use OSDK (TypeScript or Python) to query objects
3. **Practice**: Write graph traversal queries
4. **Optimize**: Profile query performance, understand pushdown

**Key Learning:** Master link mechanics and SDK usage

---

### Phase 3: Business Logic (Week 3-4)
1. **Build**: Write your first function (computed property)
2. **Build**: Implement an action (simple write operation)
3. **Test**: Unit test functions, validate action rules
4. **Deploy**: Use function in Workshop app or API

**Key Learning:** Centralize logic in ontology layer

---

### Phase 4: Advanced Patterns (Week 4-6)
1. **Design**: Model polymorphic entities with interfaces
2. **Integrate**: Call external APIs from functions
3. **Optimize**: Materialize expensive computed properties
4. **Govern**: Implement row-level security rules

**Key Learning:** Enterprise-scale ontology design

---

### Phase 5: Production Deployment (Week 6+)
1. **Architecture**: Design ontology for your actual use case
2. **Migrate**: Map existing data models to ontology
3. **Build**: Create Workshop apps powered by ontology
4. **Monitor**: Track ontology usage, performance, lineage
5. **Evolve**: Manage schema changes via branching/versioning

**Key Learning:** Operate ontology at scale

---

## 10. Key Takeaways for Senior Data Engineers

### What Makes Ontology Unique
1. **Operational, not just analytical**: Powers apps, not just reports
2. **Write + Read**: Actions enable governed data modifications
3. **Type-safe**: Generated SDKs catch errors at compile time
4. **Graph-native**: Links are first-class, not afterthoughts
5. **Versioned + Governed**: Changes tracked, auditable, reversible

### Paradigm Shifts
- **From tables to objects**: Think entities, not schemas
- **From ETL to semantic modeling**: Business logic in ontology, not pipelines
- **From app-specific APIs to universal SDK**: One ontology, many apps
- **From data catalog to operational layer**: Metadata that does work

### Architecture Decision Factors
**Use Ontology When:**
- Building applications (Workshop, external apps via SDK)
- Need type-safe APIs
- Complex relationships (graph traversal)
- Governed write operations
- Cross-team data sharing

**Use Raw Datasets When:**
- Heavy analytics (large scans, complex aggregations)
- ML feature engineering (flexibility over governance)
- One-off exploration (ad-hoc queries)
- Performance critical (avoid ontology abstraction overhead)

### Mental Models
- **Ontology = Your org's data API**
- **Objects = Rows with semantics**
- **Links = Smart foreign keys**
- **Actions = Governed stored procedures**
- **Functions = Computed columns with code**
- **Interfaces = Polymorphic type system**

---

## 11. Comparison Matrix: Foundry vs Other Platforms

| Capability | Databricks | Snowflake | AWS (Glue+Lake Formation) | Palantir Foundry |
|------------|-----------|-----------|---------------------------|------------------|
| **Semantic Layer** | dbt integration | Semantic layer (beta) | External (dbt, Looker) | Native Ontology |
| **Type-Safe APIs** | No | No | No | Yes (OSDK) |
| **Graph Traversal** | No | No | No | Yes (Links) |
| **Write Operations** | SQL/Spark | SQL | SQL | Actions (governed) |
| **Business Logic** | dbt macros/models | UDFs | Lambda/Glue jobs | Functions + Actions |
| **Application Dev** | External | External | External | Native (Workshop) |
| **Versioning** | Delta Lake | Time Travel | S3 versions | Datasets + Ontology |
| **Lineage** | Unity Catalog | Limited | Glue Data Catalog | Native, real-time |
| **Permissions** | Table/column | Row-access policies | Lake Formation | Object/property level |

**Foundry's Differentiation:** Only platform with **semantic layer that powers both analytics AND applications** with full governance.

---

## Next Steps

To continue your deep dive:

1. **Hands-On**: Request Foundry sandbox, build a simple ontology
2. **Documentation**: Read action types, functions docs in detail
3. **Case Studies**: Look for Foundry ontology examples in your domain
4. **Architecture**: Sketch how your current data model maps to ontology
5. **Comparison**: Identify gaps in your current platform vs ontology capabilities

Would you like me to:
- Deep dive into **Workshop** (how apps consume ontology)?
- Explore **Transforms & Pipelines** (how data flows into ontology)?
- Detail **integration patterns** (external systems ↔ ontology)?
- Cover **governance** (permissions, compliance, audit)?
