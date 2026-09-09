# Sending TPMS Readings to IQ (Python)

This guide walks you through writing a Python script that reads raw PressurePro TPMS packets from an RS-232-to-Ethernet bridge and posts them to the IQ External API so readings appear in IQ.

For the full endpoint reference (schemas, auth, and interactive Try-it-out), see the [External API (IQ)](../software/external-api/index.md) docs.

## What you need

| Item | Details |
| --- | --- |
| **API key** | PressurePro will create an External API key for your company and send it to you directly. Treat it like a password; do not commit it to source control or share it publicly. |
| **Device serial number** | The reader / bridge device identifier agreed with PressurePro (`deviceSerialNumber` in the request body). |
| **API base URL** | `https://prd.api.pressurepro.us` |
| **Bridge** | An RS-232-to-Ethernet converter that exposes the TPMS receiver UART stream over TCP (typical baud rate: **38400**, 8N1). |
| **Python 3.9+** | And the `requests` package (`pip install requests`). |

## How the flow works

```text
TPMS sensors  →  RF receiver (UART)  →  RS-232 ↔ Ethernet bridge  →  your Python script  →  POST /external/sensorReadings  →  IQ
```

1. Sensors transmit; your receiver outputs fixed-length binary frames on UART.
2. The Ethernet bridge forwards that byte stream on a TCP port.
3. Your script connects to the bridge, parses frames, and periodically POSTs a JSON payload to IQ.
4. IQ associates readings with your company from the API key — you do **not** send `companyId`, `source`, or `apiKeyId`.

## Authentication

Every request must include:

```http
Authorization: Bearer <your-api-key>
Content-Type: application/json
```

A successful ingest returns **HTTP 202** with a body like `{"status":"accepted"}`.

## Packet format

Each sensor reading is a **9-byte** frame (18 hex characters):

| Bytes (hex offsets) | Field | Notes |
| --- | --- | --- |
| `0–1` | Preamble | Sensor reading: `80`, `88`, `84`, `90`, or `98`. Heartbeat: `FA` (ignore for ingest). |
| `2–7` | Sensor serial | Six hex characters (e.g. `F142CA`). |
| `8–9` | Pressure | Unsigned byte → PSI (0–255). |
| `10–11` | Temperature | Encoded °F (see decode helper below). |
| `12–13` | Signal RSSI | Unsigned byte. |
| `14–15` | Ambient RF (`ambRF`) | Unsigned byte. |
| `16–17` | Checksum | Two’s complement of the sum of the preceding bytes. |

Example frame: `88315D2090374718A4`

Only post frames that pass checksum validation and have a sensor-reading preamble. Skip heartbeats (`FA…`).

## Request body

`POST /external/sensorReadings`

```json
{
  "timestamp": "2025-01-01T12:30:00Z",
  "latitude": 45.632158,
  "longitude": -90.32546,
  "deviceSerialNumber": "R15334",
  "tpms": [
    {
      "sensorSerialNumber": "F142CA",
      "pressure": 144,
      "temperature": 87.5,
      "rssi": 71,
      "ambRF": 24,
      "rxPacketData": "88315D2090374718A4"
    }
  ]
}
```

| Field | Required | Description |
| --- | --- | --- |
| `timestamp` | Yes | When the batch was taken (ISO 8601 UTC recommended). |
| `deviceSerialNumber` | Yes | Your reader / device serial. |
| `tpms` | Yes | One or more sensor readings (`minItems: 1`). |
| `latitude` / `longitude` | No | Optional site coordinates for the reader. |
| `tpms[].sensorSerialNumber` | Yes | Six-character hex sensor ID from the frame. |
| `tpms[].pressure` | Yes | Pressure in PSI. |
| `tpms[].temperature` | Yes | Temperature in °F. |
| `tpms[].rssi` | Yes | Signal RSSI. |
| `tpms[].ambRF` | Yes | Ambient / background RF. |
| `tpms[].rxPacketData` | Yes | Full 18-character hex frame (uppercase recommended). |

## Example Python script

The script below:

- Connects to your Ethernet bridge as a TCP client
- Accumulates bytes and extracts 9-byte frames
- Decodes and checksum-validates readings
- Batches unique sensors and POSTs on an interval (similar to a yard-reader polling cycle)

Configure the constants at the top, or prefer environment variables for the API key.

```python
#!/usr/bin/env python3
"""
Read PressurePro TPMS frames from an RS-232-to-Ethernet bridge
and post them to the IQ External API.
"""

from __future__ import annotations

import os
import socket
import time
from datetime import datetime, timezone
from typing import Optional

import requests

# --- Configuration (replace with your values) ---
API_BASE_URL = os.environ.get("PP_API_BASE_URL", "https://prd.api.pressurepro.us")
API_KEY = os.environ["PP_API_KEY"]  # provided directly by PressurePro
DEVICE_SERIAL = os.environ.get("PP_DEVICE_SERIAL", "R15334")

BRIDGE_HOST = os.environ.get("PP_BRIDGE_HOST", "192.168.1.50")
BRIDGE_PORT = int(os.environ.get("PP_BRIDGE_PORT", "4001"))

# Optional reader location
LATITUDE: Optional[float] = None   # e.g. 45.632158
LONGITUDE: Optional[float] = None  # e.g. -90.32546

UPLOAD_INTERVAL_SEC = 60  # how often to POST collected readings
SOCKET_TIMEOUT_SEC = 5.0

READING_PREAMBLES = {"80", "88", "84", "90", "98"}
HEARTBEAT_PREAMBLE = "FA"


def calculate_checksum(frame_hex: str) -> str:
    """Two's complement of the sum of all bytes except the checksum byte."""
    body = frame_hex[:-2]
    total = sum(int(body[i : i + 2], 16) for i in range(0, len(body), 2))
    return f"{(~total + 1) & 0xFF:02X}"


def calculate_temperature(hex_byte: str) -> float:
    """Decode temperature byte to Fahrenheit (−50°F … ~127.5°F)."""
    value = int(hex_byte, 16)
    if value >= 128:
        value -= 128
    return -50.0 + value * 2.5


def decode_frame(frame_hex: str) -> Optional[dict]:
    """Return a tpms object if the frame is a valid sensor reading; else None."""
    if len(frame_hex) != 18:
        return None

    frame = frame_hex.upper()
    preamble = frame[0:2]
    if preamble == HEARTBEAT_PREAMBLE or preamble not in READING_PREAMBLES:
        return None
    if calculate_checksum(frame) != frame[16:18]:
        return None

    return {
        "sensorSerialNumber": frame[2:8],
        "pressure": int(frame[8:10], 16),
        "temperature": calculate_temperature(frame[10:12]),
        "rssi": int(frame[12:14], 16),
        "ambRF": int(frame[14:16], 16),
        "rxPacketData": frame,
    }


def post_readings(tpms: list[dict]) -> None:
    if not tpms:
        return

    payload = {
        "timestamp": datetime.now(timezone.utc).strftime("%Y-%m-%dT%H:%M:%SZ"),
        "deviceSerialNumber": DEVICE_SERIAL,
        "tpms": tpms,
    }
    if LATITUDE is not None and LONGITUDE is not None:
        payload["latitude"] = LATITUDE
        payload["longitude"] = LONGITUDE

    url = f"{API_BASE_URL.rstrip('/')}/external/sensorReadings"
    headers = {
        "Authorization": f"Bearer {API_KEY}",
        "Content-Type": "application/json",
    }

    response = requests.post(url, json=payload, headers=headers, timeout=30)
    if response.status_code == 202:
        print(f"Accepted {len(tpms)} reading(s)")
    else:
        print(f"Upload failed: {response.status_code} {response.text}")
        response.raise_for_status()


def feed_bytes(buffer_hex: str, data: bytes, latest: dict[str, dict]) -> str:
    """Append new bytes; extract and decode complete frames into `latest`."""
    buffer_hex += data.hex()
    while len(buffer_hex) >= 18:
        preamble = buffer_hex[0:2].upper()
        if preamble not in READING_PREAMBLES and preamble != HEARTBEAT_PREAMBLE:
            # Resync one byte at a time when stream is misaligned
            buffer_hex = buffer_hex[2:]
            continue

        frame = buffer_hex[0:18]
        buffer_hex = buffer_hex[18:]
        reading = decode_frame(frame)
        if reading:
            latest[reading["sensorSerialNumber"]] = reading
            print(
                f"{reading['sensorSerialNumber']}: "
                f"{reading['pressure']} PSI, {reading['temperature']}°F"
            )
    return buffer_hex


def main() -> None:
    print(f"Connecting to bridge {BRIDGE_HOST}:{BRIDGE_PORT} …")
    with socket.create_connection((BRIDGE_HOST, BRIDGE_PORT), timeout=SOCKET_TIMEOUT_SEC) as sock:
        sock.settimeout(SOCKET_TIMEOUT_SEC)
        print("Connected. Listening for frames (Ctrl+C to stop).")

        buffer_hex = ""
        latest: dict[str, dict] = {}
        next_upload = time.monotonic() + UPLOAD_INTERVAL_SEC

        while True:
            try:
                chunk = sock.recv(4096)
                if not chunk:
                    raise ConnectionError("Bridge closed the connection")
                buffer_hex = feed_bytes(buffer_hex, chunk, latest)
            except socket.timeout:
                pass

            if time.monotonic() >= next_upload:
                try:
                    post_readings(list(latest.values()))
                except requests.RequestException as exc:
                    print(f"Network error during upload: {exc}")
                latest.clear()
                next_upload = time.monotonic() + UPLOAD_INTERVAL_SEC


if __name__ == "__main__":
    try:
        main()
    except KeyboardInterrupt:
        print("\nStopped.")
```

### Running it

```bash
export PP_API_KEY="your-api-key-from-pressurepro"
export PP_API_BASE_URL="https://prd.api.pressurepro.us"
export PP_DEVICE_SERIAL="R15334"
export PP_BRIDGE_HOST="192.168.1.50"
export PP_BRIDGE_PORT="4001"

pip install requests
python ingest_sensor_readings.py
```

## Bridge tips

- Confirm the bridge is in **TCP server** mode (your script is the client) or adjust the script if your device expects the opposite.
- Match UART settings to the receiver: **38400 baud, 8 data bits, no parity, 1 stop bit** is the usual PressurePro configuration.
- If frames never validate, check byte order / framing on the bridge and that you are not double-converting hex (the TCP stream should be raw binary, not ASCII hex text).

## Batching and retries

- Keep the **latest reading per sensor serial** in memory and upload on an interval (the example uses 60 seconds). That reduces load without losing the freshest value.
- On non-202 responses, log the status and body. **401** usually means a missing/invalid key; **400** means the JSON failed validation (check required fields and types).
- Do not send empty `tpms` arrays — wait until at least one valid reading is buffered.

## Security checklist

- Store the API key in an environment variable or secret store, not in the script file.
- Use HTTPS only (the base URLs above already do).
- Rotate the key with PressurePro if it is ever exposed.

## Next steps

- Exercise the same endpoint from the browser via the [External API (IQ)](../software/external-api/index.md) Swagger UI (Authorize with your key).
- Contact your PressurePro representative if you need an API key or a confirmed `deviceSerialNumber` for your installation.
