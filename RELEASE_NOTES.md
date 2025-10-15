# Airflow 3.x Support Release

This release adds comprehensive support for Apache Airflow 3.0+ while maintaining full backward compatibility with Airflow 2.x installations.

## Highlights

### Apache Airflow 3.x Support
The setup script now fully supports Airflow 3.x with automatic handling of all breaking changes and new requirements.

### Critical Bug Fixes
Fixed several bugs in the original script that could cause service startup failures and upgrade issues.

### Enhanced Service Management
Improved service file handling with proper sudo usage and support for new Airflow 3.x services.

---

## What's New

### 1. Airflow 3.x Compatibility

#### Automatic Provider Installation
When installing Airflow 3.x, the script automatically installs the `apache-airflow-providers-standard` package, which contains core operators that were moved out of the main Airflow package.

```bash
# Automatically executed for Airflow 3.x
pip install "apache-airflow-providers-standard" --constraint "${CONSTRAINT_URL}"
```

#### Python Version Validation
The script now validates Python version compatibility before installation:
- **Airflow 3.x**: Requires Python 3.9, 3.10, 3.11, or 3.12
- **Airflow 2.x**: Continues to support older Python versions
- Displays clear error messages for incompatible versions

#### New Systemd Services
Airflow 3.x introduces new components that require additional services:

**API Server (Required)**
- Service: `airflow-api-server`
- Purpose: Handles task execution API (replaces direct database access)
- Status: Automatically created and started for Airflow 3.x

**DAG Processor (Optional but Recommended)**
- Service: `airflow-dag-processor`
- Purpose: Improves DAG processing performance
- Status: Created for Airflow 3.x; user prompted whether to enable

Example service file for API server:
```ini
[Unit]
Description=Airflow API server daemon
After=network.target postgresql.service mysql.service
Wants=postgresql.service mysql.service

[Service]
EnvironmentFile=/path/to/airflow.env
User=airflow
Group=airflow
Type=simple
ExecStart= bash -c 'source /path/to/venv/bin/activate ; /path/to/venv/bin/airflow api-server'
Restart=on-failure
RestartSec=5s
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```

#### Smart Database Migration
The script now uses version-aware database commands:
- **Airflow 3.x**: `airflow db migrate` (only command available)
- **Airflow 2.7+**: `airflow db migrate`
- **Airflow 2.0-2.6**: `airflow db init` / `airflow db upgrade`

No manual intervention needed—the script detects the version and uses the correct command.

### 2. Critical Bug Fixes

#### Scheduler Service Template Bug (Critical)
**Location**: Line 248 (now line 298) in airflow_setup.sh
**Issue**: Environment file reference was incorrectly set as `#ENV_FILE` (commented out)
**Impact**: Scheduler service would fail to start due to missing environment variables
**Fix**: Changed to `$ENV_FILE` for proper variable expansion

**Before:**
```bash
EnvironmentFile=#ENV_FILE  # This was treated as a comment!
```

**After:**
```bash
EnvironmentFile=$ENV_FILE  # Correctly expands to actual file path
```

#### Duplicate Database Command
**Location**: Lines 171-176 in initialize_airflow_db function
**Issue**: `airflow db upgrade` was called twice
**Impact**: Unnecessary execution time and potential confusion
**Fix**: Removed duplicate command

#### Missing Sudo in Upgrade Mode
**Location**: Lines 196-208 in initialize_services function
**Issue**: Service file updates during upgrade mode weren't using sudo
**Impact**: Permission denied errors when updating system service files
**Fix**: Added `${sudo}` prefix to all service file operations

### 3. Enhanced Backup Process

The backup function now handles Airflow 3.x services:
- Backs up `airflow-api-server.service` if it exists
- Backs up `airflow-dag-processor.service` if it exists
- Gracefully stops all services before backup
- Checks if services are active before attempting to stop them

```bash
# New backup code
cp2 /etc/systemd/system/airflow-api-server.service "${BACKUP_PATH}/files"
cp2 /etc/systemd/system/airflow-dag-processor.service "${BACKUP_PATH}/files"

if systemctl is-active --quiet airflow-api-server; then
    ${sudo} systemctl stop airflow-api-server
fi
```

---

## Configuration Changes

### Updated Example Configuration

The `airflow_setup.ini` file has been updated with:
- Airflow version bumped from 2.7.3 to 3.0.0
- Updated paths to reflect new version
- Helpful comments about requirements
- Python version compatibility notes

**Key Changes:**
```ini
# Old configuration
AIRFLOW_VERSION=2.7.3
AIRFLOW_VENV=/home/airflow/venvs/airflow_2_7_3
AIRFLOW_HOME=/home/airflow/venvs/airflow_2_7_3/airflow

# New configuration
AIRFLOW_VERSION=3.0.0
AIRFLOW_VENV=/home/airflow/venvs/airflow_3_0_0
AIRFLOW_HOME=/home/airflow/venvs/airflow_3_0_0/airflow

# New comments added
# Airflow 3.x requires Python 3.9-3.12
# Note: Airflow 3.x uses 'airflow db migrate' exclusively
# Note: For Airflow 3.x, apache-airflow-providers-standard will be automatically installed
```

---

## Migration Guide

### For Current Users (Airflow 2.x → 3.x)

1. **Check Python Version**
   ```bash
   python3 --version
   # Must be 3.9, 3.10, 3.11, or 3.12 for Airflow 3.x
   ```

2. **Update Configuration File**
   - Copy your current `airflow_setup.ini`
   - Update version: `AIRFLOW_VERSION=3.0.0` (or desired 3.x version)
   - Update paths: `AIRFLOW_VENV`, `AIRFLOW_HOME`, `ENV_FILE`
   - Set `UPGRADE=True`
   - Point `OLD_*` variables to your current installation

3. **Backup First** (Recommended)
   The script will prompt you to backup. Always choose Yes:
   ```
   Do you want to back up old Airflow files? [Y/n] Y
   ```

4. **Run Setup Script**
   ```bash
   cd /path/to/airflow-mate/utils
   ./airflow_setup.sh airflow_setup.ini
   ```

5. **Verify Installation**
   ```bash
   # Check all services are running
   sudo systemctl status airflow-webserver
   sudo systemctl status airflow-scheduler
   sudo systemctl status airflow-api-server
   sudo systemctl status airflow-dag-processor  # if enabled

   # Access Airflow UI
   # Navigate to http://localhost:8081/airflow (or your configured port)
   ```

6. **Review DAGs**
   - Use `ruff` linter with AIR301/AIR302 rules to check for breaking changes
   - Update any imports from `airflow.models.*` to `airflow.sdk.*`
   - Test DAGs in a development environment first

### For New Users

Simply update the `AIRFLOW_VERSION` in `airflow_setup.ini` to your desired version (2.x or 3.x) and run:

```bash
./airflow_setup.sh airflow_setup.ini
```

The script handles all version-specific differences automatically.

---

## Technical Details

### Architecture Changes in Airflow 3.x

**Task Execution API**
- Workers no longer access the database directly
- All task execution operations go through the API server
- Improves security by isolating task execution
- Requires `airflow-api-server` service to be running

**Database Migration**
- Simplified to single `airflow db migrate` command
- Handles both fresh installations and upgrades
- Previous commands (`db init`, `db upgrade`) deprecated in 3.x

**Provider Packages**
- Core operators moved to separate package
- Must install `apache-airflow-providers-standard` for basic functionality
- Script handles this automatically

### Service Dependencies

**Airflow 2.x:**
- `airflow-webserver`
- `airflow-scheduler`

**Airflow 3.x:**
- `airflow-webserver`
- `airflow-scheduler`
- `airflow-api-server` (required)
- `airflow-dag-processor` (optional)

All services depend on PostgreSQL being available and properly configured.

---

## Backward Compatibility

**100% backward compatible** with Airflow 2.x installations:
- Existing configurations work without changes
- Appropriate commands used based on version detection
- No breaking changes to the script interface
- All new features are version-aware

---

## Testing Recommendations

Before deploying to production:

1. **Test in Non-Production Environment**
   - Set up a test instance with similar configuration
   - Test the upgrade process with a database backup
   - Verify all DAGs run correctly

2. **Validate DAG Compatibility**
   ```bash
   # Install ruff linter
   pip install ruff

   # Check for Airflow 3.x breaking changes
   ruff check --select AIR301,AIR302 /path/to/dags/
   ```

3. **Monitor Resource Usage**
   - API server adds a new process
   - DAG processor (if enabled) adds another process
   - Monitor memory and CPU usage

4. **Review Logs**
   ```bash
   # Watch service logs
   sudo journalctl -u airflow-webserver -f
   sudo journalctl -u airflow-scheduler -f
   sudo journalctl -u airflow-api-server -f
   sudo journalctl -u airflow-dag-processor -f
   ```

---

## Known Issues

None at this time.

---

## Links

- [Airflow 3.0 Official Documentation](https://airflow.apache.org/docs/apache-airflow/stable/)
- [Upgrading to Airflow 3](https://airflow.apache.org/docs/apache-airflow/stable/installation/upgrading_to_airflow3.html)
- [Airflow 3.0 Release Notes](https://airflow.apache.org/docs/apache-airflow/stable/release_notes.html)

---

## Contributors

Special thanks to everyone who contributed to this release!

---

## Questions or Issues?

If you encounter any problems or have questions:
1. Check the [CHANGELOG.md](CHANGELOG.md) for detailed changes
2. Review the [Migration Guide](#migration-guide) section above
3. Open an issue on GitHub with details about your setup and the problem

---

**Release Date**: TBD
**Compatibility**: Airflow 2.0+ and 3.0+
**Python Requirements**: 3.9-3.12 (for Airflow 3.x), 3.7+ (for Airflow 2.x)
