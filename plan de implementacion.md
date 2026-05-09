```markdown
# Plan de Implementación: App Veterinaria "Polivet" con Flutter y Firebase

Este plan te guiará para construir la aplicación móvil de la clínica veterinaria "Polivet" utilizando **Flutter**, **Firebase Authentication** y **Cloud Firestore**, siguiendo la arquitectura propuesta (modelos, servicios, providers, pantallas). El entorno de desarrollo puede ser **VS Code** o **Antigravity** (IDX / cualquier IDE con soporte Flutter). Sigue cada paso en orden.

---

## 📋 Requisitos previos

### 1. Herramientas instaladas
- [Flutter SDK](https://docs.flutter.dev/get-started/install) (versión 3.x o superior)
- [Dart SDK](https://dart.dev/get-dart) (incluido con Flutter)
- Editor: **VS Code** con extensiones:
  - Flutter
  - Dart
  - Firebase (opcional, para integración)
- **Opcional**: [Antigravity/IDX](https://idx.dev/) (entorno web con Flutter preconfigurado)
- Cuenta de Google para [Firebase Console](https://console.firebase.google.com/)

### 2. Proyecto Firebase
- Crea un proyecto en Firebase Console (ej: `polivet-vet`)
- Registra tu app (Android / iOS / Web) según los dispositivos que vayas a usar.
- Habilita **Authentication** → método **Email/Password**.
- Crea una base de datos **Firestore** en modo de prueba (para desarrollo).
- Descarga el archivo de configuración:
  - Android: `google-services.json` (colocar en `android/app/`)
  - iOS: `GoogleService-Info.plist` (colocar en `ios/Runner/`)
  - Web: copiar el objeto `firebaseConfig` para `index.html` o usar `firebase_options.dart`.

### 3. Crear proyecto Flutter
```bash
flutter create polivet_veterinaria
cd polivet_veterinaria
```

---

## 🔧 Paso 1: Configurar Firebase en el proyecto Flutter

### 1.1 Agregar dependencias en `pubspec.yaml`
```yaml
dependencies:
  flutter:
    sdk: flutter
  firebase_core: ^3.0.0
  cloud_firestore: ^5.0.0
  firebase_auth: ^5.0.0
  provider: ^6.0.0
  # Opcionales para UX
  intl: ^0.18.0
  flutter_native_splash: ^2.3.0
```

Ejecuta:
```bash
flutter pub get
```

### 1.2 Inicializar Firebase en el proyecto
Si usas **VS Code**:

- Instala [Firebase CLI](https://firebase.google.com/docs/cli)
- Luego en terminal:
```bash
dart pub global activate flutterfire_cli
flutterfire configure
```
Selecciona el proyecto Firebase creado y las plataformas deseadas. Esto generará `lib/firebase_options.dart`.

**Alternativa manual**: Configura según la documentación oficial para cada plataforma.

### 1.3 Modificar `main.dart`
Envuelve la app con la inicialización de Firebase:

```dart
import 'package:firebase_core/firebase_core.dart';
import 'firebase_options.dart';

void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await Firebase.initializeApp(options: DefaultFirebaseOptions.currentPlatform);
  runApp(MyApp());
}
```

---

## 📁 Paso 2: Crear estructura de carpetas y archivos base

Dentro de `lib/`, crea las siguientes carpetas (puedes hacerlo desde el editor o terminal):

```
lib/
├── models/
├── services/
├── providers/
├── screens/
│   ├── auth/
│   ├── home/
│   ├── mascotas/
│   ├── propietarios/
│   └── consultas/
├── widgets/
├── utils/
└── routes/
```

---

## 🧱 Paso 3: Implementar modelos (capa de datos)

Crea cada modelo basado en la estructura Firestore (usa `String` como ID, no `int`). Ejemplo `lib/models/mascota.dart`:

```dart
import 'package:cloud_firestore/cloud_firestore.dart';

class Mascota {
  final String id;
  final String nombre;
  final String sexo;
  final DateTime? fechaNac;
  final String? edad;
  final String colorYSeñalesParticulares;
  final String propietarioId;
  final String razaId;

  Mascota({required this.id, required this.nombre, required this.sexo, this.fechaNac, this.edad, required this.colorYSeñalesParticulares, required this.propietarioId, required this.razaId});

  factory Mascota.fromFirestore(DocumentSnapshot doc) {
    final data = doc.data() as Map<String, dynamic>;
    return Mascota(
      id: doc.id,
      nombre: data['nombre'] ?? '',
      sexo: data['sexo'] ?? '',
      fechaNac: (data['fechaNac'] as Timestamp?)?.toDate(),
      edad: data['edad'],
      colorYSeñalesParticulares: data['colorYSeñalesParticulares'] ?? '',
      propietarioId: data['propietarioId'] ?? '',
      razaId: data['razaId'] ?? '',
    );
  }

  Map<String, dynamic> toMap() {
    return {
      'nombre': nombre,
      'sexo': sexo,
      'fechaNac': fechaNac != null ? Timestamp.fromDate(fechaNac!) : null,
      'edad': edad,
      'colorYSeñalesParticulares': colorYSeñalesParticulares,
      'propietarioId': propietarioId,
      'razaId': razaId,
    };
  }
}
```

**Crea también los modelos:** `propietario.dart`, `usuario.dart`, `consulta_medica.dart`, `receta_medica.dart`, `historia_clinica.dart`, etc. (puedes generarlos siguiendo el mismo patrón a partir del script SQL original).

Para ahorrar tiempo, crea un archivo `lib/models/models.dart` que exporte todos:

```dart
export 'mascota.dart';
export 'propietario.dart';
export 'usuario.dart';
...
```

---

## 🛠️ Paso 4: Servicios (lógica de negocio + Firestore)

### 4.1 Servicio base genérico (`lib/services/base_service.dart`)

```dart
import 'package:cloud_firestore/cloud_firestore.dart';

class BaseService<T> {
  final String collectionName;
  final T Function(DocumentSnapshot) fromFirestore;

  BaseService({required this.collectionName, required this.fromFirestore});

  CollectionReference get _collection => FirebaseFirestore.instance.collection(collectionName);

  Future<List<T>> getAll() async {
    final snapshot = await _collection.get();
    return snapshot.docs.map((doc) => fromFirestore(doc)).toList();
  }

  Future<T?> getById(String id) async {
    final doc = await _collection.doc(id).get();
    return doc.exists ? fromFirestore(doc) : null;
  }

  Future<void> create(String id, Map<String, dynamic> data) async {
    await _collection.doc(id).set(data);
  }

  Future<void> update(String id, Map<String, dynamic> data) async {
    await _collection.doc(id).update(data);
  }

  Future<void> delete(String id) async {
    await _collection.doc(id).delete();
  }

  Stream<List<T>> streamAll() {
    return _collection.snapshots().map((snapshot) => snapshot.docs.map((doc) => fromFirestore(doc)).toList());
  }
}
```

### 4.2 Servicio de autenticación (`lib/services/auth_service.dart`)

```dart
import 'package:firebase_auth/firebase_auth.dart';

class AuthService {
  final FirebaseAuth _auth = FirebaseAuth.instance;

  Stream<User?> get user => _auth.authStateChanges();

  Future<User?> signIn(String email, String password) async {
    final cred = await _auth.signInWithEmailAndPassword(email: email, password: password);
    return cred.user;
  }

  Future<User?> signUp(String email, String password) async {
    final cred = await _auth.createUserWithEmailAndPassword(email: email, password: password);
    return cred.user;
  }

  Future<void> signOut() async {
    await _auth.signOut();
  }
}
```

### 4.3 Servicio específico para Mascota (`lib/services/mascota_service.dart`)

```dart
import 'package:polivet_veterinaria/models/mascota.dart';
import 'base_service.dart';

class MascotaService extends BaseService<Mascota> {
  MascotaService() : super(
    collectionName: 'mascotas',
    fromFirestore: (doc) => Mascota.fromFirestore(doc),
  );

  Future<List<Mascota>> getByPropietario(String propietarioId) async {
    final snapshot = await _collection.where('propietarioId', isEqualTo: propietarioId).get();
    return snapshot.docs.map((doc) => Mascota.fromFirestore(doc)).toList();
  }
}
```

**Repite para `PropietarioService`, `ConsultaMedicaService`, etc.**

---

## 📦 Paso 5: Providers (gestión de estado con Provider)

### 5.1 Provider de autenticación (`lib/providers/auth_provider.dart`)

```dart
import 'package:flutter/material.dart';
import 'package:firebase_auth/firebase_auth.dart';
import 'package:polivet_veterinaria/services/auth_service.dart';

class AuthProvider extends ChangeNotifier {
  final AuthService _auth = AuthService();
  User? _user;
  bool _loading = false;

  User? get user => _user;
  bool get loading => _loading;

  AuthProvider() {
    _auth.user.listen((event) {
      _user = event;
      notifyListeners();
    });
  }

  Future<void> login(String email, String password, BuildContext context) async {
    _loading = true;
    notifyListeners();
    try {
      await _auth.signIn(email, password);
      Navigator.pushReplacementNamed(context, '/home');
    } catch (e) {
      ScaffoldMessenger.of(context).showSnackBar(SnackBar(content: Text(e.toString())));
    } finally {
      _loading = false;
      notifyListeners();
    }
  }

  Future<void> register(String email, String password, BuildContext context) async {
    _loading = true;
    notifyListeners();
    try {
      await _auth.signUp(email, password);
      Navigator.pushReplacementNamed(context, '/home');
    } catch (e) {
      ScaffoldMessenger.of(context).showSnackBar(SnackBar(content: Text(e.toString())));
    } finally {
      _loading = false;
      notifyListeners();
    }
  }

  Future<void> logout() async {
    await _auth.signOut();
  }
}
```

### 5.2 Provider para Mascota (`lib/providers/mascota_provider.dart`)

```dart
import 'package:flutter/material.dart';
import 'package:polivet_veterinaria/models/mascota.dart';
import 'package:polivet_veterinaria/services/mascota_service.dart';

class MascotaProvider extends ChangeNotifier {
  final MascotaService _service = MascotaService();
  List<Mascota> _mascotas = [];
  bool _loading = false;
  String? _error;

  List<Mascota> get mascotas => _mascotas;
  bool get loading => _loading;
  String? get error => _error;

  Future<void> loadMascotas() async {
    _loading = true;
    notifyListeners();
    try {
      _mascotas = await _service.getAll();
    } catch (e) {
      _error = e.toString();
    } finally {
      _loading = false;
      notifyListeners();
    }
  }

  Future<void> addMascota(Mascota mascota) async {
    await _service.create(mascota.id, mascota.toMap());
    await loadMascotas();
  }

  Future<void> updateMascota(Mascota mascota) async {
    await _service.update(mascota.id, mascota.toMap());
    await loadMascotas();
  }

  Future<void> deleteMascota(String id) async {
    await _service.delete(id);
    await loadMascotas();
  }
}
```

**Crea análogos para `PropietarioProvider`, `ConsultaProvider`.**

---

## 🖼️ Paso 6: Crear las pantallas (UI)

### 6.1 Pantalla de login (`lib/screens/auth/login_screen.dart`)

```dart
import 'package:flutter/material.dart';
import 'package:provider/provider.dart';
import 'package:polivet_veterinaria/providers/auth_provider.dart';
import 'package:polivet_veterinaria/widgets/custom_button.dart';
import 'package:polivet_veterinaria/widgets/custom_text_field.dart';

class LoginScreen extends StatelessWidget {
  final TextEditingController emailController = TextEditingController();
  final TextEditingController passwordController = TextEditingController();

  @override
  Widget build(BuildContext context) {
    final authProvider = Provider.of<AuthProvider>(context);
    return Scaffold(
      body: Padding(
        padding: const EdgeInsets.all(16.0),
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            CustomTextField(controller: emailController, label: 'Email', validator: (_) => null),
            CustomTextField(controller: passwordController, label: 'Contraseña', obscureText: true),
            SizedBox(height: 20),
            authProvider.loading ? CircularProgressIndicator() : CustomButton(
              text: 'Iniciar sesión',
              onPressed: () => authProvider.login(emailController.text, passwordController.text, context),
            ),
            TextButton(
              onPressed: () => Navigator.pushNamed(context, '/register'),
              child: Text('¿No tienes cuenta? Regístrate'),
            )
          ],
        ),
      ),
    );
  }
}
```

### 6.2 Pantalla de home (`lib/screens/home_screen.dart`)

```dart
import 'package:flutter/material.dart';
import 'package:provider/provider.dart';
import 'package:polivet_veterinaria/providers/auth_provider.dart';

class HomeScreen extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text('Polivet Veterinaria'), actions: [
        IconButton(
          icon: Icon(Icons.exit_to_app),
          onPressed: () => Provider.of<AuthProvider>(context, listen: false).logout(),
        )
      ]),
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            ElevatedButton(
              onPressed: () => Navigator.pushNamed(context, '/mascotas'),
              child: Text('Mascotas'),
            ),
            ElevatedButton(
              onPressed: () => Navigator.pushNamed(context, '/propietarios'),
              child: Text('Propietarios'),
            ),
            ElevatedButton(
              onPressed: () => Navigator.pushNamed(context, '/consultas'),
              child: Text('Consultas médicas'),
            ),
          ],
        ),
      ),
    );
  }
}
```

### 6.3 Listado de mascotas (`lib/screens/mascotas/mascota_list_screen.dart`) – ver código en respuesta anterior.  
**Resumen:** usa `Consumer<MascotaProvider>` para mostrar lista, FAB para agregar, opciones de editar/eliminar.

### 6.4 Formulario de mascota (`lib/screens/mascotas/mascota_form_screen.dart`) – similar al ejemplo.

### 6.5 Pantallas similares para `Propietario` y `ConsultaMedica`.

---

## 🧩 Paso 7: Widgets reutilizables

Crear en `lib/widgets/`:

- `custom_text_field.dart`: TextFormField personalizado.
- `custom_button.dart`: ElevatedButton con estilos.
- `loading_indicator.dart`: CircularProgressIndicator centrado.
- `confirmation_dialog.dart`: muestra AlertDialog y retorna bool.

---

## 🗺️ Paso 8: Configurar rutas

En `lib/routes/app_routes.dart` define las rutas como constantes:

```dart
class AppRoutes {
  static const String login = '/';
  static const String register = '/register';
  static const String home = '/home';
  static const String mascotas = '/mascotas';
  static const String mascotaForm = '/mascotaForm';
  static const String propietarios = '/propietarios';
  static const String consultas = '/consultas';
}
```

En `lib/routes/app_pages.dart` crea un `MaterialPageRoute` para cada ruta (o usa un `Map<String, WidgetBuilder>`).  
Modifica `main.dart` para usar `onGenerateRoute` o `routes:`.

---

## 🧪 Paso 9: Probar localmente

### 9.1 Asegurar reglas de Firestore (modo pruebas)
Ve a Firebase Console → Firestore → Reglas:
```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /{document=**} {
      allow read, write: if true;
    }
  }
}
```
**Solo para desarrollo**. En producción debes definir reglas más estrictas.

### 9.2 Ejecutar la app
```bash
flutter run
```

Prueba:
- Registro/login.
- Crear una mascota.
- Editar / eliminar.
- Navegación entre pantallas.

---

## 🚀 Paso 10: Siguientes pasos y mejoras

- **Agregar otras entidades**: Receta médica, Historia clínica, Constantes fisiológicas.
- **Validaciones**: En formularios (email, teléfono, fechas).
- **Relaciones**: Mostrar el nombre del propietario en lugar del ID en la lista de mascotas (usar `StreamBuilder` o FutureBuilder).
- **Carga de imágenes**: Subir foto de mascota a Firebase Storage.
- **Búsqueda y filtros**.
- **Autenticación con Google** (opcional).
- **Reglas de seguridad** avanzadas para Firestore.
- **Despliegue** a Play Store / App Store.

---

## 📦 Resumen de archivos a crear (checklist)

- [ ] `lib/firebase_options.dart` (generado automáticamente)
- [ ] `lib/models/mascota.dart`, `propietario.dart`, `usuario.dart`, ...
- [ ] `lib/services/base_service.dart`, `auth_service.dart`, `mascota_service.dart`, ...
- [ ] `lib/providers/auth_provider.dart`, `mascota_provider.dart`, ...
- [ ] `lib/screens/auth/login_screen.dart`, `register_screen.dart`
- [ ] `lib/screens/home_screen.dart`
- [ ] `lib/screens/mascotas/mascota_list_screen.dart`, `mascota_form_screen.dart`
- [ ] `lib/screens/propietarios/` (list, form)
- [ ] `lib/screens/consultas/` (list, form)
- [ ] `lib/widgets/` (custom_text_field.dart, custom_button.dart, ...)
- [ ] `lib/utils/validators.dart`, `date_time_helper.dart`
- [ ] `lib/routes/app_routes.dart`, `app_pages.dart`
- [ ] `lib/main.dart` configurado con MultiProvider y rutas

Con este plan tendrás una aplicación funcional, con autenticación, CRUD completo y una arquitectura mantenible para seguir creciendo. ¡Éxito con tu proyecto Polivet!
```
