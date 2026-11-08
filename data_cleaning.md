---


---

<h1 id="datacleaning.r-—-migration-analysis-r-→-pyspark">DataCleaning.R — Migration Analysis (R → PySpark)</h1>
<p>Analysis only. No PySpark code is included and the R script was not modified.</p>
<hr>
<h2 id="scope-sources-and-conventions">0. Scope, sources and conventions</h2>
<h3 id="sources-analysed">0.1 Sources analysed</h3>

<table>
<thead>
<tr>
<th>Source</th>
<th>Location</th>
<th>Status</th>
</tr>
</thead>
<tbody>
<tr>
<td><code>DataCleaning.R</code> (2,179 lines)</td>
<td>workspace root</td>
<td>Read in full. Byte-identical to <code>../GRI/modules/DataCleaning.R</code> (verified with <code>cmp</code>).</td>
</tr>
<tr>
<td><code>docs/DataCleaningRScriptWiki.md</code></td>
<td>workspace</td>
<td>Read in full (wiki “Step 5”).</td>
</tr>
<tr>
<td><code>Standards.R</code> (sourced at L7 via <code>paste(core, "Standards.R")</code>)</td>
<td><code>../GRI/core/Standards.R</code></td>
<td><strong>Not in the workspace.</strong> Found outside it and read. Byte-identical to <code>../data_cleaning_scripts/Standards.R</code>. I assume <code>core</code> resolves to <code>GRI/core/</code> (<code>GRI/config/global_config.R:87</code>).</td>
</tr>
<tr>
<td><code>f_deductibles_sublimits.R</code> (sourced at L930 and L1508 via <code>functions</code>)</td>
<td><code>../GRI/functions/f_deductibles_sublimits.R</code></td>
<td><strong>Not in the workspace.</strong> Found outside it and read. A different, older version sits in <code>GRI/functions/archive/</code>; I assume the non-archive file is the active one (<code>global_config.R:83</code>).</td>
</tr>
<tr>
<td><code>Mode_MM()</code> (used but not defined in the script)</td>
<td><code>../GRI/functions/Mode_MM.R</code></td>
<td>Read. It is sourced by the caller scripts, for example <code>GRI/scripts/market_units/MUCH/01_MUCH_EQ.R:66</code>.</td>
</tr>
<tr>
<td><code>Mapping_PostalCode</code>, <code>Mapping_CRESTA_CH</code> (used, not loaded here)</td>
<td>Loaded in <code>../GRI/functions/f_mapping_tables.R:6,8</code> from <code>PostalCode.xlsx</code> / <code>CRESTACode.xlsx</code> (sheet <code>Schweiz</code>)</td>
<td>The xlsx files were <strong>not available</strong>. The CSV copies in <code>mappers/postalCode.csv</code> and <code>mappers/crestaCode_CH.csv</code> were inspected. I could not confirm that they match the xlsx.</td>
</tr>
<tr>
<td><code>exposure</code> (input data frame)</td>
<td>Built by the calling MU script</td>
<td>No sample data was available. Column names and types are inferred from the code.</td>
</tr>
<tr>
<td>Wiki questions Q4, Q5, Q6, Q7</td>
<td>Referenced in the wiki</td>
<td><strong>Missing.</strong> They are not defined in the wiki or in <code>docs/zold_DataCleaningWikiInfo.md</code>.</td>
</tr>
<tr>
<td><code>_temp2.RData</code> checkpoint</td>
<td>Mentioned in the wiki</td>
<td>It is <strong>not written by <code>DataCleaning.R</code></strong>. The caller writes it, for example <code>01_MUCH_EQ.R:557</code> and <code>01_BABE_Prop.R:735</code>.</td>
</tr>
<tr>
<td>Related analyses</td>
<td><code>docs/CRESTA_Workflow.md</code>, <code>docs/CountryCode_Workflow.md</code>, <code>docs/PostalCode_Workflow.md</code>, <code>docs/Geo_Standardization_*.md</code></td>
<td>Skimmed. They agree with this analysis for the geo block (for example, the stale <code>CountryCode_ISO3</code> issue and the Milford conflict).</td>
</tr>
</tbody>
</table><p>R is not installed on this machine, so no R behaviour could be run. Findings that depend on subtle R or dplyr evaluation rules are marked <strong>verify in R</strong>.</p>
<h3 id="notation">0.2 Notation</h3>
<ul>
<li><code>L123</code> means <code>DataCleaning.R</code> line 123. <code>FDS:L45</code> means <code>f_deductibles_sublimits.R</code> line 45. <code>STD:L10</code> means <code>Standards.R</code> line 10.</li>
<li><strong>[FACT]</strong> is read directly from the code or files.</li>
<li><strong>[WIKI]</strong> is stated in the wiki.</li>
<li><strong>[INFERRED]</strong> is my reading of intent or behaviour from the code. It is not documented.</li>
<li><strong>[ASSUMPTION]</strong> is a precondition the code relies on but does not check.</li>
<li><strong>[RISK]</strong> is probable unintended behaviour or an edge case. A business owner must confirm whether to keep it or fix it.</li>
<li><strong>[REC]</strong> is a recommendation.</li>
</ul>
<h3 id="key-findings-summary">0.3 Key findings (summary)</h3>
<ol>
<li><strong>[RISK] Policy-level harmonisation (wiki 5.8) never runs for most MUs.</strong> Both entry conditions (L918–927, L1496–1504) need <code>LOB_list[[MU]]$LOB != ""</code> unless the run is MUFR Prop CATNAT. Only MUCH and MUDE have a non-empty LOB (<code>STD:L57–68</code>). For BABE, BACH, BADE, BALU, MUAT, MUIT, MUES and non-CATNAT MUFR, all of 5.8 is skipped, including the “re-apply 5.6/5.7” step.</li>
<li><strong>[RISK] The “re-apply rules” step (L1980–2064) only runs inside the second harmonisation branch</strong> (per policy+country, or CATNAT). It never runs after the per-policy branch alone. It also re-applies only part of the rules.</li>
<li><strong>[RISK] The AAL sublimit rule uses a portfolio-wide maximum.</strong> <code>max(cbind(...), na.rm=T)</code> at L844/L913 returns one scalar over the whole dataset, not the maximum of the row.</li>
<li><strong>[RISK] The FR→MC rule (L258, postal code starts “98”) runs before the overseas split (L303–306).</strong> As a result, the FR→TF/WF/PF/NC rules (984/986/987/988) can never fire.</li>
<li><strong>[RISK] The stale temporary join.</strong> <code>CountryCode_ISO3</code> is joined at L36 using the country <em>before</em> corrections. It is still used for the CRESTA prefix at L287/L289. Any country changed between L97 and L272 gets a wrong or <code>NA_</code> prefix. For MUFR this affects every row that arrived with a French country name.</li>
<li><strong>[RISK] Silent conversion of numeric NA to 0 in single-location policies.</strong> Every numeric column in every row with <code>n == 1</code> (L710–750) goes from NA to 0. Examples are Latitude, Longitude and SI. This runs whenever <code>cat_peril</code> is non-empty. It also happens for the whole dataset after either harmonisation branch (L1493, L1969).</li>
<li><strong>[RISK] Several statements are order-dependent or have size quirks.</strong> Examples: <code>na.omit</code> on a logical mask (L73), conditions computed before the swaps they guard (postal loop L379–435), <code>ifelse(scalar, …, vector)</code> returning only the first element (L735, and throughout L1066–1121), and a global <code>fix[[j]]</code> flag used across rows (L704–738).</li>
<li><strong>[FACT] <code>DataCleaning.R</code> is also used in part.</strong> <code>01_BABE_Prop.R:728–733</code> sources only the lines between <code>##Start Geo-Corrections</code> (L12) and <code>##Stop Geo-Corrections</code> (L493). The geo block must stay callable on its own.</li>
</ol>
<hr>
<h2 id="purpose-and-outputs">1. Purpose and outputs</h2>
<h3 id="purpose-fact--wiki">1.1 Purpose [FACT + WIKI]</h3>
<p>This is a generic cleaning step (“Step 5”) shared by all Market Units (MUs), with some MU-specific branches. It runs on a location-level exposure table (one row per insured location). It does the following:</p>
<ul>
<li>Standardises the geo fields Country, CRESTA, PostalCode, City, Street and StreetNo to improve geocoding.</li>
<li>Normalises percentages.</li>
<li>Makes deductibles and sublimits consistent per location, then (conditionally) per policy.</li>
<li>Fixes YearBuilt and negative sums insured.</li>
<li>Logs every changed value.</li>
<li>Removes rows that should not be modelled: 100% Fac (2023 only), total SI = 0, and sanctioned countries.</li>
</ul>
<h3 id="inputs-fact--inferred">1.2 Inputs [FACT / INFERRED]</h3>
<p><strong>Global variables the script expects:</strong> <code>exposure</code>, <code>MU</code>, <code>sub</code>, <code>modeling_option</code> (optional), <code>year</code>, <code>perils</code>, <code>core</code>, <code>functions</code>, <code>output</code>, <code>Mapping_PostalCode</code>, <code>Mapping_CRESTA_CH</code>, <code>Mode_MM()</code>. Required packages: tidyverse/stringr, lubridate (<code>year()</code> in <code>STD:L3</code>), DescTools, foreach, doSNOW, parallel.</p>
<p><strong>Globals defined by <code>Standards.R</code> (re-sourced at L7):</strong></p>
<ul>
<li><code>c_year</code> comes from <code>year</code>. If <code>year == ""</code>, it falls back to the <strong>system year at run time</strong> (<code>STD:L2–6</code>).</li>
<li><code>cat_ded = c("","Com","B","C","BI")</code>.</li>
<li><code>cat_peril</code> comes from <code>perils</code>, for example <code>"PES+PFL+PWH+PWT"</code>. Codes are sorted, then mapped PES→EQ, PFL→FL, PWH/PWW/PWA→WS, PWT→HA.</li>
<li>Also: <code>LOB_list</code>, <code>Country_list</code>, <code>geo_cols = c("Country","CRESTA","PostalCode","City","Street","StreetNo")</code>, <code>var_string</code>, <code>coord_string</code>, <code>values_string</code>, <code>sanctioned_string</code>.</li>
<li><code>empty_columns</code> is computed on the <strong>input</strong> <code>exposure</code> (<code>STD:L84–86</code>).</li>
</ul>
<p><strong>Columns <code>exposure</code> must have (inferred from code):</strong></p>
<ul>
<li>Identification: <code>PolicyID</code>, <code>LOB</code>, <code>Currency</code>, <code>Country</code>, <code>CRESTA</code>, <code>PostalCode</code>, <code>City</code>, <code>Street</code>, <code>StreetNo</code>, <code>YearBuilt</code>.</li>
<li>Sums insured: <code>SI_Building</code>, <code>SI_Content</code>, <code>SI_Other</code>, <code>SI_BI</code>, used positionally as the range <code>SI_Building:SI_BI</code> in FDS. Any other <code>SI_*</code> columns also take part in the abs() and TSI rules.</li>
<li>Reinsurance shares: <code>Fac</code>, <code>Sux</code>, <code>*_Percent</code>.</li>
<li>Per peril P in <code>cat_peril</code> and coverage c in <code>{"", Com, B, C, BI}</code>:
<ul>
<li><code>Kind_of_P_[c_]Deductible</code></li>
<li><code>P_Deductible_[c_]Fix</code>, <code>P_Deductible_[c_]Percent</code>, <code>P_Deductible_[c_]Percent_Min</code>, <code>P_Deductible_[c_]Percent_Max</code></li>
<li><code>P_Deductible_per</code></li>
<li><code>P_Sublimit[_c]</code>, <code>P_Sublimit_per</code></li>
<li><code>P_Cover</code> (logical), <code>P_AAL</code> (logical)</li>
</ul>
</li>
<li>The harmonisation gates refer to <code>EQ_</code>, <code>WS_</code>, <code>FL_</code> and <code>HA_</code> <code>Deductible_per</code>/<code>Sublimit_per</code> <strong>explicitly</strong>, whatever <code>cat_peril</code> contains.</li>
<li><code>CATNAT_Label</code> is optional and only used for MUFR/Prop/CATNAT.</li>
</ul>
<p>Column names are built with <code>gsub("__","_",…)</code>. For the combined coverage (<code>""</code>), the names are <code>EQ_Deductible_Fix</code>, <code>Kind_of_EQ_Deductible</code>, <code>EQ_Sublimit</code>, <code>SI</code>. Index 1 is combined, 2 is Com, 3 is B, 4 is C and 5 is BI. Every rule below that says “coverage 1…5” uses this order.</p>
<h3 id="outputs-fact">1.3 Outputs [FACT]</h3>

<table>
<thead>
<tr>
<th>Output</th>
<th>Where</th>
<th>Notes</th>
</tr>
</thead>
<tbody>
<tr>
<td>Cleaned <code>exposure</code> (global env)</td>
<td>in memory</td>
<td>Adds <code>seq_ID</code>, <code>n</code> (locations per PolicyID, computed before removals) and <code>group_id</code>. Temporary mapping columns are removed (L2086–2088).</td>
</tr>
<tr>
<td><code>change_&lt;col&gt;.csv</code></td>
<td><code>output/Results/</code></td>
<td>One file per non-empty column that has at least one change versus the L39 snapshot. Encoding is ISO-8859-1 and <code>write.csv</code> includes row names.</td>
</tr>
<tr>
<td><code>Removed_100Fac.csv</code></td>
<td><code>output/Results/</code></td>
<td>Only when <code>c_year == "2023"</code>.</td>
</tr>
<tr>
<td><code>Removed_TSI0.csv</code></td>
<td><code>output/Results/</code></td>
<td>Rows where the sum of all <code>SI_*</code> is 0.</td>
</tr>
<tr>
<td><code>Removed_Sanctioned.csv</code></td>
<td><code>output/Results/</code></td>
<td>Rows where Country is in <code>sanctioned_string</code>.</td>
</tr>
<tr>
<td>Console logs and <code>warning()</code>s</td>
<td>stdout</td>
<td>Change counts, policy lists, policy-level deductibles or sublimits kept although SI = 0.</td>
</tr>
<tr>
<td><code>_temp2.RData</code></td>
<td><code>output/RData/</code></td>
<td><strong>Written by the caller, not by this script.</strong></td>
</tr>
</tbody>
</table><hr>
<h2 id="execution-flow-actual-order">2. Execution flow (actual order)</h2>
<p>The script runs top to bottom. Helper functions are defined inline right before use. The only control flow that changes what runs is the MU branch (L97), the CH CRESTA branch (L280), the per-country loop (L363), the two harmonisation branches (L918, L1496), the TSI/sanction branches, and the 2023 Fac branch (L2117).</p>

<table>
<thead>
<tr>
<th>#</th>
<th>Step</th>
<th>Lines</th>
<th>Wiki §</th>
</tr>
</thead>
<tbody>
<tr>
<td>S0</td>
<td>Re-source <code>Standards.R</code></td>
<td>6–8</td>
<td>–</td>
</tr>
<tr>
<td>S1</td>
<td>Default Country/Currency</td>
<td>14–32</td>
<td>5.1</td>
</tr>
<tr>
<td>S2</td>
<td>Temporary join #1 with Mapping_PostalCode, plus snapshot <code>exposure_org</code></td>
<td>34–40</td>
<td>–</td>
</tr>
<tr>
<td>S3</td>
<td>Remove junk values from geo columns</td>
<td>42–70</td>
<td>5.1</td>
</tr>
<tr>
<td>S4</td>
<td>MUAT “Migration” placeholder → NA</td>
<td>72–73</td>
<td>5.1</td>
</tr>
<tr>
<td>S5</td>
<td>Cast PostalCode to character</td>
<td>79–82</td>
<td>–</td>
</tr>
<tr>
<td>S6</td>
<td>Country corrections (MUFR names, aliases, city rules, special cases, CH→LI)</td>
<td>96–272</td>
<td>5.2</td>
</tr>
<tr>
<td>S7</td>
<td>CRESTA corrections</td>
<td>274–294, 340</td>
<td>5.3</td>
</tr>
<tr>
<td>S8</td>
<td>FR overseas split by postal code (<strong>after</strong> CRESTA)</td>
<td>297–308</td>
<td>5.2</td>
</tr>
<tr>
<td>S9</td>
<td>Temporary join #2 (refresh)</td>
<td>342–345</td>
<td>–</td>
</tr>
<tr>
<td>S10</td>
<td>Postal-code pre-rules</td>
<td>350–355</td>
<td>5.4</td>
</tr>
<tr>
<td>S11</td>
<td>Per-country postal / address reorganisation loop</td>
<td>357–452</td>
<td>5.4</td>
</tr>
<tr>
<td>S12</td>
<td>Global postal post-processing (strip non-alphanumerics, GB/CA O/I fixes)</td>
<td>454–465</td>
<td>5.4</td>
</tr>
<tr>
<td>S13</td>
<td>Street and StreetNo rules</td>
<td>467–479</td>
<td>5.4</td>
</tr>
<tr>
<td>S14</td>
<td>Trim geo columns (end of geo block, L493)</td>
<td>482–491</td>
<td>5.4</td>
</tr>
<tr>
<td>S15</td>
<td>Percentage normalisation</td>
<td>497–511</td>
<td>5.5</td>
</tr>
<tr>
<td>S16</td>
<td>Add <code>seq_ID</code>, <code>n</code>, <code>group_id</code></td>
<td>513–526</td>
<td>5.9 (code runs it earlier)</td>
</tr>
<tr>
<td>S17</td>
<td>Location-level deductible rules A → B → C</td>
<td>528–686</td>
<td>5.6</td>
</tr>
<tr>
<td>S18</td>
<td>Single-location mixed Fix / % of Claims</td>
<td>688–784</td>
<td>5.6</td>
</tr>
<tr>
<td>S19</td>
<td>Swap min and max when min &gt; max</td>
<td>786–807</td>
<td><strong>not in wiki</strong></td>
</tr>
<tr>
<td>S20</td>
<td>Location-level sublimit rules F → G → H</td>
<td>809–915</td>
<td>5.7</td>
</tr>
<tr>
<td>S21</td>
<td>Policy harmonisation, branch 1 (per PolicyID, parallel)</td>
<td>917–1494</td>
<td>5.8</td>
</tr>
<tr>
<td>S22</td>
<td>Policy harmonisation, branch 2 (per PolicyID+Country or PolicyID+CATNAT_Label, parallel), plus partial re-application</td>
<td>1496–2065</td>
<td>5.8</td>
</tr>
<tr>
<td>S23</td>
<td>YearBuilt validation</td>
<td>2068–2071</td>
<td>5.9</td>
</tr>
<tr>
<td>S24</td>
<td>Negative SI → abs</td>
<td>2073–2075</td>
<td>5.9</td>
</tr>
<tr>
<td>S25</td>
<td>Row-count check and removal of temporary columns</td>
<td>2081–2088</td>
<td>–</td>
</tr>
<tr>
<td>S26</td>
<td>Change log (before removals)</td>
<td>2090–2110</td>
<td>5.9</td>
</tr>
<tr>
<td>S27</td>
<td>Row removals: Fac (2023) → TSI = 0 → sanctioned</td>
<td>2113–2171</td>
<td>5.10</td>
</tr>
<tr>
<td>S28</td>
<td>Clean up the environment</td>
<td>2174–2177</td>
<td>–</td>
</tr>
</tbody>
</table><p>Dependency chain: S1 → S2 → S3 → S6. S6 must finish before S7, S8 and S9 because they read the corrected Country. S9 must finish before S11. S11 → S12 → S13 → S14. S16 must finish before S18, S21 and S22 (they use <code>n</code>, <code>seq_ID</code> and <code>group_id</code>). S17 → S18 → S19. S20 must finish before S21 and S22. S21 and S22 must finish before S23–S27. S26 depends on the snapshot from S2 and on row order being kept (<code>arrange(seq_ID)</code>).</p>
<hr>
<h2 id="step-by-step-analysis">3. Step-by-step analysis</h2>
<h3 id="s0-—-load-standards-l6–8">S0 — Load standards (L6–8)</h3>
<ul>
<li><strong>[FACT]</strong> Re-sources <code>Standards.R</code>, which redefines every constant in §1.2. As a side effect it recomputes <code>empty_columns</code> on the current <code>exposure</code> (<code>STD:L84–86</code>). A column counts as empty when every value is NA, <code>""</code> or <code>0</code>.</li>
<li><strong>[FACT]</strong> <code>c_year</code> falls back to <code>year(Sys.Date())</code>, so results that depend on c_year (YearBuilt, the 2023 Fac rule) change with the run date when <code>year == ""</code>.</li>
<li><strong>PySpark:</strong> load configuration as explicit parameters or broadcast lookup tables: MU, sub, modeling_option, c_year, perils→cat_peril, LOB_list, Country_list, string lists. Pass c_year explicitly; do not derive it from the clock. Compute <code>empty_columns</code> as one aggregation pass over the input DataFrame: for each column, count rows that are not null, not “” and not 0.</li>
</ul>
<h3 id="s1-—-default-country-and-currency-l14–32">S1 — Default Country and Currency (L14–32)</h3>
<ul>
<li><strong>Function:</strong> <code>assign_values(MU)</code> (L19–24).</li>
<li><strong>Rules [FACT]:</strong>
<ul>
<li>When Country is NA, set it to <code>Country_list[[MU]]$Country</code>.</li>
<li>When Country is <code>"??"</code>, set it to the same value.</li>
<li>When Currency is NA, set it to <code>Country_list[[MU]]$Currency</code>.</li>
<li>An empty string <code>""</code> is <strong>not</strong> defaulted. This step runs before S3 turns junk into NA, so values that S3 sets to NA keep NA (Country is part of <code>geo_cols</code>).</li>
</ul>
</li>
<li><strong>[ASSUMPTION]</strong> MU is a key of <code>Country_list</code>. Otherwise the assignment of <code>NULL</code> fails.</li>
<li><strong>Outputs:</strong> logged counts (the comparison uses <code>exposure_org</code> from L26).</li>
<li><strong>PySpark:</strong> <code>when(col.isNull() | (col == "??"), lit(default))</code> for Country, and <code>when(isNull)</code> for Currency. Defaults come from MU configuration.</li>
</ul>
<h3 id="s2-—-temporary-join-1-and-snapshot-l34–40">S2 — Temporary join #1 and snapshot (L34–40)</h3>
<ul>
<li><strong>[FACT]</strong> Left join with <code>Mapping_PostalCode</code> on <code>Country = CountryCode</code>. This adds <code>CountryCode_ISO3</code>, <code>CountryName</code>, <code>Standard</code>, <code>Single</code>, <code>Length</code>, <code>LengthMax</code>, <code>Format</code>, <code>FormatExact</code> and <code>Denomination</code> (the CSV header).</li>
<li><strong>[FACT]</strong> <code>exposure_org</code> is snapshotted <strong>after</strong> S1 and this join. <code>length_df</code> is the row count.</li>
<li><strong>Cardinality:</strong> many-to-one is expected. <strong>[FACT]</strong> The CSV has no duplicate <code>CountryCode</code>. <strong>[ASSUMPTION]</strong> The xlsx has none either. A duplicate would multiply rows, and L2081 would only warn about it.</li>
<li><strong>[RISK]</strong> If the input already has a column with the same name as a mapping column (for example <code>CountryName</code>), the join creates <code>.x</code>/<code>.y</code> suffixes and L344/L2087 would fail.</li>
<li><strong>[RISK]</strong> The change log (S26) compares against this snapshot, so changes made by S1 are <strong>not</strong> logged.</li>
<li><strong>PySpark:</strong> <code>join(broadcast(mapping), Country == CountryCode, "left")</code>. Assert unique keys in the mapping first. Keep a snapshot of the original values, for example the same DataFrame persisted and keyed by <code>seq_ID</code>, created in S2. Note that R’s <code>seq_ID</code> is only created in S16; <strong>[REC]</strong> assign the row key here in the PySpark version.</li>
</ul>
<h3 id="s3-—-remove-junk-from-geo-columns-l42–70">S3 — Remove junk from geo columns (L42–70)</h3>
<p>Each operation runs on every column in <code>geo_cols</code>, in this order:</p>
<ol>
<li>
<p><strong><code>DetectReplace_var</code></strong> (L47–50). A value matching <code>var_string</code> (case-insensitive “various”, “diverse”, “diversi”, “unbekannt”, “XXX”, “???”, French “indéterminé” and so on) becomes NA. There are two exceptions:</p>
<ul>
<li>The row’s <strong>City</strong> contains “diversi” and Country == “IT”.</li>
<li>The row’s <strong>Street</strong> contains “de la Diversit”.</li>
</ul>
<p><strong>[FACT]</strong> The exceptions look at City and Street, not at the column being processed. A row that matches an exception keeps the var_string values in <strong>all</strong> its geo columns.</p>
</li>
<li>
<p><strong><code>DetectReplace_coord</code></strong> (L52–55). A value matching <code>coord_string</code> (coordinate-like text, “Koordinaten” and so on) becomes NA.</p>
</li>
<li>
<p><strong><code>Repl_NA</code></strong> (L58–61). The literal text <code>"NA"</code> becomes NA in <code>geo_cols[-1]</code>, which <strong>excludes Country</strong>. <strong>[INFERRED]</strong> This keeps Namibia’s ISO2 code <code>"NA"</code>.</p>
</li>
<li>
<p><strong><code>values_string</code></strong> (L69–70). An exact match to one of <code>"Notavalaible","Na","N A","n a","N a","&lt;NA&gt;","n/a","N/A","N/a","0","na","..","...","-",".","X","x","XX",…,"#","NULL"</code> becomes NA, <strong>including in Country</strong>. So Country <code>"Na"</code> or <code>"na"</code> becomes NA, but <code>"NA"</code> stays.</p>
</li>
</ol>
<p>Other points:</p>
<ul>
<li><strong>[FACT]</strong> Values made NA here are <strong>not</strong> re-defaulted by S1.</li>
<li><strong>[FACT]</strong> CRESTA <code>"0"</code> already becomes NA here (S7’s first rule is redundant).</li>
<li><strong>PySpark:</strong> <code>when(col.rlike(var_pattern) &amp; col.isNotNull() &amp; ~exception_row, null)</code>. Compute <code>exception_row</code> once from City, Country and Street. Then the coordinate rule, then <code>col == "NA"</code> (skip Country), then <code>col.isin(values_string)</code>.
<ul>
<li>Regex translation: ICU (stringr) to Java. <code>(?i)</code> is supported. Check the <code>[[:digit:]+]</code> classes in <code>coord_string</code>: they are bracket sets containing <code>[:digit:]</code> and <code>+</code>, which Java supports only as <code>\p{Digit}</code>.</li>
</ul>
</li>
</ul>
<h3 id="s4-—-muat-migration-placeholder-l72–73">S4 — MUAT migration placeholder (L72–73)</h3>
<ul>
<li><strong>[FACT]</strong> Rule: City == “Migration” and PostalCode == “1000” and Street == “Migration” and StreetNo == “1” → set all four to NA. The rule is <strong>not</strong> limited to MU == “MUAT”, even though the comment says “MUAT Prop Script”.</li>
<li><strong>[RISK]</strong> It uses <code>exposure[na.omit(mask), cols] &lt;- NA</code>. <code>na.omit</code> drops the NA elements of the logical mask. NA appears when no part of the comparison is FALSE, for example a row where all four fields are NA, which is common. Dropping them shifts every later TRUE to an earlier position, so the <strong>wrong rows can be blanked</strong> (and the shortened mask may be recycled). Intended rule [INFERRED]: an exact match on all four, with NA treated as no match. <strong>Verify in R</strong> with a frame that has an all-NA row before a migration row.</li>
<li><strong>PySpark:</strong> <code>when(City=="Migration" &amp; PostalCode=="1000" &amp; Street=="Migration" &amp; StreetNo=="1", null)</code> on each of the four columns (null treated as false). Decide between parity and the intended rule (Q-G2).</li>
</ul>
<h3 id="s5-—-cast-postalcode-to-character-l79–82">S5 — Cast PostalCode to character (L79–82)</h3>
<ul>
<li><strong>[FACT]</strong> If PostalCode is not character, it is cast with <code>as.character</code>. Leading zeros already lost by a numeric import are not recovered.</li>
<li><strong>[RISK]</strong> The cast happens after the S2 snapshot. If PostalCode arrived numeric, <code>identical()</code> in S26 flags <strong>every</strong> row as changed.</li>
<li><strong>PySpark:</strong> read PostalCode as a string at ingestion (schema-on-read). Do not cast later.</li>
</ul>
<h3 id="s6-—-country-corrections-l96–272-in-this-order">S6 — Country corrections (L96–272), in this order</h3>
<ol>
<li><strong>MUFR only</strong> (L97–192): 94 exact-match assignments from French names to ISO2 (<code>ALBANIE→AL</code> … <code>LITUANIE→LT</code>). <code>mappers/mu_france_country_code.csv</code> has 93 data rows. <strong>[REC]</strong> Reconcile the two lists. Duplicates in the R list, such as BARBADE/BARBADES and AFRIQUE DU SUD/AFRIQUE DE SUD, are aliases that point to the same code.</li>
<li><strong>Aliases</strong> (L195–196): <code>FL→LI</code>, <code>UK→GB</code> (all MUs).</li>
<li><strong>City → Country</strong> (L199–254). <code>update_country(pattern, code)</code> sets Country = code when City matches the regex (case-insensitive, <code>\b</code> word boundaries; a few are anchored <code>^…$</code>) and Country is not already that code. The calls run in sequence, <strong>so a later call overrides an earlier one</strong>:
<ul>
<li><strong>[FACT]</strong> <code>^Milford$</code> → IE (L235), then immediately → GB (L236). Net effect: every “Milford” becomes GB.</li>
<li><strong>[FACT]</strong> <code>Litija</code> → DZ (L227), later → SI (L252). Net effect: SI.</li>
<li><strong>[FACT]</strong> <code>Namibia</code> → <code>"NA"</code> (L244). The country code string “NA” must survive every later step and every CSV read. In pandas, “NA” is read as NaN by default; Spark’s CSV reader keeps it.</li>
<li>The patterns include a street-like value: <code>5591 East Francis St.</code> → US.</li>
</ul>
</li>
<li><strong>Special cases</strong> (L257–263):
<ul>
<li>Street contains “HONGRIE” → HU.</li>
<li>Country is FR or MC and PostalCode starts with “98” → MC (L258).</li>
<li><strong>[RISK]</strong> This captures 984xx, 986xx, 987xx and 988xx <strong>before</strong> the overseas rules in S8 run, so FR→TF/WF/PF/NC never happens. They become MC, and their CRESTA becomes <code>FRA_99</code> in S7. This contradicts wiki 5.2.</li>
<li>PostalCode matches the US Format and City matches Richmond, Amsterdam, Munster or Bristol → US.</li>
<li>PostalCode matches the CA Format and City matches Richmond → CA.</li>
<li>PostalCode matches the GB Format and City matches Richmond or Bristol → GB.</li>
<li>City matches London and PostalCode does <strong>not</strong> match the CA Format → GB. <strong>[FACT]</strong> If PostalCode is NA, the result is NA, which is treated as FALSE, so “London” with no postal code is not changed.</li>
<li>These rules read the <strong>raw</strong> postal code: it has not been cleaned yet (S10–S12).</li>
</ul>
</li>
<li><strong>CH → LI</strong> (L268–272): Country == CH and PostalCode is one of the strings “9485” … “9499” → LI. It is an exact string match on the raw value, so “FL-9490” or "9490 " does not match.</li>
</ol>
<ul>
<li><strong>PySpark:</strong>
<ul>
<li>Represent the MUFR names and aliases as a broadcast lookup table joined on Country. The MUFR table is gated on MU.</li>
<li>Represent the city rules as an <strong>ordered</strong> rule table. Applying them sequentially with <code>when</code> chains in R order gives the same last-match-wins result. Equivalently, take the rule with the highest priority that matches.</li>
<li>Java regex: <code>\b</code> next to non-ASCII letters (Zeebrügge, Eslöv) behaves differently from ICU unless <code>(?U)</code> is used. Test these cases.</li>
<li>Use <code>rlike</code> for the ambiguous-city rules, with the per-country Format pattern looked up from the mapping (a constant per rule).</li>
</ul>
</li>
</ul>
<h3 id="s7-—-cresta-l274–294-count-at-l340">S7 — CRESTA (L274–294, count at L340)</h3>
<ol>
<li><code>CRESTA == "0"</code> → NA (L278).</li>
<li><strong>CH mapping</strong> (L280–285) runs only if <strong>more than one</strong> CH row has a CRESTA starting with a digit (<code>length(...) &gt; 1</code>).
<ul>
<li><strong>[RISK]</strong> A dataset with exactly one such row skips the mapping. The likely intent [INFERRED] is <code>&gt; 0</code>.</li>
<li>When it runs: left join <code>Mapping_CRESTA_CH</code> on (Country = CountryCode, CRESTA = CRESTAZone), then <code>CRESTA = ifelse(Country=="CH", CRESTA_ID, CRESTA)</code>.</li>
<li><strong>[RISK]</strong> For <strong>every</strong> CH row, CRESTA is replaced by the join result. CH rows that already hold <code>CHE_ZH</code>, or a zone without zero-padding (“1” instead of “01”; the mapping uses <code>01</code>…<code>26</code>), get <strong>NA</strong>. Rows with Country NA get NA (<code>ifelse</code> on NA).</li>
<li>Cardinality: many-to-one. <strong>[FACT]</strong> The CSV has no duplicate (CountryCode, CRESTAZone).</li>
<li><strong>[RISK]</strong> The CSV header has a trailing space (<code>ZoneName</code>), and <code>select(-ZoneName)</code> would fail on it. Check whether the xlsx header has the same problem.</li>
</ul>
</li>
<li><strong>GB/US</strong> (L287): letters only, at most 2 characters → <code>paste(CountryCode_ISO3, "_", CRESTA)</code>.</li>
<li><strong>Generic prefix</strong> (L289): CRESTA not starting with a letter or punctuation → <code>ISO3_CRESTA</code>.
<ul>
<li><strong>[RISK]</strong> <code>CountryCode_ISO3</code> comes from S2, which used the <strong>pre-correction</strong> Country. Countries changed in S6 get the old country’s ISO3, or <code>NA_…</code> when the old value was not a code (UK, FL, every MUFR French name). Also flagged in <code>docs/Geo_Standardization_Agnostic_Plan.md</code>.</li>
<li><strong>[FACT]</strong> The left side of the assignment uses <code>str_detect</code> (ICU) and the right side uses <code>grepl</code> (TRE) for the same pattern. Their Unicode handling could differ in theory.</li>
</ul>
</li>
<li><strong>Fixed overrides</strong> (L292–294): LI → <code>LIE_LIE</code>, HK → <code>CHN_HK</code>, MC → <code>FRA_99</code>. These are applied even when CRESTA was NA.</li>
</ol>
<ul>
<li><strong>PySpark:</strong> a conditional broadcast join to the CH mapping (decide on the <code>&gt;1</code> gate, Q-G4); <code>when</code> chains for the prefix rules using <code>concat_ws("_", iso3, cresta)</code>; Java regex <code>^\p{Alpha}+$</code> and <code>^[\p{Alpha}\p{Punct}]</code>.
<ul>
<li>To reproduce R exactly, ISO3 must come from the <strong>S2-time</strong> country. Keep it as a frozen column; do not re-join.</li>
</ul>
</li>
</ul>
<h3 id="s8-—-fr-overseas-split-l297–308">S8 — FR overseas split (L297–308)</h3>
<ul>
<li><strong>[FACT]</strong> It runs <strong>after</strong> CRESTA. When Country == FR and PostalCode has 5 characters:
<ul>
<li><code>^971</code> → GP (except 97133 and 97150), <code>^972</code> → MQ, <code>^973</code> → GF, <code>^974</code> → RE, <code>^975</code> → PM, <code>^976</code> → YT.</li>
<li><code>^984</code> → TF, <code>^986</code> → WF, <code>^987</code> → PF, <code>^988</code> → NC. These are unreachable because of S6 (L258).</li>
<li><code>97133</code> → BL, <code>97150</code> → MF.</li>
<li>The regex <code>^971+</code> means “97 followed by one or more 1s”. In practice it is a prefix test.</li>
</ul>
</li>
<li><strong>[FACT]</strong> CRESTA is not recalculated, so these rows keep the <code>FRA_…</code> prefix from S7.</li>
<li><strong>[FACT]</strong> Downstream effect: Mapping_PostalCode has <code>Single = 97399</code> for GF and <code>98799</code> for PF. In S11 every GF row’s PostalCode is overwritten with 97399.</li>
<li><strong>PySpark:</strong> an ordered <code>when</code> chain on <code>substring(PostalCode, 1, 3)</code> and <code>length == 5</code>.</li>
</ul>
<h3 id="s9-—-temporary-join-2-l342–345">S9 — Temporary join #2 (L342–345)</h3>
<ul>
<li><strong>[FACT]</strong> Drops every mapping column except <code>CountryCode</code> (which the join never added) and re-joins on the <strong>corrected</strong> Country. Countries missing from the mapping get NA attributes, and S11 then stops with an error.</li>
<li><strong>PySpark:</strong> drop the attribute columns and join the broadcast mapping again.</li>
</ul>
<h3 id="s10-—-postal-pre-rules-l350–355">S10 — Postal pre-rules (L350–355)</h3>
<ul>
<li>CH and PostalCode == “9999” → PostalCode NA.</li>
<li>PostalCode == City → PostalCode NA.</li>
<li>Street == City → Street NA.</li>
<li><strong>[FACT]</strong> If Street is a fixed-string <strong>substring</strong> of City, Street → NA (L355, <code>grepl(fixed=TRUE)</code>). Example: Street “Rue” and City “Rueil” clears the Street. The wiki only says “Street = City”.</li>
<li><strong>PySpark:</strong> <code>when(City == PostalCode, null)</code> and so on. For the substring rule, <code>instr(City, Street) &gt; 0</code> with both columns not null.</li>
</ul>
<h3 id="s11-—-per-country-postal-and-address-loop-l357–452">S11 — Per-country postal and address loop (L357–452)</h3>
<p>For each distinct Country <code>i</code>, in the order of <code>unique()</code>:</p>
<ul>
<li><code>map_code = Format[i]</code>, <code>map_code_exact = FormatExact[i]</code>. If the country is missing from Mapping_PostalCode, call <strong><code>stop()</code></strong> (L370–373).
<ul>
<li><strong>[FACT]</strong> Country NA does <strong>not</strong> trigger the stop. <code>CountryCode == NA</code> returns a vector of NAs whose length is not 0, and <code>which(Country == NA)</code> is empty, so the loop moves on. NA-country rows are skipped silently.</li>
</ul>
</li>
<li><strong>[FACT] All conditions are computed once, on the state before any change</strong> (L379–387). They are then applied in sequence to <code>sub_exp</code>, so later rules use <strong>stale</strong> flags on rows that earlier swaps already changed.</li>
</ul>

<table>
<thead>
<tr>
<th>Flag</th>
<th>Condition (on pre-loop values)</th>
</tr>
</thead>
<tbody>
<tr>
<td>C1</td>
<td>StreetNo matches Format, PostalCode NA or not matching Format, StreetNo and Format not NA</td>
</tr>
<tr>
<td>C1a</td>
<td>Same as C1 but StreetNo matches FormatExact</td>
</tr>
<tr>
<td>C0</td>
<td>Street NA; StreetNo has alphanumerics, does not match Format and is not all digits</td>
</tr>
<tr>
<td>C2</td>
<td>City matches Format; PostalCode NA or not matching</td>
</tr>
<tr>
<td>C2a</td>
<td>City matches FormatExact; PostalCode not NA and not all digits; FormatExact not NA</td>
</tr>
<tr>
<td>C4</td>
<td>Street is all digits; StreetNo has alphanumerics</td>
</tr>
<tr>
<td>C5</td>
<td><code>Single</code> not NA</td>
</tr>
<tr>
<td>C6</td>
<td><code>Length</code> not NA and PostalCode not NA. C6a adds <code>LengthMax</code> not NA</td>
</tr>
</tbody>
</table><p>Actions, in order:</p>
<ol>
<li><strong>C1 and City NA:</strong> rotate PostalCode ← StreetNo, City ← old PostalCode, StreetNo ← NA (old City).<br>
<strong>C1a:</strong> PostalCode ← StreetNo, StreetNo ← NA.
<ul>
<li><strong>[RISK]</strong> A row that is both C1 with City NA and C1a has StreetNo = NA after the rotation. C1a then sets PostalCode = NA, so the postal code is lost and the old invalid postal value sits in City.</li>
<li>C1 rows where City is not NA and StreetNo is not an exact match are left unchanged.</li>
</ul>
</li>
<li><strong>C0:</strong> swap Street and StreetNo.</li>
<li><strong>C2a:</strong> swap PostalCode and City. <strong>C2:</strong> PostalCode ← <code>str_extract(City, Format)</code>, City ← <code>str_replace(City, Format, "")</code> (first occurrence only), and an empty City becomes NA.
<ul>
<li><strong>[RISK]</strong> C2a rows are normally also C2 rows. After the swap, City holds the old non-numeric PostalCode, so <code>str_extract</code> returns NA and the postal code just swapped in is overwritten with NA.</li>
</ul>
</li>
<li><strong>C4:</strong> swap Street and StreetNo.</li>
<li><strong>C5:</strong> PostalCode ← <code>Single</code> for <strong>every</strong> row of that country, overwriting what is there.</li>
<li><strong>C6:</strong>
<ul>
<li>Remove all non-digits from PostalCode.</li>
<li>If it is shorter than <code>Length</code>, left-pad with “0” to <code>Length</code>.</li>
<li>If it is longer than <code>Length</code> and shorter than <code>LengthMax</code>, left-pad to <code>LengthMax</code>.</li>
<li>Codes longer than LengthMax, or with no LengthMax, are left as they are.</li>
<li><strong>[RISK]</strong> A non-numeric value in a country with a numeric format becomes <code>""</code> and is then padded to <code>"0000"</code>.</li>
<li><strong>[FACT]</strong> Rows that only got a PostalCode in steps 1–3 are not C6 rows, so they are not stripped or padded here.</li>
<li><strong>[FACT] / [RISK]</strong> The 9 countries with both <code>Single</code> and <code>Length</code> (for example HK, VA, GF, PF) get their Single value stripped and padded.</li>
<li><strong>[RISK] / [FACT]</strong> For US (Length 5, LengthMax 9), a 6–8 digit code is left-padded to 9 digits. For PT (4/7), 5–6 digits are padded to 7.</li>
</ul>
</li>
</ol>
<ul>
<li><strong>[FACT]</strong> GB, CA and NL have no Length in the CSV, so they keep their letters here.</li>
<li><strong>PySpark:</strong> no loop is needed. The per-country Format, FormatExact, Single, Length and LengthMax are already columns from S9.
<ul>
<li>Compute all flags as columns in <strong>one</strong> <code>select</code> from the pre-loop values. Then apply the actions as an ordered series of <code>withColumn</code> steps that read the flag columns and the current values, keeping the R sequencing (and the risks above, if parity is chosen).</li>
<li>Spark’s <code>rlike</code> takes the pattern from a column only through <code>expr("City rlike Format")</code>. Java supports the bounded lookbehinds used in Format (<code>(?&lt;=[^0-9]|^)</code>), but each pattern must be checked.</li>
<li><code>regexp_extract</code> returns <code>""</code>, not null, when there is no match. Convert <code>""</code> to null.</li>
<li>Spark has no first-occurrence <code>regexp_replace</code>. Use the extracted value and replace its first position, or anchor the pattern. <strong>[REC]</strong> Test this explicitly.</li>
<li><code>lpad</code> truncates strings longer than the target, so only call it when the value is shorter, as R does.</li>
<li>The <code>stop()</code> becomes a pre-check: count distinct non-null countries with no mapping row, and raise if the count is above 0.</li>
</ul>
</li>
</ul>
<h3 id="s12-—-global-postal-post-processing-l454–465">S12 — Global postal post-processing (L454–465)</h3>
<ul>
<li>Remove every non-alphanumeric character from PostalCode, for all countries. For example “SW1A 1AA” becomes “SW1A1AA” and “1000-001” becomes “1000001”.</li>
<li><strong>GB:</strong> if the last 3 characters are letters and the third-to-last is O or I, replace <strong>every</strong> O with 0 (or I with 1) in the whole code.
<ul>
<li><strong>[RISK]</strong> This also changes valid letters in the outward code, for example <code>OL1OAB</code> → <code>0L10AB</code>.</li>
</ul>
</li>
<li><strong>CA:</strong> the same idea with conditions on characters 1–3 and 2, and on the last 3 characters (third-to-last or last). A global replace is harmless for CA because valid Canadian codes never use O or I.</li>
<li><strong>PySpark:</strong> <code>regexp_replace(pc, "[^\\p{Alnum}]", "")</code>. Then <code>when(&lt;substring tests&gt;, regexp_replace(pc, "[Oo]", "0"))</code> in the same order as L456–463. Each line sees the result of the line before it.</li>
</ul>
<h3 id="s13-—-street-and-streetno-rules-l467–479">S13 — Street and StreetNo rules (L467–479)</h3>
<ul>
<li>Street → NA when it is any of:
<ul>
<li>all digits, or the empty string</li>
<li><code>^\d+[a-zA-Z]$</code> (for example “12a”)</li>
<li><code>^[a-zA-Z]\s\d{1,5}$</code></li>
<li><code>^[a-zA-Z]\d{1,5}$</code></li>
<li><code>^[0.9]\s\d{1,5}$</code>. <strong>[RISK]</strong> <code>[0.9]</code> matches only ‘0’, ‘.’ and ‘9’. It is probably a typo for <code>[0-9]</code>.</li>
</ul>
</li>
<li>StreetNo → NA when Street is NA.</li>
<li>StreetNo → NA when it is letters only.</li>
<li><strong>PySpark:</strong> <code>when(rlike(...), null)</code>, in order.</li>
</ul>
<h3 id="s14-—-trim-geo-columns-l482–491">S14 — Trim geo columns (L482–491)</h3>
<ul>
<li>For every geo column: replace “, ,” with ", " and “, .” with ", ". Then <code>trimws</code> with <code>" "</code>, then <code>","</code>, then <code>"\\(\\)"</code>, then <code>str_squish</code>.</li>
<li><strong>[FACT]</strong> R’s <code>trimws(whitespace=w)</code> builds the regex <code>^w+</code> / <code>w+$</code>. So <code>"\\(\\)"</code> removes only a <strong>literal “()” followed by more “)”</strong> at the start or end, not single brackets. Dots are not trimmed either. The wiki says “commas, dots, brackets”.</li>
<li><strong>[FACT]</strong> Values that end up as <code>""</code> are not turned into NA.</li>
<li><strong>PySpark:</strong> <code>regexp_replace</code> chains: <code>^ +| +$</code>, then <code>^,+|,+$</code>, then <code>^\(\)+|\(\)+$</code>, then collapse <code>\s+</code> to " " and trim. This also handles tabs and newlines, as <code>str_squish</code> does.</li>
</ul>
<h3 id="s15-—-percentages-l497–511">S15 — Percentages (L497–511)</h3>
<ul>
<li><strong>[FACT]</strong> Columns: <code>Fac</code>, <code>Sux</code> and every column ending in <code>_Percent</code>. <code>_Percent_Min</code> and <code>_Percent_Max</code> are <strong>not</strong> included.
<ul>
<li>Step 1: if 1 &lt; x &lt; 2, set x = 1.</li>
<li>Step 2: if 2 ≤ x ≤ 100, set x = x / 100.</li>
<li>Exactly 1 and values above 100 are left as they are. NA stays NA.</li>
</ul>
</li>
<li><strong>[INFERRED]</strong> The source is mixed: some values are fractions, some percentages. 1.5 becomes 100%, not 1.5% (wiki Q5 is missing).</li>
<li><strong>[ASSUMPTION]</strong> <code>Fac</code> and <code>Sux</code> exist and are numeric. Otherwise <code>across()</code> fails.</li>
<li><strong>PySpark:</strong> two passes of <code>when</code> over the selected columns. Choose the columns with a suffix test on <code>df.columns</code>.</li>
</ul>
<h3 id="s16-—-row-id-location-count-and-group-id-l513–526">S16 — Row ID, location count and group ID (L513–526)</h3>
<ul>
<li><code>seq_ID = row number</code>.</li>
<li><code>n = count per PolicyID</code> (from <code>add_count</code>).</li>
<li><code>group_id = "ID" + cur_group_id()</code> over (PolicyID, Country). For MUFR + sub “Prop” + modeling_option “CATNAT” (any case) with a <code>CATNAT_Label</code> column, it uses (PolicyID, CATNAT_Label) instead.</li>
<li><strong>[FACT]</strong> <code>cur_group_id</code> numbers groups in sorted key order (C-locale sort in dplyr ≥ 1.1, NA last; <strong>verify</strong> the installed dplyr version). After this step <code>exposure</code> is a tibble.</li>
<li><strong>[FACT]</strong> <code>n</code> is never recalculated after row removals.</li>
<li><strong>PySpark:</strong>
<ul>
<li><code>seq_ID</code>: <code>row_number()</code> over a <strong>deterministic</strong> input order. Spark has no implicit row order. Carry an ingestion index (for example <code>zipWithIndex</code> at read time) or fix a sort key.</li>
<li><code>n</code>: <code>count(*)</code> over a window partitioned by PolicyID.</li>
<li><code>group_id</code>: <code>dense_rank()</code> over keys ordered ascending with nulls last, as a string with an “ID” prefix. This matters only if group_id values have to match exactly.</li>
</ul>
</li>
</ul>
<h3 id="s17-—-location-level-deductibles-l528–686">S17 — Location-level deductibles (L528–686)</h3>
<p>For each peril P (<code>cat_peril</code>):</p>
<p><strong>Block A</strong> (L531–581):</p>
<ol>
<li>
<p>A1: <code>P_Deductible_per</code> not NA and not “None”, and Fix == 0 and Percent == 0 for all 5 coverages → “None”.</p>
</li>
<li>
<p>A2: per is NA or “None”, and any Fix &gt; 0 or Percent &gt; 0 → “Policy” (the default).</p>
</li>
<li>
<p>A3, granularity. This runs separately for each column family (Kind, Fix, Percent, Min, Max):</p>
<ul>
<li>Combined (1) is cleared when B, C or BI (3, 4, 5) is set.</li>
<li>Com (2) is cleared when B or C (3, 4) is set. <strong>BI is not part of this test.</strong></li>
<li>Com (2) is cleared when Combined (1) is still set.</li>
</ul>
<p>“Cleared” means NA for Kind and 0 for values.</p>
<ul>
<li><strong>[FACT]</strong> When both Combined and Com are present, <strong>Com is dropped and Combined is kept</strong>.</li>
<li><strong>[FACT]</strong> Because each family is handled separately, Kind and the values can get out of step. Block B rebuilds Kind from the values.</li>
</ul>
</li>
<li>
<p>A4: <code>P_Cover == FALSE</code> → Kind NA, per NA, values 0.</p>
</li>
</ol>
<p><strong>Block B</strong> (L583–634), for each coverage c:</p>
<ul>
<li>Helper columns are built and dropped every iteration: <code>SI = B+C+O+BI</code>, <code>SI_Com = B+C+O</code>, <code>SI_B</code>, <code>SI_C</code>. <code>rowSums</code> without <code>na.rm</code> gives NA when any part is NA. For c = BI, <code>SI_cat = "SI_BI"</code>.</li>
<li>B1: Kind set but Fix == 0 and Percent == 0 → Kind NA.</li>
<li>B2:
<ul>
<li>Fix &gt; 0 and Percent == 0 → “Fix”.</li>
<li>Percent &gt; 0 and Fix == 0 → “% of Claims”.</li>
<li><strong>[FACT]</strong> If both are &gt; 0, Kind is left unchanged and may stay NA.</li>
</ul>
</li>
<li>B3: Kind is not “% of Claims” (or is NA) → Min = 0, Max = 0.</li>
<li>B4: <code>SI_cat == 0</code> and per == “Location” → Kind NA, values 0. Policy-level deductibles are kept, with a <code>warning()</code> that lists the PolicyIDs.</li>
</ul>
<p><strong>Block C</strong> (L636–686): exactly the same as Block A, re-run after B.</p>
<ul>
<li><strong>[ASSUMPTION]</strong> The numeric deductible columns and <code>P_Cover</code> have <strong>no NA</strong>. A data.frame or tibble sub-assignment with NA in the logical row mask raises an error in R. <strong>Verify in R</strong> and check the upstream fill.</li>
<li><strong>PySpark:</strong> one <code>select</code> per rule with <code>when</code> expressions. Build column names from <code>cat_peril</code> × <code>cat_ded</code> with the same <code>gsub("__","_")</code> naming.
<ul>
<li>Keep the rule order. Each R statement is a new <code>withColumn</code> on the result of the one before.</li>
<li>Null semantics: an R vector assignment with an NA mask skips that row. Spark’s <code>when</code> with a null condition falls through to <code>otherwise</code>. These are equivalent.</li>
<li><strong>[REC]</strong> Add explicit null checks that raise an error, to copy R’s failure behaviour.</li>
</ul>
</li>
</ul>
<h3 id="s18-—-single-location-policies-with-mixed-kinds-l688–784">S18 — Single-location policies with mixed kinds (L688–784)</h3>
<p>For each peril, with <code>single = (n == 1)</code>:</p>
<ol>
<li><code>div</code> = single rows where at least one coverage Kind is “Fix” <strong>and</strong> at least one is “% of Claims”. In those rows every “Fix” Kind becomes “% of Claims” (L701). <strong>Verify in R</strong> how the logical-matrix assignment behaves when some Kind values are NA.</li>
<li><code>fix[[j]]</code> is one <strong>global</strong> flag per coverage: TRUE if <em>any</em> div row has Kind_j “% of Claims” and Fix_j &gt; 0.</li>
<li>Every numeric column of the single rows goes 0 → NA (L710). Then <code>perc_values</code>, <code>min_values</code> and <code>max_values</code> are computed as the unique non-NA values of each row across the 5 coverages. If any row has more than one, call <code>stop()</code>. <strong>Verify in R:</strong> with rows of different lengths, <code>apply</code> returns a list and the length check behaves differently.</li>
<li>Every numeric column of the single rows goes NA → 0 (L728).
<ul>
<li><strong>[RISK]</strong> This turns <strong>every</strong> pre-existing numeric NA in a single-location row into 0. That covers lat/lon, SI, YearBuilt if numeric, and so on. It also changes integer columns to double.</li>
</ul>
</li>
<li>For each coverage j where <code>fix[[j]]</code> is TRUE, for <strong>all</strong> div rows:
<ul>
<li>
<p>Percent_j ← <code>perc_values</code></p>
</li>
<li>
<p>Min_j ← Fix_j</p>
</li>
<li>
<p>Max_j ← <code>ifelse(length(max_values)==0, 0, max_values)</code></p>
</li>
<li>
<p>Fix_j ← 0</p>
</li>
<li>
<p><strong>[RISK]</strong> Because the flag is global, coverages that had no Fix in a given row also get their Min overwritten by 0 and their Percent filled.</p>
</li>
<li>
<p><strong>[RISK]</strong> <code>ifelse</code> with a scalar condition returns <strong>only <code>max_values[1]</code></strong>, so every div row gets the first row’s Max.</p>
</li>
</ul>
</li>
<li>Again 0 → NA, then find single rows whose Min (or Max) values differ across coverages, then NA → 0.</li>
<li>SI-weighted Min for those rows. Weights are the coverage SI divided by total SI: 1 for combined, SI_Com/SI, SI_B/SI, SI_C/SI, SI_BI/SI. The weighted sum is written to <strong>every coverage j where any affected row has Min_j &gt; 0</strong> (again a global flag). Max is handled the same way.
<ul>
<li><strong>[RISK]</strong> Total SI = 0 gives NaN. In R, <code>is.na(NaN)</code> is TRUE. In Spark, NaN is not null, so an explicit <code>isnan</code> check is needed.</li>
</ul>
</li>
</ol>
<ul>
<li><strong>PySpark:</strong>
<ul>
<li>Arrays: <code>array_distinct(filter(array(cols), x -&gt; x is not null))</code> for the unique values per row; <code>size(...) &gt; 1</code> with an error for the stop.</li>
<li>The global flags: compute them as aggregates, <code>max(cast(cond as int))</code> over the filtered rows, then broadcast them as scalars (collect once) or cross join them.</li>
<li>The single-row NA→0 side effect: only if parity is required (Q-G6).</li>
</ul>
</li>
</ul>
<h3 id="s19-—-minmax-swap-l786–807-—-not-in-the-wiki">S19 — Min/Max swap (L786–807) — not in the wiki</h3>
<ul>
<li><strong>[FACT]</strong> For every peril and coverage, where Min and Max are both non-NA and Min &gt; Max, swap them.</li>
<li><strong>[RISK]</strong> Max = 0 usually means “no maximum”. In that case the code moves the Min into Max and sets Min to 0.</li>
<li><strong>PySpark:</strong> <code>when(min &gt; max, max)</code> for Min and <code>when(min &gt; max, min)</code> for Max, both computed in one <code>select</code> so they read the original values.</li>
</ul>
<h3 id="s20-—-location-level-sublimits-l809–915">S20 — Location-level sublimits (L809–915)</h3>
<p><strong>Block F</strong> (L812–846), for each peril:</p>
<ol>
<li>F1: per not NA and not “None”, and all 5 sublimits == 0 → “None”.</li>
<li>F2: per is NA or “None”, and any sublimit &gt; 0 → “Policy”.</li>
<li>F3: granularity, the same pattern as A3.</li>
<li>F4: Cover FALSE → per NA, sublimits 0.</li>
<li>F5: <code>P_AAL == TRUE</code> and any of Com, B, C or BI &gt; 0 → Combined sublimit ← <code>max(cbind(all 5 columns), na.rm=T)</code>, and Com, B, C, BI ← 0.
<ul>
<li><strong>[RISK]</strong> <code>max()</code> returns <strong>one scalar over the whole dataset</strong>, so every such row gets the portfolio’s largest sublimit for that peril. The wiki describes a value per location.</li>
<li><strong>[ASSUMPTION]</strong> <code>P_AAL</code> has no NA.</li>
</ul>
</li>
</ol>
<p><strong>Block G</strong> (L849–879): for each coverage, if <code>SI_cat == 0</code> and <code>Sublimit_per == "Location"</code>, set the sublimit to 0. Policy-level values are kept with a warning.</p>
<p><strong>Block H</strong> (L881–915): a copy of Block F, so F5 is applied a second time.</p>
<ul>
<li><strong>PySpark:</strong> <code>when</code> chains as in S17. For F5, parity needs <code>agg(max(greatest(5 cols)))</code> over the whole DataFrame, broadcast as a literal. The likely intent [INFERRED] is the row-wise <code>greatest(...)</code> (Q-D3).</li>
</ul>
<h3 id="s21-—-policy-harmonisation-branch-1-per-policyid-l917–1494">S21 — Policy harmonisation, branch 1: per PolicyID (L917–1494)</h3>
<p><strong>Entry condition [FACT]</strong>, simplified from L918–927, with <code>L = LOB_list[[MU]]$LOB</code>:</p>
<p><code>not CATNAT AND L != "" AND (L not among the LOBs in the data OR at least one LOB other than L exists) AND any of EQ/WS/FL/HA _Deductible_per or _Sublimit_per == "Policy"</code></p>
<p>The middle term <code>!(L %in% others)</code> is always TRUE, so it can be dropped.</p>
<ul>
<li><strong>[RISK]</strong> Only MUCH (<code>MUCH_RSI</code>) and MUDE (<code>MUDE_Large_HV</code>) have a non-empty L, so <strong>only MUCH and MUDE can enter this branch</strong> (finding 1).</li>
<li><strong>[RISK]</strong> If one of the 8 named columns is missing (for example the HA columns when HA is not modelled), R’s zero-length comparison makes <code>any(...)</code> FALSE, and the branch is skipped silently. <strong>Verify in R.</strong></li>
<li><strong>[RISK]</strong> If no policy has <code>n &gt; 1</code>, <code>txtProgressBar(max = 0)</code> raises an error. Branch 2 has a guard for this (L1516); branch 1 does not.</li>
</ul>
<p><strong>Scope:</strong> <code>PolicyID_dist</code> = PolicyIDs with <code>LOB ∉ L</code> and <code>n &gt; 1</code>. Each policy is processed on a PSOCK worker (<code>foreach … %dopar%</code>, combined with <code>rbind</code>). Inside, for each peril P:</p>
<ol>
<li><strong>Fill step (case 1).</strong>
<ul>
<li>Within the policy, <code>custom_mutate</code>: per → NA where the value is “None” and the policy has more than one distinct value. NA counts as a distinct value (<code>n_distinct</code>).</li>
<li>For every Fix, Percent, Min and Max column: 0 → NA where the column has more than one distinct value in the policy.</li>
<li>Then <code>fill(..., .direction="downup")</code> on per, Kind, Fix, Percent, Min and Max, in row order (seq_ID).</li>
<li><strong>[FACT]</strong> This applies to <strong>every</strong> multi-location policy in scope, including “Location”-level ones. It also refills rows that S17 cleared (Cover = FALSE, Location-level SI = 0). The <code>case1</code> variable (L1012) is computed but never used.</li>
</ul>
</li>
<li><strong>Detection helpers</strong> (FDS):
<ul>
<li><code>ded_corr</code> keeps rows with SI &gt; 0, Cover TRUE, LOB ∉ L, and policies with at least one per == “Policy”. It adds <code>num</code> = number of distinct per values (NA included).</li>
<li><code>ded_kind_corr</code> adds, per Kind column, <code>_num0</code> (distinct non-NA values, at least 1), <code>_num</code> (distinct values, NA included), <code>lim_diff</code> and <code>con_zero</code>. <code>n</code> here is the PolicyID count from S16.</li>
</ul>
</li>
<li><strong>Case 2</strong> (L1051–1137): <code>num == 1</code> and the sum of the five <code>Kind_*_num</code> &gt; 5, meaning some coverage has more than one Kind value, NA counted as a value. On the <strong>whole policy</strong> (not only the filtered rows):
<ul>
<li><code>rat_*</code> = (coverage SI of the policy / total SI of the policy) × max(Fix_j, Min_j) over the policy.</li>
<li>Fix, Min and Max 0 → NA.</li>
<li>For each coverage j where Kind_j is not all NA:
<ul>
<li>Kind_j → “% of Claims” if it has more than one distinct value (NA counts, so NA rows also become “% of Claims”).</li>
<li>Fix_j → 0 if the Kind is “% of Claims”; otherwise the <strong>first row’s</strong> Fix_j.</li>
<li>Percent_j → 0 if the Kind is “Fix”; otherwise the first row’s Percent_j.</li>
<li>Min_j → 0 if “Fix”; otherwise Σ rat_* (one SI-weighted value for the policy).</li>
<li>Max_j → 0 if “Fix”; otherwise the first row’s Max_j.</li>
</ul>
</li>
<li><strong>[FACT]</strong> Results take the first element because <code>ifelse</code> has a scalar condition (<code>unique(kind)</code>). For example, a policy with Kind uniformly “Fix” ends up with every location carrying the first location’s Fix.</li>
<li>Then NA → 0.</li>
</ul>
</li>
<li><strong>Case 3</strong> (L1143–1224): recompute with <code>ded_all_corr</code>. Its <code>ded_pol = n_distinct(c_across(Kind cols), na.rm=TRUE)</code> is evaluated <strong>per group, not per row</strong>: <code>rowwise()</code> is commented out at FDS:L153. <strong>Verify in R.</strong> It counts distinct kinds across all coverages and all locations of the policy.
<ul>
<li>Condition: <code>num == 1</code> and <code>ded_pol &gt; 1</code>.</li>
<li>Kind “Fix” → “% of Claims” (row by row; NA stays NA).</li>
<li>Percent → first row’s <code>pmax</code> over the 5 coverages. R’s <code>pmax</code> without <code>na.rm</code> returns NA when any input is NA; Spark’s <code>greatest</code> skips nulls.</li>
<li>Min → Σ rat_*.</li>
<li>Max → first row’s value.</li>
<li>Then NA → 0.</li>
<li><strong>[RISK]</strong> If Kind_j has both NA and “% of Claims” in the policy, <code>unique()</code> has length 2 and <code>ifelse</code> returns 2 values. dplyr then raises an error unless the policy has exactly 2 rows, in which case values are assigned row by row without that being intended.</li>
</ul>
</li>
<li><strong>Case 4a</strong> (L1228–1244): <code>ded_col_corr(Fix)</code>; the sum of <code>_num</code> &gt; 5 → Fix_j = <code>Mode_MM(Fix_j)</code> for all 5 coverages, over the whole policy.
<ul>
<li><code>Mode_MM</code>: the single mode if there is one; the minimum when there is no mode (DescTools returns NA when all values are unique; <strong>verify the DescTools version</strong>); the minimum of the modes on ties. Zeros count as values.</li>
</ul>
</li>
<li><strong>Case 4b and 4c</strong> (L1248–1318): Min (respectively Max) differs → for every coverage j with Kind not all NA, set Min_j = Σ over rows and coverages of (row coverage SI / total policy SI) × Min_j,row.
<ul>
<li>The same scalar is written to every such coverage.</li>
<li>Fix, Min and Max are left as NA (from <code>na_if</code>) until the end of the branch. Later detections then count NA as a distinct value.</li>
</ul>
</li>
<li><strong>Case 5a and 5b</strong> (L1326–1386): <code>ded_loc_corr</code> has <strong>no</strong> filter on per == “Policy”, so Location-level policies are included. <code>ded_pol</code> = distinct non-zero Min (or Max) values across coverages and rows. If it is above 1, apply the same SI-weighted formula as 4b.</li>
<li><strong>Sublimits</strong> (L1388–1481):
<ul>
<li>The same fill step on per and the 5 sublimit columns.</li>
<li><code>sublimit_corr</code> (FDS:L163) gives <code>num</code>, <code>_num0</code>, <code>_num</code>, <code>lim_diff</code> and <code>con_zero</code>.</li>
<li><strong>Case 2:</strong> num == 1, the sum of <code>_num</code> &gt; 5 and <code>lim_diff</code> → <code>Sublimit_per = "Location"</code> on the whole policy.</li>
<li><strong>Case 4:</strong> num == 1, sum &gt; 5, not <code>con_zero</code>, not <code>lim_diff</code> → <code>Mode_MM</code> on each sublimit.</li>
<li><strong>Case 5:</strong> AAL → <code>Mode_MM</code> when values differ. Ties and no-mode go to the minimum, which is FALSE.
<ul>
<li><strong>[FACT]</strong> The guard <code>length(case5) &gt; 0</code> counts the columns of a data frame, so it is always TRUE. This is harmless.</li>
<li><strong>[RISK]</strong> If AAL flips to TRUE here, the F5 “move to combined” rule is never re-applied.</li>
</ul>
</li>
</ul>
</li>
<li><strong>Merge</strong> (L1491–1493): the rows outside the scope plus the processed rows, sorted by seq_ID. Then <strong>every numeric NA in the whole dataset becomes 0</strong> (finding 6).</li>
</ol>
<ul>
<li><strong>PySpark:</strong> replace <code>foreach</code> with native grouped operations, partitioned by PolicyID:
<ul>
<li>fill down-up: <code>last(col, ignorenulls=True)</code> over (unboundedPreceding, currentRow) ordered by seq_ID, then <code>first(col, ignorenulls=True)</code> over (currentRow, unboundedFollowing).</li>
<li><code>n_distinct</code> that counts NA: <code>size(collect_set(col)) + max(col.isNull().cast("int"))</code>.</li>
<li>“first element” semantics: <code>first(col)</code> over a window ordered by seq_ID with an unbounded frame.</li>
<li>Mode_MM: aggregate value counts per (group, value), keep the maximum count, take the minimum of tied values, and fall back to the minimum when every count is 1. Join the result back.</li>
<li>SI weights: window sums per policy.</li>
<li><strong>[REC]</strong> Use <code>applyInPandas</code> per policy only if parity on the dplyr quirks cannot be reached with window functions. It also simplifies parity tests of the sequential case logic.</li>
</ul>
</li>
</ul>
<h3 id="s22-—-policy-harmonisation-branch-2-per-policyid-country-or-policyid-catnat_label-l1496–2065">S22 — Policy harmonisation, branch 2: per (PolicyID, Country) or (PolicyID, CATNAT_Label) (L1496–2065)</h3>
<p><strong>Entry [FACT]:</strong> (CATNAT, or (L != “” and L present in the data)) and the same “any Policy” test. For MUDE, if the data has both L and other LOBs, <strong>both branches run</strong>: branch 2 runs on the output of branch 1.</p>
<p><strong>Scope:</strong> the <code>group_id</code>s of rows with LOB ∈ L (all rows when CATNAT) and <strong><code>n &gt; 1</code></strong>. Here <code>n</code> is the <em>policy-level</em> count, so a single-row group inside a multi-location policy is still processed.</p>
<p>Differences from branch 1:</p>
<ul>
<li>Grouping is by <code>policy_term_group_vars_current</code>. The detection helpers are <code>ded_corr_cntr</code> and <code>sublimit_corr_cntr</code> (FDS:L46, L197), filtered by <code>policy_term_group_filter</code>.</li>
<li><strong>Case 2:</strong> <code>rat_*</code> uses <code>max(Min_j)</code> only. Branch 1 uses <code>max(Fix_j, Min_j)</code>.</li>
<li><strong>Case 4</strong> (L1784–1845) combines four detections (Min, Max, Percent and Fix differ). In one mutate:
<ul>
<li>Min and Max become SI-weighted.</li>
<li>Fix and Percent become <code>DescTools::Mode(...)</code>, <strong>not</strong> <code>Mode_MM</code>. Without <code>na.rm</code>, any NA gives NA, ties give several values (and <code>ifelse</code> keeps the first), and there is no minimum fallback.</li>
<li>The SI-weighted values use <code>n</code> and <code>lim_diff</code> from FDS, which compare group-level distinct counts with the <strong>policy-level</strong> <code>n</code>.</li>
</ul>
</li>
<li>There is no Case 5a or 5b (commented out).</li>
<li><strong>Sublimits:</strong>
<ul>
<li>Case 1 has an empty body.</li>
<li><strong>Case 2 uses <code>Mode_MM</code></strong> on the sublimits (branch 1 sets per = “Location”).</li>
<li>Case 4 uses <code>Mode_MM</code>. Case 5: AAL mode.</li>
</ul>
</li>
<li>After every peril, all numeric NA in the group → 0 (L1957). After the merge, all numeric NA in the whole dataset → 0 (L1969).</li>
</ul>
<p><strong>Re-application</strong> (L1980–2064, still <strong>inside</strong> the branch-2 <code>if</code>):</p>
<ul>
<li>Deductibles: A3 (granularity) and A4 (Cover FALSE), then B1–B3 (Kind from values; Min and Max only for “% of Claims”).</li>
<li>Sublimits: F3 and F4.</li>
<li><strong>[FACT]</strong> Not re-applied: A1/A2 (deriving per None or Policy), B4 (Location-level with SI = 0), F1/F2, F5 (AAL), G, S18 and S19. The S17 SI helper columns are not rebuilt; they are not needed for these rules.</li>
</ul>
<p><strong>PySpark:</strong> the same building blocks as S21, partitioned by the group keys. Re-application reuses the S17 and S20 rule functions with a list of which rules to run.</p>
<h3 id="s23-—-yearbuilt-l2068–2071">S23 — YearBuilt (L2068–2071)</h3>
<ul>
<li><strong>[FACT]</strong> Convert to numeric, with invalid values becoming NA. Set YearBuilt to NA when it is NA after conversion, 0, 9999, &lt; 1000 or &gt; c_year. The original type is kept: a character column stays character.</li>
<li><strong>[FACT]</strong> If an earlier step had turned NA into 0 (S18, S21, S22), it becomes NA again here.</li>
<li><strong>PySpark:</strong> <code>try_cast(YearBuilt as double)</code>. Spark in ANSI mode raises an error on a plain invalid cast. Then <code>when(invalid, null)</code>, comparing against c_year as an integer.</li>
</ul>
<h3 id="s24-—-negative-si-l2073–2075">S24 — Negative SI (L2073–2075)</h3>
<ul>
<li><strong>[FACT]</strong> Every <code>SI_*</code> column below 0 → abs().</li>
<li><strong>[RISK]</strong> This runs <strong>after</strong> all the SI-weighted calculations and SI == 0 tests in S17–S22, so those steps used the negative values.</li>
<li><strong>[ASSUMPTION]</strong> All <code>SI_*</code> columns are numeric. A character <code>SI_*</code> column would be compared as text and then fail in abs().</li>
<li><strong>PySpark:</strong> <code>when(col &lt; 0, abs(col))</code> over the columns with the <code>SI_</code> prefix.</li>
</ul>
<h3 id="s25-—-integrity-check-and-temporary-column-removal-l2081–2088">S25 — Integrity check and temporary column removal (L2081–2088)</h3>
<ul>
<li><strong>[FACT]</strong> If the row count changed since S2, a <code>warning</code> is raised but the run continues. Then the mapping attribute columns are dropped and the result is converted back to a data.frame.</li>
<li><strong>PySpark:</strong> assert the counts are equal, and log or raise; then <code>drop(*mapping_attr_cols)</code>.</li>
</ul>
<h3 id="s26-—-change-log-l2090–2110">S26 — Change log (L2090–2110)</h3>
<ul>
<li><strong>[FACT]</strong> For each column <code>i</code> of <code>exposure[, !empty_columns]</code>, find the rows where <code>!identical(new, old)</code>, comparing row by row with <code>exposure_org</code> from S2.
<ul>
<li>For geo columns, the output is PolicyID + all 6 geo columns (new values) + <code>&lt;i&gt;_old</code>.</li>
<li>For other columns, the output is PolicyID + <code>i</code> + <code>&lt;i&gt;_old</code>.</li>
<li>Text is converted to ISO-8859-1 and written with <code>write.csv</code>, row names included.</li>
</ul>
</li>
<li><strong>[RISK]</strong> <code>empty_columns</code> was computed on the input columns, but the final frame has 3 more (<code>seq_ID</code>, <code>n</code>, <code>group_id</code>). The logical index is shorter than the column list and is recycled, so whether those 3 columns are logged depends on the first 3 flags. <strong>Verify in R.</strong></li>
<li><strong>[RISK]</strong> <code>identical()</code> also compares types. Columns whose type changed (PostalCode in S5, integer → double in S18 or the NA→0 merges) show every row as changed.</li>
<li><strong>[FACT]</strong> S1 changes are not in the log. Rows later removed in S27 are still in the log.</li>
<li><strong>PySpark:</strong> join the final and snapshot DataFrames on seq_ID. For each column, filter <code>NOT (new &lt;=&gt; old)</code> (null-safe) and write to CSV with ISO-8859-1 encoding. Decide whether type changes count as changes (Q-O3).</li>
</ul>
<h3 id="s27-—-row-removals-l2113–2171-in-this-order">S27 — Row removals (L2113–2171), in this order</h3>
<ol>
<li><strong>Only when <code>c_year == "2023"</code>:</strong> <code>Fac == 1</code> (after S15) → write <code>Removed_100Fac.csv</code> and remove the rows. Sux removal is commented out.</li>
<li>Sum of all <code>SI_*</code> == 0 → <code>Removed_TSI0.csv</code>, remove.</li>
<li>Country in <code>sanctioned_string</code> (AF, BY, MM, CD, KP, IQ, IR, YE, CU, LY, RU, ZW, SO, SD, SS, SY, UA, VE, CF; “per 21.02.2025”) → <code>Removed_Sanctioned.csv</code>, remove.</li>
</ol>
<ul>
<li><strong>[FACT]</strong> A row that matches several rules appears only in the file of the first rule that removes it.</li>
<li><strong>[RISK]</strong> In rules 1 and 2, <code>exposure[!cond, ]</code> with NA in <code>cond</code> (Fac NA, or any SI NA without na.rm) produces <strong>all-NA rows</strong> in R instead of keeping or removing the row. Whether this can happen depends on whether the NA→0 steps ran earlier.</li>
<li><strong>PySpark:</strong> <code>filter(cond)</code> for the removed set and <code>filter(~cond | cond.isNull())</code> for the kept set, plus a decision on NA handling (Q-O4). Write the CSVs before filtering.</li>
</ul>
<h3 id="s28-—-clean-up-l2174–2177">S28 — Clean-up (L2174–2177)</h3>
<p>Removes temporary objects from the R environment. There is no PySpark equivalent beyond <code>unpersist()</code> on cached DataFrames.</p>
<hr>
<h2 id="mapping-of-r-operations-to-pyspark">4. Mapping of R operations to PySpark</h2>

<table>
<thead>
<tr>
<th>R construct (where)</th>
<th>Semantics to preserve</th>
<th>PySpark equivalent (conceptual)</th>
</tr>
</thead>
<tbody>
<tr>
<td><code>x[cond] &lt;- v</code> (everywhere)</td>
<td>Sequential and in place; an NA mask skips the row</td>
<td>Chained <code>withColumn(c, when(cond, v).otherwise(c))</code>; a null condition falls through</td>
</tr>
<tr>
<td><code>df[cond, cols] &lt;- v</code> with NA in <code>cond</code> (S17, S20)</td>
<td>R raises an error</td>
<td>Pre-validate that there are no nulls; raise to match R</td>
</tr>
<tr>
<td><code>left_join</code> (L36, L282, L345)</td>
<td>Many-to-one assumed; a duplicate key multiplies rows</td>
<td><code>join(broadcast(map), keys, "left")</code> plus a uniqueness assertion</td>
</tr>
<tr>
<td><code>%in%</code></td>
<td>Never NA</td>
<td><code>coalesce(col.isin(list), lit(False))</code></td>
</tr>
<tr>
<td><code>is.na</code></td>
<td>TRUE for both NA and NaN</td>
<td><code>isNull() \| isnan()</code> on doubles</td>
</tr>
<tr>
<td><code>str_detect</code> / <code>grepl</code> (ICU / TRE)</td>
<td><code>(?i)</code>, <code>\b</code>, POSIX classes</td>
<td><code>rlike</code> (Java regex); translate <code>[:alpha:]</code> to <code>\p{Alpha}</code>; consider <code>(?U)</code> for Unicode <code>\b</code></td>
</tr>
<tr>
<td><code>str_detect(x, &lt;column Format&gt;)</code></td>
<td>Pattern differs per row (per country)</td>
<td><code>expr("x rlike Format")</code>, or a loop over the distinct countries</td>
</tr>
<tr>
<td><code>str_extract</code></td>
<td>NA when there is no match</td>
<td><code>regexp_extract(...)</code>; convert <code>""</code> to null</td>
</tr>
<tr>
<td><code>str_replace</code> (first match only)</td>
<td>Only the first occurrence</td>
<td>Custom: extract, then replace at the first position (or a small UDF)</td>
</tr>
<tr>
<td><code>gsub</code></td>
<td>Every occurrence</td>
<td><code>regexp_replace</code></td>
</tr>
<tr>
<td><code>str_pad(left, "0")</code></td>
<td>Only called when the value is shorter</td>
<td><code>lpad</code> inside the same guard (<code>lpad</code> truncates longer strings)</td>
</tr>
<tr>
<td><code>nchar(x, keepNA=FALSE)</code></td>
<td>NA → 2</td>
<td><code>length()</code> (null → null); the code always guards with <code>!is.na</code></td>
</tr>
<tr>
<td><code>substr(x, nchar-2, nchar)</code></td>
<td>The last 3 characters</td>
<td><code>substring(x, -3, 3)</code></td>
</tr>
<tr>
<td><code>trimws(whitespace=w)</code> / <code>str_squish</code></td>
<td>Regex <code>^w+</code> / <code>w+$</code>; collapse whitespace</td>
<td><code>regexp_replace</code> with the same regexes; <code>trim</code></td>
</tr>
<tr>
<td><code>mapply(identical, a, b)</code></td>
<td>Exact, including type</td>
<td>Null-safe <code>&lt;=&gt;</code>, plus a type check if needed</td>
</tr>
<tr>
<td><code>mapply(grepl, Street, City, fixed=TRUE)</code></td>
<td>Literal substring</td>
<td><code>instr(City, Street) &gt; 0</code></td>
</tr>
<tr>
<td><code>group_by + add_count</code></td>
<td>Count per group</td>
<td><code>count("*").over(Window.partitionBy(...))</code></td>
</tr>
<tr>
<td><code>cur_group_id()</code></td>
<td>Rank in sorted key order</td>
<td><code>dense_rank().over(orderBy(keys asc_nulls_last))</code></td>
</tr>
<tr>
<td><code>seq.int(nrow)</code></td>
<td>Row order of the input</td>
<td><code>row_number()</code> over a persisted ingestion index</td>
</tr>
<tr>
<td><code>n_distinct(x)</code> (NA counted)</td>
<td>Includes NA</td>
<td><code>size(collect_set(x)) + max(x.isNull().cast("int"))</code> over the window</td>
</tr>
<tr>
<td><code>n_distinct(x, na.rm=TRUE)</code></td>
<td>Excludes NA</td>
<td><code>size(collect_set(x))</code></td>
</tr>
<tr>
<td><code>fill(.direction="downup")</code></td>
<td>Order-dependent</td>
<td><code>last(ignorenulls)</code> over preceding rows, then <code>first(ignorenulls)</code> over following rows, ordered by seq_ID</td>
</tr>
<tr>
<td><code>ifelse(&lt;scalar&gt;, a, &lt;vector&gt;)</code> inside a grouped mutate</td>
<td>Returns <code>vector[1]</code>, broadcast to the group</td>
<td><code>first(col).over(w ordered by seq_ID, unbounded frame)</code></td>
</tr>
<tr>
<td><code>Mode_MM()</code></td>
<td>Mode; no mode → min; tie → min</td>
<td>Value counts per group → the maximum count → the minimum of tied values (or the minimum of all when every count is 1) → join back</td>
</tr>
<tr>
<td><code>DescTools::Mode()</code> (na.rm=FALSE)</td>
<td>Any NA → NA; ties → several values</td>
<td>Same aggregation; null if the group has any null; take the first or minimum tie (Q-D7)</td>
</tr>
<tr>
<td><code>max(cbind(...), na.rm=TRUE)</code> (F5)</td>
<td>One global scalar</td>
<td><code>agg(max(greatest(...)))</code> → literal or cross join</td>
</tr>
<tr>
<td><code>pmax(...)</code> (no na.rm)</td>
<td>NA if any value is NA</td>
<td><code>when(any null, null).otherwise(greatest(...))</code> (<code>greatest</code> skips nulls)</td>
</tr>
<tr>
<td><code>rowSums(cbind(...))</code> / <code>rowSums(..., na.rm=T)</code></td>
<td>NA propagates / NAs ignored</td>
<td><code>a + b + c</code> / <code>coalesce(a,0) + …</code></td>
</tr>
<tr>
<td><code>apply(..., 1, unique(na.omit(x)))</code></td>
<td>Distinct non-NA values per row</td>
<td><code>array_distinct(filter(array(cols), x -&gt; x is not null))</code></td>
</tr>
<tr>
<td><code>foreach %dopar%</code> + <code>rbind</code> + <code>arrange(seq_ID)</code></td>
<td>Parallel processing per group</td>
<td>Native window and group operations (or <code>applyInPandas</code>) + <code>unionByName</code> + <code>orderBy("seq_ID")</code></td>
</tr>
<tr>
<td><code>mutate(across(where(is.numeric), ~ifelse(is.na(.),0,.)))</code></td>
<td>Every numeric column</td>
<td><code>fillna(0, subset=numeric_cols)</code> (plus a NaN check)</td>
</tr>
<tr>
<td><code>as.numeric()</code> with suppressed warnings</td>
<td>Invalid → NA</td>
<td><code>try_cast</code> / <code>expr("try_cast(x as double)")</code></td>
</tr>
<tr>
<td><code>stop()</code> / <code>warning()</code></td>
<td>Stop the run / log and continue</td>
<td><code>raise</code> / <code>logger.warning</code></td>
</tr>
<tr>
<td><code>write.csv</code> (row names, ISO-8859-1)</td>
<td>One file per output</td>
<td><code>coalesce(1).write.option("encoding","ISO-8859-1").csv(...)</code>; add a row-number column if row names must match</td>
</tr>
</tbody>
</table><hr>
<h2 id="r-code-compared-with-the-wiki">5. R code compared with the wiki</h2>
<h3 id="rules-that-are-documented-in-the-wiki-and-match-the-code">5.1 Rules that are documented in the wiki and match the code</h3>
<p>5.1 defaults (NA/"??" → MU default); the junk, coordinate and “NA” removal; the MUAT placeholder; 5.2 MUFR names, FL→LI, UK→GB, the city list, HONGRIE, 98xxx→MC, Richmond/London/Bristol, CH 9485–9499→LI, the overseas list; 5.3 "0"→NA, the CH mapping, GB/US two-letter zones, the ISO3 prefix, LI/HK/MC fixed values; 5.4 CH 9999, the StreetNo/City extraction, the swaps, Single, padding, STOP on a missing country, GB/CA O/I, the Street/StreetNo NA rules; 5.5 percentages; 5.6 None/Policy derivation, granularity, Cover FALSE, Kind from values, Min/Max cleared, Location-level SI = 0 clearing with a Policy-level warning, the single-location conversion; 5.7 the same pattern for sublimits plus the AAL consolidation; 5.8 the case table in general terms; 5.9 YearBuilt, negative SI, seq_ID/n/group_id, change log; 5.10 the three removals and the 2023-only Fac rule.</p>
<h3 id="rules-only-in-the-code-inferred-not-documented">5.2 Rules only in the code (inferred, not documented)</h3>
<ul>
<li>The exceptions to the “diversi” rule (IT City; Street “de la Diversit”) and the fact that they protect the whole row.</li>
<li><code>values_string</code> exact matches also apply to Country (so “Na” and “na” become NA).</li>
<li>The Milford override order (GB wins) and Litija (SI wins); the Namibia “NA” code.</li>
<li>The CH CRESTA gate <code>&gt; 1</code> and the side effect that CH CRESTA becomes NA when there is no match.</li>
<li>Street contained in City → Street NA (L355).</li>
<li>The per-country loop’s C0 rule (StreetNo holding a street name when Street is NA) and the C2a swap.</li>
<li>LengthMax padding.</li>
<li>The extra Street-pattern removals (L470–473).</li>
<li><strong>The Min &gt; Max swap (S19).</strong></li>
<li>Combined is kept over Com when both are present; BI is ignored in the Com-clearing test.</li>
<li>The SI helper columns and their NA propagation.</li>
<li>All numeric NA → 0 for single-location rows and after harmonisation.</li>
<li>The harmonisation helpers’ detection logic (<code>num</code>, <code>_num0</code>, <code>lim_diff</code>, <code>con_zero</code>, <code>ded_pol</code>).</li>
<li><code>Mode_MM</code> falls back to the minimum.</li>
<li>The <code>n &gt; 1</code> scope.</li>
<li>Branch-1 Case 2 sets per = “Location”; branch-2 Case 2 uses the mode.</li>
<li>Branch-2 Case 4 also takes the mode of Percent.</li>
<li>The change log excludes columns that were empty in the input.</li>
<li>The type-sensitive comparison in the change log.</li>
</ul>
<h3 id="contradictions-between-the-wiki-and-the-code">5.3 Contradictions between the wiki and the code</h3>

<table>
<thead>
<tr>
<th>#</th>
<th>Wiki says</th>
<th>Code does</th>
<th>Lines</th>
</tr>
</thead>
<tbody>
<tr>
<td>W1</td>
<td>5.8 runs whenever Policy-level values exist, per policy (or per policy/country for LOBs in LOB_list)</td>
<td>Runs only when <code>LOB_list[[MU]]$LOB != ""</code> (MUCH, MUDE) or for MUFR CATNAT</td>
<td>L918–927, L1496–1504</td>
</tr>
<tr>
<td>W2</td>
<td>“Afterwards the per-peril rules (5.6/5.7) are re-applied”</td>
<td>Only part of the rules, and only inside branch 2</td>
<td>L1980–2064</td>
</tr>
<tr>
<td>W3</td>
<td>Overseas 984 TF, 986 WF, 987 PF, 988 NC</td>
<td>Cannot happen; L258 turns FR 98xxx into MC first</td>
<td>L258 vs L303–306</td>
</tr>
<tr>
<td>W4</td>
<td>AAL: the largest value moves to the combined column</td>
<td>The largest value <strong>in the whole dataset</strong></td>
<td>L844, L913</td>
</tr>
<tr>
<td>W5</td>
<td>Final trim of “commas, dots, brackets and whitespace”</td>
<td>Dots not trimmed; brackets only as a literal “()” sequence</td>
<td>L484–489</td>
</tr>
<tr>
<td>W6</td>
<td>“PostalCode = City or Street = City → NA”</td>
<td>Also Street that is a substring of City</td>
<td>L355</td>
</tr>
<tr>
<td>W7</td>
<td>“Street with numbers only → NA”</td>
<td>Also “12a”, “A 12”, “A12” and the <code>[0.9]</code> typo pattern</td>
<td>L469–473</td>
</tr>
<tr>
<td>W8</td>
<td>“Combined vs Com vs B/C/BI: the less granular one is cleared”</td>
<td>When Combined and Com coexist, the more granular Com is cleared</td>
<td>L559, L563, …</td>
</tr>
<tr>
<td>W9</td>
<td>5.9 lists seq_ID/n/group_id under “Other corrections” (after sublimits)</td>
<td>Created before the deductibles; they drive S18, S21 and S22</td>
<td>L513–526</td>
</tr>
<tr>
<td>W10</td>
<td>5.9 “Change log per modified column”</td>
<td>Changes from S1 defaults are not logged; type changes log every row; columns empty in the input are excluded</td>
<td>L39, L2091</td>
</tr>
<tr>
<td>W11</td>
<td>Checkpoint <code>_temp2.RData</code></td>
<td>Written by the caller scripts, not by DataCleaning.R</td>
<td>e.g. <code>01_MUCH_EQ.R:557</code></td>
</tr>
<tr>
<td>W12</td>
<td>5.8 Case 1: “Fill from the other locations”</td>
<td>Fills from the previous location, then the next (seq order); applies to Location-level policies too</td>
<td>L1037–1042</td>
</tr>
<tr>
<td>W13</td>
<td>5.8 Case 4a “Mode”</td>
<td>Branch 1: <code>Mode_MM</code> (minimum fallback); branch 2: <code>DescTools::Mode</code> (no fallback, NA-sensitive)</td>
<td>L1238, L1833</td>
</tr>
<tr>
<td>W14</td>
<td>5.2 lists the overseas split as a country rule</td>
<td>It runs after the CRESTA step, so those rows keep <code>FRA_</code> CRESTA prefixes</td>
<td>L297–308</td>
</tr>
</tbody>
</table><h3 id="missing-information">5.4 Missing information</h3>
<ul>
<li><strong>Q4, Q5, Q6 and Q7 are referenced but not defined anywhere available.</strong></li>
<li>No definition of the input schema, column types or the upstream null policy (the code depends on numeric deductible columns having no NA).</li>
<li>No reference to the <code>[OTHER TEAM]</code> items (Address Normalization, Migrated Policy Files) and how they will split responsibility.</li>
<li>No documentation of the S19 swap, the S18 NA→0 side effect, <code>Mode_MM</code>, the harmonisation helper semantics, or the partial reuse of the geo block by BABE_Prop.</li>
<li>The wiki does not say which version of Mapping_PostalCode / Mapping_CRESTA_CH applies.</li>
</ul>
<hr>
<h2 id="unresolved-questions-and-assumptions-to-validate-before-migration">6. Unresolved questions and assumptions to validate before migration</h2>
<h3 id="questions-for-the-business-or-data-owner">6.1 Questions for the business or data owner</h3>
<ul>
<li><strong>Q-G1</strong> Is the CRESTA prefix supposed to use the <strong>corrected</strong> country (S7 uses the stale S2 ISO3)? For MUFR, do the current outputs contain <code>NA_…</code> CRESTA codes?</li>
<li><strong>Q-G2</strong> S4 (MUAT placeholder): should it be MUAT-only? Should it be fixed to an exact four-field match with NA as no-match (the <code>na.omit</code> misalignment)?</li>
<li><strong>Q-G3</strong> Should FR 984/986/987/988 map to TF/WF/PF/NC (wiki) or MC (current code)?</li>
<li><strong>Q-G4</strong> CH CRESTA mapping: gate <code>&gt; 1</code> or <code>&gt; 0</code>? Should CH rows that already hold <code>CHE_xx</code> codes, or unpadded zones, keep their value instead of becoming NA?</li>
<li><strong>Q-G5</strong> The postal loop’s stale flags (C1+C1a with City NA; C2a followed by C2): keep the current output or implement the intended swap?</li>
<li><strong>Q-G6</strong> Should non-numeric postal codes in numeric-format countries become “0000…” (current) or NA?</li>
<li><strong>Q-G7</strong> GB O/I replacement across the whole code: keep, or limit to the inward-code digit position?</li>
<li><strong>Q-G8</strong> The <code>[0.9]</code> Street pattern: typo for <code>[0-9]</code>?</li>
<li><strong>Q-G9</strong> Should S14 actually trim dots and single brackets, as the wiki says?</li>
<li><strong>Q-G10</strong> Milford: should it be IE or GB? (Today all of them become GB.)</li>
<li><strong>Q-P1</strong> Percentages (wiki Q5): is 1 &lt; x &lt; 2 → 100% correct? Are values above 100 an error?</li>
<li><strong>Q-D1</strong> <strong>Should 5.8 run for MUs with an empty LOB (BABE, BACH, BADE, BALU, MUAT, MUIT, MUES, MUFR non-CATNAT)?</strong> This is the biggest scope decision.</li>
<li><strong>Q-D2</strong> Should the “re-apply 5.6/5.7” step run after both branches, with the complete rule set (A1/A2, B4, F1/F2, F5, S19)?</li>
<li><strong>Q-D3</strong> AAL consolidation: row maximum (intended?) or global maximum (current)?</li>
<li><strong>Q-D4</strong> S18: should the conversion use per-row flags instead of the global <code>fix[[j]]</code>, and per-row Max instead of <code>max_values[1]</code>?</li>
<li><strong>Q-D5</strong> S18, S21 and S22: should numeric NA → 0 really apply to non-deductible columns (lat/lon, SI, YearBuilt)?</li>
<li><strong>Q-D6</strong> S19: when Max = 0, meaning “no max”, should the swap be skipped?</li>
<li><strong>Q-D7</strong> Branch 2 Case 4: <code>DescTools::Mode</code> vs <code>Mode_MM</code>, and how to break ties?</li>
<li><strong>Q-D8</strong> Branch 1 vs branch 2 Case 2 SI weighting: <code>max(Fix, Min)</code> vs <code>max(Min)</code>. Intended difference?</li>
<li><strong>Q-D9</strong> Should the Case 1 fill also apply to Location-level policies and to rows cleared earlier (Cover FALSE, SI = 0)?</li>
<li><strong>Q-D10</strong> Should the SI-weighted Min/Max use absolute SI values (S24 runs after them)?</li>
<li><strong>Q-O1</strong> The 100% Fac removal is limited to <code>c_year == "2023"</code>. Is it now permanently inactive, or should it be configurable?</li>
<li><strong>Q-O2</strong> Is the sanctioned list (dated 21.02.2025) still current? Who owns it?</li>
<li><strong>Q-O3</strong> Change log: log type-only changes? Include S1 defaults? Include seq_ID, n and group_id?</li>
<li><strong>Q-O4</strong> Removal rules with NA Fac or NA SI: keep, remove, or fail? (R currently creates all-NA rows.)</li>
<li><strong>Q-W</strong> What are wiki Q4, Q5, Q6 and Q7?</li>
</ul>
<h3 id="technical-assumptions-to-verify">6.2 Technical assumptions to verify</h3>
<ul>
<li><strong>A1</strong> <code>core</code> = <code>GRI/core/</code>, <code>functions</code> = <code>GRI/functions/</code> (non-archive <code>f_deductibles_sublimits.R</code>).</li>
<li><strong>A2</strong> The <code>mappers/*.csv</code> files match <code>PostalCode.xlsx</code> / <code>CRESTACode.xlsx</code> (sheet <code>Schweiz</code>), including key uniqueness and the <code>ZoneName</code> header.</li>
<li><strong>A3</strong> The numeric deductible and sublimit columns, <code>P_Cover</code> and <code>P_AAL</code> never contain NA on input.</li>
<li><strong>A4</strong> <code>SI_Building … SI_BI</code> are adjacent in the column order (FDS uses the range <code>SI_Building:SI_BI</code>), and every <code>SI_*</code> column is numeric.</li>
<li><strong>A5</strong> All 8 columns <code>EQ|WS|FL|HA _Deductible_per/_Sublimit_per</code> exist even when a peril is not modelled. If not, the harmonisation gate silently evaluates to FALSE.</li>
<li><strong>A6</strong> dplyr, tidyr and DescTools versions. These decide the <code>cur_group_id</code> sort locale, <code>c_across</code> without <code>rowwise</code>, and <code>Mode</code> returning NA for all-unique input.</li>
<li><strong>A7</strong> The input row order is stable and meaningful (it drives <code>seq_ID</code>, fill order and “first row” values).</li>
<li><strong>A8</strong> <code>MU</code> is always a key of <code>Country_list</code> and <code>LOB_list</code>.</li>
<li><strong>A9</strong> <code>Namibia = "NA"</code> survives every I/O step in the target platform.</li>
<li><strong>A10</strong> The exact R behaviour of: <code>na.omit</code> mask assignment (L73), the logical-matrix assignment with NA (L701), <code>apply</code> returning a list (L714–726), the recycled <code>empty_columns</code> index (L2091), and <code>exposure[!NA, ]</code> (S27). All of these must be checked against R before choosing parity.</li>
</ul>
<hr>
<h2 id="proposed-sequence-for-implementing-and-testing-the-pyspark-version">7. Proposed sequence for implementing and testing the PySpark version</h2>
<p><strong>Phase 0 — Decisions and the reference baseline</strong> (before writing code)</p>
<ol>
<li>Resolve §6.1, at least Q-D1, Q-D2, Q-D3, Q-G1, Q-G3 and Q-D5. For each [RISK] item, decide between <strong>parity mode</strong> (bug-for-bug) and <strong>corrected mode</strong>. <strong>[REC]</strong> Build parity mode first behind feature flags, so you can prove equivalence before changing behaviour.</li>
<li>Produce golden data. Run the unmodified R pipeline on representative extracts (at least MUCH, MUDE with mixed LOBs, MUFR Prop CATNAT, MUFR non-CATNAT, BABE Prop geo-only, and one MU with an empty LOB). Capture <code>exposure</code> at the step boundaries S2, S14, S16, S20, S22 and S27 plus all CSV outputs. Do this with a copy of the script or a wrapper; the original stays untouched.</li>
<li>Export the mapping xlsx files as versioned tables and validate key uniqueness (A2).</li>
</ol>
<p><strong>Phase 1 — Foundations</strong><br>
4. A config module: MU tables, cat_peril/cat_ded, string lists, explicit c_year.<br>
5. The schema contract: a typed input schema (PostalCode as string, Cover/AAL boolean), the ingestion row index → <code>seq_ID</code>, and null-policy assertions (A3, A4, A5).<br>
6. Column-name generation for peril × coverage, matching the R <code>gsub</code> naming.</p>
<p><strong>Phase 2 — Geo block (S1–S14)</strong>, as a standalone function, because BABE_Prop uses it alone<br>
7. S1–S5, then S6 (ordered rule tables), S7 + S8 (with the frozen S2 ISO3 in parity mode), S9.<br>
8. S10–S12 (flags computed from pre-loop values, then ordered actions), S13, S14.<br>
9. Unit tests, one per rule, with crafted rows, including every edge case in §3: Milford, Litija, Namibia “NA”, the London NA postal code, FR 98xxx, a single CH numeric CRESTA row, the C1+C1a and C2a+C2 interactions, a non-numeric postal code being padded, the GB <code>OL1OAB</code> case, US ZIP+4, PT 6-digit codes, the <code>()</code> trim, and the “diversi” exception protecting a whole row.<br>
10. Parity test against the S14 snapshot: a null-safe comparison per column, keyed by seq_ID, reporting mismatch counts per column.</p>
<p><strong>Phase 3 — Percentages and IDs (S15–S16)</strong><br>
11. Implement them and run parity against S16, including the <code>group_id</code> ordering if downstream steps depend on the exact values.</p>
<p><strong>Phase 4 — Location-level rules (S17–S20)</strong><br>
12. S17 blocks A/B/C, S18 (the global flags and first-element semantics in parity mode), S19, S20 F/G/H (global AAL max in parity mode).<br>
13. Unit tests: Combined + Com coexisting, BI present with Com, Fix and Percent both &gt; 0, Cover FALSE, SI = 0 with Location vs Policy level, single-location mixed kinds over several rows, Min &gt; Max with Max = 0, AAL rows.<br>
14. Parity against the S20 snapshot.</p>
<p><strong>Phase 5 — Policy harmonisation (S21–S22)</strong><br>
15. Building blocks first, each unit-tested against small R-computed fixtures: NA-aware <code>n_distinct</code>, down-up fill, Mode_MM / DescTools Mode, the first-row broadcast, SI weights, and the FDS detection flags (<code>num</code>, <code>_num0</code>, <code>_num</code>, <code>lim_diff</code>, <code>con_zero</code>, <code>ded_pol</code> at group level).<br>
16. Branch 1 cases in R order, then branch 2 cases, then the re-application. Respect the exact entry conditions (Q-D1).<br>
17. Parity against S22 with policies from the R comments (for example “CH 4001047110”, “CH 4001531954”, “CH 0270850009”, “CH 2 141”, “CH 4000391568”), where they appear in the extracts.<br>
18. Performance: check skew on large policies, partition by PolicyID or group_id, and avoid UDFs where window functions are enough.</p>
<p><strong>Phase 6 — Final steps (S23–S27)</strong><br>
19. YearBuilt (with c_year passed as a parameter), SI abs, the integrity check, temporary column removal, the change log (the agreed definition from Q-O3), and the removals in their original order with the agreed NA behaviour (Q-O4). The outputs are CSVs with matching names, columns and encoding.<br>
20. End-to-end parity per MU extract: final row count, removed-row sets, change-log files, and a column-by-column checksum.</p>
<p><strong>Phase 7 — Corrected mode (optional)</strong><br>
21. Turn on the fixes approved in Phase 0 one at a time. For each one, document the expected differences from R and get business sign-off on them.</p>
<p><strong>[REC]</strong> The repo already has Python and PySpark work for country codes, CRESTA and postal codes (<code>country_code/</code>, <code>cresta_code/</code>, <code>cresta_std/</code>). Check it against §3 S6–S12 (especially findings 4 and 5, and the postal-loop stale flags) before reusing it.</p>

