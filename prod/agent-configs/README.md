# Wazuh Agent Configuration Guide

This directory contains optimized Wazuh agent configurations for different device types in your infrastructure.

## Configuration Files

1. **firewall-agent.conf** - For pfSense, OPNsense, and other firewall appliances
2. **sbc-agent.conf** - For Raspberry Pi, ZimaBlades, and other single-board computers
3. **kubernetes-container-agent.conf** - For Kubernetes nodes and containerized workloads
4. **server-agent.conf** - For Linux/Unix servers (web, database, application servers)
5. **endpoint-agent.conf** - For laptops, workstations, and desktop computers (Windows/Linux/Mac)

## Deployment Methods

### Method 1: Centralized Configuration (Recommended)

Upload configurations to Wazuh manager and assign via agent groups:

```bash
# Copy configuration to Wazuh manager
kubectl cp firewall-agent.conf wazuh/wazuh-manager-conductor-0:/var/ossec/etc/shared/firewall/agent.conf
kubectl cp sbc-agent.conf wazuh/wazuh-manager-conductor-0:/var/ossec/etc/shared/sbc/agent.conf
kubectl cp kubernetes-container-agent.conf wazuh/wazuh-manager-conductor-0:/var/ossec/etc/shared/kubernetes/agent.conf
kubectl cp server-agent.conf wazuh/wazuh-manager-conductor-0:/var/ossec/etc/shared/server/agent.conf
kubectl cp endpoint-agent.conf wazuh/wazuh-manager-conductor-0:/var/ossec/etc/shared/endpoint/agent.conf

# Set proper permissions
kubectl exec -n wazuh wazuh-manager-conductor-0 -- chown -R wazuh:wazuh /var/ossec/etc/shared/
kubectl exec -n wazuh wazuh-manager-conductor-0 -- chmod 660 /var/ossec/etc/shared/*/agent.conf
```

Then assign agents to groups via the Wazuh dashboard or API:

```bash
# Via Wazuh API
curl -k -X PUT "https://wazuh-api.diagon.cloud/agents/groups/{group_id}/agents/{agent_id}" \
  -H "Authorization: Bearer $TOKEN"

# Example: Assign agent 001 to firewall group
curl -k -X PUT "https://wazuh-api.diagon.cloud/agents/groups/firewall/agents/001" \
  -H "Authorization: Bearer $TOKEN"
```

### Method 2: Local Agent Configuration

Manually deploy to individual agents:

```bash
# On the agent machine
sudo cp agent.conf /var/ossec/etc/ossec.conf
sudo systemctl restart wazuh-agent
```

## Configuration Highlights

### Firewall Configuration
- **Focus**: Network traffic analysis, VPN logs, configuration changes
- **FIM**: Monitors `/conf` directory (pfSense/OPNsense)
- **Active Response**: **DISABLED** to prevent blocking critical services
- **Scan Frequency**: 12 hours (less aggressive for stability)
- **Resource Usage**: Medium

### SBC Configuration
- **Focus**: Resource-efficient monitoring with IoT-specific checks
- **Special Features**:
  - Raspberry Pi temperature monitoring (`vcgencmd measure_temp`)
  - Throttling detection (`vcgencmd get_throttled`)
  - Docker container monitoring for IoT workloads
- **FIM**: Daily scans to reduce CPU/storage usage
- **Buffer Size**: Reduced (2500 events) for limited resources
- **Resource Usage**: Low

### Kubernetes/Container Configuration
- **Focus**: Container security, Kubernetes audit logs, runtime monitoring
- **Special Features**:
  - Docker event monitoring
  - Privileged container detection
  - kubectl pod monitoring
  - CIS Kubernetes & Docker benchmarks
- **FIM**: Monitors K8s configs, container runtime configs
- **Buffer Size**: Large (10,000 events) for high log volume
- **Resource Usage**: High

### Server Configuration
- **Focus**: Comprehensive server monitoring, web services, databases
- **Special Features**:
  - Apache/Nginx log monitoring
  - MySQL/PostgreSQL monitoring
  - SSH brute force detection with active response
  - Network connection monitoring
  - Audit log analysis
- **FIM**: Extensive coverage including web roots, cron jobs, SSH configs
- **Active Response**: **ENABLED** with automatic blocking
- **Resource Usage**: Medium-High

### Endpoint Configuration
- **Focus**: User devices security monitoring (Windows/Linux/Mac)
- **Special Features**:
  - Cross-platform support (Windows Event Logs, Linux/Mac syslog)
  - USB device monitoring
  - Wireless network monitoring
  - Windows Defender status monitoring
  - PowerShell activity monitoring
  - Browser crash report monitoring
  - VPN connection monitoring
- **FIM**: OS-specific monitoring (Windows registry, Mac LaunchAgents, Linux systemd)
- **Active Response**: **ENABLED** with temporary blocking for brute force attacks
- **Resource Usage**: Medium (optimized to not impact user experience)

## Agent Profiles

Agents automatically apply configurations based on their profile. Set the profile during agent enrollment:

```bash
# During agent installation
sudo WAZUH_MANAGER='wazuh-manager.diagon.cloud' \
     WAZUH_AGENT_NAME='firewall-01' \
     WAZUH_AGENT_GROUP='firewall' \
     apt-get install wazuh-agent
```

Or modify after installation:

```bash
# Edit /var/ossec/etc/ossec.conf on agent
<client>
  <server>
    <address>wazuh-manager.diagon.cloud</address>
    <port>1514</port>
    <protocol>tcp</protocol>
  </server>
  <config-profile>firewall</config-profile>
  <enrollment>
    <groups>firewall</groups>
  </enrollment>
</client>
```

## Customization Tips

### Adjusting Scan Frequencies

For lower resource usage:
```xml
<syscheck>
  <frequency>86400</frequency> <!-- Change to daily -->
</syscheck>

<rootcheck>
  <frequency>172800</frequency> <!-- Change to every 2 days -->
</rootcheck>
```

### Adding Custom Log Sources

```xml
<localfile>
  <log_format>syslog</log_format>
  <location>/path/to/your/log.log</location>
</localfile>
```

### Disabling Resource-Intensive Features

For very constrained devices:
```xml
<wodle name="syscollector">
  <disabled>yes</disabled> <!-- Disable system inventory -->
</wodle>

<wodle name="vulnerability-detector">
  <disabled>yes</disabled> <!-- Disable vulnerability scanning -->
</wodle>
```

## Performance Tuning

### Buffer Sizes by Device Type

| Device Type | queue_size | events_per_second | Rationale |
|-------------|------------|-------------------|-----------|
| Firewall | 5000 | 250 | Moderate traffic logs |
| SBC | 2500 | 100 | Resource constrained |
| Kubernetes | 10000 | 500 | High container log volume |
| Server | 5000 | 250 | Standard server workload |
| Endpoint | 5000 | 200 | User device, balanced monitoring |

### Scan Frequency Recommendations

| Device Type | FIM Frequency | Rootcheck Frequency |
|-------------|---------------|---------------------|
| Firewall | 12h | 24h |
| SBC | 24h | 48h |
| Kubernetes | 6h | 12h |
| Server | 12h | 12h |
| Endpoint | 12h | 12h |

## Monitoring Agent Health

```bash
# Check agent status
kubectl exec -n wazuh wazuh-manager-conductor-0 -- /var/ossec/bin/agent_control -l

# Check agent connection
kubectl exec -n wazuh wazuh-manager-conductor-0 -- /var/ossec/bin/agent_control -i {agent_id}

# View agent logs on the agent
sudo tail -f /var/ossec/logs/ossec.log
```

## Troubleshooting

### Agent Not Connecting

1. Check firewall allows TCP 1514 (agent events) and TCP 1515 (enrollment)
2. Verify manager IP/hostname in agent config
3. Check agent authentication:
   ```bash
   sudo /var/ossec/bin/agent-auth -m wazuh-manager.diagon.cloud
   ```

### High CPU Usage

1. Reduce scan frequencies
2. Disable syscollector or vulnerability-detector
3. Reduce buffer sizes
4. Disable realtime FIM on large directories

### Missing Logs

1. Verify log file paths exist on agent
2. Check log file permissions (agent must be able to read)
3. Verify log format matches actual log format
4. Check agent logs for errors

### Active Response Not Working

1. Verify active-response is enabled in config
2. Check firewall allows manager → agent communication
3. Verify AR scripts are executable
4. Check agent logs for AR execution errors

## Security Best Practices

1. **Use agent groups** for centralized configuration management
2. **Enable active response** only on servers (not firewalls)
3. **Monitor agent health** regularly via dashboard
4. **Rotate agent keys** periodically for security
5. **Review alerts** and tune rules to reduce false positives
6. **Keep agents updated** to latest stable version
7. **Test configurations** in non-production first

## Additional Resources

- [Wazuh Documentation](https://documentation.wazuh.com/)
- [Agent Configuration Reference](https://documentation.wazuh.com/current/user-manual/reference/ossec-conf/index.html)
- [CIS Benchmarks](https://www.cisecurity.org/cis-benchmarks/)
- [Wazuh Ruleset](https://documentation.wazuh.com/current/user-manual/ruleset/index.html)
