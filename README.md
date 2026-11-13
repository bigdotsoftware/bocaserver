# BOCA Ticket Server

**A modular Node.js printing server for BOCA Lemur ticket printers using the FGL46 command language.**

BOCA Ticket Server provides a complete software solution for managing BOCA ticket printers, designing ticket templates, handling print jobs, managing fonts and graphics, and communicating with printers bidirectionally.

Developed by **Big Dot Software**.
https://bigdotsoftware.pl/

---

## Features

- **Printer Management** – Manage multiple BOCA printers from a single server.
- **FGL46 Engine** – Extended implementation of the BOCA FGL46 command language.
- **Professional Ticket Designer** – Browser-based visual editor for creating and editing ticket templates.
- **REST API** – Integrate ticket printing into external applications.
- **Bidirectional Communication** – TCP and USB communication with printer status monitoring and acknowledgements.
- **Template Engine** – Dynamic YAML/JSON templates with variables and configurable elements.
- **Font Management** – Upload and transfer TrueType and Soft Font files to printer memory.
- **Logo and Graphics Management** – Upload, convert and transfer graphics to printer memory.
- **Barcode Support** – Native 1D and 2D barcode generation using BOCA commands.
- **Print Queue** – Job management, history, status tracking and retry functionality.
- **USB Support** – Direct USB printing on Linux, Windows and macOS, subject to platform and licensing module requirements.
- **Printer Discovery** – Detect USB and serial devices.
- **License Management** – License validation and automatic renewal.
- **Multilingual Interface** – English and Polish web interface.

## Compatibility

The base commands and sequences are implemented according to the official **BOCA FGL46 Programming Guide Rev.16.5 (April 7, 2025)**.

Actual feature availability depends on printer firmware revision, controller board, installed options and hardware configuration.

BOCA publishes newer revisions of its programming guide. Features introduced after Rev.16.5 should be verified against the documentation and firmware version applicable to the target printer.

## Requirements

- Node.js >= 20.9
- BOCA Lemur printer with TCP/Ethernet interface (default port: 9100)
- Printer status reporting enabled for the selected interface.
- For Flash operation acknowledgements: `FLASH ACK MODE` enabled in the printer.

## Installation

```bash
unzip <package>.zip
cd <package>

# Configure your API
nano config/server.yaml
nano config/printers.yaml

npm install
npm start
```

Web interface:

```text
http://localhost:4100
```

Printer configuration:

```text
config/printers.yaml
```

Server configuration:

```text
config/server.yaml
```

## License Configuration

Before starting the server, configure the license file path using either:

- `license.file` in `config/server.yaml`
- `LICENSE_FILE` in `.env`

If both are configured, `LICENSE_FILE` takes precedence.

The path can be absolute or relative to the application directory.


### Unregistered version

The server can start without a valid license. However, ticket template printing and test printing will include the following watermark in the upper-left corner:

```text
Printed from unregistred version. Visit www.bigdotsoftware.pl
```

The watermark may overlap ticket content.

License validation is performed for every print job. If the native module is unavailable, the print job fails.

**Do not modify or distribute the native licensing module source code in customer deployments.**

This protection is intended to discourage unauthorized use, not to provide tamper-proof enforcement. A user controlling the local JavaScript, native module or printer connection may alter it.

Raw FGL and command endpoints remain available without a license and do not add the watermark. Raw commands inside templates can also affect the printed output.


### Automatic license renewal

At startup and every 24 hours, the server attempts to renew the license when fewer than 10 days remain before expiration, including after expiration.

Before replacing the original license, the server creates a backup:

```text
.license.backup-<UUID>
```

The application requires write permissions in the license directory.

If downloading or validation fails, the existing license remains unchanged. Renewal errors are recorded in the server log and do not prevent startup.

### Checking the license

```bash
curl -H "x-api-key: $API_KEY" \
  http://localhost:4100/api/license
```

A valid license returns:

```json
{
  "ok": true,
  "license": {
    "company": "...",
    "extras": [],
    "errorMessage": "",
    "expired": false
  }
}
```

The `expired` property is only present for licenses with an expiration date.

Missing or invalid licenses return HTTP 503:

```json
{
  "ok": false,
  "error": "..."
}
```

### Forcing license renewal

```bash
curl -X POST \
  -H "x-api-key: $API_KEY" \
  http://localhost:4100/api/license/renew
```

This endpoint requests renewal regardless of the 10-day renewal threshold.

A successful response:

```json
{
  "ok": true,
  "renewed": true
}
```

An unsuccessful renewal returns HTTP 503. If another renewal is already in progress, HTTP 409 is returned.

Licenses without an expiration date cannot be renewed through this endpoint.


## Server Logs

Application logs are stored in the directory configured by `storage.logs` in `config/server.yaml`.

Default location:

```text
./data/logs
```

Log files follow this naming convention:

```text
server-YYYY-MM-DD.log
```

A new file is created on the first log entry after local midnight. Timestamps inside log entries are in UTC.

Logs include:

- Application messages.
- Synchronization events.
- HTTP 4xx and 5xx response codes.

HTTP log entries do not contain request parameters.

Old log files are not automatically deleted.

Printer communication logs are stored separately and use their own rotation mechanism.

## FGL46 Engine

The FGL engine supports the following command groups and printing functionality.

### Layout and Text

- Resident fonts
- Soft Fonts
- TrueType fonts
- Resident TrueType fonts
- Reverse printing
- Font translation

### Graphics and Barcodes

- FGL graphics and ASCII graphics with binary payloads
- Resident logos and downloaded logos
- FGL graphics download to printer memory using ESC sequences.
- Native 1D barcode printing:
  - UPC
  - EAN-8
  - EAN-13
  - Code 39
  - Interleaved 2 of 5
  - Codabar
  - Code 128
  - Softstrip
- Native 2D barcode printing:
  - QR Code
  - PDF417
  - Data Matrix
  - Aztec
- orientation P/L and barcode parameters.
- Insertion of 1-bit BMP/PNG/PCX images into FGL streams using the `image` template element.

2D barcodes are generated using native BOCA printer commands, not server-side generated images.

### Printing and Control

- Print and cut
- Print without cutting
- Counters
- P1/P2 paper paths
- Buffering modes
- Multiple-ticket operations
- Purge and parking
- Cash drawer control
- Administrative and configuration commands through token-based passthrough


The index includes basic commands and specialized command groups, including RFID, configuration, Ethernet/Wi-Fi, memory and dual-path functionality.

The engine validates individual FGL command tokens and prevents injection of command separators or additional streams.

Command availability still depends on the printer model, firmware and installed hardware options.

**Important:** Commands that write to Flash memory, clear memory, change DPI or interface settings, or reset the printer are marked as persistent/destructive. Do not execute them during normal printing operations.

Do not change `dpi200/300/600` programmatically. An incorrect value may damage the printhead.

## Bidirectional Communication and Printer Status

The TCP transport maintains a persistent connection and provides bidirectional communication with the printer.

Features include:

- Persistent TCP sessions.
- Byte-stream parsing independent of TCP packet boundaries.
- XON/XOFF flow control.
- Chunked transmission with XON waiting.
- Print acknowledgements (ACK).
- NAK and error detection.
- Paper and ticket status monitoring.
- Flash error detection.
- Cutter jam detection.
- Ticket taken/waiting notifications.
- Power-on notifications.
- Text-based S2/S7/S9 responses.
- S92 `0x41` status responses.

Operations on the same printer are serialized. Status queries and font/logo transfers cannot interfere with an active print job.

### Supported status bytes

| Hex | Description |
|---|---|
| `06` | ACK – ticket printed |
| `0A / 0B` | Out of paper – path 1 / path 2 |
| `0C / 0D` | Paper loaded – path 1 / path 2 |
| `0E` | Escrow jam |
| `0F` | Low paper |
| `10` | Out of paper |
| `11 / 13` | XON / XOFF |
| `12` | Power on |
| `14` | Bad Flash memory |
| `15` | NAK / illegal print command |
| `16 / 17` | Ticket taken / ticket waiting |
| `18 / 19` | Paper jam / illegal data |
| `1A` | Power-up problem |
| `1C` | Downloading error |
| `1D / 1E / 1F` | Cutter jam / stuck ticket / cutter jam path 2 |
| `41` | Solicited good status – response to S92 |

Status reporting depends on the printer's configured status mode.

In the default solicited status mode, not all errors are reported asynchronously.

For backward compatibility with version 4.0, `S1` remains the default status query.

`S92` can be used for full status information in alternative solicited modes. Set `statusCommand: S92` only after configuring the printer for `s90` or `s91`.

### ACK modes

ACK confirms the event defined by the printer's `ACKNOWLEDGE` setting:

- `NORMAL`
- `DELAYED`

Configure the mode in `config/printers.yaml`:

```yaml
ackMode: normal
```

For delayed ACK printers:

```yaml
ackMode: delayed
```

In delayed mode, the server sends `<S3>` and combines copies from one request into a single operation, avoiding the need to wait for an ACK before sending the next ticket.

An ACK should not automatically be interpreted as physical ticket collection by the customer. When exit-opto is available, ticket-taken reporting is provided separately.

If an ACK timeout occurs, the print result is marked as `unknown`. The server does not automatically retry the operation, preventing accidental duplicate tickets.

## Font Management

BOCA Ticket Server supports font uploads and transfers to printer memory.

Supported file formats:

- `.ttf`
- `.sfp`
- `.sfl`

### Uploading a font

```bash
curl -X POST http://localhost:4100/api/assets/fonts \
  -H 'x-api-key: YOUR_API_KEY' \
  -F 'file=@./Arial.ttf'
```

### Uploading a font to the printer

```bash
curl -X POST \
  http://localhost:4100/api/printers/boca-main/fonts/Arial.ttf/upload \
  -H 'x-api-key: YOUR_API_KEY' \
  -H 'Content-Type: application/json' \
  -d '{"id":1,"awaitFlashAck":false}'
```

The binary font data follows the command sequence.

Soft Font SFP/SFL files are transferred as binary data using the selected PF/TF mode.

To wait for Flash acknowledgement:

1. Enable `FLASH ACK MODE` on the printer.
2. Set `flashAckEnabled: true` in the printer configuration.
3. Use `awaitFlashAck: true` in the upload request.

The `0x11` response is shared with XON in this mode. Flash transfer acknowledgement must therefore be enabled carefully and tested with the actual printer configuration.

### Synchronizing assets

```bash
curl -X POST \
  http://localhost:4100/api/printers/boca-main/assets/sync \
  -H "Content-Type: application/json" \
  -H "x-api-key: YOUR_API_KEY"
```

### Using fonts in templates

```yaml
elements:
  - type: text
    x: 40
    y: 80
    ttfId: 1
    pointSize: 22
    value: "Ticket {{ticketId}}"

  - type: text
    x: 80
    y: 80
    softFontId: 2
    value: "Additional font"
```

## Logo and Graphics Management

PNG, JPEG and WebP images can be uploaded to the server.

### Uploading a logo

```bash
curl -X POST \
  http://localhost:4100/api/assets/company-logo \
  -H 'x-api-key: YOUR_API_KEY' \
  -F 'file=@./logo.png'
```

### Transferring a logo to the printer

```bash
curl -X POST \
  http://localhost:4100/api/printers/boca-main/logos/company-logo.png/upload \
  -H 'x-api-key: YOUR_API_KEY' \
  -H 'Content-Type: application/json' \
  -d '{"id":1,"mode":"temporary","widthDots":300,"threshold":160,"awaitFlashAck":false}'
```

The visual designer allows you to:

- Add or replace logo files.
- Select a logo ID.
- Configure logo width.
- Choose temporary or permanent printer memory.
- Upload the selected logo to the printer.

### Using logos in templates

```yaml
elements:
  - type: logo
    x: 40
    y: 80
    id: 1
    resident: false
```

The `resident` property determines which logo command is used.

## Ticket Templates and Printing API

The printing endpoint accepts several template formats:

```http
POST /api/tickets/print
```

The `template` field supports:

1. The name of an existing saved template.
2. A JSON object containing the template definition.
3. A string containing a JSON or YAML template definition.

A template must contain an `elements` array, following the same structure as templates stored in the `templates` directory.

Templates submitted directly through the API are not saved to the templates directory.

The complete request, including the template definition and `data` values, is stored in the job history.

### Printing an existing template

```bash
curl -X POST \
  http://localhost:4100/api/tickets/print \
  -H 'x-api-key: YOUR_API_KEY' \
  -H 'Content-Type: application/json' \
  -d '{
    "printerId": "boca-main",
    "template": "sample-ticket",
    "data": {
      "ticketId": "123"
    },
    "copies": 1
  }'
```

### Printing using an inline JSON template

```bash
curl -X POST \
  http://localhost:4100/api/tickets/print \
  -H 'x-api-key: YOUR_API_KEY' \
  -H 'Content-Type: application/json' \
  -d '{
    "printerId": "boca-main",
    "template": {
      "cut": true,
      "elements": [
        {
          "type": "text",
          "x": 30,
          "y": 50,
          "value": "Ticket {{ticketId}}"
        }
      ]
    },
    "data": {
      "ticketId": "123"
    }
  }'
```

### Printing using YAML inside JSON

```bash
curl -X POST \
  http://localhost:4100/api/tickets/print \
  -H 'x-api-key: YOUR_API_KEY' \
  -H 'Content-Type: application/json' \
  -d '{
    "printerId": "boca-main",
    "template": "cut: true\nelements:\n  - type: text\n    x: 30\n    y: 50\n    value: Ticket {{ticketId}}\n",
    "data": {
      "ticketId": "123"
    }
  }'
```

### Sending the complete request as YAML

```bash
curl -X POST \
  http://localhost:4100/api/tickets/print \
  -H 'x-api-key: YOUR_API_KEY' \
  -H 'Content-Type: application/yaml' \
  --data-binary $'printerId: boca-main\ntemplate:\n  cut: true\n  elements:\n    - type: text\n      x: 30\n      y: 50\n      value: "Ticket {{ticketId}}"\ndata:\n  ticketId: "123"\n'
```

JSON and YAML requests have a 2 MB size limit.

A successful HTTP `202` response means that the job has been added to the queue. It does not confirm that the ticket has been physically printed. Check the job status to determine the final result.

### Print Queue and Job History

The web interface provides:

- Print job history.
- Job status monitoring.
- Detailed job information.
- Saved template name or inline template definition.
- Template data used for rendering.
- Copyable cURL commands.
- Manual job resubmission after confirmation.

Status indicators:

- `printed` – Green.
- `failed` – Red.
- `unknown` or interrupted – Yellow.

Resubmission uses the current version of a saved template, which may differ from the version used by the original job.

Test printing uses the current server time.

Jobs without the required data cannot be resubmitted through the interface.

When API authentication is enabled, generated cURL commands use the `API_KEY` environment variable and do not expose the key stored in the browser.

## Printer Status and Commands API

### Query printer status

```bash
curl -X POST \
  http://localhost:4100/api/printers/boca-main/status/query \
  -H 'x-api-key: YOUR_API_KEY' \
  -H 'Content-Type: application/json' \
  -d '{"command":"S1"}'
```

A `0x41` response indicates a good status.

### Read printer counter and PROM version

```bash
curl -X POST \
  http://localhost:4100/api/printers/boca-main/status/query \
  -H 'x-api-key: YOUR_API_KEY' \
  -H 'Content-Type: application/json' \
  -d '{"command":"S2"}'
```

### Send an individual FGL command

```bash
curl -X POST \
  http://localhost:4100/api/printers/boca-main/command \
  -H 'x-api-key: YOUR_API_KEY' \
  -H 'Content-Type: application/json' \
  -d '{"command":"S92"}'
```

### Retrieve the FGL command index

```bash
curl http://localhost:4100/api/fgl/commands \
  -H 'x-api-key: YOUR_API_KEY'
```


## Example Ticket Template

The following example demonstrates native barcode support:

```yaml
name: sample-ticket
cut: true

elements:
  - type: text
    x: 30
    y: 50
    font: 3
    value: "BIG DOT SOFTWARE"

  - type: qr
    x: 50
    y: 180
    value: "{{ticketUrl}}"
    pointSize: 8
    version: 7
    errorCorrection: 1

  - type: pdf417
    x: 60
    y: 360
    value: "{{ticketPayload}}"
    columns: 0
    rows: 0

  - type: dataMatrix
    x: 60
    y: 600
    value: "{{ticketId}}"

  - type: aztec
    x: 60
    y: 760
    value: "{{ticketId}}"
```

QR versions 2, 7, 11 and 15, as well as error correction settings, require compatible printer firmware and font packages.

BOCA documentation specifies minimum firmware versions for individual QR versions.

## Professional Ticket Designer

BOCA Ticket Server includes a browser-based visual ticket designer.

### Designer Features

- List and load saved ticket templates.
- Create new templates.
- Save templates with automatically suggested names (`template-1`, `template-2`, etc.).
- Prevent accidental overwriting when creating a new template.
- Overwrite an existing template after confirmation.
- Workspace preview matching the selected printer's proportions and resolution.
- Media dimensions in centimeters, inches and printer dots.
- DPI information.
- Real-time editing of element position, dimensions, font and text rotation.
- Resident fonts F1–F16.
- Custom TrueType font library.
- Upload TTF fonts directly from text element properties.
- Workspace grid and zoom controls.
- Layer list and drag-and-drop element positioning.
- Test data preview with template variables.

**Font preview limitations:** Resident font previews are browser-based approximations. Actual appearance depends on BOCA firmware and the printer's font tables.

For TTF fonts, upload the font to the printer and use the same font ID in the design.

TTF font size is specified in points. Resident bitmap font size is controlled using FGL scaling (`HW`).

The browser preview does not replace a physical test print.

### BOCA FGL46 Font Types

The designer distinguishes between:

- Bitmap resident fonts F1–F16.
- Resident TrueType fonts RTF1–RTF10.
- Uploaded TTF files.

RTF availability depends on the font package installed in the printer firmware.

The standard FGL46 interface does not provide commands for reading binary resident fonts or listing files stored in printer memory. Consequently, browser previews use substitute font families rather than fonts downloaded from the printer.

## USB Printing

The server supports USB transport in Linux, Windows and macOS, subject to platform-specific requirements.

### USB Device Discovery

The endpoint:

```http
GET /api/discovery/scan?source=usb
```

detects locally connected USB Printer-class devices without sending commands to the printers.

Platform-specific discovery:

- Linux: sysfs.
- Windows: PowerShell/PnP.
- macOS: IORegistry (`ioreg`).

The response includes device identifiers and available device names.

On Linux, it also provides:

- `/dev/usb/lp*` device path, or `null`.
- `accessible` indicating whether read/write permissions are available.
- Warnings about missing permissions or the `usblp` driver.

On Windows and macOS, `path` is `null` and `accessible` is `false`, because discovery does not establish a transport path or verify device permissions.

USB transport selects devices using VID/PID and optionally a serial number.

USB discovery does not guarantee that the device can be accessed by the server.

The existing `source=all` and `source=rs232` discovery modes also include USB discovery alongside serial port scanning.

### Linux USB Configuration

Configure the printer in `config/printers.yaml`:

```yaml
transport:
  type: usb
  path: /dev/usb/lp0
  chunkSize: 1024
  writeTimeoutMs: 15000
  statusEnabled: true
  ackMode: normal

assets:
  autoSync: false
```

Use the actual device path returned by USB discovery.

The `/dev/usb/lp*` number may change after reconnecting the printer. A stable absolute symlink created using a udev rule is recommended.

The user running the server must have read/write permissions for the device. Other applications must not hold an exclusive lock on it.

USB uses the same FGL response parser as TCP, including support for:

- Status responses.
- ACK.
- INV/INV2.
- Flash acknowledgements.

Actual operation depends on printer firmware and status/ACK configuration.

With `statusEnabled: false`, printing does not wait for ACK. An `ack: false` result does not confirm physical printing.

A `ready` status without a printer response may only indicate that the device is open.

Automatic asset synchronization can be enabled after verifying INV2 support.

Write errors or missing ACKs produce an unknown print result. The server does not automatically retry, preventing duplicate tickets.

### Windows and macOS USB Configuration

Example USB transport configuration:

```yaml
transport:
  type: usb
  vendorId: "0a43"
  productId: "0201"

  # Optional: serial number from USB discovery
  # serialNumber: "printer-serial-number"

  # Optional: select a specific Printer interface
  # interfaceNumber: 0

  statusEnabled: true
  ackMode: normal

assets:
  autoSync: false
```

VID/PID values must be hexadecimal strings.

If multiple printers with identical VID/PID values are connected, `serialNumber` is required. Devices with duplicate serial numbers are rejected.

If the printer exposes multiple Printer-class interfaces, specify `interfaceNumber`.

The selected interface must support bidirectional communication using bulk IN and OUT endpoints.

On Windows, libusb may require a WinUSB driver instead of the standard printer driver. Driver installation must be performed by an administrator and may prevent the printer from being used through the standard Windows print queue.

On macOS, another process or driver may block USB access.

Windows and macOS USB operation has not been validated on physical printer hardware.

## Important Limitations

- TCP/Ethernet and USB transports are supported.
- RS232 and CUPS are not implemented as print adapters in this version.
- The FGL command catalog includes the basic FGL46 command set and extension groups. A syntactically valid command may not be supported by a particular printer or firmware.
- Font and logo transfers require available RAM/Flash memory.
- FGL46 distinguishes between temporary and permanent memory.
- `DF9` performs memory reclamation and may restart the printer.
- `awaitFlashAck` requires `FLASH ACK MODE` to be configured on the printer.
- Without Flash ACK mode, the server confirms transmission of bytes over TCP, not successful storage in printer memory.
- Physical BOCA Lemur hardware testing was not performed in the development environment.

After updating the server, the following physical tests are recommended:

1. Printer status query.
2. Temporary font upload.
3. Temporary logo upload.
4. Permanent memory operations only after successful temporary tests.

## API Authentication

API examples use the `x-api-key` HTTP header:

```http
x-api-key: YOUR_API_KEY
```

Configure the key in `.env`.

When API protection is enabled, use the configured key for protected endpoints.

Never expose your API key in publicly accessible repositories, frontend applications or client-side scripts.


## Project Information

**Product:** BOCA Ticket Server  
**Version:** 5.2.1  
**Developer:** Big Dot Software  
**Runtime:** Node.js >= 20.9  
**Printer language:** BOCA FGL46  
**Primary interface:** REST API  
**Web interface:** Included

Website: https://bigdotsoftware.pl/

---

**BOCA Ticket Server – Professional ticket printing, template management and printer integration by Big Dot Software.**
