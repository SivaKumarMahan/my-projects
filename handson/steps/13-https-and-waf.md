# Part 13 — HTTPS and WAF

[← Part 12 — GitHub Actions CI/CD](12-github-actions-cicd.md) | [Index](../README.md) | [Part 14 — Monitoring with Azure Monitor →](14-azure-monitor.md)

---

## 13.1 Self-signed certificate on App Gateway

There's no real domain, so use a self-signed certificate. `nip.io` gives a hostname for any IP (`1.2.3.4.nip.io` → 1.2.3.4).

```bash
# laptop
source ~/three-tier-env.sh
APPGW_IP=$(az network public-ip show -g $RG -n pip-ttapp-dev-appgw --query ipAddress -o tsv)
HOST="$APPGW_IP.nip.io"

openssl req -x509 -nodes -days 30 -newkey rsa:2048 -keyout tls.key -out tls.crt \
  -subj "/CN=$HOST" -addext "subjectAltName=DNS:$HOST"
openssl pkcs12 -export -out tls.pfx -inkey tls.key -in tls.crt -passout pass:LabOnly-123

az network application-gateway ssl-cert create -g $RG --gateway-name $APPGW_NAME \
  -n ttapp-selfsigned --cert-file tls.pfx --cert-password 'LabOnly-123'
rm tls.key tls.pfx      # never keep private keys around (and .gitignore blocks them)
```

Set the GitHub variable **`TLS_ENABLED` = `true`** and re-run the **app-ci-cd** workflow (Run workflow on `main`). The Ingress gets two annotations:

- `appgw-ssl-certificate: ttapp-selfsigned`: AGIC creates an HTTPS listener on 443 with that certificate.
- `ssl-redirect: "true"`: AGIC adds an HTTP→HTTPS redirect.

```bash
curl -sI http://$APPGW_IP/ | head -3          # 301 Moved Permanently → https://
curl -sk https://$APPGW_IP/api/info; echo     # -k because the certificate is self-signed
```

**TLS termination.** HTTPS ends at App Gateway; App Gateway → pod is HTTP inside the VNet. For end-to-end TLS you would also give the pods a certificate and set `appgw.ingress.kubernetes.io/backend-protocol: "https"`.

> The certificate lives on the App Gateway. When you destroy and recreate the platform stack, upload it again (or set `TLS_ENABLED=false`). The production way is a Key Vault certificate referenced by App Gateway through a managed identity, with auto-renewal. Then expiry is handled by Key Vault, which answers "how do you make sure an expiring certificate doesn't break the app?" See Appendix C.

## 13.2 Test the WAF

```bash
# Normal request: 200
curl -s -o /dev/null -w "%{http_code}\n" "http://$APPGW_IP/api/todos"

# SQL injection pattern: blocked by the OWASP 3.2 managed rules → 403
curl -s -o /dev/null -w "%{http_code}\n" "http://$APPGW_IP/api/todos?id=1%27%20OR%20%271%27=%271"

# XSS pattern → 403
curl -s -o /dev/null -w "%{http_code}\n" "http://$APPGW_IP/?q=<script>alert(1)</script>"

# Our custom rule (User-Agent contains "BadBot") → 403
curl -s -o /dev/null -w "%{http_code}\n" -A "BadBot/1.0" "http://$APPGW_IP/"
```

See *why* it was blocked: **Log Analytics → Logs** (logs arrive in ~5 minutes):

```kusto
AGWFirewallLogs
| where TimeGenerated > ago(1h)
| where Action == "Blocked" or Action == "Matched"
| project TimeGenerated, ClientIp, RequestUri, RuleId, Message, Action
| order by TimeGenerated desc
```

If a legitimate request is blocked (a **false positive**): find the `RuleId` here, then add an **exclusion** or disable that rule in the WAF policy, in Terraform. Run new rules in `Detection` mode first, then switch to `Prevention`.

**"How do you restrict users before requests reach the application?"** Layer by layer: WAF managed and custom rules (OWASP, bots, geo-match, rate limiting), then NSG on the App Gateway subnet, then App Gateway listener and host rules, then NetworkPolicy in the cluster, then app-level authentication.

---

[← Part 12 — GitHub Actions CI/CD](12-github-actions-cicd.md) | [Index](../README.md) | [Part 14 — Monitoring with Azure Monitor →](14-azure-monitor.md)
