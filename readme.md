# 🖨️ Bambu Lab & Tuya Heater Automation

**Goal:** Automatically control a chamber heater (via Tuya Smart Plug) based on the status of a Bambu Lab 3D printer (e.g., turn ON when printing ABS/ASA, turn OFF when idle or finished).

**Prerequisites:**
* Running **n8n** server (Self-hosted).
* **Bambu Lab Printer** (P1/X1 series) with LAN Mode enabled (or cloud credentials).
* **Tuya IoT Developer Account** (Free tier is fine).
* **Tuya-compatible Smart Plug** controlling your heater.

---

## 🛠️ Step 1: Configure n8n Environment
The Tuya API requires cryptographic signing (HMAC-SHA256). By default, n8n blocks the `crypto` module in the "Code" node.

1.  Open your n8n Docker configuration (e.g., `docker-compose.yml` or Unraid template).
2.  Add the following environment variable:
    ```bash
    NODE_FUNCTION_ALLOW_BUILTIN=crypto
    ```
    *(Alternatively, set it to `*` to allow all built-in modules).*
3.  **Restart your n8n container** for changes to take effect.

---

## ☁️ Step 2: Get Tuya API Credentials
To control the plug via script, you need "Cloud" access, not just the App.

1.  Go to [Tuya IoT Platform](https://iot.tuya.com/) and log in.
2.  **Cloud > Development > Create Project**.
3.  Link your Tuya App account:
    * Go to **Devices > Link Tuya App Account**.
    * Scan the QR code with the Tuya Smart App on your phone.
    * Your smart plug should now appear in the device list.
4.  Copy these 3 values for later:
    * **Access ID (Client ID)**
    * **Access Secret (Client Secret)**
    * **Device ID** (of the smart plug)

---

## 🔄 Step 3: The n8n Workflow
Create a new workflow in n8n.

### **Node 1: Trigger (Bambu Lab)**
* **Node Type:** MQTT Trigger (if using LAN mode) or HTTP Polling (if using Cloud).
* **Recommended (MQTT):**
    * **Topic:** `device/[YOUR_SERIAL_NUMBER]/report`
    * **Credentials:** Printer IP, Port `8883`, User `bblp`, Password `[ACCESS_CODE]`.
    * *Note: This gives you live data like `nozzle_temp`, `chamber_temp`, and `gcode_state`.*

### **Node 2: Filter / Switch (Logic)**
* **Node Type:** Switch (or If).
* **Logic:**
    * Check `gcode_state`: Is it `RUNNING`?
    * Check Filament type (if available in payload) or Bed Temp: Is it > 90°C (implies ABS/ASA)?
    * Check Chamber Temp: Is it < 45°C?
* **Outcome:** If Printing ABS **AND** Chamber is Cold → **Route to "Turn ON"**.

### **Node 3: The Tuya Signer (JavaScript)**
This is the tricky part. You need to generate a signature for the API request.
* **Node Type:** Code (JavaScript).
* **Code Snippet:**
    ```javascript
    const crypto = require('crypto');

    // Your Credentials
    const clientId = 'YOUR_ACCESS_ID';
    const clientSecret = 'YOUR_ACCESS_SECRET';
    const deviceId = 'YOUR_DEVICE_ID';

    // Timestamp
    const t = Date.now().toString();

    // Command to Turn ON (value: true) or OFF (value: false)
    // CHANGE THIS BOOLEAN BASED ON THE PATH (ON vs OFF)
    const command = {
      "commands": [
        { "code": "switch_1", "value": true } 
      ]
    };

    // Calculate Signature (HMAC-SHA256)
    // Tuya Sign String: clientId + accessToken + t + nonce + stringToSign
    // Simplify for "Simple Mode" auth if enabled, or use standard V2 signing:
    // This is a simplified "Easy Mode" sign example often used in generic scripts:
    const signStr = clientId + t + "POST" + "\n" +
                    "e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855" + // Empty payload hash
                    "\n" + "" + "\n" + "/v1.0/devices/" + deviceId + "/commands";

    const sign = crypto.createHmac('sha256', clientSecret).update(signStr).digest('HEX').toUpperCase();

    return {
        json: {
            t,
            sign,
            clientId,
            deviceId,
            command
        }
    };
    ```

### **Node 4: Send Command (HTTP Request)**
* **Node Type:** HTTP Request.
* **Method:** POST.
* **URL:** `https://openapi.tuyaus.com/v1.0/devices/{{ $json.deviceId }}/commands` *(Adjust URL for your region: US/EU/CN)*.
* **Headers:**
    * `client_id`: `{{ $json.clientId }}`
    * `sign`: `{{ $json.sign }}`
    * `t`: `{{ $json.t }}`
    * `sign_method`: `HMAC-SHA256`
* **Body:** JSON.
* **JSON Value:** `{{ $json.command }}`

---

## 🧪 Troubleshooting

* **"Module 'crypto' is disallowed":** You missed Step 1. Check your environment variables.
* **Tuya "Permission Denied":** Ensure your Tuya Cloud Project has the "Smart Home Basic Service" API enabled (it usually has a trial period).
* **Wrong Region:** If `tuyaus.com` fails, try `tuyaeu.com` (Europe) or `tuyacn.com` (Asia).