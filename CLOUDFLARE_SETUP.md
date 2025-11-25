# Cloudflare Tunnel Setup Guide

This guide will help you expose your Rails Portfolio app to the internet using Cloudflare Tunnels.

## Prerequisites

1. Cloudflare account with a domain
2. Cloudflare Tunnel (cloudflared) installed and configured
3. Your app running in Docker on port 80 (already done)

## Step 1: Configure Cloudflare SSL/TLS Settings

1. **Log into Cloudflare Dashboard**
   - Go to your domain → **SSL/TLS** → **Overview**

2. **Set Encryption Mode:**
   - Set to **"Full"** or **"Full (strict)"**
   - This ensures Cloudflare encrypts traffic between Cloudflare and your server
   - **Full (strict)** requires a valid SSL certificate on your origin (recommended if you have one)
   - **Full** works without a certificate on your origin (works with Cloudflare tunnels)

3. **Enable Always Use HTTPS:**
   - Go to **SSL/TLS** → **Edge Certificates**
   - Enable **"Always Use HTTPS"**

## Step 2: Create a Cloudflare Tunnel

### Option A: Using Cloudflare Dashboard (Recommended)

1. **Create a Tunnel:**
   - Go to **Zero Trust** → **Networks** → **Tunnels**
   - Click **Create a tunnel**
   - Choose **Cloudflared** (if running on your server) or **Docker** (if using Docker container)

2. **Install Cloudflared (if not using Docker):**
   ```bash
   # On your server
   curl -L https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64 -o /usr/local/bin/cloudflared
   chmod +x /usr/local/bin/cloudflared
   ```

3. **Configure the Tunnel:**
   - **Name:** `portfolio-tunnel` (or your preferred name)
   - **Subdomain:** Choose a subdomain (e.g., `portfolio`)
   - **Domain:** Select your domain
   - **Service:** `http://localhost:80` (or `http://portfolio_web:80` if using Docker network)
   - **Path:** Leave empty (routes all traffic)
   - **Additional application settings:**
     - **HTTP Host Header:** Leave empty or set to your domain
     - **Origin Server Name:** Leave empty

4. **Save and Deploy:**
   - Copy the tunnel token
   - Run the tunnel command on your server

### Option B: Using Docker Container (Recommended for Portainer)

1. **Add Cloudflare Tunnel to docker-compose.production.yml:**

   Add this service to your `docker-compose.production.yml`:

   ```yaml
   services:
     # ... existing web service ...
     
     cloudflared:
       image: cloudflare/cloudflared:latest
       container_name: portfolio_cloudflared
       restart: unless-stopped
       command: tunnel run
       environment:
         - TUNNEL_TOKEN=${CLOUDFLARE_TUNNEL_TOKEN}
       networks:
         - default
       depends_on:
         - web

   networks:
     default:
       name: portfolio_network
   ```

2. **Update your web service to use the network:**
   ```yaml
   services:
     web:
       # ... existing config ...
       networks:
         - default
   ```

3. **In Portainer:**
   - Add environment variable: `CLOUDFLARE_TUNNEL_TOKEN`
   - Get the token from Cloudflare Dashboard → Zero Trust → Networks → Tunnels
   - Copy the token from your tunnel configuration

### Option C: Using cloudflared Command Line

1. **Install cloudflared on your server**

2. **Create tunnel:**
   ```bash
   cloudflared tunnel create portfolio-tunnel
   ```

3. **Configure route:**
   ```bash
   cloudflared tunnel route dns portfolio-tunnel portfolio.yourdomain.com
   ```

4. **Run tunnel:**
   ```bash
   cloudflared tunnel run portfolio-tunnel
   ```

   Or create a config file at `~/.cloudflared/config.yml`:
   ```yaml
   tunnel: <tunnel-id>
   credentials-file: /path/to/credentials.json
   
   ingress:
     - hostname: portfolio.yourdomain.com
       service: http://localhost:80
     - service: http_status:404
   ```

## Step 3: Configure Rails for Your Domain

1. **Update `config/environments/production.rb`:**

   Uncomment and set your domain:
   ```ruby
   config.hosts << "portfolio.yourdomain.com"
   config.hosts << /.*\.yourdomain\.com/  # Allow all subdomains
   ```

   Or use an environment variable:
   ```ruby
   config.hosts << ENV.fetch("RAILS_HOST", "portfolio.yourdomain.com")
   ```

2. **Update mailer host (if using email):**
   ```ruby
   config.action_mailer.default_url_options = { 
     host: "portfolio.yourdomain.com",
     protocol: "https"
   }
   ```

3. **Rebuild and restart your container:**
   ```bash
   # In Portainer, redeploy the stack
   # Or via command line:
   docker-compose -f docker-compose.production.yml up -d --build
   ```

## Step 4: Verify the Setup

1. **Check tunnel status:**
   - In Cloudflare Dashboard → Zero Trust → Networks → Tunnels
   - Your tunnel should show as "Healthy"

2. **Test the connection:**
   ```bash
   curl -I https://portfolio.yourdomain.com
   ```
   Should return HTTP 200 or 302

3. **Visit in browser:**
   - Navigate to `https://portfolio.yourdomain.com`
   - You should see your Rails app

## Step 5: Configure Cloudflare Security Settings (Optional but Recommended)

1. **Firewall Rules:**
   - Go to **Security** → **WAF** → **Custom rules**
   - Create rules to block suspicious traffic

2. **Rate Limiting:**
   - Go to **Security** → **WAF** → **Rate limiting rules**
   - Set limits to prevent abuse

3. **Bot Fight Mode:**
   - Go to **Security** → **Bots**
   - Enable **Bot Fight Mode** (free) or **Super Bot Fight Mode** (paid)

4. **Page Rules (Optional):**
   - Cache static assets
   - Set security headers

## Troubleshooting

### Tunnel shows as "Unhealthy"
- Check if the container is running: `docker ps`
- Verify port 80 is accessible: `curl http://localhost:80/up`
- Check tunnel logs in Cloudflare Dashboard
- Verify `TUNNEL_TOKEN` is set correctly

### "Host not authorized" error
- Check `config.hosts` in `production.rb`
- Make sure your domain is added to the allowed hosts list
- Verify the Host header is being passed correctly

### SSL/HTTPS redirect loops
- Ensure `config.assume_ssl = true` is set
- Check Cloudflare SSL/TLS mode is set to "Full" or "Full (strict)"
- Verify `config.force_ssl = true` is enabled

### Can't access the app
- Check DNS records in Cloudflare
- Verify tunnel is running and healthy
- Check container logs: `docker logs portfolio_web`
- Verify port mapping in docker-compose

### 502 Bad Gateway
- Check if the Rails app is running: `docker logs portfolio_web`
- Verify the service URL in tunnel config matches your container
- Check if using Docker network, use service name: `http://portfolio_web:80`
- If using host network, use: `http://localhost:80`

## Advanced Configuration

### Using Environment Variables for Domain

Add to `docker-compose.production.yml`:
```yaml
environment:
  - RAILS_HOST=${RAILS_HOST:-portfolio.yourdomain.com}
```

Then in `production.rb`:
```ruby
config.hosts << ENV.fetch("RAILS_HOST")
```

### Multiple Domains/Subdomains

In `production.rb`:
```ruby
config.hosts << "portfolio.yourdomain.com"
config.hosts << "www.portfolio.yourdomain.com"
config.hosts << /.*\.yourdomain\.com/
```

### Custom Headers

Cloudflare can add custom headers. In tunnel config, you can set:
- `CF-Connecting-IP` - Original client IP
- `CF-Ray` - Request ID
- `CF-Visitor` - Protocol information

These are automatically added by Cloudflare.

## Security Best Practices

1. **Keep Cloudflare tunnel updated:**
   ```bash
   docker pull cloudflare/cloudflared:latest
   ```

2. **Use Cloudflare Access (Zero Trust):**
   - Protect admin routes
   - Require authentication for sensitive pages

3. **Enable Cloudflare Analytics:**
   - Monitor traffic and threats
   - Set up alerts for suspicious activity

4. **Regular Backups:**
   - Backup your database regularly
   - Store backups securely

5. **Monitor Logs:**
   - Check Rails logs for errors
   - Monitor Cloudflare analytics
   - Set up alerts for downtime

## Next Steps

- Set up Cloudflare Access for admin routes
- Configure caching rules for better performance
- Set up monitoring and alerts
- Configure custom error pages
- Enable Cloudflare Workers for edge computing (optional)

