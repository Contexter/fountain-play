# Automated Setup for Fountain Microservices with GoAccess Monitoring

This script automates the setup of PostgreSQL schemas, PostgREST configurations, Nginx virtual hosts with SSL, and GoAccess log monitoring. It's designed to streamline the deployment process for a microservices architecture, ensuring each component is correctly configured and secured.

## Script Overview

The script performs the following actions:

- Prompts for PostgreSQL database details and the base domain for SSL certificate setup.
- Iterates through a predefined list of services, setting up PostgreSQL schemas and importing table structures.
- Configures PostgREST for each schema, ensuring isolated API endpoints.
- Sets up Nginx virtual hosts, configures SSL with Certbot, and specifies log paths for GoAccess monitoring.
- Automatically creates GoAccess configuration files and systemd services for real-time log analysis.

``` 
#!/bin/bash

# Setup automation for Fountain Microservices with GoAccess Monitoring

# PostgreSQL details
read -p "Enter PostgreSQL database name: " dbname
read -p "Enter PostgreSQL user: " dbuser
echo -n "Enter PostgreSQL password: " && read -s dbpass && echo
read -p "Enter base domain (e.g., example.com): " basedomain
echo -n "Enter your email for SSL certificates: " && read email

# Directories and configuration paths
NGINX_VHOSTS_DIR="/etc/nginx/sites-enabled"
GOACCESS_CONF_DIR="/etc/goaccess"
GOACCESS_SYSTEMD_DIR="/etc/systemd/system"
SQL_PATH="/path/to/fountain_microservices_bootstrap.sql" # Update this path

# Ensure directories exist
mkdir -p "$GOACCESS_CONF_DIR" "$NGINX_VHOSTS_DIR"

# Service definitions
services=("playwright" "metadata" "script" "act" "scene" "character" 
"dialogue" "action" "transition" "parenthetical" "note" "centeredtext" 
"pagebreak" "sectionheading" "titlepage" "casting" "characterrelationship" 
"musicsound" "props" "revisionhistory" "formattingrules" "crossreferences" 
"extendednotesresearch" "scenelocation")

# Function to create PostgREST and Nginx configs, enable SSL, and setup GoAccess
setup_service() {
  local schema=$1
  local service_port=$((3000 + $2))
  local domain="api-$schema.$basedomain"

  # Create PostgreSQL schema
  PGPASSWORD=$dbpass psql -h localhost -U $dbuser -d $dbname -c "CREATE SCHEMA IF NOT EXISTS $schema;"

  # Import table structure for the schema
  PGPASSWORD=$dbpass psql -h localhost -U $dbuser -d $dbname -c "\i $SQL_PATH"

  # Create PostgREST configuration
  cat > "/etc/postgrest/$schema.conf" <<EOF
db-uri = "postgres://$dbuser:$dbpass@localhost:5432/$dbname"
db-schema = "$schema"
db-anon-role = "web_anon"
server-port = $service_port
EOF

  # Start PostgREST
  postgrest "/etc/postgrest/$schema.conf" &

  # Setup Nginx configuration
  local nginx_conf="/etc/nginx/sites-available/$domain"
  local access_log_path="/var/log/nginx/$domain-access.log"
  local error_log_path="/var/log/nginx/$domain-error.log"

  cat > "$nginx_conf" <<EOF
server {
    listen 80;
    server_name $domain;

    access_log $access_log_path;
    error_log $error_log_path;

    location / {
        proxy_pass http://localhost:$service_port;
        proxy_set_header Host \$host;
        proxy_set_header X-Real-IP \$remote_addr;
        proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto \$scheme;
    }
}
EOF

  ln -s "$nginx_conf" "/etc/nginx/sites-enabled/"

  # Enable SSL with Certbot
  certbot --nginx -d $domain --non-interactive --agree-tos -m $email --redirect

  # Setup GoAccess monitoring for Nginx logs
  setup_goaccess "$access_log_path" "$domain"
  setup_goaccess "$error_log_path" "$domain"
}

# Function to setup GoAccess monitoring
setup_goaccess() {
  local log_path=$1
  local name=$2
  local conf_path="$GOACCESS_CONF_DIR/goaccess-$name.conf"
  local systemd_service="$GOACCESS_SYSTEMD_DIR/goaccess-$name.service"

  # Create GoAccess config if it doesn't exist
  if [ ! -f "$conf_path" ]; then
    echo "log-file $log_path" > "$conf_path"
    echo "log-format COMBINED" >> "$conf_path"
  fi

  # Create systemd service for GoAccess
  if [ ! -f "$systemd_service" ]; then
    cat > "$systemd_service" <<EOF
[Unit]
Description=GoAccess Real-time Web Log Analyzer for $name
After=network.target

[Service]
Type=simple
ExecStart=/usr/bin/goaccess -f $log_path -a -o /var/www/html/$name.html --real-time-html -p $conf_path

[Install]
WantedBy=multi-user.target
EOF

    systemctl daemon-reload
    systemctl start "goaccess-$name"
    systemctl enable "goaccess-$name"
  fi
}

# Main setup loop for each service
for i in "${!services[@]}"; do
  setup_service "${services[$i]}" "$i"
done

# Reload Nginx to apply new configurations
nginx -t && systemctl reload nginx

echo "Setup completed. Fountain microservices and GoAccess monitoring are configured."
```

## Important Notes

1. **SQL File Path**: Ensure the `SQL_PATH` variable points to your actual SQL file containing the commands to create tables and other structures within each schema.
2. **Permissions**: This script requires root privileges for operations like creating configuration files in `/etc`, managing systemd services, and setting up SSL certificates with Certbot.
3. **Email for Certbot**: A valid email address is necessary for SSL certificate setup with Certbot.
4. **Testing**: Test this script in a non-production environment to ensure it meets your needs and adjust any paths or settings specific to your setup.
5. **PostgreSQL Credentials**: The script prompts for PostgreSQL credentials and other configurations. Ensure these are entered correctly to avoid issues with database access and schema setup.

The script is ready for deployment and intended to provide a comprehensive back-end infrastructure setup, complete with monitoring capabilities.