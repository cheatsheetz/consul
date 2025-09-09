# Consul Cheat Sheet

## Installation
```bash
# Download and install
curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo apt-key add -
sudo apt-add-repository "deb [arch=amd64] https://apt.releases.hashicorp.com $(lsb_release -cs) main"
sudo apt-get update && sudo apt-get install consul

# Docker
docker run -d --name consul -p 8500:8500 consul:latest

# Start in dev mode
consul agent -dev -ui -client 0.0.0.0
```

## Basic Commands
```bash
# Start agent
consul agent -config-dir=/etc/consul.d

# Join cluster
consul join 192.168.1.100

# Members list
consul members

# Leave cluster
consul leave

# Force remove member
consul force-leave node-name

# Check cluster info
consul info
consul monitor
```

## Service Registration
```bash
# Register service via API
curl -X PUT http://localhost:8500/v1/agent/service/register \
  -d '{
    "ID": "web-01",
    "Name": "web",
    "Port": 80,
    "Check": {
      "HTTP": "http://localhost:80/health",
      "Interval": "10s"
    }
  }'

# Register via JSON file
cat > web-service.json << EOF
{
  "service": {
    "id": "web-01",
    "name": "web",
    "port": 80,
    "check": {
      "http": "http://localhost:80/health",
      "interval": "10s",
      "timeout": "3s"
    }
  }
}
EOF

consul services register web-service.json

# Deregister service
curl -X PUT http://localhost:8500/v1/agent/service/deregister/web-01
```

## Service Discovery
```bash
# List services
consul catalog services
curl http://localhost:8500/v1/catalog/services

# Get service details
consul catalog nodes -service=web
curl http://localhost:8500/v1/catalog/service/web

# DNS queries
dig @localhost -p 8600 web.service.consul
dig @localhost -p 8600 web.service.consul SRV

# Health checks
curl http://localhost:8500/v1/health/service/web
curl http://localhost:8500/v1/health/service/web?passing
```

## Key-Value Store
```bash
# Put key-value pairs
consul kv put config/database/host "db.example.com"
consul kv put config/database/port "5432"

# Get values
consul kv get config/database/host
consul kv get -recurse config/

# Delete keys
consul kv delete config/database/host
consul kv delete -recurse config/

# Export/Import
consul kv export > backup.json
consul kv import @backup.json

# API operations
curl -X PUT -d "db.example.com" http://localhost:8500/v1/kv/config/database/host
curl http://localhost:8500/v1/kv/config/database/host
curl -X DELETE http://localhost:8500/v1/kv/config/database/host
```

## Configuration
```json
// /etc/consul.d/consul.json
{
  "datacenter": "dc1",
  "data_dir": "/opt/consul",
  "log_level": "INFO",
  "server": true,
  "bootstrap_expect": 3,
  "bind_addr": "192.168.1.100",
  "client_addr": "0.0.0.0",
  "retry_join": ["192.168.1.101", "192.168.1.102"],
  "ui_config": {
    "enabled": true
  },
  "connect": {
    "enabled": true
  },
  "acl": {
    "enabled": true,
    "default_policy": "deny",
    "enable_token_persistence": true
  }
}
```

## Health Checks
```json
{
  "service": {
    "name": "web",
    "port": 80,
    "checks": [
      {
        "http": "http://localhost:80/health",
        "interval": "10s"
      },
      {
        "tcp": "localhost:80",
        "interval": "30s"
      },
      {
        "script": "/opt/health-check.sh",
        "interval": "60s"
      }
    ]
  }
}
```

## ACLs and Security
```bash
# Bootstrap ACLs
consul acl bootstrap

# Create policy
consul acl policy create \
  -name "node-policy" \
  -description "Node access policy" \
  -rules @node-policy.hcl

# Create token
consul acl token create \
  -description "Node token" \
  -policy-name "node-policy"

# Set agent token
consul acl set-agent-token default <token>

# Example policy file (node-policy.hcl)
node_prefix "" {
  policy = "write"
}

service_prefix "" {
  policy = "read"
}
```

## Consul Connect
```json
// Service mesh configuration
{
  "service": {
    "name": "web",
    "port": 8080,
    "connect": {
      "sidecar_service": {}
    }
  }
}

// Connect proxy
{
  "service": {
    "name": "web-sidecar-proxy",
    "kind": "connect-proxy",
    "proxy": {
      "destination_service_name": "web",
      "destination_service_id": "web",
      "local_service_port": 8080
    }
  }
}
```

## Consul Template
```bash
# Install consul-template
curl -LO https://releases.hashicorp.com/consul-template/0.32.0/consul-template_0.32.0_linux_amd64.zip
unzip consul-template_0.32.0_linux_amd64.zip
sudo mv consul-template /usr/local/bin/

# Example template (nginx.tpl)
upstream backend {
{{range service "web"}}
  server {{.Address}}:{{.Port}};
{{end}}
}

# Run consul-template
consul-template \
  -template="nginx.tpl:nginx.conf:systemctl reload nginx" \
  -consul-addr="localhost:8500"
```

## Monitoring
```bash
# Metrics endpoint
curl http://localhost:8500/v1/agent/metrics

# Health endpoint
curl http://localhost:8500/v1/status/leader
curl http://localhost:8500/v1/status/peers

# Log streaming
consul monitor -log-level=DEBUG

# Check cluster state
consul operator raft list-peers
```

## Backup and Restore
```bash
# Snapshot
consul snapshot save backup.snap

# Restore
consul snapshot restore backup.snap

# Inspect snapshot
consul snapshot inspect backup.snap

# Automated backup script
#!/bin/bash
DATE=$(date +%Y%m%d_%H%M%S)
consul snapshot save /backups/consul_${DATE}.snap
find /backups -name "consul_*.snap" -mtime +7 -delete
```

## Troubleshooting
```bash
# Check logs
journalctl -u consul -f

# Validate configuration
consul validate /etc/consul.d/

# Check connectivity
consul members -detailed
consul catalog nodes

# Debug DNS
dig @localhost -p 8600 consul.service.consul
nslookup consul.service.consul localhost

# Check ports
netstat -tlnp | grep consul
ss -tlnp | grep consul
```

## Official Links
- [Consul Documentation](https://www.consul.io/docs)
- [Service Discovery](https://www.consul.io/docs/discovery)
- [Key-Value Store](https://www.consul.io/docs/dynamic-app-config/kv)
- [Consul Connect](https://www.consul.io/docs/connect)