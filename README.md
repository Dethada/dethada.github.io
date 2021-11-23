# Blog

## Development
Spin up development server

For WSL2
```bash
hugo -D -w --bind 0.0.0.0 --baseURL `ip addr show eth0 | grep "inet\b" | awk '{print $2}' | cut -d/ -f1`
```
