# <img src="n8n-color.svg" alt="n8n" height="24"> <img src="bambu%20lab%20logo%202.jpg" alt="Bambu Lab" height="24"> :fire: Bambu Lab & Tuya Heater Automation (n8n)

**Goal:** Use <img src="n8n-color.svg" alt="n8n" height="16"> n8n to automatically control a chamber heater (via <img src="tuya%20logo.jpg" alt="Tuya" height="16"> Tuya Smart Plug) based on the status of a <img src="bambu%20lab%20logo%202.jpg" alt="Bambu Lab" height="16"> Bambu Lab 3D printer (e.g., turn ON when printing ABS/ASA, turn OFF when idle or finished).

**Prerequisites:**
* <img src="n8n-color.svg" alt="n8n" height="16"> Running **n8n** server (Self-hosted).
* <img src="bambu%20lab%20logo%202.jpg" alt="Bambu Lab" height="16"> **Bambu Lab Printer** (P1/X1 series) with LAN Mode enabled (or cloud credentials).
* <img src="tuya%20logo.jpg" alt="Tuya" height="16"> **Tuya IoT Developer Account** (Free tier is fine).
* <img src="tuya%20logo.jpg" alt="Tuya" height="16"> **Tuya-compatible Smart Plug** controlling your heater.

---

## :package: Hardware links
* Heater: https://www.aliexpress.com/item/1005008379688337.html?mp=1&pdp_npi=6%40dis%21NZD%21NZD%2043.95%21NZD%2030.76%21%21NZD%2029.84%21%21%21%40210328c017680504728844834e073e%2112000044784322009%21ct%21NZ%21877674695%21%211%210%21
* Smart switch: https://www.aliexpress.com/item/1005005298261294.html?mp=1&pdp_npi=6%40dis%21NZD%21NZD%2020.23%21NZD%2010.24%21%21NZD%2010.14%21%21%21%40210328c017680504751634878e073e%2112000032537944217%21ct%21NZ%21877674695%21%212%210%21

---

## :camera: Screenshots n8n


**n8n workflow:** The logic listens to printer status, checks high-temp materials, and toggles the Tuya plug to heat or stop.

![n8n workflow](Actually%20working%20flow%20with%20pause%20resume%20commands.jpg)

---

## :hot_face: Heat soak via start G-code (any slicer)
Add a custom heat soak routine in your machine start G-code so high-temp materials warm the chamber before printing. This works in Bambu Studio or OrcaSlicer.

1) Find this section in your machine start G-code (around line 73):
```gcode
;===== heatbed preheat ====================
M1002 gcode_claim_action:54
M140 S[bed_temperature_initial_layer_single] ;set bed temp
M190 S[bed_temperature_initial_layer_single] ;wait for bed temp
```

2) Replace it with this block:
```gcode
; --- MOD START: MQTT Heat Soak Trigger (>85C Only) ---
; Only runs if the sliced bed temp is STRICTLY greater than 85C.
; ABS defaults to ~90C, so this will trigger for ABS/ASA and skip PLA/PETG/TPU.
{if bed_temperature_initial_layer_single > 85}
    M1002 gcode_claim_action : 2
    M140 S100    ; Force Bed to 100C for the soak
    M106 P3 S255 ; Turn Aux Fan to 100% to circulate heat
    G0 X128 Y250 F12000 ; Park toolhead at the back
    M400 U1      ; Sync moves
    M25          ; PAUSE PRINT (Triggers MQTT status: PAUSE)
{endif}
; --- MOD END ---
```

Note: the workflow currently resumes at **47C** (good if you do not have a heater installed yet). With a heater installed, bump the resume threshold to **55-60C**. Without a heater, the bed alone can take a long time to reach higher chamber temps.

Summary:
- Original code: only heats the bed and starts printing when the bed hits target.
- New code: if bed target is > 90C, it runs a heat soak routine to warm the chamber first.

---

## :wrench: Step 1: Configure <img src="n8n-color.svg" alt="n8n" height="18"> n8n Environment
The <img src="tuya%20logo.jpg" alt="Tuya" height="16"> Tuya API requires cryptographic signing (HMAC-SHA256). By default, <img src="n8n-color.svg" alt="n8n" height="16"> n8n blocks the `crypto` module in the "Code" node.

1. Open your n8n Docker configuration (e.g., `docker-compose.yml` or Unraid template).
2. Add the following environment variable:
    ```bash
    NODE_FUNCTION_ALLOW_BUILTIN=crypto
    ```
    *(Alternatively, set it to `*` to allow all built-in modules.)*
3. **Restart your n8n container** for changes to take effect.

---

## :key: Step 2: Get <img src="tuya%20logo.jpg" alt="Tuya" height="18"> Tuya API Credentials
To control the plug via script, you need "Cloud" access, not just the App.

1. Go to [Tuya IoT Platform](https://iot.tuya.com/) and log in.
2. **Cloud > Development > Create Project**.
3. Link your Tuya App account:
    * Go to **Devices > Link Tuya App Account**.
    * Scan the QR code with the Tuya Smart App on your phone.
    * Your smart plug should now appear in the device list.
4. Copy these 3 values for later:
    * **Access ID (Client ID)** -> `YOUR_TUYA_ACCESS_ID`
    * **Access Secret (Client Secret)** -> `YOUR_TUYA_ACCESS_SECRET`
    * **Device ID** (smart plug) -> `YOUR_TUYA_DEVICE_ID`

---

## :gear: Step 3: The <img src="n8n-color.svg" alt="n8n" height="18"> n8n Workflow
Create a new workflow in <img src="n8n-color.svg" alt="n8n" height="16"> n8n.

### **Node 1: Trigger (<img src="bambu%20lab%20logo%202.jpg" alt="Bambu Lab" height="16"> Bambu Lab)**
* **Node Type:** MQTT Trigger (if using LAN mode) or HTTP Polling (if using Cloud).
* **Recommended (MQTT):**
    * **Topic:** `device/YOUR_PRINTER_SN/report`
    * **Credentials:** Printer IP, Port `8883`, User `bblp`, Password `[ACCESS_CODE]`.
    * *Note: This gives you live data like `nozzle_temp`, `chamber_temp`, and `gcode_state`.*

### **Node 2: Filter / Switch (Logic)**
* **Node Type:** Switch (or If).
* **Logic:**
    * Check `gcode_state`: Is it `RUNNING`?
    * Check Filament type (if available in payload) or Bed Temp: Is it > 90C (implies ABS/ASA)?
    * Check Chamber Temp: Is it < 45C?
* **Outcome:** If Printing ABS **AND** Chamber is Cold -> **Route to "Turn ON"**.

### **Node 3: <img src="tuya%20logo.jpg" alt="Tuya" height="16"> Token Signer (JavaScript)**
Create a signed request to fetch a Tuya access token.
* **Node Type:** Code (JavaScript).
* **Code Snippet:**
    ```javascript
    const crypto = require('crypto');

    const clientId = 'YOUR_TUYA_ACCESS_ID';
    const clientSecret = 'YOUR_TUYA_ACCESS_SECRET';

    const t = Date.now().toString();
    const method = 'GET';
    const path = '/v1.0/token?grant_type=1';
    const contentHash = crypto.createHash('sha256').update('').digest('hex');
    const stringToSign = [method, contentHash, '', path].join('\n');
    const signStr = clientId + t + stringToSign;
    const sign = crypto.createHmac('sha256', clientSecret).update(signStr).digest('hex').toUpperCase();

    return {
      json: {
        client_id: clientId,
        t,
        sign,
        sign_method: 'HMAC-SHA256'
      }
    };
    ```

### **Node 4: Get Tuya Token (<img src="tuya%20logo.jpg" alt="Tuya" height="16"> HTTP Request)**
* **Node Type:** HTTP Request.
* **Method:** GET.
* **URL:** `https://openapi.tuyaus.com/v1.0/token?grant_type=1` *(Adjust URL for your region: US/EU/CN)*.
* **Headers:**
    * `client_id`: `{{ $json.client_id }}`
    * `sign`: `{{ $json.sign }}`
    * `t`: `{{ $json.t }}`
    * `sign_method`: `HMAC-SHA256`

### **Node 5: <img src="tuya%20logo.jpg" alt="Tuya" height="16"> Command Signer (JavaScript)**
Generate a signed IoT Core request to set `switch_1` true/false.
* **Node Type:** Code (JavaScript).
* **Code Snippet:**
    ```javascript
    const crypto = require('crypto');

    const clientId = 'YOUR_TUYA_ACCESS_ID';
    const clientSecret = 'YOUR_TUYA_ACCESS_SECRET';
    const deviceId = 'YOUR_TUYA_DEVICE_ID';
    const accessToken = $json.result.access_token;

    const properties = { switch_1: true }; // set false to turn OFF
    const body = JSON.stringify({ properties });

    const t = Date.now().toString();
    const path = `/v2.0/cloud/thing/${deviceId}/shadow/properties/issue`;
    const contentHash = crypto.createHash('sha256').update(body).digest('hex');
    const stringToSign = ['POST', contentHash, '', path].join('\n');
    const signStr = clientId + accessToken + t + stringToSign;
    const sign = crypto.createHmac('sha256', clientSecret).update(signStr).digest('hex').toUpperCase();

    return {
      json: {
        client_id: clientId,
        access_token: accessToken,
        t,
        sign,
        sign_method: 'HMAC-SHA256',
        device_id: deviceId,
        properties
      }
    };
    ```

### **Node 6: Send Command (<img src="tuya%20logo.jpg" alt="Tuya" height="16"> HTTP Request)**
* **Node Type:** HTTP Request.
* **Method:** POST.
* **URL:** `https://openapi.tuyaus.com/v2.0/cloud/thing/{{ $json.device_id }}/shadow/properties/issue` *(Adjust URL for your region: US/EU/CN)*.
* **Headers:**
    * `client_id`: `{{ $json.client_id }}`
    * `access_token`: `{{ $json.access_token }}`
    * `sign`: `{{ $json.sign }}`
    * `t`: `{{ $json.t }}`
    * `sign_method`: `HMAC-SHA256`
* **Body:** JSON.
* **JSON Value:** `properties: {{ $json.properties }}`

---

## :warning: Troubleshooting

* **"Module 'crypto' is disallowed":** You missed Step 1. Check your environment variables.
* **<img src="tuya%20logo.jpg" alt="Tuya" height="16"> Tuya "No permissions"/"Permission denied":** Ensure your Tuya Cloud Project is authorized for **IoT Core** and the device is linked to the project. The free trial resource pack is enough for device control.
* **Wrong Region:** If `tuyaus.com` fails, try `tuyaeu.com` (Europe) or `tuyacn.com` (Asia).
* **Auto-resume stays false/undefined:** Add a **Parse MQTT Message** code node right after the MQTT trigger to JSON-parse `message`, then drive the conditions from `print.*`. Resume payload must be `{"print":{"command":"resume","sequence_id":"2022"}}`.
* **Edits not taking effect:** n8n runs the last **active** version. After saving changes, deactivate and re-activate the workflow so the new version is deployed.
* **Slow File Uploads / Network Instability:** The P1/A1 series printers have limited CPU resources. Maintaining an active local MQTT connection (LAN Mode) with n8n while simultaneously uploading files (Bambu Studio) can saturate the printer's processor, causing uploads to fail or crawl. **Solution:** Temporarily disable the n8n workflow during large uploads, or switch n8n to connect to the **Bambu Cloud MQTT** broker instead of the printer's local IP. This offloads the connection overhead.

---

## :bulb: Troubleshooting: Tuya IoT Configuration
If you receive "Permission Denied" or "No Permissions" errors from Tuya, you likely need to subscribe to the Cloud Services.

1.  **Log in** to the [Tuya IoT Platform](https://iot.tuya.com/).
2.  Go to **Cloud > Projects** and open your project.
3.  Click **Service API**.
4.  Ensure **"IoT Core"** is listed and **Authorized**.
    *   *If not listed:* Click "Go to Service Market", search for "IoT Core", and "Subscribe" (it is free). Then add it to your project.
5.  **Check Device Linking:**
    *   Go to **Cloud > Development > Your Project > Devices**.
    *   Ensure your Smart Plug is listed here. If not, use the "Link Tuya App Account" tab to sync it from your phone.
6.  **Command Codes:**
    *   Most plugs use `switch_1` for the main power.
    *   Some strips use `switch_1`, `switch_2`, etc.
    *   Verify your device's instruction set: **Cloud > Development > Your Project > Devices > Debug Device**.


