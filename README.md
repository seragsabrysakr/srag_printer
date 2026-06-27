# srag_printer

`srag_printer` is a flexible Flutter thermal printing package for printing PDF widgets, PDF bytes, and image frames through ESC/POS thermal printers.

It is intentionally generic. It does not know about your POS models, tax rules, ZATCA QR generation, preferences, providers, or app assets. Your app builds the content; `srag_printer` renders and prints it.

## Acknowledgements

`srag_printer` is built on top of excellent Flutter/Dart packages, including:

- `pdf` for building PDF documents with Dart widgets.
- `pdfx` for rendering PDF pages into images.
- `image` for image decoding and processing.
- `esc_pos_utils_plus` for ESC/POS command generation utilities.
- `usb_serial` for Android USB serial communication.
- `flutter_blue_plus` for Bluetooth Low Energy discovery and communication.
- `ffi` for Windows RAW driver and serial port bindings.

Many thanks to the maintainers and contributors of these packages.

## What This Package Does

- Prints any `pw.Widget` from `package:pdf/widgets.dart`.
- Prints existing PDF bytes.
- Prints existing image frames as `List<Uint8List>`.
- Converts PDF pages into thermal-printer-width image frames.
- Converts images to ESC/POS raster commands.
- Sends bytes through network, USB, Bluetooth BLE, Windows driver, or Windows serial transports.
- Provides generic receipt widgets as optional helpers.
- Provides generic device discovery by type/platform.


## Installation

Add the package to your app. During local development inside this repository:

```yaml
dependencies:
  srag_printer:
    path: packages/srag_printer
```

Then import:

```dart
import 'package:srag_printer/srag_printer.dart';
import 'package:pdf/widgets.dart' as pw;
```

## Platform Setup and Permissions

The package sends bytes to printers, but each application owns its platform
permissions. The example app includes these settings as a reference.

### Android

Add the permissions you need to `android/app/src/main/AndroidManifest.xml`.
Keep hardware features optional if your app can still use network printing when
USB or Bluetooth hardware is unavailable.

```xml
<!-- Network printers, usually raw TCP on port 9100. -->
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />

<!-- USB host mode for Android USB printers. -->
<uses-feature android:name="android.hardware.usb.host" android:required="false" />

<!-- Bluetooth for Android 11 and below. Location is required by Android for BLE scanning. -->
<uses-permission android:name="android.permission.BLUETOOTH" android:maxSdkVersion="30" />
<uses-permission android:name="android.permission.BLUETOOTH_ADMIN" android:maxSdkVersion="30" />
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" android:maxSdkVersion="30" />
<uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" android:maxSdkVersion="30" />

<!-- Bluetooth for Android 12+. -->
<uses-permission
    android:name="android.permission.BLUETOOTH_SCAN"
    android:usesPermissionFlags="neverForLocation" />
<uses-permission android:name="android.permission.BLUETOOTH_CONNECT" />

<uses-feature android:name="android.hardware.bluetooth" android:required="false" />
<uses-feature android:name="android.hardware.bluetooth_le" android:required="false" />
```

Runtime permissions are still required before Bluetooth discovery. The example
uses `permission_handler` to request `bluetoothScan`, `bluetoothConnect`, and
`locationWhenInUse` on Android.

USB note: Android USB access is device-selection dependent. The OS may show a
USB permission prompt when a USB transport opens the selected device.

### iOS

For network printers on the local network and BLE discovery/connect, add usage
descriptions to `ios/Runner/Info.plist`:

```xml
<key>NSBluetoothAlwaysUsageDescription</key>
<string>This app uses Bluetooth to discover and connect to BLE thermal printers.</string>
<key>NSBluetoothPeripheralUsageDescription</key>
<string>This app uses Bluetooth to discover and connect to BLE thermal printers.</string>
<key>NSLocalNetworkUsageDescription</key>
<string>This app connects to network thermal printers on your local network.</string>
```

iOS does not support generic USB thermal printer access in v1. Classic
Bluetooth SPP is also not available for generic printers; use network or BLE
printers that expose writable BLE characteristics.

### macOS

If your macOS app uses the app sandbox, add outbound network and Bluetooth
entitlements. The example adds these to both debug/profile and release
entitlements:

```xml
<key>com.apple.security.network.client</key>
<true/>
<key>com.apple.security.device.bluetooth</key>
<true/>
```

### Windows

No manifest permissions are required for raw TCP sockets, installed Windows
printer RAW output, or COM-port access. Users may still need normal OS-level
access to the selected printer/port, and firewall rules can affect network
printers.

### Linux

Network printing can work with normal socket access. USB/serial access usually
depends on system groups and udev rules such as `dialout` or `lp`; configure
those at the OS level for your target device.

## Platform Support

| Capability | Windows | Android | iOS |
|---|---:|---:|---:|
| Print `pw.Widget` | Yes | Yes | Yes |
| Print PDF bytes | Yes | Yes | Yes |
| Print image frames | Yes | Yes | Yes |
| Network TCP/IP | Yes | Yes | Yes |
| USB direct | Yes via serial/driver | Yes | Not in v1 |
| Windows driver RAW | Yes | No | No |
| Windows COM serial | Yes | No | No |
| Bluetooth classic | Not in v1 | Not in v1 | Not in v1 |
| BLE Bluetooth | Yes | Possible | Possible |
| Generic USB discovery | COM/system devices | USB devices | No |
| Network discovery | Manual IP/port | Manual IP/port | Manual IP/port |

## Quick Start: Network Printer

```dart
final printer = SragPrinter(
  transport: NetworkPrinterTransport(host: '192.168.1.50', port: 9100),
  config: const SragPrinterConfig(
    paper: SragPaper.mm80(headDots: 528),
  ),
);

await printer.printWidget(
  (_) => pw.Column(
    children: [
      pw.Text('Hello from srag_printer'),
      pw.Text('Printed from a PDF widget'),
    ],
  ),
);
```

## Print Any PDF Widget

Use this when your app wants total control over layout:

```dart
await printer.printWidget(
  (_) => pw.Directionality(
    textDirection: pw.TextDirection.rtl,
    child: pw.Column(
      children: [
        pw.Text('فاتورة تجريبية'),
        pw.Divider(),
        pw.Text('You can render any pdf widget here.'),
      ],
    ),
  ),
);
```

## Print PDF Bytes

```dart
final pdfBytes = await buildMyPdfSomewhereElse();
await printer.printPdf(pdfBytes);
```

## Print Image Frames

```dart
final frames = <Uint8List>[pngFrame1, pngFrame2];
await printer.printImages(frames);
```

## Receipt Widgets

The receipt widgets are optional helpers. You can use them, mix them with your own `pw.Widget`s, or ignore them completely.

```dart
await printer.printWidget(
  (_) => SragReceipt(
    textDirection: pw.TextDirection.ltr,
    children: [
      ReceiptHeader(
        titles: ['Simplified Tax Invoice', 'فاتورة ضريبية مبسطة'],
        infoRows: const [
          ReceiptKeyValue(label: 'Invoice No', value: '10001'),
          ReceiptKeyValue(label: 'Date', value: '2026-06-27'),
        ],
      ),
      ReceiptTable<Map<String, String>>(
        columns: [
          ReceiptTableColumn(title: 'Product', flex: 4, valueBuilder: (x) => x['name']!),
          ReceiptTableColumn(title: 'Qty', valueBuilder: (x) => x['qty']!),
          ReceiptTableColumn(title: 'Total', valueBuilder: (x) => x['total']!),
        ],
        rows: const [
          {'name': 'Coffee', 'qty': '1', 'total': '12.00'},
        ],
      ),
      ReceiptTotals(
        rows: const [
          ReceiptKeyValue(label: 'Subtotal', value: '12.00'),
          ReceiptKeyValue(label: 'Tax', value: '1.80'),
          ReceiptKeyValue(label: 'Total', value: '13.80'),
        ],
      ),
      ReceiptFooter(lines: const ['Thank you']),
    ],
  ),
);
```

## Custom Table Rows

```dart
ReceiptTable<MyItem>(
  columns: columns,
  rows: items,
  rowBuilder: (context, item, columns) {
    return pw.Column(
      children: [
        pw.Text(item.name),
        pw.Text('Custom options, modifiers, or notes'),
      ],
    );
  },
);
```

## QR Codes

`srag_printer` does not generate QR content. Generate the QR in your app, then pass it as an image or widget:

```dart
ReceiptQrSection.image(qrPngBytes);
ReceiptQrSection.widget(myQrPdfWidget);
```

## Discovery

Discovery is generic and platform-aware. It does not hardcode printer brands or choose for you:

```dart
final devices = await SragPrinterDiscovery.discover(
  request: const PrinterDiscoveryRequest(
    connectionType: PrinterConnectionType.usb,
  ),
);

final selected = devices.first; // Your UI or app policy chooses this.
final transport = selected.createTransport();
```

Optional filters are app-controlled:

```dart
final devices = await SragPrinterDiscovery.discover(
  request: const PrinterDiscoveryRequest(
    connectionType: PrinterConnectionType.usb,
    filters: {'nameContains': 'printer'},
  ),
);
```

## Paper and Print Options

```dart
const config = SragPrinterConfig(
  paper: SragPaper.mm58(headDots: 384),
  chunkSize: 512,
  chunkDelay: Duration(milliseconds: 6),
  feedLines: 3,
  cutAfterPrint: true,
);
```

Use `SragPaper.custom(widthMm: ..., headDots: ...)` for uncommon printers.

## Troubleshooting

- `Cannot reach printer at host:port`: check IP, port, network, and firewall.
- `No writable BLE characteristic found`: the printer may use classic Bluetooth SPP or a proprietary BLE service.
- `USB transport is supported on Android`: use Windows driver/serial transports on Windows.
- `Image frame could not be decoded`: pass PNG/JPEG bytes, not raw pixels.
- Cut command ignored: some printers do not have a cutter, or the driver may block raw ESC/POS commands.

## Migration Guide

Keep app-specific logic in your app:

```text
SalesInvoice / app model -> your adapter -> pw.Widget or receipt data -> srag_printer
```

Do not move tax, QR generation, ZATCA logic, local preferences, or app assets into this package.

## FAQ

**Can I print any Flutter widget?**  
No. The input must be a `pdf/widgets.dart` widget (`pw.Widget`), not a normal Flutter `Widget`.

**Can the package discover network printers automatically?**  
Not in v1. Use manual IP/port for reliability and predictable behavior.

**Does iOS support USB printing?**  
Generic USB thermal printing is not supported in v1.

**Does the package generate QR codes?**  
No. Pass a ready QR image or `pw.Widget`.
