# RUCKUS One AP Reboot Manager

A tool for managing Access Point reboots across RUCKUS One deployments. Safely reboot hundreds or thousands of APs with built-in safety features, progress tracking, and resume capability.

## Features

- **Bulk Export** - Export all APs from your tenant to CSV with current status
- **Smart Filtering** - Review and filter APs before rebooting
- **Simulation Mode** - Test your reboot sequence without making changes
- **Batch Processing** - Reboot APs with configurable delays between operations
- **Resume Capability** - Automatically resume interrupted operations from checkpoints
- **Visual Feedback** - Color-coded output with progress indicators and countdown timers
- **Safety Controls** - Built-in limits and confirmations for large-scale operations

## Prerequisites

- Python 3.7 or higher
- RUCKUS One API credentials
- Network access to RUCKUS One API endpoints

## Installation

1. Clone this repository:
```bash
git clone https://github.com/yourusername/ap-reboot-manager.git
cd ap-reboot-manager
```

2. Install dependencies:
```bash
pip install requests
```

3. Configure your API credentials by copying the example file:
```bash
cp config.ini.example config.ini
```

4. Edit `config.ini` with your RUCKUS One credentials:
```ini
[credentials]
client_id = your-client-id
client_secret = your-client-secret
tenant_id = your-tenant-id
region = na  # Options: na, eu, asia
```

## Quick Start Guide

### Basic Workflow

The typical workflow consists of four steps:

1. **Export** - Get all APs from your tenant
2. **Review** - Filter the CSV to select which APs to reboot
3. **Simulate** - Test your reboot plan without making changes
4. **Execute** - Perform the actual reboots

### Step 1: Export APs to CSV

Export all Access Points from your tenant:

```bash
python3 ap_reboot_manager.py --config config.ini --export
```

This creates a timestamped CSV file (e.g., `ap_export_20250811_123456.csv`) containing:
- Serial number, MAC address, model, firmware version
- AP name, venue ID, venue name
- Current IP address and operational status

### Step 2: Review and Edit CSV

Open the exported CSV and:
- Remove rows for APs you don't want to reboot
- Verify AP status (only operational APs will be rebooted)
- Check venue assignments are correct
- Save your edited CSV

### Step 3: Simulate the Reboot

Always test with simulation mode first:

```bash
python3 ap_reboot_manager.py --config config.ini --import your_aps.csv --simulate
```

Simulation shows exactly what would happen without making any changes.

### Step 4: Execute the Reboot

Perform the actual reboot with appropriate delay:

```bash
# For more than 100 APs, add --force flag
python3 ap_reboot_manager.py --config config.ini --import your_aps.csv --delay 60 --force
```

## Detailed Command Line Options

### Required Arguments

| Option | Description |
|--------|-------------|
| `--config CONFIG` | Path to config.ini file with RUCKUS One credentials |

### Mode Selection (choose one)

| Option | Description |
|--------|-------------|
| `--export` | Export all APs to CSV with current status and details |
| `--import FILE` | Import CSV file and reboot APs listed in it |

### Operation Options

| Option | Default | Description |
|--------|---------|-------------|
| `--simulate` | False | Simulate mode: show what would be done without actual reboots |
| `--delay SECONDS` | 2 | Delay between reboots in seconds (shows countdown) |
| `--force` | False | Required safety flag when rebooting more than 100 APs |
| `--skip-status-check` | False | Skip runtime status verification and trust CSV status (faster, less safe) |

### Output Options

| Option | Default | Description |
|--------|---------|-------------|
| `--output FILE` | auto-generated | Custom output filename for export |
| `--log-level LEVEL` | INFO | Set logging verbosity (DEBUG, INFO, WARNING, ERROR) |
| `--log-file FILE` | None | Write logs to specified file in addition to console |

### Recovery Options

| Option | Default | Description |
|--------|---------|-------------|
| `--resume` | False | Resume from last checkpoint after interruption |
| `--batch-size N` | 50 | Number of APs to process before saving checkpoint |

## Usage Examples

### Export Operations

```bash
# Export with auto-generated filename
python3 ap_reboot_manager.py --config config.ini --export

# Export to specific file
python3 ap_reboot_manager.py --config config.ini --export --output office_aps.csv

# Export with debug logging
python3 ap_reboot_manager.py --config config.ini --export --log-level DEBUG
```

### Import and Reboot Operations

```bash
# Simulate reboot (always do this first!)
python3 ap_reboot_manager.py --config config.ini --import aps.csv --simulate

# Basic reboot with default 2-second delay
python3 ap_reboot_manager.py --config config.ini --import aps.csv

# Reboot with 5-minute delay between APs
python3 ap_reboot_manager.py --config config.ini --import aps.csv --delay 300

# Reboot more than 100 APs
python3 ap_reboot_manager.py --config config.ini --import aps.csv --delay 60 --force

# Skip runtime status check for faster operation
python3 ap_reboot_manager.py --config config.ini --import aps.csv --skip-status-check

# Resume after interruption
python3 ap_reboot_manager.py --config config.ini --import aps.csv --resume --force
```

### Complete Workflow Example

```bash
# 1. Export all APs
python3 ap_reboot_manager.py --config config.ini --export --output maintenance.csv

# 2. Edit maintenance.csv (remove APs you don't want to reboot)

# 3. Simulate to verify  
python3 ap_reboot_manager.py --config config.ini --import maintenance.csv --simulate --delay 30

# 4. Execute with 5-minute delays
python3 ap_reboot_manager.py --config config.ini --import maintenance.csv --delay 300 --force

# 5. If interrupted, resume
python3 ap_reboot_manager.py --config config.ini --import maintenance.csv --delay 300 --force --resume
```

## The Actual Reboot Process

### What Happens During a Reboot

1. **Status Check** (unless `--skip-status-check`):
   - Fetches current AP status from API
   - Compares with CSV status
   - Only proceeds if AP is operational

2. **Reboot Command**:
   - Sends PATCH request to `/venues/{venueId}/aps/{serialNumber}/systemCommands`
   - Request body: `{"type": "REBOOT"}`
   - Retries up to 3 times with exponential backoff on failure

3. **Progress Tracking**:
   - Updates console with color-coded status
   - Saves checkpoint every 50 APs (configurable)
   - Shows countdown timer during delays

4. **Completion**:
   - Displays summary tables
   - Removes checkpoint file if fully completed
   - Logs final statistics

### Safety Features

#### Operational Status Check
- Only reboots APs with operational status (starting with `2_`)
- Skips APs that are offline or disconnected
- Reports status changes between CSV export and reboot time

#### Limits and Confirmations
- **100 AP Limit**: Requires `--force` flag for more than 100 APs
- **1000 AP Confirmation**: Interactive confirmation for more than 1000 APs
- **Graceful Shutdown**: First CTRL+C saves checkpoint, second forces exit

#### Resume Capability
- Checkpoints saved every 50 APs (configurable with `--batch-size`)
- Stores progress, statistics, and failed AP list
- Resume exactly where you left off with `--resume` flag

## Visual Feedback

### Color-Coded Output
- **Cyan**: AP names (bold)
- **Yellow**: Serial numbers and warnings
- **Green**: Successful operations
- **Red**: Errors and failures
- **White**: Headers and summaries

### Progress Indicators
Shows current position and percentage:
```
[15/100] (15.0%) Processing AP: Lobby-R770-201 (SN: 431287956842)
```

### Countdown Timer
Visual countdown during delays:
```
Waiting 60s before next AP: .........10.........20.........30.........40.........50.........60
```

### Summary Tables

**Successfully Rebooted APs:**
```
AP Name                        Serial Number        Venue ID                            
--------------------------------------------------------------------------------
Lobby-R770-201                 431287956842         a8d2e7f9b14c45a18de5c7a94f62b1e9    
Conference-R770-105            298641573947         a8d2e7f9b14c45a18de5c7a94f62b1e9    
```

**Skipped APs (Not Operational):**
```
AP Name                   Serial Number        Status                             
--------------------------------------------------------------------------------
T350-08                   847392614758         3_04_DisconnectedFromCloud         
T350-12                   619845372901         3_04_DisconnectedFromCloud         
```

## CSV Format

The export creates a CSV with these columns:

| Column | Description | Used for Reboot |
|--------|-------------|-----------------|
| serial_number | AP serial number | Yes (primary ID) |
| mac_address | AP MAC address | No |
| model | Hardware model | No |
| firmware_version | Current firmware | No |
| name | AP display name | Display only |
| venue_id | Venue UUID | Yes (required) |
| venue_name | Human-readable venue | Display only |
| ip_address | Current IP | Display only |
| status | Operational status | Yes (must be operational) |

**Important**: Only APs with status containing "Operational" or starting with "2_" will be rebooted.

## Troubleshooting

### Common Issues

**Authentication Errors**
- Verify credentials in config.ini
- Check region setting (na, eu, asia)
- Ensure API access is enabled for your account

**Validation Errors**
- Use `--skip-status-check` to bypass API validation
- Verify venue IDs are correct in CSV
- Check AP serial numbers are valid

**Network Timeouts**
- Increase delay between reboots
- Use smaller batch sizes with `--batch-size`
- Check network connectivity to RUCKUS One API

**Large Deployments**
- Process in smaller batches by filtering CSV
- Use `--skip-status-check` for better performance
- Consider multiple maintenance windows

### Debug Logging

Enable detailed logging for troubleshooting:

```bash
python3 ap_reboot_manager.py --config config.ini --import aps.csv --log-level DEBUG --log-file debug.log
```

## Performance Considerations

- **Status Checking**: Adds ~1 second per unique venue
- **Pre-caching**: Optimizes checks for multiple APs in same venue  
- **Skip Status Check**: Use `--skip-status-check` when CSV is recent
- **Default Delay**: 2 seconds balances speed with network stability
- **Checkpoints**: Save every 50 APs to protect against data loss

## Security Best Practices

- Store `config.ini` securely - never commit to version control
- Use `.gitignore` to exclude sensitive files
- Enable logging with `--log-file` for audit trails
- Test with `--simulate` before production use
- Coordinate with network operations for large deployments
- Consider user impact before mass reboots

## API Implementation Details

### Reboot Endpoint
```
PATCH /venues/{venueId}/aps/{serialNumber}/systemCommands
Body: {"type": "REBOOT"}
```

### Response Codes
- **200**: Reboot command accepted
- **404**: AP not found
- **400**: Invalid request or AP cannot be rebooted
- **401**: Authentication failed
- **429**: Rate limit exceeded

### Retry Logic
- 3 attempts with exponential backoff
- 2 seconds, 4 seconds, 8 seconds between retries
- Logs all retry attempts

## Contributing

Contributions are welcome! Please feel free to submit issues or pull requests.

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## Support

For issues or questions, please open an issue on GitHub.

---

*Built for network administrators who need reliable, safe, and efficient AP management at scale.*
