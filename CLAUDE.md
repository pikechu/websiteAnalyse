# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Self-hosted web analytics setup using [Umami](https://github.com/umami-software/umami) v2.13.2 with MySQL, deployed behind an Nginx reverse proxy with SNI-based routing.

## Structure

- `nginx/` — Nginx config templates (domains and IPs replaced with placeholders)
- `.env.example` — Environment variable template for Umami
- `umami/` — Umami source (gitignored; clone from official repo at tag v2.13.2)

## Umami Setup

```bash
git clone https://github.com/umami-software/umami.git
cd umami && git checkout v2.13.2
cp ../.env.example .env  # fill in real values
npm install --legacy-peer-deps
npm run build-db          # generate Prisma client
npm run build-tracker
npm run build-geo
NODE_OPTIONS='--max-old-space-size=768' npm run build-app
npm run update-db         # apply DB migrations
pm2 start npm --name umami -- start
```

## Nginx Architecture

Port 443 is handled by an Nginx `stream` block that routes by SNI to backend HTTP servers on internal ports (8443, 8445, etc.). `proxy_protocol on` is required so backend servers can read the real client IP for geolocation.

Each backend server block must include:
```nginx
listen PORT ssl proxy_protocol;
set_real_ip_from 127.0.0.1;
real_ip_header proxy_protocol;
```
