# Ansible Print Queue Playbook README

**Document Title:** Ansible printqueue.yml README  
**Version:** 1.0  
**Author:** Joshua Heath  
**Environment:** Production Manufacturing Floor & Office/Lab  
**Related Systems:** Raspberry Pi OS, Ansible, CUPS  
**Classification:** Sanitized Portfolio

## Summary

This playbook automates the cleanup of stale CUPS printer queues and the assignment of a new RAW CUPS output queue. It identifies the unique USB device ID of the connected printer and binds it to a standardized queue name, providing consistent, serial-aware printing without manual `lpadmin` intervention.

The result is faster printer replacement and lower operational risk. Printer changeout and configuration time drops from roughly 10–15 minutes to under 5 minutes, reducing the chance of human error and limiting disruption to production and shipping workflows.

## 1. Purpose

### Overview
This playbook is designed to standardize printer configuration on Raspberry Pi-based edge devices in production environments. It removes legacy or stale CUPS queues, detects the active USB printer, and creates a single stable queue bound to the correct device URI.

### Business Impact
- Lowers printer replacement time from 10–15 minutes to under 5 minutes
- Reduces the risk of configuration drift and operator error
- Prevents queue conflicts that can interrupt production and shipping workflows
- Supports deterministic printer behavior across similar field devices

## 2. Prerequisites

### Ansible environment
- Ansible version: current stable release using standard `ansible.builtin` modules
- Inventory: a valid `inventory.ini` file with target device entries and a maintained backup list
- Collections: no external Ansible Galaxy collections required

### Target system requirements
- OS: Raspberry Pi OS (Bookworm 64-bit baseline)
- Required package: `cups` installed and active
- Required services: `cups`, `sshd`, `vnc`, and `xinetd` (legacy LPD support) enabled and running

### Access and network
- Credentials: standard user credentials with `sudo` privileges and `--ask-become-pass`
- Connectivity: reachability from the Ansible control node to the target device on TCP ports 22 (SSH), 5900 (VNC), and 515 (LPD)

### Hardware
- Device type: any printer that supports RAW printing over USB
- Current deployment: Honeywell/Intermec devices such as PM43, PM45, and PX4i families

## 3. Variables

The following variables are defined at the top of the playbook and may be overridden at runtime.

### `printer_name`
- **Purpose:** defines the standardized CUPS output queue name
- **Default:** `[Sanitized_Queue_Name]`
- **When to change:** override with `-e` when a station requires a queue name different from the standard
- **Constraint:** must match the queue name expected by the upstream ERP system

### `duplicate_queues`
- **Purpose:** list of stale or legacy queue names to remove before creating the active queue
- **Default:** `['PM45-203', 'PM43-203', '[Sanitized_Queue_Name]']`
- **When to change:** update when additional legacy naming schemes are discovered during hardware audits

### `report_file`
- **Purpose:** local path on the control node where the execution report is appended
- **Default:** `./printer-report.txt`
- **When to change:** update when running from a different working directory or centralized log path

## 4. Usage

This playbook is intended to run from the Ansible control node where `inventory.ini` is maintained.

### Single-device execution
Use `--limit` with `--ask-become-pass` to target a specific device safely:

```bash
ansible-playbook printqueue.yml --ask-become-pass --limit 'IP_Address'
```

### Multiple-device execution
For several stations, target a comma-separated list or Ansible group:

```bash
ansible-playbook printqueue.yml --ask-become-pass --limit 'IP_Address_1,IP_Address_2,IP_Address_3'
```

### Overriding variables
If a specific station requires a custom queue name:

```bash
ansible-playbook printqueue.yml --ask-become-pass --limit 'IP_Address' -e "printer_name=CUSTOM_QUEUE_NAME"
```

## 5. Expected Output

A successful execution should produce:
- `PLAY RECAP` with 0 failed tasks
- Optional skipped tasks when no duplicate queues are found
- An appended audit report in `./printer-report.txt`

### Sample report output

```text
============================================================
Inventory Name : [TARGET_IP_ADDRESS]
Printer Queue: Sanitized_Queue_Name
USB Device URI: usb://Intermec/PM43-203-IPL?serial=25421945159
Configured Printers: device for Sanitized_Queue_Name: usb://Intermec/PM43-203-IPL?serial=25421945159
Queue Options: [Options list or "Unavailable"]
LPD Listener: Enabled
```

## 6. Troubleshooting

### Scenario 1: No USB printer is detected
**Investigation**
- Log in to the target device and run `lsusb -v`
- Run `lpinfo -v`
- Confirm the printer is powered on and physically connected

**Resolution**
- Check USB seating on both the printer and Raspberry Pi ends
- Try a different USB port on the Pi
- Replace damaged USB cables
- Verify printer compatibility with the supported device family

### Scenario 2: More than one USB printer is connected
**Investigation**
- Run `lpinfo -v | grep 'usb://'`
- Review connected devices for multiple USB printer mappings

**Resolution**
- Remove non-production or test devices
- Ensure only the target production printer remains attached during configuration
- If multi-printer support is required, modify the playbook to handle more than one queue

## 7. Operational Notes

This playbook is intended to be deterministic and repeatable. It should be used only after validating the target device, confirming the correct printer is attached, and ensuring the queue name matches upstream operational requirements.
