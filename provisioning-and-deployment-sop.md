# Linux Edge Device Provisioning and Replacement Procedure SOP

**Document Title:** Linux Edge Device Provisioning and Replacement Procedure SOP  
**Version:** 1.0  
**Author:** Joshua Heath  
**Environment:** Production Manufacturing Floor  
**Related Systems:** Raspberry Pi Fleet (3B, 3B+, 4B), Honeywell/Intermec Printers  
**Classification:** Sanitized Portfolio

## Overview

This SOP defines a repeatable process for provisioning, validating, and replacing Linux-based edge devices used in manufacturing and warehouse operations. The process is designed to standardize device setup, reduce configuration drift, and ensure each replacement device is returned to service with the expected printer queue and network configuration.

The goal is operational continuity: devices should be rebuilt using a consistent baseline, validated before release, and replaced quickly when failures occur without introducing avoidable configuration errors.

## Phase 1: Golden Image Creation & Hardening (Office/Lab Environment)

### Step 1: Flash the base operating system
- **Action:** Flash the designated Raspberry Pi OS baseline (Raspberry Pi OS 64-bit desktop) onto the target SD card using Raspberry Pi Imager or `dd`.
- **Expected result:** The device boots successfully to the initial setup/login screen with default credentials.
- **Fallback:** If the device fails to boot, verify SD card integrity, reformat the card, and re-flash the image.

### Step 2: Reduce attack surface and remove unnecessary software
- **Action:** Connect the device to the lab network and remove unnecessary GUI applications, services, and development packages to reclaim storage and reduce the attack surface.
- **Example command:**

```bash
sudo apt remove --purge firefox-esr vlc bluez geany xscreensaver thonny
```

- **Expected result:** The command completes successfully, storage is reclaimed, and unnecessary services are no longer present.
- **Fallback:** If a required dependency is removed accidentally, restore by re-flashing the base image and refine the purge list.

### Step 3: Apply the baseline configuration via Ansible
- **Action:** Obtain the device IP address from the kiosk screen, then run the baseline provisioning from the Ansible control node over SSH.
- **Example command:**

```bash
ansible-playbook registration.yml --ask-become-pass --limit 'IP_Address'
```

- **Expected result:** The playbook completes with 0 failed tasks, and core services such as `cups`, `sshd`, and `vnc` are enabled and running.
- **Fallback:** If the playbook fails, verify SSH connectivity, confirm the device is in the proper inventory group, and review the Ansible output for module-specific errors.

### Step 4: Functional validation
- **Action:**
  1. Boot the device and verify the operational web page loads in kiosk mode.
  2. Trigger a print from the ERP system.
  3. Trigger a test print using the local on-demand print button.
- **Expected result:** The web page renders correctly and both ERP and local print actions generate a properly formatted label.
- **Fallback:** If printing fails, run `lpstat -p` or `lpstat -v` to confirm the CUPS queue is active and idle, then check the ERP queue for residual labels.

---

## Phase 2: Production Device Replacement & Validation (Manufacturing Floor)

### Step 1: Perform the physical device swap
- **Action:** Power off the failing device, retrieve an approved replacement device or SD card from stock, and reconnect all required cabling.
- **Expected result:** The replacement device powers on and boots to the kiosk interface within 60 seconds.
- **Fallback:** If the device fails to boot, re-seat the cable connections, confirm the SD card is fully inserted, and swap in a different pre-built device from stock.

### Step 2: Assign the network configuration and static IP
- **Action:** From the IT workstation, SSH into the device using the baseline credentials and assign the correct static IP address from the facility MES address list.
- **Example command:**

```bash
sudo nmtui
```

- **Expected result:** The kiosk display shows the correct static IP address and the device responds to ping.
- **Fallback:** If connectivity fails, power off the device and verify the address is not already in use.

### Step 3: Configure the printer queue using Ansible
- **Action:** From the Ansible control node, run the printer queue playbook against the newly assigned device IP address.
- **Example command:**

```bash
ansible-playbook printqueue.yml --ask-become-pass --limit 'IP_Address'
```

- **Expected result:** The playbook completes with 0 failed tasks and the CUPS queue is created and bound to the detected USB printer URI.
- **Fallback:** If the playbook fails, SSH into the device and run `lsusb -v` to verify the printer URI. If the printer is not visible, check cable seating or replace the USB cable.

### Step 4: Validate end-to-end operation
- **Action:**
  1. Trigger a test print from the ERP system.
  2. Verify the label prints correctly on the attached printer.
- **Expected result:** Labels print with correct formatting, barcodes, and data, with no residual job errors in the ERP queue.
- **Fallback:** If the label does not print, check the ERP queue for residual jobs. On the device, run `lpstat -v` to verify there are no duplicate outqueues and that the printer URI matches the expected device. If the queue is stuck but clear in ERP, restart CUPS with `sudo systemctl restart cups`.

### Step 5: Apply device labeling and update documentation
- **Action:** Apply a physical label containing the last two octets of the IP address and the last four digits of the outqueue name. Update the facility MES IP address document with the date the device was assigned to the floor.
- **Expected result:** The device is physically identifiable and the documentation remains current.
- **Fallback:** Not applicable; this is an administrative step but critical for future troubleshooting and asset tracking.

## Acceptance Criteria

A device is considered ready for production use when all of the following are true:
- The OS baseline is installed and hardened.
- Required services are running and accessible.
- The correct static IP is assigned and reachable.
- The printer queue is created and bound to the correct USB device URI.
- ERP and local print tests pass successfully.
- Device labeling and facility documentation are updated.

## Operational Notes

This SOP is intended to be practical and repeatable. It assumes the device fleet is standardized to a supported Raspberry Pi OS baseline and uses consistent naming conventions for both network and print queue configuration. Consistency is critical: it reduces troubleshooting time and increases confidence that any replacement device can be restored to service quickly.
