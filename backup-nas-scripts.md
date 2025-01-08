# Issue and Resolution Summary: Jira Ticket [#CSS-2509]

## Issue Overview
On 18 November, following a power failure, two NAS units (TS-451+ and TS-453Be) developed critical issues:

1. Both units were shutting down automatically 60 seconds after startup.
2. The backup NAS had also lost all its data.

These problems were causing significant disruption to operations as both NAS units were critical for development and data storage.

---

## Troubleshooting Process

### 1. Initial Steps
I began by trying to isolate the problem. Booting the NAS units with and without drives helped determine whether the issue was hardware-related or due to RAID configurations:

- With the drives removed, the NAS stayed online but reverted to default settings.
- With the drives inserted, the units powered down after 60 seconds.

Using QNAP's troubleshooting guidelines, I accessed the NAS systems via SSH to collect logs and run commands, as the web UI wasn’t accessible due to the shutdown loop.

### 2. Identifying the Cause
After reviewing the logs and discussing possible causes with colleagues, the issue appeared to be linked to a custom script (`upscheck.sh`) developed by a previous administrator:

- This script was designed to monitor the UPS (Uninterruptible Power Supply) and shut down the NAS if anomalies were detected.
- It relied on a flag file (`/share/sysadmin/tmp/nas_ups_flagfl`) stored on the RAID array to determine whether a shutdown was required.

The power outage left the flag file in place, so the script was continuously triggering shutdowns during the boot process. Additionally, the NAS units stored critical configuration data on the RAID array instead of the internal EEPROM, due to the constraints of their embedded Linux environment. This setup meant the NAS units operated with default settings unless the RAID array was mounted.

### 3. Resolving the Problem
To address the issue:

1. I accessed the NAS via SSH and manually deleted the flag file, which allowed the system to stay online.
2. I identified a missing step in the `upscheck.sh` script. The script didn’t remove the flag file after shutting down the NAS, causing it to persist across reboots.
3. I modified the script to include a command to remove the flag file before initiating a shutdown. The updated section of the script looks like this:

   ```bash
   if [ -f $nasflag ]; then
       rm $nasflag
       echo "Flag file found and removed at $(date)" >> $logfl
       echo "Shutting down now" >> $logfl
       /sbin/poweroff >> $logfl
   else
       echo "No flag file found at $(date)" >> $logfl
   fi
   ```

4. The script was updated on both NAS units to prevent the issue from recurring.

### 4. Testing and Recovery
After implementing the fix, I performed multiple tests, including:

- Booting the NAS units with and without the drives.
- Simulating UPS anomalies to ensure the shutdown behaviour worked correctly.

Both NAS units operated as expected after the fix. I then focused on recovering the backup NAS data, which was successfully restored from the RAID array.

### 5. Documenting the Process
I created a detailed internal wiki entry covering:

- The issue’s root cause and the steps taken to resolve it.
- Instructions for manually recovering a NAS from this state, including RAID remounting and restoring configurations.
- Key learnings about the custom scripts and how to prevent similar problems in the future.

---

## Outcome
Both NAS units were restored to full functionality, and the RAID array data was successfully recovered. The script modifications ensured the shutdown issue wouldn’t recur.

### Key Takeaways:
- **Understanding Custom Configurations:** Identifying dependencies in scripts and custom setups is critical for effective troubleshooting.
- **Thorough Testing:** Testing all possible scenarios ensures the robustness of the solution.
- **Documentation:** Detailed documentation improves knowledge sharing and prepares the team for future incidents.

By working methodically and collaborating with colleagues, I was able to identify and resolve a complex issue, while also improving the resilience of our systems for the future.
