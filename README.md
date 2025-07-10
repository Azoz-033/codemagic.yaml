name: dralfadhli_lawapp
description: تطبيق مكتب المحاماة د. الفضلي لعرض القضايا والتقارير.

publish_to: 'none'

version: 1.0.0+1

environment:
  sdk: ">=2.19.0 <3.0.0"

dependencies:
  flutter:
    sdk: flutter

  firebase_core: ^2.22.0
  firebase_auth: ^4.6.1
  cloud_firestore: ^4.15.4
  firebase_storage: ^11.6.5
  file_picker: ^5.2.3
  url_launcher: ^6.1.10
  flutter_localizations:
    sdk: flutter

dev_dependencies:
  flutter_test:
    sdk: flutter
  flutter_lints: ^2.0.0

flutter:
  uses-material-design: true

  assets:
    - assets/images/logo.png

  fonts:
    - family: Cairo
      fonts:
        - asset: assets/fonts/Cairo-Regular.ttf














import 'package:flutter/material.dart';
import 'package:firebase_core/firebase_core.dart';
import 'package:flutter_localizations/flutter_localizations.dart';
import 'screens/login_screen.dart';

void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await Firebase.initializeApp();
  runApp(LawOfficeApp());
}

class LawOfficeApp extends StatefulWidget {
  @override
  _LawOfficeAppState createState() => _LawOfficeAppState();
}

class _LawOfficeAppState extends State<LawOfficeApp> {
  Locale _locale = Locale('ar');

  void setLocale(Locale locale) {
    setState(() {
      _locale = locale;
    });
  }

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'مكتب د. الفضلي',
      debugShowCheckedModeBanner: false,
      locale: _locale,
      supportedLocales: const [
        Locale('ar'),
        Locale('en'),
      ],
      localizationsDelegates: const [
        GlobalMaterialLocalizations.delegate,
        GlobalWidgetsLocalizations.delegate,
        GlobalCupertinoLocalizations.delegate,
      ],
      theme: ThemeData(
        primaryColor: Color(0xFF3b3f9c),
        scaffoldBackgroundColor: Colors.white,
        fontFamily: 'Cairo',
        appBarTheme: AppBarTheme(
          backgroundColor: Color(0xFF3b3f9c),
          centerTitle: true,
        ),
        elevatedButtonTheme: ElevatedButtonThemeData(
          style: ElevatedButton.styleFrom(
            backgroundColor: Color(0xFF3b3f9c),
            shape: RoundedRectangleBorder(
              borderRadius: BorderRadius.circular(8),
            ),
          ),
        ),
      ),
      home: Directionality(
        textDirection: _locale.languageCode == 'ar'
            ? TextDirection.rtl
            : TextDirection.ltr,
        child: LoginScreen(onLocaleChange: setLocale),
      ),
    );
  }
}













import 'package:flutter/material.dart';
import 'package:firebase_auth/firebase_auth.dart';
import 'client_cases_screen.dart';

class LoginScreen extends StatefulWidget {
  final Function(Locale) onLocaleChange;

  const LoginScreen({Key? key, required this.onLocaleChange}) : super(key: key);

  @override
  _LoginScreenState createState() => _LoginScreenState();
}

class _LoginScreenState extends State<LoginScreen> {
  final _emailController = TextEditingController();
  final _passwordController = TextEditingController();
  String _errorMessage = '';
  bool _isLoading = false;

  Future<void> _login() async {
    setState(() {
      _errorMessage = '';
      _isLoading = true;
    });
    try {
      await FirebaseAuth.instance.signInWithEmailAndPassword(
        email: _emailController.text.trim(),
        password: _passwordController.text.trim(),
      );
      Navigator.pushReplacement(
        context,
        MaterialPageRoute(builder: (context) => ClientCasesScreen()),
      );
    } on FirebaseAuthException catch (e) {
      setState(() {
        if (e.code == 'user-not-found') {
          _errorMessage = 'المستخدم غير موجود';
        } else if (e.code == 'wrong-password') {
          _errorMessage = 'كلمة المرور غير صحيحة';
        } else {
          _errorMessage = 'خطأ في تسجيل الدخول';
        }
        _isLoading = false;
      });
    }
  }

  void _changeLanguage(String langCode) {
    Locale locale = Locale(langCode);
    widget.onLocaleChange(locale);
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text('تسجيل الدخول'),
        actions: [
          PopupMenuButton<String>(
            onSelected: _changeLanguage,
            icon: Icon(Icons.language),
            itemBuilder: (context) => [
              PopupMenuItem(
                value: 'ar',
                child: Text('العربية'),
              ),
              PopupMenuItem(
                value: 'en',
                child: Text('English'),
              ),
            ],
          ),
        ],
      ),
      body: Padding(
        padding: const EdgeInsets.all(16),
        child: ListView(
          children: [
            Image.asset('assets/images/logo.png', height: 100),
            SizedBox(height: 20),
            TextField(
              controller: _emailController,
              textAlign: TextAlign.right,
              decoration: InputDecoration(labelText: 'البريد الإلكتروني'),
              keyboardType: TextInputType.emailAddress,
            ),
            TextField(
              controller: _passwordController,
              textAlign: TextAlign.right,
              obscureText: true,
              decoration: InputDecoration(labelText: 'كلمة المرور'),
            ),
            SizedBox(height: 16),
            ElevatedButton(
              onPressed: _isLoading ? null : _login,
              child: _isLoading
                  ? CircularProgressIndicator(color: Colors.white)
                  : Text('دخول'),
            ),
            if (_errorMessage.isNotEmpty)
              Padding(
                padding: const EdgeInsets.only(top: 8),
                child: Text(
                  _errorMessage,
                  style: TextStyle(color: Colors.red),
                  textAlign: TextAlign.center,
                ),
              ),
          ],
        ),
      ),
    );
  }
}

















import 'package:flutter/material.dart';
import 'package:cloud_firestore/cloud_firestore.dart';
import 'package:firebase_auth/firebase_auth.dart';
import 'package:url_launcher/url_launcher.dart';

class ClientCasesScreen extends StatelessWidget {
  final _uid = FirebaseAuth.instance.currentUser?.uid;

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text('قضاياي')),
      body: StreamBuilder<QuerySnapshot>(
        stream: FirebaseFirestore.instance
            .collection('clients')
            .doc(_uid)
            .collection('cases')
            .orderBy('lastUpdated', descending: true)
            .snapshots(),
        builder: (context, snapshot) {
          if (snapshot.hasError) {
            return Center(child: Text('حدث خطأ في تحميل البيانات.'));
          }
          if (!snapshot.hasData || snapshot.data!.docs.isEmpty) {
            return Center(child: Text('لا توجد قضايا مسجلة حالياً.'));
          }

          final cases = snapshot.data!.docs;

          return ListView.builder(
            itemCount: cases.length,
            itemBuilder: (context, index) {
              final caseData = cases[index];
              return ListTile(
                title: Text(caseData['title']),
                subtitle: Text('رقم: ${caseData['caseId']}'),
                trailing: Icon(Icons.picture_as_pdf),
                onTap: () async {
                  final url = caseData['reportUrl'];
                  if (url == null || url.isEmpty) {
                    ScaffoldMessenger.of(context).showSnackBar(
                      SnackBar(content: Text('لا يوجد تقرير متاح')),
                    );
                    return;
                  }
                  if (await canLaunch(url)) {
                    await launch(url);
                  } else {
                    ScaffoldMessenger.of(context).showSnackBar(
                      SnackBar(content: Text('لا يمكن فتح التقرير')),
                    );
                  }
                },
              );
            },
          );
        },
      ),
    );
  }
}


















import 'package:flutter/material.dart';
import 'package:file_picker/file_picker.dart';
import 'package:firebase_storage/firebase_storage.dart';
import 'package:cloud_firestore/cloud_firestore.dart';

class UploadReportScreen extends StatefulWidget {
  final String clientId;
  final String caseId;

  const UploadReportScreen({required this.clientId, required this.caseId, Key? key}) : super(key: key);

  @override
  _UploadReportScreenState createState() => _UploadReportScreenState();
}

class _UploadReportScreenState extends State<UploadReportScreen> {
  bool _isUploading = false;
  String? _uploadMessage;

  Future<void> _pickAndUpload() async {
    setState(() {
      _isUploading = true;
      _uploadMessage = null;
    });

    final result = await FilePicker.platform.pickFiles(
      type: FileType.custom,
      allowedExtensions: ['pdf'],
    );

    if (result == null) {
      setState(() {
        _isUploading = false;
        _uploadMessage = 'لم يتم اختيار ملف';
      });
      return;
    }

    final fileBytes = result.files.first.bytes;
    final fileName = result.files.first.name;

    try {
      final ref = FirebaseStorage.instance
          .ref()
          .child('reports/${widget.clientId}/${widget.caseId}/$fileName');

      await ref.putData(fileBytes!);

      final downloadUrl = await ref.getDownloadURL();

      await FirebaseFirestore.instance
          .collection('clients')
          .doc(widget.clientId)
          .collection('cases')
          .doc(widget.caseId)
          .update({'reportUrl': downloadUrl, 'last

      await FirebaseFirestore.instance
          .collection('clients')
          .doc(widget.clientId)
          .collection('cases')
          .doc(widget.caseId)
          .update({'reportUrl': downloadUrl, 'lastUpdated': DateTime.now()});

      setState(() {
        _uploadMessage = 'تم رفع التقرير بنجاح';
        _isUploading = false;
      });
    } catch (e) {
      setState(() {
        _uploadMessage = 'حدث خطأ أثناء رفع الملف';
        _isUploading = false;
      });
    }
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text('رفع تقرير جديد'),
      ),
      body: Padding(
        padding: const EdgeInsets.all(16),
        child: Column(
          children: [
            ElevatedButton.icon(
              icon: Icon(Icons.upload_file),
              label: Text('اختر ملف PDF ورفع'),
              onPressed: _isUploading ? null : _pickAndUpload,
            ),
            if (_isUploading)
              Padding(
                padding: const EdgeInsets.only(top: 20),
                child: CircularProgressIndicator(),
              ),
            if (_uploadMessage != null)
              Padding(
                padding: const EdgeInsets.only(top: 20),
                child: Text(



      await FirebaseFirestore.instance
          .collection('clients')
          .doc(widget.clientId)
          .collection('cases')
          .doc(widget.caseId)
          .update({'reportUrl': downloadUrl, 'lastUpdated': DateTime.now()});

      setState(() {
        _uploadMessage = 'تم رفع التقرير بنجاح';
        _isUploading = false;
      });
    } catch (e) {
      setState(() {
        _uploadMessage = 'حدث خطأ أثناء رفع الملف';
        _isUploading = false;
      });
    }
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text('رفع تقرير جديد'),
      ),
      body: Padding(
        padding: const EdgeInsets.all(16),
        child: Column(
          children: [
            ElevatedButton.icon(
              icon: Icon(Icons.upload_file),
              label: Text('اختر ملف PDF ورفع'),
              onPressed: _isUploading ? null : _pickAndUpload,
            ),
            if (_isUploading)
              Padding(
                padding: const EdgeInsets.only(top: 20),
                child: CircularProgressIndicator(),
              ),
            if (_uploadMessage != null)
              Padding(
                padding: const EdgeInsets.only(top: 20),
                child: Text(
                  _uploadMessage!,
                  style: TextStyle(
                      color:
                          _uploadMessage == 'تم رفع التقرير بنجاح' ? Colors.green : Colors.red),
                  textAlign: TextAlign.center,
                ),
              ),
          ],
        ),
      ),
    );
  }
}
name: dralfadhli_lawapp
description: تطبيق مكتب المحاماة د. الفضلي لعرض القضايا والتقارير.

publish_to: 'none' # يمنع النشر على pub.dev

version: 1.0.0+1

environment:
  sdk: ">=2.19.0 <3.0.0"

dependencies:
  flutter:
    sdk: flutter

  firebase_core: ^2.22.0
  firebase_auth: ^4.6.1
  cloud_firestore: ^4.15.4
  firebase_storage: ^11.6.5
  file_picker: ^5.2.3
  url_launcher: ^6.1.10
  flutter_localizations:
    sdk: flutter

dev_dependencies:
  flutter_test:
    sdk: flutter
  flutter_lints: ^2.0.0

flutter:
  uses-material-design: true

  assets:
    - assets/images/logo.png

  fonts:
    - family: Cairo
      fonts:
        - asset: assets/fonts/Cairo-Regular.ttf



