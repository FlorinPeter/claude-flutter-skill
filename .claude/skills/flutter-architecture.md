# Flutter Architecture Skill

This skill provides guidelines for building well-structured Flutter applications following MVVM architecture, with emphasis on clean widget organization and avoiding large files.

## Core Architecture Principles

### Layered Architecture

Structure applications in distinct layers that communicate only with adjacent layers:

1. **UI Layer** (Presentation)
   - Views (Widgets)
   - ViewModels (ChangeNotifier)

2. **Data Layer** (Model)
   - Repositories (Single Source of Truth)
   - Services (API wrappers)

3. **Domain Layer** (Optional - for complex apps)
   - Use-cases / Interactors
   - Business logic shared across ViewModels

### Key Principles

- **Separation of Concerns**: Divide functionality into distinct, self-contained units
- **Single Source of Truth (SSOT)**: Each data type has one authoritative source (Repository)
- **Unidirectional Data Flow**: State flows down, events flow up
- **UI as Function of State**: UI reflects current immutable state

---

## Project Structure

Organize code using a hybrid approach - **by feature** for UI, **by type** for data:

```
lib/
├── main.dart
├── app.dart
├── config/
│   ├── routes.dart
│   └── theme.dart
├── ui/                          # Organized BY FEATURE
│   ├── home/
│   │   ├── home_screen.dart
│   │   ├── home_view_model.dart
│   │   └── widgets/
│   │       ├── home_header.dart
│   │       ├── home_booking_list.dart
│   │       └── home_booking_card.dart
│   ├── booking/
│   │   ├── booking_screen.dart
│   │   ├── booking_view_model.dart
│   │   └── widgets/
│   │       ├── booking_form.dart
│   │       ├── booking_date_picker.dart
│   │       └── booking_summary.dart
│   └── shared/                  # Reusable widgets across features
│       ├── buttons/
│       │   ├── primary_button.dart
│       │   └── icon_button.dart
│       ├── cards/
│       │   └── info_card.dart
│       └── loading/
│           └── loading_indicator.dart
├── data/                        # Organized BY TYPE
│   ├── repositories/
│   │   ├── booking_repository.dart
│   │   ├── user_repository.dart
│   │   └── auth_repository.dart
│   └── services/
│       ├── api_client.dart
│       ├── local_storage_service.dart
│       └── auth_service.dart
├── domain/                      # Shared data models
│   ├── models/
│   │   ├── user.dart
│   │   ├── booking.dart
│   │   └── destination.dart
│   └── use_cases/               # Optional - for complex logic
│       └── create_booking_use_case.dart
└── utils/
    ├── result.dart
    ├── command.dart
    └── extensions/
        └── date_extensions.dart
```

---

## File Size Guidelines

**CRITICAL: Avoid large files.** Follow these limits:

| File Type | Max Lines | Action if Exceeded |
|-----------|-----------|-------------------|
| Widget/Screen | 150-200 | Extract child widgets |
| ViewModel | 150-200 | Extract to use-cases |
| Repository | 100-150 | Split by domain area |
| Service | 100-150 | Split by API endpoint group |
| Model | 50-100 | Keep focused, use composition |

### Strategies for Reducing File Size

1. **Extract widgets** into separate files in a `widgets/` subfolder
2. **Use composition** over large widget trees
3. **Split ViewModels** when handling multiple unrelated features
4. **Create use-cases** for complex business logic
5. **Group related constants/enums** in dedicated files

---

## UI Layer Implementation

### Views (Widgets)

Views are widgets that display UI state and handle user interactions. Keep views thin - delegate logic to ViewModels.

**Views should ONLY contain:**
- Widget composition and layout
- Conditional widget display based on ViewModel state
- Animations and transitions
- Simple routing/navigation
- Listening to ViewModel changes

**Views should NOT contain:**
- Business logic
- Data fetching
- State mutations
- Direct repository access

```dart
// ✅ GOOD: Clean view with separated concerns
// File: lib/ui/home/home_screen.dart (< 150 lines)

class HomeScreen extends StatelessWidget {
  const HomeScreen({super.key, required this.viewModel});

  final HomeViewModel viewModel;

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: const HomeAppBar(),
      body: ListenableBuilder(
        listenable: viewModel,
        builder: (context, _) {
          return HomeContent(
            user: viewModel.user,
            bookings: viewModel.bookings,
            onBookingTap: viewModel.selectBooking,
          );
        },
      ),
    );
  }
}
```

```dart
// ✅ GOOD: Extracted child widget
// File: lib/ui/home/widgets/home_content.dart

class HomeContent extends StatelessWidget {
  const HomeContent({
    super.key,
    required this.user,
    required this.bookings,
    required this.onBookingTap,
  });

  final User? user;
  final List<BookingSummary> bookings;
  final ValueChanged<int> onBookingTap;

  @override
  Widget build(BuildContext context) {
    return CustomScrollView(
      slivers: [
        if (user != null) HomeUserHeader(user: user!),
        HomeBookingList(
          bookings: bookings,
          onBookingTap: onBookingTap,
        ),
      ],
    );
  }
}
```

### ViewModels

ViewModels handle UI logic, convert data to UI state, and expose commands for user actions.

**ViewModel responsibilities:**
- Receive repositories via constructor injection
- Expose immutable UI state
- Provide commands for user actions
- Transform domain data to UI state
- Call `notifyListeners()` on state changes

```dart
// File: lib/ui/home/home_view_model.dart

class HomeViewModel extends ChangeNotifier {
  HomeViewModel({
    required BookingRepository bookingRepository,
    required UserRepository userRepository,
  })  : _bookingRepository = bookingRepository,
        _userRepository = userRepository {
    // Initialize commands
    load = Command0(_load)..execute();
    deleteBooking = Command1(_deleteBooking);
  }

  final BookingRepository _bookingRepository;
  final UserRepository _userRepository;

  // Commands
  late final Command0 load;
  late final Command1<void, int> deleteBooking;

  // UI State - always expose immutable data
  User? _user;
  User? get user => _user;

  List<BookingSummary> _bookings = [];
  UnmodifiableListView<BookingSummary> get bookings =>
      UnmodifiableListView(_bookings);

  // Private methods for commands
  Future<Result<void>> _load() async {
    try {
      final userResult = await _userRepository.getUser();
      if (userResult case Ok<User>(:final value)) {
        _user = value;
      }

      final bookingsResult = await _bookingRepository.getBookingsList();
      if (bookingsResult case Ok<List<BookingSummary>>(:final value)) {
        _bookings = value;
      }

      return const Result.ok(null);
    } finally {
      notifyListeners();
    }
  }

  Future<Result<void>> _deleteBooking(int id) async {
    final result = await _bookingRepository.delete(id);
    if (result case Ok()) {
      _bookings = _bookings.where((b) => b.id != id).toList();
      notifyListeners();
    }
    return result;
  }
}
```

### Listening to ViewModels

Use `ListenableBuilder` to react to ViewModel changes:

```dart
ListenableBuilder(
  listenable: viewModel,
  builder: (context, _) {
    return Text(viewModel.user?.name ?? 'Guest');
  },
)
```

For commands with loading/error states:

```dart
ListenableBuilder(
  listenable: viewModel.load,
  builder: (context, child) {
    if (viewModel.load.running) {
      return const LoadingIndicator();
    }
    if (viewModel.load.error) {
      return ErrorWidget(onRetry: viewModel.load.execute);
    }
    return child!;
  },
  child: HomeContent(viewModel: viewModel),
)
```

---

## Data Layer Implementation

### Repositories

Repositories are the **Single Source of Truth** for data. They:
- Poll services and transform raw data to domain models
- Handle caching, error handling, and retries
- Never expose raw API models to the UI layer

```dart
// File: lib/data/repositories/booking_repository.dart

abstract class BookingRepository {
  Future<Result<List<BookingSummary>>> getBookingsList();
  Future<Result<Booking>> getBooking(int id);
  Future<Result<void>> createBooking(Booking booking);
  Future<Result<void>> delete(int id);
}

class BookingRepositoryRemote implements BookingRepository {
  BookingRepositoryRemote({required ApiClient apiClient})
      : _apiClient = apiClient;

  final ApiClient _apiClient;

  @override
  Future<Result<Booking>> getBooking(int id) async {
    try {
      final result = await _apiClient.getBooking(id);
      if (result case Error(:final error)) {
        return Result.error(error);
      }

      final apiBooking = (result as Ok<BookingApiModel>).value;

      // Transform API model to domain model
      return Result.ok(Booking(
        id: apiBooking.id,
        startDate: apiBooking.startDate,
        endDate: apiBooking.endDate,
        destination: apiBooking.destination,
      ));
    } on Exception catch (e) {
      return Result.error(e);
    }
  }

  // ... other methods
}
```

### Services

Services wrap external APIs and remain stateless:

```dart
// File: lib/data/services/api_client.dart

class ApiClient {
  ApiClient({required this.baseUrl, http.Client? client})
      : _client = client ?? http.Client();

  final String baseUrl;
  final http.Client _client;

  Future<Result<List<BookingApiModel>>> getBookings() async {
    try {
      final response = await _client.get(Uri.parse('$baseUrl/bookings'));
      if (response.statusCode == 200) {
        final List<dynamic> json = jsonDecode(response.body);
        return Result.ok(
          json.map((e) => BookingApiModel.fromJson(e)).toList(),
        );
      }
      return Result.error(ApiException(response.statusCode));
    } on Exception catch (e) {
      return Result.error(e);
    }
  }

  Future<Result<void>> deleteBooking(int id) async {
    try {
      final response = await _client.delete(
        Uri.parse('$baseUrl/bookings/$id'),
      );
      if (response.statusCode == 204) {
        return const Result.ok(null);
      }
      return Result.error(ApiException(response.statusCode));
    } on Exception catch (e) {
      return Result.error(e);
    }
  }
}
```

---

## Design Patterns

### Command Pattern

Encapsulate ViewModel actions with automatic state management:

```dart
// File: lib/utils/command.dart

abstract class Command<T> extends ChangeNotifier {
  Command();

  bool _running = false;
  bool get running => _running;

  Result<T>? _result;
  Result<T>? get result => _result;

  bool get error => _result is Error<T>;
  bool get completed => _result is Ok<T>;

  /// Clears the result state
  void clearResult() {
    _result = null;
    notifyListeners();
  }

  /// Executes the command action
  Future<void> _execute(Future<Result<T>> Function() action) async {
    if (_running) return;

    _running = true;
    _result = null;
    notifyListeners();

    try {
      _result = await action();
    } finally {
      _running = false;
      notifyListeners();
    }
  }
}

/// Command with no arguments
class Command0<T> extends Command<T> {
  Command0(this._action);

  final Future<Result<T>> Function() _action;

  Future<void> execute() => _execute(_action);
}

/// Command with one argument
class Command1<T, A> extends Command<T> {
  Command1(this._action);

  final Future<Result<T>> Function(A) _action;

  Future<void> execute(A argument) => _execute(() => _action(argument));
}
```

### Result Pattern

Handle errors explicitly without exceptions:

```dart
// File: lib/utils/result.dart

sealed class Result<T> {
  const Result();

  const factory Result.ok(T value) = Ok<T>;
  const factory Result.error(Exception error) = Error<T>;

  /// Returns true if this is an Ok result
  bool get isOk => this is Ok<T>;

  /// Returns true if this is an Error result
  bool get isError => this is Error<T>;
}

final class Ok<T> extends Result<T> {
  const Ok(this.value);
  final T value;
}

final class Error<T> extends Result<T> {
  const Error(this.error);
  final Exception error;
}

// Usage with pattern matching
switch (result) {
  case Ok<User>(:final value):
    _user = value;
  case Error<User>(:final error):
    _log.warning('Failed to load user', error);
}
```

### Optimistic State Pattern

Update UI immediately for perceived performance:

```dart
class SubscribeViewModel extends ChangeNotifier {
  SubscribeViewModel({required SubscriptionRepository repository})
      : _repository = repository;

  final SubscriptionRepository _repository;

  bool _subscribed = false;
  bool get subscribed => _subscribed;

  bool _error = false;
  bool get error => _error;

  Future<void> subscribe() async {
    // Optimistic update - show subscribed immediately
    _subscribed = true;
    _error = false;
    notifyListeners();

    try {
      await _repository.subscribe();
    } on Exception {
      // Revert on error
      _subscribed = false;
      _error = true;
    } finally {
      notifyListeners();
    }
  }
}
```

---

## Dependency Injection

Use `package:provider` for dependency injection:

```dart
// File: lib/main.dart

void main() {
  runApp(
    MultiProvider(
      providers: [
        // Services
        Provider<ApiClient>(
          create: (_) => ApiClient(baseUrl: 'https://api.example.com'),
        ),

        // Repositories
        ProxyProvider<ApiClient, BookingRepository>(
          update: (_, apiClient, __) => BookingRepositoryRemote(
            apiClient: apiClient,
          ),
        ),
        ProxyProvider<ApiClient, UserRepository>(
          update: (_, apiClient, __) => UserRepositoryRemote(
            apiClient: apiClient,
          ),
        ),
      ],
      child: const MyApp(),
    ),
  );
}
```

Create ViewModels in routes:

```dart
// File: lib/config/routes.dart

final router = GoRouter(
  routes: [
    GoRoute(
      path: '/',
      builder: (context, state) => HomeScreen(
        viewModel: HomeViewModel(
          bookingRepository: context.read<BookingRepository>(),
          userRepository: context.read<UserRepository>(),
        ),
      ),
    ),
  ],
);
```

---

## Widget Organization Best Practices

### 1. Extract Widgets Aggressively

```dart
// ❌ BAD: Monolithic widget
class HomeScreen extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: Column(
        children: [
          // 50 lines of header code...
          // 100 lines of list code...
          // 30 lines of footer code...
        ],
      ),
    );
  }
}

// ✅ GOOD: Composed of smaller widgets
class HomeScreen extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: Column(
        children: [
          const HomeHeader(),
          const Expanded(child: HomeBookingList()),
          const HomeFooter(),
        ],
      ),
    );
  }
}
```

### 2. Use Named Constructors for Variants

```dart
// File: lib/ui/shared/buttons/action_button.dart

class ActionButton extends StatelessWidget {
  const ActionButton({
    super.key,
    required this.label,
    required this.onPressed,
    this.icon,
    this.style = ActionButtonStyle.primary,
  });

  const ActionButton.primary({
    super.key,
    required this.label,
    required this.onPressed,
    this.icon,
  }) : style = ActionButtonStyle.primary;

  const ActionButton.secondary({
    super.key,
    required this.label,
    required this.onPressed,
    this.icon,
  }) : style = ActionButtonStyle.secondary;

  final String label;
  final VoidCallback onPressed;
  final IconData? icon;
  final ActionButtonStyle style;

  @override
  Widget build(BuildContext context) {
    // Implementation
  }
}
```

### 3. Prefer Composition Over Inheritance

```dart
// ✅ GOOD: Composition
class ProfileCard extends StatelessWidget {
  const ProfileCard({super.key, required this.user});
  final User user;

  @override
  Widget build(BuildContext context) {
    return InfoCard(
      child: ProfileContent(user: user),
    );
  }
}

// ❌ AVOID: Deep inheritance hierarchies
class ProfileCard extends InfoCard { /* ... */ }
```

### 4. Keep Widgets Focused

Each widget should have a single responsibility:

```dart
// ✅ GOOD: Single responsibility widgets
class BookingCard extends StatelessWidget { /* Display booking */ }
class BookingForm extends StatelessWidget { /* Handle input */ }
class BookingActions extends StatelessWidget { /* Action buttons */ }

// ❌ BAD: Widget doing too much
class BookingWidget extends StatelessWidget {
  // Displays, handles input, AND manages actions
}
```

---

## Domain Models

Use immutable models with `freezed`:

```dart
// File: lib/domain/models/booking.dart

@freezed
class Booking with _$Booking {
  const factory Booking({
    required int id,
    required DateTime startDate,
    required DateTime endDate,
    required Destination destination,
    @Default([]) List<Activity> activities,
  }) = _Booking;

  factory Booking.fromJson(Map<String, dynamic> json) =>
      _$BookingFromJson(json);
}
```

Separate API models from domain models:

```dart
// File: lib/data/services/models/booking_api_model.dart

@freezed
class BookingApiModel with _$BookingApiModel {
  const factory BookingApiModel({
    required int id,
    required String start_date,  // Raw API format
    required String end_date,
    required String destination_ref,
    required List<String> activities_ref,
  }) = _BookingApiModel;

  factory BookingApiModel.fromJson(Map<String, dynamic> json) =>
      _$BookingApiModelFromJson(json);
}
```

---

## Testing Guidelines

### ViewModel Tests

Test ViewModels with fake repositories:

```dart
// File: test/ui/home/home_view_model_test.dart

void main() {
  group('HomeViewModel', () {
    late HomeViewModel viewModel;
    late FakeBookingRepository bookingRepository;
    late FakeUserRepository userRepository;

    setUp(() {
      bookingRepository = FakeBookingRepository();
      userRepository = FakeUserRepository();
      viewModel = HomeViewModel(
        bookingRepository: bookingRepository,
        userRepository: userRepository,
      );
    });

    test('loads user and bookings on init', () async {
      await viewModel.load.execute();

      expect(viewModel.user, isNotNull);
      expect(viewModel.bookings, isNotEmpty);
    });

    test('deleteBooking removes booking from list', () async {
      await viewModel.load.execute();
      final initialCount = viewModel.bookings.length;

      await viewModel.deleteBooking.execute(1);

      expect(viewModel.bookings.length, initialCount - 1);
    });
  });
}
```

### Widget Tests

```dart
// File: test/ui/home/home_screen_test.dart

void main() {
  group('HomeScreen', () {
    late HomeViewModel viewModel;
    late FakeBookingRepository bookingRepository;

    setUp(() {
      bookingRepository = FakeBookingRepository()
        ..addBooking(testBooking);
      viewModel = HomeViewModel(
        bookingRepository: bookingRepository,
        userRepository: FakeUserRepository(),
      );
    });

    testWidgets('displays booking list', (tester) async {
      await tester.pumpWidget(
        MaterialApp(
          home: HomeScreen(viewModel: viewModel),
        ),
      );
      await tester.pumpAndSettle();

      expect(find.byType(BookingCard), findsOneWidget);
    });
  });
}
```

### Repository Tests

```dart
// File: test/data/repositories/booking_repository_test.dart

void main() {
  group('BookingRepositoryRemote', () {
    late BookingRepository repository;
    late FakeApiClient apiClient;

    setUp(() {
      apiClient = FakeApiClient();
      repository = BookingRepositoryRemote(apiClient: apiClient);
    });

    test('getBooking transforms API model to domain model', () async {
      final result = await repository.getBooking(1);

      expect(result, isA<Ok<Booking>>());
      final booking = (result as Ok<Booking>).value;
      expect(booking.id, 1);
    });
  });
}
```

---

## Checklist for New Features

When implementing a new feature, ensure:

- [ ] **Structure**: Created feature folder under `ui/` with screen, viewmodel, and widgets subfolder
- [ ] **Widget Size**: No widget file exceeds 200 lines
- [ ] **ViewModel Size**: ViewModel under 200 lines, complex logic extracted to use-cases
- [ ] **Immutability**: All exposed state is immutable
- [ ] **Commands**: User actions use Command pattern
- [ ] **Repository**: Data access through repository, not direct service calls
- [ ] **Result Pattern**: Error handling uses Result, not thrown exceptions
- [ ] **Testing**: Unit tests for ViewModel, widget tests for Screen
- [ ] **Separation**: View contains no business logic, ViewModel has no UI code
