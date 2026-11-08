---


---

<h1 id="tsimport.r-—-step-by-step-reference-with-pandas-equivalents">TSImport.R — Step-by-Step Reference (with pandas equivalents)</h1>
<p>This documents <code>TSImport.R</code> in execution order: what each chunk of R does, and an<br>
idiomatic pandas translation. <code>exposure</code> is the input DataFrame; <code>location_df</code>,<br>
<code>contract_df</code>, <code>reinsurance_df</code> are the pandas equivalents of <code>location.df</code>,<br>
<code>contract.df</code>, <code>reinsurance.df</code>.</p>
<h2 id="conventions--things-not-defined-in-this-file">Conventions / things not defined in this file</h2>
<p>The script assumes a caller environment (populated by <code>source(Standards.R)</code> at<br>
line 6, or set up before this script runs) that provides:</p>
<ul>
<li><code>cat_peril_list</code> — dict/list keyed by peril, each entry carrying <code>peril</code> (R column<br>
prefix, e.g. <code>"EQ"</code>), <code>cat</code> (numeric suffix used in <code>LocPerils-&lt;cat&gt;</code> etc.), and<br>
<code>TSperil</code> (output peril code, e.g. <code>"PES"</code>).</li>
<li><code>LOB_list</code> — list of valid LOBs per MU, indexed by <code>MU</code>.</li>
<li><code>MU</code>, <code>sub</code> — current modeling-unit / sub-segment being processed (drive many<br>
hardcoded exception branches).</li>
<li><code>c_year</code>, <code>write_coord</code>, <code>core</code>, <code>template</code>, <code>TSheader</code>, <code>perils</code>, <code>cat_ded</code>,<br>
<code>terms</code>, <code>bbox</code>, <code>bbox_coord</code>, <code>write_output</code>, <code>output</code>.</li>
<li><code>Mode_MM</code> — referenced only inside commented-out (disabled) code; never defined<br>
in this file.</li>
<li>An already-open ODBC connection, closed at the very end (<code>odbcCloseAll()</code>).</li>
</ul>
<p>None of these are declared in <code>TSImport.R</code> itself — treat them as external<br>
configuration/environment inputs when porting.</p>
<p><code>ded_kind</code>, <code>ded_fix</code>, <code>ded_perc</code>, <code>ded_min</code>, <code>ded_max</code>, <code>sub_cat</code> are built as<br>
5-element (or 2-element for <code>sub_cat</code>'s first two slots) string vectors via<br>
<code>paste0</code>/<code>gsub</code>, then indexed positionally (<code>[[1]]</code>…<code>[[5]]</code>). The positional<br>
meaning (site / combined / building / contents / time-element) is inferred from<br>
naming convention and usage, not documented in-line in the R code.</p>
<p>Large commented-out blocks implement <strong>WS as Peril 1</strong> (location fields, lines<br>
~127–343), <strong>FL as Peril 3</strong> (location fields, lines ~618–1017) and <strong>HA as<br>
Peril 4</strong> (location fields, lines ~819–1017), plus legacy commented-out<br>
“CONTRACT TYPE 1/2/3” blanket-contract and EQ-specific contract blocks<br>
(lines ~1024–1112, ~2032–2285). These mirror the active EQ logic almost<br>
exactly and are disabled/future-work templates — not re-documented row by row<br>
below.</p>
<hr>
<h2 id="stage-1-—-setup--normalization-l1–56">Stage 1 — Setup &amp; Normalization (L1–56)</h2>

<table>
<thead>
<tr>
<th>Step</th>
<th>R Logic</th>
<th>Equivalent Python (pandas) snippet</th>
</tr>
</thead>
<tbody>
<tr>
<td>1.1 (L5–7)</td>
<td>Sources external <code>Standards.R</code>, injecting all the globals listed above into the environment. Nothing is defined locally.</td>
<td><code>from standards import (cat_peril_list, LOB_list, MU, sub, c_year, write_coord, core, template, TSheader, perils, cat_ded, terms, bbox, bbox_coord, write_output, output) # external config, not part of this module</code></td>
</tr>
<tr>
<td>1.2 (L13)</td>
<td><code>safe_max</code>: row-wise max that returns <code>NA</code> instead of <code>-Inf</code>/a warning when every value in the row is <code>NA</code>.</td>
<td><code># pandas .max(axis=1, skipna=True) already returns NaN for an all-NaN row</code><br><code>def safe_max(df_cols): return df_cols.max(axis=1, skipna=True)</code></td>
</tr>
<tr>
<td>1.3 (L17–20)</td>
<td>Normalizes <code>exposure</code>: for every numeric column, 0 → <code>NA</code>, otherwise round the value to a number of decimal places equal to the character-length of the value itself (mechanical description — intent unclear from code/comments); for every string column, <code>NA</code> → <code>""</code>.</td>
<td><code>num_cols = exposure.select_dtypes("number").columns</code><br><code>def _round_dyn(s):</code><br><code>n = s.astype(str).str.len(); return (s * 10**n).round() / 10**n</code><br><code>exposure[num_cols] = exposure[num_cols].mask(exposure[num_cols].eq(0)).apply(_round_dyn)</code><br><code>str_cols = exposure.select_dtypes("object").columns</code><br><code>exposure[str_cols] = exposure[str_cols].fillna("")</code></td>
</tr>
<tr>
<td>1.4 (L26–56)</td>
<td>Loads the three TS-Import header template CSVs (location/contract/reinsurance) to get column names, then allocates <code>location.df</code> with one row per <code>exposure</code> row, and empty (0-row) <code>contract.df</code> / <code>reinsurance.df</code>.</td>
<td><code>loc_cols = pd.read_csv(f"{template}{TSheader}header_LocationData.csv", nrows=0).columns</code><br><code>location_df = pd.DataFrame(index=range(len(exposure)), columns=loc_cols)</code><br><code>con_cols = pd.read_csv(f"{template}{TSheader}header_ContractData.csv", nrows=0).columns</code><br><code>contract_df = pd.DataFrame(columns=con_cols)</code><br><code>rei_cols = pd.read_csv(f"{template}{TSheader}header_ReinsuranceData.csv", nrows=0).columns</code><br><code>reinsurance_df = pd.DataFrame(columns=rei_cols)</code></td>
</tr>
</tbody>
</table><hr>
<h2 id="stage-2-—-location-file-base-fields-l58–124">Stage 2 — Location File: base fields (L58–124)</h2>

<table>
<thead>
<tr>
<th>Step</th>
<th>R Logic</th>
<th>Equivalent Python (pandas) snippet</th>
</tr>
</thead>
<tbody>
<tr>
<td>2.1 (L63–67)</td>
<td><code>ContractID</code> = <code>PolicyID</code>; <code>LocationID</code> = <code>Loc_&lt;PolicyID&gt;_&lt;1-based row index&gt;</code>; Inception/Expiration dates are fixed to <code>01/01/&lt;c_year&gt;</code> / <code>12/31/&lt;c_year&gt;</code> for every row.</td>
<td><code>location_df["ContractID"] = exposure["PolicyID"]</code><br><code>location_df["LocationID"] = "Loc_" + exposure["PolicyID"].astype(str) + "_" + (np.arange(len(exposure)) + 1).astype(str)</code><br><code>location_df["InceptionDate"] = f"01/01/{c_year}"</code><br><code>location_df["ExpirationDate"] = f"12/31/{c_year}"</code></td>
</tr>
<tr>
<td>2.2 (L72–77)</td>
<td>Street = <code>Street + StreetNo</code>, truncated to 255 chars. <code>CountryISO</code>, <code>City</code>, <code>PostalCode</code> (truncated to 15), <code>CRESTA</code> copied directly.</td>
<td><code>location_df["Street"] = (exposure["Street"] + " " + exposure["StreetNo"]).str.slice(0, 255)</code><br><code>location_df["CountryISO"] = exposure["Country"]</code><br><code>location_df["City"] = exposure["City"]</code><br><code>location_df["PostalCode"] = exposure["PostalCode"].str.slice(0, 15)</code><br><code>location_df["CRESTA"] = exposure["CRESTA"]</code></td>
</tr>
<tr>
<td>2.3 (L79–81)</td>
<td>Latitude/Longitude/<code>UserGeocodeMatchLevel</code> are written <strong>only</strong> for rows whose <code>GeoMatchLevelCode</code> is in the whitelist <code>write_coord</code>; other rows stay blank.</td>
<td><code>mask = exposure["GeoMatchLevelCode"].isin(write_coord)</code><br><code>location_df.loc[mask, "Latitude"] = exposure.loc[mask, "Latitude"]</code><br><code>location_df.loc[mask, "Longitude"] = exposure.loc[mask, "Longitude"]</code><br><code>location_df.loc[mask, "UserGeocodeMatchLevel"] = exposure.loc[mask, "GeoMatchLevelCode"]</code></td>
</tr>
<tr>
<td>2.4 (L89–96)</td>
<td>Building/Contents/Other/Time-element (BI) sums insured copied straight across; <code>DaysCovered</code> fixed at 365; <code>Currency</code>, <code>YearBuilt</code> copied.</td>
<td><code>location_df["BuildingValue"] = exposure["SI_Building"]</code><br><code>location_df["ContentsValue"] = exposure["SI_Content"]</code><br><code>location_df["OtherValue"] = exposure["SI_Other"]</code><br><code>location_df["TimeElementValue"] = exposure["SI_BI"]</code><br><code>location_df["DaysCovered"] = 365</code><br><code>location_df["Currency"] = exposure["Currency"]</code><br><code>location_df["YearBuilt"] = exposure["YearBuilt"]</code></td>
</tr>
<tr>
<td>2.5 (L98–102)</td>
<td>Occupancy/Construction code type hardcoded to <code>"AIR"</code>; codes copied; <code>ConstructionOther</code> mirrors <code>ConstructionCode</code>.</td>
<td><code>location_df["OccupancyCodeType"] = "AIR"</code><br><code>location_df["OccupancyCode"] = exposure["OccupancyCode"]</code><br><code>location_df["ConstructionCodeType"] = "AIR"</code><br><code>location_df["ConstructionCode"] = exposure["ConstructionCode"]</code><br><code>location_df["ConstructionOther"] = exposure["ConstructionCode"]</code></td>
</tr>
<tr>
<td>2.6 (L105–108)</td>
<td>UDF1–4 repurposed to carry <code>Fac</code>, <code>Sux</code>, <code>Origin</code>, <code>Product</code> — used later when building the Reinsurance file.</td>
<td><code>location_df["UDF1"] = exposure["Fac"]</code><br><code>location_df["UDF2"] = exposure["Sux"]</code><br><code>location_df["UDF3"] = exposure["Origin"]</code><br><code>location_df["UDF4"] = exposure["Product"]</code></td>
</tr>
<tr>
<td>2.7 (L110–124)</td>
<td><code>CustomFloodSOP</code>/<code>CustomFloodZone</code>: if the source column is entirely <code>NA</code> across all rows, write a single fallback constant (<code>-999</code> / <code>0</code>) to every row; otherwise copy the column and fill remaining <code>NA</code>s with that same fallback.</td>
<td><code>location_df["CustomFloodSOP"] = (-999 if exposure["SOP"].isna().all() else exposure["SOP"].fillna(-999))</code><br><code>location_df["CustomFloodZone"] = (0 if exposure["FloodZone"].isna().all() else exposure["FloodZone"].fillna(0))</code></td>
</tr>
</tbody>
</table><p><em>(Lines 126–344: commented-out “WS as Peril 1” location-field block — disabled template, not documented.)</em></p>
<hr>
<h2 id="stage-3-—-per-peril-location-fields-l346–1017">Stage 3 — Per-Peril Location Fields (L346–1017)</h2>
<p>The active block loops over <code>cat_peril_list</code> (currently only EQ, <code>cat_peril_list["EQ"]</code> →<br>
<code>cat=2</code>, <code>TSperil="PES"</code>) and fills <code>LocPerils-&lt;cat&gt;</code>, <code>LocLimitType-&lt;cat&gt;</code>,<br>
<code>DeductType-&lt;cat&gt;</code>, limit/deductible amount columns, Min/Max deductible columns,<br>
and a <code>SublimitArea-&lt;cat&gt;</code> dedupe key. Written once as the loop body (R itself<br>
loops over <code>cat_peril_list</code>):</p>

<table>
<thead>
<tr>
<th>Step</th>
<th>R Logic</th>
<th>Equivalent Python (pandas) snippet</th>
</tr>
</thead>
<tbody>
<tr>
<td>3.1 (L351–367)</td>
<td>For each peril, build the exposure column names from the peril prefix (<code>cover</code>, <code>aal</code>, <code>ded_per</code>, <code>ded_kind[1..5]</code>, <code>ded_fix[1..5]</code>, <code>ded_perc[1..5]</code>, <code>ded_min[1..5]</code>, <code>ded_max[1..5]</code>, <code>sub_per</code>, <code>sub_cat[1..5]</code>), then initialize <code>LocPerils-&lt;cat&gt;</code>/<code>LocLimitType-&lt;cat&gt;</code>/<code>DeductType-&lt;cat&gt;</code> to the peril code / <code>"N"</code> / <code>"N"</code> wherever the peril is covered.</td>
<td><code>for i, p in cat_peril_list.items():</code><br><code>cat, peril, ts = p["cat"], p["peril"], p["TSperil"]</code><br><code>cover = f"{peril}_Cover"; aal = f"{peril}_AAL"</code><br><code>ded_per = f"{peril}_Deductible_per"; sub_per = f"{peril}_Sublimit_per"</code><br><code>covered = exposure[cover].eq("TRUE")</code><br><code>location_df.loc[covered, f"LocPerils-{cat}"] = ts</code><br><code>location_df.loc[covered, [f"LocLimitType-{cat}", f"DeductType-{cat}"]] = "N"</code></td>
</tr>
<tr>
<td>3.2 (L369–384)</td>
<td>Location-level sublimit (<code>sub_per == "Location"</code>): a per-occurrence site limit (<code>S</code>, <code>LimitBldg</code>) or, when the AAL flag is set, an aggregate annual limit (<code>AGGL</code>/<code>AggregateLimit</code>); falls back to combined building/content/time limits (<code>C</code>) when no single SI-level sublimit column is populated.</td>
<td><code>loc_limit = exposure[sub_per].eq("Location") &amp; covered</code><br><code>sub1, sub2 = exposure[sub_cat[0]], exposure[sub_cat[1]]</code><br><code>occ = loc_limit &amp; sub1.notna() &amp; ~exposure[aal]</code><br><code>agg = loc_limit &amp; sub1.notna() &amp; exposure[aal]</code><br><code>location_df.loc[occ, [f"LocLimitType-{cat}", f"LimitBldg-{cat}"]] = ["S", sub1[occ]]</code><br><code>location_df.loc[agg, [f"AggregateLimitType-{cat}", f"AggregateLimit-{cat}"]] = ["AGGL", sub1[agg]]</code><br><code>comb = loc_limit &amp; sub1.isna() &amp; sub2.isna()</code><br><code>location_df.loc[comb, [f"LocLimitType-{cat}"]] = "C"</code><br><code>location_df.loc[comb, [f"LimitBldg-{cat}", f"LimitContent-{cat}", f"LimitTime-{cat}"]] = exposure.loc[comb, ["EQ_Sublimit_B", "EQ_Sublimit_C", "EQ_Sublimit_BI"]].values</code></td>
</tr>
<tr>
<td>3.3 (L387–420)</td>
<td>Location-level fixed deductible (<code>ded_per == "Location"</code>): Site (<code>S</code>) from <code>ded_fix[1]</code>, Combined-excl-time (<code>CB</code>) from <code>ded_fix[2]</code>, or By-coverage (<code>C</code>) spreading <code>ded_fix[3..5]</code> across Building/Content/Time.</td>
<td><code>loc_ded = exposure[ded_per].eq("Location") &amp; covered</code><br><code>s = exposure[ded_kind[0]].eq("Fix") &amp; exposure[ded_fix[0]].notna() &amp; loc_ded</code><br><code>location_df.loc[s, [f"DeductType-{cat}", f"DeductBldg-{cat}"]] = ["S", exposure.loc[s, ded_fix[0]]]</code><br><code>location_df.loc[s, [f"DeductContent-{cat}", f"DeductOther-{cat}", f"DeductTime-{cat}"]] = 0</code><br><code># CB from ded_fix[1] and C spreading ded_fix[2:5] across Bldg/Content/Time follow the same pattern</code></td>
</tr>
<tr>
<td>3.4 (L423–459)</td>
<td>Location-level percentage-of-loss deductible (<code>PL</code>): flagged when either of the “overall”/“combined” kind columns is <code>"% of Claims"</code> with a non-<code>NA</code> percent; the stored amount is the row-wise max of those percent columns. Separately tracks Min/Max as <code>MinMaxDeductType</code> (<code>MI</code>/<code>MA</code>/<code>MM</code>) with <code>MinDeduct</code>/<code>MaxDeduct</code>.</td>
<td><code>pl = loc_ded &amp; ((exposure[ded_kind[0]].eq("% of Claims") &amp; exposure[ded_perc[0]].notna()) \| (exposure[ded_kind[1]].eq("% of Claims") &amp; exposure[ded_perc[1]].notna()))</code><br><code>location_df.loc[pl, f"DeductType-{cat}"] = "PL"</code><br><code>location_df.loc[pl, f"DeductBldg-{cat}"] = exposure.loc[pl, [ded_perc[0], ded_perc[1]]].max(axis=1, skipna=True)</code><br><code>location_df.loc[pl, [f"DeductContent-{cat}", f"DeductOther-{cat}", f"DeductTime-{cat}"]] = 0</code><br><code># Min/Max: analogous mask on ded_min/ded_max -&gt; MinMaxDeductType "MI"/"MA"/"MM", MinDeduct/MaxDeduct = row-wise max</code></td>
</tr>
<tr>
<td>3.5 (L461–498)</td>
<td>Location-level percentage-of-loss deductible by coverage (<code>CPL</code>): same trigger pattern but spread across <code>ded_perc[3..5]</code> into Building/Content/Time individually, with its own Min/Max bookkeeping.</td>
<td><code>cpl = loc_ded &amp; exposure[[ded_kind[2], ded_kind[3], ded_kind[4]]].eq("% of Claims").any(axis=1)</code><br><code>location_df.loc[cpl, f"DeductType-{cat}"] = "CPL"</code><br><code>location_df.loc[cpl, f"DeductBldg-{cat}"] = exposure.loc[cpl, ded_perc[2]]</code><br><code>location_df.loc[cpl, f"DeductContent-{cat}"] = exposure.loc[cpl, ded_perc[3]]</code><br><code>location_df.loc[cpl, f"DeductTime-{cat}"] = exposure.loc[cpl, ded_perc[4]]</code><br><code># Min/Max bookkeeping mirrors 3.4</code></td>
</tr>
<tr>
<td>3.6 (L501–530)</td>
<td>Policy-level <code>PL</code>/<code>CPL</code> deductible (<code>ded_per == "Policy"</code>): same two patterns as 3.4/3.5, re-scoped to policy-level rows (no Min/Max tracked here).</td>
<td><code>pol_ded = exposure[ded_per].eq("Policy") &amp; covered</code><br><code># PL: same max-of-percent-columns logic as step 3.4, scoped to pol_ded</code><br><code># CPL: same per-coverage percent spread as step 3.5, scoped to pol_ded</code></td>
</tr>
<tr>
<td>3.7 (L532–535)</td>
<td>Data-quality fix-up: rows left with <code>LocLimitType == "N"</code> and <code>DeductType == "S"</code> (an invalid N/S combination) are patched to a Site limit (<code>S</code>) equal to total location SI.</td>
<td><code>invalid = loc_ded &amp; location_df[f"LocLimitType-{cat}"].eq("N") &amp; location_df[f"DeductType-{cat}"].eq("S")</code><br><code>location_df.loc[invalid, f"LimitBldg-{cat}"] = exposure.loc[invalid, "SI_Total_Loc"]</code><br><code>location_df.loc[invalid, f"LocLimitType-{cat}"] = "S"</code></td>
</tr>
<tr>
<td>3.8 (L570–579)</td>
<td>Builds a <code>SublimitArea-&lt;cat&gt;</code> key (peril + PolicyID + Country) for rows that carry a policy-level deductible/sublimit/AAL <strong>and</strong> whose LOB is in the current MU’s <code>LOB_list</code> — used later to line location rows up with the matching contract-level sublimit.</td>
<td><code>pol_ded_lob = ((exposure[ded_per].eq("Policy")) \| exposure[aal] \| (exposure[ded_per].eq("None") &amp; exposure[sub_per].eq("Policy"))) &amp; exposure["LOB"].isin(LOB_list[MU]) &amp; covered</code><br><code>location_df.loc[pol_ded_lob, f"SublimitArea-{cat}"] = ts + "_" + exposure.loc[pol_ded_lob, "PolicyID"].astype(str) + "_" + exposure.loc[pol_ded_lob, "Country"]</code></td>
</tr>
<tr>
<td>3.9 (L598–610)</td>
<td>MU exception — MUIT <code>Prop_HI</code>: for rows with a positive <code>Temp</code> value (the location limit stashed separately from the policy sublimit), force a Site location limit from <code>Temp</code>.</td>
<td><code>if MU == "MUIT" and sub == "Prop_HI" and "Temp" in exposure.columns:</code><br><code>for i, p in cat_peril_list.items():</code><br><code>cat = p["cat"]; cover = f"{p['peril']}_Cover"</code><br><code>hit = exposure["Temp"].notna() &amp; (exposure["Temp"] &gt; 0) &amp; exposure[cover].eq("TRUE")</code><br><code>location_df.loc[hit, [f"LocLimitType-{cat}", f"LimitBldg-{cat}"]] = ["S", exposure.loc[hit, "Temp"]]</code></td>
</tr>
<tr>
<td>3.10 (L612–615)</td>
<td>MU exception — MUIT <code>Mortgage</code>: overwrite <code>LocationID</code> to key on <code>Temp</code> instead of <code>PolicyID</code>.</td>
<td><code>if MU == "MUIT" and sub == "Mortgage":</code><br><code>location_df["LocationID"] = "Loc_" + exposure["Temp"].astype(str) + "_" + (np.arange(len(exposure)) + 1).astype(str)</code></td>
</tr>
</tbody>
</table><p><em>(Lines 617–1018: commented-out “FL as Peril 3” and “HA as Peril 4” location-field blocks — disabled templates mirroring the EQ logic above, not documented.)</em></p>
<hr>
<h2 id="stage-4-—-contract-file-l1020–2511">Stage 4 — Contract File (L1020–2511)</h2>
<p><em>(Lines 1024–1112: commented-out legacy “all perils, blanket contract” block — disabled template, not documented.)</em></p>
<p>Builds <code>contract.list[[peril]]</code> inside the same <code>for(i in names(cat_peril_list))</code><br>
loop, then <code>bind_rows</code>s it into <code>contract.df</code>. For every peril it branches into<br>
five mutually-exclusive <strong>PART A–E</strong> cases based on (policy sublimit present?,<br>
policy deductible present / its kind), and each PART is duplicated once for<br>
non-LOB policies (<code>pol_ded</code>, contract-only fields) and once for LOB policies<br>
(<code>pol_ded_lob</code>, which additionally write a <code>SublimitArea</code>/<code>SublimitPerils</code> key<br>
and route values into the contract’s <code>Sublimit*</code> fields instead of <code>Limit*</code>/<code>Ded*</code>).</p>

<table>
<thead>
<tr>
<th>Step</th>
<th>R Logic</th>
<th>Equivalent Python (pandas) snippet</th>
</tr>
</thead>
<tbody>
<tr>
<td>4.1 (L1117–1134)</td>
<td>Starts <code>contract.list</code>, re-derives the same per-peril column names as Stage 3, and splits exposure rows into <code>pol_ded</code> (covered, LOB <strong>not</strong> in <code>LOB_list[MU]</code>) vs. <code>pol_ded_lob</code> (covered, LOB <strong>in</strong> <code>LOB_list[MU]</code>) — these two scopes select which near-identical copy of PART A–E below applies.</td>
<td><code>contract_frames = {}</code><br><code>for i, p in cat_peril_list.items():</code><br><code>cat, peril, ts = p["cat"], p["peril"], p["TSperil"]</code><br><code>cover = f"{peril}_Cover"</code><br><code>covered = exposure[cover].eq("TRUE")</code><br><code>pol_ded = ~exposure["LOB"].isin(LOB_list[MU]) &amp; covered</code><br><code>pol_ded_lob = exposure["LOB"].isin(LOB_list[MU]) &amp; covered</code></td>
</tr>
<tr>
<td>4.2 — CONTRACT TYPE 1 (L1136–1171)</td>
<td>Rows with no policy sublimit and no (relevant) policy deductible get one bare <code>"N"</code> (no-limit) contract layer per policy, de-duplicated.</td>
<td><code>work = covered &amp; exposure[sub_per].ne("Policy") &amp; no_relevant_policy_ded</code><br><code>rows = pd.DataFrame(index=exposure.index[work], columns=con_cols)</code><br><code>rows["Perils"], rows["LayerPerils"] = perils, ts</code><br><code>rows["ContractID"] = exposure.loc[work, "PolicyID"]</code><br><code>rows["LimitType"] = "N"</code><br><code>rows["LayerID"] = ts + "_Layer_" + exposure.loc[work, "PolicyID"].astype(str)</code><br><code>frame = rows.drop_duplicates()</code></td>
</tr>
<tr>
<td>4.3 — PART A (L1173–1256, repeated for LOB scope L1565–1653)</td>
<td>No (or location-level) sublimit, <code>PL</code>/<code>CPL</code> deductible: contract <code>LimitType = "N"</code>; deductible Min/Max (<code>MI</code>/<code>MA</code>/<code>MM</code>) rolled up from the row-wise max of the relevant <code>ded_min</code>/<code>ded_max</code> columns. LOB-scoped copy additionally stamps <code>SublimitArea</code>/<code>SublimitPerils</code> (peril+PolicyID+Country) so it lines up with the Stage-3.8 location key.</td>
<td><code>work = pol_ded &amp; exposure[sub_per].ne("Policy") &amp; location_df[f"DeductType-{cat}"].isin(["PL", "CPL"])</code><br><code>dmin = exposure.loc[work, ded_min].max(axis=1, skipna=True)</code><br><code>dmax = exposure.loc[work, ded_max].max(axis=1, skipna=True)</code><br><code>rows["DedType"] = np.select([dmin.notna() &amp; dmax.notna(), dmin.notna(), dmax.notna()], ["MM", "MI", "MA"], default=pd.NA)</code><br><code>rows["DedAmt1"], rows["DedAmt2"] = dmin, dmax</code><br><code># LOB copy additionally: rows["SublimitArea"] = ts + "_" + PolicyID + "_" + Country; rows["SublimitPerils"] = ts</code></td>
</tr>
<tr>
<td>4.4 — PART B (L1260–1369, repeated L1655–1769)</td>
<td>Policy-level sublimit present, <code>PL</code>/<code>CPL</code> deductible: sublimit → <code>LimitType "B"</code>/<code>Limit1</code>, or <code>AGGL</code>/<code>AggregateLimit</code> when AAL is set, or combined <code>LimitA</code>/<code>LimitC</code>/<code>LimitD</code> when no single sublimit column is populated; deductible Min/Max as in PART A. LOB copy writes the same shapes into <code>SubLimitLimitType</code>/<code>SubLimitOcc</code>/<code>SublimitLimitA-D</code>/<code>SublimitDedType</code>/<code>SublimitDedAmt1-2</code> plus the <code>SublimitArea</code> key.</td>
<td><code>work = pol_ded &amp; exposure[sub_per].eq("Policy") &amp; location_df[f"DeductType-{cat}"].isin(["PL", "CPL"])</code><br><code>b = exposure[sub_cat[0]].notna() &amp; ~exposure[aal]</code><br><code>agg = exposure[sub_cat[0]].notna() &amp; exposure[aal]</code><br><code>c = exposure[sub_cat[0]].isna() &amp; exposure[sub_cat[1]].isna()</code><br><code>rows.loc[b &amp; work, ["LimitType", "Limit1"]] = ["B", exposure.loc[b &amp; work, sub_cat[0]]]</code><br><code>rows.loc[agg &amp; work, ["LimitType", "AggregateLimitType", "AggregateLimit"]] = ["N", "AGGL", exposure.loc[agg &amp; work, sub_cat[0]]]</code><br><code>rows.loc[c &amp; work, ["LimitType", "LimitA", "LimitC", "LimitD"]] = ["C", exposure.loc[c &amp; work, sub_cat[2]], exposure.loc[c &amp; work, sub_cat[3]], exposure.loc[c &amp; work, sub_cat[4]]]</code><br><code># + Min/Max DedType/DedAmt as in PART A; LOB copy targets Sublimit* fields instead</code></td>
</tr>
<tr>
<td>4.5 — PART C (L1373–1450, repeated L1772–1853)</td>
<td>Policy-level sublimit <strong>and</strong> a fixed policy deductible: sublimit handled as PART B; fixed deductible routed to <code>DedType</code> <code>B</code>/<code>CB</code>/<code>C</code> depending on which fixed-deductible column (<code>ded_fix[1..5]</code>) is populated, spreading <code>C</code> across <code>DedAmt1</code>/<code>DedAmt3</code>/<code>DedAmt4</code>. LOB copy again targets <code>Sublimit*</code>.</td>
<td><code>work = pol_ded &amp; exposure[sub_per].eq("Policy") &amp; exposure[ded_per].eq("Policy") &amp; exposure[ded_kind].eq("Fix").any(axis=1)</code><br><code>b, cb = exposure[ded_fix[0]].notna(), exposure[ded_fix[1]].notna()</code><br><code>c = exposure[ded_fix[0]].isna() &amp; exposure[ded_fix[1]].isna()</code><br><code>rows.loc[b &amp; work, ["DedType", "DedAmt1"]] = ["B", exposure.loc[b &amp; work, ded_fix[0]]]</code><br><code>rows.loc[cb &amp; work, ["DedType", "DedAmt1"]] = ["CB", exposure.loc[cb &amp; work, ded_fix[1]]]</code><br><code>rows.loc[c &amp; work, ["DedType", "DedAmt1", "DedAmt3", "DedAmt4"]] = ["C", exposure.loc[c &amp; work, ded_fix[2]], exposure.loc[c &amp; work, ded_fix[3]], exposure.loc[c &amp; work, ded_fix[4]]]</code><br><code># + PART B's sublimit routing</code></td>
</tr>
<tr>
<td>4.6 — PART D (L1453–1504, repeated L1855–1912)</td>
<td>No (or location-level) sublimit, fixed policy deductible: same fixed-deductible routing as PART C; contract <code>LimitType = "N"</code> (contract-only copy) or <code>SubLimitLimitType = "N"</code> + <code>SublimitArea</code> key (LOB copy).</td>
<td><em>(Same deductible-routing snippet as 4.5, with <code>LimitType</code>/<code>SubLimitLimitType</code> forced to <code>"N"</code> and no sublimit-amount fields set.)</em></td>
</tr>
<tr>
<td>4.7 — PART E (L1506–1563, repeated L1913–1973)</td>
<td>Policy-level sublimit, no policy deductible (deductible is <code>"None"</code>/blank, or sits at location level as a fixed amount): sublimit routed exactly as PART B; no deductible fields touched.</td>
<td><em>(Same sublimit-routing snippet as 4.4, deductible block omitted.)</em></td>
</tr>
<tr>
<td>4.8 (L1975–2021)</td>
<td>Post-loop fix-up: contract layers with <code>LimitType == "C"</code> that coexist, on the same <code>ContractID</code>, with an invalid (<code>S</code>/<code>CB</code>/<code>CT</code>/<code>PL</code>) location-level deductible or location limit are downgraded to Blanket (<code>"B"</code>).</td>
<td><code>laylimit_c = frame.loc[frame["LimitType"].eq("C"), "ContractID"].unique()</code><br><code>for k in ["S", "CB", "CT", "PL"]:</code><br><code>locded = location_df.loc[loc_ded &amp; location_df[f"DeductType-{cat}"].eq(k), "ContractID"].unique()</code><br><code>loclim = location_df.loc[loc_limit &amp; location_df[f"LocLimitType-{cat}"].eq(k), "ContractID"].unique()</code><br><code>bad = set(laylimit_c) &amp; (set(locded) \| set(loclim))</code><br><code>frame.loc[frame["ContractID"].isin(bad), "LimitType"] = "B"</code></td>
</tr>
<tr>
<td>4.9 (L2030)</td>
<td>Concatenates all per-peril contract frames into one <code>contract.df</code>.</td>
<td><code>contract_df = pd.concat(contract_frames.values(), ignore_index=True)</code></td>
</tr>
<tr>
<td>4.10 (L2293–2295)</td>
<td>Hard assertion: every contract row must have an LOB; abort otherwise.</td>
<td><code>assert contract_df["LOB"].notna().all(), "Missing LOB in contract_df"</code></td>
</tr>
<tr>
<td>4.11 (L2310–2336)</td>
<td>Single-policy hardcoded exception (MUCH EQ, PolicyID containing <code>"4000730606"</code>, aka BLKB): replaces that policy’s per-occurrence limit/deductible with an annual-aggregate structure (<code>AggregateDedType = "AGGD"</code>, <code>AggregateLimitType = "AGGL"</code>) pulled from a unique value in <code>exposure</code>.</td>
<td><code>if MU == "MUCH" and sub == "EQ" and exposure["PolicyID"].str.contains("4000730606").any():</code><br><code>blkb = contract_df["ContractID"].str.contains("CH 4000730606")</code><br><code>exp_blkb = exposure["PolicyID"].str.contains("CH 4000730606")</code><br><code>contract_df.loc[blkb, ["LimitType", "Limit1", "DedType", "DedAmt1"]] = ["N", pd.NA, pd.NA, pd.NA]</code><br><code>contract_df.loc[blkb, ["AggregateDedType", "AggregateLimitType"]] = ["AGGD", "AGGL"]</code><br><code>contract_df.loc[blkb, "AggregateDedAmt"] = exposure.loc[exp_blkb, ded_fix[0]].unique()[0]</code><br><code>contract_df.loc[blkb, "AggregateLimit"] = exposure.loc[exp_blkb, sub_cat[0]].unique()[0]</code></td>
</tr>
<tr>
<td>4.12 (L2338–2348)</td>
<td>MU exception — MUIT <code>Mortgage</code>: forces a combined EQ+FL (<code>"PES+PFL"</code>) Blanket layer whose <code>Limit1</code> is looked up per <code>ContractID</code> from an externally supplied <code>terms</code> table.</td>
<td><code>if MU == "MUIT" and sub == "Mortgage":</code><br><code>contract_df["Perils"] = "PES+PFL"; contract_df["LimitType"] = "B"</code><br><code>contract_df["LayerID"] = "Layer_" + contract_df["ContractID"].astype(str)</code><br><code>contract_df["Limit1"] = contract_df["ContractID"].map(terms.loc["Policy_Limit"])</code></td>
</tr>
<tr>
<td>4.13 (L2350–2363)</td>
<td>MU exception — MUIT <code>Prop_HI</code>: rolls each peril’s per-row AAL value up to one max-per-policy figure and books it as that peril’s contract-level <code>AggregateLimit</code> (<code>AGGL</code>), alongside the already-set per-occurrence <code>Limit1</code>.</td>
<td><code>if MU == "MUIT" and sub == "Prop_HI":</code><br><code>for i, p in cat_peril_list.items():</code><br><code>aal_pol = exposure.groupby("PolicyID")[f"{p['peril']}_AAL_Value"].max()</code><br><code>aal = contract_df["ContractID"].map(aal_pol)</code><br><code>hit = contract_df["LayerPerils"].eq(p["TSperil"]) &amp; aal.gt(0)</code><br><code>contract_df.loc[hit, ["AggregateLimitType", "AggregateLimit"]] = ["AGGL", aal[hit]]</code></td>
</tr>
<tr>
<td>4.14 (L2365–2369)</td>
<td>MU exception — MUIT <code>Prop_HR</code>/<code>Prop_HI</code>: copies a precomputed product-eligibility flag (<code>Temp2</code>) into location <code>UDF5</code>.</td>
<td><code>if MU == "MUIT" and sub in ("Prop_HR", "Prop_HI") and "Temp2" in exposure.columns:</code><br><code>location_df["UDF5"] = exposure["Temp2"]</code></td>
</tr>
<tr>
<td>4.15 (L2371–2476)</td>
<td>MU exception — MUFR <code>Prop</code>: writes <code>Participation</code> into each covered peril’s <code>Participation1</code> location field; for policies with a positive policy-level <code>Limit</code>, demotes the existing per-peril layer limit/deductible into the contract <code>Sublimit*</code> fields (tagged with a new <code>SublimitArea</code>) and replaces the layer with a single cross-peril Policy Aggregate Limit (PAL) layer — Blanket (<code>"B"</code>) or Excess (<code>"E"</code> + <code>AttachmentAmt</code>) depending on whether an attachment point is set. Policies with a policy <code>Limit</code> but no existing contract row get a bare PAL layer appended.</td>
<td><code>if MU == "MUFR" and sub == "Prop":</code><br><code>for i, p in cat_peril_list.items():</code><br><code>covered = exposure[f"{p['peril']}_Cover"].eq("TRUE")</code><br><code>location_df.loc[covered, f"Participation1-{p['cat']}"] = exposure.loc[covered, "Participation"]</code><br><code>limit_pol = exposure.groupby("PolicyID")["Limit"].max()</code><br><code>attach_pol = exposure.groupby("PolicyID")["AttachmentAmt"].max()</code><br><code>pal_pol = limit_pol[limit_pol.gt(0)].index</code><br><code>hit = contract_df["ContractID"].isin(pal_pol)</code><br><code># move LimitType/Limit1/LimitA-D/DedType/DedAmt1-4 -&gt; SubLimitLimitType/SubLimitOcc/SublimitLimitA-D/SublimitDedType/SublimitDedAmt1-4, clear the originals</code><br><code>contract_df.loc[hit, "SublimitArea"] = contract_df.loc[hit, "LayerPerils"] + "_" + contract_df.loc[hit, "ContractID"].astype(str)</code><br><code>contract_df.loc[hit, ["LayerPerils", "LayerID"]] = [perils, "PAL_Layer_" + contract_df.loc[hit, "ContractID"].astype(str)]</code><br><code>contract_df.loc[hit, "Limit1"] = contract_df.loc[hit, "ContractID"].map(limit_pol)</code><br><code>attach = contract_df.loc[hit, "ContractID"].map(attach_pol).fillna(0)</code><br><code>contract_df.loc[hit, "LimitType"] = np.where(attach.gt(0), "E", "B")</code><br><code>contract_df.loc[hit &amp; attach.gt(0), "AttachmentAmt"] = attach[attach.gt(0)]</code><br><code># fallback: append a bare PAL row for any pal_pol ContractID missing entirely from contract_df</code></td>
</tr>
<tr>
<td>4.16 (L2478–2481)</td>
<td>De-duplicates <code>contract.df</code>; sorts both <code>contract.df</code> and <code>location.df</code> by <code>ContractID</code>.</td>
<td><code>contract_df = contract_df.drop_duplicates().sort_values("ContractID")</code><br><code>location_df = location_df.sort_values("ContractID")</code></td>
</tr>
</tbody>
</table><p><em>(Lines 2032–2285: commented-out legacy EQ “type 2”/“type 3” contract blocks — disabled templates, not documented.)</em></p>
<hr>
<h2 id="stage-5-—-reinsurance-file--output-l2512–2733">Stage 5 — Reinsurance File &amp; Output (L2512–2733)</h2>

<table>
<thead>
<tr>
<th>Step</th>
<th>R Logic</th>
<th>Equivalent Python (pandas) snippet</th>
</tr>
</thead>
<tbody>
<tr>
<td>5.1 (L2516–2546)</td>
<td>Facultative reinsurance rows: one per unique <code>(ContractID, UDF1, LayerPerils)</code> where <code>UDF1</code> (the Fac %) is positive. <code>CededPercentage = UDF1</code>, <code>ReinsuranceType = "SS"</code>, <code>ExposureType = "LYR"</code>; <code>ReinsuranceID</code> built from <code>MU</code> (and <code>sub</code> for MUDE <code>Large</code>/<code>NonLarge</code>/<code>TV</code>) plus <code>LayerPerils</code>.</td>
<td><code>fac = contract_df.drop_duplicates(["ContractID", "UDF1", "LayerPerils"])</code><br><code>fac = fac[fac["UDF1"].gt(0)]</code><br><code>rows = pd.DataFrame(index=fac.index, columns=rei_cols)</code><br><code>rows[["ContractID", "ExposureID"]] = fac[["ContractID", "LayerID"]]</code><br><code>rows["CededPercentage"], rows["Currency"] = fac["UDF1"], fac["Currency"]</code><br><code>rows["ReinsurancePerils"] = fac["LayerPerils"]</code><br><code>rows[["ReinsuranceType", "ExposureType"]] = "SS", "LYR"</code><br><code>prefix = f"FAC_{MU}_{sub}_" if (MU == "MUDE" and sub in ("Large", "NonLarge", "TV")) else f"FAC_{MU}_"</code><br><code>rows["ReinsuranceID"] = prefix + fac["LayerPerils"]</code><br><code>reinsurance_df = pd.concat([reinsurance_df, rows], ignore_index=True)</code></td>
</tr>
<tr>
<td>5.2 (L2548–2580)</td>
<td>Surplus-share reinsurance rows: same pattern keyed on <code>UDF2</code> (Sux %). For MUIT, <code>ReinsuranceID</code> branches on LOB into <code>"Property"</code> (<code>MUIT_Incendio</code>/<code>MUIT_Arte</code>) vs. <code>"Engineering"</code> (<code>MUIT_Engineering</code>), warning if any LOB is neither.</td>
<td><code>ss = contract_df.drop_duplicates(["ContractID", "UDF2", "LayerPerils"])</code><br><code>ss = ss[ss["UDF2"].gt(0)]</code><br><code>rows = pd.DataFrame(index=ss.index, columns=rei_cols)</code><br><code>rows[["ContractID", "ExposureID"]] = ss[["ContractID", "LayerID"]]</code><br><code>rows["CededPercentage"], rows["Currency"] = ss["UDF2"], ss["Currency"]</code><br><code>rows["ReinsurancePerils"] = ss["LayerPerils"]</code><br><code>rows[["ReinsuranceType", "ExposureType"]] = "SS", "LYR"</code><br><code>if MU == "MUIT":</code><br><code>is_prop = ss["LOB"].isin(["MUIT_Incendio", "MUIT_Arte"]); is_eng = ss["LOB"].eq("MUIT_Engineering")</code><br><code>if (is_prop == ~is_eng).sum() != len(ss): warnings.warn("unknown LOBs with a Surplus coverage")</code><br><code>rows.loc[is_prop, "ReinsuranceID"] = f"SS_{MU}_Property_" + ss.loc[is_prop, "LayerPerils"]</code><br><code>rows.loc[is_eng, "ReinsuranceID"] = f"SS_{MU}_Engineering_" + ss.loc[is_eng, "LayerPerils"]</code><br><code>elif MU == "MUDE" and sub in ("Large", "NonLarge", "TV"):</code><br><code>rows["ReinsuranceID"] = f"SS_{MU}_{sub}_" + ss["LayerPerils"]</code><br><code>else:</code><br><code>rows["ReinsuranceID"] = f"SS_{MU}_" + ss["LayerPerils"]</code><br><code>reinsurance_df = pd.concat([reinsurance_df, rows], ignore_index=True)</code></td>
</tr>
<tr>
<td>5.3 (L2586–2590)</td>
<td>De-duplicates <code>reinsurance.df</code>; re-sorts <code>reinsurance.df</code>, <code>contract.df</code>, <code>location.df</code> by <code>ContractID</code>.</td>
<td><code>reinsurance_df = reinsurance_df.drop_duplicates().sort_values("ContractID")</code><br><code>contract_df = contract_df.sort_values("ContractID")</code><br><code>location_df = location_df.sort_values("ContractID")</code></td>
</tr>
<tr>
<td>5.4 (L2592–2599)</td>
<td>Consistency checks (warn, don’t fail): any <code>ContractID</code> in <code>location.df</code> missing from <code>contract.df</code>; and <code>ContractID</code>s with a <code>SublimitArea-*</code> on the location side must match those with a <code>SublimitArea</code> on the contract side.</td>
<td><code>missing = set(location_df["ContractID"]) - set(contract_df["ContractID"])</code><br><code>if missing: warnings.warn(f"Not all Contracts are reflected in the Contract file: {missing}")</code><br><code>sub_cols = [f"SublimitArea-{c}" for c in range(1, 5)]</code><br><code>loc_sub_ids = set(location_df.loc[location_df[sub_cols].notna().any(axis=1), "ContractID"])</code><br><code>con_sub_ids = set(contract_df.loc[contract_df["SublimitArea"].notna(), "ContractID"])</code><br><code>if loc_sub_ids != con_sub_ids: warnings.warn("Not all Contracts are reflected in the Contract file")</code></td>
</tr>
<tr>
<td>5.5 (L2606–2654)</td>
<td>Optional bounding-box filter: when <code>bbox &amp; write_output</code>, keep only location rows inside a lat/long box, cascade-filter <code>contract.df</code>/<code>reinsurance.df</code> to the surviving <code>ContractID</code>s, re-encode text to ISO-8859-1, and write <code>*_bbox.csv</code> files (plus a CRESTA-stripped <code>*_location_bbox_exCRESTA.csv</code> variant if <code>CRESTA</code> is populated).</td>
<td><code>if bbox and write_output:</code><br><code>lon_min, lat_min, lon_max, lat_max = bbox_coord</code><br><code>in_box = location_df["Latitude"].between(lat_min, lat_max) &amp; location_df["Longitude"].between(lon_min, lon_max)</code><br><code>location_df = location_df[in_box.fillna(False)]</code><br><code>contract_df = contract_df[contract_df["ContractID"].isin(location_df["ContractID"])]</code><br><code>reinsurance_df = reinsurance_df[reinsurance_df["ContractID"].isin(location_df["ContractID"])]</code><br><code>for name, df in [("location_bbox", location_df), ("contract_bbox", contract_df), ("reinsurance_bbox", reinsurance_df)]:</code><br><code>df.to_csv(f"{output}TSImport/{MU}_{sub}_{name}.csv", index=False, encoding="latin-1")</code><br><code>if location_df["CRESTA"].notna().any():</code><br><code>location_df.assign(CRESTA=pd.NA).to_csv(f"{output}TSImport/{MU}_{sub}_location_bbox_exCRESTA.csv", index=False, encoding="latin-1")</code></td>
</tr>
<tr>
<td>5.6 (L2662–2720)</td>
<td>Standard (non-bbox) output: writes <code>location.csv</code>/<code>contract.csv</code>/<code>reinsurance.csv</code>; for MUIT, additionally writes a <code>*_location_Hail.csv</code> remapping <code>OccupancyCode</code> 328→321; and (whenever <code>CRESTA</code> is populated) a <code>*_location_exCRESTA.csv</code> with <code>CRESTA</code> blanked.</td>
<td><code>if write_output and not bbox:</code><br><code>location_df.to_csv(f"{output}TSImport/{MU}_{sub}_location.csv", index=False, encoding="latin-1")</code><br><code>contract_df.to_csv(f"{output}TSImport/{MU}_{sub}_contract.csv", index=False, encoding="latin-1")</code><br><code>if len(reinsurance_df): reinsurance_df.to_csv(f"{output}TSImport/{MU}_{sub}_reinsurance.csv", index=False, encoding="latin-1")</code><br><code>if MU == "MUIT":</code><br><code>loc_hail = location_df.copy(); loc_hail["OccupancyCode"] = loc_hail["OccupancyCode"].replace(328, 321)</code><br><code>loc_hail.to_csv(f"{output}TSImport/{MU}_{sub}_location_Hail.csv", index=False, encoding="latin-1")</code><br><code>if location_df["CRESTA"].notna().any():</code><br><code>location_df.assign(CRESTA=pd.NA).to_csv(f"{output}TSImport/{MU}_{sub}_location_exCRESTA.csv", index=False, encoding="latin-1")</code></td>
</tr>
<tr>
<td>5.7 (L2722–2728)</td>
<td>Housekeeping: closes the ODBC connection(s) opened earlier in the caller, and removes a large batch of temporary working variables (pattern-matched by name prefix) to free memory.</td>
<td><code># No direct pandas equivalent — in a Python port this is just letting intermediate</code><br><code># masks/frames (pol_ded, pl_ded, fix_ded_*, index_*, max_*, contract_frames, ...) go</code><br><code># out of scope / del'ing them, and closing any DB connection opened earlier.</code></td>
</tr>
</tbody>
</table><hr>
<h2 id="open-ambiguities--external-dependencies-worth-flagging-when-porting">Open ambiguities / external dependencies worth flagging when porting</h2>
<ul>
<li><strong>Not defined in this file</strong>: <code>cat_peril_list</code>, <code>LOB_list</code>, <code>MU</code>, <code>sub</code>, <code>c_year</code>,<br>
<code>write_coord</code>, <code>core</code>, <code>template</code>, <code>TSheader</code>, <code>perils</code>, <code>cat_ded</code>, <code>terms</code>,<br>
<code>bbox</code>, <code>bbox_coord</code>, <code>write_output</code>, <code>output</code>, and the ODBC connection closed by<br>
<code>odbcCloseAll()</code> — all supplied by <code>Standards.R</code> or the calling script.</li>
<li><code>Mode_MM</code> is referenced only inside commented-out (disabled) code and is never<br>
defined in this file either.</li>
<li><code>ded_kind</code>/<code>ded_fix</code>/<code>ded_perc</code>/<code>ded_min</code>/<code>ded_max</code>/<code>sub_cat</code> are positional<br>
string vectors; their index-to-meaning mapping (site / combined / building /<br>
contents / time-element) is inferred from naming and usage, not stated<br>
explicitly in the R source.</li>
<li>The numeric rounding rule in Stage 1.3 (round each value to a number of<br>
decimal places equal to the character-length of the value itself) reads as<br>
unusual; it is transcribed mechanically since its intent isn’t documented.</li>
<li><code>aal</code> is reused as both a derived column-name string and (implicitly) a<br>
boolean condition name in different scopes — a potential source of confusion<br>
when porting.</li>
</ul>

