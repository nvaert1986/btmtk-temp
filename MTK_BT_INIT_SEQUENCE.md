# MediaTek MT7925/MT7927 Bluetooth Driver Initialization Sequence

## Driver Information
- **Driver**: mtkbtfilterx.sys
- **Chip Family**: MediaTek MT7925/MT7927 (Neptune Platform)
- **Architecture**: x86-64 Windows kernel driver
- **Build Path**: MT7927_2226_WIN10_AB

## Overview
This document details the Bluetooth initialization command sequences extracted from the Windows driver to help debug Linux driver startup issues.

---

## Initialization Sequence

### Phase 1: USB Device Enumeration

#### 1.1 USB Interface Configuration
```
Function: MTKUSB_SelectInterface, GetUSBInterface, SelectUSBInterface
Location: mtkbtfilterx.sys

Steps:
1. Device enumerates on USB bus
2. Host reads device descriptors
3. Set USB configuration (usually config 1)
4. Select interface 0, alternate setting 0
5. Configure endpoints:
   - Bulk IN endpoint (for ACL data in)
   - Bulk OUT endpoint (for ACL data out)
   - Interrupt IN endpoint (for events)
   - Isoch endpoints (for SCO if supported)
```

**Critical**: The driver uses `URB_FUNCTION_SELECT_CONFIGURATION` and `URB_FUNCTION_SELECT_INTERFACE` URBs.

#### 1.2 USB Endpoint Configuration
- **Bulk IN**: Endpoint address typically 0x81 or 0x82
- **Bulk OUT**: Endpoint address typically 0x01 or 0x02
- **Interrupt IN**: Endpoint address typically 0x83 (for HCI events)

---

### Phase 2: Initial Driver Setup

#### 2.1 MTKFilterInit Function
```
Function: MTKFilterInit (offset: 0x78b40)
Entry Point: DriverEntry → MTKFilterInit

Key operations:
- Initialize kernel synchronization objects (mutexes, events, spinlocks)
- Allocate device extension structures
- Set up I/O dispatch routines
- Register PnP and Power management handlers
```

#### 2.2 Start Thread Work Item
```
Function: MTKFilterStartThreadWorkItem (offset: 0x78750)

Creates background worker threads for:
- USB I/O processing
- HCI event handling
- Firmware download management
```

---

### Phase 3: WMT (Wireless Module Test) Commands

MediaTek uses proprietary WMT commands for chip initialization. These are sent as HCI vendor-specific commands.

#### 3.1 WMT Command Format
```
Byte Layout:
[0]    0x01        - HCI command packet type
[1]    0x6F        - HCI vendor-specific command for MTK
[2]    Length      - Payload length (not including header)
[3]    WMT Opcode  - WMT command opcode
[4..N] Parameters  - Command-specific parameters
```

**Alternative Format** (seen in driver):
```
[0]    0x6F        - May also appear as first byte in some contexts
[1]    0xFC        - Sub-opcode
[2..N] Parameters
```

#### 3.2 WMT Command Opcodes

| Opcode | Command | Description | Function in Driver |
|--------|---------|-------------|-------------------|
| 0x00 | TEST | Test command | - |
| 0x01 | RESET | Reset chip | `btmtk_usb_send_wmt_reset_cmd` |
| 0x02 | EXIT | Exit WMT mode | - |
| 0x06 | FUNC_CTRL | BT/WiFi function on/off | `btmtk_usb_send_wmt_set_bt_fun_cmd` |
| 0x07 | PATCH_DOWNLOAD | Firmware download | `btmtk_send_wmt_dl_cmd` |
| 0x08 | GET_FW_VER | Get firmware version | - |
| 0x0D | EFUSE_READ | Read eFuse data | `btmtk_usb_send_wmt_read_4_efuse_cmd` |
| 0x0E | EFUSE_WRITE | Write eFuse data | `btmtk_usb_send_wmt_write_16_efuse_cmd` |
| 0x0F | FEATURE_SET | Configure features | `btmtk_usb_send_wmt_set_efem_cmd` |
| 0x10 | REGISTER_WRITE | Write chip register | `btmtk_usb_send_wmt_write_cr_cmd` |
| 0x11 | REGISTER_READ | Read chip register | - |

---

### Phase 4: Detailed Initialization Steps

#### Step 1: Initial Reset
```
Function: btmtk_usb_send_wmt_reset_cmd (offset: 0x79ce0)

Command Format:
01 6F 04 01 02 01 00 00
    └─┬─┘ │  └──┬─────┘
      │   │     └─ Parameters (reset type)
      │   └─ Opcode: 0x01 (RESET)
      └─ Length: 0x04

Purpose:
- Reset Bluetooth chip to known state
- Clear any previous state
- Prepare for firmware loading

Expected Response:
Event: 0x04 0xE4 (HCI Command Complete for vendor command)
```

**Critical for Linux**: You MUST wait for the completion event before proceeding. Typical timeout: 1000ms.

---

#### Step 2: Check Need for ROM Patch
```
Function: btmtk_usb_check_need_load_rom_patch_7663 (offset: 0x7a720)
          btmtk_load_rom_patch (offset: 0x7ac40)

Operations:
1. Read chip ID and firmware version
2. Check if ROM patch is needed
3. Determine which firmware binary to load

Chip Detection:
- MT7925: Check for chip ID via HCI
- MT7927: Check for chip ID via HCI
```

---

#### Step 3: Load ROM Patch (Firmware)

This is the **MOST CRITICAL** phase where many Linux drivers fail.

```
Function: btmtk_load_rom_patch (offset: 0x7ac40)
          btmtk_load_image_79xx (offset: 0x7ab80)
          btmtk_send_wmt_dl_cmd (offset: 0x7a8e0)

Sequence:
1. Open firmware file (from Windows: registry path, Linux: /lib/firmware/)
2. Parse firmware header
3. Download firmware in segments

Segment Download Loop:
for each segment in firmware:
    a) Send WMT PATCH_DOWNLOAD command (opcode 0x07)
       Format: 01 6F <len> 07 <segment_data>

    b) Wait for completion event (0x04 0xE4)
       CRITICAL: Must wait for each segment!

    c) Check response status
       If status != 0: ABORT and report error

4. Send DMA Complete command
5. Verify patch result
```

##### 3.1 WMT Patch Download Command Format
```
Byte Layout:
[0]     0x01           - HCI command type
[1]     0x6F           - MTK vendor command
[2-3]   Length (LE)    - Payload length (2 bytes, little-endian)
[4]     0x07           - Opcode: PATCH_DOWNLOAD
[5]     Phase          - Download phase:
                         0x01 = Start
                         0x02 = Continuing
                         0x03 = End
[6-N]   Data           - Firmware data segment
```

**Example**:
```
01 6F 00 84 07 02 [... 132 bytes of firmware data ...]
   │  └──┬─┘ │  │
   │     │   │  └─ Phase: 0x02 (continuing)
   │     │   └─ Opcode: 0x07
   │     └─ Length: 0x8400 = 132 bytes
   └─ Vendor cmd
```

##### 3.2 DMA Complete Command
```
Function: btmtk_usb_send_wmt_dma_complete_cmd (offset: 0x79cb0)

Command:
01 6F 05 07 03 00 00 00 00
         │  │  └───┬──────┘
         │  │      └─ Parameters (phase: end + padding)
         │  └─ Opcode: 0x07 (PATCH_DOWNLOAD)
         └─ Length: 0x05

Purpose: Signal firmware download complete, trigger DMA verification
```

##### 3.3 Get ROM Patch Result
```
Function: btmtk_usb_get_rom_patch_result (offset: 0x79c90)

After DMA complete, read result via HCI event
Expected event: Command Complete with result status
Status 0x00 = Success
Status != 0x00 = Failure (do NOT continue)
```

**Common Linux Driver Bug**: Not waiting for ROM patch result before enabling BT function.

---

#### Step 4: Register Configuration (Optional but Important)

##### 4.1 Write Control Register
```
Function: btmtk_usb_send_wmt_write_cr_cmd (offset: 0x79d00)

Command Format:
01 6F 08 10 <addr_4bytes> <value_4bytes>
         │  └────┬───────┘ └─────┬──────┘
         │       │                └─ Register value (LE)
         │       └─ Register address (LE, 32-bit)
         └─ Opcode: 0x10 (REGISTER_WRITE)

Common registers written during init:
- Clock configuration registers
- Power management registers
- GPIO configuration
- PHY settings
```

##### 4.2 eFuse Operations
```
Function: btmtk_usb_send_wmt_read_4_efuse_cmd (offset: 0x79d60)
          btmtk_usb_send_wmt_write_16_efuse_cmd (offset: 0x79d90)

Read eFuse (4 bytes):
01 6F 06 0D <addr_2bytes> <length_2bytes>

Write eFuse (16 bytes):
01 6F 12 0E <addr_2bytes> <16_bytes_data>

Purpose:
- Read chip calibration data
- Read Bluetooth address
- Configure chip-specific parameters
```

---

#### Step 5: Enable Bluetooth Function

**THIS IS WHERE MANY LINUX DRIVERS HANG!**

```
Function: btmtk_usb_send_wmt_set_bt_fun_cmd (offset: 0x7a6f0)

Command Format:
01 6F 04 06 01 00 00 00
         │  │  └───┬────┘
         │  │      └─ Params: 0x01 = ON, 0x00 = OFF
         │  └─ Opcode: 0x06 (FUNC_CTRL)
         └─ Length: 0x04

Steps:
1. Send FUNC_CTRL command with parameter 0x01 (enable)
2. **WAIT** for Command Complete event (timeout: 2000ms)
3. Check event status = 0x00 (success)
4. **WAIT** additional 100-500ms for chip to stabilize

Event Response:
04 E4 05 02 06 01 00 00
│  │  │  │  │  │  └──┬─┘
│  │  │  │  │  │     └─ Status
│  │  │  │  │  └─ Function (0x01 = BT)
│  │  │  │  └─ Opcode (0x06)
│  │  │  └─ Event type (0x02 = WMT)
│  │  └─ Length
│  └─ Event code (0xE4 = vendor)
└─ Event packet type
```

**Critical Issues for Linux Driver**:
1. **Timeout**: If no response within 2000ms, something is wrong (firmware not loaded properly)
2. **Premature Continue**: Do NOT send further commands until this completes
3. **Power State**: Chip must be in correct power state (check USB suspend/resume)

---

#### Step 6: Additional Configuration

##### 6.1 Set Power Mode
```
Function: btmtk_usb_send_hci_set_power_cmd (offset: 0x7a0c0)

Commands for different chip generations:
- ConnAC2: btmtk_usb_send_hci_dynamically_set_connac2_power_cmd
- ConnAC3: btmtk_usb_send_hci_dynamically_set_connac3_power_cmd

HCI Command Format (vendor-specific):
01 FC 08 <power_params>
   └─┬─┘
     └─ Vendor opcode (varies by chip)
```

##### 6.2 Set Sleep Mode
```
Function: btmtk_usb_send_hci_set_sleep_cmd (offset: 0x7a040)

Configure low-power sleep mode
```

##### 6.3 WiFi Coexistence (for combo chips)
```
Function: btmtk_usb_send_wmt_wifi_on_7663 (offset: 0x7a750)

For MT7925/MT7927 which have WiFi+BT combo:
- Configure BT/WiFi coexistence
- Set priority rules
- Configure shared antenna
```

---

### Phase 5: HCI Layer Initialization

After WMT sequence completes:

#### 5.1 Standard HCI Reset
```
Send: 01 03 0C 00
      │  └─┬─┘ └─ Length: 0
      │    └─ Opcode: 0x0C03 (HCI_Reset)
      └─ Command packet type

Wait for: 04 0E 04 01 03 0C 00
          │  │  │  │  └─┬─┘ └─ Status: 0x00
          │  │  │  │    └─ Opcode
          │  │  │  └─ Num HCI command packets
          │  │  └─ Length
          │  └─ Event: Command Complete
          └─ Event packet type
```

#### 5.2 Read Local Version
```
Send: 01 01 10 00
      Opcode: 0x1001 (HCI_Read_Local_Version_Information)

Response contains:
- HCI version
- LMP version
- Manufacturer ID (should be MediaTek)
- Sub-version
```

#### 5.3 Read BD_ADDR
```
Send: 01 09 10 00
      Opcode: 0x1009 (HCI_Read_BD_ADDR)

Response: Bluetooth device address (6 bytes)
```

---

## USB Vendor Requests

MediaTek driver also uses USB control transfers for register access:

### USB Control Request Format
```
bmRequestType | bRequest | wValue     | wIndex | wLength | Data
--------------|----------|------------|--------|---------|------
0x40 (OUT)    | 0x01     | <varies>   | 0      | len     | data
0xC0 (IN)     | 0x02     | <varies>   | 0      | len     | -
```

### Common bRequest Values:
- **0x01**: Write register
- **0x02**: Read register
- **0x63**: Write UHWR (Ultra High-speed Wireless Register)
- **0x64**: Read UHWR
- **0x30-0x37**: Vendor-specific operations

### Key Functions:
- `MTKUSB_VendorRequest` (offset: 0x79bd0)
- `MTKUSB_WriteRegRequest` (offset: 0x7a8e0)
- `USBHwHal_WriteUHWRegister` (offset: 0x7a8f0)
- `USBHwHal_WriteMacRegister` (offset: 0x7a900)
- `USBHwHal_ReadUHWRegister` (offset: 0x7a910)

### Request Sequence During Init:
```
1. Write clock configuration register
   bRequest = 0x01, wValue = clock_config_addr

2. Write power management register
   bRequest = 0x01, wValue = power_config_addr

3. Read chip status register
   bRequest = 0x02, wValue = status_addr

4. Write GPIO configuration
   bRequest = 0x01, wValue = gpio_config_addr
```

---

## Debugging Linux Driver Hang Issues

### Most Common Failure Points:

1. **Firmware Loading Failure**
   - **Symptom**: Hangs after firmware download start
   - **Cause**: Firmware file missing, corrupted, or wrong version
   - **Fix**: Ensure firmware binary matches chip ID exactly
   - **Check**: Look for `/lib/firmware/mediatek/BT_RAM_CODE_MT7927_*.bin`

2. **WMT FUNC_CTRL Timeout**
   - **Symptom**: Hangs when enabling BT function (step 5)
   - **Cause**: Firmware not properly loaded or verified
   - **Fix**: Ensure ROM patch result check passed in step 3.3
   - **Debug**: Add printk before/after each WMT command

3. **Missing Completion Event Wait**
   - **Symptom**: Random hangs or device not responding
   - **Cause**: Sending commands too fast without waiting
   - **Fix**: Add proper wait_event_timeout after each command
   - **Timeout**: Minimum 100ms, recommended 1000ms for WMT commands

4. **USB Endpoint Not Ready**
   - **Symptom**: Early hang during device enumeration
   - **Cause**: Interrupt endpoint not configured
   - **Fix**: Ensure usb_submit_urb() for interrupt IN succeeds
   - **Check**: Verify endpoint addresses match device descriptors

5. **Power Management Issues**
   - **Symptom**: Device disappears or doesn't respond after sleep
   - **Cause**: USB autosuspend enabled too early
   - **Fix**: Disable USB autosuspend during init:
     ```c
     usb_disable_autosuspend(udev);
     ```

### Recommended Debug Logging:

Add these debug points to your Linux driver:

```c
// Before each WMT command
pr_info("btmtk: Sending WMT opcode 0x%02x\n", opcode);

// After each WMT command
pr_info("btmtk: WMT opcode 0x%02x completed, status=0x%02x\n",
        opcode, status);

// Firmware loading
pr_info("btmtk: Loading firmware segment %d/%d\n",
        seg_num, total_segs);

// Function enable
pr_info("btmtk: Enabling BT function...\n");
// ... wait ...
pr_info("btmtk: BT function enabled\n");
```

### Timing Requirements:

| Operation | Min Wait | Recommended | Max Timeout |
|-----------|----------|-------------|-------------|
| WMT Reset | 50ms | 100ms | 1000ms |
| Firmware segment | 10ms | 50ms | 500ms |
| DMA complete | 100ms | 200ms | 2000ms |
| FUNC_CTRL enable | 200ms | 500ms | 3000ms |
| HCI Reset | 100ms | 200ms | 1000ms |

---

## Register Addresses (Partial List)

These are commonly accessed registers extracted from the driver:

```
Note: Exact addresses are chip-specific.
MT7925/MT7927 use memory-mapped registers accessed via USB vendor requests.

Common register ranges:
0x70000000 - 0x7CFFFFFF: Chip configuration registers
0x80000000 - 0x8FFFFFFF: Wireless subsystem registers

Specific registers (examples from disassembly):
- Clock control: 0x7C060000 range
- Power management: 0x70020000 range
- BGF (Bluetooth/GPS/FM) base: 0x18000000 (relative)
```

**Note**: To get exact register addresses for your chip, you need to:
1. Capture USB traffic from Windows driver using Wireshark + USBPcap
2. Look for vendor control requests (bmRequestType 0x40/0xC0)
3. Extract wValue fields which contain register addresses

---

## Command Sequence Summary

### Quick Reference - Startup Order:

```
1. USB enumeration
2. MTKFilterInit / setup driver structures
3. WMT RESET
4. Check chip ID / ROM patch needed
5. Load ROM patch (firmware download)
   a. Multiple WMT PATCH_DOWNLOAD commands
   b. DMA Complete
   c. Verify result
6. Write configuration registers (optional)
7. Read eFuse data (BD_ADDR, calibration)
8. WMT FUNC_CTRL enable BT function ← MOST CRITICAL
9. Set power mode
10. HCI Reset
11. HCI initialization (read version, BD_ADDR, etc.)
12. Ready for operation
```

### Pseudocode for Linux Driver:

```c
int btmtk_usb_setup(struct btmtk_usb_data *data)
{
    int ret;

    // 1. Setup USB interface
    ret = usb_set_interface(data->udev, 0, 0);
    if (ret < 0)
        return ret;

    // 2. Submit interrupt URB for events
    ret = usb_submit_urb(data->intr_urb, GFP_KERNEL);
    if (ret < 0)
        return ret;

    // 3. Send WMT Reset
    ret = btmtk_send_wmt_reset(data);
    if (ret < 0)
        return ret;
    msleep(100);

    // 4. Load firmware
    ret = btmtk_load_firmware(data);
    if (ret < 0) {
        pr_err("Firmware load failed\n");
        return ret;
    }

    // 5. Enable BT function - CRITICAL STEP
    ret = btmtk_send_wmt_func_ctrl(data, 0x01); // Enable
    if (ret < 0) {
        pr_err("Failed to enable BT function\n");
        return ret;
    }

    // 6. Wait for chip stabilization
    msleep(500);

    // 7. HCI Reset
    ret = btmtk_send_hci_reset(data);
    if (ret < 0)
        return ret;

    // 8. Read version info
    ret = btmtk_hci_read_local_version(data);
    if (ret < 0)
        return ret;

    pr_info("btmtk: Initialization complete\n");
    return 0;
}
```

---

## Additional Resources

### Function Call Graph (Key Functions):

```
DriverEntry
  └─> MTKFilterInit
       └─> MTKFilterAddDevice
            └─> MTKFilterStartThreadWorkItem
                 └─> btmtk_load_rom_patch
                      ├─> btmtk_usb_send_wmt_reset_cmd
                      ├─> btmtk_load_image_79xx
                      │    └─> btmtk_send_wmt_dl_cmd (loop)
                      │         └─> btmtk_usb_send_wmt_dma_complete_cmd
                      ├─> btmtk_usb_get_rom_patch_result
                      ├─> btmtk_usb_send_wmt_set_bt_fun_cmd ← HANG POINT
                      └─> btmtk_usb_send_hci_set_power_cmd
```

### Key Driver Offsets:

| Function | Offset | Purpose |
|----------|--------|---------|
| DriverEntry | 0x140089000 | Entry point |
| MTKFilterInit | 0x78b40 | Main init |
| btmtk_usb_send_wmt_cmd | 0x79bd0 | Send WMT command |
| btmtk_usb_send_wmt_reset_cmd | 0x79ce0 | Reset chip |
| btmtk_usb_send_wmt_dma_complete_cmd | 0x79cb0 | Firmware DMA done |
| btmtk_usb_send_wmt_set_bt_fun_cmd | 0x7a6f0 | Enable/disable BT |
| btmtk_load_rom_patch | 0x7ac40 | Load firmware |
| btmtk_send_wmt_dl_cmd | 0x7a8e0 | Download FW segment |

---

## Conclusion

The most critical aspects for Linux driver success:

1. **Firmware must load completely** - Verify each segment
2. **Wait for completion events** - Do not rush commands
3. **Enable BT function with proper timeout** - This is where most hang
4. **Check USB power state** - Avoid autosuspend during init
5. **Match firmware version to chip** - Wrong FW causes silent failures

If your Linux driver hangs during startup, focus on:
- Adding debug logs around WMT FUNC_CTRL (step 5)
- Verifying firmware load completion (step 3.3)
- Checking USB interrupt endpoint is receiving events
- Increasing timeouts for WMT commands to 2-3 seconds

Good luck with your Linux driver development!
