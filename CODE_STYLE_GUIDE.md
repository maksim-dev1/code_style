# Архитектура Flutter-проекта — гайд с нуля

> Пошаговое руководство, как поднять новый Flutter-проект и писать в нём код.
> Рассчитано на начинающих: каждое правило — с объяснением **зачем**, с примерами
> и чек-листами. Стек: **BLoC + Freezed + Drift**. Имена хелперов (`AppLogger`,
> `AppColors`, …) — условные: назови как удобно, важно лишь, что это **единый
> источник** на весь проект.
>
> Язык кода и комментариев — **русский** (комментарии `///` для публичного API,
> `//` для внутреннего).

---

# Часть I. Основы

## 0. Философия за 30 секунд

- **Чистая архитектура** — код делится на слои, каждый отвечает за своё. UI не лезет
  в базу данных, бизнес-логика не знает про виджеты.
- **Однонаправленный поток данных** — UI отправляет *события* в BLoC, BLoC меняет
  *состояние*, UI перерисовывается по состоянию. И всё.
- **Мелкие фокусные части** — одна фича = одна забота. Маленькое легко понять,
  переиспользовать и тестировать.

Эти три идеи объясняют 90% правил ниже. Если правило кажется странным — вернись сюда.

---

## 1. Технологии (стек)

| Область | Инструмент | Зачем |
|---|---|---|
| Управление состоянием | `flutter_bloc` (**Bloc**, не Cubit) | предсказуемый поток «событие → состояние» |
| Неизменяемые модели | `freezed` (+ `freezed_annotation`) | `copyWith`, `==`, union-типы «из коробки» |
| Локальная БД | `drift` (+ `drift_dev`) | типобезопасный SQLite с реактивными запросами |
| Секреты | `flutter_secure_storage` | пароли/токены в Keychain/Keystore, не в БД |
| DI (внедрение зависимостей) | `provider` (только для DI!) | прокинуть репозитории/блоки вниз по дереву |
| Логирование | `talker` / `talker_flutter` | единый логгер вместо `print` |
| Уникальные id | `uuid` | генерация id сущностей |
| Кодогенерация | `build_runner` | генерит код для freezed/drift |

**Запрещено:** Riverpod, GetX, `print()`/`debugPrint()` (только логгер), хардкод
цветов/отступов (только токены). `provider` — **только** для DI, не как замена BLoC.

---

## 2. Поднять проект с нуля

```bash
# 1. Создать проект
flutter create my_app && cd my_app

# 2. Зависимости
flutter pub add flutter_bloc freezed_annotation drift drift_flutter \
  flutter_secure_storage provider talker_flutter uuid
flutter pub add --dev build_runner freezed drift_dev

# 3. Кодогенерация (после правок freezed/drift-файлов)
dart run build_runner build
# во время разработки удобнее watch — пересобирает на лету:
dart run build_runner watch
```

> ⚠️ Добавил **нативный** плагин (sqlite, secure_storage, file_picker)? Нужен полный
> `flutter run` — hot reload их не подхватывает.

### Структура папок

```
lib/
├── core/                      # инфраструктура, общая для всех фич
│   ├── database/              # Drift: tables.dart + app_database.dart
│   └── services/              # logger, secure_storage, …
├── feature/<name>/            # одна папка = одна фича
│   ├── domain/
│   │   ├── entities/          # модели (freezed, без JSON, без секретов)
│   │   └── repositories/      # АБСТРАКТНЫЙ интерфейс репозитория (порт)
│   ├── data/
│   │   ├── dtos/              # модели API (freezed + json) — когда есть бэкенд
│   │   ├── mappers/           # DTO/строка БД ↔ entity
│   │   └── repositories/      # <name>_repository_impl.dart — реализация
│   └── presentation/
│       ├── bloc/              # <name>_event.dart, _state.dart, _bloc.dart
│       ├── <name>_provider.dart   # DI: RepositoryProvider + BlocProvider
│       └── view/              # экраны
├── shared/presentation/ui/    # общие виджеты (формы, кнопки, индикаторы)
└── theme/                     # дизайн-токены (цвета/отступы/радиусы/шрифты)
```

> Имена папок не догма — но **слои `domain`/`data`/`presentation` обязательны**, и
> одна фича живёт в одной папке.

---

### 2.1 Точка входа: `main.dart` и корневой виджет

`main()` — место, где приложение «собирается»: инициализируем сервисы, кладём общие
зависимости в DI и запускаем корневой виджет.

```dart
Future<void> main() async {
  // 1. Первой строкой — до любых нативных вызовов (БД, prefs, secure storage).
  WidgetsFlutterBinding.ensureInitialized();

  // 2. Инициализируем сервисы ОДИН раз (сначала те, от кого зависят остальные).
  AppLogger.init();
  final db = AppDatabase();
  final prefs = await PrefsService.init();
  Bloc.observer = AppLogger.blocObserver;       // лог всех блоков (опционально)

  // 3. Кладём ОБЩИЕ зависимости в DI и запускаем приложение.
  runApp(
    MultiProvider(
      providers: [
        Provider<AppDatabase>.value(value: db),
        Provider<PrefsService>.value(value: prefs),
        Provider<SecureStorageService>(create: (_) => SecureStorageService()),
      ],
      child: const App(),
    ),
  );
}
```

Корневой виджет — тема + первый экран + глобальные настройки `MaterialApp`:

```dart
class App extends StatelessWidget {
  const App({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'My App',
      theme: AppTheme.light,
      darkTheme: AppTheme.dark,
      themeMode: ThemeMode.system,
      home: const NotesScreen(),     // или роутер (§11)
    );
  }
}
```

Правила точки входа:
- `WidgetsFlutterBinding.ensureInitialized()` — **первой строкой**, до `await`/нативных плагинов.
- Сервисы/БД создаём **один раз здесь** и раздаём через DI (§6, §6.1), а не по месту.
- В корневом `MultiProvider` — только **глобальные** зависимости (БД, prefs, secure storage,
  http-клиент). Фиче-специфичные репозитории/блоки — в фиче-провайдерах (§4 Шаг 6).
- Конфиг окружения (dev/prod) выбираем тоже здесь (§16).
- `main` держим тонким: инициализация + `runApp`, без бизнес-логики.

---

## 3. Три слоя — что это и зачем

Представь фичу как пирог из трёх слоёв. Каждый знает только про слой под собой.

- **domain** — «что» приложение умеет, без деталей. Тут лежат:
  - **сущности (entities)** — чистые модели данных (freezed, без JSON, без секретов);
  - **интерфейсы репозиториев** — *обещание* «я умею сохранять/читать заметки», без
    реализации (`abstract interface class`).
  - Domain ни от чего не зависит (ни от Flutter, ни от БД).
- **data** — «как» именно: реализации репозиториев на Drift/HTTP, мапперы (перевод
  строки БД ↔ сущность), DTO для API.
- **presentation** — то, что видит пользователь: BLoC (логика экрана), провайдеры
  (DI) и сами экраны/виджеты.

**Главное правило слоёв:** `presentation` → знает `domain` → знает... ничего.
`data` реализует интерфейсы из `domain`. UI **никогда** не обращается к БД/сервисам
напрямую — только через свой BLoC.

Почему так: завтра поменяешь Drift на сервер — перепишешь только `data`, а `domain`
и `presentation` не тронешь.

### 3.1 Одна фича = одна забота (дроби на мелкие фичи)

**Стремись делить функциональность на мелкие фокусные фичи.** Чем меньше зона
ответственности блока — тем проще его состояние и тем меньше багов.

Признаки, что пора дробить:
- В состоянии **две независимые оси** (меняются по отдельности) — например «данные
  формы» и «статус проверки подключения». Это две заботы → две фичи.
- Кусочек UI (кнопка, индикатор) живёт своей жизнью → отдельная фича + свой провайдер.
- Логика/виджет **повторяются** на нескольких экранах (сквозная забота) → отдельная
  переиспользуемая фича.

Что даёт: маленькое состояние легко выразить union-типом без флагов (см. §7), блоки
переиспользуются, экран собирается композицией провайдеров.

> Здравый смысл: не дроби ради дробления. Если две вещи **всегда меняются вместе** и
> по отдельности не нужны — это одна забота.

---

# Часть II. Рецепт фичи и базовые правила

## 4. Рецепт фичи (повторять для каждой сущности)

Будем делать фичу **«Заметки» (`notes`)**. Порядок всегда один:

> **entity → таблица БД → mapper → репозиторий (интерфейс + impl) → BLoC
> (event/state/bloc) → provider → экран.**

### Шаг 1. Entity (`domain/entities/note.dart`)

Чистая модель на freezed. Только публичные поля. Без JSON и без секретов.

```dart
import 'package:freezed_annotation/freezed_annotation.dart';

part 'note.freezed.dart';

/// Доменная сущность заметки.
@freezed
abstract class Note with _$Note {
  const factory Note({
    required String id,
    required String title,
    required DateTime createdAt,
    @Default('') String body,          // значение по умолчанию
    @Default(false) bool pinned,
    @Default(<Tag>[]) List<Tag> tags,  // вложенные сущности списком
  }) = _Note;
}

/// Вложенная сущность (для примера композиции мапперов ниже).
@freezed
abstract class Tag with _$Tag {
  const factory Tag({required String id, required String name}) = _Tag;
}
```

Правила: `@freezed` + `abstract class X with _$X` + один `const factory`. Обязательные
поля — `required`, опциональные — `?`, дефолты — `@Default(...)`. Док-комментарий `///`.

### Шаг 2. DTO, таблица БД и Mapper (`data/`)

У проекта два внешних источника: **сервер (DTO)** и **локальная БД (Drift)**. Под
каждый — своя модель данных в `data`; обе превращаются в доменную сущность **одним
маппером**.

**DTO** — модель ответа API (freezed + JSON). Поля «как пришло с сервера» (даты строками).

```dart
// data/dtos/note_dto.dart
@freezed
abstract class NoteDto with _$NoteDto {
  const factory NoteDto({
    required String id,
    required String title,
    required String body,
    required String createdAt,            // строка с сервера — распарсим в маппере
    @Default(<TagDto>[]) List<TagDto> tags,
  }) = _NoteDto;

  factory NoteDto.fromJson(Map<String, dynamic> json) => _$NoteDtoFromJson(json);
}
// TagDto объявляется так же.
```

**Таблица БД (Drift)** — у строки своё имя `NoteRow`, чтобы не путать с сущностью `Note`.

```dart
// core/database/tables.dart
@DataClassName('NoteRow')
class Notes extends Table {
  TextColumn get id => text()();
  TextColumn get title => text()();
  TextColumn get body => text().withDefault(const Constant(''))();
  DateTimeColumn get createdAt => dateTime()();
  @override
  Set<Column> get primaryKey => {id};
}
```

**Mapper** — `abstract class` со статическими методами: **все преобразования одной
сущности — в одном классе**. Тело — стрелка `=>`, параметр именованный.

> ⚠️ Dart **не умеет перегрузку** по типу параметра — двух методов `toEntity` быть не
> может. Поэтому имена уникальны: сервер — `toEntity`/`toDto`; БД — `fromRow`/`toCompanion`.

```dart
// data/mappers/note_mapper.dart
abstract class NoteMapper {
  // — — — Сервер: DTO ↔ Entity — — —

  /// DTO → Entity (ответ API → доменная модель).
  static Note toEntity({required NoteDto dto}) => Note(
    id: dto.id,
    title: dto.title,
    body: dto.body,
    createdAt: DateTime.parse(dto.createdAt),
    // вложенные сущности — свой маппер; список — через .map:
    tags: dto.tags.map((tagDto) => TagMapper.toEntity(dto: tagDto)).toList(),
  );

  /// Entity → DTO — только когда нужно отправить на сервер (тело запроса).
  static NoteRequestDto toDto({required Note note}) => NoteRequestDto(
    title: note.title,
    body: note.body,
    tagIds: note.tags.map((tag) => tag.id).toList(),
  );

  // — — — Локальная БД: строка ↔ Entity — — —

  /// Строка БД → Entity.
  static Note fromRow({required NoteRow row}) => Note(
    id: row.id,
    title: row.title,
    body: row.body,
    createdAt: row.createdAt,
  );

  /// Entity → companion для вставки/обновления.
  static NotesCompanion toCompanion({required Note note}) => NotesCompanion.insert(
    id: note.id,
    title: note.title,
    body: Value(note.body),
    createdAt: note.createdAt,
  );
}

/// Каждая вложенная сущность — отдельный маппер (со своими методами по нужде).
abstract class TagMapper {
  static Tag toEntity({required TagDto dto}) => Tag(id: dto.id, name: dto.name);
}
```

Правила маппера:
- `abstract class <Имя>Mapper` — **один класс на сущность**, все её преобразования внутри.
  (Если у фичи только сервер — допустимо `<Имя>DtoMapper`.)
- Методы `static`, **тело-стрелка `=>`**, параметр именованный (`{required ... }`).
- **Имена уникальны** (Dart без перегрузки): сервер — `toEntity({required dto})` /
  `toDto({required entity})`; локальная БД — `fromRow({required row})` /
  `toCompanion({required entity})` (своя DAO-модель — по аналогии `fromDao` / `toDao`).
- Обратные направления (`toDto`/`toCompanion`) заводим **только по необходимости**.
- **Каждая вложенная сущность — свой маппер**; композиция `Child.toEntity(dto: ...)`,
  списки — `.map((dto) => Child.toEntity(dto: dto)).toList()`.
- Доп. данные (например родительский id) — дополнительными именованными `required`-параметрами.
- Несколько связанных мапперов — в одном файле. Парсинг (даты, enum через `.values[...]`,
  склейка `searchKeys`/`filterKeys`) внутри маппера допустим. Мёртвый код не оставляем,
  ошибки парсинга молча не глотаем.

### Шаг 3. Репозиторий: интерфейс (domain) + реализация (data)

Сначала **обещание** (порт) — `abstract interface class`, каждый метод с `///`:

```dart
// domain/repositories/note_repository.dart
abstract interface class NoteRepository {
  /// Реактивный список заметок (сам обновляется при изменениях в БД).
  Stream<List<Note>> watchAll();

  /// Разовое чтение по id (или null).
  Future<Note?> getById(String id);

  /// Сохранить (создать или обновить).
  Future<void> save(Note note);

  /// Удалить по id.
  Future<void> delete(String id);
}
```

Затем **реализация** — `implements`, зависимости именованными `required`-параметрами,
поля `final` с `_` (см. §6 про DI):

```dart
// data/repositories/note_repository_impl.dart
class NoteRepositoryImpl implements NoteRepository {
  final AppDatabase _db;

  NoteRepositoryImpl({required AppDatabase db}) : _db = db;

  @override
  Stream<List<Note>> watchAll() {
    final query = _db.select(_db.notes)..orderBy([(t) => OrderingTerm.desc(t.createdAt)]);
    return query.watch().map((rows) => rows.map((r) => NoteMapper.fromRow(row: r)).toList());
  }

  @override
  Future<void> save(Note note) =>
      _db.into(_db.notes).insertOnConflictUpdate(NoteMapper.toCompanion(note: note));

  // ... getById / delete
}
```

**Соглашение по именам:** `watch*` → `Stream` (подписка), `get*` → `Future` (разовый ответ).

#### Future или Stream — зависит от задачи

- **`Future<List<Note>> getAll()`** — спросил один раз, получил список **на этот
  момент**. Хочешь свежий — спрашиваешь снова.
- **`Stream<List<Note>> watchAll()`** — **подписка**: источник сам шлёт новый список
  **каждый раз, когда данные меняются**. Подписался один раз — данные «текут» сами.

**Зачем нужен реактивный `watchAll()`.** Посмотри на блок из Шага 5: события
`NoteSaved`/`NoteDeleted` просто пишут в БД и **ничего не делают со списком**:

```dart
NoteSaved(:final note) => _repository.save(note),   // только записали
NoteDeleted(:final id) => _repository.delete(id),    // только удалили
```

А список обновляется сам — через подписку `emit.forEach(_repository.watchAll(), ...)`.
Записали → Drift заметил изменение таблицы → `watchAll()` выдал новый список → блок
эмитнул `NotesLoaded(новый список)` → UI перерисовался. Что это даёт:
1. **Единый источник правды — база.** UI всегда показывает то, что реально в БД.
2. **Нет ручного refresh** после каждого `save`/`delete` — иначе легко забыть и
   показать устаревшее.
3. **Экраны синхронны сами.** Удалил на одном экране — список на другом (подписан на
   тот же `watchAll`) обновится без событий между ними.

**Что выбрать:**

| Задача | Метод |
|---|---|
| Список/данные, которые меняются и видны на экране (локальная БД) | `Stream watchAll()` |
| Разовое чтение (один объект по id, проверка, экспорт) | `Future getById()` / `getAll()` |
| **Сервер (REST API)** — подписаться нельзя | `Future` + **ручное** обновление после изменений |
| «Живые» данные с сервера | WebSocket / polling (отдельный механизм) |

Реактивность здесь — заслуга **локальной БД**: Drift умеет `.watch()`. У REST API
такого нет (endpoint не «подписываемый») — там список грузится `Future`-методом, а
после `save`/`delete` репозиторий/блок перезапрашивает данные вручную. Поэтому правило
«список — реактивно» относится к **локальной БД**; для сервера — `Future` + ручной
рефреш.

### Шаг 4. Секреты (если есть)

Пароли, токены, приватные ключи — **только** через `SecureStorageService`
(`flutter_secure_storage`), ключ строится из id сущности. Никогда не в обычной БД и
не в entity. Если у заметок секретов нет — этот шаг пропускаем.

### Шаг 5. BLoC

Три файла в `presentation/bloc/`. Подробные правила — в §5.

```dart
// notes_event.dart — события на freezed (union)
@freezed
sealed class NotesEvent with _$NotesEvent {
  const factory NotesEvent.subscriptionRequested() = NotesSubscriptionRequested;
  const factory NotesEvent.saved(Note note) = NoteSaved;
  const factory NotesEvent.deleted(String id) = NoteDeleted;
}
```

```dart
// notes_state.dart — состояние как union ФАЗ (не флаги!)
@freezed
sealed class NotesState with _$NotesState {
  const factory NotesState.loading() = NotesLoading;
  const factory NotesState.loaded(List<Note> notes) = NotesLoaded;
  const factory NotesState.error(String message) = NotesError;
}
```

```dart
// notes_bloc.dart — один on<> со switch
class NotesBloc extends Bloc<NotesEvent, NotesState> {
  final NoteRepository _repository;

  NotesBloc({required NoteRepository repository})
      : _repository = repository,
        super(const NotesLoading()) {
    on<NotesEvent>(
      (event, emit) => switch (event) {
        NotesSubscriptionRequested() => _onSubscription(emit),
        NoteSaved(:final note) => _repository.save(note),
        NoteDeleted(:final id) => _repository.delete(id),
      },
    );
  }

  Future<void> _onSubscription(Emitter<NotesState> emit) {
    return emit.forEach<List<Note>>(
      _repository.watchAll(),
      onData: NotesLoaded.new,
      onError: (e, st) {
        AppLogger.instance.error('Не удалось загрузить заметки', e, st);
        return NotesError(e.toString());
      },
    );
  }
}
```

### Шаг 6. Provider (`presentation/notes_provider.dart`)

`StatelessWidget`, оборачивающий `child`: собирает репозиторий и блок, стартует
подписку. Экран про БД ничего не знает.

```dart
class NotesProvider extends StatelessWidget {
  const NotesProvider({required this.child, super.key});

  final Widget child;

  @override
  Widget build(BuildContext context) {
    return RepositoryProvider<NoteRepository>(
      create: (context) => NoteRepositoryImpl(db: context.read<AppDatabase>()),
      child: BlocProvider<NotesBloc>(
        create: (context) =>
            NotesBloc(repository: context.read<NoteRepository>())
              ..add(const NotesSubscriptionRequested()),
        child: child,
      ),
    );
  }
}
```

Правила: сначала `RepositoryProvider`, внутри `BlocProvider`. **Стартовое событие
диспатчим здесь** (`..add(...)`), а НЕ в конструкторе блока.

### Шаг 7. Экран (коннектор + UI-виджеты)

См. §7. Публичный `NotesScreen` оборачивает приватный `_NotesView` в провайдер;
`_NotesView` — коннектор с `BlocBuilder`; вёрстка — в отдельных UI-виджетах.

---

## 5. BLoC — самые важные правила

- **Полноценный `Bloc`, не Cubit.**
- **Ровно один** `on<BaseEvent>` со `switch` по типам событий. НЕ делай отдельный
  `on<>` на каждое событие.
  ```dart
  on<NotesEvent>((event, emit) => switch (event) {
    NotesSubscriptionRequested() => _onSubscription(emit),
    NoteDeleted(:final id)       => _repository.delete(id),  // деструктуризация поля
  });
  ```
- **События — `@freezed sealed class`** с union-конструкторами. Имя redirect-класса
  (`= NoteDeleted`) — то, по которому матчим в `switch` и которым создаём событие.
- **Состояние — `@freezed sealed class` = union ФАЗ.** Загрузка, успех и ошибка —
  это РАЗНЫЕ состояния. ❌ Запрещено `bool loading`/`String? error` полями в одном
  состоянии и переключение поведения по флагу.
  - Если данные должны **переживать смену фазы** (список виден и во время фоновой
    дозагрузки) — выноси их в отдельный `@freezed`-объект и передавай в нужные
    варианты.
  - Доменный статус (например, `connecting/live/error` живого соединения) — это
    *данные*, его можно держать полем; UI-флагом-фазой он быть не должен.
- Конструктор — именованные `required`-параметры; зависимости в `final _`-поля.
- **НЕ** диспатчить стартовое событие в конструкторе — это делает provider.
- Загрузка данных — **по источнику** (см. §4 Шаг 3): локальная БД → реактивная
  подписка `emit.forEach(repo.watchAll(), onData:, onError:)`; сервер (API) →
  `Future`-загрузка `emit(loading) → try { emit(loaded(await repo.getAll())) } catch { emit(error) }`.
- Каждое событие обрабатывает приватный метод `_onXxx`. Ошибки — `try { } on Object
  catch (e, st) { AppLogger... }` или в `onError`. Молча не глотать.

---

## 6. Внедрение зависимостей (DI) — единый стиль ВЕЗДЕ

Касается всех классов с зависимостями: репозитории, блоки, сервисы, провайдеры данных.

```dart
class NoteRepositoryImpl implements NoteRepository {
  // 1. Поля — приватные final, объявлены первыми.
  final AppDatabase _db;

  // 2. Конструктор — именованные required-параметры (имена БЕЗ подчёркивания).
  // 3. Присвоение — в списке инициализации.
  NoteRepositoryImpl({required AppDatabase db}) : _db = db;
}
```

- ✅ Именованные `required`-параметры — вызов читаемый и не ломается при перестановке.
- ✅ Присвоение через список инициализации `: _x = x`. Параметр без `_`, поле с `_`.
- ❌ Позиционный конструктор и `this._x` (`Impl(this._db)`) — **запрещено**.
- Зависимость, создаваемая внутри (не инжектится), — `final x = X();` прямо в поле.
- Исключение: приватные конструкторы внутреннего создания (`._(...)`, обёртки) —
  не provider-инъекция, под правило не подпадают.

### 6.1 Сервисы — оборачивай переиспользуемые внешние зависимости

**Если к внешнему инструменту обращаешься больше одного раза — оберни его в сервис**
(класс с понятным API) и предоставь **один раз через DI**. Кандидаты: `SharedPreferences`,
secure storage, HTTP-клиент, БД, логгер, аналитика, геолокация, файлы.

Антипример (как НЕ надо), типичная боль:

```dart
// ❌ В каждом месте заново создаём подключение и дёргаем сырой API вне DI.
final prefs = await SharedPreferences.getInstance();   // снова и снова
await prefs.setBool('is_first_launch', false);          // строковые ключи раскиданы по коду
```

Проблемы: подключение пересоздаётся при каждом обращении; ключи-строки дублируются и
легко опечататься; зависимость живёт вне DI, её не подменить в тестах, не переиспользовать.

Как надо — сервис + DI:

```dart
/// Сервис настроек приложения поверх SharedPreferences.
class PrefsService {
  final SharedPreferences _prefs;
  PrefsService({required SharedPreferences prefs}) : _prefs = prefs;

  // Инициализируем подключение ОДИН раз — в main(), до runApp.
  static Future<PrefsService> init() async =>
      PrefsService(prefs: await SharedPreferences.getInstance());

  bool get isFirstLaunch => _prefs.getBool(_kFirstLaunch) ?? true;
  Future<void> setFirstLaunch(bool v) => _prefs.setBool(_kFirstLaunch, v);

  static const _kFirstLaunch = 'is_first_launch';  // ключи — внутри сервиса, в одном месте
}
```

```dart
// main.dart — создаём один раз и кладём в DI:
final prefs = await PrefsService.init();
runApp(
  MultiProvider(
    providers: [Provider<PrefsService>.value(value: prefs)],
    child: const App(),
  ),
);
// дальше везде: context.read<PrefsService>().isFirstLaunch
```

Что это даёт:
- **Подключение/инициализация — один раз** (не `getInstance()` на каждый чих).
- **Единый API и ключи в одном месте** — нет строк-ключей по всему коду.
- **Внутри DI** — легко переиспользовать, подменить в тестах, поменять реализацию.
- Сложный сервис — как репозиторий: **`abstract interface` (порт) + реализация**; простой
  (тонкая обёртка) — допустим одним классом. Живёт в `core/services/` (общий) или в фиче.

---

## 7. UI: коннектор и чистые виджеты

Иерархия: **Screen (обёртка с Provider) → View-коннектор (`Bloc*`) → UI-виджеты
(данные через конструктор) → Section → Card → Tile.**

### 7.1 Разделяй коннектор и UI-виджеты

Состоянием рулит **тонкий виджет-коннектор** (`BlocBuilder`/`BlocListener`/
`BlocConsumer`): читает состояние, достаёт данные, выбирает, какой UI-виджет показать
и с какими данными. Вёрстка — в **отдельных чистых UI-виджетах**, которые получают
данные через конструктор и про блок ничего не знают.

```dart
// Экран = провайдер + коннектор.
class NotesScreen extends StatelessWidget {
  const NotesScreen({super.key});
  @override
  Widget build(BuildContext context) => const NotesProvider(child: _NotesView());
}

// Коннектор: только Bloc* + (state → данные) + выбор UI-виджета. Без тяжёлой вёрстки.
class _NotesView extends StatelessWidget {
  const _NotesView();
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: BlocBuilder<NotesBloc, NotesState>(
        builder: (context, state) => switch (state) {
          NotesLoading() => const Center(child: CircularProgressIndicator()),
          NotesError(:final message) => _ErrorView(message: message),
          NotesLoaded(:final notes) => _NotesList(
            notes: notes,
            onDelete: (id) => context.read<NotesBloc>().add(NoteDeleted(id)),
          ),
        },
      ),
    );
  }
}

// UI-виджет: чистый. Данные и колбэки через конструктор, про Bloc ничего не знает.
class _NotesList extends StatelessWidget {
  const _NotesList({required this.notes, required this.onDelete});
  final List<Note> notes;
  final ValueChanged<String> onDelete;
  @override
  Widget build(BuildContext context) {
    /* только вёрстка по notes; действие — через onDelete */
  }
}
```

Правила:
- **Коннектор**: `Bloc*` + маппинг `state → данные` + выбор UI-виджета. Никакой
  тяжёлой вёрстки внутри `builder`.
- **UI-виджет**: данные/колбэки через конструктор (`final`); **никаких**
  `context.read`/`watch`/`BlocBuilder` внутри. Так он переиспользуем и тестируем.
- Действия наверх — **колбэками** (`VoidCallback`, `ValueChanged<T>`); `bloc.add(...)`
  зовёт коннектор.
- Сайд-эффекты (навигация, снэкбары) — в `BlocListener` коннектора, не в UI-виджете.
- Контроллеры (`TextEditingController`) освобождай в `dispose()`.
- Возврат из модалок — `Navigator.pop(context, value)`; несколько значений —
  record-кортежем: `Navigator.pop(context, (title: t, body: b))`.

### 7.2 Виджеты: что предпочитать, чего избегать

**Выноси куски UI в отдельные виджеты-классы, а НЕ в методы `Widget _buildX()`.**
Класс можно сделать `const` — Flutter не пересобирает его зря; метод пересобирается
каждый раз с родителем и не оптимизируется.

```dart
// ❌ так — метод, возвращающий виджет
Widget _buildHeader() => Row(children: [...]);
// ✅ так — отдельный виджет-класс (по возможности const)
class _Header extends StatelessWidget { const _Header(); @override Widget build(...) {...} }
```

**`const` где возможно** (`const SizedBox(height: 8)`, `const Text('...')`) — меньше перестроек.

| Не используй | Используй | Почему |
|---|---|---|
| `Container(color: …)` ради одного цвета | `ColoredBox` / `DecoratedBox` | `Container` тяжелее; бери его только когда нужно несколько свойств сразу |
| `Container(padding: …)` ради отступа | `Padding` | то же |
| `Container(width/height: …)` ради размера/зазора | `SizedBox` | то же |
| `Opacity(...)` для статичной прозрачности | `color.withValues(alpha: …)` | `Opacity` создаёт отдельный слой композиции (дорого); виджет — только для анимации |
| голый `GestureDetector` для тапа | `InkWell` / `TextButton` / `IconButton` | ripple, доступность, нормальная зона нажатия |
| `ListView(children: [всё сразу])` для длинных списков | `ListView.builder` | ленивый рендер видимых элементов |
| `ListView`/`GridView` прямо в `Column` без ограничений | `Expanded`-обёртка или slivers | иначе «unbounded height»-ошибка |
| `shrinkWrap: true` для длинного списка | `CustomScrollView` + slivers | `shrinkWrap` отключает ленивость |
| `MediaQuery.of(context).size` | `MediaQuery.sizeOf(context)` / `LayoutBuilder` | `.of` перестраивает на любое изменение MediaQuery |
| `FutureBuilder`/`StreamBuilder` для данных БД/сети | BLoC (см. §7.1) | бизнес-логика и состояние — в блоке, не в виджете |
| `setState` для бизнес-данных | BLoC | `setState` — только для чисто UI-состояния (раскрыт/свёрнут, контроллеры) |
| `Stack`+`Positioned` для обычной раскладки | `Row`/`Column`/`Align` | `Stack` — только для реального наложения |

- `Expanded`/`Flexible` — **только** внутри `Row`/`Column`/`Flex`; вне — ошибка.
- Длинные прокручиваемые экраны со смешанным контентом — `CustomScrollView` + `Sliver*`.

---

## 8. Дизайн-токены (вместо хардкода)

❌ Никаких «магических» цветов/чисел в виджетах. ✅ Только из единых токенов.

```dart
final colors = context.appColors;          // ThemeExtension-расширение
Container(color: colors.surface);
Padding(padding: const EdgeInsets.all(AppSpacing.lg));
BorderRadius.circular(AppRadius.md);
Text('Привет', style: AppTypography.body);
```

- **Цвета** — семантические (`context.appColors.text/surface/primary/...`).
  Прозрачность — `color.withValues(alpha: 0.18)`, **не** `withOpacity`.
- **Отступы** — `AppSpacing.xs/sm/md/lg/xl/...` (шкала кратна 4).
- **Радиусы** — `AppRadius.sm/md/lg/full`. **Текст** — стили из `AppTypography`.
- Нужного токена нет → добавь его в систему токенов, а не пиши число.

---

## 9. Логирование и ошибки

```dart
AppLogger.instance.info('Что-то произошло');
AppLogger.instance.warning('Подозрительно');
AppLogger.instance.error('Что не удалось сделать', error, stackTrace);
```

- Логируем **только** через логгер (синглтон на `talker`). `print`/`debugPrint`
  запрещены.
- Сообщение — описание действия: «Не удалось загрузить заметки».
- Ошибки ловим `on Object catch (e, st)` (или в `onError` стрима) и **всегда** логируем.
  Никаких пустых `catch {}`.

---

## 10. Правила написания кода

**Именование**
- Классы/типы — `UpperCamelCase`; переменные, методы, параметры, константы —
  `lowerCamelCase`; приватное — с `_`. Файлы — `snake_case.dart`.
- Имена по смыслу: `noteRepository`, а не `repo2`/`data`/`tmp`. Булевы — с `is/has/can`
  (`isLoading`, `hasError`, `canSubmit`).
- Без сокращений-загадок и транслита (`usr`, `spisok`). Аббревиатуры как слово: `httpClient`.

**Переменные и типы**
- `final` по умолчанию для всего; `var` — только когда тип очевиден из правой части;
  `late final` — для инициализации из конструктора/`widget`. `var`-перемен без нужды нет.
- **`const` везде, где можно** (конструкторы событий, `EdgeInsets`, виджеты) — это и стиль,
  и производительность.
- Типы у публичного API указываем явно; не плоди `dynamic`.

**Null-safety**
- Избегай `!` (null-assertion). Обрабатывай null явно: `?.`, `??`, `if (x != null)`,
  pattern matching. `!` — только когда null логически невозможен и это очевидно из кода.
- Не делай поля nullable «на всякий случай» — `?` только если null реально осмыслен.

**Поток управления**
- **Ранний возврат (guard)** вместо глубокой вложенности `if`:
  `if (x == null) return;` в начале, а не оборачивание всего тела в `if`.
- `switch`-выражения для маппинга значений вместо лестниц `if (x is ...)`/`else if`.
- Не больше одного вложенного тернарника; сложное ветвление — `switch` или отдельный метод.
- Маленькие функции/методы: одна задача. Длинный метод или `build` → разбей на части
  (методы для логики, **виджеты-классы** для UI — см. §7.2).

**Асинхронность**
- Всегда `await` у `Future` (или осознанно `unawaited(...)`); не теряй ошибки и результат.
- `async` без единого `await` — не нужен. Ошибки — `try { } on Object catch (e, st)` + лог.

**Комментарии и чистота**
- `///` — публичный API (классы/методы), `//` — внутреннее. Группировка — `// — — — … — — —`.
- Пиши **зачем**, а не пересказ кода. Без закомментированного «мёртвого» кода и пустых TODO.
- Именованные параметры для конструкторов с >1 аргумента; `required` где обязательно.

**Прочее**
- Сгенерированные файлы (`*.freezed.dart`, `*.g.dart`) **не редактировать руками**.
- После правок freezed/drift → `dart run build_runner build`.
- Перед коммитом → `flutter analyze` без ошибок (новые предупреждения не игнорируем).

---

# Часть III. Углублённые темы (по мере роста проекта)

## 11. Навигация / роутинг

- Небольшое приложение — хватает `Navigator` (`push`/`pop`). Нужны deep links / web /
  сложные стеки — бери `go_router` (маршруты в одном месте).
- Аргументы экрана — **через конструктор** (типобезопасно), не через `settings.arguments` с `as`.
  ```dart
  final result = await Navigator.push<bool>(            // ждём результат
    context, MaterialPageRoute(builder: (_) => NoteEditScreen(note: note)),
  );
  // на том экране: Navigator.pop(context, true);
  ```
- **Прокинуть существующий BLoC на новый экран** — `BlocProvider.value` (через `push`
  новый экран попадает в новое дерево, провайдер сверху НЕ наследуется):
  ```dart
  Navigator.push(context, MaterialPageRoute(
    builder: (_) => BlocProvider.value(value: context.read<NotesBloc>(), child: const NoteScreen()),
  ));
  ```
- Решение «куда идти» — в обработчике/`BlocListener` коннектора, не размазано по вёрстке.

---

## 12. Сеть / API: HTTP-запросы (`dio`)

Когда у приложения появляется бэкенд, весь обмен по HTTP живёт в **data-слое**: тонкий
api-сервис делает запросы и отдаёт **DTO**, репозиторий мапит DTO в доменные сущности
(§4). Выше data-слоя сырые `Map`/`Response`/`DioException` **не поднимаются** — UI и
блок про HTTP ничего не знают (как и про БД).

### 12.1 Один настроенный `Dio` в DI

Создаём **один** `Dio` с baseUrl, таймаутами и интерсепторами и кладём в DI. `Dio()` на
каждый запрос — запрещено (теряются настройки/интерсепторы, лишние аллокации).

```dart
Dio buildDio(AppConfig cfg) => Dio(
  BaseOptions(
    baseUrl: cfg.apiUrl,                         // из конфига окружения (§16), не хардкод
    connectTimeout: const Duration(seconds: 15),
    receiveTimeout: const Duration(seconds: 20),
    sendTimeout: const Duration(seconds: 20),
    headers: {'Content-Type': 'application/json'},
  ),
)
  ..interceptors.add(AuthInterceptor(secrets: secrets))   // порядок важен — см. §12.3
  ..interceptors.add(AppLogger.dioLogger);                // логгер — последним
```

```dart
// main.dart — создаём один раз и раздаём через DI:
Provider<Dio>(create: (_) => buildDio(cfg)),
```

- **Таймауты обязательны.** Без них зависший сокет держит запрос вечно (а пользователь —
  бесконечный спиннер). `connectTimeout`/`receiveTimeout`/`sendTimeout` — в `BaseOptions`.
- baseUrl и общие заголовки — в `BaseOptions`, а не на каждом вызове.

### 12.2 Api-сервис (data-provider) → DTO

HTTP-вызовы оборачиваем в сервис фичи (`data/services/` или `data/providers/`). Он знает
про эндпоинты и DTO, **не знает** про БД/виджеты. Возвращает **DTO**; в доменную сущность
мапит репозиторий (§4).

```dart
class NoteApi {
  final Dio _dio;
  NoteApi({required Dio dio}) : _dio = dio;

  Future<List<NoteDto>> fetchNotes() async {
    final resp = await _dio.get<dynamic>('/v1/notes');
    final list = _unwrap(resp.data) as List;                  // §12.4
    return list.cast<Map<String, dynamic>>().map(NoteDto.fromJson).toList();
  }

  Future<NoteDto> createNote(NoteRequestDto body) async {
    final resp = await _dio.post<dynamic>('/v1/notes', data: body.toJson());
    return NoteDto.fromJson(_unwrap(resp.data) as Map<String, dynamic>);
  }
}
```

- Путь, query (`queryParameters:`), тело (`data:`) — здесь. Тело — `toJson()` из DTO, а
  не россыпь строковых ключей по коду.
- Один метод = один эндпоинт; парсинг ответа в DTO — тут же. Сборка URL — через
  `queryParameters`, не конкатенацией строк (dio сам экранирует).

### 12.3 Интерсепторы: токен, refresh, логирование

Сквозные заботы (одно и то же на каждом запросе) — в интерсепторы, а не копипастой по
методам.

- **Авторизация** — в `onRequest` дописать `Authorization: Bearer <token>` из secure storage.
- **Refresh на 401** — в `onError` при `401` обновить access по refresh-токену и **повторить**
  исходный запрос; параллельные 401 синхронизировать (один refresh, остальные ждут его),
  чтобы не рефрешить N раз. Refresh не удался → разлогинить (почистить токены) и отдать
  понятную ошибку «сессия истекла».
- **Логирование** — `talker_dio_logger` (`AppLogger.dioLogger`), добавлять **последним**.

```dart
class AuthInterceptor extends Interceptor {
  final SecureStorageService _secrets;
  AuthInterceptor({required SecureStorageService secrets}) : _secrets = secrets;

  @override
  Future<void> onRequest(options, handler) async {
    final token = await _secrets.getToken();
    if (token != null) options.headers['Authorization'] = 'Bearer $token';
    handler.next(options);
  }

  @override
  Future<void> onError(err, handler) async {
    if (err.response?.statusCode == 401 && await _refreshOnce()) {
      return handler.resolve(await _retry(err.requestOptions));   // повтор с новым токеном
    }
    handler.next(err);
  }
}
```

> Эквивалент — refresh внутри обёртки `_send()` api-сервиса (см. реальный `CloudApiService`
> в beacon). Главное: **refresh в одном месте**, а не на каждом вызове. При **ротации**
> refresh-токена (старый инвалидируется при обновлении) сохраняй новую пару целиком.

### 12.4 Распаковка ответа: envelope `{success, data}`

Многие API заворачивают полезную нагрузку: `{"success": true, "data": {...}}` или
`{"data": [...], "meta": {...}}`. Держи распаковку в **одном хелпере**, а не повторяй
`resp.data['data']` в каждом методе.

```dart
/// Достаёт нагрузку из конверта `{success, data}` (или отдаёт объект как есть).
static Object? _unwrap(dynamic raw) {
  if (raw is Map && raw['data'] != null) return raw['data'];
  return raw;
}
```

Сменился формат — правишь одно место. (Из жизни: BFF-шлюз отдаёт `{success, data}`, а
часть «родных» ручек — голый объект; один `_unwrap` покрывает оба случая.)

### 12.5 Ошибки сети: `DioException` → доменная ошибка

`DioException` ловим в data-слое и переводим в доменную ошибку (§13) — наружу не
выпускаем. Тип сбоя — в `DioExceptionType`:

```dart
Never _mapError(DioException e) {
  final message = switch (e.type) {
    DioExceptionType.connectionTimeout ||
    DioExceptionType.sendTimeout ||
    DioExceptionType.receiveTimeout => 'Тайм-аут соединения',
    DioExceptionType.connectionError => 'Нет связи. Проверьте интернет.',
    DioExceptionType.badResponse => _describeStatus(e.response?.statusCode, e.response?.data),
    _ => 'Ошибка сети',
  };
  AppLogger.instance.error('Сбой запроса ${e.requestOptions.uri}', e, e.stackTrace);
  throw NetworkFailure(message);     // доменная ошибка, НЕ e.toString() пользователю
}
```

- Текст из тела ошибки (`{"error": {"message": ...}}` или `{"error": "..."}`) парсим в
  одном месте; пользователю — человеческое сообщение, в лог — стек (§13).
- `4xx` (валидация/права — обычно не повторяемые) и `5xx`/таймаут (повторяемые) — разные
  ветки: первые показываем как есть, вторые можно ретраить (§12.7).

### 12.6 Отмена запроса (`CancelToken`)

Запрос, чей результат больше не нужен (ушли с экрана, начался новый поиск), — отменяй:
экономит трафик и спасает от «emit после close».

```dart
final _cancel = CancelToken();
_dio.get('/v1/search', queryParameters: {'q': q}, cancelToken: _cancel);
// при отмене/закрытии блока:
_cancel.cancel('экран закрыт');
```

`DioExceptionType.cancel` — это **не** ошибка для пользователя: проглатываем молча (можно
залогировать debug). Для живого поиска удобнее `restartable()` transformer (§15) + новый
`CancelToken` на каждый ввод.

### 12.7 Повторы, идемпотентность, файлы

- **Ретрай** — только для **идемпотентных** запросов (GET) и сетевых сбоев/5xx, с
  экспоненциальным бэкоффом и лимитом попыток. POST/платёж без ключа идемпотентности
  вслепую не повторяем (риск дубля).
- **Загрузка/выгрузка файлов** — `FormData`/`MultipartFile`; прогресс — `onSendProgress`/
  `onReceiveProgress`. Большие файлы — стримом, не целиком в память.
- **Заголовки/таймаут на конкретный вызов** — `Options(headers: ..., receiveTimeout: ...)`.

### 12.8 Типизированный клиент (`retrofit`) — опционально

Когда эндпоинтов много, ручные методы заменяет `retrofit` (кодоген поверх dio):
аннотируешь интерфейс — генерится реализация, возвращающая DTO.

```dart
@RestApi()
abstract class NoteClient {
  factory NoteClient(Dio dio) = _NoteClient;
  @GET('/v1/notes') Future<List<NoteDto>> fetchNotes();
  @POST('/v1/notes') Future<NoteDto> create(@Body() NoteRequestDto body);
}
```

Слой тот же (api → DTO → маппер), просто меньше ручного парсинга. Для пары запросов
проще ручной сервис; для большого API — `retrofit`.

### 12.9 Живые данные: WebSocket / SSE / polling

REST-эндпоинт «подписать» нельзя (§4): для данных, которые приходят **сами** (метрики,
статусы, чат, уведомления), нужен отдельный канал. Главная идея — **обернуть его в сервис,
который отдаёт `Stream<T>` доменных сущностей**, и тогда блок подписывается на него точно
так же, как на реактивный `watchAll()` из БД.

- **WebSocket** (`web_socket_channel`) — двунаправленный поток. Сервис парсит кадр
  (JSON → DTO → entity) и публикует в `Stream<T>`; исходящие сообщения — методом `send`.
- **SSE / long-polling** — односторонний поток событий; та же обёртка `Stream<T>`.
- **Periodic polling** — если живого канала нет: `Timer.periodic` дёргает `Future`-метод
  api-сервиса; интервал — настройка, тикер **прерываемый** и закрывается в `close()`.

Правила живого канала:
- **Один канал на ресурс** через app-level сервис (как единый `Dio`), с подсчётом
  подписчиков: открываем на первого, закрываем на последнего. Не плоди по сокету на экран.
- **Авто-reconnect с бэкоффом** при обрыве; **не спамь лог** на каждой попытке (одна
  строка на обрыв). Реагируй на жизненный цикл приложения (ушли в фон → приостанови/закрой,
  вернулись → подними).
- **broadcast**-поток, если слушателей несколько; всегда отписывайся
  (`StreamSubscription.cancel()`) в `close()`/`dispose()` (§24).
- Токен для WS передавай в query или первом сообщении (произвольные заголовки в вебсокете
  ограничены).

Блок остаётся **тонким подписчиком** — `emit.forEach(service.watch(id), onData: ...)`,
ровно как с реактивной БД; меняется только источник потока. (Пример из жизни: beacon
держит один аккаунт-WS на все машины и раздаёт метрики/статусы по `server_id` через хаб.)

---

## 13. Обработка ошибок: доменные `Failure` + показ пользователю

- Технические исключения (`DioException`, `SocketException`, парсинг) лови в data/репозитории
  и **переводи в доменные ошибки**, понятные приложению.
- **Два слоя ошибки:** что *показать пользователю* (человеческое сообщение в `error`-состоянии)
  и что *только залогировать* (stacktrace в `AppLogger`/Sentry). Юзеру не показываем
  `e.toString()`/стек.
- Подходы:
  - **A (проще для новичка):** бросать доменное исключение (`NetworkFailure`, `NotFoundFailure`),
    блок ловит → `emit(error('Нет связи. Проверьте интернет.'))`.
  - **B:** `sealed Result<T>` = `Success(data)` / `Failure(message)` — явный результат без
    исключений.
- Перевод «код/тип ошибки → текст» — в одном месте (как `_describeErrorCode`), не дублировать.

---

## 14. Кэш и оффлайн: один источник правды (SSOT)

Гибрид «сервер + локальная БД». Правило: **UI всегда читает из локальной БД** (реактивно,
`watch`), а сеть только **обновляет БД**.

```dart
// Репозиторий
Stream<List<Note>> watchAll();        // ← UI подписан сюда (из БД)
Future<void> refresh();               // сходить в сеть и записать в БД

// impl
Future<void> refresh() async {
  final dtos = await _api.getNotes();                 // сеть
  await _db.upsertAll(dtos.map((d) => NoteMapper.toCompanion(...)));  // пишем в БД
  // watchAll() сам толкнёт свежие данные в UI
}
```

Зачем: работает **оффлайн** (показываем последнее сохранённое), **один источник правды**
(нет рассинхрона «список из сети vs из БД»), UI-код простой. Блок: подписка на `watchAll()` +
событие `refresh` (на старте и pull-to-refresh).

---

## 15. BLoC: event transformers, debounce, отмена

По умолчанию события обрабатываются **конкурентно** (bloc 8+). Управление — пакет
`bloc_concurrency`, указывается в `on<>`:

```dart
on<SearchChanged>(_onSearch, transformer: restartable());
```

- **`sequential()`** — строго по очереди (когда важен порядок / есть гонки за состояние).
- **`droppable()`** — игнорировать новые, пока идёт текущее (кнопка «Отправить» — без дублей).
- **`restartable()`** — отменять предыдущую обработку при новом событии (поиск/автокомплит —
  берём только последний запрос).
- **Debounce** ввода поиска — кастомный transformer на `debounceTime` (`stream_transform`) +
  `restartable`.
- Длинные async-обработчики на общем `on<BaseEvent>` → `sequential()` либо перечитывай `state`
  после каждого `await`.

---

## 16. Окружения (flavors / env)

- Не хардкодь `baseUrl`/ключи — выноси в конфиг окружения (dev/stage/prod). Варианты:
  `--dart-define`, отдельные `env.*.json`, flavors. Секретные ключи — **не в гит**.
- Какое окружение поднимать — выбираем в точке входа (§2.1), там же инициализируем
  сервисы под выбранный конфиг.

---

## 17. Формы и ввод

- `TextEditingController`/`FocusNode` — поля стейта; **обязательно `dispose()`** (иначе утечка).
- UI-состояние формы (введённое, тоглы) — в `State` через `setState`; сохранение/результат — в BLoC.
- Валидация — `Form` + `TextFormField(validator:)` + `formKey.currentState!.validate()`, либо
  ручная проверка перед отправкой. Не блокируй сохранение молча — показывай, что не так.
- Переход по полям — `textInputAction` / `onFieldSubmitted`.

---

## 18. Состояния экрана: loading / empty / error / data

- Любой экран с данными имеет 4 типовых вида: **загрузка / пусто / ошибка / данные**.
  Моделируй union'ом состояния (§5), разбирай `switch` в коннекторе (§7.1).
- Заведи общие `LoadingView` / `EmptyView` / `ErrorView(onRetry:)` в `shared/` — единый вид везде.
- «Пусто» ≠ «загрузка»: пустой список после загрузки — это `loaded([])` → показывай заглушку
  «пока ничего нет», а не спиннер.
- Ошибка — с кнопкой **«Повторить»** (шлёт событие перезагрузки).

---

## 19. Списки: refresh и пагинация

- Pull-to-refresh: `RefreshIndicator(onRefresh: () async => bloc.add(const Refresh()))`.
- Пагинация: лови приближение к концу (через `ScrollController` или `index == length - 1` в
  `builder`) → событие `LoadMore`. «Грузим ещё» / «есть ещё страницы» — это **данные страницы**
  (поля внутри `loaded`), а не отдельная фаза экрана.
- Длинные списки — только `ListView.builder`/slivers (§7.2).

---

## 20. Темизация (ThemeExtension, light/dark)

- Токены — через **`ThemeExtension`** (как `AppColors`): класс с `copyWith`/`lerp`, кладёшь в
  `ThemeData.extensions`, читаешь `Theme.of(context).extension<AppColors>()!` (обёртка
  `context.appColors`).
- Light/dark — два набора токенов; `MaterialApp(theme:, darkTheme:, themeMode:)`.
- Сырые `Color(0xFF…)` по коду не разносим: палитра → семантические токены → виджеты (§8).

---

## 21. Локализация (intl)

- Строки **не хардкодим** в виджетах — выносим в ARB (`flutter_localizations` + `intl`,
  `flutter gen-l10n`), доступ `AppLocalizations.of(context)`.
- Числа/даты/деньги — через `intl` (`DateFormat`, `NumberFormat`), не вручную.
- Даже если язык один — вынесенные строки проще менять и переиспользовать.

---

## 22. Адаптивность и доступность

- Размер/ориентация — `LayoutBuilder` / `MediaQuery.sizeOf`; брейкпоинты телефон/планшет;
  не верстай под один размер. Гибкость — `Flexible`/`Expanded`/`Wrap`.
- Доступность: тап-таргет ≥ 48dp, достаточный контраст, `Semantics`/label у иконок-кнопок,
  поддержка масштаба шрифта (не отключай `textScaler`).

---

## 23. Тесты (`bloc_test`, widget, `mocktail`)

- **Что тестировать в первую очередь:** блоки (логика) и мапперы/репозитории — чистая логика,
  быстро и ценно.
- BLoC — `bloc_test`: `blocTest(build: …, act: (b) => b.add(E()), expect: () => [состояния])`;
  зависимости — моки (`mocktail`).
- Мапперы — обычный unit-тест: dto → entity, проверь поля и парсинг крайних случаев.
- Виджеты — `testWidgets` + `pumpWidget` (оборачивай в нужные провайдеры с мок-блоком),
  проверяй отрисовку и тапы.
- Не гонись за 100% — покрывай логику и крайние случаи, не геттеры.

---

## 24. Надёжность: жизненный цикл, утечки, иммутабельность, производительность

- **Освобождай ресурсы:** контроллеры, `FocusNode`, `StreamSubscription`, `Timer` — в
  `dispose()`; в блоке отписывайся в `close()`.
- **После `await` проверяй живость:** `if (!mounted) return;` (виджет) / `if (isClosed) return;`
  (блок) перед использованием `context`/`emit`.
- **Иммутабельность коллекций:** не мутируй список/мапу в стейте (`list.add(x)`) — создавай
  новый (`[...list, x]`), иначе Bloc/freezed не увидят изменение и UI не обновится.
- **Производительность:** `const` и виджеты-классы (§7.2), builder-списки, без тяжёлых
  вычислений в `build` (выноси/кэшируй), `RepaintBoundary` для дорогих анимаций. Профилируй в
  DevTools, не угадывай.

---

## 25. Инструменты

- **Freezed-шпаргалка:** обычная модель — `@freezed class X with _$X { const factory X({...}) = _X; }`;
  union — `sealed class` + несколько `const factory X.a() = A;`; JSON — добавь
  `factory X.fromJson(...)`; `@Default(...)`, `@JsonKey(name: '...')` для имён полей; методы/геттеры —
  через приватный `const X._();`.
- **build_runner:** во время разработки `dart run build_runner watch --delete-conflicting-outputs`;
  разово — `build`. Сгенерированные файлы (`*.g.dart`/`*.freezed.dart`) обычно **коммитим**.
- **Линтер:** `analysis_options.yaml` с набором правил (`flutter_lints` / `very_good_analysis`) —
  он автоматически ловит часть правил отсюда (`prefer_const`, `unawaited_futures`, …).
  `flutter analyze` — перед коммитом / в CI.
- **Файлы:** один публичный класс — один файл; имя файла = `snake_case` от класса. Бэррел-экспорт
  (`export '...';`) — по желанию, чтобы укоротить импорты.
- **Разрешения (`permission_handler`):** запрос — через сервис-обёртку (§6.1); «спрашивали ли уже»
  храни в prefs; обрабатывай `denied` / `permanentlyDenied` (веди пользователя в настройки).

---

# Часть IV. Шпаргалки

## 26. Чек-лист новой фичи

`entity → DTO → таблица БД → mapper → (секреты) → репозиторий (интерфейс + impl) → bloc
(event/state/bloc) → provider → экран (коннектор + UI-виджеты).`

- [ ] Entity — freezed, без секретов/JSON, с `///`.
- [ ] DTO — freezed + `fromJson` (модель ответа API); вложенные — свои DTO.
- [ ] Таблица БД с именем строки (`XRow`) — если есть локальная БД.
- [ ] Mapper — один `abstract class <Имя>Mapper` на сущность, методы `static` (стрелка),
      имена уникальны: `toEntity`/`toDto` (сервер), `fromRow`/`toCompanion` (БД); вложенные
      сущности — свои мапперы; обратные направления — по необходимости.
- [ ] Repository — `abstract interface` + `Impl implements`; список реактивно (`watchAll()`
      для локальной БД); DI именованными `required`-параметрами.
- [ ] Bloc — sealed-события (freezed), состояние = union фаз (freezed), один `on<>`+`switch`,
      без автостарта в конструкторе.
- [ ] Provider — `RepositoryProvider` → `BlocProvider`, стартовое событие здесь.
- [ ] Экран — провайдер + коннектор; вёрстка в отдельных UI-виджетах; токены вместо хардкода.
- [ ] Логирование через `AppLogger`; ошибки залогированы.
- [ ] `build_runner build` прогнан, `flutter analyze` чистый.

---

## 27. Частые ошибки новичков (так НЕ надо)

- ❌ Обращаться к БД/`Dio` прямо из виджета. ✅ Только через BLoC.
- ❌ Один `State` с кучей флагов `isLoading`/`hasError`/`isEmpty`. ✅ Union фаз
  `loading/loaded/error`.
- ❌ `BlocBuilder`, в котором 200 строк вёрстки. ✅ Коннектор выбирает UI-виджет,
  вёрстка — в нём.
- ❌ Логика в `initState`/виджете (запросы, вычисления). ✅ В BLoC через события.
- ❌ Цвета/отступы числами в виджете. ✅ Токены.
- ❌ `print()` для отладки. ✅ `AppLogger`.
- ❌ Cubit «потому что проще». ✅ Полноценный Bloc с событиями.
- ❌ Запихивать две независимые заботы в один блок. ✅ Дробить на фичи (§3.1).
- ❌ Дёргать `SharedPreferences.getInstance()`/сырой клиент при каждом обращении, вне DI.
  ✅ Сервис-обёртка + DI, подключение один раз (§6.1).
- ❌ `Dio()` на каждый запрос / без таймаутов / `e.toString()` пользователю / парсинг
  `resp.data` прямо в виджете. ✅ Один `Dio` в DI с таймаутами и интерсепторами, api-сервис
  → DTO, `DioException` → доменная ошибка (§12).
- ❌ Мутировать список/мапу в состоянии (`list.add`). ✅ Новый объект (`[...list, x]`) (§24).
- ❌ Редактировать `*.freezed.dart`/`*.g.dart`. ✅ Менять исходник и гнать `build_runner`.
