# URL Discovery

Devloop discovers the target URL by parsing Traefik labels in the project's compose file, with fallback to interactive selection when ambiguous.

## Algorithm (v0.1: single-match path only)

1. **Find compose file:** Look for `compose.yml`, `compose.yaml`, `docker-compose.yml`, or `docker-compose.yaml` in project root.

2. **Parse Traefik host labels** using a YAML parser (never regex YAML). Example with `yq`:

   ```bash
   yq '.services[].labels[]? | select(test("traefik\\.http\\.routers\\..*\\.rule"))' compose.yml
   ```

   Or with Python:

   ```bash
   python3 -c "
   import yaml, re, sys
   data = yaml.safe_load(open('compose.yml'))
   hosts = []
   for svc, cfg in (data.get('services') or {}).items():
       for label in (cfg.get('labels') or []):
           m = re.match(r'traefik\.http\.routers\.([^.]+)\.rule=Host\(\`([^\`]+)\`\)', label)
           if m:
               hosts.append((svc, m.group(1), m.group(2)))
   for h in hosts: print(h)
   "
   ```

3. **Filter:** Keep only entries with HTTP routers (skip TCP/UDP). Skip routers whose service has `tls.passthrough=true` (typically not the dev frontend).

4. **Resolve scheme:** Default `https://` if a sibling label `traefik.http.routers.<name>.tls=true` exists, else `http://`.

5. **Construct URL:** `<scheme>://<hostname>` — no path, no port (Traefik handles routing).

## Outcomes

- **Single URL discovered:** Use it directly.
- **Multiple URLs (deferred to v0.3):** For v0.1, abort with a message listing the candidates and asking the user to remove ambiguity in the compose file or invoke devloop after picking one. Do NOT silently pick.
- **Zero URLs:** Abort. Run `mx router inspect` and `mx ps`, dump output, ask the user to provide the URL manually.

## Reachability check

Before dispatching the first subagent:

```bash
curl -k -I -m 5 <url>
```

Retry every 5s for 30s total. Acceptable: any 2xx or 3xx response. Treat 401/403 as reachable (auth-gated apps are normal). Anything else after the timeout → abort with output of `mx router status` and `mx ps`.

The `-k` flag accepts self-signed certs (common in local Traefik setups).
