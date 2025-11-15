# S09 - SBOM & SCA Summary Report

## SBOM Generation (Syft)
- ✅ **Tool**: Syft (Anchore) via Docker
- ✅ **Format**: CycloneDX JSON  
- ✅ **File**: `sbom.json`

## SCA Scanning (Grype)
- ✅ **Tool**: Grype v0.103.0 via Docker
- ✅ **Database**: Updated 2025-11-14
- ✅ **Format**: JSON
- ✅ **File**: `sca_report.json`

### Vulnerability Summary:
| Severity | Count | Action Required |
|----------|-------|-----------------|
| **Critical** | 0 | ✅ None |
| **High** | 0 | ✅ None |
| **Medium** | 3 | ⚠️ Review |
| **Low** | 0 | ✅ None |

### Found Vulnerabilities in Jinja2 3.1.4:
1. **GHSA-q2x7-8rv6-6q7h** - Sandbox breakout through format method (CVE-2024-56326)
   - **Severity**: Medium (CVSS 7.8)
   - **Fix**: Update to Jinja2 >= 3.1.5

2. **GHSA-cpwx-vrp4-4pq7** - Sandbox breakout through attr filter (CVE-2025-27516)
   - **Severity**: Medium/High (CVSS 8.8)
   - **Fix**: Update to Jinja2 >= 3.1.6

3. **GHSA-gmj6-6f8f-6699** - Sandbox breakout through malicious filenames (CVE-2024-56201)
   - **Severity**: Medium/High (CVSS 8.8)
   - **Fix**: Update to Jinja2 >= 3.1.5

## Risk Assessment:
- **Overall Risk**: 🟡 **MEDIUM**
- **Immediate Action**: Update Jinja2 to 3.1.6+
- **All vulnerabilities are sandbox bypass issues in Jinja2**

## Recommendation:
Update `requirements.txt`: `jinja2>=3.1.6`