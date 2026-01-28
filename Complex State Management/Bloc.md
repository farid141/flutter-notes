# Bloc

Berikut penjelasan **komprehensif dan end-to-end** mengenai **Bloc sebagai state management** di Flutter, mulai dari konsep dasar, arsitektur, alur data, hingga best practice penggunaannya.

---

## 1. Apa itu Bloc?

**Bloc (Business Logic Component)** adalah pola arsitektur sekaligus library (`bloc` dan `flutter_bloc`) yang digunakan untuk:

* **Memisahkan business logic dari UI**
* Mengelola **state secara terpusat, terprediksi, dan testable**
* Menerapkan prinsip **unidirectional data flow**

Bloc sangat cocok untuk aplikasi **menengah hingga besar** yang membutuhkan:

* konsistensi state
* maintainability
* kemudahan testing

---

## 2. Masalah yang Diselesaikan Bloc

Tanpa state management yang baik:

* Logic bercampur dengan UI
* State sulit ditelusuri sumber perubahannya
* Sulit di-test
* Rebuild UI tidak terkontrol

Bloc mengatasi ini dengan:

* **Satu sumber kebenaran (single source of truth)**
* **State immutable**
* **Perubahan state hanya melalui event**

---

## 3. Konsep Inti Bloc

### 3.1 Event

**Event** merepresentasikan **aksi atau intent** dari user atau sistem.

Contoh:

```dart
abstract class CounterEvent {}

class IncrementPressed extends CounterEvent {}
class DecrementPressed extends CounterEvent {}
```

Event:

* Bersifat **input**
* Tidak mengandung logic
* Biasanya berasal dari UI

---

### 3.2 State

**State** merepresentasikan **kondisi aplikasi pada satu waktu tertentu**.

Contoh:

```dart
class CounterState {
  final int value;

  CounterState(this.value);
}
```

Karakteristik state:

* **Immutable**
* Menjelaskan “apa yang terjadi”, bukan “apa yang dilakukan”
* Bisa berupa:

  * class tunggal
  * union/sealed class (loading, success, error)

---

### 3.3 Bloc

**Bloc** adalah pusat pengolahan:

* menerima event
* menjalankan business logic
* mengeluarkan state baru

Contoh:

```dart
class CounterBloc extends Bloc<CounterEvent, CounterState> {
  CounterBloc() : super(CounterState(0)) {
    on<IncrementPressed>((event, emit) {
      emit(CounterState(state.value + 1));
    });

    on<DecrementPressed>((event, emit) {
      emit(CounterState(state.value - 1));
    });
  }
}
```

---

## 4. Alur Data (Unidirectional Data Flow)

Alur standar Bloc:

```Alur
UI
 ↓ dispatch
Event
 ↓
Bloc (business logic)
 ↓ emit
State (immutable)
 ↓
UI rebuild / side effect
```

Poin penting:

* UI **tidak pernah mengubah state langsung**
* Bloc **tidak tahu tentang UI**
* Semua perubahan state **terlacak dan eksplisit**

---

## 5. Integrasi Bloc dengan UI (flutter_bloc)

### 5.1 BlocProvider

Digunakan untuk **menyediakan Bloc ke widget tree**.

```dart
BlocProvider(
  create: (_) => CounterBloc(),
  child: CounterPage(),
);
```

Atau multiple:

```dart
MultiBlocProvider(
  providers: [
    BlocProvider(create: (_) => AuthBloc()),
    BlocProvider(create: (_) => ProfileBloc()),
  ],
  child: App(),
);
```

---

### 5.2 BlocBuilder

Untuk **render UI berdasarkan state**.

```dart
BlocBuilder<CounterBloc, CounterState>(
  builder: (context, state) {
    return Text('${state.value}');
  },
);
```

* Dipanggil setiap state berubah
* Harus **pure UI**
* Tidak boleh ada side effect

---

### 5.3 BlocListener

Untuk **side effects**, bukan UI.

```dart
BlocListener<AuthBloc, AuthState>(
  listener: (context, state) {
    if (state is AuthSuccess) {
      Navigator.pushNamed(context, '/home');
    }
  },
);
```

Side effects:

* navigation
* snackbar
* dialog
* logging
* analytics

---

### 5.4 MultiBlocListener

Mengelompokkan banyak listener (bukan state management).

---

### 5.5 BlocConsumer

Gabungan `BlocBuilder` + `BlocListener`.

```dart
BlocConsumer<LoginBloc, LoginState>(
  listener: (context, state) {
    if (state is LoginError) {
      showError(state.message);
    }
  },
  builder: (context, state) {
    if (state is LoginLoading) {
      return CircularProgressIndicator();
    }
    return LoginForm();
  },
);
```

---

## 6. Cubit vs Bloc

### Cubit

Versi lebih sederhana dari Bloc.

```dart
class CounterCubit extends Cubit<int> {
  CounterCubit() : super(0);

  void increment() => emit(state + 1);
}
```

| Cubit                 | Bloc                 |
| --------------------- | -------------------- |
| Tanpa event           | Menggunakan event    |
| Lebih ringkas         | Lebih eksplisit      |
| Cocok state sederhana | Cocok logic kompleks |

**Rule praktis**:

* Sederhana → Cubit
* Kompleks / enterprise → Bloc

---

## 7. Struktur Folder (Best Practice)

Contoh skala menengah:

```Struktur Folder
feature/
 ├── bloc/
 │   ├── login_bloc.dart
 │   ├── login_event.dart
 │   └── login_state.dart
 ├── view/
 │   └── login_page.dart
 └── widget/
```

Atau feature-first:

```Struktur Folder
login/
 ├── bloc/
 ├── view/
 └── model/
```

---

## 8. Testing Bloc (Keunggulan Utama)

Bloc sangat mudah di-test karena:

* Logic terpisah
* State predictable

Contoh:

```dart
blocTest<CounterBloc, CounterState>(
  'emit CounterState(1) when increment pressed',
  build: () => CounterBloc(),
  act: (bloc) => bloc.add(IncrementPressed()),
  expect: () => [CounterState(1)],
);
```

---

## 9. Kapan Bloc Digunakan?

Gunakan Bloc jika:

* Aplikasi punya banyak screen
* State saling bergantung
* Perlu testability tinggi
* Banyak async process (API, socket)

Hindari Bloc jika:

* App sangat kecil
* State hanya lokal dan sederhana

---

## 10. Kesimpulan Akhir

* **Bloc adalah state management yang kuat dan terstruktur**
* Menggunakan:

  * Event → intent
  * Bloc → business logic
  * State → kondisi aplikasi
* UI hanya:

  * mengirim event
  * merespons state
* Memberikan:

  * skalabilitas
  * maintainability
  * testability

Secara prinsip:

> **UI pasif, Bloc aktif, State immutable**

Jika Anda ingin, saya dapat melanjutkan dengan:

* studi kasus real (login, pagination, form)
* perbandingan Bloc vs Provider vs Riverpod
* anti-pattern umum dalam penggunaan Bloc
