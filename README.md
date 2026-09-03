# Security Analytics & Threat Detection

SOC-style analytics across roughly 1.3 million log and URL records: frequency-based threat hunting, access log enrichment, a NetFlow data exfiltration rule taken from written policy to working detection, and three machine learning classifiers benchmarked against each other.

**[Read the full project report (PDF)](./Security%20Analytics%20and%20Threat%20Detection%20Report.pdf)**

> All work was performed against training datasets in an isolated analysis environment. Raw datasets and source log files are not included in this repository.

---

## Tools

| Tool | Use |
|---|---|
| Python, pandas, NumPy | Log parsing, enrichment, feature processing |
| Elastic and Kibana | Ingestion, querying, dashboards |
| ipwhois | ASN and IP ownership attribution |
| scikit-learn | Logistic Regression, Random Forest, MLP classifiers |

Data sources included DNS, SSL/TLS, endpoint, Windows event, HTTP, fileinfo and SMTP-style events, NetFlow records, and labeled phishing and benign website feature sets.

---

## 1. Domain Frequency Analysis

Extracted primary domains from a URL corpus and scored them by frequency, on the premise that rare domains are worth an analyst's attention before common ones.

- Processed approximately **1,000,000 URL records**
- Applied a frequency threshold that reduced the set to **156,871 records** for review
- Produced a domain, frequency and score table usable as a threat hunting queue

---

## 2. Access Log Analysis

Enriched raw authentication and resource-access logs with the identity context an analyst needs to triage them.

- Analyzed approximately **333,952 access log records**
- Joined **100 employee records** and **81 resource records** by employee and resource ID
- Result: every log line carried user, department, resource, location, IP, action and outcome, turning raw events into reviewable access activity

---

## 3. NetFlow Data Exfiltration Hunting

The most complete detection engineering piece in the project: taking a rule written in plain English and turning it into something a SIEM can actually execute.

**The rule as written:**

> Large data uploads to external IPs in the "none" company category should be examined for possible data exfiltration. A large number of connections to "other" companies should also be examined.

### Enrichment

Raw NetFlow records contain IP addresses but nothing about who owns them, which makes "external" impossible to define in a query. The `ipwhois` module resolved ownership, and the source `nflog.json` was rewritten with three added fields:

- `int_src_ip` and `int_dest_ip`: is each address internal?
- `ASNCompany`: the owning organization from ASN lookup, with unattributed addresses marked `NONE`

Without ownership attribution, internal-to-external traffic cannot be separated from ordinary internal traffic. This step is what makes the rest possible.

### The query

```
int_src_ip : true AND int_dest_ip : false AND ASNCompany : "NONE"
```

Internal source, external destination, unattributed owner. Run against the enriched log in Kibana, this returned **148 results**: a workable investigation queue out of the full volume.

### Thresholds

| Criterion | Threshold |
|---|---|
| Data volume to external host | > 10 MB |
| Session duration | > 2,000 seconds |

### Findings

Three transfers cleared the volume threshold by a wide margin, all to external hosts with no ASN attribution, which is exactly the combination the rule was written to catch:

| Transfer size | Assessment |
|---|---|
| 23.27 MB | Suspicious, highest single outbound volume |
| 17.85 MB | Suspicious |
| 14.48 MB | Suspicious |

One connection ran for **292,330 seconds**, roughly 81 hours, against a 2,000-second threshold. It moved less data than the top transfers but is arguably the more concerning finding, since a session of that length is consistent with a persistent channel rather than normal user activity.

**Thresholds are judgment calls.** 10 MB and 2,000 seconds are not universal truths. They reflect what is normal for this environment. Too tight and the queue drowns in noise; too loose and real activity walks past.

---

## 4. AI-Assisted Flow Review

A second pass over a NetFlow capture using assisted analysis, then validating every claim against the underlying records.

| Attribute | Value |
|---|---|
| Records | 165 flows |
| Fields | 20, including timestamps, flow IDs, IPs, ports, protocols, packet and byte counts |
| Most active source | 192.168.163.136 (90 occurrences) |
| Most active destination | 72.21.91.29 (40 occurrences) |
| Common destination ports | 80, 443, 1900 |

Flagged for investigation: outbound traffic over port 1900 inconsistent with normal SSDP behavior, a host showing failed TLS activity alongside elevated outbound volume, horizontal scanning from 192.168.163.136 across multiple destinations, and RPC and management port activity consistent with possible lateral movement.

The point of this exercise was the validation step. Assisted analysis accelerated the first pass, but each finding still had to be confirmed against the flow records before it counted as anything.

---

## 5. Machine Learning Classification

Trained and compared three classifiers on a labeled malicious and benign website dataset of approximately **24,232 records across 13 core features**.

| Model | Accuracy | Notes |
|---|---|---|
| Logistic Regression | ~82.9% | ROC-AUC ~0.854. Interpretable coefficients, useful baseline |
| Random Forest | ~91 to 92% | Strongest performer, clear feature importance output |
| Neural network (MLP) | ~85 to 87% | Convergence warnings indicated more tuning and scaling were needed |

Top signals were page interaction and script-related indicators, including inline script counts and onload counts.

**What this showed:** the more complex model did not win. Random Forest beat the MLP, and Logistic Regression stayed useful because its coefficients could be explained to someone who has to act on the output.

---

## SOC Workflow Demonstrated

1. Collect and load data from domains, access logs, NetFlow records and labeled datasets
2. Clean and enrich logs with ownership, employee, department and resource context
3. Translate written detection criteria into executable queries with defined field logic
4. Surface suspicious events using filtering, baselining, frequency analysis and thresholds
5. Investigate specific behaviors: abnormal outbound traffic, failed TLS, suspicious port usage, long-lived sessions
6. Validate assisted findings against manual evidence to control false positives
7. Train and evaluate models for malicious and benign classification
8. Document findings in a form suitable for escalation and review

---

## Takeaways

**Raw logs are not evidence.** A NetFlow record is addresses and byte counts. An access log is IDs. Enrichment, whether ASN attribution or employee mapping, is what turns either into something you can reason about.

**Detection rules need translating.** "Large uploads to external IPs" is a sentence. Turning it into a query with defined field logic and numeric thresholds is the analyst's job, and it is where false positive rates actually get decided.

**Volume alone is not the story.** The 292,330-second session moved less data than the top transfers. Long-lived low-volume channels are exactly what beats volume-only detection.

**Assisted analysis needs a verification step.** Speed on the first pass only counts if every finding is confirmed against the evidence before escalation.

**Metrics need interpretation, not just reporting.** Accuracy alone hid the fact that the interpretable model was the more useful one operationally.
