# ESP32 Medicine Reminder Flutter App

This Flutter app controls an ESP32 medicine reminder buzzer over HTTP while the ESP32 is running in Wi-Fi Access Point mode.

Base URL used by the app:

`http://192.168.4.1`

Endpoints used:

- `GET /on`
- `GET /off`
- `GET /status`

## Features

- ON button to trigger medicine reminder
- OFF button to stop buzzer/reminder
- Live status refresh every 2 seconds
- Manual refresh button
- Connection error handling
- Works with ESP32 AP mode without internet

## Android Setup Fix

Your Android SDK path should be:

`C:\Users\kaasi\AppData\Local\Android\Sdk`

The missing part was:

`C:\Users\kaasi\AppData\Local\Android\Sdk\cmdline-tools\latest\bin\sdkmanager.bat`

### Install Android command-line tools in PowerShell

```powershell
New-Item -ItemType Directory -Force -Path "C:\Users\kaasi\AppData\Local\Android\Sdk\cmdline-tools\latest"
Invoke-WebRequest -Uri "https://dl.google.com/android/repository/commandlinetools-win-13114758_latest.zip" -OutFile "$env:TEMP\cmdline-tools.zip"
Expand-Archive -Path "$env:TEMP\cmdline-tools.zip" -DestinationPath "$env:TEMP\cmdline-tools-extract" -Force
Move-Item -Path "$env:TEMP\cmdline-tools-extract\cmdline-tools\*" -Destination "C:\Users\kaasi\AppData\Local\Android\Sdk\cmdline-tools\latest" -Force
[Environment]::SetEnvironmentVariable("ANDROID_HOME","C:\Users\kaasi\AppData\Local\Android\Sdk","User")
[Environment]::SetEnvironmentVariable("ANDROID_SDK_ROOT","C:\Users\kaasi\AppData\Local\Android\Sdk","User")
$userPath = [Environment]::GetEnvironmentVariable("Path","User")
$newPaths = @(
  "C:\Users\kaasi\AppData\Local\Android\Sdk\platform-tools",
  "C:\Users\kaasi\AppData\Local\Android\Sdk\cmdline-tools\latest\bin"
)
$combined = ($userPath.Split(';') + $newPaths | Select-Object -Unique) -join ';'
[Environment]::SetEnvironmentVariable("Path",$combined,"User")
```

Close PowerShell and open a new one, then run:

```powershell
sdkmanager --version
sdkmanager --sdk_root="C:\Users\kaasi\AppData\Local\Android\Sdk" "platform-tools" "platforms;android-35" "build-tools;35.0.0"
flutter doctor --android-licenses
flutter doctor
```

## Run On Android Phone

1. Enable Developer Options on the phone.
2. Enable USB Debugging.
3. Connect the phone by USB.
4. Accept the debugging prompt on the phone.
5. Check device detection:

```powershell
adb devices
flutter devices
```

6. Run the app:

```powershell
cd C:\Users\kaasi\Downloads\jun\med_reminder_app
flutter run
```

## ESP32 AP Mode Notes

- Connect the phone to the ESP32 Wi-Fi network.
- Keep the ESP32 IP as `192.168.4.1`.
- If the phone disconnects because there is no internet, turn off mobile data or disable automatic network switching.

## Android Manifest Requirements

The app uses plain HTTP to the ESP32 local IP, so Android needs:

- `android.permission.INTERNET`
- `android:usesCleartextTraffic="true"`

These are already added in `android/app/src/main/AndroidManifest.xml`.

## main.dart

Current Flutter app code:

```dart
import 'dart:async';

import 'package:flutter/material.dart';
import 'package:http/http.dart' as http;

void main() {
  runApp(const MyApp());
}

class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      debugShowCheckedModeBanner: false,
      title: 'Medicine Reminder',
      theme: ThemeData(
        colorScheme: ColorScheme.fromSeed(seedColor: Colors.teal),
        useMaterial3: true,
      ),
      home: const HomePage(),
    );
  }
}

class HomePage extends StatefulWidget {
  const HomePage({super.key});

  @override
  State<HomePage> createState() => _HomePageState();
}

class _HomePageState extends State<HomePage> {
  static const String baseUrl = 'http://192.168.4.1';

  String _status = 'Checking device...';
  String _message = 'Connect your phone to the ESP32 Wi-Fi access point.';
  bool _isBusy = false;
  bool _isConnected = false;
  Timer? _pollTimer;

  @override
  void initState() {
    super.initState();
    _fetchStatus();
    _pollTimer = Timer.periodic(
      const Duration(seconds: 2),
      (_) => _fetchStatus(),
    );
  }

  @override
  void dispose() {
    _pollTimer?.cancel();
    super.dispose();
  }

  Future<void> _sendCommand(String endpoint) async {
    setState(() {
      _isBusy = true;
      _message = 'Sending command...';
    });

    try {
      final response = await http
          .get(Uri.parse('$baseUrl/$endpoint'))
          .timeout(const Duration(seconds: 5));

      if (response.statusCode == 200) {
        await _fetchStatus(showLoading: false);
        if (!mounted) return;
        setState(() {
          _message = 'Command sent successfully.';
        });
      } else {
        if (!mounted) return;
        setState(() {
          _isConnected = false;
          _message = 'ESP32 returned HTTP ${response.statusCode}.';
        });
      }
    } catch (_) {
      if (!mounted) return;
      setState(() {
        _isConnected = false;
        _message =
            'Connection failed. Check ESP32 Wi-Fi and USB/mobile network settings.';
      });
    } finally {
      if (mounted) {
        setState(() {
          _isBusy = false;
        });
      }
    }
  }

  Future<void> _fetchStatus({bool showLoading = true}) async {
    if (showLoading) {
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
