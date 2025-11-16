# Wetty SSH Terminal Setup

Web-based SSH terminal with rate limiting and authentication.

## Access URLs

- **Public (HTTPS)**: https://ssh.shampadsr.com
- **Local**: http://localhost:3002

## Features

### Authentication
- **Username prompt**: Enabled (no default user set)
- **Password prompt**: Enabled (required for SSH login)
- Users will be prompted for both username and password

### Rate Limiting
Configured in the containerized nginx with smart tier-based limiting:

**Static Files (CSS, JS, Images, Fonts)**
- **No rate limiting** - Essential for page functionality
- Files are cached for 1 hour to reduce server load
- Applies to: `.css`, `.js`, `.jpg`, `.jpeg`, `.png`, `.gif`, `.ico`, `.svg`, `.woff`, `.woff2`, `.ttf`, `.eot`

**WebSocket/Socket.io Connections** (SSH connection establishment)
- **Rate**: 30 requests per second per IP
- **Burst**: 10 requests
- **Max concurrent connections**: 5 per IP
- This is where actual SSH connections are established

**HTML and Other Requests** (Page loads)
- **Rate**: 30 requests per second per IP
- **Burst**: 100 requests (allows multiple refreshes)
- **Max concurrent connections**: 10 per IP

**Rate Limit Exceeded**
- Returns HTTP 429 with custom error page
- Auto-retry countdown timer (3 seconds)
- Rate limits are **per IP address** (not global)

### Security Headers
- X-Frame-Options: SAMEORIGIN
- X-Content-Type-Options: nosniff
- X-XSS-Protection: 1; mode=block

## Architecture

```
Internet → System nginx (443) → Containerized nginx (3002) → Wetty (3000) → SSH (172.17.0.1)
```

1. **System nginx** (`/etc/nginx/sites-enabled/shampadsr.com`):
   - Handles HTTPS/SSL termination
   - Proxies `ssh.shampadsr.com` → `localhost:3002`
   - Long timeout for SSH sessions (7 days)

2. **Containerized nginx** (port 3002):
   - Applies rate limiting
   - Adds security headers
   - Proxies to wetty container

3. **Wetty container** (port 3000):
   - Web terminal interface
   - Connects to SSH host at `172.17.0.1`

## Configuration Files

- [`docker-compose.yml`](docker-compose.yml) - Container orchestration
- [`nginx.conf`](nginx.conf) - Rate limiting and proxy configuration
- [`429.html`](429.html) - Custom rate limit error page
- `/etc/nginx/sites-enabled/shampadsr.com` - System nginx SSL/domain configuration

## Managing the Service

### Start services
```bash
docker-compose up -d
```

### Stop services
```bash
docker-compose down
```

### View logs
```bash
docker logs wetty-nginx
docker logs wetty
```

### Adjust rate limits
Edit [`nginx.conf`](nginx.conf):

**Global rate (line 8):**
```nginx
limit_req_zone $binary_remote_addr zone=wetty_limit:10m rate=30r/s;
```

**WebSocket connections (lines 53-54):**
```nginx
limit_req zone=wetty_limit burst=10 nodelay;
limit_conn wetty_conn 5;
```

**Page loads (lines 74-75):**
```nginx
limit_req zone=wetty_limit burst=100 nodelay;
limit_conn wetty_conn 10;
```

**To disable rate limiting entirely:**
Comment out or remove the `limit_req` and `limit_conn` directives in the location blocks.

Then restart:
```bash
docker-compose restart nginx
```

### Custom Error Page
The custom 429 error page ([`429.html`](429.html)) provides:
- User-friendly rate limit explanation
- Current rate limit details
- Auto-refresh countdown timer (3 seconds)
- Manual retry button

To customize, edit `429.html` and restart nginx.

## Troubleshooting

### Check if containers are running
```bash
docker ps --filter name=wetty
```

### Test local connection
```bash
curl -I http://localhost:3002
```

### Check nginx logs
```bash
docker logs wetty-nginx
sudo tail -f /var/log/nginx/access.log
sudo tail -f /var/log/nginx/error.log
```

### Reload system nginx
```bash
sudo nginx -t
sudo systemctl reload nginx
```
