# Web Solution With WordPress

This repository documents a complete WordPress deployment on AWS using two EC2 instances and LVM-managed storage.

## Overview

- Web server: RHEL 10, Apache, PHP, WordPress
- Database server: RHEL 10, MariaDB
- Storage: 3 EBS volumes combined with LVM
- Documentation: step-by-step guide with screenshots

## Repository Contents

- `development-steps.md`: Full implementation guide from EC2 provisioning to final WordPress validation
- `STEP 0.x ... .png`: Screenshots that support each stage of the deployment

## Architecture Summary

| Component | Details |
|---|---|
| Web Server | Apache httpd 2.4.63 on RHEL 10 |
| PHP | 8.3.29 |
| Database | MariaDB 10.11 on a separate EC2 instance |
| Storage | `webdata-vg` volume group with `apps-lv` and `logs-lv` |
| Application | WordPress |

## Deployment Flow

1. Create the web and database EC2 instances.
2. Attach and configure three EBS volumes on the web server.
3. Build LVM storage and persist the mounts.
4. Install and configure MariaDB on the database server.
5. Install Apache, PHP, and WordPress on the web server.
6. Complete the WordPress setup wizard and verify the site.

## How To Use This Repository

1. Open `development-steps.md`.
2. Follow each step in order.
3. Use the screenshots as a visual reference for validation.

## Notes

- This project is intended as deployment documentation rather than an application source tree.
- Infrastructure values in the guide reflect the environment used during the implementation.