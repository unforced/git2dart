# Android Platform Setup

git2dart supports Android with full HTTPS/TLS capabilities via OpenSSL.

## Requirements

- Android API 21+ (Android 5.0+)
- Currently supports arm64-v8a architecture

## SSL/HTTPS Setup (Required)

Android requires manual SSL certificate initialization in your app's `main()` function.

### Quick Setup

Add this to your app's `main()` function:

```dart
import 'dart:io';
import 'package:flutter/material.dart';
import 'package:git2dart/git2dart.dart';

void main() async {
  WidgetsFlutterBinding.ensureInitialized();

  // Initialize SSL certificates for Android
  if (Platform.isAndroid) {
    // 1. Initialize libgit2 first
    final version = Libgit2.version;
    
    // 2. Extract and configure SSL certificates
    final certPath = await AndroidSSLHelper.initialize();
    Libgit2.setSSLCertLocations(file: certPath);
  }

  runApp(MyApp());
}
```

### Why is this needed?

Android apps cannot access system CA certificates via standard filesystem paths. git2dart bundles Mozilla's trusted root certificates and extracts them to your app's cache directory on first run.

### Initialization Order

**Critical**: You must initialize libgit2 BEFORE configuring SSL certificates. The initialization order is:

1. Call `Libgit2.version` (or any Libgit2 method) to initialize libgit2
2. Call `AndroidSSLHelper.initialize()` to extract certificates
3. Call `Libgit2.setSSLCertLocations()` to configure SSL

If you configure SSL before libgit2 initializes, the configuration will be overwritten and HTTPS operations will fail.

## Storage Recommendations

Use app-private storage to avoid Android permission issues:

```dart
import 'package:path_provider/path_provider.dart';

Future<void> cloneRepo() async {
  // Use app-private directory
  final appDir = await getApplicationDocumentsDirectory();
  final repoPath = '${appDir.path}/my-repo';

  final repo = Repository.clone(
    url: 'https://github.com/user/repo.git',
    localPath: repoPath,
  );
}
```

## Complete Example

```dart
import 'dart:io';
import 'package:flutter/material.dart';
import 'package:git2dart/git2dart.dart';
import 'package:path_provider/path_provider.dart';

void main() async {
  WidgetsFlutterBinding.ensureInitialized();

  if (Platform.isAndroid) {
    final version = Libgit2.version;
    final certPath = await AndroidSSLHelper.initialize();
    Libgit2.setSSLCertLocations(file: certPath);
  }

  runApp(MyApp());
}

class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      home: GitDemo(),
    );
  }
}

class GitDemo extends StatefulWidget {
  @override
  _GitDemoState createState() => _GitDemoState();
}

class _GitDemoState extends State<GitDemo> {
  String _status = 'Ready';

  Future<void> _cloneRepo() async {
    setState(() => _status = 'Cloning...');

    try {
      final appDir = await getApplicationDocumentsDirectory();
      final repoPath = '${appDir.path}/my-repo';

      final repo = Repository.clone(
        url: 'https://github.com/flutter/flutter.git',
        localPath: repoPath,
      );

      setState(() => _status = 'Cloned successfully!');
    } catch (e) {
      setState(() => _status = 'Error: $e');
    }
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text('git2dart Android Demo')),
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            Text(_status),
            SizedBox(height: 20),
            ElevatedButton(
              onPressed: _cloneRepo,
              child: Text('Clone Repository'),
            ),
          ],
        ),
      ),
    );
  }
}
```

## Troubleshooting

### SSL errors

If you see SSL errors like "SSL error: unknown" or "no TLS stream available":

1. Verify you've added SSL initialization to `main()`
2. Check initialization order (libgit2 first, then SSL)
3. Ensure `WidgetsFlutterBinding.ensureInitialized()` is called before SSL setup

### Authentication for private repositories

Use personal access tokens for private repositories:

```dart
Repository.clone(
  url: 'https://github.com/user/private-repo.git',
  localPath: repoPath,
  callbacks: Callbacks(
    credentials: (url, usernameFromUrl, allowedTypes) {
      return Credential.userpassPlaintext(
        username: 'your-username',
        password: 'your-personal-access-token',
      );
    },
  ),
);
```

## Technical Details

- **libgit2 version**: 1.9.1
- **TLS backend**: OpenSSL 3.0.15
- **CA certificates**: Mozilla root certificate store
- **Library size**: ~5.7MB (includes OpenSSL)
- **Architectures**: arm64-v8a (additional architectures available on request)

## See Also

- [Main Documentation](README.md)
- [Repository Guide](types/repository.md)
- [Remote Operations](types/remote.md)
- [Credentials](types/credentials.md)
