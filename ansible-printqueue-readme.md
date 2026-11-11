# Ansible Print Queue Playbook README

Document Title: Ansible printqueue.yml README  
Version: 1.0  
Author: Joshua Heath  
Environment: Production Manufacturing Floor & Office/Lab  
Related Systems: Raspberry Pi OS, Ansible, CUPS  
Classification: Sanitized Portfolio

## 1. Purpose

Overview: This playbook automates the cleanup of stale CUPS printer queues and the dynamic assignment of a new RAW CUPS outqueue. It acquires the unique USB device ID of the attached printer and binds it to a standardized queue name, ensuring reliable, serial-aware printing without manual lpadmin intervention.

Business Impact: Reduces physical printer replacement and configuration time from 10 to 15 minutes to under 5 minutes. This streamlined, deterministic process minimizes the risk of human error and prevents interruptions to continuous production line workflows and warehouse shipping operations.

## 2. Prerequisites

Before executing this playbook, ensure the following conditions are met:

### Ansible Environment
- Ansible Version: Current stable release (playbook uses standard ansible.builtin modules).
- Inventory: A valid inventory.ini file containing the target device(s), alongside a maintained backup master list.
- Collections: No external Ansible Galaxy collections are required; it relies entirely on core modules.

### Target System Requirements
- OS: Raspberry Pi OS (Bookworm 64-bit baseline).
- Pre-installed Packages: cups must be installed and active.
- Active Services: cups, sshd, vnc, and xinetd (for legacy LPD support) must be enabled and running.

### Access & Network
- Credentials: Standard defined user credentials, plus sudo privileges (required for --ask-become-pass).
- Network Connectivity: The Ansible control node must have network reachability to the target device on TCP ports 22 (SSH), 5900 (VNC), and 515 (LPD).

### Hardware
- Device: Technically agnostic to the printer brand, provided the device supports RAW printing over USB. (Currently deployed with Honeywell/Intermec devices).

## 3. Variables

The following variables are defined at the top of the playbook and can be overridden via the command line or inventory if necessary.

### printer_name
- Purpose: Defines the standardized CUPS outqueue name.
- Default: [Sanitized_Queue_Name]
- When to change: Override this via -e if the specific station requires a unique queue name that deviates from the standard.
- Constraints: Must match the outqueue name expected by the upstream ERP system.

### duplicate_queues
- Purpose: A list of legacy or stale queue names to actively purge before creating the new queue, preventing routing conflicts.
- Default: ['PM45-203', 'PM43-203', '[Sanitized_Queue_Name]']
- When to change: Update this list if new legacy queue naming conventions are discovered during hardware audits.

### report_file
- Purpose: The local path on the Ansible control node where the post-execution audit report is appended.
- Default: ./printer-report.txt
- When to change: Change this path if running the playbook from a different working directory or integrating with a centralized logging directory.

## 4. Usage

This playbook is designed to be executed from the Ansible control node where the inventory.ini file resides.

### Standard Execution (Single Device)
To target a specific device safely, always use the --limit flag alongside --ask-become-pass to elevate privileges for CUPS configurations.

```bash
ansible-playbook printqueue.yml --ask-become-pass --limit 'IP_Address'
```

### Standard Execution (Multiple Specific Devices)
If replacing printers at multiple stations simultaneously, you may target a comma-separated list or an Ansible group, but never run this against all hosts indiscriminately.

```bash
ansible-playbook printqueue.yml --ask-become-pass --limit 'IP_Address_1,IP_Address_2,IP_Address_3'
```

### Overriding Variables
If a specific station requires a non-standard queue name, pass it at runtime:

```bash
ansible-playbook printqueue.yml --ask-become-pass --limit 'IP_Address' -e "printer_name=CUSTOM_QUEUE_NAME"
```

## 5. Expected Output

A successful playbook execution displays:
- A PLAY RECAP with 0 failed tasks.
- Skipped tasks may appear if no duplicate queues were detected (this is expected and correct behavior).
- The printer report is appended to ./printer-report.txt on the Ansible control node.

### Sample Report Output
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

### Common Failure Scenarios

#### Scenario 1: Playbook fails at "Determine first USB printer URI" or "Fail when no USB printer is detected"
Investigation:
- SSH into the target device and run: `lsusb -v`
- Run: `lpinfo -v`
- Verify the printer is physically connected and powered on.

Resolution:
- Check USB cable seating at both the printer and Raspberry Pi ends.
- Try a different USB port on the Raspberry Pi.
- Replace the USB cable if damaged.
- Verify the printer model is supported (e.g., Honeywell/Intermec PM43, PM45, or PX4i).

#### Scenario 2: Playbook identifies more than one printer attached
Investigation:
- Run: `lpinfo -v | grep 'usb://'`
- Check for multiple USB printers connected to the device.

Resolution:
- Disconnect any non-production or test printers.
- The playbook uses the first detected USB printer URI; ensure only the target production printer is connected during configuration.
- If multiple printers are genuinely required at this station, this playbook must be modified to handle multi-queue configurations.
