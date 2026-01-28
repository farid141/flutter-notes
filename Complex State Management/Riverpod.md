# Riverpod

adalah *state management library* untuk Flutter yang dikembangkan oleh **Remi Rousselet** (pembuat `Provider`).
Riverpod `memperbaiki` berbagai keterbatasan Provider, seperti:

- Tidak bergantung pada `BuildContext`
- Aman dari *provider scoping* yang salah
- Mendukung *code generation* (melalui `riverpod_generator`)
- Lebih mudah untuk *testing* dan *refactoring*

Untuk mengelola state yang lebih dalam kita dapat melakukan dengan beberapa cara. Ada 2 konsep utama dalam penggunaan `wide-state-management` ini, yaitu `provider` dan `consumer`.

```bash
flutter pub add flutter_riverpod
flutter pub add riverpod_annotation
flutter pub add riverpod_generator
flutter pub add build_runner
```

Disini akan dibahas secara spesifik wide state management menggunakan riverpod.

## 1. Wrap Komponen

Wrap kompnonen dengan `ProviderScope`

```dart
void main(){
    runApp(const ProviderScope(child: App()))
}
```

### 2. Provider

Terdapat beberapa jenis provider yang bisa digunakan:

- Provider → Read-only values
- StateProvider → Simple state management
- FutureProvider → Async data (e.g., API calls)
- StreamProvider → Real-time updates (e.g., Firebase)
- StateNotifierProvider → Advanced state management

| Jenis Provider          | Kegunaan                                                      |
| ----------------------- | ------------------------------------------------------------- |
| `Provider`              | Menyediakan data statis atau objek non-state                  |
| `StateProvider`         | Menyimpan dan mengubah nilai sederhana (counter, toggle, dll) |
| `StateNotifierProvider` | Mengelola state kompleks dengan class `StateNotifier`         |
| `FutureProvider`        | Untuk operasi async (seperti HTTP request)                    |
| `StreamProvider`        | Untuk data berbasis stream                                    |

```dart
// read-only
final mealsProvider = Provider((ref){
    return dummyMeals
})

// read and write (bisa dilakukan diluar build?)
final counterProvider = StateProvider<int>((ref) => 0);
final count = ref.watch(counterProvider); // read in component
onPressed: () => ref.read(counterProvider.notifier).state++,  // write in component
```

### StateProvider VS StateNotifier

Perbedaan dari keduanya adalah `StateNotifier` lebih rapi dibandingkan `StateProvider`. Kita ambil kasus **Todo App** (sederhana tapi cukup kompleks untuk membedakan).

#### StateProvider

Kalau pakai `StateProvider`, kita langsung simpan list todo di dalamnya.

Semua logika (nambah, hapus, toggle) dilakukan di luar.

```dart
import 'package:flutter_riverpod/flutter_riverpod.dart';

// Model sederhana
class Todo {
  final String id;
  final String title;
  final bool completed;

  Todo({
    required this.id,
    required this.title,
    this.completed = false,
  });

  Todo copyWith({String? title, bool? completed}) {
    return Todo(
      id: id,
      title: title ?? this.title,
      completed: completed ?? this.completed,
    );
  }
}

// StateProvider: langsung simpan list Todo
final todosProvider = StateProvider<List<Todo>>((ref) => []);

// Contoh penggunaan di widget
// Tambah todo
void addTodo(WidgetRef ref, String title) {
  final todos = ref.read(todosProvider.notifier);
  todos.state = [
    ...todos.state,
    Todo(id: DateTime.now().toString(), title: title),
  ];
}

// Toggle
void toggleTodo(WidgetRef ref, String id) {
  final todos = ref.read(todosProvider.notifier);
  todos.state = todos.state.map((todo) {
    if (todo.id == id) {
      return todo.copyWith(completed: !todo.completed);
    }
    return todo;
  }).toList();
}
```

👉 Kekurangan: logika bercampur di luar provider, kode bisa cepat berantakan kalau fitur makin banyak.

#### StateNotifier

Di sini kita bikin class khusus (`TodoNotifier`) untuk mengelola semua logika.
State lebih terorganisir, gampang dikembangkan.

```dart
import 'package:flutter_riverpod/flutter_riverpod.dart';

// Model sama dengan sebelumnya
class Todo {
  final String id;
  final String title;
  final bool completed;

  Todo({
    required this.id,
    required this.title,
    this.completed = false,
  });

  Todo copyWith({String? title, bool? completed}) {
    return Todo(
      id: id,
      title: title ?? this.title,
      completed: completed ?? this.completed,
    );
  }
}

// StateNotifier untuk mengelola list Todo
class TodoNotifier extends StateNotifier<List<Todo>> {
  TodoNotifier() : super([]);

  void add(String title) {
    state = [
      ...state,
      Todo(id: DateTime.now().toString(), title: title),
    ];
  }

  void toggle(String id) {
    state = state.map((todo) {
      if (todo.id == id) {
        return todo.copyWith(completed: !todo.completed);
      }
      return todo;
    }).toList();
  }

  void remove(String id) {
    state = state.where((todo) => todo.id != id).toList();
  }
}

// Provider
final todosProvider =
    StateNotifierProvider<TodoNotifier, List<Todo>>((ref) => TodoNotifier());
```

👉 Keuntungan: semua logika terkapsulasi di dalam `TodoNotifier`, gampang dipakai di UI:

```dart
ref.read(todosProvider.notifier).add("Belajar Riverpod"); // Tambah
ref.read(todosProvider.notifier).toggle(todoId); // Toggle
ref.read(todosProvider.notifier).remove(todoId); // Remove
```

Intinya, dalam `StateNotifierProvider`, kita dapat membuat method berisi logic dalam class notifier.

- **`StateProvider`** → data + logika bercampur di widget atau helper function.
- **`StateNotifier`** → logika dikelompokkan di satu tempat (class), lebih mudah dipelihara & scalable.

### Consumer

Digunakan untuk mengakses state riverpod dalam widget. Ada 3 cara:

1. Untuk widget stateless gunakan (`ConsumerWidget`). Method build akan memiliki parameter kedua `WidgetRef ref`.

    ```dart
    class MyHomeWidget extends ConsumerWidget {
      @override
      Widget build(BuildContext context, WidgetRef ref) {
        // Gunakan ref.watch untuk mendengarkan perubahan data secara reaktif
        final counter = ref.watch(counterProvider);
        
        return Text('Nilai: $counter');
      }
    }
    ```

2. Untuk merombak dalam level widget (bukan build), gunakan consumer

    ```dart
    Consumer(
      builder: (context, ref, child) {
        final data = ref.watch(dataProvider);
        return Text('Data: $data');
      },
    ),
    ```

3. Ganti class widget untuk extend dari `ConsumerWidget` jika stateless dan `ConsumerStatefulWidget` jika statefull.

    ```dart
    class MyWidget extends ConsumerStatefulWidget {
      @override
      _MyWidgetState createState() => _MyWidgetState();
    }

    class _MyWidgetState extends ConsumerState<MyWidget> {
      @override
      void initState() {
        super.initState();
        // Gunakan ref.read untuk aksi sekali jalan (one-time)
        ref.read(counterProvider.notifier).increment();
      }

      @override
      Widget build(BuildContext context) {
        final value = ref.watch(counterProvider);
        return Text('$value');
      }
    }
    ```

## Konvensi Struktur Proyek Riverpod (Clean Architecture Style)

Struktur yang direkomendasikan agar proyek tetap **scalable dan maintainable**:

```yml
lib/
│
├── main.dart
├── app/
│   ├── app.dart                 # Root widget (MaterialApp)
│   ├── router.dart              # Route management
│   └── theme.dart               # Theme setup
│
├── core/
│   ├── exceptions/
│   ├── failure.dart
│   ├── utils/
│   ├── constants/
│   └── services/                # Shared services (e.g., Dio, Firebase)
│
├── features/
│   ├── counter/
│   │   ├── data/                # Repository, data sources
│   │   ├── domain/              # Entities & business logic
│   │   ├── presentation/        # UI layer
│   │   │   ├── pages/
│   │   │   ├── widgets/
│   │   │   └── controllers/     # StateNotifiers or Providers
│   │   └── providers.dart       # Provider declarations
│   │
│   └── auth/
│       ├── data/
│       ├── domain/
│       ├── presentation/
│       └── providers.dart
│
└── shared/
    ├── widgets/                 # Common reusable widgets
    └── providers/               # Global providers
```

### Catatan

- Setiap **feature** memiliki folder sendiri (misalnya `counter`, `auth`, `todo`).
- Semua provider dideklarasikan di `providers.dart` pada masing-masing fitur.
- Jika menggunakan code generation (`riverpod_generator`), file provider akan otomatis dibuat dengan suffix `.g.dart`.

---

## Keunggulan Riverpod

- Tidak tergantung pada `BuildContext`
- Mudah untuk testing
- Kompatibel dengan arsitektur berskala besar (Clean Architecture, MVVM, dsb)
- Bisa dipakai di luar widget tree (misalnya di background isolate)
- Mendukung *auto dispose* untuk efisiensi memori
