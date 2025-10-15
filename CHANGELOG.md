# Changelog

All notable changes to airflow-mate will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added
- **Airflow 3.x Support**: Full compatibility with Apache Airflow 3.0+
  - Automatic installation of `apache-airflow-providers-standard` package for Airflow 3.x
  - Python version validation (3.9-3.12 required for Airflow 3.x)
  - New systemd service files for `airflow-api-server` (required) and `airflow-dag-processor` (optional)
  - Automatic detection and management of Airflow 3.x specific services during backup and upgrade
  - Version-aware database migration commands (`airflow db migrate` for 3.x)

### Fixed
- **Critical Bug**: Fixed scheduler service template with incorrect environment file reference (`#ENV_FILE` → `$ENV_FILE`) in [airflow_setup.sh:248](utils/airflow_setup.sh#L248)
- **Duplicate Command**: Removed duplicate `airflow db upgrade` call in database initialization function
- **Missing Privileges**: Added proper sudo usage in service file updates during upgrade mode
- **Service Management**: Enhanced backup function to properly stop all Airflow services including 3.x specific ones

### Changed
- **Database Migration**: Updated database initialization logic to use appropriate commands based on Airflow version
  - Airflow 3.x: exclusively uses `airflow db migrate`
  - Airflow 2.7+: uses `airflow db migrate`
  - Airflow 2.0-2.6: uses `airflow db init` and `airflow db upgrade`
- **Service Creation**: Improved service file generation with proper sudo handling
- **Backup Process**: Extended backup functionality to include new Airflow 3.x service files
- **Configuration**: Updated example configuration from Airflow 2.7.3 to 3.0.0 with helpful comments

### Documentation
- Added inline comments explaining Airflow 3.x requirements and changes
- Updated configuration file with Python version compatibility notes
- Added information about standard providers automatic installation

## Migration Guide

### Upgrading from Airflow 2.x to 3.x

This script now supports seamless upgrades from Airflow 2.x to 3.x. Key considerations:

#### Prerequisites
1. **Python Version**: Ensure Python 3.9-3.12 is installed (check with `python3 --version`)
2. **Current Version**: It's recommended to upgrade to the latest 2.x version before migrating to 3.x
3. **Backup**: Always backup your instance before upgrading (the script handles this automatically)

#### What's New in Airflow 3.x
- **Task Execution API**: Workers now communicate via API server instead of direct database access
- **New Services**:
  - `airflow-api-server` (required): Handles task execution API
  - `airflow-dag-processor` (optional but recommended): Improves DAG processing performance
- **Standard Providers**: Core operators moved to `apache-airflow-providers-standard` package
- **Database Commands**: Only `airflow db migrate` is used (no more `db init` or `db upgrade`)

#### Update Process
1. Update your `airflow_setup.ini` file:
   - Set `AIRFLOW_VERSION=3.0.0` (or desired 3.x version)
   - Update `AIRFLOW_VENV`, `AIRFLOW_HOME`, and `ENV_FILE` paths
   - Set `UPGRADE=True` if upgrading existing installation
   - Update `OLD_*` variables to point to your current 2.x installation

2. Run the setup script:
   ```bash
   ./airflow_setup.sh airflow_setup.ini
   ```

3. The script will automatically:
   - Validate Python version compatibility
   - Install Airflow 3.x with standard providers
   - Migrate the database using appropriate commands
   - Create new systemd services (api-server, dag-processor)
   - Start all required services

#### Post-Migration
1. **Verify Services**: Check all services are running
   ```bash
   sudo systemctl status airflow-webserver
   sudo systemctl status airflow-scheduler
   sudo systemctl status airflow-api-server
   sudo systemctl status airflow-dag-processor  # if enabled
   ```

2. **Check DAGs**: Review your DAGs for Airflow 3.x compatibility
   - Use `ruff` with AIR301/AIR302 rules to identify breaking changes
   - Update imports from `airflow.models.*` to `airflow.sdk.*` where needed

3. **Monitor Logs**: Check service logs for any issues
   ```bash
   sudo journalctl -u airflow-webserver -f
   sudo journalctl -u airflow-api-server -f
   ```

## Backward Compatibility

The script maintains full backward compatibility with Airflow 2.x installations:
- Automatically detects Airflow version from configuration
- Uses appropriate commands and creates relevant services for each version
- No changes needed for existing Airflow 2.x deployments

## [Previous Releases]

For information about earlier releases, please refer to the [GitHub releases page](https://github.com/tahaislam/airflow-mate/releases).
