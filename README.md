# Syteline Evening Utilities - SQL Server Jobs Conversion

A project for converting SQL Server Jobs in Syteline9 to run as background tasks using the `SubmitBgTaskSp` stored procedure.

## Project Structure

- **scripts/** - T-SQL scripts for converting jobs and managing background tasks
  - `00_template_bg_task_conversion.sql` - Template for converting a job to a background task
  - Individual utility conversion scripts
  
- **jobs/** - SQL Server Job definitions and configurations
  - Current job definitions
  - Conversion mapping documents
  
- **docs/** - Project documentation
  - Migration patterns
  - SubmitBgTaskSp parameter documentation
  - Conversion guide

## Overview

### Current State
- SQL Server Jobs running directly on the Syteline9 server
- Jobs execute stored procedures directly

### Target State
- Background tasks submitted via `SubmitBgTaskSp`
- Centralized task execution and monitoring through Syteline
- Improved logging and error handling

## Key Concepts

**SubmitBgTaskSp Parameters:**
- `@BgTaskName` - Name of the background task
- `@ERP_Number` - Syteline ERP identifier
- `@StoredProcedureName` - The utility stored procedure to execute
- `@Parameters` - Task parameters and configuration
- `@Priority` - Task execution priority

## Getting Started

1. Review the template conversion script in `scripts/00_template_bg_task_conversion.sql`
2. Identify SQL Server Jobs to convert
3. Create conversion scripts in the `scripts/` directory
4. Document each conversion in `docs/`
5. Test conversions in a development environment

## Notes

- Maintain backward compatibility where possible
- All background task submissions must include error handling
- Log all task submissions and execution results
- Test thoroughly before deploying to production
