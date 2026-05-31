# sub2clash

> **Status: Deprecated / Archived.** This project is no longer maintained.

`sub2clash` was a small Go service that pulled an ss/ssr/vmess subscription on a
cron schedule, parsed the base64 nodes, filtered them by an env-var white/black
list, queried the Clash controller for per-node latency, and wrote out a Clash
config file with regional `url-test` groups (JP / HK / US / SG …).

## Why it's deprecated

The whole pipeline has been absorbed by upstream Clash cores. With
[**mihomo**](https://github.com/MetaCubeX/mihomo) (the maintained successor to
the now-defunct `dreamacro/clash`), every responsibility of this project maps
to a built-in feature:

| sub2clash responsibility            | mihomo built-in                                       |
| ----------------------------------- | ----------------------------------------------------- |
| Cron-pull subscription              | `proxy-providers.interval`                            |
| Decode ss/ssr/vmess base64          | `proxy-providers` (also accepts Clash YAML natively)  |
| `SUB_WHITELIST` / `SUB_BLACKLIST`   | `proxy-providers.filter` / `exclude-filter` (regex)   |
| Query `/proxies` for delay sort     | `health-check` + `proxy-groups` `type: url-test`      |
| Per-region groups (JP / HK / US …)  | `proxy-groups.use` + `filter` per group               |

In other words: the Go binary is unnecessary — modern subscription endpoints
serve Clash YAML directly, and mihomo consumes it without a middleman.

## Recommended migration

Replace the three-container stack (`clash` + `yacd` + `sub2clash`) with two
containers:

```yaml
# compose.yaml
services:
  mihomo:
    image: metacubex/mihomo:latest
    container_name: mihomo
    restart: unless-stopped
    volumes:
      - /opt/clash:/root/.config/mihomo
    ports:
      - "7890:7890"   # mixed http+socks
      - "9090:9090"   # external-controller
    cap_add: [NET_ADMIN]
  metacubexd:
    image: ghcr.io/metacubex/metacubexd:latest
    container_name: metacubexd
    restart: unless-stopped
    depends_on: [mihomo]
    ports:
      - "7892:80"     # dashboard
```

And the relevant `config.yaml` skeleton:

```yaml
proxy-providers:
  main:
    type: http
    url: "<your subscription URL>"
    interval: 3600
    path: ./providers/main.yaml
    health-check:
      enable: true
      url: https://www.gstatic.com/generate_204
      interval: 600
    exclude-filter: "(?i)(剩余|到期|拉取于|官网|套餐|流量|失联|过期)"

proxy-groups:
  - {name: SELECT, type: select, proxies: [AUTO, JP, HK, TW, US, SG, DIRECT]}
  - {name: AUTO,   type: url-test, use: [main],
     url: https://www.gstatic.com/generate_204, interval: 600, tolerance: 50}
  - {name: JP, type: url-test, use: [main], filter: "(?i)(日本|JP|Japan)",
     url: https://www.gstatic.com/generate_204, interval: 600, tolerance: 50}
  # …repeat per region
```

## Source preserved as reference

The original Go implementation, `Dockerfile`, and the `clash-premium`/`yacd`
compose are kept in this repository as a historical snapshot. They are not
updated and should not be deployed against current subscription sources.
