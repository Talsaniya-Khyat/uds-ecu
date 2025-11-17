# PyUDSim: A High-Fidelity UDS ECU Simulator

![Python Version](https://img.shields.io/badge/python-3.8+-blue.svg)
![Status](https://img.shields.io/badge/status-stable-green.svg)

A powerful, multi-ECU UDS (ISO 14229-1) simulator in Python for automotive security testing, research, and development.

This tool runs on a virtual CAN interface (`vcan`) and realistically mimics the behavior of real-world ECUs. It enforces strict session and security rules, supports multi-frame ISO-TP communication, and is driven by a simple JSON configuration.

---

## 🚀 Key Features

* **Multi-ECU Simulation:** Simulate an entire car network (Engine, BCM, etc.) from a single config file.
* **High-Fidelity State:** The simulator is stateful. It tracks the current **Diagnostic Session** and **Security Access** level for each ECU.
* **Strict UDS Compliance:** Services are realistically locked down.
    * Try to flash in the *default session*? You'll get NRC `0x7F` (Service Not Supported In Active Session).
    * Try to write a DID without unlocking? You'll get NRC `0x33` (Security Access Denied).
* **Full ISO-TP Support:** Automatically handles multi-frame (ISO 15765-2) responses for services like `0x19` (Read DTC) or `0x22` (Read DID).
* **Config-Driven:** Easily define your own DIDs, DTCs, security rules, and routines in `ecu_config.json`. No code changes required.
* **Realistic Security:** The `0x27` (Security Access) service generates a random seed and writes the correct key to `key_solution.txt` for easy testing.
* **Modular Framework:** Built with an object-oriented `BaseEcu` class, making it simple to extend or add new, custom services.

---

## 🔧 Installation & Setup

This tool is built for a Linux environment with `can-utils`.

### 1. Clone the Repository
```bash
git clone [https://github.com/your-username/pyudsim.git](https://github.com/your-username/pyudsim.git)
cd pyudsim
````

### 2\. Install Dependencies

The only dependency is `python-can`.

```bash
pip3 install -r requirements.txt
# (or just: pip3 install python-can)
```

### 3\. Set Up Virtual CAN (vcan)

You must create a virtual CAN interface for the simulator and your test tools to communicate on.

```bash
# Load the vcan kernel module
sudo modprobe vcan

# Create the 'vcan0' interface
sudo ip link add dev vcan0 type vcan

# Bring the interface online
sudo ip link set up vcan0
```

-----

## 🎮 How to Use

This simulator is designed to run in one terminal while you attack it from another.

### Terminal 1: Run the Simulator

Run `main.py` and point it to your CAN interface. Using `--debug` is highly recommended.

```bash
python3 main.py --iface vcan0 --debug
```

You will see the ECUs from your `ecu_config.json` file load and start listening.

```
Starting UDS Simulator on vcan0...
CAN Bus connection successful.
Loading ECUs from config...
  [+] Loaded ECU: EngineECU (Req: 0x7E0, Resp: 0x7E8)
  [+] Loaded ECU: BodyControlModule (Req: 0x7E1, Resp: 0x7E9)

Simulator running. Listening for 2 ECU(s). Press Ctrl+C to exit.
```

### Terminal 2: Send CAN Commands

Use `can-utils` (like `cansend`) or your favorite CAN tool to interact with the ECUs.

All UDS requests must be single-frame or multi-frame ISO-TP.

**Format:** `cansend <interface> <request_id>#<ISO-TP_DATA>`

**Example:**

  * **Service:** `10 03` (Go to Extended Session)
  * **ISO-TP Frame:** `02 10 03` (Length: 2, SID: 10, Sub-function: 03)
  * **Command:**
    ```bash
    cansend vcan0 7E0#021003
    ```

-----

## 📖 Example Walkthrough: Unlocking an ECU

Let's test the realism. We'll try to write to a DID on the `EngineECU` (ID `7E0`).

**1. Go to Extended Session (10 03)**

```bash
cansend vcan0 7E0#021003
```

  * **Response (in Terminal 1):** `< 065003003201F400` (Success\!)

**2. Try to write to DID `100B` (2E 100B AA)**

  * This DID requires Security Access (see `ecu_config.json`).
  * *This will fail.*

<!-- end list -->

```bash
cansend vcan0 7E0#052E100BAA
```

  * **Response:** `< 037F2E3300000000` (NRC: `0x33` - Security Access Denied)

**3. Request Seed (27 01)**

```bash
cansend vcan0 7E0#022701
```

  * **Response:** `< 066701AABBCCDD00` (Success, with a 4-byte seed, e.g., `AABBCCDD`)

**4. Get the Key**

  * The simulator has just created a file named `key_solution.txt`.

<!-- end list -->

```bash
cat key_solution.txt
```

```
ECU: EngineECU
SEED: 0xAABBCCDD
KEY:  0x2D239A99
```

**5. Send the Key (27 02)**

  * Use the key from the file.

<!-- end list -->

```bash
cansend vcan0 7E0#0627022D239A99
```

  * **Response:** `< 0267020000000000` (Success\! The ECU is now unlocked.)

**6. Try to Write Again (2E 100B AA)**

  * Now that we are in the extended session AND unlocked...
  * *This will succeed.*

<!-- end list -->

```bash
cansend vcan0 7E0#052E100BAA
```

  * **Response:** `< 036E100B00000000` (Positive Response\! The DID was written.)

-----

## ⚙️ Configuration (`ecu_config.json`)

This file is the brain of the simulator. You can define all your ECUs, DIDs, DTCs, and security rules here.

```json
{
  "ecus": [
    {
      "name": "EngineECU",
      "request_id": "7E0",
      "response_id": "7E8",
      "session_timeout": 5,
      "initial_dtcs": [
        { "code": "P0101", "status": 42 },
        { "code": "P0300", "status": 10 }
      ],
      "dids": {
        "F190": { "data": "4543552D53572D56312E322E33" },
        "100B": {
          "data": "FEEDFACE",
          "writable": true,
          "session": [3],
          "security": [1]
        }
      },
      "routines": {
        "FF01": {
          "name": "Check System",
          "status": "idle",
          "results": "C0FFEE",
          "session": [3],
          "security": [0, 1]
        }
      }
    }
  ]
}
```

  * **`initial_dtcs`**: A list of DTCs the ECU will have at boot.
  * **`dids`**: Define Data Identifiers.
      * **`data`**: The hex data the DID will return.
      * **`writable`**: (Optional) `true` if `0x2E` is allowed.
      * **`session`**: (Optional) A list of session IDs (1, 2, 3) where this DID is accessible.
      * **`security`**: (Optional) A list of security levels (0=Locked, 1=Unlocked) where this DID is accessible.
  * **`routines`**: Define Routines for `0x31`, each with its own `session` and `security` rules.

-----

## ✅ Implemented Services

| SID | Service Name | Status | Notes |
|---|---|---|---|
| `0x10` | Diagnostic Session Control | **Fully Implemented** | Responds with P2/P2\* timings. |
| `0x11` | ECU Reset | **Fully Implemented** | Simulates `hardReset` & `softReset`. |
| `0x14` | Clear DTCs | **Fully Implemented** | Clears the internal DTC list. Requires Extended. |
| `0x19` | Read DTC Information | **Fully Implemented** | Sub-funcs `0x01`, `0x02`. Requires Extended. |
| `0x22` | Read Data by Identifier | **Fully Implemented** | Reads from JSON, enforces session/security. |
| `0x23` | Read Memory by Address | **Fully Implemented** | Reads from 64KB sim-memory. Requires SecAccess. |
| `0x27` | Security Access | **Fully Implemented** | Seed/Key (XOR algo). Creates `key_solution.txt`. |
| `0x2E` | Write Data by Identifier | **Fully Implemented** | Writes to config state. Requires SecAccess. |
| `0x2F` | Input Output Control | **Fully Implemented** | Controls JSON IO state. Requires SecAccess. |
| `0x31` | Routine Control | **Fully Implemented** | Start/Stop/Results. Enforces rules from JSON. |
| `0x34` | Request Download | **Fully Implemented** | Starts flash sequence. Requires Prog/SecAccess. |
| `0x36` | Transfer Data | **Fully Implemented** | Handles data blocks for flashing. |
| `0x37` | Request Transfer Exit | **Fully Implemented** | Verifies flash and exits state. |
| `0x3D` | Write Memory by Address | **Fully Implemented** | Writes to 64KB sim-memory. Requires Prog/SecAccess. |
| `0x3E` | Tester Present | **Fully Implemented** | Resets session timer. |
| `0x85` | Control DTC Setting | **Fully Implemented** | Turns DTC reporting On/Off. Requires Extended. |
| *other* | All Other SIDs | **Stubbed** | Return NRC `0x11` (Service Not Supported). |

-----

## 🤝 Contributing

Pull Requests are welcome\! If you add a new feature or fix a bug, please feel free to contribute.

1.  Fork the repository.
2.  Create your feature branch (`git checkout -b feature/AmazingFeature`).
3.  Commit your changes (`git commit -m 'Add some AmazingFeature'`).
4.  Push to the branch (`git push origin feature/AmazingFeature`).
5.  Open a Pull Request.


