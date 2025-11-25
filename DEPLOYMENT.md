# Production Deployment Guide for Portainer

This guide will help you deploy the Rails Portfolio app to production using Portainer.

## Prerequisites

1. Portainer running on your home lab
2. Access to the server where Portainer is running
3. The master key from `config/master.key`

## Step 1: Prepare Your Code

1. **Get your Rails Master Key:**
   ```bash
   cat config/master.key
   ```
   Copy this value - you'll need it for the environment variable.

2. **Build and test the Docker image locally (optional but recommended):**
   ```bash
   docker build -t portfolio:latest .
   ```

## Step 2: Deploy via Portainer

### Option A: Using Portainer's Stack Editor (Recommended)

1. **Access Portainer:**
   - Log into your Portainer instance
   - Navigate to **Stacks** in the left sidebar
   - Click **Add stack**

2. **Configure the Stack:**
   - **Name:** `portfolio` (or your preferred name)
   - **Build method:** Select **Repository**
   - **Repository URL:** Your Git repository URL (or upload files manually)
   - **Compose path:** `docker-compose.production.yml`
   - Or use **Web editor** and paste the contents of `docker-compose.production.yml`

3. **Set Environment Variables:**
   - Click **Environment variables** or use the **env** section
   - Add the following variables:
     ```
     RAILS_MASTER_KEY=<your-master-key-from-step-1>
     RAILS_MAX_THREADS=5
     RAILS_LOG_LEVEL=info
     ```
   - **Important:** Never commit `RAILS_MASTER_KEY` to Git!

4. **Deploy:**
   - Click **Deploy the stack**
   - Wait for the build to complete
   - Check logs if there are any issues

### Option B: Using Git Repository

1. **Push your code to a Git repository** (GitHub, GitLab, etc.)

2. **In Portainer:**
   - Go to **Stacks** → **Add stack**
   - Select **Repository**
   - Enter your repository URL
   - Set **Compose path** to: `docker-compose.production.yml`
   - Add environment variables as shown in Option A
   - Enable **Auto-update** if you want automatic deployments on push

### Option C: Manual File Upload

1. **Prepare files on your server:**
   - Upload the entire project directory to your server
   - Or clone the repository on the server

2. **In Portainer:**
   - Go to **Stacks** → **Add stack**
   - Select **Web editor**
   - Paste the contents of `docker-compose.production.yml`
   - Set **Build method** to use the local files
   - Add environment variables

## Step 3: Configure Networking

### Internal Access (Port 80)
The app will be available on port 80 of the host. If you're using a reverse proxy (recommended):

### Using a Reverse Proxy (Nginx/Traefik/Caddy)

1. **Set up your reverse proxy** to forward requests to `portfolio_web:80` (internal Docker network)

2. **Update production.rb** if using SSL:
   ```ruby
   config.assume_ssl = true
   config.force_ssl = true
   ```

3. **Configure allowed hosts** in `config/environments/production.rb`:
   ```ruby
   config.hosts << "yourdomain.com"
   config.hosts << /.*\.yourdomain\.com/
   ```

### Direct Access
If accessing directly (not recommended for production):
- The app will be available at `http://your-server-ip:80`
- Make sure port 80 is open in your firewall

## Step 4: Database Setup

The app uses SQLite by default, which is stored in the `portfolio_storage` volume. The database will be automatically created and migrated on first startup via the `docker-entrypoint` script.

### Backup Strategy
To backup your SQLite database:
```bash
# From Portainer, open a console in the container
docker exec -it portfolio_web bash
# Or use Portainer's console feature

# Backup the database
cp /rails/storage/production.sqlite3 /rails/storage/backup-$(date +%Y%m%d).sqlite3
```

Or backup the entire volume:
```bash
docker run --rm -v portfolio_storage:/data -v $(pwd):/backup alpine tar czf /backup/portfolio-backup-$(date +%Y%m%d).tar.gz /data
```

## Step 5: Monitoring and Maintenance

### View Logs
- In Portainer, go to **Containers** → `portfolio_web` → **Logs**
- Or use: `docker logs portfolio_web -f`

### Restart the Application
- In Portainer: **Containers** → `portfolio_web` → **Restart**
- Or redeploy the stack

### Update the Application
1. Pull the latest code
2. In Portainer: **Stacks** → `portfolio` → **Editor** → Update and **Update the stack**
3. Or if using Git: Push to repository and Portainer will auto-update (if enabled)

### Health Checks
The container includes a health check that monitors `/up` endpoint. You can check status in Portainer's container view.

## Environment Variables Reference

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `RAILS_MASTER_KEY` | Yes | - | Master key from `config/master.key` |
| `RAILS_ENV` | No | `production` | Rails environment |
| `RAILS_MAX_THREADS` | No | `5` | Maximum Puma threads |
| `RAILS_LOG_LEVEL` | No | `info` | Log level (debug, info, warn, error, fatal) |
| `PORT` | No | `80` | Port for Rails server |
| `SOLID_QUEUE_IN_PUMA` | No | `true` | Enable background jobs in Puma |
| `JOB_CONCURRENCY` | No | `1` | Number of job workers |
| `WEB_CONCURRENCY` | No | `1` | Number of Puma workers |

## Troubleshooting

### Container won't start
1. Check logs in Portainer
2. Verify `RAILS_MASTER_KEY` is set correctly
3. Check if port 80 is already in use

### "key must be 16 bytes" or "ArgumentError" on startup
This error means `RAILS_MASTER_KEY` is missing, empty, or incorrectly formatted.

**Solution:**
1. **Get your master key:**
   ```bash
   cat config/master.key
   ```
   Copy the entire 32-character string (no spaces, no newlines)

2. **In Portainer:**
   - Go to your stack → **Editor** → **Environment variables**
   - Find `RAILS_MASTER_KEY`
   - Make sure the value is exactly 32 characters
   - Remove any leading/trailing spaces or newlines

3. **Verify in container:**
   ```bash
   # In Portainer, open container console
   echo $RAILS_MASTER_KEY | wc -c
   # Should output: 33 (32 chars + newline) or 32
   ```

4. **If still having issues:**
   - Try setting it directly in docker-compose.production.yml temporarily:
     ```yaml
     - RAILS_MASTER_KEY=cfa64967788d6caeee50b331bc0539a2  # Your actual key
     ```
   - Or use Portainer's **Secrets** feature instead of environment variables

### Database errors
1. Check volume permissions
2. Verify storage volume is mounted correctly
3. Check logs for migration errors

### Performance issues
1. Adjust `RAILS_MAX_THREADS` and `WEB_CONCURRENCY`
2. Monitor resource usage in Portainer
3. Consider upgrading server resources

### Can't access the app
1. Check firewall rules
2. Verify port mapping in Portainer
3. Check if reverse proxy is configured correctly
4. Review `config.hosts` in production.rb

## Security Considerations

1. **Never commit `RAILS_MASTER_KEY`** to Git
2. Use environment variables or Portainer secrets for sensitive data
3. Enable SSL/TLS via reverse proxy
4. Keep Docker images updated
5. Regularly backup your database
6. Use strong passwords for any admin interfaces
7. Consider using Docker secrets for sensitive environment variables

## Next Steps

- Set up automated backups
- Configure monitoring (e.g., Prometheus, Grafana)
- Set up log aggregation
- Configure email notifications for errors
- Set up CI/CD for automated deployments

