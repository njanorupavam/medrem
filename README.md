# Medicine Reminder: Flutter App + ESP32 Buzzer

## What this project is
This project is a **medicine reminder system** made of two parts:

1. **ESP32 firmware (buzzer device)** that exposes simple HTTP endpoints.
2. **Flutter app (mobile + web)** that sends commands to the ESP32 and shows the timer status.

The app and ESP32 must be on the **same Wi-Fi network** so the app can reach the ESP32's local IP address.

---

## Who is it for
- **Patients** who need reminders to take medicine
- **Caregivers or family members** who need a simple web or mobile interface to trigger or stop reminders
- **Developers/students** learning IoT + Flutter integrations

---

## Why it exists
- A phone or browser can control the reminder without touching the hardware.
- The ESP32 can stay near the patient while the controller UI stays anywhere on the same Wi-Fi network.

---

## Where it runs
- **ESP32** runs the buzzer firmware (Arduino-style C/C++).
- **Flutter app** runs on:
  - Android / iOS (mobile)
  - Windows / macOS / Linux (desktop)
  - Web browser (GitHub Pages or local web server)

---

## When to use it
- When you want a **simple medicine alarm** with a remote on/off control.
- When you want a **visual schedule view** (calendar) and a **quick timer UI**.

---

## How it works (full flow)
1. The ESP32 boots and connects to your local Wi-Fi.
2. The ESP32 exposes HTTP endpoints like `/status`, `/time`, `/set`, `/stop`.
3. The Flutter app calls these endpoints over HTTP using the ESP32's local IP (for example: `http://10.130.109.134`).
4. The UI shows the timer status and lets the user set time, stop the alarm, and select calendar days.
5. The app polls the ESP32 every second to keep status and timer fresh.

---

## Core features
- Live status updates
- Live timer display
- Set alarm time (hour + minute)
- Stop alarm/buzzer
- Quick preset time buttons (5, 10, 20 mins)
- Calendar UI with multi-day selection
- Dynamic island style schedule summary

---

## ESP32 API endpoints (current)
These are called by the Flutter app:

- `GET /status` -> returns JSON like `{ "status": "ON" }`
- `GET /time` -> returns JSON like `{ "time": "00:45" }`
- `GET /set?hour=H&minute=M` -> sets alarm time
- `GET /stop` -> stops buzzer

**Important:** These endpoints are **not secured**. Anyone on the same Wi-Fi network who knows the ESP32 IP can call them.

---

## Security note (very important)
Right now, **anyone on the same Wi-Fi network** can control the buzzer if they know the ESP32 IP address. There is **no authentication** in the current firmware or app.

If you want to restrict access, add one of these later:
- a simple API key in the ESP32 endpoints
- basic auth in the firmware
- network isolation (guest Wi-Fi, firewall, VLAN)

---

## Languages and technologies used (and why)

### 1) Dart + Flutter
- **Why:** Flutter provides a single UI codebase for Android, iOS, web, and desktop.
- **Where used:** `lib/main.dart`, UI widgets, HTTP networking, calendar UI.

### 2) Arduino C/C++ (ESP32 firmware)
- **Why:** ESP32 firmware is written in Arduino-style C/C++ for microcontroller performance and Wi-Fi control.
- **Where used:** `buzz/buzz.ino` handles Wi-Fi connection, buzzer control, and HTTP routing.

### 3) HTML/CSS/JavaScript (Web output)
- **Why:** Flutter web builds to HTML/JS for browser execution.
- **Where used:** `docs/` and `build/web/` output folders.

### 4) YAML
- **Why:** Flutter and tools use YAML for configuration.
- **Where used:** `pubspec.yaml`, `analysis_options.yaml`, GitHub Actions workflow.

### 5) Kotlin/Gradle (Android)
- **Why:** Android builds use Gradle and Kotlin DSL for configuration.
- **Where used:** `android/` folder.

### 6) Swift/Objective-C (iOS)
- **Why:** iOS build files are generated for native integration.
- **Where used:** `ios/` folder.

### 7) CMake/C++ (Desktop)
- **Why:** Flutter desktop builds use CMake toolchains.
- **Where used:** `windows/`, `linux/`, `macos/` folders.

---

## Project structure overview
- `lib/main.dart` -> Flutter UI + HTTP logic
- `buzz/buzz.ino` -> ESP32 firmware
- `pubspec.yaml` -> Flutter dependencies
- `docs/` -> Web build for GitHub Pages
- `build/web/` -> Local web build output

---

## App configuration (important settings)

### Base URL (ESP32 IP)
In `lib/main.dart`:

```dart
String baseUrl = "http://10.130.109.134"; // change to your ESP32 IP
```

You must update this to match the ESP32's IP on your network.

---

## How to run locally (web)
```powershell
cd C:\Users\kaasi\Downloads\jun\med_reminder_app
flutter run -d chrome
```

---

## How to build the web app
```powershell
flutter build web --release --base-href /medrem/
```

Then copy to `docs/` for GitHub Pages:
```powershell
Copy-Item -Path build\web -Destination docs -Recurse -Force
New-Item -Path "docs/.nojekyll" -ItemType File -Force
```

---

## GitHub Pages deployment
The repo uses the `/docs` folder for Pages.

1. Go to: `Settings -> Pages`
2. Source: **Deploy from a branch**
3. Branch: **main**
4. Folder: **/docs**

Live site:
```
https://njanorupavam.github.io/medrem
```

---

## What the calendar currently does
The calendar UI **allows multi-day selection**, but **those selections are not sent to the ESP32 yet**. They are displayed in the UI only. You can wire this later by adding a new endpoint like `/schedule` and POSTing a JSON payload.

---

## Limitations (current)
- No authentication on ESP32 endpoints
- Base URL is hardcoded and must be updated manually
- Calendar schedule is UI-only (not yet synced to ESP32)

---

## Frequently asked "WH" questions (complete answers)

**Who uses it?**
Patients, caregivers, and anyone who needs a scheduled medicine alert.

**What does it do?**
It shows a timer, lets you set an alarm, and triggers/stops a buzzer on ESP32.

**When should it be used?**
When you want timed medicine reminders without touching the buzzer device.

**Where does it run?**
Flutter app runs on web/mobile/desktop; buzzer logic runs on the ESP32.

**Why is Flutter used?**
One codebase can run everywhere and create a modern UI quickly.

**How does the app communicate with ESP32?**
Via HTTP GET requests to ESP32 endpoints on the same Wi-Fi network.

---

## Future improvements (optional)
- Add authentication token to ESP32 endpoints
- Add schedule sync from calendar to ESP32
- Save last-used ESP32 IP in local storage
- Add push notifications on mobile
      setState(() {
        _message = 'Refreshing status...';
      });
    }

    try {
      final response = await http
          .get(Uri.parse('$baseUrl/status'))
          .timeout(const Duration(seconds: 5));

      if (!mounted) return;

      if (response.statusCode == 200) {
        setState(() {
          _isConnected = true;
          _status =
              response.body.trim().isEmpty ? 'Unknown' : response.body.trim();
          _message = 'ESP32 connected.';
        });
      } else {
        setState(() {
          _isConnected = false;
          _status = 'Unavailable';
          _message = 'ESP32 returned HTTP ${response.statusCode}.';
        });
      }
    } catch (_) {
      if (!mounted) return;
      setState(() {
        _isConnected = false;
        _status = 'Connection Error';
        _message =
            'Could not reach http://192.168.4.1. Connect the phone to ESP32 AP mode.';
      });
    }
  }

  Color _statusColor() {
    final value = _status.toLowerCase();
    if (value.contains('on') || value.contains('take')) {
      return Colors.red;
    }
    if (value.contains('off') || value.contains('wait')) {
      return Colors.green;
    }
    return Colors.orange;
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Medicine Reminder'),
        centerTitle: true,
      ),
      body: Padding(
        padding: const EdgeInsets.all(20),
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          crossAxisAlignment: CrossAxisAlignment.stretch,
          children: [
            Card(
              child: Padding(
                padding: const EdgeInsets.all(20),
                child: Column(
                  children: [
                    Text(
                      'Device Status',
                      style: Theme.of(context).textTheme.titleMedium,
                    ),
                    const SizedBox(height: 12),
                    Text(
                      _status,
                      textAlign: TextAlign.center,
                      style: TextStyle(
                        fontSize: 28,
                        fontWeight: FontWeight.bold,
                        color: _statusColor(),
                      ),
                    ),
                    const SizedBox(height: 12),
                    Text(
                      _message,
                      textAlign: TextAlign.center,
                    ),
                    const SizedBox(height: 12),
                    Row(
                      mainAxisAlignment: MainAxisAlignment.center,
                      children: [
                        Icon(
                          _isConnected ? Icons.wifi : Icons.wifi_off,
                          color: _isConnected ? Colors.green : Colors.red,
                        ),
                        const SizedBox(width: 8),
                        Text(_isConnected ? 'Connected' : 'Disconnected'),
                      ],
                    ),
                  ],
                ),
              ),
            ),
            const SizedBox(height: 24),
            SizedBox(
              height: 52,
              child: ElevatedButton(
                onPressed: _isBusy ? null : () => _sendCommand('on'),
                child: const Text('ON'),
              ),
            ),
            const SizedBox(height: 16),
            SizedBox(
              height: 52,
              child: ElevatedButton(
                onPressed: _isBusy ? null : () => _sendCommand('off'),
                child: const Text('OFF'),
              ),
            ),
            const SizedBox(height: 16),
            SizedBox(
              height: 52,
              child: OutlinedButton(
                onPressed: _isBusy ? null : _fetchStatus,
                child: const Text('Refresh Status'),
              ),
            ),
            if (_isBusy) ...[
              const SizedBox(height: 20),
              const Center(child: CircularProgressIndicator()),
            ],
          ],
        ),
      ),
    );
  }
}
```
