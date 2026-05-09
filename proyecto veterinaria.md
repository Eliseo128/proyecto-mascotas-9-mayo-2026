He generado una estructura completa de carpetas y archivos para tu proyecto Flutter con Firebase, organizada por entidades y con operaciones CRUD.  

A continuación se muestra el árbol de archivos y el contenido clave.  

## 📁 Estructura de carpetas (dentro de `lib/`)

```
lib/
├── main.dart
├── models/
│   ├── mascota.dart
│   ├── propietario.dart
│   ├── usuario.dart
│   ├── consulta_medica.dart
│   ├── ... (todos los modelos adaptados a Firestore con ID String)
│   └── models.dart (exporta todos)
├── services/
│   ├── auth_service.dart
│   ├── base_service.dart (CRUD genérico)
│   ├── mascota_service.dart
│   ├── propietario_service.dart
│   ├── usuario_service.dart
│   └── consulta_service.dart
├── providers/
│   ├── auth_provider.dart
│   ├── mascota_provider.dart
│   ├── propietario_provider.dart
│   └── consulta_provider.dart
├── screens/
│   ├── auth/
│   │   ├── login_screen.dart
│   │   └── register_screen.dart
│   ├── home_screen.dart
│   ├── mascotas/
│   │   ├── mascota_list_screen.dart
│   │   ├── mascota_form_screen.dart
│   │   └── mascota_detail_screen.dart
│   ├── propietarios/
│   │   ├── propietario_list_screen.dart
│   │   ├── propietario_form_screen.dart
│   │   └── propietario_detail_screen.dart
│   └── consultas/
│       ├── consulta_list_screen.dart
│       └── consulta_form_screen.dart
├── widgets/
│   ├── custom_text_field.dart
│   ├── custom_button.dart
│   ├── loading_indicator.dart
│   └── confirmation_dialog.dart
├── utils/
│   ├── constants.dart
│   ├── validators.dart
│   └── date_time_helper.dart
└── routes/
    ├── app_routes.dart
    └── app_pages.dart
```

---

## 1. Modelos adaptados a Firestore (ID como `String`)

Ejemplo para `Mascota` (`lib/models/mascota.dart`):

```dart
import 'package:cloud_firestore/cloud_firestore.dart';

class Mascota {
  final String id; // Firestore document ID
  final String nombre;
  final String sexo;
  final DateTime? fechaNac;
  final String? edad;
  final String colorYSeñalesParticulares;
  final String propietarioId; // referencia al documento Propietario
  final String razaId;        // referencia a Raza

  Mascota({
    required this.id,
    required this.nombre,
    required this.sexo,
    this.fechaNac,
    this.edad,
    required this.colorYSeñalesParticulares,
    required this.propietarioId,
    required this.razaId,
  });

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

Similar para `Propietario` (`lib/models/propietario.dart`):

```dart
class Propietario {
  final String id;
  final String nombrePropietario;
  final String apellidoPropietario;
  final String direccion;
  final String telefono;
  final String ciudad;
  final String correo;

  Propietario({...});

  factory Propietario.fromFirestore(DocumentSnapshot doc) {...}
  Map<String, dynamic> toMap() {...}
}
```

---

## 2. Servicio base genérico (`lib/services/base_service.dart`)

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
    if (!doc.exists) return null;
    return fromFirestore(doc);
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
    return _collection.snapshots().map((snapshot) =>
        snapshot.docs.map((doc) => fromFirestore(doc)).toList());
  }
}
```

## 3. Servicio específico para Mascota (`lib/services/mascota_service.dart`)

```dart
import 'package:veterinaria/models/mascota.dart';
import 'base_service.dart';

class MascotaService extends BaseService<Mascota> {
  MascotaService()
      : super(
          collectionName: 'mascotas',
          fromFirestore: (doc) => Mascota.fromFirestore(doc),
        );

  // Métodos adicionales específicos, por ejemplo:
  Future<List<Mascota>> getByPropietario(String propietarioId) async {
    final snapshot = await _collection
        .where('propietarioId', isEqualTo: propietarioId)
        .get();
    return snapshot.docs.map((doc) => Mascota.fromFirestore(doc)).toList();
  }
}
```

---

## 4. Provider para Mascota (`lib/providers/mascota_provider.dart`)

```dart
import 'package:flutter/material.dart';
import 'package:veterinaria/models/mascota.dart';
import 'package:veterinaria/services/mascota_service.dart';

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
    _error = null;
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
    try {
      await _service.create(mascota.id, mascota.toMap());
      await loadMascotas();
    } catch (e) {
      _error = e.toString();
      notifyListeners();
    }
  }

  Future<void> updateMascota(Mascota mascota) async {
    try {
      await _service.update(mascota.id, mascota.toMap());
      await loadMascotas();
    } catch (e) {
      _error = e.toString();
      notifyListeners();
    }
  }

  Future<void> deleteMascota(String id) async {
    try {
      await _service.delete(id);
      await loadMascotas();
    } catch (e) {
      _error = e.toString();
      notifyListeners();
    }
  }
}
```

---

## 5. Pantalla de listado de mascotas (`lib/screens/mascotas/mascota_list_screen.dart`)

```dart
import 'package:flutter/material.dart';
import 'package:provider/provider.dart';
import 'package:veterinaria/providers/mascota_provider.dart';
import 'package:veterinaria/screens/mascotas/mascota_form_screen.dart';
import 'package:veterinaria/widgets/confirmation_dialog.dart';
import 'package:veterinaria/widgets/loading_indicator.dart';

class MascotaListScreen extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    final provider = Provider.of<MascotaProvider>(context);
    return Scaffold(
      appBar: AppBar(title: Text('Mis Mascotas')),
      body: provider.loading
          ? LoadingIndicator()
          : provider.error != null
              ? Center(child: Text('Error: ${provider.error}'))
              : ListView.builder(
                  itemCount: provider.mascotas.length,
                  itemBuilder: (_, i) {
                    final m = provider.mascotas[i];
                    return ListTile(
                      title: Text(m.nombre),
                      subtitle: Text('Dueño: ${m.propietarioId}'),
                      trailing: IconButton(
                        icon: Icon(Icons.delete),
                        onPressed: () async {
                          final confirm = await showConfirmationDialog(context);
                          if (confirm) provider.deleteMascota(m.id);
                        },
                      ),
                      onTap: () => Navigator.push(
                        context,
                        MaterialPageRoute(
                          builder: (_) => MascotaFormScreen(mascota: m),
                        ),
                      ),
                    );
                  },
                ),
      floatingActionButton: FloatingActionButton(
        child: Icon(Icons.add),
        onPressed: () => Navigator.push(
          context,
          MaterialPageRoute(builder: (_) => MascotaFormScreen()),
        ),
      ),
    );
  }
}
```

---

## 6. Pantalla de formulario de mascota (crear/editar)

```dart
class MascotaFormScreen extends StatefulWidget {
  final Mascota? mascota;
  const MascotaFormScreen({Key? key, this.mascota}) : super(key: key);

  @override
  _MascotaFormScreenState createState() => _MascotaFormScreenState();
}

class _MascotaFormScreenState extends State<MascotaFormScreen> {
  final _formKey = GlobalKey<FormState>();
  late TextEditingController _nombreController;
  late TextEditingController _sexoController;
  late TextEditingController _colorController;
  late TextEditingController _propietarioIdController;
  late TextEditingController _razaIdController;

  @override
  void initState() {
    super.initState();
    final m = widget.mascota;
    _nombreController = TextEditingController(text: m?.nombre ?? '');
    _sexoController = TextEditingController(text: m?.sexo ?? '');
    _colorController = TextEditingController(text: m?.colorYSeñalesParticulares ?? '');
    _propietarioIdController = TextEditingController(text: m?.propietarioId ?? '');
    _razaIdController = TextEditingController(text: m?.razaId ?? '');
  }

  @override
  void dispose() {
    _nombreController.dispose();
    _sexoController.dispose();
    _colorController.dispose();
    _propietarioIdController.dispose();
    _razaIdController.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    final provider = Provider.of<MascotaProvider>(context, listen: false);
    return Scaffold(
      appBar: AppBar(title: Text(widget.mascota == null ? 'Nueva Mascota' : 'Editar Mascota')),
      body: Padding(
        padding: const EdgeInsets.all(16.0),
        child: Form(
          key: _formKey,
          child: Column(
            children: [
              CustomTextField(controller: _nombreController, label: 'Nombre', validator: (v) => v!.isEmpty ? 'Requerido' : null),
              CustomTextField(controller: _sexoController, label: 'Sexo'),
              CustomTextField(controller: _colorController, label: 'Color/señales'),
              CustomTextField(controller: _propietarioIdController, label: 'ID Propietario'),
              CustomTextField(controller: _razaIdController, label: 'ID Raza'),
              SizedBox(height: 20),
              CustomButton(
                text: 'Guardar',
                onPressed: () async {
                  if (_formKey.currentState!.validate()) {
                    final mascota = Mascota(
                      id: widget.mascota?.id ?? DateTime.now().millisecondsSinceEpoch.toString(),
                      nombre: _nombreController.text,
                      sexo: _sexoController.text,
                      colorYSeñalesParticulares: _colorController.text,
                      propietarioId: _propietarioIdController.text,
                      razaId: _razaIdController.text,
                    );
                    if (widget.mascota == null) {
                      await provider.addMascota(mascota);
                    } else {
                      await provider.updateMascota(mascota);
                    }
                    Navigator.pop(context);
                  }
                },
              ),
            ],
          ),
        ),
      ),
    );
  }
}
```

---

## 7. Servicio de autenticación (`lib/services/auth_service.dart`)

```dart
import 'package:firebase_auth/firebase_auth.dart';

class AuthService {
  final FirebaseAuth _auth = FirebaseAuth.instance;

  Stream<User?> get user => _auth.authStateChanges();

  Future<User?> signInWithEmail(String email, String password) async {
    try {
      final cred = await _auth.signInWithEmailAndPassword(email: email, password: password);
      return cred.user;
    } catch (e) {
      throw Exception('Error login: $e');
    }
  }

  Future<User?> registerWithEmail(String email, String password) async {
    try {
      final cred = await _auth.createUserWithEmailAndPassword(email: email, password: password);
      return cred.user;
    } catch (e) {
      throw Exception('Error registro: $e');
    }
  }

  Future<void> signOut() async {
    await _auth.signOut();
  }
}
```

---

## 8. Provider de autenticación (`lib/providers/auth_provider.dart`)

```dart
import 'package:flutter/material.dart';
import 'package:firebase_auth/firebase_auth.dart';
import 'package:veterinaria/services/auth_service.dart';

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
      await _auth.signInWithEmail(email, password);
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
      await _auth.registerWithEmail(email, password);
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

---

## 9. `main.dart` con MultiProvider y rutas

```dart
import 'package:firebase_core/firebase_core.dart';
import 'package:flutter/material.dart';
import 'package:provider/provider.dart';
import 'package:veterinaria/firebase_options.dart';
import 'package:veterinaria/providers/auth_provider.dart';
import 'package:veterinaria/providers/mascota_provider.dart';
import 'package:veterinaria/providers/propietario_provider.dart';
import 'package:veterinaria/screens/auth/login_screen.dart';
import 'package:veterinaria/screens/home_screen.dart';
import 'package:veterinaria/screens/mascotas/mascota_list_screen.dart';

void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await Firebase.initializeApp(options: DefaultFirebaseOptions.currentPlatform);
  runApp(MyApp());
}

class MyApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MultiProvider(
      providers: [
        ChangeNotifierProvider(create: (_) => AuthProvider()),
        ChangeNotifierProvider(create: (_) => MascotaProvider()),
        ChangeNotifierProvider(create: (_) => PropietarioProvider()),
        // ... otros providers
      ],
      child: MaterialApp(
        title: 'Veterinaria Polivet',
        initialRoute: '/',
        routes: {
          '/': (context) => StreamBuilder<User?>(
                stream: Provider.of<AuthProvider>(context).user,
                builder: (_, snapshot) {
                  if (snapshot.connectionState == ConnectionState.active) {
                    final user = snapshot.data;
                    if (user == null) return LoginScreen();
                    return HomeScreen();
                  }
                  return Scaffold(body: Center(child: CircularProgressIndicator()));
                },
              ),
          '/home': (context) => HomeScreen(),
          '/mascotas': (context) => MascotaListScreen(),
          // demás rutas
        },
      ),
    );
  }
}
```

---

## 10. Widgets reutilizables (ejemplo)

`lib/widgets/custom_text_field.dart`:

```dart
import 'package:flutter/material.dart';

class CustomTextField extends StatelessWidget {
  final TextEditingController controller;
  final String label;
  final String? Function(String?)? validator;
  const CustomTextField({Key? key, required this.controller, required this.label, this.validator}) : super(key: key);

  @override
  Widget build(BuildContext context) {
    return TextFormField(
      controller: controller,
      decoration: InputDecoration(labelText: label),
      validator: validator,
    );
  }
}
```

`lib/widgets/confirmation_dialog.dart`:

```dart
import 'package:flutter/material.dart';

Future<bool> showConfirmationDialog(BuildContext context) async {
  return await showDialog(
        context: context,
        builder: (_) => AlertDialog(
          title: Text('Confirmar'),
          content: Text('¿Está seguro?'),
          actions: [
            TextButton(onPressed: () => Navigator.pop(context, false), child: Text('Cancelar')),
            TextButton(onPressed: () => Navigator.pop(context, true), child: Text('Aceptar')),
          ],
        ),
      ) ??
      false;
}
```

---

## ✅ Resumen de lo que obtienes

- **Estructura clara** por capas (modelos, servicios, providers, vistas).
- **CRUD completo** para cada entidad reutilizando el servicio base.
- **Autenticación** con Firebase Auth integrada.
- **Gestión de estado** con `Provider`.
- **Widgets reutilizables** para acelerar el desarrollo.
- **Rutas definidas** y navegación.

Para replicar para otras entidades (`Propietario`, `ConsultaMedica`, etc.) solo debes:

1. Crear su modelo en `models/`.
2. Crear su servicio extendiendo `BaseService`.
3. Crear su provider.
4. Crear sus pantallas de listado, formulario y detalle (basadas en las de `Mascota`).

Esta estructura es escalable y mantenible para un proyecto real de clínica veterinaria.
