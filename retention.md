---


---

<h1 id="feature-breakdown-pii-protection-and-retention-governance">Feature Breakdown: PII Protection and Retention Governance</h1>
<h2 id="feature-1-identify-and-tag-sensitive-columns-in-source-and-curated-layers">Feature 1: Identify and Tag Sensitive Columns in Source and Curated Layers</h2>
<h3 id="description">Description</h3>
<p>Identify, classify, and register sensitive data elements (including PII) across source and curated layers so masking and access controls can be applied consistently.</p>
<h3 id="breakdown">Breakdown</h3>
<ul>
<li>Register PII fields in metadata with consistent sensitivity tags.</li>
<li>Define and document classification taxonomy (PII, SPI, Confidential, Internal, Public).</li>
<li>Assign data owner and steward for each sensitive domain and dataset.</li>
<li>Implement tag propagation and maintenance process as schemas evolve.</li>
</ul>
<h3 id="deliverables">Deliverables</h3>
<ul>
<li>Sensitive data inventory with table/column-level tags.</li>
<li>Metadata registration process for PII and sensitive attributes.</li>
<li>Operating procedure for ongoing tag maintenance.</li>
</ul>
<h3 id="success-criteria">Success Criteria</h3>
<ul>
<li>All in-scope sensitive columns are tagged in metadata.</li>
<li>Ownership and stewardship are assigned for every classified dataset.</li>
<li>Tagging is maintained through schema and pipeline changes.</li>
</ul>
<h2 id="feature-2-implement-masking-and-column-level-access-controls">Feature 2: Implement Masking and Column-Level Access Controls</h2>
<h3 id="description-1">Description</h3>
<p>Protect sensitive attributes using reusable masking rules and role-based column visibility through Unity Catalog policy controls.</p>
<h3 id="breakdown-1">Breakdown</h3>
<ul>
<li>Define reusable masking policies by data classification level.</li>
<li>Map user groups and service principals to masking/access policies.</li>
<li>Apply consistent controls across source and curated layers.</li>
<li>Validate policy behavior with positive/negative access tests.</li>
</ul>
<h3 id="deliverables-1">Deliverables</h3>
<ul>
<li>Standard masking rule set by sensitivity class.</li>
<li>Unity Catalog column-level access policy mappings.</li>
<li>Access test suite for masked/unmasked role scenarios.</li>
</ul>
<h3 id="success-criteria-1">Success Criteria</h3>
<ul>
<li>PII exposure is restricted to approved roles only.</li>
<li>Masking behavior is consistent across all in-scope datasets.</li>
<li>Control outcomes are validated and auditable.</li>
</ul>
<h2 id="feature-3-data-retention-and-purge-capability">Feature 3: Data Retention and Purge Capability</h2>
<h3 id="description-2">Description</h3>
<p>Enable policy-driven data retention management, including tagging retention periods, purging opted-out or expired data, and recording a full audit trail of retention operations.</p>
<h3 id="breakdown-2">Breakdown</h3>
<ul>
<li>Add functionality to tag data assets with retention period and legal basis.</li>
<li>Define retention policies by data domain and regulatory requirement.</li>
<li>Implement purge workflows for opted-out, deleted, or expired records.</li>
<li>Ensure purge logic covers both table storage and derived/curated copies.</li>
<li>Create auditable retention operation logs with actor, timestamp, scope, and outcome.</li>
<li>Add validation controls to verify purge completeness and policy compliance.</li>
</ul>
<h3 id="deliverables-2">Deliverables</h3>
<ul>
<li>Retention policy catalog and tagging standard.</li>
<li>Automated or scheduled purge workflows.</li>
<li>Audit trail schema and retention operation log process.</li>
<li>Validation and reconciliation procedures for purge outcomes.</li>
</ul>
<h3 id="success-criteria-2">Success Criteria</h3>
<ul>
<li>Retention periods are defined and tagged for in-scope datasets.</li>
<li>Opted-out and expired data is removed within policy SLAs.</li>
<li>Every retention/purge operation is traceable through audit logs.</li>
</ul>

