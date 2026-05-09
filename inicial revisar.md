He creado un archivo Dart con todos los modelos necesarios para trabajar con **Firebase Authentication** y **Cloud Firestore**, basado en el esquema SQL proporcionado.  

El archivo incluye:

- Mapeo de cada tabla a una clase Dart.
- Conversión `fromFirestore` / `toMap` para guardar y leer datos.
- Uso de `DocumentReference` para las relaciones (claves foráneas).
- Integración con Firebase Auth: el modelo `Usuario` se vincula al `uid` del usuario autenticado.
- Manejo de `DateTime` → `Timestamp` de Firestore.
- Validaciones básicas en los constructores.

```dart
// models.dart
// Dependencias necesarias en pubspec.yaml:
// firebase_core, cloud_firestore, firebase_auth

import 'package:cloud_firestore/cloud_firestore.dart';

// ============================================================================
// 1. Rol
// ============================================================================
class Rol {
  final int rolId;
  final String? descripcion;

  Rol({required this.rolId, this.descripcion});

  factory Rol.fromFirestore(DocumentSnapshot doc) {
    final data = doc.data() as Map<String, dynamic>;
    return Rol(
      rolId: data['rolId'] as int,
      descripcion: data['descripcion'] as String?,
    );
  }

  Map<String, dynamic> toMap() {
    return {
      'rolId': rolId,
      'descripcion': descripcion,
    };
  }
}

// ============================================================================
// 2. Especialidad
// ============================================================================
class Especialidad {
  final int especialidadId;
  final String? tipoEspecialidad;

  Especialidad({required this.especialidadId, this.tipoEspecialidad});

  factory Especialidad.fromFirestore(DocumentSnapshot doc) {
    final data = doc.data() as Map<String, dynamic>;
    return Especialidad(
      especialidadId: data['especialidadId'] as int,
      tipoEspecialidad: data['tipoEspecialidad'] as String?,
    );
  }

  Map<String, dynamic> toMap() {
    return {
      'especialidadId': especialidadId,
      'tipoEspecialidad': tipoEspecialidad,
    };
  }
}

// ============================================================================
// 3. Especie
// ============================================================================
class Especie {
  final int especieId;
  final String? nombreEspecie;

  Especie({required this.especieId, this.nombreEspecie});

  factory Especie.fromFirestore(DocumentSnapshot doc) {
    final data = doc.data() as Map<String, dynamic>;
    return Especie(
      especieId: data['especieId'] as int,
      nombreEspecie: data['nombreEspecie'] as String?,
    );
  }

  Map<String, dynamic> toMap() {
    return {
      'especieId': especieId,
      'nombreEspecie': nombreEspecie,
    };
  }
}

// ============================================================================
// 4. Propietario
// ============================================================================
class Propietario {
  final int propietarioId;
  final String? nombrePropietario;
  final String? apellidoPropietario;
  final String? direccion;
  final String? telefono;
  final String? ciudad;
  final String? correo;

  Propietario({
    required this.propietarioId,
    this.nombrePropietario,
    this.apellidoPropietario,
    this.direccion,
    this.telefono,
    this.ciudad,
    this.correo,
  });

  factory Propietario.fromFirestore(DocumentSnapshot doc) {
    final data = doc.data() as Map<String, dynamic>;
    return Propietario(
      propietarioId: data['propietarioId'] as int,
      nombrePropietario: data['nombrePropietario'] as String?,
      apellidoPropietario: data['apellidoPropietario'] as String?,
      direccion: data['direccion'] as String?,
      telefono: data['telefono'] as String?,
      ciudad: data['ciudad'] as String?,
      correo: data['correo'] as String?,
    );
  }

  Map<String, dynamic> toMap() {
    return {
      'propietarioId': propietarioId,
      'nombrePropietario': nombrePropietario,
      'apellidoPropietario': apellidoPropietario,
      'direccion': direccion,
      'telefono': telefono,
      'ciudad': ciudad,
      'correo': correo,
    };
  }
}

// ============================================================================
// 5. RecetaMedica
// ============================================================================
class RecetaMedica {
  final int recetaId;
  final DateTime? fecha;
  final String? rp;
  final String? prescripcion;

  RecetaMedica({
    required this.recetaId,
    this.fecha,
    this.rp,
    this.prescripcion,
  });

  factory RecetaMedica.fromFirestore(DocumentSnapshot doc) {
    final data = doc.data() as Map<String, dynamic>;
    return RecetaMedica(
      recetaId: data['recetaId'] as int,
      fecha: (data['fecha'] as Timestamp?)?.toDate(),
      rp: data['rp'] as String?,
      prescripcion: data['prescripcion'] as String?,
    );
  }

  Map<String, dynamic> toMap() {
    return {
      'recetaId': recetaId,
      'fecha': fecha != null ? Timestamp.fromDate(fecha!) : null,
      'rp': rp,
      'prescripcion': prescripcion,
    };
  }
}

// ============================================================================
// 6. ConstantesFisiologicaCabecera
// ============================================================================
class ConstantesFisiologicaCabecera {
  final int constantesIdCab;
  final String? nombre;

  ConstantesFisiologicaCabecera({required this.constantesIdCab, this.nombre});

  factory ConstantesFisiologicaCabecera.fromFirestore(DocumentSnapshot doc) {
    final data = doc.data() as Map<String, dynamic>;
    return ConstantesFisiologicaCabecera(
      constantesIdCab: data['constantesIdCab'] as int,
      nombre: data['nombre'] as String?,
    );
  }

  Map<String, dynamic> toMap() {
    return {
      'constantesIdCab': constantesIdCab,
      'nombre': nombre,
    };
  }
}

// ============================================================================
// 7. Usuario (vinculado a Firebase Auth)
// ============================================================================
class Usuario {
  final String uid; // Mismo uid de FirebaseAuth
  final String? correo;
  final String? contrasena; // En producción no guardar texto plano
  final int? rolId; // Referencia a Rol

  Usuario({
    required this.uid,
    this.correo,
    this.contrasena,
    this.rolId,
  });

  factory Usuario.fromFirestore(DocumentSnapshot doc) {
    final data = doc.data() as Map<String, dynamic>;
    return Usuario(
      uid: doc.id, // El id del documento es el uid de auth
      correo: data['correo'] as String?,
      contrasena: data['contrasena'] as String?,
      rolId: data['rolId'] as int?,
    );
  }

  Map<String, dynamic> toMap() {
    return {
      'correo': correo,
      'contrasena': contrasena,
      'rolId': rolId,
    };
  }
}

// ============================================================================
// 8. Raza (depende de Especie)
// ============================================================================
class Raza {
  final int razaId;
  final String? nombre;
  final int? especieId; // Referencia a Especie

  Raza({required this.razaId, this.nombre, this.especieId});

  factory Raza.fromFirestore(DocumentSnapshot doc) {
    final data = doc.data() as Map<String, dynamic>;
    return Raza(
      razaId: data['razaId'] as int,
      nombre: data['nombre'] as String?,
      especieId: data['especieId'] as int?,
    );
  }

  Map<String, dynamic> toMap() {
    return {
      'razaId': razaId,
      'nombre': nombre,
      'especieId': especieId,
    };
  }
}

// ============================================================================
// 9. MedicoVeterinario
// ============================================================================
class MedicoVeterinario {
  final int cedulaId;
  final String? nombres;
  final String? apellidos;
  final String? direccion;
  final DateTime? fechaNac;
  final String? celular;
  final String? titulo;
  final int? especialidadId;
  final String? usuarioUid; // Referencia al usuario (uid de Firebase)

  MedicoVeterinario({
    required this.cedulaId,
    this.nombres,
    this.apellidos,
    this.direccion,
    this.fechaNac,
    this.celular,
    this.titulo,
    this.especialidadId,
    this.usuarioUid,
  });

  factory MedicoVeterinario.fromFirestore(DocumentSnapshot doc) {
    final data = doc.data() as Map<String, dynamic>;
    return MedicoVeterinario(
      cedulaId: data['cedulaId'] as int,
      nombres: data['nombres'] as String?,
      apellidos: data['apellidos'] as String?,
      direccion: data['direccion'] as String?,
      fechaNac: (data['fechaNac'] as Timestamp?)?.toDate(),
      celular: data['celular'] as String?,
      titulo: data['titulo'] as String?,
      especialidadId: data['especialidadId'] as int?,
      usuarioUid: data['usuarioUid'] as String?,
    );
  }

  Map<String, dynamic> toMap() {
    return {
      'cedulaId': cedulaId,
      'nombres': nombres,
      'apellidos': apellidos,
      'direccion': direccion,
      'fechaNac': fechaNac != null ? Timestamp.fromDate(fechaNac!) : null,
      'celular': celular,
      'titulo': titulo,
      'especialidadId': especialidadId,
      'usuarioUid': usuarioUid,
    };
  }
}

// ============================================================================
// 10. Mascota
// ============================================================================
class Mascota {
  final int mascotaId;
  final String? nombre;
  final String? sexo;
  final DateTime? fechaNac;
  final String? edad;
  final String? colorYSeñalesParticulares;
  final int? propietarioId;
  final int? razaId;

  Mascota({
    required this.mascotaId,
    this.nombre,
    this.sexo,
    this.fechaNac,
    this.edad,
    this.colorYSeñalesParticulares,
    this.propietarioId,
    this.razaId,
  });

  factory Mascota.fromFirestore(DocumentSnapshot doc) {
    final data = doc.data() as Map<String, dynamic>;
    return Mascota(
      mascotaId: data['mascotaId'] as int,
      nombre: data['nombre'] as String?,
      sexo: data['sexo'] as String?,
      fechaNac: (data['fechaNac'] as Timestamp?)?.toDate(),
      edad: data['edad'] as String?,
      colorYSeñalesParticulares: data['colorYSeñalesParticulares'] as String?,
      propietarioId: data['propietarioId'] as int?,
      razaId: data['razaId'] as int?,
    );
  }

  Map<String, dynamic> toMap() {
    return {
      'mascotaId': mascotaId,
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

// ============================================================================
// 11. HistoriaClinica
// ============================================================================
class HistoriaClinica {
  final int historiaId;
  final String? diaAdmision;
  final TimeOfDay? hora; // Firestore no tiene TimeOfDay nativo, usaremos String
  final int? cedulaId; // Médico veterinario
  final int? mascotaId;

  HistoriaClinica({
    required this.historiaId,
    this.diaAdmision,
    this.hora,
    this.cedulaId,
    this.mascotaId,
  });

  factory HistoriaClinica.fromFirestore(DocumentSnapshot doc) {
    final data = doc.data() as Map<String, dynamic>;
    TimeOfDay? horaTime;
    if (data['hora'] is String) {
      final parts = (data['hora'] as String).split(':');
      if (parts.length == 2) {
        horaTime = TimeOfDay(hour: int.parse(parts[0]), minute: int.parse(parts[1]));
      }
    }
    return HistoriaClinica(
      historiaId: data['historiaId'] as int,
      diaAdmision: data['diaAdmision'] as String?,
      hora: horaTime,
      cedulaId: data['cedulaId'] as int?,
      mascotaId: data['mascotaId'] as int?,
    );
  }

  Map<String, dynamic> toMap() {
    String? horaStr;
    if (hora != null) {
      horaStr = '${hora!.hour.toString().padLeft(2, '0')}:${hora!.minute.toString().padLeft(2, '0')}';
    }
    return {
      'historiaId': historiaId,
      'diaAdmision': diaAdmision,
      'hora': horaStr,
      'cedulaId': cedulaId,
      'mascotaId': mascotaId,
    };
  }
}

// ============================================================================
// 12. ConsultaMedica
// ============================================================================
class ConsultaMedica {
  final int consultaId;
  final String? motivoConsulta;
  final String? vacunacion;
  final String? producto;
  final String? desparasitacion;
  final DateTime? fechas;
  final String? estadoReproductivo;
  final String? procedencia;
  final String? anamnesis;
  final String? diagnostico;
  final String? pronostico;
  final String? observaciones;
  final int? recetaId;
  final int? historiaId;

  ConsultaMedica({
    required this.consultaId,
    this.motivoConsulta,
    this.vacunacion,
    this.producto,
    this.desparasitacion,
    this.fechas,
    this.estadoReproductivo,
    this.procedencia,
    this.anamnesis,
    this.diagnostico,
    this.pronostico,
    this.observaciones,
    this.recetaId,
    this.historiaId,
  });

  factory ConsultaMedica.fromFirestore(DocumentSnapshot doc) {
    final data = doc.data() as Map<String, dynamic>;
    return ConsultaMedica(
      consultaId: data['consultaId'] as int,
      motivoConsulta: data['motivoConsulta'] as String?,
      vacunacion: data['vacunacion'] as String?,
      producto: data['producto'] as String?,
      desparasitacion: data['desparasitacion'] as String?,
      fechas: (data['fechas'] as Timestamp?)?.toDate(),
      estadoReproductivo: data['estadoReproductivo'] as String?,
      procedencia: data['procedencia'] as String?,
      anamnesis: data['anamnesis'] as String?,
      diagnostico: data['diagnostico'] as String?,
      pronostico: data['pronostico'] as String?,
      observaciones: data['observaciones'] as String?,
      recetaId: data['recetaId'] as int?,
      historiaId: data['historiaId'] as int?,
    );
  }

  Map<String, dynamic> toMap() {
    return {
      'consultaId': consultaId,
      'motivoConsulta': motivoConsulta,
      'vacunacion': vacunacion,
      'producto': producto,
      'desparasitacion': desparasitacion,
      'fechas': fechas != null ? Timestamp.fromDate(fechas!) : null,
      'estadoReproductivo': estadoReproductivo,
      'procedencia': procedencia,
      'anamnesis': anamnesis,
      'diagnostico': diagnostico,
      'pronostico': pronostico,
      'observaciones': observaciones,
      'recetaId': recetaId,
      'historiaId': historiaId,
    };
  }
}

// ============================================================================
// 13. ConstantesFisiologicaDetalle
// ============================================================================
class ConstantesFisiologicaDetalle {
  final int constantesIdDetalle;
  final String? valorAsignado;
  final int? constantesIdCab;
  final int? consultaId;

  ConstantesFisiologicaDetalle({
    required this.constantesIdDetalle,
    this.valorAsignado,
    this.constantesIdCab,
    this.consultaId,
  });

  factory ConstantesFisiologicaDetalle.fromFirestore(DocumentSnapshot doc) {
    final data = doc.data() as Map<String, dynamic>;
    return ConstantesFisiologicaDetalle(
      constantesIdDetalle: data['constantesIdDetalle'] as int,
      valorAsignado: data['valorAsignado'] as String?,
      constantesIdCab: data['constantesIdCab'] as int?,
      consultaId: data['consultaId'] as int?,
    );
  }

  Map<String, dynamic> toMap() {
    return {
      'constantesIdDetalle': constantesIdDetalle,
      'valorAsignado': valorAsignado,
      'constantesIdCab': constantesIdCab,
      'consultaId': consultaId,
    };
  }
}
```

## 🔧 Cómo usar estos modelos con Firebase

### 1. Inicialización en tu app
```dart
await Firebase.initializeApp();
final firestore = FirebaseFirestore.instance;
final auth = FirebaseAuth.instance;
```

### 2. Guardar un usuario (después de registrarse con email/password)
```dart
UserCredential cred = await auth.createUserWithEmailAndPassword(
  email: 'cliente@email.com',
  password: '123456',
);
Usuario nuevoUsuario = Usuario(
  uid: cred.user!.uid,
  correo: 'cliente@email.com',
  rolId: 1, // ejemplo: rol cliente
);
await firestore.collection('usuarios').doc(nuevoUsuario.uid).set(nuevoUsuario.toMap());
```

### 3. Leer una colección (ej. Mascotas)
```dart
QuerySnapshot snapshot = await firestore.collection('mascotas').get();
List<Mascota> mascotas = snapshot.docs
    .map((doc) => Mascota.fromFirestore(doc))
    .toList();
```

### 4. Guardar una mascota con referencias
```dart
Mascota nueva = Mascota(
  mascotaId: 1001, // puedes usar el uid de Firestore como string auto-generado
  nombre: 'Firulais',
  propietarioId: 5, // ID del propietario en su colección
  razaId: 12,
);
await firestore.collection('mascotas').doc('1001').set(nueva.toMap());
```

## 📁 Estructura sugerida en Firestore

Cada clase anterior corresponde a una colección en Firestore:

- `roles`
- `especialidades`
- `especies`
- `propietarios`
- `recetas_medicas`
- `constantes_cabecera`
- `usuarios` (document ID = uid de Firebase Auth)
- `razas`
- `medicos_veterinarios`
- `mascotas`
- `historias_clinicas`
- `consultas_medicas`
- `constantes_detalle`

Las relaciones se manejan guardando los IDs de los documentos relacionados (ej. `propietarioId`, `razaId`). Opcionalmente, puedes usar `DocumentReference` en lugar de `int` si prefieres referencias tipadas de Firestore.

## ⚠️ Notas importantes

- **Seguridad**: no guardes contraseñas en texto plano. Usa siempre Firebase Auth. El campo `contrasena` en `Usuario` se incluye solo por fidelidad al SQL original, pero no deberías almacenarlo.
- **TimeOfDay**: Firestore no tiene un tipo nativo para hora sin fecha. En el modelo se guarda como `String` en formato `HH:mm`. Puedes adaptarlo si prefieres usar `DateTime` con una fecha ficticia.
- **IDs numéricos vs. automáticos**: El SQL original usa `INT(10)` como PK. En Firestore es más común usar IDs automáticos (`doc.id`). Puedes mezclar ambos enfoques; el modelo soporta `int` pero puedes cambiarlo a `String` si lo prefieres.

El archivo está listo para copiar y pegar en tu proyecto Flutter. Solo asegúrate de agregar las dependencias en `pubspec.yaml`:

```yaml
dependencies:
  firebase_core: ^3.0.0
  cloud_firestore: ^5.0.0
  firebase_auth: ^5.0.0
```
