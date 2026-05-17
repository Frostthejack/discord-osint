# Discord OSINT Commands

Discord slash commands for OSINT tools, added to the Hermes Agent Discord gateway.

## Commands

| Command | Tool | Description |
|---------|------|-------------|
| `/nmap` | nmap | Port scanning |
| `/httpx` | httpx | HTTP probing |
| `/dig` | dig | DNS lookup |
| `/whois` | whois | Domain WHOIS |
| `/ipinfo` | ip-api.com | IP geolocation |
| `/subfinder` | subfinder | Subdomain enumeration |
| `/gau` | gau | Get All URLs |
| `/waybackurls` | waybackurls | Wayback Machine URLs |
| `/exiftool` | exiftool | Metadata extraction |
| `/curl_headers` | curl | HTTP headers |
| `/hash_lookup` | hashlookup.io | Hash lookup |
| `/base64` | base64 | Encode/decode |
| `/nuclei` | nuclei | Vulnerability scanning |
| `/ghunt` | ghunt | Google account OSINT |
| `/shodan` | shodan | Device search |
| `/url_scan` | VirusTotal | URL scanning |
| `/jwt_decode` | jwt | JWT decoding |
| `/qr_decode` | qr | QR code decoding |

## Project Structure

- `src/` — Source code (symlinks to hermes-agent gateway)
- `docs/` — Symlink to vault project docs
- `project-state.md` — Symlink to vault project-state.md
