# Security Analytics & Threat Detection

Nine SOC-style analytics workflows across roughly 1.3 million log and URL records, from domain frequency triage through to three machine learning classifiers benchmarked against each other.

**[Read the full project report (PDF)](./Security%20Analytics%20and%20Threat%20Detection%20Report.pdf)**

> All work was performed against training datasets in an isolated analysis environment. Raw datasets and source log files are not included in this repository.

---

## Tools

| Tool | Use |
|---|---|
| Python, pandas, NumPy | Log parsing, enrichment, feature processing |
| Kibana | Querying, filtering, dashboards |
| scikit-learn | Logistic Regression, Random Forest, MLP classifiers |
| ipwhois | ASN and IP ownership attribution |

Data sources included DNS, SSL/TLS, endpoint, Windows event, HTTP, fileinfo and SMTP-style events, plus labeled phishing and benign website feature sets.

---

## Workflows and Findings

### Domain frequency analysis

Extracted primary domains from a URL corpus and scored them by frequency, on the premise that rare domains are worth an analyst's attention before common ones.

- Processed approximately **1,000,000 URL records**
- Applied a frequency threshold that reduced the set to **156,871 records** for review
- Produced a domain, frequency and score table usable as a threat hunting queue

### Access log analysis

Enriched raw authentication and resource-access logs with the identity context an analyst needs to triage them.

- Analyzed approximately **333,952 access log records**
- Joined **100 employee records** and **81 resource records** by employee and resource ID
- Result: every log line carried user, department, resource, location, IP, action and outcome, turning raw events into reviewable access activity

### AI-assisted NetFlow review

Ran an assisted first pass over a NetFlow capture, then validated every claim against the underlying evidence.

| Attribute | Value |
|---|---|
| Records | 165 flows |
| Fields | 20, including timestamps, flow IDs, IPs, ports, protocols, packet and byte counts |
| Most active source | 192.168.163.136 (90 occurrences) |
| Most active destination | 72.21.91.29 (40 occurrences) |
| Common destination ports | 80, 443, 1900 |

Flagged for investigation: outbound traffic over port 1900 inconsistent with normal SSDP behavior, a host showing failed TLS activity alongside elevated outbound volume, horizontal scanning from 192.168.163.136 across multiple destinations, and RPC and management port activity consistent with possible lateral movement.

The important part of this exercise was the validation step. Assisted analysis accelerated the first pass, but each finding still had to be confirmed against the flow records before it counted.

> Deeper NetFlow exfiltration hunting, including the ASN enrichment pipeline and Kibana detection rule, lives in a dedicated repository: **[netflow-threat-detection](https://github.com/pshirolk3012/netflow-threat-detection)**

### Machine learning classification

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
2. Clean and enrich logs with employee, department and resource context
3. Surface suspicious events using filtering, baselining, frequency analysis, thresholds and correlation
4. Investigate specific behaviors: abnormal outbound traffic, failed TLS, suspicious port usage
5. Validate assisted findings against manual evidence to control false positives
6. Train and evaluate models for malicious and benign classification
7. Document findings in a form suitable for escalation and review

---

## Takeaways

**Rare beats loud.** Frequency analysis turns an unmanageable URL corpus into a prioritized hunting queue without a single signature.

**Enrichment is the analysis.** A raw access log is not evidence. Joined to employee and resource context, the same rows answer who touched what and whether they should have.

**Assisted analysis needs a verification step.** Speed on the first pass is only valuable if every finding is confirmed against the evidence before escalation.

**Metrics need interpretation, not just reporting.** Accuracy alone hid the fact that the interpretable model was the more useful one operationally.
