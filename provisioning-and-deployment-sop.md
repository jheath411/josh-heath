# Linux Edge Device Provisioning and Replacement Procedure SOP

Document Title: Linux Edge Device provisioning and replacement Procedure SOP  
Version: 1.0  
Author: Joshua Heath  
Environment: Production Manufacturing Floor  
Related Systems: Raspberry Pi Fleet (3B, 3B+, 4B), Honeywell/Intermec Printers  
Classification: Sanitized Portfolio

## Phase 1: Golden Image Creation & Hardening (Office/Lab Environment)

### Step 1: Flash Base Operating System
- Action: Flash the designated Raspberry Pi OS baseline (Raspberry Pi Desktop 64-bit) to the target sd-card using Raspberry Pi Imager or dd.
- Expected Result: Device boots successfully to the initial setup/login screen with default credentials.
- Fallback: If the device fails to boot, verify the sd-card integrity, reformat the card, and re-flash the image.

### Step 2: Attack Surface Reduction & Bloat Removal
- Action: Connect the device to the lab network. Execute a targeted package purge to remove unnecessary GUI applications, services, and development tools to reclaim storage and minimize the attack surface.
  - Command Example: `sudo apt remove --purge firefox-esr vlc bluez geany xscreensaver thonny`
- Expected Result: Command completes successfully. Storage space is reclaimed and accessible from the OS (`df -h`), unnecessary services are no longer present (`systemctl list-units`).
- Fallback: If a required dependency is accidentally removed, restore by re-flashing the base image and refine the purge list.

### Step 3: Apply Baseline Configuration via Ansible
- Action: Acquire the device IP address from the onscreen kiosk interface, execute the baseline Ansible provisioning from the control node via SSH.
  - Command Example: `ansible-playbook registration.yml --ask-become-pass --limit 'IP Address'`
- Expected Result: Playbook executes with 0 failed tasks. Core services (for example, cups, sshd, vnc) are enabled and running.
- Fallback: If the playbook fails, check SSH connectivity, verify the device is in the correct Ansible inventory group, and review the Ansible output for specific module errors.

### Step 4: Functional Validation
- Action:
  1. Boot the device and verify the operation web page loads and is in kiosk mode.
  2. Trigger a print directly from the ERP system.
  3. Trigger a test print using the local on-demand print button on the device.
- Expected Result: Web page renders and responds to touch interactions. Both the ERP queue and local button successfully generate a correctly formatted label on the connected printer.
- Fallback: If printing fails, run `lpstat -p` or `lpstat -v` to verify the CUPS queue is enabled and idle. Check the ERP queue for residual labels.

---

## Phase 2: Production Device Replacement & Validation (Manufacturing Floor)

### Step 1: Physical Device Swap
- Action: Power off the device and retrieve either a replacement device or sd-card from the rapid replacement stock. Connect all cables and ensure touch interface reacts and device acquires IP address.
- Expected Result: Device powers on and boots to the kiosk interface within 60 seconds.
- Fallback: If the device fails to boot, re-seat cable connections, verify the sd-card is fully inserted, swap to a different pre-built device from stock.

### Step 2: Network Configuration & Static IP Assignment
- Action: From your IT workstation, SSH into the device using the baseline credentials. Assign the designated static IP address from the [Facility MES IP Addresses] that is currently not in use and has a clear ERP outqueue.
  - Command Example: `sudo nmtui`
- Expected Result: Kiosk interface displays the correct static IP address and can be pinged.
- Fallback: If network connectivity fails, power off device and verify address is not in use.

### Step 3: Automated Printer Queue Configuration via Ansible
- Action: From the ansible control node, execute the printer queue playbook targeting the newly assigned device IP.
  - Example Command: `ansible-playbook printqueue.yml --ask-become-pass --limit 'IP Address'`
- Expected Result: Playbook completes with 0 failed tasks. The CUPS queue is created and bound to the detected USB printer URI.
- Fallback: If the playbook fails, SSH into the device and run `lsusb -v` to verify the connected printer URI. If the printer is not visible check cable connections or swap cables.

### Step 4: End-to-End Validation
- Action:
  1. Trigger a test print from the ERP.
  2. Verify the label prints correctly on the attached printer.
- Expected Result: Labels print with the correct formatting, barcodes, and data. No errors or additional labels in ERP queue.
- Fallback: If the label does not print, check the ERP outqueue for residual labels. On device run `lpstat -v` to check for duplicate outqueues and verify printer URI match. If the queue is stuck on device but clear in the ERP run `sudo systemctl restart cups`.

### Step 5: Device Labeling & Documentation Update
- Action: Apply a physical label with the last two octets of the IP address and last four digits of the outqueue name, to the Raspberry Pi device. Update the [Facility MES IP Addresses] document with the date the device was assigned to the floor.
- Expected Result: Device is physically identifiable and documentation is current.
- Fallback: N/A (This is an administrative step, but crucial for future troubleshooting).
